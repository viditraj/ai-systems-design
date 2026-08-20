# Hospital Agentic Care Assistant — Mock Interview

This walkthrough simulates a senior AI/system-design interview.

## 1. Interviewer: Design an AI assistant for a hospital.

### Candidate

I would first clarify that the assistant has two major user groups: patients and clinicians.

For patients, the system can answer approved knowledge questions, provide administrative help, summarize their own information where authorized, and coordinate follow-ups.

For clinicians, it can summarize longitudinal records and assist with workflow coordination.

For anything that can change clinical state or create a clinically meaningful action, I would put deterministic policy enforcement and human approval outside the LLM.

My key design principle is:

> The LLM proposes and explains; deterministic services and humans authorize.

---

## 2. Interviewer: What is the interesting AI problem here?

### Candidate

I would not build this as a simple chatbot. The interesting problem is a long-running agentic workflow.

A workflow can start today, wait for a clinician tomorrow, receive a lab-result event later, and then continue.

So I need:

- durable workflow state
- event-driven continuation
- idempotent tools
- human-in-the-loop states
- stateless workers

This lets a workflow survive worker failures and user/device changes.

---

## 3. Interviewer: Draw the high-level architecture.

### Candidate

```text
Patient / Clinician
        |
    API Gateway
        |
 Conversation Service
        |
 Intent + Risk Router
        |
   Agent Orchestrator
        |
 +------+-------+------------------+
 |              |                  |
 RAG       Patient Context     Workflow State
 |              |                  |
Vector DB      EHR              Durable DB
                |
                v
           Tools Gateway
                |
          Policy Engine
                |
        +-------+--------+
        |       |        |
   Scheduling Referral Notifications
                |
               Kafka
                |
        Workflow Workers
                |
          Resume Workflow
```

Then I would add the human approval service and audit path.

---

## 4. Interviewer: Why is the LLM not the policy engine?

### Candidate

Because model output is probabilistic. Clinical authorization must be deterministic, versioned, testable, and auditable.

For example, the model might propose a medication-related action. The LLM can generate the proposal, but a policy service checks whether the operation is permitted and whether a clinician must approve it.

That separation also makes prompt injection less dangerous because an attacker cannot use model output to directly bypass authorization.

---

## 5. Interviewer: Where does patient data live?

### Candidate

I would separate four concerns:

```text
Medical knowledge -> Vector DB
Current patient truth -> EHR / clinical systems
Workflow state -> Durable workflow DB
Audit events -> Append-only audit store
```

I would not use the vector database as the authoritative patient database.

For every request, a Patient Context Service retrieves only the minimum relevant information.

---

## 6. Interviewer: Why do you need Kafka?

### Candidate

Not every request needs Kafka.

For a synchronous read such as current appointment status, direct service-to-service communication is fine.

Kafka becomes valuable for long-running workflows and fan-out events such as:

```text
LabResultReceived
ApprovalGranted
AppointmentScheduled
NotificationRequested
WorkflowStepCompleted
```

Kafka decouples producers from worker pools and lets us scale each consumer group independently.

---

## 7. Interviewer: What happens if the agent worker dies midway?

### Candidate

Workers are stateless.

The workflow state is persisted externally.

```text
Worker A
  -> load state
  -> execute step
  -> persist state
  -> publish event
  -> crash

Worker B
  -> consume event
  -> load state
  -> continue
```

There is no requirement for sticky sessions.

---

## 8. Interviewer: What if the workflow needs clinician approval?

### Candidate

I model approval as an explicit workflow state.

```text
RUNNING
   |
   v
WAITING_FOR_CLINICIAN
   |
   +------> REJECTED
   |
   +------> APPROVED
               |
               v
             EXECUTE
```

The browser does not need to remain open.

The clinician can approve from another device. When `ApprovalGranted` arrives, the workflow resumes.

That is the important difference between true human-in-the-loop orchestration and simply telling the user to contact a doctor.

---

## 9. Interviewer: How do you avoid duplicate side effects?

### Candidate

Because Kafka and networks can cause retries, I assume a side-effecting tool may execute more than once unless I design for idempotency.

I use:

```text
workflow_id + step_id + action_type
```

as an idempotency key.

The side-effect service stores the key and result. A duplicate request returns the original result instead of performing the action again.

---

## 10. Interviewer: How do you scale to millions of users?

### Candidate

I separate scaling dimensions.

```text
API tier             -> horizontal replicas
Agent workers        -> horizontal replicas
RAG service          -> horizontal replicas
Model gateway       -> GPU/model replicas
Workflow consumers   -> partitions + consumer groups
Notification workers -> queue-based scaling
Audit workers        -> independent consumers
```

I would not scale the entire system as one monolith.

---

## 11. Interviewer: How would Kafka be partitioned?

### Candidate

For workflow ordering, partition by `workflow_id`.

That means:

```text
WF-1 -> partition 3
WF-2 -> partition 7
WF-3 -> partition 1
```

A single workflow preserves event order, but thousands of independent workflows can execute concurrently.

If the domain requires patient-wide ordering, I would consider `patient_id`, but I would be careful about hot partitions.

---

## 12. Interviewer: What if traffic is 10x larger than expected?

### Candidate

I would scale each bottleneck independently.

