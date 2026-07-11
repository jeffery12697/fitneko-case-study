# Deep dive 1 — Async intake: acknowledge fast, reply later

## The problem

A LINE webhook delivers a message and expects a prompt HTTP 200. The work FitNeko needs to do — LLM parsing, nutrition estimation, sometimes downloading and analyzing an image — takes seconds, occasionally tens of seconds with retries. Blocking the webhook on that is a non-starter: LINE retries slow webhooks, retries mean duplicate processing, and one slow LLM call would stall every user behind it.

The complication is that replying on LINE is cheapest via the **reply token** that arrives with the event — but reply tokens are single-use and expire in roughly a minute. So the design has to decouple *receiving* from *replying* while treating the token as a perishable resource.

## The shape of the solution

```
webhook  →  INSERT intake_jobs (status=pending, reply_token, received_at)  →  HTTP 200
                                    ↓ (background, 5s poll)
worker   →  claim job (FOR UPDATE SKIP LOCKED)  →  parse / analyze / persist
         →  reply IF token still inside its 55-second window
```

Everything a future step needs — including the reply token and *when it arrived* — is written to PostgreSQL before the webhook returns. There is no in-process queue to lose on restart.

## Idempotency at the front door

LINE may redeliver an event. The enqueue is an upsert keyed on LINE's message ID:

```sql
INSERT INTO intake_jobs (line_message_id, ...) VALUES (...)
ON CONFLICT (line_message_id) DO NOTHING
```

Redelivery becomes a no-op instead of a duplicate diet log. This one constraint replaces a whole class of dedup logic downstream.

## Claiming work without a queue broker

The worker polls PostgreSQL rather than using a message broker — a deliberate simplicity choice at this scale. Claiming is race-safe via row locking:

```go
// ClaimNextPending: one transaction —
// SELECT ... WHERE status='pending' ... FOR UPDATE SKIP LOCKED
// then UPDATE status='processing', attempts = attempts + 1, started_at = now()
```

`SKIP LOCKED` lets multiple workers coexist without ever fighting over a row, and a job stuck in `processing` for more than 10 minutes becomes claimable again — crash recovery without a supervisor.

The worker loop itself is deliberately boring:

```go
func (w *Worker) Run(ctx context.Context, pollInterval time.Duration) {
    for {
        processed, err := w.ProcessOne(ctx)
        if err != nil {
            log.Printf("worker: ProcessOne error: %v", err)
        }
        if !processed {
            // nothing to do — sleep one interval, respecting ctx cancellation
        }
    }
}
```

There is also a `ProcessJobByID` entry point, so the same worker code runs in a queue-triggered deployment (SQS → Lambda) without modification — the polling loop is just one of two drivers.

## The reply token as a perishable resource

The token's age is checked at *send time*, not at receive time, because the delay happens in between:

```go
const replyTokenSafeWindow = 55 * time.Second

func (w *Worker) replyWithToken(ctx context.Context, job Job, text string) error {
    if job.ReplyToken == "" {
        _, _ = w.repo.MarkReplyMissing(ctx, job.ID)
        return nil
    }
    if job.ReplyTokenReceivedAt == nil ||
        time.Since(*job.ReplyTokenReceivedAt) > replyTokenSafeWindow {
        _, _ = w.repo.MarkReplyExpired(ctx, job.ID)
        return nil
    }
    if err := w.replyClient.ReplyText(ctx, job.ReplyToken, text); err != nil {
        _, _ = w.repo.MarkReplyFailed(ctx, job.ID, safeErrorMessage(err))
        return err
    }
    _, _ = w.repo.MarkReplySent(ctx, job.ID)
    return nil
}
```

55 seconds is a safety margin under LINE's ~60-second token lifetime. Note what happens on expiry: the outcome is *recorded* (`reply_status = expired`) rather than swallowed. The diet log still gets written — the user's data is never held hostage by a messaging deadline — and the reply ledger makes "user never got an answer" a queryable condition instead of a mystery.

## Trade-offs I accepted

- **Polling latency.** Up to 5 seconds of added latency per message. Acceptable for a diet logger; the `JobNotifier` hook exists to switch to push (SQS) when it isn't.
- **No automatic retry backoff per job.** A failed job is marked `failed` with a truncated error; stale-claim recovery handles crashes, but poison messages need manual attention. Fine at current volume, revisit with scale.
- **No push-message fallback on token expiry.** LINE push messages have quota costs; for now an expired reply is logged and dropped. The data model already records enough to add a fallback later.
