# Hospital Agentic Care Assistant

> Production-oriented AI assistant for hospitals that helps clinicians and patients navigate care workflows, retrieves grounded medical knowledge, summarizes longitudinal records, coordinates follow-up tasks, and escalates high-risk situations to human clinicians.

## Problem Statement

Design an agentic AI assistant for a large hospital network that can:

- Answer patient questions using approved medical knowledge and hospital policies.
- Summarize a patient's longitudinal record across encounters, labs, medications, imaging, and notes.
- Coordinate non-diagnostic operational tasks such as appointment scheduling, reminders, referrals, and care-plan follow-ups.
- Detect potentially high-risk situations and route them to an appropriate clinical workflow instead of autonomously making a diagnosis or treatment decision.
- Support clinicians with evidence-backed summaries and suggested next actions.
- Maintain a complete audit trail of model reasoning requests, retrieved evidence, tool calls, approvals, and human overrides.
- Continue a workflow safely after retries, worker failures, or a handoff to another clinician/service.

## The New Concept This System Teaches

### Event-Driven Agentic Workflows + Durable Human-in-the-Loop Gates

Unlike a simple chatbot, the assistant is not a single request/response chain. A patient-care workflow may last minutes, hours, or days and can be interrupted by external events.

Examples:

```text
Patient asks for follow-up
    -> retrieve care plan
    -> detect due lab
    -> create lab-order suggestion
    -> wait for clinician approval
    -> appointment scheduled
    -> lab result arrives later
    -> event resumes workflow
    -> clinician review if abnormal
    -> patient notification
```

The key design idea is to persist workflow state externally and resume from events rather than keeping the whole workflow inside one HTTP request or one LLM context window.

This teaches:

1. Durable execution for long-running agents.
2. Event-driven orchestration with Kafka/events.
3. Human-in-the-loop approval as a first-class workflow state.
4. Clinical safety boundaries around an LLM.
5. Separation of knowledge, patient state, and transactional systems.
6. Idempotent tool execution and exactly-once business effects.
7. Horizontal scaling of stateless agent workers.

## Core Safety Principle

**The LLM can interpret, summarize, retrieve, and propose actions. It cannot independently authorize clinical decisions.**

Clinical policy, authorization, medication/order constraints, and escalation rules are enforced by deterministic services and human approval workflows.

## Architecture

See [architecture.md](./architecture.md) for:

- logical architecture
- request flows
- event-driven workflow
- clinical safety boundary
- state management
- human approval workflow
- failure handling
- scaling architecture
- observability
- capacity estimates
- trade-offs

## Main Components

| Component | Responsibility |
|---|---|
| API Gateway | Authentication, authorization, rate limiting, WAF, routing |
| Conversation Service | User-facing request/response lifecycle |
| Agent Orchestrator | Runs bounded workflows and resumes durable workflows |
| Intent & Risk Router | Classifies request type and risk level |
| RAG Service | Retrieves approved medical knowledge and hospital policies |
| Patient Context Service | Builds an authoritative, privacy-filtered patient context |
| Clinical Policy Engine | Deterministic safety and workflow rules |
| Tools Gateway | Security boundary for tool access |
| Scheduling Tool | Appointment lookup/reservation |
| Referral Tool | Referral workflow operations |
| Notification Tool | Patient notifications/reminders |
| Clinician Approval Service | Human approval/rejection workflow |
| Workflow State Store | Durable checkpoints for long-running workflows |
| Event Bus | Kafka-based asynchronous domain events |
| Audit Service | Immutable audit trail |
| Model Gateway | Centralized LLM routing, quotas, fallback, observability |

## Data Stores

Keep different classes of data in their proper source of truth:

```text
Approved medical knowledge / SOPs -> Vector DB / Search Index
Patient longitudinal record        -> EHR / clinical databases
Active workflow state              -> Durable workflow DB
Hot ephemeral state                -> Redis
Events                             -> Kafka
Audit trail                        -> Append-only audit store
Metrics                            -> Prometheus
Dashboards                         -> Grafana
```

The vector database is **not** the source of truth for current patient information.

## Request Flows

### 1. Patient Knowledge Question

