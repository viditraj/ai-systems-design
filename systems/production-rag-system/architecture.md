# Detailed Architecture — Production-Grade RAG System

## 1. Architecture Goal

Design a RAG platform where retrieval quality, security, freshness, latency, scalability, and evaluation are first-class system properties rather than incidental features of an LLM application.

The architecture separates five concerns:

```text
Ingestion      -> build trustworthy searchable knowledge
Retrieval      -> find relevant authorized evidence
Ranking        -> maximize precision in bounded context
Generation     -> produce grounded user-facing output
Evaluation     -> continuously measure every layer
```

The LLM is one component in the pipeline, not the entire RAG system.

## 2. Requirements and Assumptions

### Functional

- Ingest structured and unstructured enterprise sources.
- Support PDFs, Office documents, HTML, Markdown, source code, tickets, and knowledge bases.
- Handle incremental updates and deletions.
- Preserve versions and effective dates.
- Enforce tenant and document permissions.
- Support keyword, semantic, metadata, and hybrid retrieval.
- Rerank candidates before generation.
- Build bounded, diverse context.
- Produce citations.
- Abstain when evidence is insufficient.
- Support multilingual and exact-term queries.
- Expose search and generation quality telemetry.

### Non-functional

Use the target scale from the system README:

```text
1M users
500 tenants
100M documents
~1.2B chunks
10K peak QPS
20M document updates/day
5K ingestion events/sec
99.95% availability
Retrieval p95 <250 ms
First-token p95 <2 sec
Priority-source freshness <5 min
```

The design should scale horizontally and degrade gracefully if a subsystem becomes unavailable.

## 3. High-Level Architecture

```text
                              ┌──────────────────────┐
                              │         USER         │
                              └──────────┬───────────┘
                                         │
                                         ▼
                              ┌──────────────────────┐
                              │     API GATEWAY      │
                              │ Auth / WAF / Rate    │
                              │ Limit / Tenant Route │
                              └──────────┬───────────┘
                                         │
                                         ▼
                              ┌──────────────────────┐
                              │     QUERY SERVICE    │
                              │ Validation / Trace   │
                              └──────────┬───────────┘
                                         │
                           ┌─────────────┴─────────────┐
                           │                           │
                           ▼                           ▼
                    ┌───────────────┐          ┌───────────────┐
                    │ Query Cache   │          │ Query Router  │
                    └───────┬───────┘          └───────┬───────┘
                            │                          │
                            └────────────┬─────────────┘
                                         ▼
                           ┌─────────────────────────┐
                           │ Authorization Context   │
                           │ Tenant / User / ACL     │
                           └────────────┬────────────┘
                                        │
                  ┌─────────────────────┼─────────────────────┐
                  │                     │                     │
                  ▼                     ▼                     ▼
             ┌──────────┐         ┌───────────┐        ┌──────────────┐
             │   BM25   │         │ Vector ANN│        │ Metadata/ACL │
             │  Search  │         │  Search   │        │   Filtering  │
             └────┬─────┘         └─────┬─────┘        └──────┬───────┘
                  │                     │                     │
                  └─────────────────────┼─────────────────────┘
                                        ▼
                              ┌──────────────────────┐
                              │ Fusion / RRF / Merge │
                              └──────────┬───────────┘
                                         │
                                         ▼
                              ┌──────────────────────┐
                              │       RERANKER       │
                              └──────────┬───────────┘
                                         │
                                         ▼
                              ┌──────────────────────┐
                              │    CONTEXT BUILDER   │
                              │ dedupe / diversity / │
                              │ token budget / order │
                              └──────────┬───────────┘
                                         │
                                         ▼
                              ┌──────────────────────┐
                              │    EVIDENCE GATE     │
                              └──────────┬───────────┘
                                   ┌─────┴─────┐
                                   │           │
                                strong       weak
                                   │           │
                                   ▼           ▼
                                 LLM       Retry / Clarify /
                                   │          Abstain
                                   ▼
                         ┌────────────────────────┐
                         │ Citation Validation    │
                         └───────────┬────────────┘
                                     │
                                     ▼
                               Stream Response
```

## 4. Ingestion Architecture

The ingestion plane is asynchronous and independently scalable from online serving.

