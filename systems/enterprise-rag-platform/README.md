# Enterprise RAG Platform

> Design a production-grade Retrieval-Augmented Generation platform for a global enterprise that answers employee questions over millions of documents while maintaining high retrieval quality, strict authorization, low latency, freshness SLAs, predictable cost, and measurable answer quality.

## Real-World Company Problem Statement

**Scenario: Microsoft Copilot-style enterprise knowledge platform**

A global technology company wants to build an internal AI knowledge assistant that lets employees ask natural-language questions across engineering, product, HR, finance, legal, support, and operations knowledge.

The platform must ingest information from SharePoint, Confluence, Google Drive, internal Git repositories, PDFs, Word documents, wikis, tickets, and approved databases. Employees should receive grounded answers with citations and links to authoritative sources.

The system must handle:

- 5 million employees and contractors.
- 100+ million documents and 10+ billion chunks.
- 100,000+ document updates/minute during peak ingestion periods.
- 200,000 queries/sec globally at peak, with regional traffic skew.
- p95 interactive query latency under 2.5 seconds.
- 99.99% query availability.
- Knowledge freshness of under 5 minutes for high-priority sources.
- Strict document- and tenant-level authorization.
- Multi-language content and queries.
- Exact identifiers such as incident IDs, product names, error codes, and policy numbers.
- Answers that must be traceable to source evidence.
- Continuous offline and online evaluation.

The interviewer should push the candidate beyond a basic vector database + LLM design. The expected discussion includes retrieval quality, indexing strategy, sharding, caching, freshness, authorization, evaluation, latency budgets, model routing, cost controls, failure isolation, and rollout strategy.

## Core Design Goal

**Optimize for grounded answer quality under explicit latency, freshness, security, scale, and cost constraints.**

RAG is not just a vector search problem. It is a distributed information retrieval and inference system.

## Requirements

### Functional

1. Ingest heterogeneous enterprise sources.
2. Parse, normalize, enrich, chunk, embed, and index documents.
3. Support keyword, semantic, metadata, and hybrid retrieval.
4. Support citations and source previews.
5. Enforce permissions before generation.
6. Handle document versions, deletions, and stale data.
7. Support follow-up questions and conversational context without blindly embedding the entire chat history.
8. Support multi-language queries.
9. Return explicit "I don't know" / insufficient-evidence responses.
10. Support evaluation, experimentation, and model/index versioning.

### Non-functional

| Requirement | Target |
|---|---:|
| Query availability | 99.99% |
| Interactive latency | p95 < 2.5s |
| Freshness | < 5 min for priority sources |
| Retrieval service availability | 99.99% |
| Citation correctness | > 98% on golden set |
| Retrieval Recall@20 | > 95% on target benchmark |
| Grounded answer rate | > 95% on benchmark |
| Unauthorized retrievals | 0 tolerated |
| Horizontal scalability | 10x traffic without redesign |

## Why This Is a Senior GenAI System Design Problem

A junior design often stops at:

```text
Documents -> Embeddings -> Vector DB -> Top-K -> LLM
```

A senior design must answer:

- How do we retrieve the right evidence when semantic search and exact search disagree?
- How do we prevent ACL leaks before any unauthorized content reaches the LLM?
- How do we support 10+ billion chunks without making every query expensive?
- How do we keep indexes fresh without rebuilding huge partitions?
- How do we measure whether retrieval improvements actually improve final answers?
- How do we detect answer regressions before production rollout?
- Which stages belong on CPU vs GPU?
- What is the end-to-end latency budget?
- When should we skip retrieval or use a different path entirely?
- How do we control token, embedding, reranker, and storage costs?

## Reference Architecture

See [architecture.md](./architecture.md) for the detailed design, request lifecycle, ingestion pipeline, retrieval architecture, sharding model, security controls, caching strategy, evaluation framework, capacity estimates, and production trade-offs.

## High-Level Flow

```text
                         +-------------------+
                         | Enterprise Sources|
                         +---------+---------+
                                   |
                              Event Bus / CDC
                                   |
                    +--------------v---------------+
                    | Ingestion / Processing Layer |
                    | parse -> clean -> chunk      |
                    | metadata -> ACL -> embed     |
                    +--------------+---------------+
                                   |
                    +--------------v---------------+
                    | Distributed Search / Indexing |
                    | BM25 + ANN + Metadata + ACL  |
                    +-------------------------------+

User -> API Gateway -> Query Router -> Query Rewrite
                         |                |
                         |                +--> query normalization
                         v
                 Permission Context
                         |
              +----------+----------+
              |                     |
           BM25                    ANN
              |                     |
              +----------+----------+
                         v
                 Candidate Fusion
                         |
                     Reranker
                         |
                 Evidence Filter
                         |
                 Context Builder
                         |
                  LLM / Model Router
                         |
              Grounding / Citation Check
                         |
                    Final Answer
```

## Key Design Principles

### 1. Retrieval is multi-stage

Use:

```text
Query
  -> classify / rewrite
  -> ACL-aware candidate generation
  -> hybrid retrieval
  -> fusion
  -> rerank
  -> diversify / deduplicate
  -> context selection
  -> generation
```

Do not send a large unfiltered top-K directly to the LLM.

### 2. Authorization is part of retrieval

ACLs must be represented in the retrieval layer and enforced before content enters the generation context.

The LLM is never the authorization boundary.

### 3. Retrieval and generation are evaluated separately

A high-quality answer cannot compensate for missing evidence.

Measure retrieval quality independently from generation quality, then measure end-to-end business outcomes.

### 4. Freshness is an indexing problem, not a prompt problem

Use event-driven incremental updates, tombstones, versioning, and asynchronous embedding/indexing rather than repeatedly rebuilding the entire corpus.

### 5. Optimize the latency budget, not just model inference

A realistic p95 budget could be:

```text
Gateway / auth          50 ms
Query rewrite           120 ms
Parallel retrieval      250 ms
Fusion + rerank         300 ms
Context construction     80 ms
LLM generation         1200 ms
Grounding check         250 ms
Network / overhead      150 ms
--------------------------------
Total                  ~2.4 s
```

The exact budget should be validated against workload and model characteristics.

## Advanced Topics To Discuss

- Hybrid retrieval: BM25 + ANN.
- Reciprocal Rank Fusion / weighted fusion.
- Cross-encoder or lightweight reranker.
- Query decomposition for multi-hop questions.
- Parent-child retrieval.
- Hierarchical retrieval.
- Metadata filtering.
- Freshness-aware ranking.
- Semantic caching.
- Query/result caching.
- Hot-shard protection.
- Tenant-aware sharding.
- Vector compression / quantization.
- ANN index choice and tuning.
- Event-driven indexing.
- Backpressure and dead-letter queues.
- Embedding model version migration.
- Index versioning and blue/green rollout.
- Retrieval fallback paths.
- Model routing and adaptive generation.
- Cost-aware context selection.
- Citation verification.
- Prompt injection defense in retrieved content.
- Poisoned / malicious documents.
- PII / sensitive content filtering.
- Multi-region replication and disaster recovery.
- Online experimentation.
- Golden datasets and continuous evaluation.

## Interview Focus

The mock interview intentionally prioritizes:

**Performance, scale, retrieval quality, evaluation, reliability, security, freshness, cost, and operational maturity.**

See [mock-interview.md](./mock-interview.md).
