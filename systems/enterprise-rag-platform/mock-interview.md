# Enterprise RAG Platform — Mock Interview

## Interview Prompt

> Design a RAG platform for a global technology company that lets 5 million employees ask questions over 100M+ documents and 10B+ chunks. The platform must support 200K peak queries/sec, p95 latency below 2.5s, 99.99% availability, strict ACL enforcement, and sub-5-minute freshness for priority sources.

The interviewer is evaluating whether you can design a **production GenAI system**, not whether you can assemble a vector database and an LLM.

---

## 1. Clarify the Problem

### Interviewer
"How would you start?"

### Strong Candidate

I would first clarify four dimensions: workload, quality, freshness, and security.

I would confirm that this is primarily knowledge retrieval rather than transactional data access, ask whether answers need citations, determine whether permissions are document-level or source-level, establish latency and availability SLAs, and identify how frequently sources change.

Given the stated constraints, I would design this as a globally distributed, multi-stage retrieval and generation platform with separate ingestion, indexing, query, and evaluation planes.

### Senior-level follow-up

Ask about:

- Peak QPS vs average QPS.
- Query burstiness.
- Read/write ratio.
- Document/chunk growth rate.
- Number of ACL groups.
- Geographic distribution.
- Percentage of multilingual traffic.
- Percentage of queries requiring multi-hop retrieval.
- Citation and abstention requirements.

---

## 2. Requirements

### Functional

- Ingest multiple enterprise sources.
- Incrementally update documents.
- Delete and supersede content.
- Retrieve using semantic + lexical signals.
- Enforce ACLs.
- Rerank candidates.
- Generate grounded answers with citations.
- Support follow-up questions.
- Support no-answer responses.
- Evaluate every major model/index change.

### Non-functional

- p95 < 2.5s.
- 99.99% availability.
- <5 min priority freshness.
- 200K peak QPS.
- Zero tolerated ACL leakage.
- 10x scale headroom.

---

## 3. Back-of-the-Envelope Estimation

### Given

```text
Users                     = 5M
Documents                 = 100M
Chunks                    = 10B
Peak QPS                  = 200K
Priority freshness SLA    = 5 min
```

### Query traffic

Assume an average of 20K QPS and 200K QPS peak.

For 200K QPS:

```text
200,000 queries/sec
= 17.28B queries/day at peak-equivalent traffic
```

Do not claim all traffic will run at peak. Use a realistic average-to-peak ratio and size for both.

### Chunk storage intuition

If an indexed chunk averages 1 KB compressed content plus metadata and vector-related overhead, raw logical chunk storage is already on the order of 10 TB before index overhead, replication, metadata, and vector memory.

The key interview point is that **vector index memory can become the dominant cost**, so vector compression, quantization, sharding, and tiering matter.

### Ingestion

Suppose 10M chunks change in 5 minutes during a major update window:

```text
10M / 300 ~= 33K chunk updates/sec
```

That motivates an event-driven pipeline with separate ingest, embedding, and indexing capacity and priority queues.

---

## 4. High-Level Architecture

### Interviewer
"Draw the system."

### Candidate

```text
Sources
  -> CDC/Webhooks
  -> Event Bus
  -> Ingestion Workers
  -> Parse / Normalize / Chunk / ACL
  -> Embedding
  -> Distributed BM25 + ANN + Metadata Indexes

User
  -> API Gateway
  -> AuthN/AuthZ Context
  -> Query Router / Rewrite
  -> Parallel BM25 + ANN
  -> Hybrid Fusion
  -> Reranker
  -> ACL Verification
  -> Context Builder
  -> Model Router
  -> LLM
  -> Grounding/Citation Verification
  -> Answer
```

I would explicitly say that authorization is part of retrieval and that the LLM is not the security boundary.

---

## 5. Interviewer: "Why hybrid search?"

### Strong Answer

Because enterprise corpora contain both semantic questions and exact lexical signals.

