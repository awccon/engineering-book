# Retrieval-Augmented Generation

An LLM knows a lot about the world and nothing about Beacon's knowledge base, a customer's
tickets, or the internal fix the network team documented last week. **Retrieval-Augmented
Generation (RAG)** bridges that gap: retrieve the relevant pieces of your own data, put them
in the context, and ask the model to answer **using them**, with citations.

RAG is the most common pattern in AI application engineering, and the one where engineering
quality matters most. A RAG demo takes an afternoon. A RAG system that answers correctly,
cites honestly, says "I don't know" when appropriate, respects permissions, and stays correct
as documents change takes real work. This chapter covers that work.

---

## 1. The problem: the model doesn't know your data

Options for making a model "know" your content:

| Approach | How | Fits when |
|---|---|---|
| **Put everything in the prompt** | Paste all documents into the context | Small, stable corpora that fit comfortably in the context window |
| **Fine-tuning** | Train the model further on your data | Teaching style, format or specialized behavior; **not** reliable for injecting facts that change |
| **RAG** | Retrieve relevant passages at query time; answer from them | Large or changing corpora; need for citations; per-user permissions |
| **Tools** | Let the model query systems (Chapter 4) | Structured, live data: ticket status, account details |

RAG is the default for "answer questions from our documents," because it keeps knowledge
**fresh** (update the index, not the model), **attributable** (cite sources), and **permissioned**
(retrieve only what the user may see).

---

## 2. The mental model: two pipelines

```text
 INGESTION (offline, on change)
 documents ─► extract text ─► clean ─► chunk ─► enrich metadata ─► embed ─► index (vectors + text + metadata)

 QUERY (online, per question)
 question ─► (rewrite/expand) ─► retrieve (hybrid) ─► filter by permissions ─► re-rank ─► select top passages
          ─► build prompt (instructions + passages with IDs + question) ─► generate ─► verify citations ─► answer
```

> **🧱 Durable:** RAG quality is mostly **retrieval** quality. If the right passage isn't in the
> context, no prompt can produce the right answer; if wrong passages are in the context, the model
> will often use them. Measure and improve retrieval first.

---

## 3. Ingestion: from documents to chunks

### Extracting text

Real documents are messy: PDFs (with columns, headers, footers, tables, scanned pages), Word files,
HTML with navigation and boilerplate, Markdown, slides. Extraction options:

- Format libraries (PDF text extraction, `DocumentFormat.OpenXml`, HTML parsers).
- **Layout-aware document intelligence** services (Azure AI Document Intelligence and similar) that
  preserve headings, tables and reading order, and OCR scanned pages.
- **Multimodal models** that read page images directly, useful for complex layouts, diagrams and
  tables (more expensive).

Clean the result: remove boilerplate, repeated headers and footers, navigation menus; normalize
whitespace; keep meaningful structure (headings, lists, table rows).

### Chunking

Documents are split into **chunks**: units small enough to embed well and fit several into the
prompt, large enough to carry complete meaning.

| Strategy | How | Trade-offs |
|---|---|---|
| Fixed size with overlap | e.g. 400 tokens, 50-token overlap | Simple; can split mid-thought |
| **Structure-aware** | Split on headings, sections, paragraphs; merge small ones | Respects meaning; best for documentation |
| Semantic | Split where embedding similarity between sentences drops | Adaptive; more complex |
| Parent-child ("small to big") | Embed small chunks for precise matching; return the larger parent section to the model | Precise retrieval with enough context |

Practical defaults for knowledge-base articles: split by headings, target ~200–500 tokens, keep
lists and tables intact, and **prefix each chunk with its document title and section path** so it's
understandable on its own:

```text
VPN disconnects every hour › Windows › Fix: renew the certificate
1. Open the Beacon VPN client...
```

This "contextual chunk header" significantly improves both retrieval and generation, because a chunk
saying "Click Renew" is meaningless without knowing what it belongs to.

### Metadata

Store with each chunk: source document ID, title, section, URL, last updated date, language, product
area, **access control information** (tenant, visibility, groups), and the embedding model. Metadata
powers filtering, citations, freshness and permissions.

---

## 4. Query time: retrieval done well

### Query transformation

User questions are often poor search queries: vague, conversational, or dependent on earlier turns
("what about on Mac?"). Techniques:

- **Contextualize follow-ups**: use a small model to rewrite the latest question into a standalone
  query using the conversation history.
- **Query expansion**: generate alternative phrasings or keywords; search with each and merge.
- **Hypothetical answer embedding (HyDE)**: have a model draft a plausible answer and embed that
  (answers resemble documents more than questions do). Useful in some domains; measure.

### Retrieve, filter, re-rank, select

