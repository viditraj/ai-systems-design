# Enterprise RAG Platform — Architecture

## 1. Design Target

We need a globally distributed RAG platform for 100M+ documents and 10B+ chunks, with 200K peak queries/sec, p95 interactive latency below 2.5s, strict authorization, and sub-5-minute freshness for priority sources.

The architecture separates five planes:

```text
1. Control Plane       -> schemas, models, policies, experiments
2. Ingestion Plane     -> source sync, parsing, chunking, embedding
3. Index Plane         -> lexical, vector, metadata and ACL indexes
4. Query Plane         -> retrieve, rerank, build context, generate
5. Evaluation Plane    -> offline benchmark + online telemetry + rollout
```

---

## 2. High-Level Architecture

```text
                                  +----------------------+
                                  | Enterprise Sources   |
                                  | SharePoint/Drive/etc |
                                  +----------+-----------+
                                             |
                                      CDC / Webhooks
                                             |
                                      +------v------+
                                      | Event Bus   |
                                      | Kafka/PubSub |
                                      +------+------+
                                             |
                                 +-----------v------------+
                                 | Ingestion Orchestrator |
                                 +-----------+------------+
                                             |
                             +---------------+----------------+
                             |                                |
                      Parse / Normalize                 ACL Resolver
                             |                                |
                      Chunk / Metadata               Access Metadata
                             |                                |
                         Embedding                       Versioning
                             +---------------+----------------+
                                             |
                                  +----------v----------+
                                  | Indexing Layer      |
                                  | BM25 | ANN | Meta   |
                                  | ACL  | Freshness     |
                                  +----------+-----------+
                                             |
                              Regional replicated indexes

User
  |
  v
+-------------------+
| API Gateway / WAF |
+---------+---------+
          |
   AuthN / Tenant Context
          |
+---------v----------+
| Query Router       |
| classify/rewrite   |
+---------+----------+
          |
   +------+------+
   |             |
   v             v
Semantic       Keyword
Search         Search
   |             |
   +------+------+
          v
   Hybrid Fusion / RRF
          |
     ACL validation
          |
       Reranker
          |
  Dedup / diversify
          |
 Context selection
          |
  +-------v--------+
  | Model Router   |
  | small / medium |
  | / large model  |
  +-------+--------+
          |
       LLM
          |
 Grounding / citation
       verifier
          |
      Response
```

---

## 3. Query Lifecycle

### Step 1 — Authentication and context

The gateway authenticates the user and propagates an immutable security context:

```text
user_id
tenant_id
region
roles
security_groups
resource_scopes
locale
experiment_bucket
```

The retrieval service uses this context to apply deterministic authorization filters.

### Step 2 — Query routing

A lightweight model or classifier decides whether the request is:

- knowledge retrieval
- live transactional lookup
- summarization over provided text
- unsupported / unsafe
- multi-hop knowledge request

Do not use RAG for questions whose authoritative answer lives in a transactional system.

### Step 3 — Query normalization

Examples:

```text
"why is INC-48291 failing after 12?"
        |
        +--> extract identifier: INC-48291
        +--> normalize product/version: 12
        +--> generate semantic query
        +--> preserve exact tokens for lexical retrieval
```

Keep both the original and rewritten query. Over-aggressive rewriting can destroy identifiers and should be benchmarked.

### Step 4 — Parallel candidate generation

Run lexical and semantic retrieval concurrently.

```text
             Query
            /     \
           /       \
        BM25       ANN
         |           |
      top 100      top 100
           \       /
            \     /
          Fusion / RRF
              |
           top 50
```

Candidate generation should happen close to the data and use pre-filtered shards whenever possible.

### Step 5 — Reranking

A reranker scores a smaller candidate set using richer query-document interactions.

```text
100-200 candidates
        |
 reranker (GPU/CPU optimized)
        |
    top 10-20
```

Use a fast reranker for the common path and a stronger reranker only for hard queries where offline evaluation shows a measurable benefit.

### Step 6 — Evidence construction

Before the LLM sees content:

1. Verify authorization.
2. Remove stale or superseded versions.
3. Deduplicate near-identical chunks.
4. Prefer diverse sources when appropriate.
5. Preserve document title, section, URL, version, timestamp, and source ID.
6. Fit evidence into a token budget.

