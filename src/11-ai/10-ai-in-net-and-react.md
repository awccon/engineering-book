# AI in .NET and React

The previous chapters covered AI concepts and individual techniques. This chapter assembles
them into Beacon as a product: the backend architecture that hosts AI features in .NET, the
frontend patterns that make AI output usable and trustworthy in React, and the Azure services
that run it all. It's Book VII's "how the pieces fit" for AI.

---

## 1. The problem: AI features are full-stack features

An AI reply draft isn't just a prompt. It's:

- a UI affordance in the ticket view ("Draft reply"), with loading, streaming and error states,
- an authorized API endpoint that builds context from permitted data,
- retrieval over the knowledge base,
- a model call through a gateway with timeouts, budgets and telemetry,
- output validation and safety checks,
- an editing experience where the agent changes the draft before sending,
- feedback capture and evaluation,
- per-tenant settings and kill switches.

Each piece uses something from earlier books. The engineering challenge is assembling them so AI
features feel native, safe and maintainable.

---

## 2. The mental model: AI as a layer in the architecture

```text
 React (features/*)
  ├─ AI affordances: "Draft reply", "Summarize", assistant panel
  ├─ streaming UI, sources, citations, feedback, "AI-generated" labels
  └─ calls Beacon API via the BFF (Book VI)
        │
 Beacon.Api
  ├─ Endpoints: authorize, rate-limit, check tenant AI settings
  ├─ Application services: ReplyDraftService, HelpAssistant, TicketTriage, IncidentAgent
  │    ├─ AiContextBuilder: loads permitted data via normal services/repositories
  │    ├─ Retrieval: hybrid search over article_chunks (pgvector + full-text)
  │    └─ Output validators: schemas, citations, safety rules
  ├─ AI gateway (IAiChat / IChatClient pipeline): model routing, timeouts, retries,
  │    OpenTelemetry, token/cost accounting, budgets, kill switches
  └─ Background: triage worker, indexer (outbox-driven), eval sampling
        │
 Model providers / Microsoft Foundry · PostgreSQL + pgvector · App Configuration (flags) · App Insights
```

> **🧱 Durable:** AI is a **dependency**, not an architecture. The same layering that kept Beacon
> maintainable (domain, application services, infrastructure, thin endpoints) applies: prompts and
> model calls live behind interfaces in infrastructure; business rules stay in code.

---

## 3. The .NET building blocks

> **🔄 Current (as of October 2026):** The .NET AI stack centers on **Microsoft.Extensions.AI**
> (abstractions and middleware), **Microsoft.Extensions.VectorData** (vector store abstractions with
> connectors including PostgreSQL), **Microsoft Agent Framework** (agents and workflows), the
> **ModelContextProtocol** C# SDK, provider SDKs, and **Aspire** integrations for local orchestration
> and telemetry. Package names and APIs are still evolving; pin versions and follow release notes.

| Need | Building block |
|---|---|
| Chat and embeddings, provider-neutral | `IChatClient`, `IEmbeddingGenerator<,>` |
| Cross-cutting middleware | `.UseOpenTelemetry()`, `.UseLogging()`, `.UseDistributedCache()`, `.UseFunctionInvocation()`, custom `DelegatingChatClient`s |
| Typed structured output | `GetResponseAsync<T>()` (Chapter 4) |
| Tools | `AIFunctionFactory.Create(method)` |
| Vector search | pgvector via EF Core/Npgsql, or `Microsoft.Extensions.VectorData` connectors |
| Agents and workflows | Microsoft Agent Framework (Chapter 7) |
| Exposing tools to other AI clients | MCP C# SDK (Chapter 4) |
| Evaluation in .NET | `Microsoft.Extensions.AI.Evaluation` libraries (quality and safety evaluators), complementing the Python harness (Chapter 8) |

### Custom middleware: budgets and kill switches

The `IChatClient` pipeline is where Beacon enforces policies once, for every feature:

```csharp
// src/Beacon.Infrastructure/Ai/PolicyChatClient.cs
public sealed class PolicyChatClient(IChatClient inner, IFeatureManager flags, IAiBudget budget,
                                     ICurrentTenant tenant, ILogger<PolicyChatClient> log)
    : DelegatingChatClient(inner)
{
    public override async Task<ChatResponse> GetResponseAsync(
        IEnumerable<ChatMessage> messages, ChatOptions? options = null, CancellationToken ct = default)
    {
        var feature = options?.AdditionalProperties?.GetValueOrDefault("beacon.feature") as string ?? "unknown";

        if (!await flags.IsEnabledAsync($"Ai.{feature}") || !await tenant.AiEnabledAsync(ct))
            throw new AiFeatureDisabledException(feature);                     // mapped to 503/feature-off UX

        if (!await budget.TryReserveAsync(tenant.Id, feature, ct))
            throw new AiBudgetExceededException(feature);                      // mapped to 429

        var response = await base.GetResponseAsync(messages, options, ct);
        await budget.RecordAsync(tenant.Id, feature, response.Usage, options?.ModelId, ct);
        return response;
    }

    // GetStreamingResponseAsync: same checks before streaming; usage recorded from the final update
}
```

```csharp
// registration
builder.Services.AddChatClient(sp => CreateProviderClient(sp))
    .Use((inner, sp) => ActivatorUtilities.CreateInstance<PolicyChatClient>(sp, inner))
    .UseOpenTelemetry(sourceName: "Beacon.Ai", configure: o => o.EnableSensitiveData = false)
    .UseLogging();
```

`EnableSensitiveData = false` keeps prompt and response content out of telemetry (Chapter 9).

---

## 4. Backend patterns for AI features

### Pattern: on-demand, streamed, user-initiated

The agent clicks "Draft reply." The endpoint authorizes, builds context, streams the draft, validates
at the end:

```csharp
group.MapPost("/{id}/reply-draft", async (TicketId id, ReplyDraftService drafts, CancellationToken ct) =>
    TypedResults.ServerSentEvents(drafts.StreamAsync(id, ct)))
    .RequireAuthorization(Policies.Staff)
    .RequireRateLimiting("ai-drafts");
```

### Pattern: background enrichment

New tickets are classified and embedded by workers (Chapters 4–5), driven by outbox events, with
results stored as data the UI displays like any other field (`ProductArea`, `SuggestedPriority`,
`SimilarTickets`). The UI never waits for the model.

### Pattern: cached derived content

Thread summaries are expensive to regenerate on every view. Cache them keyed by ticket ID **and a hash
of the thread content** (invalidated naturally when a comment is added), via HybridCache (Book III,
Chapter 7).

### Pattern: human-confirmed actions

Suggestions (priority, assignee, incident links) are stored as **proposals** with the AI decision ID;
accepting one calls the normal domain method (`ticket.Escalate()`), so domain rules, authorization,
audit and events apply exactly as for manual actions.

### Testing

- **Unit tests** with fake `IChatClient`s for context building, validation and error handling.
- **Integration tests** with a deterministic fake model server (canned SSE streams) through the real
  pipeline: authorization, rate limits, streaming and policy middleware.
- **Evaluation** (Chapter 8) for quality, separately and continuously.

---

## 5. Frontend patterns: UX for AI

AI output is probabilistic and sometimes wrong. The UI's job is to make it **useful when right and
harmless when wrong**.

### Principles

| Principle | Implementation |
|---|---|
| **Make AI output recognizable** | Label ("AI draft," "AI summary"), distinct styling, an icon |
| **Show the evidence** | Sources and citations, clickable, near the claims |
| **Keep humans in control** | Drafts go into an editable field; suggestions need accept/dismiss; nothing irreversible from one click |
| **Stream and show progress** | Tokens appear as generated; sources first; skeletons; elapsed time for long tasks |
| **Allow stopping** | A Stop button that aborts the request (and the upstream model call) |
| **Graceful failure** | Clear messages for timeouts, rate limits, feature disabled; manual path always available |
| **Collect feedback** | Thumbs up/down with optional reason; implicit signals (edits, accepts) |
| **Accessibility** | Streaming text in a polite live region at sensible granularity (sentences, not tokens); focus management; keyboard access (Book VI, Chapter 6) |

