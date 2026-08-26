# Production-Grade RAG System

> Design a high-scale Retrieval-Augmented Generation platform that serves enterprise knowledge with high retrieval quality, predictable latency, strong freshness guarantees, multi-tenant isolation, cost controls, and continuous offline/online evaluation.

## Problem Statement

Build a production RAG platform for a large enterprise that can answer questions over millions of documents from PDFs, Word files, Confluence, SharePoint, web content, engineering documentation, tickets, and internal knowledge bases.

The system must:

- ingest new and changed documents incrementally
- preserve document structure, versions, metadata, and access controls
- retrieve relevant evidence using hybrid search
- rerank candidates before generation
- generate grounded answers with citations
- abstain when evidence is insufficient
- support multi-tenant isolation and document-level authorization
- handle document deletions and ACL changes safely
- support low-latency online serving at high QPS
- evaluate retrieval and generation independently
- provide observability, quality monitoring, and regression detection
- control LLM and retrieval cost

### Target Scale

Assume:

```text
Tenants:                    500
Registered users:           1,000,000
Documents:                  100M
Average chunks/document:    12
Total chunks:               ~1.2B
Document updates/day:       20M
Peak user queries:          10,000 RPS
Peak ingestion events:      5,000 events/sec
Target availability:        99.95%
Target retrieval p95:       <250 ms
Target first-token p95:     <2.0 sec
Knowledge freshness SLA:   <5 min for priority sources
```

These numbers are design assumptions for interview discussion, not requirements of any particular company.

## Core Design Principle

**Optimize retrieval quality first, then control generation latency and cost with a bounded evidence pipeline.**

The LLM should not be responsible for discovering the right documents from scratch. Retrieval, authorization, ranking, context construction, and evaluation should be explicit production subsystems.

## Real-World Company Use Case

### Enterprise Engineering Knowledge Assistant

A large technology company has engineering documentation spread across OneDrive/SharePoint, Confluence, Git repositories, PDFs, incident tickets, and internal runbooks.

An engineer asks:

> "A PowerScale node is repeatedly rebooting after the latest upgrade. What are the likely causes, what diagnostics should I run, and which internal runbook documents this?"

The system should retrieve the most relevant operational documentation, prioritize current versions, respect the engineer's permissions, answer only from retrieved evidence, and cite the exact sources.

The same platform should handle simpler queries such as:

- "What is the current password rotation policy?"
- "Show the deployment rollback procedure."
- "Compare the 2025 and 2026 storage upgrade runbooks."
- "What changed in the incident response process this quarter?"

## Architecture

See [architecture.md](./architecture.md) for the production architecture, retrieval pipeline, ingestion design, scaling, latency budgets, security, evaluation, failure modes, and trade-offs.

## Main Components

| Component | Responsibility |
|---|---|
| API Gateway | Authentication, tenant routing, rate limiting |
| Query Service | Request validation, query normalization, request IDs |
| Query Router | Search mode, tenant, language, freshness, and model policy routing |
| Query Cache | Low-latency cache for repeated queries where safe |
| Retrieval Service | Coordinates lexical, dense, metadata, and filtered retrieval |
| Vector Index | ANN semantic retrieval |
| Keyword Index | BM25 / exact lexical retrieval |
| Metadata / ACL Index | Fast filtering by tenant, permissions, version, date, source |
| RRF / Fusion Layer | Combines heterogeneous retrieval signals |
| Reranker | Cross-encoder / late-interaction relevance ranking |
| Context Builder | Deduplication, diversity, token budgeting, ordering |
| LLM Gateway | Model routing, timeouts, fallbacks, budgets, streaming |
| Citation Service | Maps claims/evidence to source chunks and documents |
| Ingestion Connectors | Pull/change-event ingestion from source systems |
| Parser / OCR | Extracts text, layout, tables, metadata |
| Chunker | Structure-aware chunking and parent-child relationships |
| Embedding Service | Batch and online embedding generation |
| Indexer | Atomic upserts, deletes, version activation |
| Event Bus | Durable ingestion and reindex events |
| Object Store | Raw source snapshots and normalized documents |
| Evaluation Pipeline | Golden sets, retrieval metrics, generation metrics, regressions |
| Observability | Tracing, quality telemetry, latency, cost, freshness |

## Online Request Flow