1. **Hybrid retrieval** (Chapter 5): semantic + keyword, top 20–50 candidates.
2. **Permission filtering** in the query itself (tenant, visibility), never after generation.
3. **Re-rank** candidates against the query (cross-encoder or LLM-based).
4. **Select** the top passages within a token budget (e.g. 4–8 passages, ≤ 3,000 tokens), possibly
   expanding to parent sections, and de-duplicate.

### Grounded generation

The prompt (Chapter 3) instructs the model to answer **only** from the passages, cite them by ID, and
say when they don't contain the answer:

```text
<instructions>
Answer the customer's question using only the passages in <sources>.
- After each sentence that uses a source, cite it like [S2].
- If the sources don't answer the question, say: "I couldn't find this in our help articles"
  and suggest contacting support. Don't use outside knowledge.
- Keep the answer under 150 words, with numbered steps for instructions.
</instructions>

<sources>
<source id="S1" title="VPN disconnects every hour" url="/kb/vpn-disconnects" updated="2026-09-12">
VPN disconnects every hour › Windows › Fix: renew the certificate
...
</source>
<source id="S2" ...>...</source>
</sources>

<question>My laptop keeps dropping the work network about every hour on Windows 11</question>
```

### Verifying the answer

After generation, code checks:

- **Every citation refers to a provided source** (no `[S9]` when there were four sources).
- **The answer cites at least one source**, unless it's the "couldn't find" response.
- Optionally, a **groundedness check**: a small model (or NLI-style classifier) verifies each cited
  sentence is supported by its source (Chapter 8).

Failed checks → retry once with feedback, or fall back to "here are the most relevant articles"
(showing the retrieval results without a generated answer).

---

## 5. RAG architectures beyond the basic pipeline

| Pattern | Idea | When |
|---|---|---|
| **Agentic RAG** | The model decides when and what to search, possibly several times, via a search tool (Chapter 4) | Multi-part questions, exploration |
| **Multi-index routing** | Classify the question, search the right index (KB, past tickets, product docs) | Distinct corpora |
| **GraphRAG / knowledge graphs** | Extract entities and relationships; answer questions about connections | "Which customers are affected by issues related to X?" |
| **Long-context models** | Put whole documents in context instead of chunks | Few large documents; cost and latency permitting |
| **Summaries and hierarchical indexes** | Index document summaries to choose documents, then chunks within them | Large heterogeneous corpora |

> **🧭 When not to build complex RAG:** Start with the simplest pipeline (structure-aware chunks,
> hybrid search, a grounded prompt, citation checks) and an evaluation set. Add query rewriting,
> re-ranking, agentic search or graphs only when evaluation shows a specific failure they fix. Many
> "advanced RAG" techniques add latency and cost for little gain on a given dataset.

---

## 6. Permissions, freshness and other production concerns

- **Document-level security**: the user must only receive passages from documents they may read.
  Filter at retrieval time with the same rules as the API (Book III, Chapter 6). A RAG system that
  ignores permissions is a data leak with a friendly interface.
- **Freshness**: re-index on change via the outbox; show "last updated" in citations; prefer newer
  documents when content conflicts.
- **Conflicting sources**: instruct the model to mention conflicts rather than silently picking one,
  and fix the content.
- **Latency budget**: embedding (~50 ms) + retrieval (~20–100 ms) + re-ranking (~100–300 ms) +
  generation (1–5 s, streamed). Stream the answer; show sources as soon as retrieval completes.
- **Caching**: query embeddings; retrieval results for popular questions (respecting permissions).
- **Feedback**: thumbs up/down and "this didn't help" with the question, sources and answer logged
  (with consent and privacy rules; Chapter 9) for evaluation.

---

## 7. In practice: Beacon's help assistant

Beacon's customer portal gets an assistant: customers ask questions in natural language; answers come
from **published** knowledge-base articles **visible to their organization**, with citations, and an
easy path to open a ticket.

### Ingestion

