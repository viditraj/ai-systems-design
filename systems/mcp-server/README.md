# Production-Grade MCP Server

> Production-grade Model Context Protocol (MCP) platform that lets enterprise AI agents discover and safely invoke tools across GitHub, Jira, Jenkins, Kubernetes, Slack, PagerDuty, and internal services.

## Problem Statement

Design a centralized MCP platform for an enterprise with multiple AI agents that need secure, observable, and scalable access to hundreds of tools exposed by internal and third-party systems.

A typical request might be:

> "Investigate the failed production deployment, identify the root cause, create a Jira ticket, and notify the on-call engineer."

The agent may need to discover and invoke tools such as:

- `github.search_code`
- `github.create_pr`
- `jira.search_issue`
- `jira.create_issue`
- `jenkins.get_build`
- `kubernetes.get_pods`
- `pagerduty.get_incident`
- `slack.send_message`

Assume:

- 100,000 employees
- 10,000 AI agents / agent sessions at peak
- 2,000+ registered tools across 300+ MCP servers
- 5,000 tool calls/sec at peak
- p99 MCP routing overhead under 100 ms, excluding tool execution
- Multi-tenant enterprise environment
- Strict audit, authorization, and data-isolation requirements

## Core Design Principle

**MCP standardizes the tool boundary, but deterministic infrastructure remains responsible for identity, authorization, policy enforcement, execution safety, and observability.**

The LLM is not trusted to directly access enterprise systems.

## Real-World Company Use Case

### Enterprise Engineering Agent Platform

A large technology company wants a common MCP platform for its internal engineering agents instead of every agent implementing separate GitHub, Jira, Jenkins, Kubernetes, Slack, and PagerDuty integrations.

An SRE agent receives:

> "Production deployment `orders-v42` failed. Investigate the failure, identify the root cause, create a Jira ticket with the evidence, and notify the on-call engineer."

The agent can use MCP tools to:

```text
Kubernetes -> inspect pods/events
Jenkins    -> inspect failed pipeline
GitHub     -> inspect recent commits/PRs
PagerDuty  -> inspect active incident
Jira       -> create incident ticket
Slack      -> notify on-call
```

The MCP platform must prevent a read-only agent from invoking destructive tools, isolate tenants, avoid credential leakage, handle thousands of concurrent sessions, and provide a complete audit trail.

## Architecture

See [architecture.md](./architecture.md) for the detailed architecture, request flows, scaling, security, reliability, and operational design.

## Main Components

| Component | Responsibility |
|---|---|
| API Gateway | Authentication, TLS termination, rate limiting, WAF |
| MCP Gateway | Session management, routing, protocol handling |
| Tool Registry | Tool/server discovery, schemas, versions, health |
| Identity Service | User, agent, tenant identity |
| Authorization / Policy Engine | Per-tool and per-argument authorization |
| Secret Broker | Short-lived credentials and secret isolation |
| MCP Server Pool | Domain-specific tool implementations |
| Tool Execution Layer | Sandboxing, concurrency limits, timeouts |
| Approval Service | Human approval for sensitive/destructive tools |
| Task Queue | Durable execution for long-running tools |
| Cache | Tool metadata, schemas, session data |
| Audit Store | Immutable tool-call and policy records |
| Observability | Logs, metrics, traces, tool-level SLOs |

## Request Flows

### 1. Tool Discovery

```text
Agent -> MCP Gateway -> Tool Registry
                    -> Authorization Filter
                    -> Tool Metadata / Schema
                    -> Agent
```

Only tools available to the user/agent should be returned when policy filtering is enabled.

### 2. Tool Invocation

```text
Agent
  |
  v
MCP Gateway
  |
  +--> Authenticate identity
  +--> Validate session
  +--> Validate tool + arguments
  +--> Authorization / Policy
  +--> Rate / Budget Check
  |
  v
MCP Router
  |
  v
MCP Server
  |
  v
Enterprise API
  |
  v
Result Validation -> Audit -> Agent
```

### 3. High-Risk Tool

```text
Agent -> Policy Engine
           |
           +--> LOW/MEDIUM RISK -> Execute
           |
           +--> HIGH/CRITICAL -> Human Approval
                                      |
                                      v
                                  Execute Tool
```

Examples include deleting Kubernetes resources, merging protected GitHub branches, rotating secrets, or modifying production configuration.

## MCP Design

### Protocol Boundary

The platform should preserve MCP semantics while centralizing cross-cutting concerns:

```text
MCP Client
   |
   | JSON-RPC / MCP transport
   v
MCP Gateway
   |
   +--> initialize / session lifecycle
   +--> tools/list
   +--> tools/call
   +--> resources/*
   +--> prompts/*
   |
   v
MCP Server
```

The gateway should avoid becoming a semantic replacement for MCP servers. It should provide routing, policy, identity, and operational controls around the protocol.

### Tool Registry

Each registered tool should maintain metadata such as:

```json
{
  "tool_id": "kubernetes.get_pods",
  "server_id": "kubernetes-prod",
  "version": "v2",
  "tenant": "platform",
  "risk": "low",
  "scopes": ["k8s:read"],
  "timeout_ms": 3000,
  "concurrency_limit": 100,
  "input_schema_hash": "...",
  "healthy": true
}
```

Tool schemas should be versioned and validated independently of the LLM.

## Security

### Authentication

Use enterprise identity and workload identity rather than long-lived credentials embedded in agents.

```text
User / Agent Identity
        |
        v
 Identity Provider
        |
        v
 Short-lived token
        |
        v
 MCP Gateway
```

### Authorization

Authorization should be checked at multiple layers:

```text
User/Agent
   -> Tenant Policy
   -> Tool Permission
   -> Resource Permission
   -> Argument Constraints
   -> Execute
```

The MCP server must still validate authorization rather than trusting the gateway alone.

### Secrets

Agents should never receive raw API keys. The execution layer obtains short-lived credentials from a secret broker and injects them only into the tool process.

### Prompt / Tool Injection

Treat tool descriptions and tool results as untrusted input. A compromised document or tool response must not be able to grant the agent new capabilities.

Capability elevation should require deterministic policy evaluation and, for sensitive operations, explicit approval.

## Scalability

### Stateless Gateway

Keep the MCP gateway horizontally scalable wherever the transport permits:

```text
                 Load Balancer
                      |
       +--------------+--------------+
       |              |              |
  MCP Gateway     MCP Gateway     MCP Gateway
       |              |              |
       +--------------+--------------+
                      |
             Shared Registry / Cache
                      |
              MCP Server Fleet
```

Session state that must survive process failure should live in a shared durable or distributed state layer.

### Tool Catalog Scaling

With thousands of tools, returning every tool schema to every agent creates context and latency problems.

Use:

- Hierarchical tool namespaces
- Metadata filtering
- Capability-based discovery
- Tool search / retrieval
- Per-agent tool allowlists
- Schema caching

The agent should receive only the smallest relevant tool set.

### Backpressure

A single agent can generate many tool calls. Enforce:

- Per-agent concurrency limits
- Per-tenant quotas
- Per-tool concurrency limits
- Per-turn tool-call budgets
- Queue depth limits
- Cost / execution budgets

## Reliability

| Failure | Handling |
|---|---|
| MCP server timeout | Timeout and bounded retry for safe operations |
| Tool API timeout | Query status before retrying side effects |
| Duplicate write | Idempotency key |
| MCP server crash | Retry or fail over when safe |
| Gateway crash | Reconnect session / recover durable workflow state |
| Registry unavailable | Serve cached schemas; fail closed for sensitive changes |
| Policy engine unavailable | Fail closed for protected operations |
| Approval service unavailable | Keep action pending; never execute protected action |
| Tool schema mismatch | Version compatibility check and reject |
| Dependency overload | Circuit breaker + backpressure |

## Long-Running Tools

Not every MCP tool completes synchronously.

Example:

```text
Agent -> tools/call
          |
          v
     Create Task
          |
          v
     Task Queue
          |
          v
     Worker -> Jenkins/Kubernetes
          |
          v
      Task Status
          |
          v
        Agent
```

Use durable task state for operations that can take seconds or minutes. The client should be able to obtain progress/status without replaying the original side effect.

## Observability

Every tool call should emit structured telemetry:

```text
request_id
trace_id
user_id
agent_id
tenant_id
mcp_server
tool_name
tool_version
risk_level
authorization_decision
latency
queue_time
execution_status
error_code
result_size
```

Recommended dashboards:

- Tool-call rate by server/tool
- p50/p95/p99 latency
- Error rate
- Authorization denials
- Approval latency
- Queue depth
- Active sessions
- Per-tenant usage
- Tool cost / compute consumption

## Design Trade-offs

### Centralized MCP Gateway vs Direct Client -> Server

**Centralized:** stronger governance, consistent security, better observability, easier tenant isolation; additional latency and a central dependency.

**Direct:** lower latency and simpler path; much harder to enforce consistent enterprise security and policy.

For a large enterprise, use the gateway for governance while allowing controlled direct connections for low-risk internal deployments where appropriate.

### Stateless vs Stateful Sessions

Prefer stateless request handling when possible. Store only required protocol/session state in a shared store so gateway instances can scale horizontally.

### Synchronous vs Asynchronous Tool Calls

Use synchronous calls for short read operations. Use asynchronous durable execution for long-running or retry-sensitive operations.

## Interview Focus Areas

A strong candidate should cover:

1. MCP protocol and client/server boundary
2. Tool discovery and schema management
3. Authentication and authorization
4. Multi-tenancy
5. Tool execution isolation
6. Rate limiting and backpressure
7. Idempotency and retries
8. Long-running tool execution
9. Human approval for risky actions
10. Observability and auditability
11. Tool versioning and compatibility
12. Horizontal scaling and failure recovery

## Further Reading

- [Mock Interview](./mock-interview.md)
- [Architecture](./architecture.md)
