# How LLMs Work (for Engineers)

> **🔄 Current (as of October 2026):** Model names, context window sizes, prices and
> capabilities change every few months. This book uses provider-neutral concepts and keeps
> model identifiers in configuration. Check your provider's documentation for current models.

This book (Book XI) is about **AI application engineering**: building reliable, secure,
cost-effective software features on top of large language models (LLMs). It's not about
training models or machine-learning research. But you can't engineer well with a component
you don't understand. Treating an LLM as a magic box leads to the classic failures: trusting
confident wrong answers, prompts that work in a demo and fail in production, surprise bills,
and security holes.

This chapter builds a working mental model of what an LLM is, what it does when you call it,
and why it behaves the way it does. Every later chapter relies on it.

---

## 1. The problem: a new kind of component

Every component you've used so far is **deterministic**: the same input produces the same
output, and behavior is defined by code you can read. An LLM is different:

- Its behavior emerges from billions of learned parameters, not readable logic.
- It's **probabilistic**: the same input can produce different outputs.
- It's **capable** of tasks no hand-written code handles well: summarizing, drafting,
  classifying messy text, extracting structure from prose, answering questions, writing code.
- It's **fallible** in unfamiliar ways: it can produce fluent, plausible, completely wrong
  output with no error signal.

Engineering with LLMs means harnessing the capability while containing the fallibility.

---

## 2. The mental model: next-token prediction at scale

### Tokens

LLMs don't see characters or words; they see **tokens**: chunks of text from a fixed
vocabulary (tens of thousands to a few hundred thousand entries). Common words are single
tokens; rare words split into several; spaces and punctuation are often part of tokens.

```text
 "Beacon resolves tickets quickly."  →  ["Be", "acon", " resolves", " tickets", " quickly", "."]
```

Rules of thumb for English: ~4 characters or ~0.75 words per token. Other languages, code and
unusual strings often use more tokens per character. Tokens matter because **everything is
measured in tokens**: context window limits, prices, rate limits and latency.

### Prediction

At its core, an LLM is a function that takes a sequence of tokens and outputs a **probability
distribution over the next token**:

```text
 input:  "The ticket was resolved by the support"
 output: " team" 0.41 | " agent" 0.33 | " engineer" 0.09 | " staff" 0.05 | ...
```

**Generation** is a loop: pick a token from the distribution, append it to the input, predict
again, until a stop condition (a special end token, a stop sequence, or a maximum length).

```text
 prompt ──► model ──► next-token distribution ──► sample a token ──┐
   ▲                                                                │
   └──────────────────── append token, repeat ◄─────────────────────┘
```

This explains several properties immediately:

- **Output is produced token by token**, so it can be **streamed** (Chapter 2).
- **Latency grows with output length**: generating 1,000 tokens takes roughly 10× longer
  than 100.
- **The model has no plan beyond what it has written so far** (in the plain loop). Asking it
  to reason step by step before answering often improves results, because the reasoning tokens
  become part of the context the answer is conditioned on.

### The transformer, in one paragraph

Modern LLMs are **transformers**. Each token becomes a vector (an **embedding**); many layers
of **attention** let every token's representation incorporate information from relevant
earlier tokens ("resolved" attends to "ticket"); feed-forward layers transform these
representations; the final layer produces the next-token distribution. The model's
**parameters** (weights), learned during training, determine all of this. You don't need the
math, but one consequence matters: attention over the whole context is what lets a model use
instructions, documents and conversation history you provide.

---

## 3. How models are made (enough to reason about behavior)

| Stage | What happens | Effect on behavior |
|---|---|---|
| **Pre-training** | Predict the next token over a huge corpus of text and code | General knowledge, language ability, reasoning patterns, and the biases and errors of the data |
| **Instruction tuning / post-training** | Train on examples of following instructions and conversations | The model acts as a helpful assistant that follows instructions |
| **Preference and safety training** (RLHF, RLAIF, constitutional methods) | Optimize toward responses humans (or rules) prefer | Helpfulness, harmlessness, refusals, tone |
| **Reasoning training** | Reward correct outcomes on verifiable tasks (math, code) | "Thinking" models that spend tokens reasoning before answering |

Consequences for engineers:

- **Knowledge cutoff**: the model knows the world up to its training data. It doesn't know
  your company's data, last week's news, or Beacon's tickets, unless you put that information
  in the prompt (retrieval, Chapter 6) or give it tools (Chapter 4).