For example:

```text
"How do I restart the service?"
```

benefits from semantic retrieval, while:

```text
"What changed in INC-48291 after v12.4.7?"
```

benefits heavily from exact term matching.

I would run BM25 and ANN in parallel and combine candidates using RRF or a learned/weighted fusion method, then rerank a smaller set.

I would validate the choice with Recall@K, MRR, NDCG, latency, and cost rather than assuming hybrid is automatically better.

---

## 6. Interviewer: "How do you retrieve at 10B chunks?"

### Strong Answer

I would not query a single enormous index.

I would distribute the index by region and then shard within region, balancing tenant locality and load distribution. I would use ANN with compression/quantization and keep hot, frequently accessed content separate from cold data when economics justify it.

The query fan-out must be bounded. The coordinator should use shard-level deadlines and avoid letting a single slow shard determine p99 latency.

For very high query volumes, I would replicate read-only indexes and independently scale search from the query orchestrator.

---

## 7. Interviewer: "How do you enforce ACLs?"

### Strong Answer

ACLs are captured during ingestion and associated with document/chunk metadata.

At query time, the user's authorization context is converted into deterministic filters. Candidate retrieval is restricted to allowed content and we perform a final authorization check before context construction.

I would also include authorization scope in cache keys and prevent unauthorized content from entering shared semantic caches.

The LLM must never be trusted to decide whether a document is permitted.

---

## 8. Interviewer: "What if filtering after vector search is easier?"

### Strong Answer

I would reject pure post-filtering for protected enterprise data.

If the ANN search first retrieves unauthorized content, that content can influence ranking and could leak through caches, telemetry, or bugs even if later removed.

The preferred design is filter-aware retrieval or pre-filtered candidate generation, followed by final deterministic authorization verification.

If the chosen vector engine has limitations around filtered ANN, I would evaluate partitioning/indexing strategies and quantify the recall/latency impact rather than silently accepting the security risk.

---

## 9. Interviewer: "How do you keep knowledge fresh?"

### Strong Answer

I would use an event-driven incremental ingestion architecture.

```text
Source update
 -> event
 -> parse
 -> diff/content hash
 -> chunk changes
 -> embedding only for changed chunks
 -> index delta
 -> freshness watermark
```

Deletes and superseded versions need first-class handling. I would maintain version metadata and tombstones so stale chunks are not accidentally retrieved.

For freshness SLAs, I would expose metrics such as:

```text
source_update_time -> index_visible_time
```

and alert on the age distribution, not just average freshness.

---

## 10. Interviewer: "How do you optimize latency?"

### Strong Answer

I would define a p95 budget before optimizing individual components.

Example:

```text
Gateway/Auth          50 ms
Rewrite               120 ms
BM25 + ANN            250 ms
Fusion                 30 ms
Rerank                300 ms
Context                80 ms
LLM                  1200 ms
Grounding             250 ms
Overhead              170 ms
--------------------------
Total               ~2450 ms
```

I would parallelize independent work, keep candidate sets bounded, use fast rerankers, reduce prompt tokens, route simple queries to smaller models, stream the response, and use deadline-aware fallbacks.

For tail latency, I would monitor p95 and p99 by stage. Average latency can hide a shard or model hotspot that destroys user experience.

---

## 11. Interviewer: "How do you scale to 200K QPS?"

### Strong Answer

I would scale the query orchestrator, search tier, reranking tier, and model inference tier independently.

```text
                    +--> Query workers
Load Balancer ------+--> Search coordinators
                    +--> Rerank workers
                    +--> LLM inference pool
```

Search indexes would have replicas. Model inference would use batching and continuous batching where supported.

I would use admission control and per-tenant quotas so one customer or workload cannot saturate the platform.

The system needs regional isolation so a failure in one region does not take down the entire service.

---

## 12. Interviewer: "Where does caching help?"

### Strong Answer

There are several cacheable layers:

1. Query normalization / rewrite.
2. Retrieval results.
3. Reranking results for repeated candidate sets.
4. Context construction.
5. Semantic answer cache where policy and freshness allow it.
6. Model prompt-prefix caching where supported.

But cache keys must include authorization scope, locale, and an index/version freshness dimension.

A global `query -> answer` cache is unsafe for an enterprise knowledge system.

---

## 13. Interviewer: "How do you evaluate RAG?"

### Strong Answer

I separate evaluation into retrieval, generation, security, and operations.

### Retrieval

- Recall@5 / 10 / 20.
- MRR.
- NDCG.
- Hit rate.
- Correct source/version retrieval.

### Generation

- Answer correctness.
- Groundedness / faithfulness.
- Citation precision.
- Citation recall.
- Completeness.
- Abstention quality.

### Security

- ACL leakage rate.
- Prompt injection success rate.
- Unauthorized citation rate.

### Production

- p50/p95/p99 latency.
- Error rate.
- Cost/request.
- Token usage.
- Cache hit rate.
- Freshness lag.

I would maintain a golden benchmark and run it for every embedding, retriever, reranker, prompt, and model change.

---

## 14. Interviewer: "Recall@20 is 95%, but answer correctness is only 75%. What do you investigate?"

### Strong Answer

I would decompose the pipeline instead of changing the LLM immediately.

```text
Relevant document retrieved?
        |
        +-- no --> retrieval problem
        |
       yes
        v
Relevant chunk ranked high?
        |
        +-- no --> reranker problem
        |
       yes
        v
Correct version selected?
        |
       no --> freshness/version problem
        |
       yes
        v
Evidence retained in context?
        |
       no --> context packing problem
        |
       yes
        v
Model used evidence correctly?
        |
       no --> generation / prompt problem
```

This is one of the most important senior-level RAG debugging patterns: **measure each stage separately before changing the whole architecture.**

---

## 15. Interviewer: "What if retrieval quality improves but latency increases 2x?"

### Strong Answer

I would treat it as a quality-latency frontier problem.

For example, moving from 50 to 300 reranking candidates might improve Recall@K marginally but break the SLA.

Possible responses:

- Reduce candidate count.
- Use a cheaper first-stage reranker.
- Apply adaptive reranking only to low-confidence queries.
- Cache frequent retrievals.
- Increase parallelism.
- Optimize the index.
- Use a stronger model only where it changes business outcomes.

I would make the decision from experiment data, e.g.:

```text
Variant A: +4% quality, +600 ms
Variant B: +2% quality, +70 ms
```

For an interactive product, B may be the better production choice.

---

## 16. Interviewer: "How would you handle no-answer questions?"

### Strong Answer

Abstention must be designed explicitly.

Signals can include:

- weak retrieval confidence
- poor agreement across retrievers
- low reranker scores
- conflicting evidence
- missing authoritative source
- verifier failure

The system should prefer:

```text
"I couldn't find reliable evidence for that."
```

over hallucinating a confident answer.

I would measure abstention precision and recall rather than simply minimizing the number of abstentions.

---

## 17. Interviewer: "What happens when the LLM is down?"

### Strong Answer

The system should degrade deliberately.

```text
LLM failure
   |
   +--> alternate model
   |
   +--> smaller fallback model
   |
   +--> retrieval-only response with sources
   |
   +--> safe error / retry
```

The fallback should be bounded by latency and cost budgets. I would never retry an unavailable model indefinitely.

---

## 18. Interviewer: "How do you roll out a new embedding model?"

### Strong Answer

Never replace the production index in place.

```text
Current v1 index
      |
New v2 index built in parallel
      |
Offline benchmark
      |
Shadow queries
      |
Canary traffic
      |
Gradual rollout
```

Track:

- Recall@K.
- MRR/NDCG.
- latency.
- memory footprint.
- indexing throughput.
- answer quality.

Rollback should be a fast control-plane switch.

---

