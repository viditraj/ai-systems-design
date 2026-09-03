# AI Coding Agent

> Production-grade AI coding agent that understands a large repository, plans software changes, retrieves relevant code, edits files, runs tests in an isolated sandbox, reviews its own changes, and opens a pull request with auditable, resumable execution.

## Real-World Example

Design a coding agent similar to modern AI developer tools such as an IDE coding agent or autonomous software-engineering agent.

A developer can ask:

> "Fix the authentication timeout bug in the payments service, add a regression test, run the relevant tests, and open a PR."

The agent should independently inspect the repository, understand relevant code, formulate a plan, make bounded edits, validate them, and produce a reviewable patch.

The agent **must not be trusted with unrestricted production credentials or arbitrary host execution**. Code execution happens in an isolated sandbox and repository changes are validated before a PR is created.

## Problem Statement

Build a coding agent that can:

- Understand large, multi-language repositories.
- Search code using lexical, semantic, and structural retrieval.
- Inspect files, symbols, dependencies, tests, build configuration, and git history.
- Plan multi-file changes.
- Read and write files through controlled tools.
- Run builds, tests, linters, and static analysis.
- Recover from failed commands and revise its plan.
- Keep durable state for long-running tasks.
- Create commits and pull requests.
- Explain what changed and why.
- Detect risky changes and require human approval.
- Prevent prompt injection from source code, issues, docs, and command output.
- Support concurrent tasks across many repositories.

## Example User Experience

```text
Developer
   |
   | "Fix login timeout and add tests"
   v
Coding Agent
   |
   +--> Understand repository
   +--> Find authentication code
   +--> Inspect recent changes
   +--> Build execution plan
   +--> Edit files
   +--> Run tests
   +--> Diagnose failures
   +--> Revise patch
   +--> Run final validation
   +--> Create branch / commit
   +--> Open PR
   v
Developer reviews PR
```

## Core Design Principle

**The LLM proposes reasoning and tool calls; deterministic infrastructure controls repository access, execution, validation, and side effects.**

Never give the model unrestricted shell access, arbitrary network access, production credentials, or direct write access to protected branches.

## Architecture

See [architecture.md](./architecture.md) for the detailed architecture, repository indexing, agent loop, sandboxing, execution model, scaling, reliability, security, and interview deep dives.

## Main Components

| Component | Responsibility |
|---|---|
| API Gateway | Authentication, authorization, rate limiting, request routing |
| Agent API | Creates and streams coding tasks |
| Agent Orchestrator | Durable state machine for planning, execution, validation, and recovery |
| Planner | Converts the developer request into a structured implementation plan |
| Repository Profiler | Detects languages, frameworks, build systems, test frameworks, package managers, and repository conventions |
| Code Indexer | Parses source code, symbols, imports, tests, docs, and configuration |
| Lexical Search | Exact search for identifiers, errors, paths, and symbols |
| Code Embeddings | Semantic retrieval over code and documentation |
| Symbol / Dependency Graph | Structural navigation and dependency-aware retrieval |
| Context Builder | Selects and compresses relevant repository context |
| Tool Gateway | Validates every agent tool call |
| File Tools | Controlled read, write, patch, and diff operations |
| Git Tools | Branch, diff, commit, rebase, and PR operations |
| Execution Sandbox | Isolated environment for builds, tests, linters, and scripts |
| Test / Validation Service | Executes deterministic checks and collects results |
| Model Gateway | Model routing, token budgets, fallback, cost and latency controls |
| Task Queue | Durable asynchronous execution |
| State Store | Workflow checkpoints, task state, idempotency, and artifacts |
| Artifact Store | Logs, diffs, test reports, patches, and generated artifacts |
| Policy Engine | Risk classification and deterministic action controls |
| Observability | Logs, traces, metrics, evaluations, and audit events |

## Request Types

### 1. Code Question

```text
Developer -> Agent -> Repository Retrieval -> LLM -> Explanation
```

### 2. Code Change

```text
Developer
   -> Planner
   -> Repository Retrieval
   -> Structured Plan
   -> File Edits
   -> Tests
   -> Review
   -> Diff / PR
```

### 3. Bug Fix

```text
Issue / Developer Request
        |
        v
Triage + Repository Understanding
        |
        v
Failure Reproduction
        |
        v
Root Cause Analysis
        |
        v
Patch + Regression Test
        |
        v
Validation
        |
        v
PR
```