- **Knowledge is lossy**: rare facts are remembered poorly and can be confidently wrong.
- **Behavior is shaped, not programmed**: instructions are followed probabilistically and can
  conflict with training (Chapter 3).

---

## 4. The context window: the model's entire world

The **context window** is the maximum number of tokens the model can consider at once: the
system prompt, conversation history, retrieved documents, tool definitions and results, and
the output being generated.

```text
 ┌──────────────────────────── context window ────────────────────────────┐
 │ system prompt │ tool definitions │ retrieved docs │ conversation │ output │
 └──────────────────────────────────────────────────────────────────────┘
```

Key facts:

- **The model is stateless between calls.** Every request must include everything the model
  should consider. "Memory" in chat products is the application re-sending history (or
  summaries, or retrieved facts) each time.
- **Bigger isn't free**: more input tokens cost more and add latency (prompt caching helps;
  Chapter 2).
- **Attention isn't uniform**: models can miss or underweight information in very long
  contexts, especially in the middle. Relevant, well-organized context beats "put everything in."
- **Everything in the context can influence the output**, including text from untrusted
  sources (documents, emails, web pages). That's the root of **prompt injection** (Chapter 9).

> **🧱 Durable:** An LLM call is a pure function of its input tokens (plus randomness). If it
> isn't in the context, the model doesn't know it; if it is in the context, it may influence
> the output, whether you intended that or not.

---

## 5. Sampling: why outputs vary

The model outputs probabilities; **sampling** picks tokens from them:

| Parameter | Effect |
|---|---|
| **Temperature** | Scales the distribution: low (0–0.3) → more deterministic, picks likely tokens; high (0.8–1.0+) → more varied and creative |
| **Top-p / top-k** | Restrict sampling to the most likely tokens (cumulative probability p, or k tokens) |
| **Max tokens** | Hard limit on output length |
| **Stop sequences** | Strings that end generation |

Even at temperature 0, outputs aren't guaranteed identical across calls (implementation
details, hardware parallelism, model updates). **Design for variability**: validate outputs,
test with multiple runs, and never assume a byte-identical response.

Some reasoning-focused models fix or limit sampling parameters and instead expose a
**reasoning effort** or **thinking budget** setting.

---

## 6. Capabilities and limitations

### What LLMs are good at

