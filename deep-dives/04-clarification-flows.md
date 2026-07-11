# Deep dive 4 — Clarification flows: when the bot asks back

## The problem

Some messages can't be resolved in one turn, and guessing is worse than asking:

- A user sends a photo of a Taiwan/US **nutrition label**. The label says *per serving: 150 kcal* — but how much did they actually eat? Logging "one serving" by default silently corrupts their data.
- A user asks for a **TDEE-computed goal** but omitted their height. The bot needs one more fact before it can act, and shouldn't demand the whole form again.
- A **goal proposal** with caloric targets shouldn't write to the profile until the user says yes.

So the bot asks back — which means holding *pending state* between turns, in a system whose entry point is a stateless webhook and whose deployment might be serverless.

```
User:  [photo of a nutrition label]
Bot:   這是「每份 150 kcal」的營養標示，你實際吃了多少？（例如 1份 / 30g / 半包）
User:  半包
Bot:   已記錄 🏷️ ×0.5 包（150 kcal → 依半包換算）
```

## Design rule 1: pending state is a cache, not a database

A pending clarification is intentionally modeled as **expirable, loseable state**. Every store operation degrades to "no pending clarification found," and the worst-case outcome is always the same: the bot asks again. That single decision drives everything else:

- **TTL everywhere.** In-memory entries expire on read after 10 minutes; a stale "how much did you eat?" answered the next morning must not scale yesterday's label.
- **Failures degrade, never propagate.** If the DynamoDB store errors, it's logged and treated as a miss. A flaky dependency can make the bot slightly forgetful — it can never make the bot wrong.

The store is an interface with two implementations chosen by config:

```go
// memory: in-process TTL map guarded by a mutex — single-server & local dev
type InMemoryClarificationStore struct {
    ttl   time.Duration
    clock func() time.Time // injected for deterministic expiry tests
    mu    sync.Mutex
    data  map[string]PendingClarification // key = userID
}

// dynamodb: single-table design, pk = "clarification#" + userID,
// expires_at drives DynamoDB TTL — for serverless, multi-instance deployments
```

The injected `clock` is a small thing that pays for itself: TTL expiry is unit-tested by advancing a fake clock, not by sleeping.

## Design rule 2: the next message must *earn* the match

The dangerous part of any "pending question" system is capturing an unrelated next message as the answer. The follow-up matcher is written to fall through aggressively:

```go
func (w *Worker) tryHandleAsLabelFollowUp(ctx context.Context, job Job) (bool, error) {
    pending, ok, _ := w.labelStore.GetActiveForUser(ctx, job.UserID)
    if !ok {
        return false, nil // nothing pending → normal processing
    }
    amount, parsed := ParseConsumedAmount(job.RawText) // 1份 / 30g / 半包 / 2 servings
    if !parsed {
        if looksLikeQuantityAttempt(job.RawText) {
            // user tried to answer but was ambiguous → re-ask with examples
            return true, w.replyWithToken(ctx, job, composeAmbiguousFollowUpReply(...))
        }
        return false, nil // unrelated message → fall through, pending stays alive
    }
    scaled, _ := ScaleNutritionForConsumed(pending.Label, amount)
    // create the diet log from scaled values, then consume the pending entry
    w.labelStore.MarkConsumed(ctx, pending.ID)
    return true, nil
}
```

Three outcomes, deliberately distinguished:

1. **Parseable amount** → scale the label, log it, consume the pending entry.
2. **Looks like an attempt but ambiguous** (`一點點`) → re-ask with concrete examples, keep the pending entry.
3. **Clearly unrelated** (`今天天氣如何`) → process normally; the pending question quietly waits out its TTL.

Case 3 is the one most implementations get wrong — trapping the user in a question they've moved past.

## Design rule 3: resume before parse

For service-level clarifications (missing TDEE fields, unconfirmed goal proposals), the resume check runs *before* the normal parse pipeline: load pending state for the user, and if the incoming text plausibly answers it (`YES` / `確認` / a bare number for a missing weight), apply it in context. The original parsed message is carried inside `PendingClarification` so nothing is re-derived — the conversation resumes exactly where it paused, one missing field at a time.

## Trade-offs I accepted

- **One pending clarification per user.** New pending state overwrites old. Multi-question stacks would add real complexity for a case users barely hit; last-question-wins matches how chat actually feels.
- **Deterministic follow-up matching only.** `ParseConsumedAmount` is regex-based, not LLM-based — consistent with the project's wider rule: side-effectful decisions (what gets logged) prefer deterministic code, with the LLM upstream of them.
- **Two storage backends to maintain.** The interface is small (get/upsert/consume), and the memory implementation doubles as the test double for everything above it.
