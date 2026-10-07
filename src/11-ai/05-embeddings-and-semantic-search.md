# Embeddings and Semantic Search

A customer types "my laptop keeps dropping the work network." The knowledge base has an
article titled "VPN disconnects every hour." Keyword search (Book IV, Chapter 3's full-text
search) may miss it: the words barely overlap. A human support agent would connect them
instantly, because they mean similar things.

**Embeddings** let software do the same: they turn text into vectors of numbers where
**similar meaning means nearby vectors**. That powers semantic search, recommendations,
duplicate detection, clustering and, most importantly for this book, the retrieval step of
RAG (Chapter 6). This chapter explains embeddings, vector search and how to combine them with
the full-text search Beacon already has.

---

## 1. The problem: matching meaning, not words

Keyword search matches tokens (after stemming): it's precise for exact terms ("error 0x80070005",
product names, ticket IDs) and blind to paraphrase, synonyms, other languages and vague
descriptions. Users describe problems in their own words, not the words of the documentation.

---

## 2. The mental model: text as points in space

An **embedding model** maps a piece of text to a fixed-length vector (often 256–3,072 numbers):

```text
 "VPN disconnects every hour"            → [0.021, -0.113, 0.087, ..., 0.004]   (e.g. 1,024 dims)
 "laptop keeps dropping the work network" → [0.019, -0.098, 0.091, ..., 0.010]
 "how to change my invoice address"       → [-0.077, 0.045, -0.012, ..., 0.066]
```

The model is trained so that texts with similar meaning have vectors that point in similar
directions. Similarity is measured with **cosine similarity** (the angle between vectors) or
related measures:

```text
 similarity("VPN disconnects…", "laptop keeps dropping…") = 0.82   (close)
 similarity("VPN disconnects…", "change invoice address")  = 0.11   (far)
```

Imagine a map where "network problems," "VPN," "Wi-Fi drops" cluster in one region and "billing,"
"invoice," "payment" in another; real embeddings have hundreds of dimensions, but the intuition holds.

> **🧱 Durable:** Embeddings compress meaning into geometry. Once text is a point in space,
> "find related things" becomes "find nearby points," a problem databases can index.

### Properties that matter

- **Embeddings are model-specific.** Vectors from different models (or versions) aren't comparable.
  Changing the embedding model means **re-embedding everything**.
- **Dimensions and cost**: more dimensions can capture more nuance but cost more storage and
  compute. Many modern models support shortened vectors with modest quality loss.
- **Input limits**: embedding models accept a maximum number of tokens; long documents must be
  **chunked** (Chapter 6).
- **Domain fit**: general models handle most business text well; highly specialized vocabularies
  (medical, legal, internal jargon) may benefit from domain-specific or fine-tuned models.
- **Multilingual models** place "VPN se desconecta" near "VPN disconnects."
- **Asymmetric tasks**: some models embed queries and documents differently (an "input type" or
  instruction prefix); use them as documented.

Embedding models are much cheaper and faster than chat models; embedding an entire knowledge base
is usually inexpensive.

---

## 3. Vector search

### Exact vs approximate

To find the most similar documents to a query vector, you could compare it with every stored
vector (**exact k-nearest neighbors**). That's fine for thousands of vectors, slow for millions.
**Approximate nearest neighbor (ANN)** indexes trade a little recall for huge speedups:

| Index | Idea | Trade-offs |
|---|---|---|
| **HNSW** (Hierarchical Navigable Small World) | A multi-layer graph; search hops toward closer neighbors | Excellent recall and speed; more memory; slower to build |
| **IVF** (inverted file, e.g. IVFFlat) | Cluster vectors; search only the nearest clusters | Faster to build, less memory; needs training on data; recall depends on how many clusters are probed |
| **DiskANN** and others | Graph indexes optimized for SSDs | Very large datasets |

### Where to store vectors

| Option | Notes |
|---|---|
| **PostgreSQL + pgvector** | Vectors next to your relational data; SQL filtering, joins, transactions, one system to operate (Book IV, Chapter 3) |
| **Azure AI Search** | Managed search with vector, full-text and hybrid ranking, semantic re-ranking, document skills |
| **Dedicated vector databases** (Qdrant, Weaviate, Pinecone, Milvus) | Specialized features and scale; another system to run |
| **Cosmos DB, Azure SQL, Redis, Elasticsearch/OpenSearch** | Many general databases now support vectors |

> **🧭 When not to add a vector database:** If you already run PostgreSQL and your corpus is up
> to millions of chunks, **pgvector** is usually enough, and keeping vectors next to the data they
> describe (with the same permissions, filters and transactions) avoids a whole class of
> synchronization and security problems. Add a dedicated search service when you need features
> or scale PostgreSQL can't provide.

---

## 4. pgvector in practice

