# Netflix Customer Support Agent

> Production-grade AI customer support agent for Netflix that answers questions, handles account-specific queries, processes eligible refunds automatically, escalates high-value refunds to human agents, and supports explicit human handoff.

## Problem Statement

Build a conversational AI agent that can:

- Answer general Netflix policy and product questions using trusted SOPs.
- Answer user-specific questions using authoritative account, billing, and subscription services.
- Process refunds automatically up to a configurable policy threshold.
- Create a ticket for human review when a refund exceeds the automatic approval threshold or requires manual judgment.
- Immediately route the conversation to a human when the customer asks to speak with a human.
- Preserve conversation context during human handoff.
- Provide auditable, reliable, and idempotent financial operations.

## Core Design Principle

**The LLM can reason and request actions, but it must never be the final authority for financial authorization.**

Refund authorization is enforced by a deterministic, versioned Policy Engine outside the LLM.

## High-Level Architecture

![Netflix Customer Support Agent Architecture](./architecture.png)

## Main Components

| Component | Responsibility |
|---|---|
| API Gateway | Authentication, rate limiting, routing, WAF boundary |
| Conversation Service / Orchestrator | Owns conversation state and coordinates the agent workflow |
| Intent Detection | Classifies human handoff, general query, refund, and user-specific requests |
| RAG DB | Stores approved Netflix SOPs, policies, and product knowledge |
| Tools Gateway | Security boundary for all backend tool calls |
| User Details Tool | Retrieves authenticated customer/account information |
| Billing Tool | Retrieves billing and transaction information |
| Refund Tool | Requests a refund after policy validation |
| Policy Engine | Deterministically evaluates refund eligibility and approval limits |
| Refund Queue | Asynchronous, durable refund processing path |
| Ticket API | Creates human-review cases |
| Human Handoff Service | Transfers an active conversation to the support queue |
| Response Composer | Converts workflow results into the final customer response |

## Request Flows

### 1. General Question

```text
User
  -> API Gateway
  -> Conversation Service
  -> Intent Detection
  -> RAG Retrieval
  -> Response Composer
  -> User
```

Example: `How do I change my Netflix password?`

The answer should be grounded in the currently active Netflix SOP/product documentation.

### 2. User-Specific Question

```text
User
  -> Intent Detection
  -> Tools Gateway
  -> User Details / Billing Tool
  -> Authoritative Backend Service
  -> Response Composer
  -> User
```

Example: `When does my subscription renew?`

Customer-specific data should come from authoritative services, **not from the vector database**.

### 3. Automatic Refund

```text
User
  -> Refund Intent
  -> Billing Tool
  -> Refund Tool
  -> Policy Engine
  -> AUTO_APPROVE
  -> Refund Request Queue
  -> Refund Processor
  -> Refund Service
  -> RefundCompleted Event
  -> Conversation / Notification Service
  -> User
```

The threshold is configuration/policy data, not an LLM prompt constant.

### 4. Refund Requiring Human Review

```text
User
  -> Refund Intent
  -> Billing Tool
  -> Policy Engine
  -> HUMAN_REVIEW
  -> Ticket API
  -> Human Agent Queue
  -> Response Composer
  -> User
```

### 5. Explicit Human Handoff

```text
User
  -> Intent Detection
  -> HUMAN_HANDOFF
  -> Human Handoff Service
  -> Agent Queue
  -> Human Agent
```

This path should be handled early and should not force the user through unnecessary LLM reasoning.

The human agent receives the conversation history, customer context, detected intent, actions already performed, transaction/refund status, and ticket information.

## Response Path

The response path is centralized through the Conversation Service:

```text
RAG / Tool / Refund / Ticket / Human Workflow
                     |
                     v
              Workflow Result
                     |
                     v
             Response Composer
              /            \
        Template            LLM
             \              /
              v            v
               Customer Response
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

Not every response needs an LLM. Deterministic outcomes such as `refund successful` can use a safe response template, while nuanced explanations can use the LLM.

## Refund Safety

The refund path uses defense in depth:

```text
LLM request
    -> Tools Gateway
    -> Authenticate user
    -> Validate transaction ownership
    -> Validate transaction state
    -> Policy Engine
    -> Check refund limits
    -> Check idempotency
    -> Refund Service