```text
User
  |
  v
API Gateway
  |
  v
Query Service
  |
  +--> Auth / Tenant / ACL Context
  |
  v
Query Router
  |
  +--> Cache Lookup
  |
  v
Query Understanding
  |
  +--------------------+--------------------+
  |                    |                    |
  v                    v                    v
BM25 Search      Vector ANN Search     Metadata / ACL Filter
  |                    |                    |
  +--------------------+--------------------+
                       |
                       v
                  Candidate Fusion
                       |
                       v
                    Reranker
                       |
                       v
                 Context Builder
                       |
                       v
                 Evidence Gate
                 /            \
          sufficient          weak
             |                  |
             v                  v
            LLM          Clarify / Retry /
             |             Abstain
             v
      Citation Validation
             |
             v
         Stream Answer
```

## Retrieval Strategy

### Hybrid Retrieval

Use multiple retrieval signals instead of relying on embeddings alone:

```text
                         Query
                           |
              +------------+------------+
              |            |            |
              v            v            v
            BM25        Dense ANN    Metadata / ACL
              |            |            |
              +------------+------------+
                           |
                           v
                    Rank Fusion / RRF
                           |
                           v
                      Top 50-100
                           |
                           v
                       Reranker
                           |
                           v
                        Top 8-15
                           |
                           v
                    Context Builder
                           |
                           v
                        Top 5-8
```

### Why hybrid?

Dense retrieval is strong for semantic similarity, while lexical retrieval is important for identifiers, product names, error codes, version strings, exact terminology, and highly specific phrases.

### Do not maximize K blindly

Increasing candidate count can reduce quality and increase:

- reranker CPU/GPU cost
- network transfer
- prompt tokens
- generation latency
- duplicate evidence
- conflicting versions

The goal is **high recall at the candidate stage and high precision at the context stage**.

## Query Understanding

A lightweight query layer can classify:

```text
Query
 |
 +--> Conversational / standalone
 +--> Keyword-heavy / semantic
 +--> Freshness-sensitive
 +--> Metadata constrained
 +--> Multi-hop
 +--> Unsupported / unsafe
```

The router may select different retrieval policies, for example:

```text
"What is error 0xC001?"
 -> BM25-heavy

"Explain why the node may reboot"
 -> Dense-heavy + reranker

"What changed this week?"
 -> time filter + current-version preference
```

Do not add an LLM query-rewrite call to every request. A deterministic rewrite or small model should only be used where offline evaluation proves it improves recall enough to justify its latency and cost.

## Freshness and Versioning

RAG systems fail badly when old and new documents coexist without explicit version semantics.

Each indexed chunk should carry:

```json
{
  "tenant_id": "t1",
  "document_id": "doc-123",
  "chunk_id": "doc-123#p7",
  "version": 12,
  "is_current": true,
  "effective_from": "2026-08-01T00:00:00Z",
  "effective_to": null,
  "source_updated_at": "2026-08-26T08:20:00Z",
  "acl_version": 4
}
```

Prefer atomic version activation:

```text
New document
   |
parse/chunk/embed/index
   |
validate index health
   |
mark new version ACTIVE
   |
deactivate previous version
```

Reads should never observe a half-indexed document version.

## Ingestion Architecture

```text
SharePoint / Confluence / Git / Files / Jira / Web
                           |
                           v
                    Change Connectors
                           |
                           v
                       Event Bus
                           |
                           v
                  Ingestion Orchestrator
                           |
             +-------------+-------------+
             |                           |
             v                           v
      Raw Object Store          Parser / OCR / Layout
                                         |
                                         v
                                  Structure-Aware Chunker
                                         |
                               +---------+---------+
                               |                   |
                               v                   v
                         Embedding Queue       Metadata/ACL
                               |                   |
                               v                   |
                         Vector Index       Keyword Index
```

### Incremental ingestion

A changed source should cause a targeted re-index, not a full corpus rebuild.

```text
Change Event
   -> document_id
   -> fetch latest source
   -> parse
   -> chunk
   -> embed only changed chunks
   -> update indexes
   -> activate version
```

Use content hashes to avoid recomputing embeddings when a source change only affects irrelevant metadata.

## Chunking Strategy

Avoid one fixed chunk size for every document.

Use structure-aware chunks based on:

- headings and sections
- paragraphs
- lists
- table boundaries
- code blocks
- semantic boundaries
- parent/child document relationships

A useful hierarchy is:

```text
Document
  |
  +--> Section
        |
        +--> Chunk
              |
              +--> sentence/span metadata
```

Retrieve small chunks for precision but retain parent-section metadata for context construction.

## Context Construction

The reranker output should not be copied blindly into the prompt.

Context Builder responsibilities:

1. remove near duplicates
2. enforce tenant/ACL constraints
3. prefer current versions
4. maximize evidence diversity
5. preserve source order when useful
6. fit within a hard token budget
7. attach citation IDs

Example:

```text
Retrieved: 12 chunks
   |
   +--> duplicate removal -> 9
   +--> stale-version filter -> 7
   +--> diversity selection -> 6
   +--> token budget -> 5
   |
   v
LLM context
```