```text
Patient
  -> API Gateway
  -> Conversation Service
  -> Intent/Risk Router
  -> RAG Service
  -> Response Composer
  -> Conversation Service
  -> Patient
```

Example:

`What does fasting before my blood test mean?`

The answer is grounded in approved patient education and hospital policy content.

### 2. Longitudinal Record Summary

```text
Clinician
  -> API Gateway
  -> Conversation Service
  -> Intent/Risk Router
  -> Patient Context Service
  -> EHR / Labs / Medications / Imaging
  -> Summary Agent
  -> Evidence Validator
  -> Clinician
```

The assistant generates a structured summary with source references instead of inventing facts.

### 3. Follow-Up Workflow

```text
Patient
  -> "I need my follow-up appointment"
  -> Orchestrator
  -> Patient Context
  -> Scheduling Tool
  -> Workflow State Store
  -> AppointmentScheduled Event
  -> Notification Service
  -> Patient
```

### 4. Human Approval Workflow

For actions with clinical impact:

```text
LLM proposal
    |
    v
Clinical Policy Engine
    |
    +---- SAFE / NO CLINICAL IMPACT ----> Tool execution
    |
    +---- REQUIRES APPROVAL -----------> Clinician Approval Queue
                                             |
                               +-------------+-------------+
                               |                           |
                            APPROVE                      REJECT
                               |                           |
                               v                           v
                         Execute tool                 End workflow
```

The approval decision is persisted independently from the LLM.

## Durable Workflow State

A workflow should be restartable from any worker.

Example:

```json
{
  "workflow_id": "WF-123",
  "patient_id": "P-456",
  "type": "follow_up",
  "status": "WAITING_FOR_CLINICIAN",
  "step": "review_lab_followup",
  "version": 7,
  "pending_action": "schedule_followup",
  "approval_id": "APR-991",
  "created_at": "2026-08-20T10:00:00Z",
  "updated_at": "2026-08-20T10:10:00Z"
}
```

Workers are stateless. They load the latest workflow state, execute one bounded transition, persist the checkpoint, and emit the resulting event.

## Event-Driven Workflow

Example event sequence:

```text
LabResultReceived
      |
      v
Workflow Resumer
      |
      v
Risk Evaluation
      |
      +---- NORMAL ----> Complete
      |
      +---- NEEDS_REVIEW
                  |
                  v
          ClinicianApprovalRequested
                  |
                  v
             Clinician UI
                  |
          ApprovalGranted event
                  |
                  v
             Execute Tool
                  |
                  v
           WorkflowCompleted
```

Kafka provides durable event transport and decouples the workflow from the lifespan of an individual API/worker process.

## Clinical Safety Boundary

The assistant should have explicit boundaries:

### Allowed without clinician approval

- Explain hospital policies.
- Summarize existing records.
- Retrieve current appointment information.
- Send administrative reminders.
- Draft questions for a clinician.

### Approval required

- Changes to medication-related workflows.
- Clinical orders.
- Referrals with clinical implications.
- Actions triggered by high-risk findings.
- Any operation explicitly classified as clinician-only.

### Immediate escalation

For potentially emergent symptoms or configured high-risk signals, the assistant should stop autonomous workflow progression and route to the hospital's emergency/clinical escalation pathway.

The model must not be allowed to override this policy by generating persuasive text.

## Tool Security

All tools go through a Tools Gateway:

```text
Agent
  -> Tools Gateway
  -> Authenticate service + user
  -> Authorize operation
  -> Validate arguments
  -> Clinical Policy Engine
  -> Idempotency Check
  -> Backend Service
```

The LLM never receives direct database credentials or unrestricted API access.

## Idempotency

A tool request may be retried because of a worker timeout or Kafka redelivery.

For side-effecting actions use an idempotency key such as:

```text
workflow_id + step_id + action_type
```

The Tool Gateway or backend service stores the outcome and returns the same result on a duplicate request.

This prevents duplicate appointment creation, duplicate notifications, or repeated administrative actions.

## RAG Design

Knowledge sources may include:

- hospital policies
- approved patient education
- clinical SOPs
- medication reference content approved for the use case
- operational FAQs

Metadata should include:

