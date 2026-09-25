# 2026-09 — Latency: fifty round trips to a database in another country

*Four pieces that belong to one investigation: per-stage timing in the logs, a one-hour spike measuring database latency from two regions, moving every Lambda from Tokyo to Singapore next to the database, and doubling the worker's memory for image preprocessing. A known-food message went from about three seconds to under half a second. Nothing user-visible changed except the wait.*

## The problem

In real use, messages were slow enough to hurt the experience. After drinks gained a zero-token parse path, the bot's latency was bimodal: a median of 2.95 s, a p90 of 13.5 s. The obvious story was "the LLM is slow, so keep more traffic away from it," and the proposal on the table was to pre-seed the cache of past AI estimates with common foods.

Two things were wrong with that story. Nobody knew where the non-LLM time went, because the logs only timed the model call. And the cache could not have helped the sentences that missed it: the zero-token path only accepts a message with an explicit quantity and unit, so "chicken leg bento" with no count skips the cache entirely.

## Decisions

**Time every stage before touching anything.** A tiny package marks stages and emits one log line per request path with each stage's milliseconds and a total, plus a label for which layer answered. The first real request on the LLM path took 14.2 s, of which the model was 6.5 s. The other 7.7 s was almost all sequential database round trips at 85 ms each: gates, profile reads, memory context, achievement checks. A known-food hit skipped the model but still made about fifty of them, which is why "zero tokens" still meant three seconds.

**Measure the fix before designing it.** The database is on Neon, which has no Tokyo region; the Lambdas were in Tokyo. Before writing a migration plan, two throwaway Lambdas ran the same query loop from each region against the same database:

| From | Per query (p50) | Connection setup |
|---|---|---|
| ap-southeast-1 (same region as the database) | 1.2 ms | 20 ms |
| ap-northeast-1 (where the Lambdas were) | 69–81 ms | 435–494 ms |

Sixty to seventy times. The compute moves to the data, since the data cannot move to Tokyo, and trimming round trips in code drops down the backlog.

**Stage the move so every step can prove it is a no-op.** The first pull request changed no region at all: it pinned the three S3 buckets to their old region (bucket names are global, and the CDN does not care where its origin lives), kept log history across the replacement, and split the deploy pipeline's region into per-environment values. The gate for merging it was a Terraform plan with zero destroys. Only then did one variable change move dev, and once dev had served real messages from the new region, the same change moved prod that night.

That staging paid for itself immediately. The first switch plan showed 7 adds and 0 destroys, not the ~60 replacements expected. In Terraform's AWS provider v6 a resource that does not declare a region keeps its old one when the provider's region changes, so the plan would have left the queues, APIs and tables in Tokyo while pointing IAM policies at Singapore ARNs that did not exist. It was aborted at review, and a second zero-diff pull request gave every regional resource an explicit region. The same review caught subnets with hardcoded Tokyo availability zones.

**Buy CPU with memory, after measuring the curve.** Image preprocessing (decode, resize, re-encode) took 1.8–2.5 s per photo, and Lambda allocates CPU in proportion to memory. A throwaway Lambda ran the same pipeline on a 12-megapixel photo at four sizes:

| Worker memory | Preprocessing (median) |
|---|---|
| 512 MB (before) | 4.65 s |
| 768 MB | 3.10 s |
| 1024 MB | 1.40 s |
| 1769 MB | 1.34 s |

The knee is at 1024 MB. In production the same photos dropped from 2.5 s to 0.9 s, and the cost stays inside Lambda's free monthly allowance at current traffic.

Two cheaper-looking alternatives were measured and rejected. A box-filter pre-shrink halved the resize time but produced moiré on thin lines like a label's small print, and a misread label is a wrong number in someone's log. Skipping our resize and letting the provider scale the image cost 23 % more input tokens and 2.5 s more model time.

## Results

| Path | Before | After |
|---|---|---|
| Known food or branded drink | ~3.1 s | 0.24–0.42 s |
| Pre-parse gates | 520–990 ms | 45 ms |
| After-record achievement checks | 0.8–1.5 s | 51 ms |
| Webhook gates | 0.3–1.1 s | 7 ms |
| Label photo preprocessing | 2.5 s | 0.9 s |

The LLM path also got faster, but most of that came from the model migration in the [next entry](2026-09-model-migration.md).

## What the process caught

The script that copied secrets to the new region piped each value through the CLI's text output, which ends with a newline, into a write that kept it. Every secret gained one byte, every LINE signature check failed, and the webhook answered each message with a 401 in about a millisecond without writing a log line. That signature looks exactly like "LINE is not calling us at all." The script now compares each secret's length in both regions without ever reading the value, and refuses to finish on a mismatch.

## Hindsight, honestly

- **Start from the measured wait, find the longest stage, then plan.** Real messages were slow enough to hurt the experience, and that is what started this work. The order is the point: measure every stage, find the one that takes the longest, and only then write the improvement plan.
