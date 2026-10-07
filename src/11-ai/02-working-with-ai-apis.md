# Working with AI APIs

> **🔄 Current (as of October 2026):** In .NET, **Microsoft.Extensions.AI** provides
> provider-neutral abstractions (`IChatClient`, `IEmbeddingGenerator`) with implementations
> for major providers. Provider SDKs (Anthropic, OpenAI, Azure) offer full access to
> provider-specific features. API parameter names and features differ between providers and
> evolve; check current docs.

Chapter 1 described what an LLM call is. This chapter makes those calls from Beacon, and
handles everything production adds: authentication, streaming, timeouts, retries, rate
limits, errors, cost tracking, prompt caching, and abstractions that keep Beacon from being
locked into one provider.

An LLM API is, from the application's point of view, an HTTP dependency that is **slow,
expensive, rate-limited and occasionally unavailable**, and whose output needs validation.
Everything from Book III about calling HTTP services applies, with some new twists.

---

## 1. The problem: a slow, metered, probabilistic dependency

Compared with Beacon's other dependencies:

| | PostgreSQL query | LLM call |
|---|---|---|
| Latency | 1–20 ms | 0.5–60+ seconds |
| Cost per call | Negligible | Fractions of a cent to dollars |
| Rate limits | Connection pool | Requests and tokens per minute, per organization |
| Failure modes | Timeouts, errors | Timeouts, 429s, 5xx overloads, truncated output, refusals, wrong answers |
| Output | Exact | Variable, needs validation |

A naïve integration (a synchronous call inside a request handler, no timeout, no streaming)
produces a slow, fragile, expensive feature.

---

## 2. The mental model: the request lifecycle

```text
 Beacon.Api
  ├─ build context (system prompt, retrieved docs, history)       ← Chapters 3, 6
  ├─ check budget / rate limits                                   ← Chapter 9
  ├─ call provider (HTTPS, auth, timeout, retry with back-off)
  │    └─ stream tokens back as they're generated ──► BFF ──► browser (SSE)
  ├─ validate output (structure, citations, safety)                ← Chapters 4, 8, 9
  ├─ record telemetry: model, tokens, latency, cost, outcome      ← Chapter 9
  └─ return / persist result
```

---

## 3. Providers, SDKs and abstractions

### Direct provider SDKs