## Grounding and Abstention

Generation should be evidence-constrained.

```text
Top evidence
    |
    v
Evidence Quality Gate
    |
    +--> strong -> answer from evidence
    |
    +--> weak -> retry retrieval / ask clarification
    |
    +--> insufficient -> abstain
```

A production RAG service should distinguish:

```text
"I don't know from the available sources."
```

from a hallucinated answer that merely sounds plausible.

## Citation Architecture

Each generated answer should be traceable to source evidence:

```text
Claim 1 -> chunk A -> document X -> source URL/page
Claim 2 -> chunk C -> document Y -> source URL/page
```

Store evidence IDs with the response trace rather than trying to reconstruct citations after generation.

For high-value applications, run a post-generation citation-support check and reject/regenerate unsupported claims.

## Performance Budget

Treat latency as a budget instead of tuning components independently.

Example target for p95 first token:

```text
API/auth             40 ms
Query processing     30 ms
BM25                 40 ms
Vector ANN           60 ms
Fusion               10 ms
Reranker            120 ms
Context build        30 ms
LLM TTFT           1500 ms
-------------------------
Total              ~1830 ms
```

The exact numbers depend on hardware and model choice. The important interview point is to budget end-to-end latency and identify the dominant contributor.

### Performance techniques

- parallelize BM25 and ANN retrieval
- colocate retrieval services with indexes where practical
- use ANN indexes and tuned search parameters
- cache embeddings and frequent query results
- batch embedding generation during ingestion
- rerank only a bounded candidate set
- use a small model for optional query rewriting
- use streaming generation
- avoid unnecessary LLM calls
- use model routing by query complexity
- compress context rather than blindly increasing context length

## Caching

Use separate caches for different semantics:

```text
L1 request cache      -> repeated safe requests
Embedding cache       -> repeated query embeddings
Search cache          -> stable retrieval results
Rerank cache          -> repeated candidate sets
LLM response cache    -> only for deterministic/safe workloads
```

Cache keys must include relevant tenant, ACL, index-version, model, and prompt-version dimensions.

Never allow a cached response generated under one authorization context to be served to another user.

## Back-of-the-Envelope Capacity

### Query traffic

```text
Peak queries = 10,000 RPS
```

If retrieval performs two searches in parallel:

```text
BM25 traffic  ≈ 10,000 QPS
Vector traffic ≈ 10,000 QPS
```

If reranking evaluates 50 candidates/query:

```text
Reranker candidate rate
= 10,000 * 50
= 500,000 candidates/sec
```

This is why reranking is often a major scaling hotspot and must be bounded, batched, accelerated, or selectively applied.

### Storage

Assume 1.2B chunks and a 1536-dimensional float32 embedding:

```text
1.2B * 1536 * 4 bytes
≈ 7.0 TB raw vector values
```

Real storage is substantially larger after ANN structures, replicas, metadata, indexes, and overhead.

A production design therefore needs sharding, replication, compression/quantization, tiering, and retention policies.

## Scaling Strategy

### Retrieval tier

Shard indexes by a combination of tenant, corpus, or document hash depending on access patterns.

```text
                         Query Router
                              |
              +---------------+---------------+
              |               |               |
              v               v               v
          Shard 1          Shard 2          Shard N
           BM25+ANN         BM25+ANN         BM25+ANN
              |               |               |
              +---------------+---------------+
                              |
                         Merge / Rerank
```

Avoid broadcasting every query to every shard when routing metadata can reduce fan-out.

### Hot tenants

One large tenant can dominate retrieval capacity. Use tenant-aware quotas, weighted routing, and shard isolation so one customer cannot exhaust shared resources.

### Ingestion scaling

Partition ingestion events by tenant/document ID. Scale parser and embedding workers independently because parsing, OCR, and embedding have different CPU/GPU profiles.

## Evaluation Framework

Do not evaluate RAG only by asking an LLM whether the final answer “looks good.”

Separate the pipeline:

```text
                Evaluation Dataset
                       |
          +------------+------------+
          |                         |
          v                         v
    Retrieval Eval             Generation Eval
          |                         |
          v                         v
Recall@K / MRR / NDCG      Correctness / Faithfulness
          |                Citation Support / Abstention
          +------------+------------+
                       |
                       v
                Online Evaluation
                       |
          CTR / resolution / feedback
          latency / cost / escalation
```

### Retrieval metrics

- Recall@K
- Precision@K
- MRR
- NDCG
- hit rate
- version correctness
- ACL correctness
- freshness hit rate

### Generation metrics

- answer correctness
- groundedness / faithfulness
- citation precision / recall
- completeness
- refusal / abstention quality
- contradiction rate