### 4. Autonomous Defect Resolution

```text
Jira / GitHub Issue
        |
        v
Agent Task Queue
        |
        v
Repo Profiler + Code Retrieval
        |
        v
Planner
        |
        v
Sandbox
        |
        v
Patch -> Test -> Review -> Retry
        |
        v
PR + Summary
```

## Repository Understanding

A coding agent cannot depend on embeddings alone.

Use multiple representations:

```text
Repository
   |
   +--> File Tree
   +--> AST / Parser
   +--> Symbols
   +--> Imports / Calls
   +--> Dependency Graph
   +--> Tests
   +--> Build Metadata
   +--> Git History
   +--> Documentation
   |
   +------------------+
                      v
              Repository Index
              /      |       \
             /       |        \
        Lexical   Vector    Graph
        Search    Search    Search
```

For code retrieval, combine:

- BM25 / lexical search for exact identifiers.
- Embeddings for semantic similarity.
- Symbol search for definitions and references.
- Dependency graph traversal for related code.
- Test mapping for likely regression tests.
- Git history for previous fixes and ownership context.

## Tool Safety

Every tool call passes through a Tool Gateway:

```text
LLM
 |
 v
Tool Call
 |
 v
Schema Validation
 |
 v
Policy Check
 |
 +--> Read-only operation -> Execute
 |
 +--> File mutation -> Workspace validation -> Execute
 |
 +--> Shell command -> Sandbox policy -> Execute
 |
 +--> Git push / PR -> Approval policy -> Execute
```

The model cannot bypass the gateway by generating arbitrary API requests.

## Sandbox

All generated code is executed in an isolated workspace.

```text
                 Agent
                   |
                   v
             Sandbox Manager
                   |
        +----------+----------+
        |                     |
        v                     v
   Ephemeral VM          Container / MicroVM
        |                     |
        +----------+----------+
                   |
                   v
            Repository Clone
                   |
                   v
        Build / Test / Lint
```

Recommended controls:

- Ephemeral filesystem.
- CPU and memory limits.
- Execution timeout.
- Process limits.
- Network deny-by-default.
- Allowlisted package registries when required.
- No production credentials.
- Secret redaction.
- Read-only base image.
- Workspace cleanup after task completion.

## Agent Loop

Use a bounded state machine rather than an infinite autonomous loop.

```text
UNDERSTAND
    |
    v
PLAN
    |
    v
RETRIEVE
    |
    v
EDIT
    |
    v
VALIDATE
    |
    +---- failure ----> DIAGNOSE -> EDIT
    |
    v
REVIEW
    |
    v
PR / COMPLETE
```

Each transition is persisted so the task can resume after worker failure.

## Validation Pyramid

Do cheap checks first and expensive checks later.

```text
                Full Integration Tests
                       /\
                      /  \
                 E2E /    \
                    /      \
              Integration Tests
                   /          \
                  /            \
             Unit Tests       \
                /              \
               /                \
          Lint / Type Check / Build
```

A practical pipeline:

1. Patch/schema validation.
2. Formatting.
3. Lint.
4. Type checking / compilation.
5. Targeted unit tests.
6. Related integration tests.
7. Broader test suite when required.
8. Security/static analysis.

## Pull Request Flow

```text
Validated Workspace
       |
       v
Generate Diff
       |
       v
AI Review + Deterministic Checks
       |
       v
Commit to Agent Branch
       |
       v
Push
       |
       v
Create PR
       |
       v
PR Description
 ├── Problem
 ├── Root Cause
 ├── Changes
 ├── Tests
 ├── Risks
 └── Agent Trace / Evidence
```

Protected branches should still enforce normal repository rules and human review.

## Long-Running Tasks

Do not keep the workflow attached to a single HTTP request.

```text
POST /agent/tasks
        |
        v
Task Queue
        |
        v
Agent Worker
        |
        v
Durable State
 ├── queued
 ├── understanding
 ├── planning
 ├── editing
 ├── testing
 ├── waiting_for_approval
 ├── retrying
 ├── completed
 └── failed
```

The API returns a task ID and exposes progress through SSE/WebSockets or polling.

## Failure Handling