```text
source_id
version
effective_from
effective_to
specialty
region
audience
approval_status
```

Retrieval should filter by audience, specialty, region, and currently effective version before generation.

## Patient Context vs RAG

Do not put the entire patient chart into the vector store as a substitute for the EHR.

```text
Medical knowledge -> RAG
Patient facts       -> EHR / clinical systems
Workflow state      -> Workflow DB
Conversation state  -> Conversation store
```

The Patient Context Service assembles only the minimum relevant data for the current workflow.

## Privacy and Authorization

Use defense in depth:

```text
Identity
  -> Role / purpose-of-use check
  -> Patient relationship check
  -> Minimum necessary data selection
  -> Tool authorization
  -> Audit
```

Examples:

- Patient can access their own allowed information.
- A clinician can access patients within their authorized scope.
- An agent worker cannot bypass service authorization merely because an LLM requested the data.

## Response Path

The Conversation Service owns the final user response:

```text
Workflow result
      |
      v
Evidence / Safety Check
      |
      +---- deterministic outcome -> safe template
      |
      +---- explanation needed ----> Response LLM
      |
      v
Conversation Service
      |
      v
API Gateway
      |
      v
User
```

Not every result needs an LLM-generated response.

## Failure Handling

| Failure | Handling |
|---|---|
| LLM timeout | Retry bounded step; use fallback model for safe informational requests |
| RAG unavailable | Use cached approved content or escalate; do not fabricate medical facts |
| EHR unavailable | Do not answer patient-specific questions from stale model context |
| Kafka redelivery | Idempotent workflow transition |
| Worker crash | Another worker loads persisted workflow state and resumes |
| Approval service unavailable | Persist pending approval; do not execute the action |
| Tool timeout | Check operation status before retrying side effects |
| Duplicate event | Event/workflow transition idempotency |
| Policy engine unavailable | Fail closed for clinically sensitive actions |
| Stale policy document | Retrieval filter blocks expired content |

## Scaling Architecture

The key scaling strategy is **stateless workers + durable state + partitioned events**.

```text
                         ┌──────────────────────┐
                         │      Clients         │
                         │ Patient / Clinician  │
                         └──────────┬───────────┘
                                    │
                              API Gateway
                                    │
                           Load Balancer / WAF
                                    │
                  ┌─────────────────┴─────────────────┐
                  │                                   │
          Conversation Pods                    Async Ingest
                x N                                  |
                  │                                   |
                  v                                   v
          Agent Worker Pool                      Kafka
             x N                                    │
                  │                 ┌───────────────┼────────────────┐
                  │                 │               │                │
                  v                 v               v                v
          Model Gateway      Workflow Workers   Notification     Audit Workers
                  │                x N              x N              x N
          ┌───────┴────────┐        │
          │                │        v
      Primary LLM      Fallback     Workflow DB
                                   /
                                  /
                           Redis / Cache
                                   |
                      ┌────────────┼─────────────┐
                      │            │             │
                      v            v             v
                    EHR        Tools GW       RAG Service
                      │            │             │
                      │        Policy Engine    Vector DB
                      │            │
                      └────────────┼─────────────┘
                                   v
                              Audit Store
```

### Independent scaling dimensions

Scale independently:

- API / conversation pods by request rate.
- Agent workers by CPU-bound orchestration throughput.
- LLM inference/model gateway by token throughput.
- RAG service by retrieval QPS.
- Workflow workers by event lag.
- Notification workers by queue depth.
- Audit workers by event volume.

### Kafka partitioning

Partition workflow events by `patient_id` or `workflow_id` to preserve ordering for a workflow while allowing many partitions to be processed concurrently.

Avoid a single global partition because it becomes a throughput bottleneck.

### Backpressure

When model or downstream systems are saturated:

```text
Ingress
  -> Queue
  -> Admission control
  -> Priority scheduling
  -> Worker pools
```

Patient safety / emergency escalation events should have higher priority than low-value analytics or batch tasks.

## Statelessness

A worker can die after any step without losing the workflow:

```text
Worker A
  -> load state
  -> execute step
  -> commit state
  -> publish event
  -> crash

Worker B
  -> consume event
  -> load state
  -> continue workflow
```

