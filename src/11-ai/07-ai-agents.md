# AI Agents

"Agent" is the most hyped word in AI. Strip away the hype, and an **agent** is a system
where a model **decides its own next steps**: which tools to call, in what order, how many
times, and when it's done, in a loop, toward a goal. That autonomy is powerful for open-ended
tasks. It also multiplies cost, latency, unpredictability and risk.

This chapter explains how agents work, the spectrum from fixed workflows to autonomous agents,
the architecture of a reliable agent, and, just as important, when **not** to build one.

---

## 1. The problem: tasks with unknown steps

Some tasks have fixed steps: classify → retrieve → draft → check (Chapter 3's decomposition).
Code orchestrates them; the model fills in each step. Other tasks don't:

- "Investigate why this customer's tickets about printing keep reopening."
- "Find all open tickets related to yesterday's VPN outage, link them to the incident, and
  draft a status update for each customer."
- "Reconcile these two exports and explain the differences."

The number and order of steps depends on what's discovered along the way. Hard-coding every path
is impossible; letting a model plan and adapt is the agent approach.

---

## 2. The mental model: the agent loop

```text
 goal + context + tools
        │
        ▼
 ┌─► model: think about the state, choose an action (tool call) or finish
 │        │
 │        ▼
 │   application: validate, authorize, execute the tool (or ask a human)
 │        │
 │        ▼
 └── observation (tool result) appended to context
        │
     finish ─► final answer / artifact
```

It's Chapter 4's tool-calling loop, run for more iterations with more freedom. Everything else
about agents is about making that loop **effective** (good tools, good context, good planning) and
**safe** (limits, permissions, oversight).

### The autonomy spectrum

| Level | Who decides the steps | Example | Predictability |
|---|---|---|---|
| **Single LLM call** | Code | Summarize a thread | High |
| **Workflow / chain** | Code (fixed sequence, maybe branches) | Classify → retrieve → draft → verify | High |
| **Router** | Model picks one path among a few | "Is this billing or technical?" → different pipelines | Medium-high |
| **Tool-using assistant** | Model picks tools, few iterations | Help assistant searching KB and checking ticket status | Medium |
| **Autonomous agent** | Model plans and executes many steps | "Investigate and fix…" | Low |
| **Multi-agent systems** | Several agents coordinate | Planner + researcher + writer | Lowest |

> **🧱 Durable:** Use the **least autonomy that solves the problem**. Workflows are easier to test,
> cheaper, faster and more predictable. Move up the spectrum only when the task genuinely requires
> the model to decide what to do next.

---

## 3. Workflow patterns before agents

Many "agent" problems are better solved with structured workflows where code controls the flow and
models handle individual steps:

| Pattern | Description | Beacon example |
|---|---|---|
| **Prompt chaining** | Output of step N feeds step N+1, with checks in between | Draft reply → check citations → adjust tone |
| **Routing** | Classify input, dispatch to specialized handling | Billing questions to a billing prompt with billing articles |
| **Parallelization** | Run independent subtasks concurrently, or the same task several times and vote | Summarize 20 related tickets in parallel, then synthesize |
| **Orchestrator-workers** | A model breaks a task into subtasks; workers handle them; results combine | Incident summary: per-customer impact computed by workers |
| **Evaluator-optimizer** | One call generates; another critiques against criteria; repeat | Draft → critique for policy violations → revise |

These patterns are deterministic in structure, so each step can be evaluated (Chapter 8) and failures
localized.

---

## 4. Agent architecture

A production agent has more parts than a loop:

```text
 ┌──────────────────────────────── Agent runtime ────────────────────────────────┐
 │ Goal / task spec      System prompt: role, rules, when to stop, how to report  │
 │ Tools                 Small, well-described, permission-checked (Chapter 4)    │
 │ Context management    What to keep, summarize or drop as the loop grows        │
 │ Memory                Short-term: the conversation/scratchpad                  │
 │                       Long-term: stored facts, past outcomes (retrieved, RAG)  │
 │ Planning              Explicit plan/todo list the agent updates                 │
 │ Guardrails            Step/time/token budgets; allowed actions; approvals      │
 │ Human-in-the-loop     Pause for confirmation before consequential actions       │
 │ State & durability    Persist progress so long tasks survive restarts           │
 │ Observability         Trace every step: prompts, tool calls, results, costs     │
 └───────────────────────────────────────────────────────────────────────────────┘
```

### Tools (again)

Agent quality depends more on tools than on prompts. Good agent tools:

- **Match how the agent thinks**: `find_related_tickets(description, since)` rather than raw SQL access.
- **Return concise, decision-relevant results**, with pagination and summaries for large data.
- **Fail informatively** so the agent can recover.
- **Separate read and write** clearly; write tools have confirmations, idempotency and narrow scope.

### Context management

Long loops accumulate tool results until the context is full, expensive and noisy. Techniques:

- **Summarize or truncate** old observations; keep decisions and key facts.
- **Store large results externally** (a file or database row) and give the agent a handle and a
  summary.
- **Sub-agents** with their own clean context for focused subtasks, returning only conclusions.

### Planning and stopping

- Ask the agent to **write and update a plan** (a checklist) as part of its state; it improves
  coherence on long tasks and makes progress visible.
- Define **completion criteria** explicitly ("you're done when every related ticket is linked and a
  draft exists for each").
- Enforce **budgets** in code: max steps, max tool calls, max tokens, max wall-clock time, max cost.
  When exceeded, stop and report what was done.

### Human in the loop

Consequential actions pause for approval:

```text
 agent proposes: link 14 tickets to INC-77 and send 9 customer updates
 → UI shows the list and drafts → agent lead approves 12 links and 8 drafts, edits one
 → agent executes approved actions only
```

This is Chapter 4's risk policy applied to multi-step work. It also builds trust: people learn what the
agent does well before giving it more autonomy.

---

## 5. Frameworks

You can build agents directly on chat APIs with tool calling (a loop of maybe 100 lines), or use a
framework:

> **🔄 Current (as of October 2026):** In .NET, **Microsoft Agent Framework** (the successor that
> unifies Semantic Kernel and AutoGen) provides agents, tools, multi-agent orchestration, workflows
> with checkpoints and human-in-the-loop, and MCP support, built on `Microsoft.Extensions.AI`. In
> Python, popular options include LangGraph, the OpenAI Agents SDK, the Claude Agent SDK, CrewAI and
> PydanticAI. Hosted options (Microsoft Foundry Agent Service, provider-hosted agent runtimes) manage
> state and execution. The field moves fast; prefer frameworks with durable state, observability and
> clear abstractions over the most features.

Frameworks help with: tool schemas and invocation, conversation and state persistence, streaming,
multi-agent coordination, checkpoints for long-running workflows, and tracing. They also add
abstraction layers that can make debugging harder. For a single assistant with a few tools,
`Microsoft.Extensions.AI` with `UseFunctionInvocation()` (Chapter 4) is often enough.

### Multi-agent systems

Several agents with different roles (planner, researcher, coder, reviewer) passing work between them.
Useful for genuinely separable expertise and for isolating context. Costs: more tokens, coordination
failures, harder debugging and evaluation. Start with one agent; split only when a single agent's
context or instructions become unmanageable.

---

## 6. Evaluating and operating agents

Agents are harder to evaluate than single calls because there are many valid paths. Evaluate:

- **Outcome**: did it achieve the goal? (Often checkable: were the right tickets linked?)
- **Process**: unnecessary steps, wrong tool use, loops, policy violations, cost.
- **Safety**: attempts to use tools outside scope, susceptibility to injected instructions (Chapter 9).

Operate them with **traces of every step** (OpenTelemetry GenAI conventions; Book IX, Chapter 6),
per-run cost and step counts, and alerts on budget overruns or unusual tool use.

> **⚠️ What can go wrong:** Agents compound errors. A 95%-reliable step repeated 10 times succeeds
> end to end only about 60% of the time. Each additional autonomous step needs either higher
> per-step reliability or a checkpoint where errors are caught.

---

## 7. When not to build an agent

> **🧭 When not to use an agent:**
> - The steps are known in advance → a **workflow** in code.
> - The task is a single transformation (summarize, classify, extract) → **one call**.
> - Mistakes are costly and hard to detect → **human-driven process** with AI assistance at each step.
> - Latency matters (interactive UI with sub-second expectations) → avoid multi-step loops.
> - You can't evaluate it → you can't improve it or trust it in production.
>
> A useful test: if you can draw the flowchart, write the flowchart in code and let models handle
> the boxes.

---

## 8. In practice: Beacon's incident assistant

Beacon gets one genuinely agentic feature, for team leads during outages: **"Incident assistant."**
Given an incident description, it finds related open tickets, proposes links, and drafts customer
updates, with every write action approved by a human.

### Tools

| Tool | Kind | Scope |
|---|---|---|
| `search_tickets(query, since, status)` | Read | Lead's teams only; returns IDs, titles, customers, created times, short excerpts |
| `get_ticket(id)` | Read | Same scope; full thread summary rather than raw comments |
| `find_similar_tickets(ticket_id)` | Read | Embedding similarity (Chapter 5) within scope |
| `propose_link(ticket_id, incident_id, reason)` | **Proposal** (no side effect) | Adds to a pending-actions list |
| `propose_customer_update(customer_id, draft)` | **Proposal** | Adds a draft to pending actions |
| `update_plan(items)` | State | The agent's visible checklist |

There are **no direct write tools**. The agent can only **propose**; the application executes approved
proposals through the normal services (with domain rules, authorization, audit and outbox).

### Runtime limits

```csharp
var budget = new AgentBudget(MaxSteps: 25, MaxToolCalls: 40, MaxInputTokens: 400_000, MaxDuration: TimeSpan.FromMinutes(5));
```

Exceeding any limit stops the run with a summary of findings so far.

### Flow

```text
 Lead: "VPN outage in EU since 08:40, gateway eu-2"
 Agent: plan → search tickets since 08:00 mentioning VPN/network → find similar to the top hits
        → inspect borderline ones → propose 17 links with reasons → group by customer
        → draft 9 customer updates from the incident description (no invented ETAs)
 UI: pending actions grouped and editable → lead approves → application executes
```

### Evaluation

A test set of 15 historical incidents with the tickets experts linked. Metrics: precision and recall of
proposed links, step count, cost per run, and policy checks (no ETAs or credits in drafts). The agent
runs nightly against the set (Chapter 8); a prompt or model change that lowers link precision below the
agreed threshold fails the evaluation gate.

### Why this is an agent and the help assistant mostly isn't

The help assistant (Chapter 6) has a fixed pipeline with one generation step: a workflow. The incident
assistant must explore an unknown set of tickets, decide what to inspect, and adapt: that's where agent
autonomy pays for itself, and even there, every consequential action goes through a human.

---

## 9. What can go wrong

- **Building an agent for a workflow problem**, adding cost and unpredictability.
- **Broad or raw tools** (arbitrary SQL, shell access, "send any email").
- **No budgets**: runaway loops and bills.
- **No human checkpoints** for consequential actions.
- **Context overflow** from accumulated tool results.
- **Compounding errors** across many steps.
- **Untraceable behavior**: no record of what the agent did and why.
- **Prompt injection** through tool results driving the agent to misuse tools (Chapter 9).

---

## 10. How an experienced engineer thinks about this

- **Least autonomy that works**: call → workflow → assistant → agent.
- **Tools are the agent's interface to the world**: narrow, safe, informative.
- **Agents propose; systems dispose**, especially for writes.
- **Budgets, checkpoints and traces** are not optional.
- **Evaluate outcomes and processes** before trusting an agent with more.

---

## 11. Check yourself

**Questions**

1. What distinguishes an agent from a workflow?
2. Describe the agent loop. Where do validation and authorization happen?
3. Name five workflow patterns and an example of each.
4. Why use "proposal" tools instead of write tools for consequential actions?
5. How do you manage context in long-running agent loops?
6. Why do errors compound in agents, and how do you mitigate that?
7. Give three situations where you shouldn't build an agent.

**Exercises**

1. Rewrite a multi-step task you'd be tempted to give an agent as a fixed workflow; compare reliability.
2. Build a minimal agent loop with two read tools and budgets (steps, tokens, time) using
   `Microsoft.Extensions.AI`.
3. Implement proposal tools and an approval UI for the incident assistant.
4. Create an evaluation set of five incidents and measure link precision and recall.

**Interview-style questions**

- "What is an AI agent? How is it different from a chatbot?"
- "How would you make an AI agent safe to use in production?"
- "When would you not use an agent?"

---

## 12. Going deeper

- Anthropic, ["Building effective agents"](https://www.anthropic.com/engineering/building-effective-agents)
  — workflows vs agents, patterns and when to use each.
- [Microsoft Agent Framework documentation](https://learn.microsoft.com/agent-framework/)
- [OpenTelemetry semantic conventions for generative AI](https://opentelemetry.io/docs/specs/semconv/gen-ai/)

**Next:** [Chapter 8 — Evaluation and Hallucination Management](08-evaluation-and-hallucination-management.md)
measures whether any of this actually works.
