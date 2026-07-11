# Deep dive 2 — Deterministic parsing before the LLM

## The problem

Every message a user sends could go straight to an LLM — that's the lazy architecture, and it's wrong three ways:

1. **Cost & latency.** `help`, `確認`, `刪除` are unambiguous. Paying an LLM round-trip to classify them is pure waste.
2. **Reliability.** An LLM will occasionally misread `不對` as a food called "不對". For commands with side effects (delete, confirm, goal changes), "occasionally" is unacceptable.
3. **Testability.** Deterministic code gets deterministic tests. LLM-classified intents can only be tested statistically.

The design question isn't "rules or LLM" — it's **where exactly the line sits**, and how to keep it from drifting as features grow.

## The pipeline

Eleven rules run in a fixed order. Each rule sees a normalized view of the input and either claims it or passes:

```go
type Rule interface {
    Name() string
    Apply(input NormalizedInput) (ParsedMessage, bool)
}

type NormalizedInput struct {
    Raw     string // as received
    Trimmed string // whitespace-trimmed
    Lower   string // lowercased for ASCII matching
}
```

```
Help → PersonalFood → AssistedGoal → Goal → Preference → Summary
     → Confirm → Delete → Correction → RecordWeight → FoodLog
     → (no match) → LLM fallback
```

The runner is a first-match-wins loop; anything unclaimed falls through as `general_chat`, which is the LLM's cue:

```go
for _, rule := range rules {
    if msg, ok := rule.Apply(input); ok {
        // attach log-targeting / user nutrition where the intent supports it
        return msg
    }
}
return ParsedMessage{Intent: IntentGeneralChat, RawText: text, ...} // → LLM
```

The LLM parser *wraps* the rule parser rather than sitting beside it — callers see one `Parse` and don't know which path answered:

```go
type LLMParser struct {
    client      llm.Client
    localParser MessageParser // the rule pipeline
    timeout     time.Duration
    textCache   *textLRU      // profile-aware cache for LLM results
}
```

## Why the order matters

Order encodes precedence, and precedence encodes product decisions:

- **Help first.** `怎麼用` must never be interpreted as anything else, so it outranks everything.
- **Meta-commands (Summary/Confirm/Delete/Correction) before food logging.** `不對` is a correction to the last log, not a food. Rules with side effects on existing data get to claim input before rules that create data.
- **FoodLog last, and deliberately narrow.** It exact-matches only a handful of known strings. `吃了拿鐵` falls through to the LLM on purpose — free-text food parsing is exactly the job LLMs are good at. The rule layer's job is to protect the *commands*, not to compete with the LLM at its own game.

A concrete lesson baked into this pipeline: an earlier version let a bare `設定` trigger goal-setting, which misfired on messages that merely contained the word. The fix was to require explicit markers (`目標`, `每天`, `每日`, `goal`). Deterministic rules make this kind of precision *possible* — you can't patch an LLM prompt with the same confidence.

## The payoff, measured in tests

Each rule is a pure function of `NormalizedInput`, so the test suite covers the entire intent surface with table-driven unit tests — no DB, no network, no flakiness. The parser packages carry the densest test coverage in the codebase (~20 of the 40 test files), which is exactly where you want it: this layer decides whether the bot *deletes a user's data* or *logs a meal*.

## Trade-offs I accepted

- **Rules are zh-TW/English-specific.** Keyword rules don't generalize to new languages for free; adding one means revisiting the rule layer, not just the prompts.
- **The rule/LLM boundary needs maintenance.** Every new deterministic intent is a new rule *and* a new position in the precedence order. The fixed-order list keeps that decision explicit and reviewable in one place, rather than scattered across regexes.
- **First-match-wins hides overlaps.** Two rules that could both claim an input won't be flagged automatically; tests for rule *interaction* (not just each rule alone) guard against this.