```

If the LLM asks for an amount that violates policy, the request is rejected regardless of what the model generated.

### Idempotency

Refunds are financial operations and must be idempotent. A retry after a timeout must not create a second refund.

Example key:

```text
conversation_id + transaction_id + refund_operation
```

## Policy Engine

The Policy Engine should return a structured decision such as:

```json
{
  "eligible": true,
  "decision": "AUTO_APPROVE",
  "requested_amount": 35,
  "auto_approval_limit": 50,
  "policy_id": "REFUND-017",
  "policy_version": 17
}
```

Or:

```json
{
  "eligible": true,
  "decision": "HUMAN_REVIEW",
  "reason": "AMOUNT_EXCEEDS_AUTO_APPROVAL_LIMIT",
  "requested_amount": 120,
  "auto_approval_limit": 50,
  "policy_id": "REFUND-017",
  "policy_version": 17
}
```

## RAG Design

The vector store contains trusted, approved knowledge such as:

- Refund SOPs
- Billing policies
- Subscription policies
- Product FAQs
- Region-specific policies

Documents should carry metadata such as policy ID, version, region, and effective dates. Retrieval should filter for the active policy version and applicable region before generation.

**RAG provides knowledge; the Policy Engine enforces financial policy.**

## Scalability

Keep agent workers stateless and scale them horizontally. Persistent state lives outside the LLM context.

Potential infrastructure:

- Redis for low-latency conversation/session state.
- Durable database for workflow state and important conversation records.
- Vector database for SOP retrieval.
- Kafka/event bus for refund and support events.
- Object storage for source SOP documents.

## Reliability

| Failure | Handling |
|---|---|
| LLM timeout | Retry/fallback for safe informational requests |
| Refund API timeout | Query operation status before retrying; never blindly issue another refund |
| Duplicate refund request | Idempotency key |
| RAG unavailable | Fallback to safe response or human escalation |
| Policy service unavailable | Do not authorize refund; fail closed |
| Human queue unavailable | Persist handoff/ticket request and retry asynchronously |

For financial operations, **fail closed** is preferred over guessing.

## Security

- Authenticate the customer before account-specific operations.
- Authorize access to account and transaction data.
- Use least-privilege tool permissions.
- Keep the Tools Gateway as a security boundary.
- Encrypt sensitive data in transit and at rest.
- Avoid storing customer PII in the RAG index.
- Log policy decisions and financial actions in an immutable audit trail.
- Treat retrieved documents as data, not executable instructions.

## Observability

Track infrastructure, agent, RAG, business, and safety metrics.

Important examples:

- p50/p95/p99 latency
- LLM latency and token usage
- Tool-call failures
- Intent accuracy
- Retrieval quality
- Automatic refund rate
- Human-review rate
- Refund success/failure rate
- Customer satisfaction
- Unauthorized action attempts

The critical financial safety metric is:

> **Unauthorized automatic refund rate = 0**

## Key Trade-offs

### LLM vs deterministic workflow

Use the LLM for language understanding and flexible reasoning, but deterministic services for authorization and transactions.

### Synchronous vs asynchronous refunds

Use an asynchronous queue for actual refund processing. The chat request can acknowledge the request without blocking on downstream payment processing.

### One LLM vs multiple models

Start with one capable model for simplicity. Introduce smaller specialist models for routing/classification when latency or cost requires it.

### Template vs LLM responses

Use templates for deterministic transactional outcomes and LLM generation for nuanced explanations. This reduces cost, latency, and hallucination risk.

## Interview Takeaways

1. **LLM is not the security boundary.**
2. **Policy enforcement must live outside the LLM.**
3. **RAG knowledge and transactional customer data should be separated.**
4. **All actions go through a controlled Tools Gateway.**
5. **Financial operations require idempotency.**
6. **Human handoff is a first-class workflow.**
7. **Conversation Service owns the request and response lifecycle.**
8. **Asynchronous events decouple long-running refund processing from chat latency.**
9. **Every financial decision should be auditable and policy-versioned.**
10. **The system should fail closed when authorization or policy evaluation is unavailable.**