### A streaming hook

```ts
// web/src/shared/ai/useStreamingText.ts
import { useCallback, useRef, useState } from 'react';

type Status = 'idle' | 'streaming' | 'done' | 'error' | 'stopped';

export function useStreamingText(start: (signal: AbortSignal, onToken: (t: string) => void) => Promise<void>) {
  const [text, setText] = useState('');
  const [status, setStatus] = useState<Status>('idle');
  const [error, setError] = useState<unknown>(null);
  const controller = useRef<AbortController | null>(null);

  const run = useCallback(async () => {
    controller.current?.abort();
    const c = new AbortController();
    controller.current = c;
    setText(''); setError(null); setStatus('streaming');
    try {
      await start(c.signal, token => setText(prev => prev + token));
      setStatus('done');
    } catch (e) {
      if ((e as Error).name === 'AbortError') setStatus('stopped');
      else { setError(e); setStatus('error'); }
    }
  }, [start]);

  const stop = useCallback(() => controller.current?.abort(), []);
  return { text, status, error, run, stop };
}
```

### The reply draft UI

```tsx
// web/src/features/tickets/components/ReplyDraftPanel.tsx (condensed)
export function ReplyDraftPanel({ ticketId, onUseDraft }: { ticketId: string; onUseDraft: (text: string) => void }) {
  const { text, status, error, run, stop } = useStreamingText((signal, onToken) => streamReplyDraft(ticketId, onToken, signal));

  return (
    <section aria-labelledby="draft-heading" className="ai-panel">
      <h3 id="draft-heading"><SparkleIcon aria-hidden /> AI draft <span className="badge">Review before sending</span></h3>

      {status === 'idle' && <button onClick={run}>Draft a reply</button>}
      {status === 'streaming' && <button onClick={stop}>Stop</button>}
      {status === 'error' && <AiErrorMessage error={error} onRetry={run} />}

      {text && <MarkdownWithCitations text={text} />}            {/* sanitized; links allow-listed (Chapter 9) */}

      {status === 'done' && (
        <div className="actions">
          <button onClick={() => onUseDraft(text)}>Insert into reply</button>
          <button onClick={run}>Regenerate</button>
          <FeedbackButtons feature="ReplyDraft" />
        </div>
      )}
      <p className="sr-only" role="status" aria-live="polite">
        {status === 'streaming' ? 'Generating draft…' : status === 'done' ? 'Draft ready.' : ''}
      </p>
    </section>
  );
}
```

"Insert into reply" copies the draft into the normal reply editor, where the agent edits and sends
through the existing, non-AI path. The AI never sends anything.

### The help assistant