```text
                Global Load Balancer
                       |
            +----------+----------+
            |                     |
         Region A              Region B
            |                     |
       Gateway xN             Gateway xN
            |                     |
      Agent Workers xN       Agent Workers xN
            |                     |
         Kafka xN               Kafka xN
            |                     |
     Workflow Workers      Workflow Workers
```

The important point is that API QPS and workflow-event throughput are separate capacity dimensions.

A system can have moderate HTTP traffic but very high event volume because one interaction fans out into many asynchronous steps.

---

## 13. Interviewer: How do you handle backpressure?

### Candidate

I would use queueing plus priority-aware admission control.

For example:

```text
Ingress
  -> Admission Control
  -> Kafka
  -> Priority Consumers
```

High-priority clinical review events should not be starved by a burst of low-priority notification or analytics work.

I would also enforce concurrency limits on expensive model calls.

---

## 14. Interviewer: How do you handle model provider failure?

### Candidate

I put an abstraction behind a Model Gateway.

```text
Agent
  -> Model Gateway
      -> Primary model
      -> Fallback model
```

Routing depends on task type.

For simple classification or summarization, a smaller model may be sufficient. For complex reasoning, the system can route to the stronger model.

If all models fail, the system should not fabricate a clinical answer.

---

## 15. Interviewer: How do you design RAG for a medical system?

### Candidate

I would use metadata-aware hybrid retrieval.

```text
Query
  -> metadata filtering
  -> lexical + vector retrieval
  -> reranking
  -> top-K evidence
  -> generation
```

Metadata should include:

- approval status
- effective date
- specialty
- region
- intended audience
- document version

The system should prevent expired or unapproved policies from being selected.

---

## 16. Interviewer: What do you monitor?

### Candidate

I would monitor three layers.

### Platform

- p95/p99 latency
- error rate
- CPU/GPU utilization
- DB pool usage
- Kafka lag
- queue depth

### Agent

- token usage
- model latency
- tool-call count
- workflow duration
- retry count
- completion rate

### Safety

- risk-classification false negatives
- policy blocks
- human override rate
- unauthorized tool attempts
- duplicate side effects prevented
- stale knowledge retrieval

I would connect them using distributed tracing so a single workflow can be followed across model calls, Kafka events, tools, and databases.

---

## 17. Interviewer: What is your most important safety metric?

### Candidate

For clinical workflows, I would not make response fluency the primary safety metric.

I would focus on:

> **High-risk false-negative escalation rate.**

For workflows classified as requiring mandatory approval, the goal is to have zero unauthorized clinical actions.

---

## 18. Interviewer: What if the EHR is down?

### Candidate

The system must distinguish generic knowledge from patient-specific information.

If the EHR is unavailable, I can still answer a generic policy question from approved knowledge if appropriate.

But I should not answer:

```text
"What medications am I currently taking?"
```

from stale model context.

For patient-specific data, fail safely and communicate that the authoritative system is unavailable.

---

## 19. Interviewer: Why not put the entire medical chart into the prompt?

### Candidate

That creates three problems:

1. token cost
2. latency
3. unnecessary data exposure

Instead, a Patient Context Service retrieves only the relevant slices.

```text
Question
  -> context planner
  -> relevant encounters/labs/medications
  -> minimum necessary context
  -> model
```

This is both an efficiency and privacy design.

---

## 20. Interviewer: How would you estimate capacity?

### Candidate

I would state assumptions instead of pretending to know proprietary traffic.

For example:

```text
5M interactions/day
5 workflow events per interaction
```

Then:

```text
5M / 86,400 ≈ 58 requests/sec
```

At 10x peak:

```text
≈ 580 requests/sec
```

For events:

```text
5M * 5 / 86,400 ≈ 289 events/sec average
```

At 10x peak:

```text
≈ 2,900 events/sec
```

Then I benchmark agent-worker throughput, model throughput, database QPS, and Kafka partition capacity.

---

## 21. Interviewer: What is the biggest failure mode in your design?

### Candidate

The biggest architectural mistake would be letting the LLM become the system of record or authorization layer.

A second mistake would be keeping workflow state only in process memory or only in the chat context.

A third would be using synchronous calls for every step, which makes a long-running workflow fragile and expensive.

---

## 22. Interviewer: Summarize the architecture in 30 seconds.

### Candidate

I would build a stateless conversational and agent tier around durable workflow state and an event bus. The LLM handles language understanding, summarization, retrieval, and action proposals, while a Tools Gateway and deterministic Clinical Policy Engine enforce authorization and safety. Kafka carries long-running workflow events so workers can scale horizontally and resume after failures. Patient truth remains in the EHR, knowledge remains in RAG, workflow state remains durable, and every sensitive action is auditable and idempotent.

---

# What This System Teaches You

This interview problem is intentionally different from a standard RAG chatbot.

The concepts to practice are:

```text
1. Durable agent workflows
2. Event-driven architecture
3. Kafka partitioning
4. Human-in-the-loop state machines
5. Idempotent tool execution
6. Policy / authorization boundaries
7. Stateless workflow workers
8. Backpressure and priority queues
9. Model gateway + fallback routing
10. High-stakes AI evaluation
```

The key mental model is:

> **An enterprise agent is a distributed workflow system with an LLM inside it, not an LLM with a few API calls around it.**