```text
Sources
 ├── SharePoint / Confluence
 ├── Git / Wiki
 ├── PDFs / Office
 ├── Jira / Tickets
 └── Web / Internal APIs
          │
          ▼
   Connector Fleet
          │
          ▼
    Event Bus / Queue
          │
          ▼
   Ingestion Orchestrator
          │
     ┌────┴─────┐
     │          │
     ▼          ▼
 Raw Store   Parser/OCR
                │
                ▼
       Structure Normalizer
                │
                ▼
        Structure-Aware Chunker
                │
       ┌────────┼────────┐
       │        │        │
       ▼        ▼        ▼
   Embedding  Metadata   Hash/Dedup
     Queue      ACL
       │        │
       ▼        ▼
 Vector Index  Keyword Index
```

### Incremental update protocol

```text
Source Change
     ↓
Document ID + source version
     ↓
Fetch current document
     ↓
Hash content
     ↓
No content change? -> metadata/ACL-only path
     ↓
Parse / normalize / chunk
     ↓
Embed changed chunks only
     ↓
Write new index version
     ↓
Validate document completeness
     ↓
Atomically activate new version
     ↓
Retire old version
```

Use content hashes at document and chunk level. This avoids paying for re-embedding when only ACLs, timestamps, or unrelated metadata change.

## 5. Document Representation

Use a canonical representation before chunking.

```json
{
  "tenant_id": "tenant-a",
  "document_id": "doc-123",
  "source": "confluence",
  "title": "Storage Recovery Runbook",
  "version": 42,
  "source_url": "...",
  "updated_at": "2026-08-26T08:20:00Z",
  "effective_from": "2026-08-20T00:00:00Z",
  "content_hash": "sha256:...",
  "acl_version": 7,
  "classification": "internal",
  "sections": []
}
```

The canonical model lets downstream chunking, embedding, indexing, evaluation, and citation services operate independently of source-specific APIs.

## 6. Chunking and Hierarchy

### Why naive fixed-size chunking fails

A 500-token splitter can:

- break a procedure between steps
- split tables from headers
- separate code from its explanation
- mix adjacent sections with different topics
- lose parent document semantics

### Hierarchical approach

```text
Document
  ↓
Section
  ↓
Subsection
  ↓
Semantic Chunk
  ↓
Sentences / spans
```

Store parent IDs so retrieval can return a precise child chunk while context construction can recover the section heading and adjacent context.

For source code and technical documentation, use syntax-aware boundaries instead of ordinary prose splitting.

## 7. Indexing Strategy

Maintain separate logical indexes even if the physical platform combines them.

```text
             Canonical Chunk
                    │
        ┌───────────┼────────────┐
        ▼           ▼            ▼
     Vector       BM25        Metadata
      index       index         index
        │           │            │
        └───────────┼────────────┘
                    ▼
              Retrieval API
```

### Vector index

Stores embedding plus compact metadata needed for coarse filtering.

### Keyword index

Stores normalized text, title, headings, identifiers, aliases, source, and version metadata for lexical relevance.

### Metadata/ACL index

Optimized for tenant filtering, permissions, version status, dates, language, document type, and source-specific constraints.

## 8. Authorization-Aware Retrieval

The retrieval request should be associated with an authorization context:

```text
user_id
tenant_id
groups / roles
resource permissions
classification clearance
```

Apply constraints before results are eligible for reranking/generation.

```text
User
 ↓
Auth Service
 ↓
Allowed filters
 ↓
BM25 / ANN candidate retrieval
 ↓
Only authorized candidates
 ↓
Reranker
 ↓
LLM
```

### Why not filter after retrieval?

Because unauthorized text may already influence ranking, summaries, or model behavior. It also increases the blast radius of cache or tracing mistakes.

### ACL changes

Treat ACL version as part of cache/index correctness:

```text
ACL change
   ↓
new acl_version
   ↓
update permission index
   ↓
invalidate affected cache namespaces
```

For highly dynamic permissions, prefer compact capability/group IDs and fast authorization filtering rather than copying huge user lists into every chunk.

## 9. Query Processing

Not every query should take the same path.

```text
                    Query
                      │
                      ▼
                 Query Router
                      │
          ┌───────────┼────────────┐
          │           │            │
          ▼           ▼            ▼
       lexical     semantic      metadata
       heavy       heavy         constrained
          │           │            │
          └───────────┼────────────┘
                      ▼
                 Retrieval Plan
```

Useful routing signals include:

- query length
- presence of IDs / error codes
- recency language
- detected language
- tenant policy
- corpus type
- historical retrieval success

Keep query routing deterministic or use a very small classifier. The senior design question is not “can an LLM classify the query?” but “does the quality gain justify another model call on the critical latency path?”

