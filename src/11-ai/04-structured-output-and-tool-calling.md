# Structured Output and Tool Calling

Free-form text is fine for a summary a human reads. But much of what makes AI useful inside
an application requires the model's output to be **consumed by code**: a classification that
sets a ticket's priority, extracted fields that fill a form, a decision about which action
to take. And many tasks require the model to **get information or act** beyond its context:
look up a ticket, search the knowledge base, check an order's status.

This chapter covers both: getting reliable, typed, validated data out of a model, and giving
the model tools to call. Together they turn an LLM from a text generator into a component
that participates in your application's logic, which also raises the stakes for validation
and security.

---

## 1. The problem: text in, data out

Asking "respond in JSON" in a prompt works most of the time. In production, "most of the time"
means a steady trickle of failures:

- Markdown code fences around the JSON.
- A friendly sentence before the JSON.
- Missing or misspelled fields, wrong types, invented enum values.
- Truncated JSON when `max_tokens` is hit.
- Valid JSON that's semantically wrong ("priority": "Urgent" for a cosmetic request).

The first four are **format** problems, now largely solvable with platform features. The last
is a **correctness** problem that no format feature solves; that's what validation, confidence
handling and evaluation are for.

---

## 2. The mental model: constrain, validate, decide

```text
 schema (from your C# types) ──► model with structured output / tool definitions
                                        │
                                        ▼
                              JSON matching the schema           ← format guaranteed (or retried)
                                        │
                                        ▼
                     deserialize + validate business rules       ← correctness checks in code
                                        │
                   ┌────────────────────┼─────────────────────┐
                   ▼                    ▼                     ▼
              accept & apply     accept as suggestion     reject / retry / human
              (low risk, high     (medium risk: shown      (invalid or low
               confidence)         for confirmation)        confidence)
```

> **🧱 Durable:** The model's output is **untrusted input**, exactly like a request body from
> the internet (Book III, Chapter 5). Constrain its format, validate its content, and decide
> what it's allowed to affect.

---

## 3. Structured output

Providers offer ways to make the model produce JSON that conforms to a **JSON Schema**:

| Mechanism | How | Reliability |
|---|---|---|
| Prompt instructions ("respond with JSON like…") | Text only | Mostly works; needs robust parsing and retries |
| **JSON mode** | API flag: output is valid JSON | Valid syntax, but schema not enforced |
| **Structured outputs / schema-constrained decoding** | API takes a JSON Schema; decoding is constrained so output always matches | Schema-valid by construction (within supported schema features) |
| **Tool calling with an input schema** | Define a "tool" whose input is the data you want; the model "calls" it | Widely supported; strong adherence |

> **🔄 Current (as of October 2026):** Major providers (Anthropic, OpenAI, Google, Azure) support
> schema-constrained structured outputs and strict tool schemas. Supported JSON Schema features
> vary (e.g. limits on recursion, `oneOf`, string formats); check your provider's documentation.

### Generating schemas from C# types

Define the contract once in C# and derive the schema from it, the same "single source of truth"
principle as Book VII, Chapter 2:

```csharp
public sealed record TicketClassification(
    [property: Description("Product area the ticket is about")] ProductArea Area,
    [property: Description("Urgency based on business impact")] TicketPriority Urgency,
    [property: Description("One sentence explaining the decision")] string Reason,
    [property: Description("0.0–1.0: how clear-cut the classification is")] double Confidence);

public enum ProductArea { Network, Accounts, Billing, Hardware, Software, Other }
```

`Microsoft.Extensions.AI` can request typed output directly:

```csharp
ChatResponse<TicketClassification> response =
    await chat.GetResponseAsync<TicketClassification>(messages, options, cancellationToken: ct);

if (response.TryGetResult(out var classification)) { /* typed object */ }
```

