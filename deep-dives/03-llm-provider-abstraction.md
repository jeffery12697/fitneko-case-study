# Deep dive 3 — One interface, two LLM providers

## The problem

FitNeko needs structured data out of an LLM — a parsed food log, an intent label, a nutrition estimate — never prose. I wanted to run either OpenAI or Anthropic behind a config flag (`LLM_PROVIDER`), which surfaced a real impedance mismatch: **the two providers enforce structured output in fundamentally different ways.**

- **OpenAI (Responses API):** native structured outputs — you attach a JSON schema with `"strict": true` and the API guarantees conforming JSON.
- **Anthropic (Messages API):** no direct equivalent at the time; the reliable pattern is **tool-use forcing** — define a single tool whose input schema is your output schema, set `tool_choice: {type: "tool", name: ...}`, and read the structured payload out of the `tool_use` content block.

Same contract, two very different wire protocols.

## The interface

The abstraction is task-shaped, not chat-shaped. There is no `SendMessage(prompt) string` — every method names a domain operation and returns a typed result:

```go
type Client interface {
    AnalyzeDietText(ctx context.Context, input UserInput) (AnalyzeDietTextResult, error)
    ClassifyIntent(ctx context.Context, input UserInput) (IntentResult, error)
    ParseDietLog(ctx context.Context, input UserInput) (DietParseResult, error)
    EstimateNutrition(ctx context.Context, input NutritionEstimateInput) (NutritionEstimateResult, error)
    ComposeReply(ctx context.Context, input ReplyInput) (ReplyResult, error)
}
```

This choice did most of the heavy lifting. Because the interface speaks in domain results, each provider is free to achieve "give me exactly this JSON" however its API allows — schema mode for OpenAI, forced tool-use for Anthropic — and nothing above the `llm` package knows the difference. A `mock` implementation of the same interface gives fully deterministic local development with zero API keys.

`AnalyzeDietText` deserves a note: it's a *combined* call (intent + parse + estimate in one round-trip) with the separate calls kept as a fallback path — one API call instead of three for the common case, without betting everything on the combined prompt.

## Shared retry, provider-specific requests

Retry policy is a property of *talking to an LLM over HTTP*, not of any one provider, so it lives once:

```go
// doHTTPWithRetry: exponential backoff from 500ms,
// retries 429 / 5xx / network errors, honors Retry-After.
// Takes a request *factory*, since a request body can't be replayed.
func doHTTPWithRetry(ctx context.Context, httpClient *http.Client,
    reqFactory func() (*http.Request, error), provider string, cfg retryConfig,
) ([]byte, error)
```

The request-factory signature is the subtle part: retrying an `*http.Request` naively fails because its body reader is already consumed. Both providers pass a closure that rebuilds the request, and the retry loop stays provider-agnostic.

## Where the abstraction stops: vision

Image analysis (meal photos, nutrition labels) routes to OpenAI **unconditionally**, even when `LLM_PROVIDER=anthropic`. That's a deliberate asymmetry: the vision task was built and tuned against one provider, and pretending the abstraction covered vision would have meant either blocking Anthropic text support on vision parity, or shipping an untested vision path.

The general principle: **abstract what you actually need to swap, and let the rest be concrete.** A leaky abstraction that admits its leak in one obvious place (`NewOpenAIVisionAnalyzer` in the composition root) beats a uniform one that hides an untested code path.

## Trade-offs I accepted

- **Two prompt implementations to keep in sync.** Each domain method has per-provider prompt/schema plumbing. The typed interface at least guarantees they can't drift in *shape*, only in behavior — and behavior is covered by shared contract tests.
- **Vision lock-in.** Swapping vision providers later is real work, not a config change. Accepted consciously rather than paid for speculatively.
- **Combined-call complexity.** `AnalyzeDietText` plus its decomposed fallback is more code than either alone; the latency and cost savings on the hot path justified it.