## 10. Parallel Hybrid Retrieval

The critical path should execute independent searches concurrently.

```text
                 Query
                   │
            ┌──────┴──────┐
            │             │
            ▼             ▼
          BM25        Vector ANN
            │             │
            └──────┬──────┘
                   │
              Merge / RRF
                   │
                   ▼
              Candidate Set
```

This is preferable to sequential retrieval when both indexes are required.

### Rank fusion

A simple RRF model is:

```text
score(d) = Σ 1 / (k + rank_i(d))
```

where each retrieval source contributes according to the document's rank in that source.

In production, compare RRF against learned fusion using offline evaluation rather than assuming one method is universally best.

## 11. Reranking

Reranking is often the highest quality/latency trade-off point in RAG.

```text
BM25 top 100 ─┐
              ├─> Fusion -> top 50 -> Reranker -> top 8
ANN top 100 ──┘
```

Do not send hundreds of candidates into a cross-encoder at 10K QPS without capacity planning.

### Selective reranking

Possible policies:

```text
Very high confidence lexical hit -> skip heavy reranker
Mixed/conflicting candidates    -> heavy reranker
Complex user query              -> heavy reranker
High-value tenant/workload      -> heavy reranker
```

The policy should be driven by measured quality and latency, not intuition.

## 12. Context Builder

The context builder is a first-class ranking component.

Responsibilities:

```text
Candidate chunks
   ↓
remove duplicate content
   ↓
prefer current versions
   ↓
apply diversity / MMR-like selection
   ↓
restore parent headings
   ↓
fit token budget
   ↓
assign citation IDs
```

### Token budget

Treat the generation prompt as a hard resource budget.

For example:

```text
System/instructions        1,000 tokens
Conversation context        1,000
Retrieved evidence          4,000
Expected output              800
Safety / formatting           300
-----------------------------------
Target prompt              ~7,100
```

The exact numbers vary by model. The architectural rule is to bound context rather than fill the model window just because capacity exists.

## 13. Evidence Gate and Abstention

The generation step should be conditional.

```text
Evidence
   ↓
Quality / coverage checks
   ├── enough evidence → generate
   ├── weak evidence   → alternate retrieval
   └── insufficient    → abstain / clarify
```

Signals can include:

- reranker score distribution
- agreement between lexical and dense retrieval
- number of independent supporting chunks
- document freshness
- citation availability
- historical calibrated thresholds

A threshold should be calibrated on an evaluation set. Do not use an arbitrary score cutoff just because it “looks high.”

## 14. Citation Validation

Capture evidence IDs throughout the pipeline.

```text
chunk_id -> document_id -> source -> location
```

The generator references evidence IDs. A validation layer can verify that each citation points to an actual retrieved chunk and that the cited text supports the generated claim.

For critical domains, use a second lightweight verifier or deterministic overlap/entailment checks before emitting a citation.

## 15. LLM Gateway

The LLM gateway abstracts model providers and provides production controls:

```text
Request
  ↓
Model Router
  ↓
Budget / quota checks
  ↓
Provider / model selection
  ↓
Timeout / circuit breaker
  ↓
Streaming
  ↓
Telemetry
```

Responsibilities:

- model routing
- timeout handling
- concurrency limits
- token budgets
- cost accounting
- prompt/version tracking
- provider fallback
- streaming
- content policy hooks

## 16. Latency Budget

A senior candidate should start from an end-to-end target and allocate a budget.

Example p95 time to first token:

```text
API/Auth              40 ms
Query routing         30 ms
BM25                  40 ms
ANN                   60 ms
Fusion                10 ms
Reranking            120 ms
Context building      30 ms
Network/queue         50 ms
LLM TTFT           1,500 ms
--------------------------------
Total              1,880 ms
```

The specific budget is illustrative. During production tuning, measure actual distributions and optimize the largest contributors first.

### Common latency mistakes

1. Sequentially executing independent retrievers.
2. Running an LLM rewrite on every request.
3. Reranking hundreds of candidates.
4. Sending excessive context to the LLM.
5. Waiting for the complete answer before streaming.
6. Using a single heavyweight model for routing, retrieval, and generation.

## 17. Capacity Planning

### Query fan-out

At 10K RPS:

```text
BM25   ≈ 10K QPS
ANN    ≈ 10K QPS
```

If each query sends 50 candidates to a reranker:

```text
10K * 50 = 500K candidate pairs/sec
```