### Step 7 — Generation

Use a model router based on query complexity, evidence quality, SLA, and cost.

```text
Simple fact lookup ----> small/fast model
Normal synthesis ------> medium model
Complex multi-hop -----> large reasoning model
Low confidence --------> abstain / clarify / retrieve more
```

### Step 8 — Grounding validation

A verifier checks whether claims are supported by retrieved evidence and whether cited sources actually contain the cited information.

For latency-sensitive traffic, use a lightweight verifier or confidence model; reserve expensive verification for high-risk domains.

---

## 4. Ingestion Architecture

### Event-driven incremental ingestion

```text
Source change
    |
    v
CDC/Webhook -> Event Bus -> Ingestion Worker
                          |
             +------------+-------------+
             |                          |
           parse                      ACL sync
             |
          normalize
             |
          chunking
             |
        metadata enrich
             |
         embed batch
             |
      update index version
```

### Why not full re-indexing?

At 10B+ chunks, rebuilding everything after each source change is too expensive and violates freshness targets. Incremental indexing allows localized updates.

### Chunk versioning

Store:

```text
document_id
chunk_id
version_id
content_hash
embedding_model_version
index_version
updated_at
deleted_at
```

A content hash prevents duplicate embedding work.

### Backpressure

When embedding or indexing capacity is saturated:

```text
source events -> durable queue -> backlog
                              |
                         priority lanes
                              |
                 critical / normal / bulk
```

High-priority sources receive reserved capacity to meet freshness SLAs.

---

## 5. Index Architecture

### Logical indexes

Maintain separate logical indexes for:

- lexical retrieval
- vector retrieval
- metadata filters
- ACL / authorization metadata
- document version state

A document can be deleted or replaced without immediately compacting every underlying structure.

### Sharding

Candidate strategy:

```text
Region -> tenant tier -> hash(document_id) -> shard
```

For very large tenants, shard by document namespace or hash to avoid a single tenant becoming a hot shard.

Do not blindly shard only by tenant: a large enterprise tenant can dominate traffic.

### Replication

Use:

```text
Primary region
    |
 async replication
    +----> secondary region
    +----> disaster recovery region
```

Query indexes should support regional reads. Writes can originate from source-local ingestion and propagate asynchronously.

### Vector optimization

For 10B+ chunks, memory efficiency is critical. Consider:

- scalar / product quantization
- compressed embeddings
- multi-stage ANN
- smaller embedding dimensions where benchmark quality is acceptable
- hot/cold index tiers
- approximate search followed by exact reranking

The decision must come from recall-vs-memory benchmarks, not model popularity.

---

## 6. ACL / Security Architecture

Authorization is enforced before generation.

```text
User Security Context
        |
        v
ACL Filter -> candidate retrieval -> verify permission
                                      |
                                      v
                                allowed evidence
                                      |
                                      v
                                     LLM
```

### Defense in depth

1. Source connector captures ACLs.
2. ACL metadata is indexed with chunks.
3. Query-time filters restrict candidates.
4. Final authorization check occurs before context construction.
5. Response citations are validated against the same policy context.

### Prompt injection

Retrieved documents are untrusted data.

The prompt must clearly separate:

```text
SYSTEM INSTRUCTIONS
USER QUERY
RETRIEVED EVIDENCE (UNTRUSTED)
```

A malicious document saying "ignore previous instructions" must never become an instruction to the model.

---

## 7. Caching Strategy

Caching should reduce cost and tail latency without violating authorization or freshness.

### Query cache

Cache normalized query + security scope + index version.

Bad key:

```text
query_only
```

Better:

```text
hash(
  normalized_query,
  tenant_id,
  permission_scope,
  locale,
  index_version
)
```

### Semantic cache

Use nearest-neighbor similarity over prior queries, but only when:

- authorization scopes match
- relevant documents have not changed
- freshness SLA permits reuse
- confidence threshold passes

Semantic caching can create wrong answers when ACL or freshness is ignored.

---

## 8. Performance Engineering

### Latency budget

Target p95 < 2.5s.

```text
Gateway + auth          50 ms
Query rewrite           120 ms
Parallel BM25 + ANN     250 ms
Fusion                  30 ms
Reranking               300 ms
Context build            80 ms
LLM generation        1,200 ms
Grounding check         250 ms
Network / overhead      170 ms
--------------------------------
Total                  2,450 ms
```