- Source: `articles` (Markdown, from Book X's import tool and the KB editor).
- Chunking: split by headings, ~350 tokens, title and section path prefixed, tables kept whole.
- Metadata: article ID, slug, section anchor, updated date, language, `visibility` (`public` or a
  tenant ID for customer-specific articles).
- Embeddings stored in `article_chunks` (Chapter 5), refreshed by the outbox-driven `ArticleIndexer`.

### Query pipeline

```csharp
// src/Beacon.Infrastructure/Ai/HelpAssistant.cs (condensed)
public async IAsyncEnumerable<AssistantEvent> AnswerAsync(
    AssistantRequest request, [EnumeratorCancellation] CancellationToken ct)
{
    // 1. Standalone query from conversation (small model; skipped for first turns)
    var query = request.History.Count == 0 ? request.Question : await rewriter.RewriteAsync(request, ct);

    // 2. Hybrid retrieval with permission filter (tenant from the authenticated user, never from the client)
    var candidates = await search.HybridAsync(query, visibleTo: currentUser.TenantId, top: 30, ct);

    // 3. Re-rank and select within a token budget
    var sources = (await reranker.RerankAsync(query, candidates, ct))
        .Where(s => s.Score >= options.MinRelevance)
        .Take(6)
        .Select((s, i) => s with { Label = $"S{i + 1}" })
        .ToList();

    yield return new AssistantEvent.Sources(sources.Select(s => new SourceRef(s.Label, s.Title, s.Url, s.UpdatedAt)).ToList());

    if (sources.Count == 0)
    {
        yield return new AssistantEvent.NoAnswer("I couldn't find this in our help articles.");
        yield break;
    }

    // 4. Grounded generation, streamed
    var prompt = prompts.Render("HelpAnswer.v4", new { sources, question = request.Question });
    var text = new StringBuilder();
    await foreach (var token in ai.StreamAsync(AiFeature.Assistant, prompt, ct))
    {
        text.Append(token);
        yield return new AssistantEvent.Token(token);
    }

    // 5. Verify citations after streaming completes
    var check = CitationChecker.Check(text.ToString(), sources);
    yield return check.IsValid
        ? new AssistantEvent.Done(check.CitedLabels)
        : new AssistantEvent.Flagged("Some citations couldn't be verified; please check the linked articles.");
}
```

### UI

- Sources appear first (as cards), then the answer streams in with clickable `[S1]` citations.
- A persistent "Didn't solve it? **Open a ticket**" button pre-fills the ticket form with the question
  and the conversation (Chapter 4's "no write tools" design).
- Feedback buttons log the question, source IDs, answer and prompt version.

### Evaluation results (from Chapter 8's harness)

| Version | Retrieval recall@6 | Answer correct (judged) | Citation valid | "Couldn't find" when appropriate |
|---|---|---|---|---|
| v1: fixed 500-token chunks, semantic only | 71% | 64% | 88% | 52% |
| v2: heading-based chunks + title prefixes | 82% | 74% | 91% | 60% |
| v3: + hybrid search | 90% | 81% | 93% | 71% |
| v4: + re-ranking + relevance threshold + prompt with explicit "couldn't find" rule | 93% | 87% | 98% | 89% |

Each change was justified by the numbers, and the biggest gains came from **chunking and retrieval**,
not prompt wording.

---

## 8. What can go wrong

- **Poor extraction and chunking**: tables shredded, chunks without context, boilerplate everywhere.
- **Retrieval misses**, then the model answers from general knowledge (hallucination with confidence).
- **No "I don't know" path**, so weak retrieval always produces an answer.
- **Ignoring permissions** in retrieval.
- **Stale indexes** after content changes or deletions.
- **Invalid or fabricated citations** not checked.
- **Over-engineering** with advanced techniques before measuring the basics.
- **Prompt injection through documents** (a document containing "ignore your instructions…";
  Chapter 9).

---

## 9. How an experienced engineer thinks about this

- **Retrieval first**: most RAG quality problems are retrieval problems.
- **Chunks must stand alone**: structure-aware splitting with context headers.
- **Ground, cite, verify**: answers tied to sources, checked by code.
- **Permissions and freshness are part of retrieval**, not afterthoughts.
- **Evaluate every change** on a representative question set.

---

## 10. Check yourself

**Questions**

1. Compare RAG, fine-tuning, long context and tools for making a model use your data.
2. Describe the ingestion and query pipelines.
3. What makes a good chunk? What's a contextual chunk header?
4. What does query rewriting solve in conversations?
5. Why must permission filtering happen at retrieval time?
6. How can code verify citations?
7. Why should you start with a simple pipeline and evaluate before adding techniques?

**Exercises**

1. Build Beacon's ingestion with structure-aware chunking and title prefixes; inspect 20 chunks for
   standalone readability.
2. Implement the query pipeline with hybrid search and citation checking; test with questions the KB
   can and can't answer.
3. Add a tenant-specific article and verify another tenant's users never receive it in sources.
4. Run the evaluation from Chapter 8 for two chunking strategies and compare.

**Interview-style questions**

- "How does Retrieval-Augmented Generation work?"
- "How would you improve a RAG system that gives wrong answers?"
- "How do you handle permissions in a RAG system?"

---

## 11. Going deeper

- Anthropic, "Introducing Contextual Retrieval" (engineering blog) — chunk context and hybrid search.
- [Microsoft docs: RAG in Azure AI Search](https://learn.microsoft.com/azure/search/retrieval-augmented-generation-overview)
- Chip Huyen, *AI Engineering*, chapter on RAG and agents.

**Next:** [Chapter 7 — AI Agents](07-ai-agents.md) gives models more autonomy, and examines when
that's worth the risk.
