# Hospital Agentic Care Assistant — Architecture Deep Dive

## 1. High-Level Architecture

```text
Patient / Clinician
        |
        v
  API Gateway / WAF
        |
        v
 Conversation Service
        |
        v
 Intent + Risk Router
        |
        +-------------------+
        |                   |
        v                   v
 Knowledge Path       Agent Workflow Path
        |                   |
   RAG Service         Agent Orchestrator
        |                   |
   Vector DB                +-------------------+
                            |                   |
                            v                   v
                     Patient Context       Workflow State
                            |                   |
                            v                   v
                           EHR               Durable DB
                            |
                            +--------+----------+
                                     |
                               Tools Gateway
                                     |
                              Policy Engine
                                     |
                       +-------------+--------------+
                       |             |              |
                       v             v              v
                  Scheduling     Referral      Notification
                       |             |              |
                       +-------------+--------------+
                                     |
                                     v
                                  Kafka
                           +---------+---------+
                           |                   |
                           v                   v
                   Workflow Workers      Audit Workers
                           |
                           v
                     Resume / Continue
```

The critical architectural property is that the user-facing API is not the owner of a multi-hour workflow. The API creates or resumes a durable workflow and returns the current state/result.

## 2. Request Classification

The router separates four classes:

```text
GENERAL_KNOWLEDGE
PATIENT_CONTEXT
ADMINISTRATIVE_ACTION
CLINICAL_RISK
```

The last class is especially important because an apparently conversational prompt may carry clinical significance.

Example:

```text
"My chest feels tight and I am short of breath."
```

The system must not treat this as a normal RAG question. Risk policy should trigger the hospital's configured escalation path.

## 3. Agent Contract

The orchestrator operates on structured state rather than arbitrary prompt text.

```json
{
  "workflow_id": "WF-123",
  "request_id": "REQ-777",
  "patient_id": "P-456",
  "user_role": "clinician",
  "intent": "follow_up",
  "risk_level": "low",
  "current_step": "schedule",
  "status": "RUNNING",
  "state_version": 7,
  "pending_approval": null
}
```

Each workflow step receives typed input and produces a typed transition.

## 4. Durable State Machine

Think of the workflow as a finite state machine:

```text
                    +----------------+
                    |     CREATED    |
                    +-------+--------+
                            |
                            v
                    +----------------+
                    |     PLANNING   |
                    +-------+--------+
                            |
                            v
                    +----------------+
                    | TOOL_VALIDATION|
                    +---+---------+--+
                        |         |
                 safe   |         | approval needed
                        |         v
                        |   +-------------+
                        |   |   WAITING   |
                        |   | FOR HUMAN   |
                        |   +------+------+ 
                        |          |
                        |     approved/rejected
                        |          |
                        +----------+
                                   |
                                   v
                            +-------------+
                            |   EXECUTE   |
                            +------+------+ 
                                   |
                                   v
                            +-------------+
                            |  COMPLETED  |
                            +-------------+
```

Persist the transition atomically before publishing a continuation event where possible.

## 5. Why Kafka?

Kafka is not being used because every AI request needs a queue. It is used where the workflow benefits from durable asynchronous events:

- lab/result events
- appointment updates
- clinician approvals
- notification requests
- workflow continuation
- analytics/audit fan-out

A synchronous read such as `GET current appointment` can stay synchronous.

## 6. Event Envelope

Use a common event envelope:

```json
{
  "event_id": "EVT-123",
  "event_type": "ApprovalGranted",
  "event_version": 2,
  "occurred_at": "2026-08-20T12:10:00Z",
  "workflow_id": "WF-123",
  "patient_id": "P-456",
  "producer": "clinician-approval-service",
  "trace_id": "TRACE-999",
  "payload": {}
}
```

`event_id` enables deduplication. `event_version` enables schema evolution.

## 7. Exactly-Once Business Effects

Kafka itself should not be treated as magic exactly-once business execution.

For a side effect:

```text
consume event
   |
   v
check idempotency key
   |
   +---- already processed -> return stored result
   |
   +---- new -> execute operation
                    |
                    v
             store operation result
                    |
                    v
             acknowledge event
```

This provides effectively-once behavior for external side effects.

## 8. Human-in-the-Loop

A human approval request is durable state:

```text
workflow.status = WAITING_FOR_CLINICIAN
approval_id = APR-991
```

The worker can stop. The user can close the browser. A clinician can approve from another device. The approval event later resumes the workflow.

This is different from:

```text
LLM: "Please ask your doctor."
```

The system actually pauses and waits for a recorded decision.

## 9. LLM Boundary

```text
                    LLM
                     |
            propose next action
                     |
                     v
               Structured Plan
                     |
                     v
             Policy / Validator
                     |
          +----------+----------+
          |                     |
       allowed               blocked
          |                     |
          v                     v
       Tool GW             Audit + Explain
```

The LLM cannot directly:

- write to the EHR
- issue a prescription
- modify a medication record
- bypass approval
- bypass authorization

## 10. Patient Context Assembly

Do not send the entire chart to the model for every prompt.

```text
User request
    |
    v
Context Planner
    |
    +--> encounter history
    +--> active medications
    +--> recent labs
    +--> relevant imaging
    +--> care plan
    |
    v
Minimum Necessary Context
    |
    v
Model
```

This reduces token cost, latency, and unnecessary exposure of patient data.

## 11. Retrieval Architecture

```text
Question
   |
   v
Query Rewrite / Intent
   |
   v
Metadata Filter
   |
   +--> specialty
   +--> region
   +--> audience
   +--> effective date
   +--> approval status
   |
   v
Hybrid Retrieval
   |
   +--> lexical search
   +--> vector search
   |
   v
Reranker
   |
   v
Top-K evidence
   |
   v
Response Generator
```

For high-stakes content, retrieval should be conservative: approved and currently effective evidence should be preferred over semantically similar but stale content.

## 12. Scaling the Model Tier

The Model Gateway decouples the application from model providers:

```text
Agent Workers
      |
      v
 Model Gateway
   /    |     \
  /     |      \
Fast   Primary  Fallback
Small  Reasoning  Model
```

Routing policy can choose a smaller model for classification/summarization and a stronger model for complex reasoning.

The application sees one stable interface.

## 13. Scaling Worker Pools

Different workflows should use different queues/pools:

```text
Kafka Topics
  |
  +--> patient-interactions
  +--> clinical-review
  +--> notifications
  +--> workflow-resume
  +--> audit
          |
          v
Consumer Groups
  |
  +--> Interaction Workers x N
  +--> Review Workers x N
  +--> Notification Workers x N
  +--> Resume Workers x N
  +--> Audit Workers x N
```

This prevents a burst of notifications from starving clinical workflow processing.

## 14. Partitioning Strategy

### Workflow events

Partition by `workflow_id`.

Benefit: all events for one workflow remain ordered while independent workflows execute concurrently.

### Patient-scoped events

Partition by `patient_id` when patient-level ordering is required by the domain.

### Avoid

Partitioning everything by one hospital/tenant ID can create a hot partition for a large customer.

## 15. Backpressure

Suppose the LLM provider slows down:

```text
API
 |
v
Admission Controller
 |
v
Kafka / queue
 |
v
Priority Scheduler
 |
+--> high priority clinical review
+--> normal interactions
+--> low priority batch jobs
```

The system should degrade by delaying lower-priority work, not by making unsafe clinical guesses.

## 16. Resilience Pattern

A worker crash is normal:

```text
Worker A
  load WF-123
  perform step 4
  persist step 4 result
  publish Step4Completed
  CRASH

Worker B
  consume Step4Completed
  load WF-123
  perform step 5
```

No sticky server session is required.

## 17. API Idempotency

For client retries:

```http
POST /workflows
Idempotency-Key: client-abc-123
```

The Conversation Service stores the request/result mapping. A network retry should not create two workflows.

## 18. Observability Model