## 19. Interviewer: "How do you protect against malicious documents?"

### Strong Answer

Retrieved documents are untrusted data.

I would defend with:

- source sanitization
- content classification
- prompt-injection detection
- clear evidence/instruction separation in prompts
- output validation
- citation verification
- adversarial benchmark cases

The model must never interpret arbitrary document text as a higher-priority instruction.

---

## 20. Interviewer: "What is your biggest architectural concern?"

### Strong Answer

At this scale, I would worry less about the LLM and more about the distributed retrieval system: authorization-aware indexing, fan-out latency, index memory, freshness, hot shards, and evaluation drift.

The LLM is only one stage in the pipeline.

A RAG system can fail even with an excellent model if the right evidence is not retrieved, if the wrong version is selected, if ACLs are wrong, or if latency prevents the product from being usable.

---

## 21. Common Weak Answers

### "Just use a vector database"

Insufficient because it ignores lexical retrieval, ACLs, sharding, freshness, reranking, and scale.

### "Use top 5 chunks"

Candidate count must be benchmarked. Different query classes need different retrieval depth.

### "Use a bigger model"

Does not fix missing or wrong evidence and may increase latency and cost.

### "Filter permissions after retrieval"

Dangerous for enterprise data; unauthorized data may influence ranking or leak through other layers.

### "Evaluate with a few example questions"

Insufficient. Use a structured golden set with retrieval labels, answer labels, security cases, and regression gates.

### "Add a cache"

A cache without authorization and freshness-aware keys can serve incorrect or unauthorized answers.

---

## 22. Final 5-Minute Senior Answer

If the interviewer asks for a concise final design:

> I would build the RAG platform as a globally distributed retrieval and inference system with separate ingestion, index, query, and evaluation planes. The ingestion pipeline is event-driven and incrementally processes parsing, chunking, ACL enrichment, embeddings, and versioned index updates so priority sources stay within a five-minute freshness SLA. On the query path, I would authenticate the user, preserve the security context, normalize the query, and run BM25 and ANN retrieval in parallel, followed by fusion, ACL-aware filtering, reranking, deduplication, and token-budgeted context construction. A model router selects a small, medium, or large model based on query complexity and confidence, while a grounding/citation verifier validates evidence before returning the answer. To meet 200K peak QPS and p95 under 2.5 seconds, I would use regional replicas, horizontally scalable stateless services, bounded search fan-out, parallel retrieval, adaptive reranking, caching, model batching, and explicit stage-level latency budgets. Reliability would use deadlines, circuit breakers, replicas, and controlled fallbacks. Evaluation would be first-class: Recall@K, MRR, NDCG, groundedness, citation correctness, abstention quality, ACL safety, p95/p99 latency, cost, and freshness would all be tracked, with golden-set regression gates and staged canary rollout for index or model changes."

---

## Interviewer Scorecard

| Area | Weak | Strong | Senior |
|---|---|---|---|
| RAG fundamentals | Vector DB + LLM | Hybrid retrieval | Multi-stage retrieval with evidence control |
| Scale | Generic scaling | Sharding/replicas | Capacity model, hot shards, bounded fan-out |
| Performance | Talks average latency | p95/p99 | Stage budgets + adaptive trade-offs |
| Evaluation | LLM answer quality | Retrieval + generation metrics | Full pipeline + regression/online eval |
| Security | Mentions ACL | ACL-aware retrieval | Authorization as retrieval invariant + cache isolation |
| Freshness | Periodic reindex | Incremental indexing | Event-driven updates + version/watermark design |
| Cost | Smaller model | Caching | Quality-cost frontier and per-request economics |
| Reliability | Retries | Fallbacks | Deadlines, circuit breakers, regional failure isolation |
| AI depth | Prompting | RAG pipeline | Model routing, reranking, context economics, eval loops |
| Production maturity | Basic deployment | Monitoring | Canary, rollback, SLOs, incident/debug methodology |