If a reranker instance handles 2K candidate pairs/sec under the chosen SLA:

```text
500K / 2K ≈ 250 instances
```

The exact throughput must be benchmarked on target hardware. The important design insight is that candidate multiplication makes reranking a potential dominant fleet size.

### Vector memory

For 1.2B chunks × 1536 dimensions × float32:

```text
1.2B * 1536 * 4
≈ 7.0 TB raw vectors
```

With replicas and ANN structures, the footprint is much larger.

Therefore evaluate:

- vector quantization
- lower-dimensional embeddings
- product quantization / scalar quantization
- tiered indexes
- tenant/corpus partitioning
- replicas versus availability requirements

## 18. Sharding Strategy

There are several possible shard keys:

```text
Tenant-sharded
Corpus-sharded
Hash(document_id)-sharded
Hybrid routing
```

### Tenant-sharded

Strong isolation and predictable tenant capacity, but uneven tenants can create hot shards.

### Hash-sharded

Better distribution, but every query may require broader fan-out unless metadata routing narrows it.

### Hybrid

Route first by tenant/corpus metadata, then hash within a partition.

For 500 tenants, a practical design is often a hybrid model with isolated capacity pools for very large/hot tenants.

## 19. Caching

Cache layers should reflect semantics.

```text
L1 API cache
   ↓
Query embedding cache
   ↓
Retrieval result cache
   ↓
Rerank cache
   ↓
Optional generation cache
```

Every cache key must account for data that can change correctness:

```text
tenant_id
acl_version
index_version
query_normalized
retrieval_policy
model_version
prompt_version
```

Do not cache an answer solely by raw query text.

### Cache invalidation

Invalidate or namespace by index and ACL versions. A versioned namespace is often safer than trying to synchronously delete every affected cache entry.

## 20. Ingestion Throughput

20M document updates/day is:

```text
20,000,000 / 86,400
≈ 231 updates/sec average
```

At a 10x burst factor:

```text
≈ 2,310 updates/sec
```

This is why ingestion should use queues and independently scalable worker pools.

Parser, OCR, embedding, and indexing workers should be separated because their bottlenecks differ.

## 21. Freshness SLO

Expose freshness as a measurable service-level metric.

```text
source_updated_at
       ↓
index_activated_at
       ↓
freshness_lag = activation - source update
```

Dashboard:

```text
p50 freshness lag
p95 freshness lag
p99 freshness lag
max lag by tenant/source
stale documents served
```

A system that is fast but serves stale policy documents is not a high-quality RAG system.

## 22. Failure Handling

### Vector search down

```text
ANN unavailable
   ↓
BM25 fallback
   ↓
Rerank lexical candidates
   ↓
Generate only if evidence confidence is acceptable
```

### Reranker down

```text
Reranker unavailable
   ↓
Use fused rank
   ↓
Lower candidate budget
   ↓
Increase abstention conservatism
```

### LLM down

```text
LLM unavailable
   ↓
Return retrieval-only evidence/search result
```

The service should fail degraded rather than fabricate an answer.

### Ingestion failure

Keep the last known-good document version active until the new version is completely indexed and validated.

## 23. Multi-Tenancy

Tenant boundaries should exist in every layer:

```text
API
 ↓
Auth
 ↓
Query
 ↓
Index filter
 ↓
Cache namespace
 ↓
Telemetry
 ↓
Storage
```

Avoid relying on a single final filter. Defense in depth reduces the blast radius of implementation mistakes.

## 24. Security and Prompt Injection

Treat retrieved documents as data, not instructions.

```text
System instructions
    !=
User query
    !=
Retrieved content
    !=
Tool/API output
```

Even if a document says “ignore previous instructions,” the generation layer should treat it as evidence text.

Security controls should live outside the LLM:

- tenant authorization
- ACL filtering
- source validation
- tool allowlists where applicable
- structured output
- response validation
- audit trails

## 25. Evaluation Architecture

Evaluation should operate like a separate production system.

```text
                      Golden Dataset
                            |
          +-----------------+-----------------+
          |                                   |
          v                                   v
     Retrieval Eval                      Generation Eval
          |                                   |
    Recall/MRR/NDCG                 Correctness/Faithfulness
    ACL/Freshness                   Citation/Abstention
          |                                   |
          +-----------------+-----------------+
                            |
                            v
                    Regression Tracker
                            |
                  +---------+---------+
                  |                   |
                  v                   v
             Offline Gate         Online Experiment
```