Every operation receives:

```text
request_id
trace_id
conversation_id
workflow_id
event_id
user_id/patient reference (appropriately protected)
model_call_id
tool_call_id
```

Trace example:

```text
HTTP request
  -> intent model
  -> RAG retrieval
  -> workflow step
  -> Kafka publish
  -> consumer
  -> tool call
  -> policy decision
  -> response generation
```

This makes agentic systems debuggable like distributed systems rather than opaque chatbots.

## 19. Metrics

### Golden signals

- latency
- traffic
- errors
- saturation

### Kafka

- consumer lag
- partition skew
- rebalance count
- DLQ messages

### Agent

- model latency
- tokens/input-output
- tool calls per workflow
- workflow completion time
- retry count

### RAG

- Recall@K
- MRR
- no-answer rate
- citation rate
- stale retrieval rate

### Safety

- risk false negatives
- policy blocks
- clinician overrides
- unauthorized tool attempts
- prevented duplicate side effects

## 20. Failure Matrix

| Dependency | Failure | Response |
|---|---|---|
| LLM | timeout | bounded retry/fallback |
| LLM | unsafe output | validation/policy block |
| RAG | unavailable | cached approved knowledge or safe escalation |
| EHR | unavailable | no patient-specific guessing |
| Kafka | broker issue | retry producer/consumer; preserve state |
| DB | unavailable | pause state transition; retry |
| Tool | timeout | operation-status lookup before retry |
| Human approval | unavailable | keep workflow waiting |
| Notification | unavailable | queue for retry |

## 21. Security Zones

```text
Internet
  |
 WAF
  |
API Zone
  |
Agent Zone
  |
Tools / Policy Zone
  |
Clinical Systems Zone
```

The Agent Zone should have strictly controlled outbound access. Clinical systems should never be directly reachable from the public API tier.

## 22. Scale-Out Architecture at 10x Traffic

At 10x peak:

```text
                  Global Load Balancer
                           |
              +------------+------------+
              |                         |
          Region A                  Region B
              |                         |
        Gateway x 10                Gateway x 10
              |                         |
      Conversation x 20        Conversation x 20
              |                         |
       Agent Workers x 50      Agent Workers x 50
              |                         |
          Kafka x 30             Kafka x 30
              |                         |
       Workflow x 30            Workflow x 30
              |                         |
       Regional stores          Regional stores
```

Exact counts depend on benchmarked throughput; the important concept is independent horizontal scaling at each bottleneck.

## 23. Interview Deep-Dive Questions

### Why not keep everything in Redis?

Because long-running workflows need durable state, recovery, auditability, and safe resume after a cache loss or worker failure.

### Why Kafka if you already have a workflow database?

The DB stores authoritative workflow state. Kafka transports events between independently scalable services and supports replay/fan-out.

### Why not let LangGraph own the state?

A graph library can model orchestration, but production durability should have an explicit persistent state contract. The worker should be reconstructable from durable workflow state.

### Why not use RAG for patient records?

Current patient facts should come from authoritative clinical systems. RAG is best for relatively stable knowledge, policies, and approved content.

### How do you resume after a 2-day wait for a clinician?

Persist `WAITING_FOR_CLINICIAN` plus an approval ID. A later `ApprovalGranted` event starts the next transition.

### How do you prevent duplicate appointment creation?

Use an idempotency key derived from workflow and step identity and persist the operation result.

### How do you prevent an LLM from bypassing clinical controls?

The LLM only emits structured requests. The Tools Gateway and deterministic Policy Engine authorize every side effect.

## 24. Design Summary

The scalable design is:

```text
Stateless API + Stateless Agent Workers
                |
                v
       Durable Workflow State
                |
                v
               Kafka
                |
                v
      Independent Worker Pools
                |
        +-------+-------+
        |               |
   Clinical Tools    Notifications
        |
   Policy / Auth
        |
       EHR
```

This architecture turns an AI assistant into a reliable distributed workflow system rather than a chat endpoint with an LLM attached.