It generates a JSON Schema from the type (via `System.Text.Json`'s schema exporter), asks the
provider for structured output where supported, and deserializes the result.

### Validating beyond the schema

The schema guarantees shape, not sense. Validate in code:

```csharp
static Result<TicketClassification> Validate(TicketClassification c) =>
    c switch
    {
        { Confidence: < 0 or > 1 } => Error.Validation("Confidence out of range"),
        { Reason.Length: 0 or > 300 } => Error.Validation("Bad reason"),
        _ => c,
    };
```

and apply **business rules**: a model may suggest `Urgent`, but the application decides whether an
automated suggestion can set `Urgent` without a human (section 7).

### Confidence and abstention

Self-reported confidence from a model is **not calibrated** (a model can say 0.95 and be wrong).
It's still useful as a signal combined with other evidence. Better strategies:

- Give the model an explicit **"uncertain" / `Other`** option so it doesn't force a choice.
- Use **agreement**: run a cheap classifier twice, or two different prompts; disagreement means
  uncertainty.
- **Calibrate thresholds empirically** with your evaluation set (Chapter 8): at what reported
  confidence is the classification actually right 95% of the time?

---

## 4. Tool calling (function calling)

### The mental model

The model **can't execute anything**. Tool calling is a protocol:

```text
 1. You send: messages + tool definitions (name, description, JSON Schema for inputs)
 2. Model responds: "I want to call get_ticket with {"id": "T-42"}"   (stop_reason: tool_use)
 3. YOUR CODE decides whether to run it, runs it, and gets a result
 4. You send: messages + the tool call + the tool result
 5. Model responds: an answer using the result (or another tool call)
 ... loop until the model produces a final answer
```

```json
// a tool definition
{
  "name": "search_knowledge_base",
  "description": "Search published knowledge-base articles. Use for questions about how to fix or configure something.",
  "input_schema": {
    "type": "object",
    "properties": {
      "query": { "type": "string", "description": "Search terms in the customer's words" },
      "max_results": { "type": "integer", "minimum": 1, "maximum": 8 }
    },
    "required": ["query"]
  }
}
```

Key implications:

- **Your code is always in control**: the model proposes, the application disposes. That's where
  authorization, validation and limits go.
- **Tool descriptions are prompts**: the model decides when and how to call a tool from its name,
  description and schema. Write them carefully (what it does, when to use it, what it returns).
- **Tool results go back into the context**, so they cost tokens and can contain untrusted text
  (prompt injection via tool results; Chapter 9).

### Implementing tools in .NET

`Microsoft.Extensions.AI` turns methods into tools (`AIFunctionFactory`) and handles the loop with
`UseFunctionInvocation()` middleware (Chapter 2):

```csharp
public sealed class SupportTools(ITicketQueries tickets, IKnowledgeBaseSearch kb, ICurrentUser user)
{
    [Description("Get the status, priority and latest updates of one of the current customer's tickets.")]
    public async Task<TicketStatusResult?> GetMyTicket(
        [Description("Ticket id, e.g. T-42")] string ticketId, CancellationToken ct)
    {
        if (!TicketId.TryParse(ticketId, null, out var id)) return null;
        return await tickets.GetForReporterAsync(id, user.Id, ct);   // authorization: only the caller's tickets
    }

    [Description("Search published knowledge-base articles. Returns titles, ids and relevant excerpts.")]
    public Task<IReadOnlyList<KbHit>> SearchKnowledgeBase(
        [Description("Search terms")] string query, CancellationToken ct)
        => kb.SearchAsync(query, maxResults: 5, ct);
}

var tools = new SupportTools(...);
var options = new ChatOptions
{
    Tools = [AIFunctionFactory.Create(tools.GetMyTicket), AIFunctionFactory.Create(tools.SearchKnowledgeBase)],
    ToolMode = ChatToolMode.Auto,
};
var response = await chat.GetResponseAsync(messages, options, ct);   // middleware runs the tool loop
```

Note what `GetMyTicket` does: it **ignores any user ID the model might supply** and uses the
authenticated user from `ICurrentUser`. The model chooses *which* ticket; the application decides
*whether this user may see it*.

### Model Context Protocol (MCP)

**MCP** is an open protocol for exposing tools, resources and prompts to AI applications in a
standard way: an **MCP server** offers tools (e.g. "search Beacon tickets"), and any MCP-capable
**client** (AI assistants, IDEs, agent frameworks) can use them. Instead of writing integrations
for each AI app, you write one server.

> **🔄 Current (as of October 2026):** MCP has broad adoption across AI assistants, IDEs and agent
> frameworks, with official SDKs in many languages, including C# (`ModelContextProtocol` package).
> Remote MCP servers use HTTP transports with OAuth-based authorization.

Beacon could expose an MCP server so agents' AI assistants can search tickets and the knowledge
base, with the same authentication and authorization as the API (Book III, Chapter 6). It's an
API surface, and must be secured like one (Chapter 9).

---

## 5. Designing good tools

| Guideline | Why |
|---|---|
| **Few, well-named, purpose-built tools** | Models choose better among a small set of clearly distinct tools |
| **Descriptive descriptions**: what, when to use, what it returns, limits | The description is the only documentation the model sees |
| **Strict input schemas** with enums, ranges, required fields | Less room for invalid calls |
| **Return concise, relevant results** (not entire database rows or 50 KB documents) | Tokens, cost, focus, and less injection surface |
| **Meaningful errors** as results ("ticket not found or not accessible") | The model can recover or explain |
| **Read-only by default**; actions require confirmation (section 7) | Limits damage from mistakes and manipulation |
| **Authorization inside the tool**, from the authenticated context | Never trust identity or permissions passed by the model |
| **Idempotent actions** with keys | Retries and repeated calls don't duplicate effects |
| **Limits**: max calls per conversation, timeouts per tool | Prevents loops and runaway costs |

---

## 6. Parallel calls, loops and failure modes

- Models may request **several tool calls at once** (search two queries in parallel). Execute
  independent read-only calls concurrently, with limits.
- **Loops**: a model can call the same tool repeatedly without progress. Cap iterations (e.g. 5–10)
  and detect repeated identical calls.
- **Hallucinated arguments**: invented IDs, wrong formats. Validate and return clear errors.
- **Wrong tool choice**: improve descriptions; reduce overlapping tools; add examples in the system
  prompt.
- **Over-calling**: the model searches when the answer is already in context; instruct it when tools
  are unnecessary, and measure (Chapter 8).

---

## 7. Deciding what the model's output may affect

A risk-based policy for actions driven by model output:

| Risk of a wrong action | Policy | Beacon example |
|---|---|---|
| Low, easily reversible | Apply automatically, log, allow correction | Set ticket's product area tag |
| Medium | Apply as a **suggestion** a human confirms with one click | Priority change to Urgent; suggested assignee |
| High or irreversible | **Never** automatic; human decides, AI may summarize | Refunds, account deletion, closing tickets for customers |

And always:

- **Audit** every automated action with the model, prompt version and inputs (Book I, Chapter 11's
  audit log).
- **Make corrections easy**, and feed them back into evaluation sets (Chapter 8).

---

## 8. In practice: automatic ticket triage in Beacon

When a ticket is created, a background worker (Book III, Chapter 8) classifies it:

```csharp
// src/Beacon.Infrastructure/Ai/TicketTriage.cs
public sealed class TicketTriage(IChatClient chat, IOptions<AiOptions> ai, IPromptTemplates prompts,
                                 BeaconDbContext db, ILogger<TicketTriage> log)
{
    public async Task TriageAsync(TicketId id, CancellationToken ct)
    {
        var ticket = await db.Tickets.AsNoTracking().SingleAsync(t => t.Id == id, ct);
        var prompt = prompts.Render("TicketClassification", new { title = ticket.Title, description = ticket.Description });

        var response = await chat.GetResponseAsync<TicketClassification>(
            [new(ChatRole.System, prompt.System), new(ChatRole.User, prompt.User)],
            new ChatOptions { ModelId = ai.Value.Models["Classification"], Temperature = 0, MaxOutputTokens = 300 },
            cancellationToken: ct);

        if (!response.TryGetResult(out var c) || Validate(c) is { IsSuccess: false })
        {
            log.LogWarning("Triage produced no valid classification for {TicketId}", id);
            return;                                                    // ticket stays untriaged; agents triage manually
        }

        // Policy: area tag is applied automatically; priority is only ever a suggestion
        await db.Tickets.Where(t => t.Id == id).ExecuteUpdateAsync(s => s
            .SetProperty(t => t.ProductArea, c.Area.ToString())
            .SetProperty(t => t.SuggestedPriority, c.Confidence >= 0.8 ? c.Urgency : (TicketPriority?)null), ct);

        await db.AiDecisions.AddAsync(new AiDecision(id, "TicketClassification.v2", response.ModelId,
                                                     JsonSerializer.Serialize(c), DateTimeOffset.UtcNow), ct);
        await db.SaveChangesAsync(ct);
    }
}
```

The agent UI shows "Suggested priority: Urgent (AI) — Accept / Dismiss." Accepting or dismissing is
recorded; dismissals become new cases in the classification evaluation set (Chapter 8). The 0.8
threshold is chosen from evaluation data (Chapter 8): the confidence level above which suggestions
match expert labels often enough for the team's quality bar.

### A tool-using assistant (preview)

The customer self-service assistant (Chapters 6–9) uses `SupportTools` from section 4:
`GetMyTicket` (read-only, authorized to the caller's tickets) and `SearchKnowledgeBase`. It has no
write tools at all; "open a ticket for me" produces a **pre-filled new-ticket form** the customer
submits themselves, which keeps the human in control of the action.

---

## 9. What can go wrong

- **Parsing free text as JSON** without schema enforcement or validation.
- **Trusting schema-valid output as correct.**
- **Treating self-reported confidence as calibrated.**
- **Tools that trust model-supplied identity or permissions.**
- **Write tools without confirmation** for consequential actions.
- **Too many overlapping tools** with vague descriptions.
- **Unbounded tool loops** and huge tool results.
- **No audit trail** for automated decisions.

---

## 10. How an experienced engineer thinks about this

- **Model output is untrusted input.** Constrain, validate, then decide.
- **Schemas from your types**: one source of truth.
- **The model proposes, the code decides**: authorization and policy live in tools and services.
- **Match automation to risk**: automatic, suggested or human-only.
- **Log and learn**: decisions audited, corrections fed back into evaluation.

---

## 11. Check yourself

**Questions**

1. What's the difference between JSON mode and schema-constrained structured output?
2. Why is schema-valid output not necessarily correct?
3. Describe the tool-calling loop. Who executes the tool?
4. Why must tools take identity from the authenticated context, not from model arguments?
5. What makes a good tool description and schema?
6. What is MCP, and what problem does it solve?
7. How do you decide whether a model's output may trigger an action automatically?

**Exercises**

1. Implement `TicketClassification` with structured output and validation, and run it over 50 sample
   tickets.
2. Build `SupportTools` and a console chat loop; try to make the model fetch another customer's ticket
   and confirm the tool refuses.
3. Measure classification accuracy against expert labels at different confidence thresholds.
4. Build a minimal MCP server exposing `search_knowledge_base` and connect it to an MCP-capable client.

**Interview-style questions**

- "How do you get reliable structured data from an LLM?"
- "Explain function/tool calling. How do you secure it?"
- "When would you let an AI system take actions automatically?"

---

## 12. Going deeper

- [Anthropic: Tool use](https://docs.claude.com/en/docs/agents-and-tools/tool-use/overview)
- [Microsoft docs: Function calling with Microsoft.Extensions.AI](https://learn.microsoft.com/dotnet/ai/conceptual/understanding-tool-calling)
- [Model Context Protocol](https://modelcontextprotocol.io/)

**Next:** [Chapter 5 — Embeddings and Semantic Search](05-embeddings-and-semantic-search.md)
lets Beacon find information by meaning, not just keywords.