- Understanding and producing natural language in many languages and styles.
- Summarizing, rewriting, translating, changing tone.
- Classification and extraction from messy text ("which product and urgency is this email
  about?").
- Following structured instructions and formats.
- Reasoning over information provided in context.
- Writing and explaining code.
- Using tools when given clear definitions (Chapter 4).

### What LLMs are bad at (or unreliable at)

- **Facts not in context**, especially specific, rare or recent ones: they **hallucinate**,
  producing plausible fabrications (Chapter 8).
- **Exact computation** (large arithmetic, counting characters, precise date math): give them
  a tool or do it in code.
- **Guaranteed consistency**: the same question can get different answers.
- **Knowing what they don't know**: confidence in the text isn't calibrated to correctness.
- **Strict security boundaries**: they can't reliably separate instructions from data in their
  context (Chapter 9).

### Models as a portfolio

Providers offer families of models at different sizes: large, slower, more capable and more
expensive; small, fast and cheap. Many tasks (classification, extraction, routing) work well on
small models; complex reasoning or nuanced writing needs larger ones. Production systems often
**route** different tasks to different models (Chapter 9's cost management).

Model access options:

- **Proprietary models via APIs** (Anthropic Claude, OpenAI GPT, Google Gemini), directly or
  through cloud platforms (Microsoft Foundry on Azure, Amazon Bedrock, Google Vertex AI).
- **Open-weight models** (Llama, Mistral, Qwen, and others) you can host yourself or run via
  hosting providers: more control and data locality, more operational work.

---

## 7. What an LLM call looks like

Regardless of provider, a chat-style call has the same shape:

```json
{
  "model": "<model-id-from-config>",
  "system": "You are Beacon's support assistant. Answer using only the provided articles...",
  "messages": [
    { "role": "user", "content": "My VPN disconnects every hour. What should I try?" }
  ],
  "max_tokens": 800,
  "temperature": 0.2
}
```

```json
{
  "content": [{ "type": "text", "text": "Hourly disconnects are usually caused by..." }],
  "stop_reason": "end_turn",
  "usage": { "input_tokens": 1830, "output_tokens": 214 }
}
```

- **System prompt**: instructions and context that frame the whole conversation.
- **Messages**: alternating `user` and `assistant` turns; the application sends the history.
- **Usage**: tokens consumed, the basis for cost (Chapter 9).
- **Stop reason**: finished naturally, hit `max_tokens`, hit a stop sequence, or wants to call a
  tool (Chapter 4). Always check it: a truncated answer looks like a complete one.

Chapter 2 makes these calls from C#.

---

## 8. In practice: where AI fits in Beacon

Not every feature should use an LLM. For each candidate, ask: is the task fuzzy (language,
judgment) or exact (rules, math)? What's the cost of a wrong answer? Can a human review it?

| Candidate feature | LLM fit | Risk if wrong | Design |
|---|---|---|---|
| **Suggest a reply** to a customer using knowledge-base articles | Excellent | Medium | Draft for an agent to review and edit (Chapters 6, 10) |
| **Classify** new tickets (product area, urgency) | Excellent | Low–medium | Automatic, with confidence thresholds and easy correction (Chapter 4) |
| **Summarize** long ticket threads for handover | Excellent | Low | Shown as "AI summary," linked to the source comments |
| **Semantic search** of the knowledge base | Excellent (embeddings) | Low | Chapter 5 |
| **Customer self-service assistant** | Good | Medium–high | Grounded in articles, with citations and handoff to humans (Chapters 6–9) |
| **Compute SLA breach times** | Poor | High | Plain code (Book I already does this) |
| **Decide refunds automatically** | Poor | High | Rules + human approval; AI may summarize the case |

Book XI implements the first five, in this order, with evaluation and safety built in. The
guiding principle for Beacon: **AI drafts, suggests, summarizes and searches; humans and code
make the consequential decisions.**

---

## 9. What can go wrong

- **Treating output as fact**: hallucinations in customer-facing answers.
- **Assuming determinism**: tests and parsers that break on variations.
- **Assuming the model knows your data**, or knows recent events.
- **Stuffing the context** with everything, increasing cost and reducing quality.
- **Ignoring stop reasons**: truncated output treated as complete.
- **Using an LLM for exact tasks** (math, rules) that code does perfectly.
- **Forgetting that context content can act as instructions** (prompt injection).

---

## 10. How an experienced engineer thinks about this

- **An LLM call is a probabilistic function of its context.** Control the context; validate
  the output.
- **Use LLMs for fuzzy tasks; code for exact ones.**
- **Design for wrong answers**: review steps, citations, confidence, fallbacks.
- **Everything is tokens**: cost, latency and limits.
- **Choose models per task**, not one model for everything.

---

## 11. Check yourself

**Questions**

1. What is a token, and why do tokens matter for cost and limits?
2. Describe the generation loop. Why can output be streamed?
3. What does "the model is stateless" mean for chat applications?
4. What do temperature and max tokens control? Is temperature 0 fully deterministic?
5. Name four tasks LLMs are good at and four they're unreliable at.
6. Why should you always check the stop reason?
7. Why does Beacon use AI to draft replies rather than send them automatically?

**Exercises**

1. Use a provider's tokenizer tool to count tokens for an English paragraph, the same text in
   another language, and a block of JSON. Compare.
2. Send the same prompt ten times at temperature 0 and 1; compare the variation.
3. Ask a model about a niche fact you know well and evaluate its answer and confidence.
4. List three AI feature ideas for a product you know, and classify each using section 8's table.

**Interview-style questions**

- "Explain how a large language model generates text."
- "What is a context window, and what are its practical implications?"
- "What are hallucinations, and why do they happen?"

---

## 12. Going deeper

- Andrej Karpathy, "Intro to Large Language Models" (talk) and "Let's build GPT" (video).
- [Anthropic documentation: Models overview](https://docs.claude.com/en/docs/about-claude/models/overview)
  and [Prompt engineering overview](https://docs.claude.com/en/docs/build-with-claude/prompt-engineering/overview)
- Jay Alammar, "The Illustrated Transformer."
- Chip Huyen, *AI Engineering* — a book-length treatment of building applications on
  foundation models.

**Next:** [Chapter 2 — Working with AI APIs](02-working-with-ai-apis.md) makes real calls from
Beacon, with streaming, retries and abstractions.
