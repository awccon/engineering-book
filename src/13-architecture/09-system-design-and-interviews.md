# System Design and Interviews

System design is where everything in this book comes together: HTTP and APIs, data modeling and
indexes, caching, queues, consistency, failure, security, cost and AI, combined into a coherent answer to
"how would you build this?". It's a core skill for senior engineering work, and it's also a standard
interview format, often the one that decides the level you're hired at.

This chapter gives you a **repeatable method** for designing systems, works through one complete design
in detail, summarizes the key ideas behind the most common design problems, and then covers how to
prepare for modern technical interviews as a whole: coding, system design and behavioral rounds.

---

## 1. The problem: big, vague questions

"Design a notification system." "Design a URL shortener." "How would you build Beacon's AI assistant?"
These questions are deliberately open-ended. There's no single correct answer, and the first instinct
(start drawing boxes, or name a technology) is usually the wrong one.

Whether in a design review or an interview, the failure modes are the same:

- **Designing before understanding**: building a globally distributed system for a requirement that
  turns out to be 100 users.
- **Technology bingo**: "Kafka, Redis, Kubernetes, Cassandra" without saying why each one is there.
- **Too shallow**: a diagram of boxes with no data model, no API, no numbers and no failure analysis.
- **Too deep too early**: 20 minutes on database indexing before the overall design exists.
- **No trade-offs**: presenting one design as obviously right, without discussing alternatives or costs.

A method fixes all of these.

---

## 2. The mental model: a design is a set of justified trade-offs

A good system design isn't a diagram; it's an **argument**. It says: given these requirements and
constraints, here's a structure, here's why each part exists, here's what it costs, here's how it fails,
and here's what we'd change if the requirements changed.

> **🧱 Durable:** Every component in a design should answer "**what requirement does this serve, and what
> would happen without it?**" If you can't answer, remove it. Simple designs that meet the requirements
> beat impressive designs that exceed them.

The method below is a sequence of steps, each producing something concrete. In a real project they take
days and produce a design document (Chapter 8). In a 45–60 minute interview they take minutes each, but
the structure is the same.

---

## 3. A system design method

### Step 1: Clarify requirements (5–10 minutes in an interview)

**Functional requirements**: what the system does. List the core use cases, and agree on what's **out**
of scope.

**Non-functional requirements**: the qualities that drive architecture:

- **Scale**: users, requests per second, data volume, growth.
- **Latency**: what response time do users need, for which operations?
- **Availability**: what's the cost of downtime? What SLO? (Chapter 5)
- **Consistency**: which data must be strongly consistent, which can be eventual? (Chapter 3)
- **Durability**: can any data be lost? How much?
- **Security and compliance**: multi-tenancy, personal data, audit, data residency.
- **Cost**: is there a budget? Is it optimized for cost or for speed of delivery?

Ask questions. Interviewers often leave requirements vague on purpose to see whether you'll clarify them.
State assumptions explicitly when you can't get an answer: "I'll assume 10 million users, 10% daily
active."

### Step 2: Estimate (3–5 minutes)

Back-of-the-envelope numbers (Chapter 5) for traffic (average and peak requests/s, read/write ratio),
storage (per item × items per day × retention) and bandwidth. The purpose is not precision, it's to find
out **which parts are hard**: is it read-heavy, write-heavy, storage-heavy, latency-sensitive? A system
with 50 writes/s needs a very different design from one with 500,000.

### Step 3: Define the API (3–5 minutes)

The main operations, as endpoints or messages, with key parameters. This pins down the functional scope
and exposes questions ("is this paginated? what's the idempotency story?").

### Step 4: Data model (5 minutes)

The main entities, their key fields and relationships, and the access patterns: what are the queries,
and what indexes or partition keys do they need? Choose the storage based on access patterns, not
fashion. A relational database is the right default for most business data (Book IV).

### Step 5: High-level design (10 minutes)

The main components and how a request flows through them. Start simple (client, API, database) and add
components only as requirements demand: a cache because reads dominate, a queue because work can be
asynchronous and bursty, a CDN because content is static and global, a search index because queries are
full-text.

### Step 6: Deep dives (10–15 minutes)