This allows users to switch devices and allows any worker instance to continue the conversation/workflow.

## Multi-Region Scaling

For a large hospital network:

```text
                    Global DNS / Traffic Manager
                              |
              ┌───────────────┴───────────────┐
              |                               |
          Region A                        Region B
              |                               |
      Gateway + Workers               Gateway + Workers
              |                               |
          Regional Kafka                  Regional Kafka
              |                               |
         Regional stores                 Regional stores
```

Prefer data residency and locality for patient data. Cross-region replication should follow healthcare, contractual, and organizational requirements rather than treating the workflow database as globally writable by default.

## Observability

### Infrastructure

- CPU / memory
- pod restarts
- request rate
- p50/p95/p99 latency
- Kafka consumer lag
- queue depth
- DB connection pool utilization

### Agent

- intent accuracy
- risk classification accuracy
- tool-call success rate
- workflow completion rate
- step retries
- human approval rate
- escalation rate
- token usage
- model latency

### RAG

- retrieval latency
- Recall@K
- MRR / ranking quality
- groundedness
- stale-document retrieval rate
- citation coverage

### Safety

- policy blocks
- unauthorized tool attempts
- high-risk escalation rate
- clinician override rate
- duplicate side-effect prevention

### Reliability

- workflow recovery success
- event redelivery rate
- idempotency hits
- stuck workflow count
- dead-letter queue size

Use Prometheus for metrics and Grafana for dashboards. Distributed tracing should connect a user request to workflow ID, Kafka event, tool call, model call, and final response.

## Capacity Estimation

Assume:

```text
5M patient/clinician interactions/day
10 steps/workflow on average
5 messages or events per interaction
```

Average ingress:

```text
5,000,000 / 86,400 ≈ 58 requests/sec
```

At 10x peak:

```text
≈ 580 requests/sec
```

The event volume can be substantially higher because one interaction may fan out into multiple workflow events.

If the average workflow generates 10 events:

```text
5M * 10 / 86,400 ≈ 579 events/sec average
10x peak ≈ 5,800 events/sec
```

Design Kafka and worker capacity around event throughput rather than only HTTP QPS.

## Evaluation

Evaluate multiple layers independently:

| Layer | Metrics |
|---|---|
| Intent | precision / recall / confusion matrix |
| Risk | false-negative escalation rate |
| RAG | Recall@K, MRR, citation correctness |
| Tool use | correct tool + correct arguments |
| Workflow | state-transition accuracy, recovery success |
| Safety | unauthorized clinical action rate = 0 |
| UX | task completion, handoff rate, latency |
| Reliability | duplicate side effects, stuck workflows, event lag |

For high-risk workflows, reducing false negatives is usually more important than maximizing model fluency.

## Key Trade-offs

### Durable workflow engine vs custom state machine

A workflow engine can simplify retries, timers, and recovery. A custom state machine offers more control but requires more reliability engineering.

### Kafka vs direct synchronous service calls

Use synchronous calls for low-latency reads and validation. Use events for long-running, retryable, fan-out workflows.

### Single agent vs multiple agents

Start with a single orchestrator plus bounded specialist tools. Introduce specialist agents only when there is a clear latency, quality, isolation, or ownership benefit.

### Redis vs durable workflow DB

Redis is ideal for hot ephemeral state; it should not be the only source of truth for a workflow that may resume days later.

### LLM autonomy vs policy controls

More autonomy can reduce manual work but increases safety and reliability risk. Keep clinical authorization outside the LLM.

## Interview Takeaways

1. **Long-running agents need durable state, not just an LLM context window.**
2. **Events let workflows resume after worker failures and external events.**
3. **Human approval is a state transition, not a chat message.**
4. **LLMs are not security or clinical authorization boundaries.**
5. **Separate RAG knowledge, patient truth, workflow state, and audit data.**
6. **Tool calls must be authorized and idempotent.**
7. **Kafka partitions allow horizontal event processing while preserving workflow ordering.**
8. **Stateless workers enable device mobility and failure recovery.**
9. **Safety metrics must measure dangerous false negatives, not only response quality.**
10. **Scale event throughput and model throughput separately from API throughput.**