### System metrics

- p50/p95/p99 latency
- TTFT
- tokens/request
- cost/request
- cache hit rate
- retrieval timeout rate
- index freshness lag
- failed/partial ingestion rate

## Golden Dataset

Maintain a versioned evaluation corpus containing:

```text
query
expected_documents
expected_chunks
answer/reference
citations
freshness requirement
ACL context
language
query type
```

Include hard cases:

- ambiguous questions
- exact error codes
- conflicting document versions
- deleted documents
- ACL boundary cases
- multilingual queries
- long documents
- tables
- code/config snippets
- intentionally unanswerable questions
- prompt injection inside documents

Run the suite on every major change to embeddings, chunking, retrievers, rerankers, prompts, or models.

## Online Evaluation and Experimentation

Offline metrics are necessary but not sufficient.

Use controlled experiments for:

- embedding model changes
- chunk-size policies
- reranker models
- retrieval depth
- context budgets
- prompt versions
- LLM versions

Track both quality and operational impact:

```text
Quality ↑
Latency ↓
Cost ↓
Abstention quality ↑
```

Do not promote a retrieval model because NDCG improved by itself if production latency or cost becomes unacceptable.

## Reliability and Failure Handling

| Failure | Handling |
|---|---|
| Vector index unavailable | Fallback to lexical retrieval where safe |
| BM25 unavailable | Dense fallback with reduced confidence |
| Reranker unavailable | Skip reranker or use lightweight ranking fallback |
| Embedding service unavailable | Use cached embeddings / queue ingestion |
| LLM timeout | Retry safe request or route to fallback model |
| LLM unavailable | Return retrieval-only results or safe degraded response |
| Event bus lag | Expose freshness lag and continue serving last known good index |
| Partial indexing | Keep prior document version active until new version is validated |
| ACL update failure | Fail closed for affected documents |
| Cache poisoning risk | Namespace keys by tenant/auth/index version |
| Hot shard | Rebalance, isolate tenant, apply backpressure |

## Security

### Authorization

Authorization must be applied before evidence reaches the LLM.

```text
Identity
  |
  v
Tenant + ACL Context
  |
  v
Filtered Retrieval
  |
  v
Allowed Evidence
  |
  v
LLM
```

### Prompt Injection

Documents are untrusted data. A malicious PDF should not be able to change system behavior simply because it is retrieved.

Use:

- strict system/developer message boundaries
- evidence delimiters
- structured outputs where appropriate
- tool allowlists
- post-generation validation
- policy enforcement outside the LLM

## Cost Optimization

At scale, retrieval and generation must be treated as separate cost centers.

Key levers:

```text
Query Router
   |
   +--> cheap path for simple search
   +--> standard RAG path
   +--> expensive reasoning path only when needed
```

Use smaller models for classification/query rewriting, compress prompts, minimize retrieved chunks, cache repeated work, quantize/self-host retrieval components where appropriate, and route only difficult questions to expensive models.

Track cost by tenant, application, model, and query type.

## Key Trade-offs

### Vector-only vs Hybrid

**Vector-only:** simpler architecture and strong semantic matching; weaker for exact identifiers and terminology.

**Hybrid:** more components and operational cost; generally stronger retrieval robustness.

### Precomputed vs On-demand embeddings

**Precomputed:** lower query latency; higher storage and indexing cost.

**On-demand:** simpler for very small corpora; poor fit for large production corpora.

### Cross-encoder vs lightweight reranker

**Cross-encoder:** stronger relevance, higher compute cost.

**Lightweight/late-interaction:** better throughput, may sacrifice some peak accuracy.

### Large context vs aggressive filtering

Large contexts reduce retrieval engineering pressure but increase cost, latency, and distraction. Prefer a bounded evidence set supported by evaluation.

### Single global index vs tenant/shard-aware indexes

A global index simplifies some operations, while tenant-aware partitioning improves isolation and query efficiency but increases routing complexity.

## Production Interview Focus Areas

A strong Senior GenAI Engineer should be able to reason about:

1. retrieval quality vs latency vs cost
2. hybrid search and rank fusion
3. chunking and document hierarchy
4. freshness and version activation
5. ACL-aware retrieval and tenant isolation
6. ANN index sizing and sharding
7. reranker bottlenecks
8. cache correctness under authorization
9. groundedness and abstention
10. offline evaluation and golden datasets
11. online experiments and regression detection
12. failure-mode design and graceful degradation
13. model routing and token economics
14. observability for quality, latency, cost, and freshness
15. capacity planning at high QPS

## Further Reading

- [Detailed Architecture](./architecture.md)
- [Mock Interview](./mock-interview.md)