Pick the two or three hardest parts, the ones your estimates and requirements identified, and design them
properly. This is where depth shows: consistency, concurrency, partitioning, specific algorithms,
failure handling. In interviews, the interviewer often steers this; follow their interest.

### Step 7: Failure, scale and evolution (5 minutes)

- **Bottlenecks**: what breaks first at 10× load?
- **Failure modes**: what happens when each component fails? (Chapter 3's dependency table)
- **Security**: trust boundaries and the main threats (Chapter 6).
- **Operations**: how do you monitor it, deploy it, and know it's healthy?
- **Evolution**: what would you do differently with more time, scale or different requirements?

```text
Clarify ─► Estimate ─► API ─► Data model ─► High-level ─► Deep dives ─► Failure & evolution
  what       how big    what     what's        how it        the hard      how it breaks,
  & why      & where's  it does  stored and    flows         parts         how it grows
             the hard            how queried
             part
```

---

## 4. Worked example: design a webhook delivery platform

The prompt: "Design a system that lets a SaaS product deliver events to customers' HTTP endpoints."
This is a real problem (Stripe, GitHub, Shopify and Beacon all have one), and it exercises much of this
book.

### Step 1: Requirements

After clarifying questions:

- **Functional**: customers register endpoints (URL, event types, secret); the product publishes events;
  the platform delivers each event to every matching endpoint via HTTP POST; failed deliveries are
  retried; customers can see delivery history and manually redeliver; endpoints that keep failing are
  disabled with a notification.
- **Non-functional**:
  - 50,000 customers, 200,000 endpoints.
  - 5,000 events/s average, 50,000/s peak (bulk operations); average fan-out of 1.5 endpoints per event.
  - **At-least-once delivery**, no lost events. Ordering per resource is "best effort" (customers use
    timestamps and IDs).
  - Delivery latency: p95 under 5 s from event to first attempt in normal operation.
  - Retries for up to 3 days with backoff.
  - 30 days of delivery history.
  - Tenant isolation: one customer's slow or failing endpoint mustn't delay others.
  - Security: endpoints are untrusted URLs on the internet.
- **Out of scope**: customer-side SDKs, event schema design, billing.

### Step 2: Estimates

```text
Deliveries:   5,000 events/s × 1.5 = 7,500 deliveries/s average; 75,000/s peak
Attempts:     assume 5% of deliveries fail first time, average 3 retries → ~8,600 attempts/s average
Concurrency:  Little's Law: endpoint latency avg 300 ms, p99 10 s (timeout)
              7,500/s × 0.3 s ≈ 2,250 requests in flight on average; peaks and slow endpoints → plan for 30,000+
History:      7,500/s × 86,400 s ≈ 650 M deliveries/day; × 30 days ≈ 19.5 B attempt records
              at ~500 bytes (metadata, response code, truncated response) ≈ 10 TB
Payloads:     avg 2 KB × 5,000/s × 86,400 × 30 days ≈ 26 TB if stored for redelivery
```

What the numbers reveal: high **outbound concurrency** with slow, unreliable endpoints (the core
challenge); **very large history** storage (needs a different store from the hot path); and a
**fan-out** stage that needs to be fast.

### Step 3: API

```text
Customer-facing (management):
  POST   /v1/endpoints                      { url, eventTypes[], description } → { id, secret (once) }
  GET    /v1/endpoints/{id}
  PATCH  /v1/endpoints/{id}                 { eventTypes, enabled }
  POST   /v1/endpoints/{id}/rotate-secret
  GET    /v1/endpoints/{id}/deliveries?status=&cursor=
  POST   /v1/deliveries/{id}/redeliver

Internal (from the product):
  PublishEvent { eventId, tenantId, type, occurredAt, resourceId, payload }   (via message broker)

Outbound (to customer):
  POST <customer url>
  Headers: Webhook-Id, Webhook-Timestamp, Webhook-Signature (HMAC-SHA256 over id.timestamp.body)
  Body: { id, type, createdAt, data }
  Expect: 2xx within 10 s
```

The outbound headers follow the shape of the open "Standard Webhooks" specification, which is worth
mentioning: it shows awareness of existing conventions.

### Step 4: Data model

```text
endpoints        (tenant_id, endpoint_id, url, event_types[], secret_encrypted, status,
                  failure_streak, disabled_at)             → relational DB; small (200k rows)
events           (event_id, tenant_id, type, payload_ref, created_at)
                                                           → payloads in object storage, 30-day lifecycle
delivery_tasks   (delivery_id, event_id, endpoint_id, attempt, next_attempt_at, status)
                                                           → the work queue (see deep dive)
delivery_attempts(delivery_id, attempt, at, status_code, latency_ms, error, response_snippet)
                                                           → append-only history store, partitioned by day
```

Access patterns: lookup endpoints by `(tenant_id, event_type)` on every event (cache it); find due retries
by time; list deliveries per endpoint, newest first (history store keyed by `endpoint_id, time`).

### Step 5: High-level design

```text
 Product services
      │ PublishEvent (outbox → broker)
      ▼
 ┌───────────┐   fan-out    ┌──────────────────────┐    ┌──────────────────────────┐
 │ Event     │─────────────►│ Delivery queue        │───►│ Dispatchers (stateless,   │──► Customer
 │ ingest +  │ endpoint     │ partitioned by        │    │ autoscaled; per-endpoint  │    endpoints
 │ fan-out   │ lookup       │ endpoint_id           │    │ concurrency limits; SSRF  │    (internet)
 └───────────┘ (cached)     └──────────────────────┘    │ guard; signing)           │
      │                            ▲                     └──────────┬───────────────┘
      │ payload                    │ retries (delay)                │ attempt results
      ▼                            │                                ▼
 Object storage              Retry scheduler ◄──────────── Results pipeline ──► History store
 (payloads, 30 d)            (due-time index)                         │          (partitioned,
                                                                      ▼           30 d TTL)
                                                     Endpoint health (failure streaks, auto-disable,
                                                     customer notifications)

 Management API ── relational DB (endpoints) ── cache (endpoint lookup by tenant+type)
```

The flow: an event arrives, fan-out looks up matching endpoints (from cache) and enqueues one delivery task
per endpoint; dispatchers take tasks, sign and POST, record the result; failures are rescheduled with
backoff; results stream to the history store and update endpoint health.

### Step 6: Deep dives

**Deep dive 1: isolating slow and failing endpoints.** The core problem. If dispatchers process a shared
queue in order, a customer whose endpoint takes 10 s to time out will occupy dispatcher capacity and
delay everyone (head-of-line blocking).

Solutions, layered:

- **Per-endpoint concurrency limits** (say 10 in flight per endpoint) so one endpoint can't take more than
  its share (`beacon-relay`'s semaphores in Book XII).
- **Partition the queue by endpoint** so tasks for a blocked endpoint don't sit in front of others, and
  dispatchers can skip endpoints at their limit.
- **Separate lanes**: endpoints with recent failures or high latency move to a "slow lane" with its own
  dispatcher pool. Healthy endpoints never wait behind unhealthy ones.
- **Short timeouts** (10 s), async I/O (thousands of concurrent requests per dispatcher instance), and
  connection pooling per host.
- **Circuit breaker per endpoint**: after N consecutive failures, stop attempting for a while; schedule
  the backlog for later instead of hammering a dead endpoint.

**Deep dive 2: reliability and retries.**

- Events enter through the product's **outbox** (Chapter 4), so none are lost between the product's
  database and the broker.
- Delivery tasks are acknowledged only after the attempt result is durably recorded: **at-least-once**.
- Each delivery has a stable `Webhook-Id`; customers deduplicate on it (document this clearly).
- Retry schedule with exponential backoff and jitter: 30 s, 2 min, 10 min, 1 h, 4 h, then every 8 h until
  3 days. The **retry scheduler** stores tasks by `next_attempt_at` (a time-indexed table or a broker's
  scheduled messages) and re-enqueues them when due.
- Retry only on network errors, timeouts, 408, 429 (honoring `Retry-After`) and 5xx. A 4xx (except 408 and
  429) is a permanent failure for that attempt and counts toward disabling.
- After the final retry, mark the delivery failed; the customer can redeliver manually from the history.

**Deep dive 3: security.**

- **SSRF**: endpoints are attacker-controlled URLs. Validate at registration and at **every** delivery
  after DNS resolution (to defeat DNS rebinding): HTTPS only, block private, loopback, link-local and
  cloud metadata ranges, don't follow redirects. Run dispatchers in an isolated network with egress only
  to the internet (Chapter 6's threat model).
- **Authenticity**: HMAC signature with a per-endpoint secret over ID, timestamp and body; customers
  verify and reject old timestamps (replay protection). Support secret rotation with two valid secrets
  during a transition.
- **Tenant isolation**: delivery tasks carry tenant and endpoint IDs; payloads are fetched by reference
  with tenant checks; history queries are tenant-scoped.

**Deep dive 4: history at scale.** ~20 billion attempt records per 30 days is too much for the
transactional database. Options: a time-partitioned table in a columnar or wide-column store, or
partitioned PostgreSQL with daily partitions dropped after 30 days (dropping a partition is instant,
unlike deleting rows). Keep only metadata and a truncated response; payloads stay in object storage
with a lifecycle rule.

### Step 7: Failure and evolution

- **Broker outage**: product outboxes accumulate; delivery resumes when it recovers (latency SLO missed,
  nothing lost). Alert on outbox age.
- **Dispatcher crash**: unacknowledged tasks are redelivered; customers may see duplicates (expected,
  documented).
- **Mass customer outage** (a big cloud region fails, thousands of endpoints time out): circuit breakers
  move them to slow lanes; retry backlog grows; capacity for healthy endpoints is protected. Watch for a
  **retry wave** when they recover: rate-limit backlog draining per endpoint.
- **Noisy tenant** (a bulk operation creates 10M events): per-tenant rate limits on ingest, separate
  lanes for bulk traffic.
- **Observability**: per-endpoint success rate and latency, queue depth per lane, retry backlog, age of
  oldest undelivered event, end-to-end latency SLO.
- **Evolution**: event type filtering by customer-defined rules; ordered delivery per resource for
  customers who need it (partition by resource, at the cost of throughput); delivery to queues
  (Service Bus, SQS) instead of HTTP for large customers; a customer-facing replay of "all events since
  X".

That's a complete design: every component justified by a requirement or an estimate, the hard parts
designed in depth, failure modes considered.

---

## 5. Common design problems and their key ideas

Most interview questions are variations of a few dozen classic problems. You don't need to memorize
solutions, but you should know the **central idea** that each one tests:

| Problem | The core challenge | Key ideas |
|---|---|---|
| URL shortener | Generating short unique IDs; read-heavy redirect | Base62 IDs from a counter or random with collision check; cache hot links; 301 vs 302 for analytics |
| Rate limiter | Counting fairly and quickly across instances | Token bucket / sliding window; Redis with atomic scripts; local + global limits; fail-open vs fail-closed |
| News feed / timeline | Fan-out of posts to many followers | Fan-out on write vs on read; hybrid for celebrity accounts; ranking; cursor pagination |
| Chat / messaging | Real-time delivery, ordering, presence | WebSockets; per-conversation sequence numbers; offline delivery; read receipts; fan-out to devices |
| Notification system | Multi-channel delivery, preferences, rate limits | Queues per channel; templates; user preferences; idempotency; provider failover; digesting |
| File storage / sync | Large files, deduplication, sync conflicts | Chunking; content hashes; object storage; metadata DB; resumable uploads; conflict resolution |
| Search autocomplete | Very low-latency prefix lookups | Tries or prefix indexes; precomputed top-k per prefix; caching at edge; offline aggregation |
| Distributed cache | Partitioning and consistency | Consistent hashing; replication; eviction policies; cache stampede protection |
| Payment system | Correctness and no double charges | Idempotency keys; ledger (double-entry); state machines; reconciliation; outbox; strong consistency |
| Ride sharing / location | Real-time geo queries | Geohash or spatial indexes; frequent location updates; matching; regional partitioning |
| Metrics / logging pipeline | Very high write volume | Logs/streams (Kafka); batching; time-series storage; downsampling; retention tiers |
| Job scheduler | Run jobs reliably at the right time, once | Time-indexed queue; leases with fencing tokens; idempotent jobs; at-least-once |
| LLM chat / RAG assistant | Grounded answers, latency, cost, safety | Retrieval with permission filtering; streaming; caching; evaluation; prompt-injection defenses; budgets (Book XI) |
| AI agent platform | Safe tool use, long-running tasks | Tool permissions; human approval; durable workflows; step budgets; audit; sandboxing (Book XI, Ch 7) |

> **🔄 Current (as of October 2026):** System design interviews increasingly include **AI-system**
> questions: "design a customer support assistant", "design a RAG system over company documents", "design
> an evaluation pipeline for an LLM feature". Interviewers look for the same fundamentals (data flow,
> latency, cost, failure modes, security) plus AI-specific concerns: retrieval quality, evaluation,
> hallucination handling, prompt injection, model routing and token budgets. Book XI covers all of these.

---

## 6. Preparing for modern technical interviews

Interviews are a skill separate from engineering, and like any skill, they improve with deliberate
practice. Most processes for mid-level and senior roles include some combination of the following.

### Coding interviews

**What's tested**: problem solving, data structures and algorithms, code clarity, testing and
communication, usually in 45–60 minutes.

**How to prepare:**

- **Know the core data structures and their costs**: arrays, hash maps and sets, stacks, queues, linked
  lists, trees (binary search trees, heaps, tries), graphs; and the patterns: two pointers, sliding
  window, binary search, BFS/DFS, recursion and backtracking, dynamic programming, sorting, intervals.
  Book I, Chapter 5 covered .NET's collections; the same costs apply in any language.
- **Practice by pattern, not by volume**: 100–150 well-chosen problems with real understanding beat 500
  memorized ones. After solving, compare with other solutions and note the pattern.
- **Use one language fluently.** C# is fine; know its collection APIs (`Dictionary`, `HashSet`,
  `PriorityQueue<TElement, TPriority>`, `Queue`, `Stack`, `SortedSet`, LINQ) without looking them up.
- **Practice talking while solving**: clarify the problem, discuss an approach and its complexity before
  coding, code cleanly, test with examples including edge cases, then optimize.

**The process in the room:**

1. Restate the problem and ask clarifying questions (input sizes, edge cases, constraints).
2. Work through an example by hand.
3. Propose a brute-force approach and its complexity; then improve it.
4. Agree on the approach, then code it, narrating key decisions.
5. Test with the example and edge cases (empty input, one element, duplicates, very large values).
6. Discuss complexity and possible improvements.

### Practical and take-home exercises

Many companies use more realistic formats: build a small API, debug an existing codebase, extend a
feature, review a pull request. For these, the skills from this book apply directly: clean structure,
tests, error handling, a clear README explaining decisions and trade-offs, and **scope discipline**:
deliver a complete, working core rather than an ambitious half-finished system.

> **🔄 Current (as of October 2026):** AI coding assistants have changed technical interviewing. Some
> companies now allow, or even expect, AI tool use in some rounds and assess how well candidates direct,
> verify and correct AI output; others prohibit it and have moved toward live, conversational formats and
> on-site interviews partly to make unassisted work verifiable. Always ask what's allowed, and never use
> AI tools where they're prohibited. Either way, interviewers increasingly probe **understanding**: why
> the code works, what its edge cases are, how you'd change it. That's hard to fake and easy to show if
> you actually know the material.

### System design interviews

Use the method from section 3. Additional advice:

- **Drive the conversation.** State the steps you'll follow, and manage time. The interviewer will
  redirect you if they want something else.
- **Think out loud.** The interviewer evaluates your reasoning, not just the final diagram.
- **Use numbers.** Estimates that drive decisions are a strong signal.
- **Discuss trade-offs explicitly**: "I'd use a relational database here because we need transactions
  for X; the cost is harder horizontal scaling, which at our estimated 2,000 writes/s isn't a problem."
- **Go deep where it matters** and say where you're simplifying.
- **Practice aloud**, with a timer, ideally with a partner who asks follow-up questions.

What interviewers look for, by level (roughly):

| Level | Expectation |
|---|---|
| Mid-level | A working design covering the main components; reasonable choices; can discuss trade-offs when asked |
| Senior | Drives the discussion; clarifies requirements; estimates; identifies the hard parts unprompted and designs them in depth; considers failure, security and operations |
| Staff+ | All of the above plus: multiple viable architectures compared; organizational, migration and cost considerations; evolution over years; knows where the real risks are and what to leave simple |

### Behavioral interviews

These assess how you work with others, handle difficulty and make decisions. They matter more as seniority
increases, and they're often where candidates are least prepared.

- **Prepare 8–10 stories** from your experience covering: a difficult technical problem, a project you
  led, a conflict or disagreement, a failure or mistake, a time you influenced without authority, a
  decision with incomplete information, mentoring someone, and a time you improved a process or system.
- **Structure them with STAR**: Situation (brief context), Task (your responsibility), Action (what *you*
  did, specifically; most of the answer), Result (outcome, with numbers if possible, and what you
  learned).
- **Be honest about failures.** "What went wrong, what I learned, what I do differently now" is a strong
  answer. Claiming you've never failed is a weak one.
- **Say "I", not "we"**, when describing your actions. The interviewer is assessing you.

### Questions to ask them

Interviews go both ways. Good questions also signal maturity:

- "How do you decide what to build, and who's involved?"
- "What does your deployment process look like? How often do you deploy?"
- "Tell me about a recent incident and how it was handled."
- "How do you handle technical debt?"
- "How is AI tooling used in your engineering process?"
- "What would success look like for this role in the first six months?"

### A preparation plan

For an engineer who has worked through this book, about 8–12 weeks of part-time preparation is typical:

| Weeks | Focus |
|---|---|
| 1–6 | Coding: 4–5 problems a week by pattern; one timed mock interview every week or two |
| 3–8 | System design: one design a week using the method; read 2–3 engineering blog posts on real architectures; mock with a partner |
| 6–9 | Behavioral: write and rehearse stories; mock interview |
| Ongoing | Revisit weak areas; research each company (product, stack, engineering blog) |

---

## 7. In practice: a 45-minute system design, condensed

Here's how the first part of an interview for "Design Beacon's AI help assistant" might go, condensed,
to show the method in conversation:

> **Interviewer:** Design an AI assistant that answers customers' questions using the company's
> knowledge base.
>
> **Candidate:** Let me start with requirements. Who are the users: customers, agents, or both? How big
> is the knowledge base, and how often does it change? Are there per-customer documents that other
> customers mustn't see? What languages? And what happens when the assistant can't answer: hand off to a
> human?
>
> **Interviewer:** Customers. About 20,000 articles, updated daily. Some articles are restricted to
> certain plans. English and German. Hand off to a ticket if unsure.
>
> **Candidate:** Then the key non-functional requirements are answer quality and grounding (no invented
> answers), permission-aware retrieval, time to first token under about 2 seconds, cost per conversation,
> and safety against prompt injection from article content or user input. I'll estimate: if 200,000
> customers ask a question on 2% of days, that's 4,000 conversations a day, about 3 turns each, so about
> 12,000 model calls a day, a fraction of a call per second on average and maybe 5 per second at peak.
> That's small, so throughput isn't the hard part. Quality, permissions and cost are.
>
> **Candidate:** I'll go through the API, the data model with chunks and embeddings, the retrieval and
> generation flow, then deep dive into retrieval quality, permissions and evaluation...

Notice what happened in the first three minutes: scope clarified, assumptions stated, numbers estimated,
and the **hard parts identified** (not throughput, but quality, permissions and cost), which tells the
interviewer where the deep dives will go. The rest of the answer would follow Book XI's RAG design
(Chapter 6) with the method from section 3.

---

## 8. What can go wrong

- **Jumping to solutions** without clarifying requirements.
- **Technology names without reasons**.
- **No numbers**, so the design isn't connected to the problem's scale.
- **Over-engineering**: global distribution and microservices for a small problem.
- **Ignoring failure, security and operations** until asked.
- **Monologuing** without checking in with the interviewer, or **waiting** for the interviewer to lead.
- **Memorized designs** recited without adapting to the actual requirements.
- **Neglecting behavioral preparation** because it seems "soft".
- **Cramming** algorithm problems without understanding the patterns.

---

## 9. When not to use it

> **🧭 When not to use it:** The full seven-step method is for significant new systems and for
> interviews. For a feature inside an existing system, a lighter version (requirements, data model
> changes, API, failure modes) is enough, and for small changes, a PR description is the design document.
> And interview-style designs optimize for showing breadth in 45 minutes; real designs need more
> investigation, prototypes and input from the people who'll operate and use the system.

---

## 10. How an experienced engineer thinks about this

- **Requirements and numbers come first**, because they determine which parts are hard.
- **Starts simple and adds complexity only when a requirement demands it.**
- **Justifies every component** and can say what happens without it.
- **Goes deep on the hard parts** rather than evenly shallow everywhere.
- **Considers failure, security, operations and cost** as part of the design, not afterthoughts.
- **Presents trade-offs**, not just choices, and can describe alternatives.
- **Treats interviews as a skill to practice**, separate from but built on real engineering experience.
- **Uses real experience as the best preparation**: the designs in this book (Beacon's outbox, relay,
  RAG pipeline, tenant isolation) are exactly the stories and depth interviewers look for.

---

## 11. Check yourself

**Questions**

1. What are the seven steps of the system design method? What does each produce?
2. Why do estimates matter in system design? What did the webhook platform's estimates reveal?
3. How does the webhook design prevent one slow endpoint from delaying others?
4. Why is at-least-once delivery the right guarantee for webhooks, and what must customers do as a result?
5. How do you defend a webhook platform against SSRF? Why validate at delivery time as well as
   registration?
6. Name the core challenge and one key idea for five of the classic design problems in section 5.
7. What changes in what interviewers expect between mid-level, senior and staff levels?
8. What's the STAR structure, and why should you say "I" rather than "we"?

**Exercises**

1. Do the full method, timed at 45 minutes, for: a notification system for Beacon (email, in-app, push,
   SMS) with user preferences and digests. Then review your design against Chapters 3–6.
2. Design a rate limiter for Beacon's public API: per-tenant and per-endpoint limits, across many API
   replicas. Compare token bucket and sliding window, and decide what happens when Redis is unavailable.
3. Pick three problems from section 5 and do each with a partner acting as interviewer, who should ask
   at least two follow-up questions each.
4. Write eight STAR stories from your own experience, then rehearse each aloud in under three minutes.
5. Solve 20 coding problems across five patterns in C#, then explain each solution's complexity aloud.

**Interview-style questions**

- "Design a webhook delivery system."
- "Design a URL shortener. How would it change at 100× the traffic?"
- "Design a system that lets customers ask questions about their own documents using an LLM."
- "Design a job scheduler that runs millions of jobs a day reliably."
- "Tell me about the most complex system you've designed. What would you change now?"

---

## 12. Going deeper

- Alex Xu, *System Design Interview* (volumes 1 and 2) — worked designs for the classic problems.
- Martin Kleppmann, *Designing Data-Intensive Applications* — the depth behind the deep dives.
- [Standard Webhooks specification](https://www.standardwebhooks.com) — conventions for signing and
  delivering webhooks.
- Engineering blogs from companies that publish detailed architecture posts (Stripe, Cloudflare,
  Discord, Shopify, GitHub, Netflix, Uber) — real designs with real trade-offs.
- Gayle Laakmann McDowell, *Cracking the Coding Interview*, and its successor *Beyond Cracking the Coding
  Interview* (with Mike Mroczka, Aline Lerner and Nil Mamano) — coding interview preparation.
- [The Amazon Builders' Library](https://aws.amazon.com/builders-library/) — production design patterns
  explained by the engineers who built them.

---

## Book XIII wrap-up

This book moved from code to systems and from knowing to judging. You can apply design principles with
honesty about their costs; choose application architectures and draw module boundaries; reason about
partial failure, idempotency and consistency; connect systems with messages and sagas; plan capacity,
limit blast radius and build observability in; threat model and secure the whole system, including its
supply chain; keep software maintainable and modernize legacy systems incrementally; and make and
communicate decisions the way experienced engineers do.

Beacon became a modular monolith with enforced boundaries, an event backbone, idempotent APIs and
consumers, tenant isolation in depth, a hardened supply chain and a documented set of architecture
decisions.

**Next:** [Book XIV — Real-World Projects](../14-projects/README.md) puts the whole of Beacon together,
reviews it for production readiness, and gives you further projects to build on your own.