The actual LLM budget depends on token count and generation length. The design must measure each stage separately.

### Performance levers

- Parallel retrieval.
- Co-located search and metadata services.
- Connection pooling.
- ANN tuning by recall/latency benchmark.
- Candidate cap before expensive reranking.
- Token-budget-aware context construction.
- Prompt prefix caching where supported.
- Response streaming.
- Small-model routing for easy queries.
- Adaptive reranking for difficult queries.
- Hot query and metadata caching.
- Batch embeddings on ingestion.
- GPU batching for rerankers/LLMs.

### Tail latency

Optimize p95/p99, not just average latency.

Protect the system from slow shards with:

- per-shard deadlines
- hedged requests for stragglers
- bounded fan-out
- partial results
- adaptive replica selection

For example, if 20 shards are queried in parallel, the slowest shard can dominate the request even when 19 shards return quickly.

---

## 9. Scalability

### Query path

Stateless query services scale horizontally:

```text
Load Balancer
     |
+----+----+----+
|    |    |    |
Q1   Q2   Q3  ... QN
```

Search and model inference scale independently.

### Multi-region

Route users to the nearest healthy region. Maintain local read replicas of search indexes and model inference capacity.

### Hot tenant / hot query protection

A single tenant or trending query can create disproportionate load. Apply:

- per-tenant quotas
- priority queues
- admission control
- cache warming
- replica fan-out limits

### Ingestion autoscaling

Scale consumers based on:

- queue depth
- event age
- embedding GPU utilization
- indexing throughput
- freshness SLA violation rate

---

## 10. Reliability and Failure Handling

| Failure | Strategy |
|---|---|
| ANN shard unavailable | Query replica; degrade to lexical retrieval if needed |
| BM25 unavailable | Use ANN-only degraded path; measure quality impact |
| Reranker unavailable | Return fused candidates / cheaper reranker |
| Embedding service down | Queue ingestion; preserve source events |
| LLM unavailable | Model fallback / retrieval-only response / controlled retry |
| Grounding verifier timeout | Safe degraded response according to risk policy |
| Kafka backlog | Prioritize critical streams and autoscale consumers |
| Index corruption | Roll back index version |
| Stale replica | Route away using freshness watermark |
| Authorization service down | Fail closed for protected retrieval |

Never blindly retry side-effect-free and expensive operations indefinitely. Every downstream call needs bounded timeout, retry budget, and circuit breaking.

---

## 11. Evaluation Architecture

Evaluation is a first-class production subsystem.

```text
                    +----------------+
                    | Golden Dataset  |
                    +-------+--------+
                            |
         +------------------+------------------+
         |                  |                  |
     Retrieval           Generation         Security
     evaluation          evaluation          evaluation
         |                  |                  |
 Recall@K              correctness         ACL leakage
 MRR                   faithfulness        prompt injection
 NDCG                  citation accuracy    policy safety
         |                  |                  |
         +------------------+------------------+
                            |
                     Release Gate
                            |
                  production rollout
```

### Offline retrieval metrics

- Recall@K
- Precision@K
- MRR
- NDCG
- hit rate
- source/version correctness
- ACL-filter correctness

### Generation metrics

- groundedness / faithfulness
- answer correctness
- citation precision
- citation recall
- completeness
- abstention quality

### Operational metrics

- p50/p95/p99 latency
- tokens/request
- model cost/request
- cache hit rate
- retrieval fan-out
- reranker rate
- error rate
- freshness lag

### Evaluation dataset composition

Do not build the benchmark from only easy questions.

Include:

- exact identifier queries
- semantic queries
- ambiguous questions
- multi-hop questions
- conflicting versions
- stale documents
- deleted documents
- multilingual queries
- permission boundaries
- prompt injection payloads
- no-answer questions
- long documents
- adversarial phrasing

### Online evaluation

Log:

```text
query_id
retrieval_model_version
embedding_version
index_version
candidate_ids
reranked_ids
selected_context
model_version
answer
citations
latency_by_stage
tokens
cost
user feedback
```

Sample traces for human review and compare variants with controlled experiments.

---

## 12. Retrieval Evaluation: A Senior Interview Deep Dive