Each provider publishes SDKs (Python, TypeScript, often C#/.NET, Go, Java). They expose every
provider-specific feature (tool use details, prompt caching controls, extended thinking,
batch APIs, file uploads).

### Cloud platforms

Models are also available through **Microsoft Foundry** (formerly Azure AI Foundry; including
Azure OpenAI and other providers' models), **Amazon Bedrock** and **Google Vertex AI**. Reasons
to use them: enterprise agreements and billing, data residency in your cloud region, private
networking (Book IX, Chapter 5), managed identity authentication (Book IX, Chapter 2), and
centralized governance.

### Provider-neutral abstractions

`Microsoft.Extensions.AI` defines interfaces that application code depends on:

```csharp
public interface IChatClient : IDisposable
{
    Task<ChatResponse> GetResponseAsync(IEnumerable<ChatMessage> messages, ChatOptions? options = null, CancellationToken ct = default);
    IAsyncEnumerable<ChatResponseUpdate> GetStreamingResponseAsync(IEnumerable<ChatMessage> messages, ChatOptions? options = null, CancellationToken ct = default);
    // ...
}
```

Implementations exist for major providers; middleware (like ASP.NET Core's pipeline, Book III,
Chapter 2) adds caching, logging, OpenTelemetry and function invocation around any client:

```csharp
builder.Services.AddChatClient(sp => CreateProviderClient(sp))      // provider-specific construction
    .UseOpenTelemetry()                                              // traces and metrics (GenAI semantic conventions)
    .UseLogging()
    .UseFunctionInvocation();                                        // tool calling loop (Chapter 4)
```

> **🧭 When not to abstract:** Abstractions keep business code portable and testable (fake
> `IChatClient` in tests), but they expose the common subset. When you need a provider-specific
> feature (fine-grained prompt caching, a specific tool type, batch processing), use the
> provider SDK for that path, behind your own interface. Don't contort the design to stay
> "neutral" at the cost of capabilities you need.

---

## 4. Authentication and configuration

- **API keys** are secrets: Key Vault, managed identity to read them (Book IX, Chapter 2),
  never in code or the frontend.
- **Never call LLM APIs from the browser** with your key. The BFF/API calls the provider; the
  browser talks only to Beacon.
- **Cloud platforms with Entra ID** (Microsoft Foundry) can use managed identities: no key at all.
- **Model IDs in configuration**, not code, so you can switch models per environment and per
  feature without redeploying:

```json
"Ai": {
  "Provider": "anthropic",
  "Models": {
    "ReplyDraft": "<large-model-id>",
    "Classification": "<small-model-id>",
    "Summary": "<small-model-id>"
  },
  "TimeoutSeconds": 60
}
```

---

## 5. Streaming

Generating a 400-token reply can take several seconds. Waiting for the whole response before
showing anything feels broken. **Streaming** returns tokens as they're generated, so users see
text appear within a second.

Providers stream with **Server-Sent Events** (Book III, Chapter 8). In .NET:

```csharp
await foreach (var update in chat.GetStreamingResponseAsync(messages, options, ct))
{
    if (update.Text is { Length: > 0 } text)
        await writer.WriteAsync(text, ct);     // forward to the client
}
```

Beacon streams end to end: provider → Beacon.Api (`IAsyncEnumerable`) → SSE endpoint
(`TypedResults.ServerSentEvents`, .NET 10) → BFF (YARP passes SSE through; disable response
buffering) → React (reading the stream). Section 9 shows the full path.

Streaming complications:

- **Errors can occur mid-stream** after a `200 OK` was sent; the protocol needs an error event.
- **Validation is harder**: you can't validate a complete answer before the user sees the
  beginning. For outputs that must be validated first (structured data, Chapter 4), don't stream
  to the user; stream internally if useful.
- **Cancellation**: if the user navigates away, cancel the upstream request (`CancellationToken`
  all the way through; Book I, Chapter 9) so you stop paying for tokens nobody will read.

---

## 6. Reliability: timeouts, retries and rate limits

### Errors you'll see

| Status / condition | Meaning | Action |
|---|---|---|
| 400 | Invalid request (too many tokens, bad parameters) | Fix the request; don't retry |
| 401 / 403 | Auth problem | Alert; don't retry |
| **429** | Rate limited (requests or tokens per minute) | Retry after `Retry-After`, back off, queue |
| **500 / 503 / "overloaded"** | Provider-side issue | Retry with exponential back-off and jitter; consider fallback model/provider |
| Timeout | Slow generation or network | Retry once if idempotent; tune timeouts per feature |
| `stop_reason: max_tokens` | Output truncated | Increase limit, shorten the task, or handle partial output |
| Refusal | Model declined (policy) | Handle gracefully in UX; review the prompt if unexpected |

### Resilience configuration

The HTTP client resilience handler (Book III, Chapter 1) applies, with longer timeouts than
typical APIs:

```csharp
builder.Services.AddHttpClient("ai")
    .AddStandardResilienceHandler(o =>
    {
        o.AttemptTimeout.Timeout = TimeSpan.FromSeconds(60);
        o.TotalRequestTimeout.Timeout = TimeSpan.FromSeconds(150);
        o.Retry.MaxRetryAttempts = 2;                          // retries on 429/5xx, honoring Retry-After
        o.CircuitBreaker.SamplingDuration = TimeSpan.FromSeconds(120);
    });
```

Retrying a generation is safe (no side effects at the provider), but costs money and time;
cap attempts.

### Rate limits and throughput

Providers limit **requests per minute** and **tokens per minute** (input and output). Under load:

- **Queue** non-interactive work (summaries, classification of new tickets) through Book III,
  Chapter 8's background workers, with concurrency limits matched to your quota.
- **Prioritize** interactive requests (an agent waiting for a draft) over batch work.
- **Batch APIs**: many providers offer asynchronous batch processing at a discount, ideal for
  bulk jobs like reclassifying 100,000 historical tickets.
- **Fallbacks**: a secondary model or provider for critical paths, behind the same abstraction.

---

## 7. Cost and latency levers

Cost ≈ input tokens × input price + output tokens × output price (output tokens typically cost
several times more). Levers:

- **Right-size the model** per task (Chapter 1).
- **Control output length**: `max_tokens`, and instructions like "answer in at most 5 sentences."
- **Trim input**: retrieve only relevant passages (Chapter 6); summarize long histories.
- **Prompt caching**: providers can cache a stable **prefix** of the prompt (system prompt,
  tool definitions, a large document) so repeated calls reuse it at a fraction of the input
  cost and with lower latency. Structure prompts with **stable content first, variable content
  last** to benefit.
- **Response caching** (application-level) for identical requests: e.g. the summary of a ticket
  thread that hasn't changed, keyed by a hash of the thread content (Book III, Chapter 7).
- **Batch** non-urgent work.

Chapter 9 adds budgets and per-feature cost tracking.

---

## 8. Testing AI integrations

- **Unit tests**: fake `IChatClient` returning canned responses, to test your code around the
  model (parsing, validation, error handling, prompt construction).
- **Contract tests**: record real responses once (with sensitive data removed) and replay them,
  to catch provider format changes.
- **Evaluation** (Chapter 8): test the *quality* of model outputs with datasets and scoring, a
  different activity from unit testing.
- **Never call real models in fast unit test suites**: slow, costly, non-deterministic.

---

## 9. In practice: Beacon's AI gateway and a streaming summary

### An internal AI service

Beacon wraps model access in one place, so policies (timeouts, telemetry, budgets, safety
checks) apply consistently:

```csharp
// src/Beacon.Core/Ai/IAiFeatures.cs
namespace Beacon.Core.Ai;

public enum AiFeature { ReplyDraft, Classification, ThreadSummary, Assistant }

public interface IAiChat
{
    IAsyncEnumerable<string> StreamAsync(AiFeature feature, AiPrompt prompt, CancellationToken ct);
    Task<AiResult> CompleteAsync(AiFeature feature, AiPrompt prompt, CancellationToken ct);
}

public sealed record AiPrompt(string System, IReadOnlyList<AiMessage> Messages, int MaxOutputTokens);
public sealed record AiMessage(string Role, string Content);
public sealed record AiResult(string Text, string StopReason, int InputTokens, int OutputTokens, string Model);
```

```csharp
// src/Beacon.Infrastructure/Ai/AiChat.cs (sketch)
public sealed class AiChat(IChatClient chat, IOptions<AiOptions> options, ILogger<AiChat> log) : IAiChat
{
    public async IAsyncEnumerable<string> StreamAsync(
        AiFeature feature, AiPrompt prompt, [EnumeratorCancellation] CancellationToken ct)
    {
        var model = options.Value.Models[feature.ToString()];
        var chatOptions = new ChatOptions { ModelId = model, MaxOutputTokens = prompt.MaxOutputTokens, Temperature = 0.2f };
        var messages = new List<ChatMessage> { new(ChatRole.System, prompt.System) };
        messages.AddRange(prompt.Messages.Select(m => new ChatMessage(new ChatRole(m.Role), m.Content)));

        using var activity = BeaconTelemetry.Activities.StartActivity($"ai.{feature}");
        await foreach (var update in chat.GetStreamingResponseAsync(messages, chatOptions, ct))
        {
            if (!string.IsNullOrEmpty(update.Text)) yield return update.Text;
        }
        // token usage and finish reason arrive in the final update(s): recorded via OpenTelemetry middleware (Chapter 9)
    }

    // CompleteAsync: same, non-streaming, returning usage and stop reason; logs a warning when truncated.
}
```

### A streaming thread summary endpoint

```csharp
// Beacon.Api: GET /api/tickets/{id}/summary (SSE)
group.MapGet("/{id}/summary", async (TicketId id, ITicketRepository repo, IAuthorizationService authz,
                                    ClaimsPrincipal user, IAiChat ai, CancellationToken ct) =>
{
    var ticket = await repo.FindAsync(id, ct);
    if (ticket is null || !(await authz.AuthorizeAsync(user, ticket, TicketOperations.Work)).Succeeded)
        return Results.NotFound();

    var prompt = ThreadSummaryPrompt.Build(ticket);       // Chapter 3: comments as data, clear instructions
    return TypedResults.ServerSentEvents(ai.StreamAsync(AiFeature.ThreadSummary, prompt, ct), eventType: "token");
});
```

Authorization comes **first** (Book III, Chapter 6): the model never sees a ticket the user
can't see.

### Consuming the stream in React

```ts
// web/src/features/tickets/api/useThreadSummary.ts
export async function streamSummary(ticketId: string, onToken: (t: string) => void, signal: AbortSignal) {
  const res = await fetch(`/api/tickets/${encodeURIComponent(ticketId)}/summary`, {
    headers: { Accept: 'text/event-stream', 'X-CSRF': '1' }, credentials: 'include', signal,
  });
  if (!res.ok || !res.body) throw new ApiError(res.status);

  const reader = res.body.pipeThrough(new TextDecoderStream()).getReader();
  let buffer = '';
  for (;;) {
    const { value, done } = await reader.read();
    if (done) break;
    buffer += value;
    const events = buffer.split('\n\n');
    buffer = events.pop() ?? '';
    for (const evt of events) {
      // A string item is written as raw text; multi-line text arrives as several "data:" lines
      const lines = evt.split('\n').filter(l => l.startsWith('data:')).map(l => l.replace(/^data: ?/, ''));
      if (lines.length) onToken(lines.join('\n'));
    }
  }
}
```

(The browser's `EventSource` doesn't support custom headers, so `fetch` with a stream reader is
used; libraries can simplify this.) The component appends tokens to state as they arrive, shows
"AI summary — may contain mistakes; check the thread," and aborts the request on unmount (Book
VI, Chapter 2).

---

## 10. What can go wrong

- **API keys in the frontend** or in source control.
- **Synchronous, unstreamed calls** making the UI feel frozen.
- **No timeouts or retry caps**, holding requests open for minutes.
- **Retry storms** under rate limiting, making 429s worse.
- **Not cancelling upstream** when users leave: paying for unread tokens.
- **Ignoring truncation** (`max_tokens`).
- **Hard-coded model names** that break when models are retired.
- **Calling real models in unit tests.**
- **Sending data to a provider** without checking data processing terms and customer agreements
  (Chapter 9).

---

## 11. How an experienced engineer thinks about this

- **Treat the model as a slow, metered, unreliable dependency** with all the usual resilience
  patterns.
- **Stream for interactive features; queue for background ones.**
- **One gateway for policies**: timeouts, telemetry, budgets, safety.
- **Abstract for portability and testing**, but use provider features when they matter.
- **Authorization before context**: never put data in a prompt the user couldn't see directly.

---

## 12. Check yourself

**Questions**

1. How does an LLM dependency differ from a database dependency?
2. Why should business code depend on an abstraction like `IChatClient`? When is that not enough?
3. Why stream? What does streaming complicate?
4. Which errors should be retried, and how?
5. What is prompt caching, and how should prompts be structured to benefit?
6. Why cancel upstream calls when the client disconnects?
7. Why must authorization happen before building the prompt?

**Exercises**

1. Implement `IAiChat` with `Microsoft.Extensions.AI` against a provider of your choice, with
   OpenTelemetry and resilience configured.
2. Build the streaming summary endpoint and React consumer; verify cancellation stops the
   upstream request.
3. Write unit tests for prompt construction and error handling with a fake `IChatClient`.
4. Measure latency and cost for the same summary with a small and a large model.

**Interview-style questions**

- "How would you integrate an LLM API into a production web application?"
- "How do you handle rate limits and failures from an AI provider?"
- "How do you stream LLM responses to a browser?"

---

## 13. Going deeper

- [Microsoft docs: Microsoft.Extensions.AI](https://learn.microsoft.com/dotnet/ai/microsoft-extensions-ai)
- [Anthropic API documentation](https://docs.claude.com/en/api/overview), including streaming,
  prompt caching and batch processing.
- [Microsoft Foundry documentation](https://learn.microsoft.com/azure/ai-foundry/)

**Next:** [Chapter 3 — Prompt Engineering](03-prompt-engineering.md) designs the instructions
and context that determine what the model produces.
