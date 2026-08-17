# Enterprise AI Assistant

> Production-grade enterprise AI assistant that answers questions over internal knowledge, retrieves information with permission-aware hybrid RAG, performs actions through controlled tools, supports human approval for risky operations, and scales to 100,000 employees.

## Problem Statement

Build an enterprise AI assistant that can:

- Answer questions over internal company knowledge.
- Summarize documents and explain policy changes.
- Search PDFs, Word documents, Confluence, SharePoint, Slack, Jira, and internal databases.
- Perform actions such as creating/updating Jira tickets, scheduling meetings, and sending documents.
- Respect enterprise permissions and tenant boundaries.
- Require human approval for high-impact actions.
- Recover safely from tool failures and retries.
- Provide citations, auditability, observability, and evaluation.

Assume 100,000 registered employees, up to 5,000 concurrent users, approximately 500 requests/sec at peak, and knowledge freshness within a few minutes.

## Core Design Principle

**The LLM can reason and propose actions, but deterministic infrastructure controls authorization, policy, and execution.**

The LLM is not the security boundary and is not trusted to directly execute arbitrary enterprise operations.

## Architecture

See the detailed architecture, data flows, permission model, RAG design, agent workflow, scaling, failure handling, observability, and interview discussion in [architecture.md](./architecture.md).

## Main Components

| Component | Responsibility |
|---|---|
| API Gateway | Authentication, rate limiting, WAF, routing |
| Identity / Authorization | User identity, groups, tenant and resource permissions |
| AI Orchestrator | Coordinates planning, retrieval, tools, approval, and response lifecycle |
| Query / Intent Router | Determines knowledge, action, clarification, or unsupported requests |
| Memory Service | Stores short-lived conversation state and approved user preferences |
| RAG Service | Permission-aware hybrid retrieval and reranking |
| Knowledge Ingestion Pipeline | Connectors, parsing, chunking, metadata, ACL indexing, embeddings |
| Vector Index | Semantic retrieval |
| Keyword Search Index | Exact lexical retrieval / BM25 |
| Reranker | Reorders candidate documents/chunks |
| LLM Gateway | Model routing, timeouts, fallback, token/cost controls |
| Policy Engine | Deterministic authorization and action-risk decisions |
| Tools Gateway | Validates and controls all enterprise API calls |
| Approval Service | Human-in-the-loop approval for risky actions |
| Tool Executors | Jira, Slack, calendar, SharePoint, Confluence, databases, etc. |
| Task Queue | Durable asynchronous workflow execution |
| State Store | Workflow state, idempotency, audit metadata |
| Observability | Logs, traces, metrics, AI evaluation, audit events |

## Request Flows

### 1. General Knowledge Question

```text
User -> API Gateway -> AI Orchestrator -> Intent Router
     -> Permission-Aware Hybrid Retrieval -> Reranker
     -> LLM -> Answer + Citations -> User
```

### 2. User-Specific Question

```text
User -> Intent Router -> Tools Gateway
     -> Authoritative Enterprise Service -> Response Composer -> User
```

Live user-specific data should come from authoritative backend services rather than the vector database.

### 3. Multi-Step Action

```text
User -> Planner -> Structured Plan -> Policy Engine
     -> Tools Gateway -> Tool Executor -> Validate Result
     -> Next Step / Complete
```

### 4. High-Risk Action

```text
User Request -> Planner -> Policy Engine
     -> HUMAN_APPROVAL -> Approval Service
     -> Tool Executor -> Verification -> User
```

High-impact or destructive actions should not be executed solely because an LLM requested them.

## RAG Design

Use permission-aware hybrid retrieval:

```text
Query
  |
  +--> BM25 / Keyword Search
  +--> Vector Search
          |
          +--> Reciprocal Rank Fusion
                         |
                         v
                      Reranker
                         |
                         v
                 Top-K Allowed Chunks
                         |
                         v
                        LLM
```

Index metadata should include tenant ID, document ID, source, section, version, timestamps, effective dates, classification, and permission/group metadata.

Permission filters should be applied before content is allowed into the generation context. Customer/account data should remain in authoritative services rather than the RAG index.

## Tool Safety

Every tool request passes through a controlled Tools Gateway:

```text
LLM / Planner -> Tool Schema Validation -> Authentication
              -> Authorization -> Policy Engine
              -> Idempotency Check -> Tool Executor
              -> Result Validation
```

The LLM cannot bypass the gateway by emitting arbitrary API requests.

## Human Approval

| Action | Risk | Default |
|---|---|---|
| Search documents | Low | Automatic |
| Read Jira issue | Low | Automatic |
| Create Jira ticket | Medium | Configurable |
| Modify Jira issue | Medium | Configurable / approval |
| Send external email | High | Approval |
| Delete enterprise data | Critical | Always approval |
| Financial or security-sensitive action | Critical | Always approval |

Approval records should contain the exact proposed action, parameters, user identity, reason, policy decision, and relevant evidence.

## Reliability

| Failure | Handling |
|---|---|
| LLM timeout | Retry safe calls; use model fallback where appropriate |
| Vector search unavailable | Fallback to keyword search or safe no-answer path |
| Tool API timeout | Query operation status before retrying side effects |
| Duplicate action | Idempotency key |
| Policy service unavailable | Fail closed for protected actions |
| Approval service unavailable | Persist pending task; do not execute protected action |
| Worker crash | Resume from durable workflow state |
| Invalid tool arguments | Schema validation rejects the call |

## Scalability

Keep request-facing services stateless and scale independently:

```text
API Gateway -> Load Balancer -> Orchestrator Workers x N
                           -> Shared State / Queues / Databases
```

A representative production stack could use Kubernetes for service scaling, Redis for low-latency state/caching, PostgreSQL for durable workflow metadata, Kafka for event-driven ingestion and task events, and a horizontally scalable vector/search layer.

## Model Routing

```text
Request -> Task / Complexity Router
             |
             +--> Small model: classification, rewrite, extraction
             +--> Medium model: standard RAG answers
             +--> Large reasoning model: complex planning
```

Track quality and cost by route.

## Observability

### Infrastructure

- CPU / memory / GPU utilization
- request rate
- p50 / p95 / p99 latency
- error rate
- queue depth
- database and search health

### AI System

- retrieval recall / precision
- MRR / NDCG
- reranker quality
- groundedness / faithfulness
- hallucination rate
- tool-selection accuracy
- invalid tool-call rate
- task completion rate
- approval / escalation rate

### Security

- denied retrievals
- authorization failures
- blocked tool calls
- prompt-injection attempts
- policy violations

## Evaluation

Build a golden evaluation set containing questions, expected documents, expected answers, citations, expected tools/actions, and expected safety outcomes.

Evaluate retrieval and generation separately, including Recall@K, MRR, NDCG, correctness, faithfulness, citation support, tool selection, argument accuracy, task completion, and safety.

Include stale documents, conflicting versions, permission boundaries, ambiguous queries, malicious documents, prompt injection, and tool failures in the test set.

## Key Trade-offs

### Agent vs deterministic workflow

Use LLMs for language understanding and flexible planning, but deterministic workflows and policy engines for protected operations.

### Vector-only vs hybrid retrieval

Hybrid retrieval improves robustness across semantic questions and exact identifiers, error codes, names, and document terms.

### One model vs model routing

A single model simplifies operations; routing reduces cost and latency once workload patterns justify the complexity.

### Synchronous vs asynchronous execution

Synchronous execution is suitable for short interactions. Long-running tasks should use durable queues and resumable workflows.

### Single agent vs multi-agent

Start with one constrained orchestrator. Introduce specialist agents only when measurable specialization benefits outweigh coordination complexity.

## Interview Takeaways

1. **The LLM is not the security boundary.**
2. **Authorization must happen before unauthorized content reaches generation.**
3. **RAG knowledge and live transactional data should remain separate.**
4. **Hybrid retrieval is stronger than relying on semantic search alone.**
5. **Every side-effecting action goes through a controlled Tools Gateway.**
6. **High-risk actions require deterministic policy checks and human approval where appropriate.**
7. **Idempotency is essential for safe retries of side effects.**
8. **Long-running agent workflows should be asynchronous and resumable.**
9. **Evaluation must cover retrieval, generation, tools, workflows, security, and business outcomes.**
10. **LLM for reasoning; deterministic infrastructure for control.**