| Failure | Handling |
|---|---|
| Model timeout | Retry safe request or use model fallback |
| Tool timeout | Retry idempotent reads; reconcile side effects |
| Test failure | Feed structured failure back to diagnosis state |
| Sandbox crash | Recreate sandbox from checkpoint |
| Worker crash | Resume durable workflow |
| Bad patch | Revert workspace to last checkpoint |
| Repository changed during task | Rebase/re-index and revalidate |
| Git push conflict | Fetch, rebase, validate, retry |
| Dependency install failure | Use cached dependencies or approved mirror |
| Malicious command | Sandbox/policy rejects execution |

## Security

Treat all repository content as untrusted input.

Potential prompt injection sources include:

- Source comments.
- README files.
- Issue descriptions.
- Commit messages.
- Test output.
- Generated files.
- Dependency metadata.
- Tool output.

Use defense in depth:

```text
Untrusted Repository Content
          |
          v
Explicit Context Boundaries
          |
          v
Structured Tool Schemas
          |
          v
Tool Allow-list
          |
          v
Sandbox Policy
          |
          v
Credential Isolation
          |
          v
Deterministic Validation
          |
          v
Human Approval for High-Risk Actions
```

## Scalability

Separate interactive APIs from asynchronous agent execution.

```text
Users
  |
  v
API Gateway
  |
  +--> Task API ---------> Task DB
  |
  +--> Streaming API <---- Event Bus
  |
  v
Task Queue
  |
  +--> Planner Workers
  +--> Retrieval Workers
  +--> Coding Workers
  +--> Validation Workers
  +--> PR Workers
```

Repository indexing should be incremental. Re-index only changed files and affected symbols rather than rebuilding the entire repository after every commit.

Cache immutable artifacts such as dependency layers, base images, parser results, embeddings, and frequently accessed repository metadata.

## Model Routing

Use different models for different tasks:

```text
Task Router
 |
 +--> Small model: classification / extraction / summaries
 |
 +--> Code model: code completion / patch generation
 |
 +--> Reasoning model: debugging / architecture / planning
 |
 +--> Deterministic tools: search / build / tests / git
```

The agent should not spend an expensive reasoning call on deterministic operations.

## Evaluation

Evaluate the complete software-engineering task, not only generated code.

### Offline

- Patch correctness.
- Test pass rate.
- Build success rate.
- Regression rate.
- SWE-bench-style task resolution.
- Code quality.
- Tool-selection accuracy.
- Retrieval Recall@K / MRR.

### Online

- Task completion rate.
- PR acceptance / merge rate.
- Human intervention rate.
- Time to validated patch.
- Test retry count.
- Cost per resolved issue.
- Rollback / revert rate.
- Sandbox failure rate.

A useful primary metric is:

```text
Validated Task Success
= correct patch
  + required tests pass
  + no critical regression
  + repository policy satisfied
```

## Key Trade-offs

### Embeddings vs Code Graph

Embeddings are good for semantic intent; graphs are better for definitions, references, and dependency relationships. A hybrid approach is stronger.

### One Agent vs Specialized Agents

Start with one orchestrator and deterministic tools. Add specialist agents only when measured performance improves.

### Full Repository Context vs Retrieval

Sending the entire repository to the model is expensive and impossible for large codebases. Use hierarchical retrieval and context selection.

### Persistent Sandbox vs Ephemeral Sandbox

Persistent environments are faster but create contamination and security risks. Ephemeral environments provide stronger isolation and reproducibility.

### Autonomous PR vs Human Review

Agents can create PRs autonomously, but protected branches and high-risk changes should retain human review.

### Synchronous vs Asynchronous

Short code questions can be synchronous. Multi-file fixes and full test suites should be asynchronous and resumable.

## Interview Takeaways

1. **A coding agent is an orchestration system, not just an LLM prompt.**
2. **Repository understanding requires lexical, semantic, and structural retrieval.**
3. **The sandbox is a security boundary.**
4. **LLM output must never directly become unrestricted shell execution.**
5. **Agent workflows need durable checkpoints and bounded retries.**
6. **Tests are deterministic feedback signals for the agent loop.**
7. **Incremental repository indexing is essential at scale.**
8. **Git operations and external side effects need idempotency and policy controls.**
9. **Measure validated task success, not merely code-generation quality.**
10. **Human review remains important for protected or high-risk changes.**