Suppose Recall@20 is 93% but end-to-end answer quality is only 78%.

Do not immediately replace the LLM.

Investigate in this order:

```text
1. Is the relevant document retrieved?
2. Is the relevant chunk ranked highly enough?
3. Is the correct version selected?
4. Is evidence removed by ACL or filtering?
5. Is reranker losing useful candidates?
6. Is context packing dropping evidence?
7. Is the LLM failing to use evidence?
8. Is citation generation wrong?
```

This isolates whether the problem is retrieval, ranking, context construction, or generation.

---

## 13. Model Strategy

Use model routing rather than one expensive model for every query.

```text
Classifier
   |
   +--> simple FAQ -> small model
   |
   +--> normal RAG -> medium model
   |
   +--> multi-hop / complex -> large reasoning model
   |
   +--> high-risk domain -> stronger verifier / policy path
```

Model decisions should be measurable against a quality-cost frontier.

### Embedding model upgrades

Never replace an embedding model in place.

Use:

```text
v1 index
v2 index (shadow)
       |
 offline benchmark
       |
 online canary
       |
 traffic ramp
       |
 v2 primary
```

Maintain compatibility metadata so every retrieved result can be traced to an embedding and index version.

---

## 14. Cost Model

Major cost drivers:

```text
1. Embedding ingestion
2. Vector storage / replication
3. Search compute
4. Reranking
5. LLM input tokens
6. LLM output tokens
7. Evaluation / observability
```

High-impact optimizations:

- Deduplicate content before embedding.
- Incremental indexing.
- Smaller or quantized embeddings where benchmark-safe.
- Cache hot queries and reusable context.
- Limit candidate count before reranking.
- Use smaller models for easy queries.
- Cap generation length.
- Summarize context only when it improves cost without hurting grounding.
- Archive cold content to cheaper storage/index tiers.

A senior answer should state cost per query and cost per 1M queries as tracked business metrics, not merely say "use a cheaper model."

---

## 15. Key Trade-offs

### Vector-only vs hybrid

Vector-only is simpler but weak for exact terms. Hybrid retrieval usually performs better when the corpus contains identifiers, names, codes, and terminology.

### Cross-encoder vs LLM reranking

Cross-encoders are usually cheaper and more predictable for large candidate sets. LLM reranking can improve difficult cases but can violate interactive latency or cost targets.

### Large chunks vs small chunks

Large chunks preserve context but waste tokens and reduce retrieval precision. Small chunks improve precision but can lose necessary context. Parent-child retrieval is a strong compromise.

### Precompute vs on-demand

Precomputing embeddings and indexes lowers query latency but increases storage and update cost.

### Strong authorization filtering vs post-filtering

Post-filtering is simpler but dangerous because unauthorized content may already influence ranking or cache behavior. Apply authorization as early as practical and verify again before generation.

### Global vs regional index

A global index simplifies consistency but increases cross-region latency and blast radius. Regional read indexes improve latency and fault isolation at the cost of replication complexity.

---

## 16. Production Rollout

Use staged rollout:

```text
Offline benchmark
      |
Shadow traffic
      |
1% canary
      |
5% -> 25% -> 50% -> 100%
```

Release gates should include:

- Recall regression threshold.
- Citation correctness.
- Groundedness.
- p95/p99 latency.
- error rate.
- cost per request.
- freshness SLA.
- zero critical ACL violations.

Rollback should be a control-plane operation that points traffic to the previous known-good model/index configuration.

---

## 17. What a Strong Senior Candidate Should Say

A strong candidate will explicitly connect product requirements to engineering decisions:

```text
Freshness SLA  -> event-driven incremental indexing
High QPS       -> stateless query tier + horizontally scaled search
Low p95        -> parallel retrieval + bounded fan-out + model routing
Exact terms    -> hybrid BM25 + ANN
Large corpus   -> sharding + compression + hot/cold tiers
ACL isolation  -> pre-filtering + final authorization verification
Quality        -> reranking + benchmark-driven tuning
Reliability    -> regional replicas + fallbacks + circuit breakers
Cost           -> caching + smaller models + token budgets
Trust          -> citations + grounding verification + abstention
```

The architecture should evolve from measured bottlenecks rather than adding every possible AI component upfront.