The customer assistant (Chapter 6) uses the same hook with structured events (sources, tokens, done,
flagged): source cards render first, the answer streams with citation links, and "Open a ticket"
pre-fills the new-ticket form (Book VI, Chapter 4) with the conversation. Assistant conversations are
kept in component state (and optionally server-side per Chapter 9's retention rules), not in global
stores (Book VI, Chapter 3).

---

## 6. Running it on Azure

| Concern | Choice (Book IX alignment) |
|---|---|
| Model access | Microsoft Foundry deployments in the EU region (data residency), or a direct provider API with an EU processing option; accessed via managed identity where supported, otherwise API key in Key Vault |
| Networking | Private endpoint to the Foundry resource (Book IX, Chapter 5); outbound via NAT for external providers |
| Configuration | Model IDs and feature flags in App Configuration; per-tenant AI settings in the database |
| Vector search | pgvector in the existing PostgreSQL Flexible Server (extension allow-listed) |
| Workers | `ca-beacon-worker` handles triage, indexing, eval sampling; KEDA scaling on outbox backlog |
| Observability | OpenTelemetry GenAI spans and metrics (model, tokens, latency) to Application Insights; cost dashboards by feature and tenant |
| Releases | AI features behind flags, rolled out by tenant (Book IX, Chapter 9); eval gates in CI (Chapter 8) |

---

## 7. In practice: shipping "Draft reply" end to end

The full checklist for Beacon's first AI feature, mapped to the book:

1. **Define success** with support leads: 50 real threads, expert notes, criteria (grounded, safe,
   helpful, ≤150 words) → eval set (Chapter 8).
2. **Prompt** `ReplyDraft.v1` with grounding, citations and business rules (Chapter 3).
3. **Retrieval** of KB articles via hybrid search (Chapters 5–6), permission-filtered.
4. **Service and endpoint** with authorization, rate limits, policy middleware, SSE (sections 3–4).
5. **Validation**: citation check; safety rule check ("refund," "credit," "guarantee" → flagged).
6. **UI**: panel, streaming, stop, insert-into-editor, labels, feedback, accessibility (section 5).
7. **Telemetry**: feature, prompt version, model, tokens, latency, cost; edit distance between draft and
   sent reply (Chapter 8).
8. **Security review**: threat table row (Chapter 9); red-team cases added to evals.
9. **Rollout**: flag on for the internal team → 3 pilot tenants → general availability; kill switch
   tested.
10. **Iterate**: weekly review of feedback and sampled outputs; prompt and retrieval improvements gated
    by evals.

A single feature touches nearly every book: that's what AI application engineering is.

---

## 8. What can go wrong

- **AI logic scattered** through endpoints and components instead of behind interfaces.
- **No central gateway**: inconsistent timeouts, telemetry, budgets and kill switches.
- **Blocking UIs** waiting for long generations.
- **AI output presented as authoritative**, without labels, sources or editing.
- **Actions executed directly from model output** instead of through domain services.
- **Screen readers announcing every token**, or nothing at all.
- **Features launched to everyone at once** without flags, evals or kill switches.

---

## 9. How an experienced engineer thinks about this

- **AI is a dependency inside a well-layered system.**
- **Policies once, centrally**: routing, budgets, telemetry, flags in the client pipeline.
- **UX makes AI trustworthy**: labels, evidence, control, graceful failure.
- **Every AI feature is full-stack**, and every book's practices apply.
- **Ship gradually, measure continuously.**

---

## 10. Check yourself

**Questions**

1. Where do prompts, model calls and business rules live in Beacon's architecture, and why?
2. What does a `DelegatingChatClient` let you enforce centrally?
3. Compare on-demand streamed, background enrichment and cached derived-content patterns.
4. Why should AI-suggested actions go through normal domain methods?
5. Name five UX principles for AI features.
6. How should streaming text be made accessible?
7. What does a gradual AI feature rollout look like?

**Exercises**

1. Implement `PolicyChatClient` with feature flags and a per-tenant daily token budget, with tests.
2. Build `ReplyDraftPanel` with streaming, stop, insert and feedback, and test it with MSW streaming
   responses.
3. Add an edit-distance metric between AI drafts and sent replies, and chart it per week.
4. Write the rollout plan (flags, pilot tenants, success metrics, rollback triggers) for the help assistant.

**Interview-style questions**

- "How would you architect AI features in an existing .NET application?"
- "What makes a good UX for AI-generated content?"
- "How do you roll out an AI feature safely?"

---

## 11. Going deeper

- [Microsoft docs: AI for .NET developers](https://learn.microsoft.com/dotnet/ai/)
- [Microsoft.Extensions.AI.Evaluation](https://learn.microsoft.com/dotnet/ai/evaluation/libraries)
- Google PAIR, [People + AI Guidebook](https://pair.withgoogle.com/guidebook/) — UX for AI.
- Microsoft, [Guidelines for Human-AI Interaction](https://www.microsoft.com/en-us/research/project/guidelines-for-human-ai-interaction/)

**Next:** [Chapter 11 — Using AI as a Developer](11-using-ai-as-a-developer.md) turns to AI as a tool for
your own work.