```sql
create extension if not exists vector;

create table article_chunks (
    id          bigint generated always as identity primary key,
    article_id  bigint not null references articles (id) on delete cascade,
    chunk_index int not null,
    content     text not null,
    embedding   vector(1024) not null,
    model       text not null,                        -- which embedding model produced it
    updated_at  timestamptz not null default now(),
    unique (article_id, chunk_index)
);

create index article_chunks_embedding_hnsw
    on article_chunks using hnsw (embedding vector_cosine_ops);
```

Query: the nearest chunks to a query embedding, **with relational filters**:

```sql
select c.article_id, a.slug, a.title, c.content,
       1 - (c.embedding <=> @query_embedding) as similarity      -- <=> is cosine distance
from article_chunks c
join articles a on a.id = c.article_id
where a.status = 'published'
order by c.embedding <=> @query_embedding
limit 8;
```

Operators: `<=>` cosine distance, `<->` Euclidean (L2) distance, `<#>` negative inner product.
Use the operator matching the index's operator class and the model's recommendation.

Notes:

- **Filtering + ANN**: filters (`status = 'published'`, tenant, permissions) applied after an
  approximate search can return fewer results than requested; pgvector supports iterative index
  scans to keep searching until enough rows pass the filter. For strongly selective filters (one
  tenant among thousands), partitioning or partial indexes per tenant help.
- **Tune** HNSW build (`m`, `ef_construction`) and search (`hnsw.ef_search`) parameters with recall
  measurements, not guesses.
- **Storage**: 1,024 float32 dimensions ≈ 4 KB per vector; `halfvec` (16-bit) halves it with little
  quality loss.

---

## 5. Hybrid search and re-ranking

Semantic and keyword search fail differently:

| Query | Keyword search | Semantic search |
|---|---|---|
| "laptop keeps dropping the work network" | Weak (few shared words) | ✓ Finds the VPN article |
| "error 0x80070005" | ✓ Exact match | Weak (codes have little "meaning") |
| "T-4821" (a ticket ID) | ✓ | ✗ |
| "printer offline 3rd floor" | ✓ partially | ✓ partially |

**Hybrid search** runs both and merges the results. A simple, robust merging method is **Reciprocal
Rank Fusion (RRF)**: each document scores `Σ 1 / (k + rank)` across result lists (with k ≈ 60), so
documents ranked high in either list rise to the top, without needing comparable scores.

```sql
with semantic as (
    select c.article_id, row_number() over (order by c.embedding <=> @q_vec) as rank
    from article_chunks c join articles a on a.id = c.article_id
    where a.status = 'published'
    order by c.embedding <=> @q_vec
    limit 30
),
keyword as (
    select a.id as article_id,
           row_number() over (order by ts_rank(a.search, websearch_to_tsquery('english', @q_text)) desc) as rank
    from articles a
    where a.status = 'published' and a.search @@ websearch_to_tsquery('english', @q_text)
    limit 30
)
select article_id, sum(1.0 / (60 + rank)) as score
from (select * from semantic union all select * from keyword) r
group by article_id
order by score desc
limit 8;
```

**Re-ranking** adds a second stage: take the top 20–50 candidates and score each against the query
with a more accurate (and more expensive) **cross-encoder re-ranker** or an LLM, then keep the best
few. Retrieval finds candidates fast; re-ranking orders them well. Managed services (Azure AI Search's
semantic ranker, provider re-rank APIs) offer this.

---

## 6. Keeping embeddings fresh

Embeddings are derived data. They must be updated when the source changes:

- **On change**: when an article is published or edited, enqueue a re-embedding job (Book III,
  Chapter 8). The **outbox** (Book IV, Chapter 5) guarantees the job is created whenever the change
  commits.
- **Content hashing**: store a hash of each chunk's text; skip re-embedding unchanged chunks.
- **Model upgrades**: store `model` per row; re-embed in the background into a new column or table,
  switch queries when complete (a blue-green migration for vectors; Book IX, Chapter 9).
- **Deletion**: chunks must be removed with their source (the `on delete cascade` above), including
  for privacy requests (Chapter 9).

---

## 7. Beyond search: other uses of embeddings

- **Duplicate and related ticket detection**: "this looks like T-4810, opened 10 minutes ago by
  someone at the same company" (a likely outage).
- **Clustering**: group a month of tickets by topic to find recurring problems the knowledge base
  doesn't cover.
- **Routing**: nearest-neighbor classification against labeled examples (a cheap alternative to an
  LLM classifier for some tasks).
- **Recommendation**: "articles related to this one."

---

## 8. In practice: semantic and hybrid search in Beacon

### Embedding pipeline