### Evaluation dimensions

| Layer | Metrics |
|---|---|
| Ingestion | completeness, parse failure rate, freshness lag |
| Retrieval | Recall@K, MRR, NDCG, hit rate |
| Security | ACL violation rate, tenant leakage rate |
| Reranker | Precision@K, NDCG delta, latency |
| Context | redundancy, token usage, evidence coverage |
| Generation | correctness, faithfulness, citation support |
| UX | acceptance, follow-up rate, resolution rate |
| Platform | p50/p95/p99, error rate, throughput |
| Economics | cost/request, tokens/request, cache hit rate |

## 26. Offline Evaluation Methodology

Create stratified slices instead of one average score.

```text
All queries
 ├── semantic
 ├── exact-match / error-code
 ├── freshness-sensitive
 ├── multilingual
 ├── long-context
 ├── no-answer
 ├── permission boundary
 └── adversarial / injection
```

An embedding model change should be evaluated on all relevant slices.

### Example promotion rule

Do not promote a new embedding model simply because:

```text
NDCG +2%
```

A stronger rule could require:

```text
Recall@20 >= baseline
NDCG >= baseline + threshold
ACL leakage = 0
freshness correctness >= baseline
p95 retrieval <= budget
cost/query <= budget
```

The exact gates depend on the product.

## 27. Online Evaluation

Offline evaluation catches known cases. Production traffic reveals distribution shift.

Use shadow traffic or A/B tests for:

- retriever versions
- rerankers
- chunking policies
- prompt versions
- model versions
- context budgets

Track quality together with:

```text
Latency
Cost
Abstention
User feedback
Resolution rate
Escalation rate
```

A statistically significant quality improvement is not automatically a win if it breaks the latency SLO or doubles cost.

## 28. Observability

Trace the entire request as one distributed span tree:

```text
request_id
  ├── auth
  ├── query_route
  ├── embedding
  ├── bm25
  ├── ann
  ├── fusion
  ├── reranker
  ├── context_build
  ├── llm
  └── citation_validation
```

Record structured metadata such as:

```text
tenant_id
query_type
retrieval_policy
candidate_count
reranker_count
top_scores
context_tokens
prompt_tokens
completion_tokens
model
cache_hit
freshness_lag
latency per stage
final outcome
```

Sample sensitive content carefully; operational telemetry should not become a secondary data-exfiltration path.

## 29. Cost Model

At high QPS, cost is dominated by generated tokens and model calls, but retrieval infrastructure can also become material.

A useful decomposition is:

```text
Cost/request = retrieval compute
             + reranker compute
             + embedding/query compute
             + LLM input tokens
             + LLM output tokens
             + cache/storage amortization
```

Optimize in this order:

1. remove unnecessary model calls
2. reduce context tokens
3. cache safe repeated work
4. use smaller routing/rewrite models
5. bound reranker candidates
6. choose efficient model serving
7. compress/quantize indexes where quality permits

## 30. Trade-offs

### Hybrid retrieval vs vector-only

Hybrid is operationally heavier but more robust for enterprise terminology and exact matches.

### Rerank everything vs selective reranking

Rerank-everything maximizes quality when capacity is abundant; selective reranking is often required at high QPS.

### Large context vs retrieval precision

More context is not a substitute for better ranking. Large prompts increase cost and can dilute evidence.

### Global index vs tenant partitioning

Global indexes simplify fleet management but increase isolation and noisy-neighbor challenges.

### Query rewrite vs direct retrieval

Rewrite can improve recall for conversational queries but adds latency and another model failure mode. Add it only when offline evaluation proves net benefit.

## 31. Senior-Level Design Checklist

A strong senior candidate should proactively address:

```text
[ ] Requirements and SLOs
[ ] Corpus size and update rate
[ ] Query QPS and concurrency
[ ] Latency budget
[ ] Hybrid retrieval
[ ] Candidate-to-reranker fan-out
[ ] Index sizing and sharding
[ ] ACL filtering
[ ] Tenant isolation
[ ] Freshness / version activation
[ ] Chunking strategy
[ ] Context token budget
[ ] Abstention
[ ] Citation validation
[ ] Caching correctness
[ ] Offline evaluation
[ ] Online experimentation
[ ] Cost controls
[ ] Failure modes
[ ] Observability
[ ] Degraded-mode behavior
```

The expected design quality is not measured by the number of components. It is measured by whether each component exists to satisfy a quantified quality, performance, reliability, security, or cost requirement.