```csharp
// src/Beacon.Infrastructure/Ai/ArticleIndexer.cs (sketch)
public sealed class ArticleIndexer(BeaconDbContext db, IEmbeddingGenerator<string, Embedding<float>> embedder,
                                   IOptions<AiOptions> ai, IChunker chunker)
{
    public async Task IndexAsync(long articleId, CancellationToken ct)
    {
        var article = await db.Articles.AsNoTracking().SingleOrDefaultAsync(a => a.Id == articleId, ct);
        if (article is null || article.Status != "published")
        {
            await db.ArticleChunks.Where(c => c.ArticleId == articleId).ExecuteDeleteAsync(ct);
            return;
        }

        var chunks = chunker.Chunk(article.Title, article.Body);          // Chapter 6
        var vectors = await embedder.GenerateAsync(chunks.Select(c => c.Text), cancellationToken: ct);

        await using var tx = await db.Database.BeginTransactionAsync(ct);
        await db.ArticleChunks.Where(c => c.ArticleId == articleId).ExecuteDeleteAsync(ct);
        db.ArticleChunks.AddRange(chunks.Select((c, i) => new ArticleChunk
        {
            ArticleId = articleId, ChunkIndex = i, Content = c.Text,
            Embedding = new Pgvector.Vector(vectors[i].Vector), Model = ai.Value.Models["Embedding"],
        }));
        await db.SaveChangesAsync(ct);
        await tx.CommitAsync(ct);
    }
}
```

(`Pgvector.EntityFrameworkCore` maps the `vector` type; `IEmbeddingGenerator` comes from
`Microsoft.Extensions.AI`.) The indexer runs in the worker app, triggered by `ArticlePublished` and
`ArticleUpdated` outbox events.

### Search endpoint

`GET /api/kb/search?q=...` embeds the query, runs the hybrid RRF query from section 5, and returns
titles, slugs and highlighted snippets. Results are the same for everyone with access (published
articles), so short-lived caching of query embeddings for popular queries is reasonable (Book III,
Chapter 7).

### Duplicate ticket hints

When a ticket is created, its title and description are embedded (stored in `tickets.embedding`),
and agents see "Possibly related: T-4810 (0.89), T-4799 (0.84)" for open tickets from the same team
in the last 24 hours. A burst of near-duplicates triggers an "possible outage" alert to the team lead.

### Measuring search quality

Book XI, Chapter 8 builds the evaluation; for search, the metrics are:

- **Recall@k**: for each test query, is the expert-chosen article in the top k results?
- **MRR** (mean reciprocal rank): how high is the first correct result?

On an illustrative set of 120 test queries, keyword-only recall@5 was 61%, semantic-only 78%, hybrid 89%. Those
numbers, not intuition, justified the hybrid design.

---

## 9. What can go wrong

- **Mixing vectors from different models** (or versions) in one index.
- **Semantic search alone** for exact identifiers, codes and names.
- **Stale embeddings** after content changes or deletions.
- **Ignoring access control** in vector search: returning chunks from documents the user can't see.
- **Filtered ANN returning too few results.**
- **Tuning by intuition** instead of measuring recall.
- **A new vector database** when pgvector would do, adding synchronization and security problems.
- **Embedding sensitive data** with a third-party provider without considering data agreements
  (Chapter 9).

---

## 10. How an experienced engineer thinks about this

- **Embeddings turn meaning into geometry**; nearest neighbors become "related."
- **Hybrid beats either alone** for real queries; measure to confirm.
- **Vectors are derived data**: keep them fresh, versioned by model, and deletable.
- **Keep vectors near the data and its permissions** when possible.
- **Retrieval quality is measurable**: recall@k and MRR on a test set.

---

## 11. Check yourself

**Questions**

1. What is an embedding, and what does cosine similarity measure?
2. Why can't you compare embeddings from different models?
3. Compare exact and approximate nearest-neighbor search. What do HNSW and IVF do?
4. Why combine semantic and keyword search? How does Reciprocal Rank Fusion work?
5. What is re-ranking, and why is it a second stage?
6. How do you keep embeddings in sync with source content?
7. How do you measure search quality?

**Exercises**

1. Add pgvector to Beacon's database, embed 50 articles, and run semantic queries.
2. Implement hybrid search with RRF and compare recall@5 against keyword-only and semantic-only on
   20 test queries.
3. Implement duplicate-ticket hints and tune the similarity threshold on real examples.
4. Change the embedding model and design a zero-downtime re-embedding migration.

**Interview-style questions**

- "What are embeddings, and how does vector search work?"
- "How would you build search for a knowledge base?"
- "When would you use a dedicated vector database?"

---

## 12. Going deeper

- [pgvector documentation](https://github.com/pgvector/pgvector)
- [Microsoft docs: Embeddings in .NET](https://learn.microsoft.com/dotnet/ai/conceptual/embeddings)
- [Azure AI Search: Hybrid search](https://learn.microsoft.com/azure/search/hybrid-search-overview)
- Pinecone's "Learn" articles on vector indexes (HNSW, IVF) for intuitive explanations.

**Next:** [Chapter 6 — Retrieval-Augmented Generation](06-retrieval-augmented-generation.md)
combines retrieval with generation to answer questions from Beacon's own knowledge.
