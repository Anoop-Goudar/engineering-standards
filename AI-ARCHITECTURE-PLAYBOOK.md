# AI Application Architecture Playbook — v1.0

**Final frozen baseline**

> **Core principle: Choose the simplest architecture that satisfies the actual
> requirements and measurable quality targets.**

For every complexity addition, ask:

> **What requirement does this component satisfy that a simpler design cannot?**

## Contents

1. [Purpose](#1-purpose)
2. [Core Principles](#2-core-principles)
3. [Architecture Process](#3-architecture-process)

**Phase 1 — Understand**

4. [Product and System Constraints](#4-product-and-system-constraints)
5. [Define What "Good" Means](#5-define-what-good-means)
6. [Trust and Threat Model](#6-trust-and-threat-model)

**Phase 2 — Decompose**

7. [Define System Boundaries](#7-define-system-boundaries)
8. [Data Architecture](#8-data-architecture)
9. [Runtime and State Semantics](#9-runtime-and-state-semantics)
10. [Derived Data Lineage](#10-derived-data-lineage)
11. [Identify AI Responsibilities](#11-identify-ai-responsibilities)
12. [Execution Model](#12-execution-model)
13. [Durable Work and Backpressure](#13-durable-work-and-backpressure)
14. [State and Coordination](#14-state-and-coordination)

**Phase 3 — Design AI**

15. [Create a Starter Evaluation Set](#15-create-a-starter-evaluation-set)
16. [Choose AI Pattern](#16-choose-ai-pattern)
17. [Pattern Decision Gates](#17-pattern-decision-gates)
18. [Context Engineering](#18-context-engineering)
19. [Retrieval and Knowledge](#19-retrieval-and-knowledge)
20. [Memory](#20-memory)
21. [Model Strategy](#21-model-strategy)
22. [Prompt and Version Management](#22-prompt-and-version-management)
23. [Structured Outputs and Validation](#23-structured-outputs-and-validation)
24. [Tools and Agency](#24-tools-and-agency)
25. [Untrusted Content and Prompt Injection](#25-untrusted-content-and-prompt-injection)

**Phase 4 — Design Production System**

26. [Security Architecture](#26-security-architecture)
27. [Safety and Moderation](#27-safety-and-moderation)
28. [Reliability](#28-reliability)
29. [Observability](#29-observability)
30. [Auditability vs Observability](#30-auditability-vs-observability)
31. [Cost Architecture](#31-cost-architecture)
32. [Scaling and Overload Behavior](#32-scaling-and-overload-behavior)

**Phase 5 — Validate**

33. [Failure Mode Analysis](#33-failure-mode-analysis)
34. [Full Evaluation Architecture](#34-full-evaluation-architecture)
35. [Evaluation Dataset](#35-evaluation-dataset)
36. [Complexity Review](#36-complexity-review)
37. [Architecture Review](#37-architecture-review)
38. [Launch Readiness Gate](#38-launch-readiness-gate)

**Phase 6 — Implement**

39. [Technology Selection](#39-technology-selection)
40. [Interfaces and Contracts](#40-interfaces-and-contracts)
41. [Implementation Plan](#41-implementation-plan)
42. [AI-Assisted Implementation Contract](#42-ai-assisted-implementation-contract)
43. [ADR](#43-adr)
44. [Reusable Architecture Canvas](#44-reusable-architecture-canvas)
45. [Core Mental Model](#45-core-mental-model)
46. [Architecture Loop](#46-architecture-loop)
47. [Final Rule](#47-final-rule)

---

## 1. Purpose

Define a reusable method for architecting production-grade AI applications
before implementation.

## 2. Core Principles

1. Architecture before implementation
2. Simplest sufficient architecture
3. AI is a component, not the architecture
4. Deterministic where deterministic is possible
5. Every probabilistic component needs evaluation
6. Security is architectural
7. Design for failure

## 3. Architecture Process

**Understand → Decompose → Design AI → Design Production System → Validate → Implement**

---

# Phase 1 — Understand

## 4. Product and System Constraints

Define:

- Users and use cases
- Functional requirements
- Non-functional requirements
- Data sensitivity
- Availability
- Latency
- Throughput
- Regulatory/compliance requirements
- Budget
- Deployment constraints
- Integration constraints

## 5. Define What "Good" Means

Turn vague requirements into measurable targets.

Start with the system-level targets:

- Quality
- Accuracy
- Latency
- Availability
- Cost/request
- Safety
- False-positive/false-negative tolerance
- Recovery expectations

Then define success criteria per AI responsibility. Different responsibilities
require different metrics and thresholds:

| Responsibility | Typical evaluation focus |
|---|---|
| Generation | Task quality, factuality, relevance, safety |
| Extraction | Field-level precision/recall, schema validity |
| Classification | Precision, recall, F1, critical-class false negatives |
| RAG | Retrieval recall/precision, answer groundedness, citation/attribution quality |
| Tool use | Correct tool selection, argument validity, authorization, task success |
| Workflow | Step correctness, state-transition correctness, completion rate |
| Agent | Task success, tool trajectory, termination behavior, cost/latency |
| Safety | Detection recall, intervention correctness, critical false negatives |

The exact metrics and thresholds must be chosen from the product requirements
and risk profile rather than assumed from this table.

## 6. Trust and Threat Model

Identify:

- Who is trusted?
- What is untrusted?
- What data is sensitive?
- What can the model access?
- What actions can the model trigger?
- What happens if the model is manipulated?
- What happens if a user attempts to access another user's data?

### Trust-Boundary Walk

Explicitly inspect important boundary pairs, such as:

- Document → OCR
- OCR → LLM
- Service → Service
- Client → Identity Provider

For each boundary ask:

- What is trusted on each side?
- What is authenticated?
- What is validated?
- What data crosses the boundary?
- What permissions must be re-established?

### Scope Hierarchy

Model authorization explicitly as:

**Principal → Account/Tenant → Persona/Sub-identity → Resource**

Ask:

> **Is this an authorization problem within the same account, or a
> cross-tenant isolation problem?**

Do not assume that being authenticated to an account automatically authorizes
access to every persona, sub-identity, or resource within it.

---

# Phase 2 — Decompose

## 7. Define System Boundaries

Identify:

- Client
- API/application layer
- AI components
- Data stores
- External services
- Background processing
- Workers
- Third-party identity providers

### Service Boundary Gate

Do not create a separate service merely because a component is conceptually
different.

> **What independent scaling, deployment, ownership, security, reliability, or
> technology boundary requires this to become a separate service?**

## 8. Data Architecture

Define:

- Source-of-truth data
- Derived data
- Transactional data
- Search/index data
- Cache data
- Object/blob storage
- Retention
- Ownership
- Access boundaries

## 9. Runtime and State Semantics

For every stateful component determine:

- Where state lives
- Who owns it
- Whether it is authoritative
- Consistency requirements
- Read/write semantics
- Concurrency behavior
- Recovery behavior
- Whether state can be reconstructed

### Runtime Persistence Gate

Ask:

> **Does the actual runtime guarantee the persistence and visibility semantics
> this component assumes?**

Example:

> **In-memory cache + serverless runtime = potentially instance-local and
> ephemeral.**

Never assume an in-memory cache is globally shared or durable merely because
application code can access it.

## 10. Derived Data Lineage

For every derived artifact identify:

**Source → Transformation → Storage → Consumer → Invalidation/Rebuild**

Examples:

- OCR output
- embeddings
- summaries
- classifications
- caches
- indexes
- analytics

## 11. Identify AI Responsibilities

Explicitly separate:

**Deterministic logic**

- validation
- authorization
- calculations
- routing
- business rules
- state transitions

**Probabilistic logic**

- classification
- extraction
- generation
- reasoning
- semantic matching
- summarization

## 12. Execution Model

For each operation determine whether it is:

- Synchronous request/response
- Asynchronous job
- Scheduled job
- Event-driven
- Streaming
- Human-in-the-loop

## 13. Durable Work and Backpressure

Ask:

> **Can this work be lost, duplicated, retried, delayed, or overwhelm the
> system?**

### Queue Introduction Gate

Introduce a durable queue only when requirements justify:

- durable work
- retry semantics
- decoupling
- backpressure
- independent worker scaling
- failure isolation

Otherwise, keep execution simpler.

## 14. State and Coordination

Determine:

- Required consistency
- Concurrency model
- Locking
- Idempotency
- Deduplication
- Ordering
- Distributed coordination

### Concurrency Decision Guide

Choose the simplest strategy that satisfies the actual concurrency requirement:

| Strategy | Use when |
|---|---|
| Server-side serialization | Operations for the same logical entity can be safely processed one at a time and throughput requirements permit serialization. |
| Optimistic concurrency | Conflicts are possible but relatively uncommon, and conflicting writes can be detected and retried or rejected safely. |
| Last-write-wins | Concurrent updates are independent or the product explicitly accepts the newest accepted write as authoritative. |
| Distributed locking | Multiple workers/processes must coordinate exclusive access to a shared resource and simpler serialization or optimistic concurrency cannot satisfy the requirement. |
| CRDT | Multiple replicas must accept concurrent updates independently and the state can be modeled with formally mergeable operations. |

Do not choose a concurrency mechanism because it is familiar or technically
available. Define the required consistency and conflict semantics first, then
select the simplest mechanism that satisfies them.

---

# Phase 3 — Design AI

## 15. Create a Starter Evaluation Set

Create **20–30 representative examples** before making major AI architecture
decisions.

This is a development decision set, **not statistically sufficient by default**.

For high-consequence, safety-critical, security-sensitive, rare-failure, or
critical false-negative responsibilities, use larger and/or adversarially
constructed evaluation sets.

## 16. Choose AI Pattern

Choose the simplest pattern that satisfies the requirement:

- **A. Simple Model Call**
- **B. Structured Generation**
- **C. RAG**
- **D. Workflow**
- **E. Bounded Tool-Using System**
- **F. Agent**

## 17. Pattern Decision Gates

### RAG Gate

Ask:

- Is external/private knowledge required?
- Can deterministic retrieval satisfy the requirement?
- Is semantic retrieval actually necessary?
- How large/variable is the corpus?
- What quality target must retrieval meet?
- What permissions apply to retrieved information?
- How frequently does knowledge change?
- How will indexes/retrieval data be invalidated or rebuilt?

**RAG complexity gate**

> **What measurable retrieval requirement cannot be met by a simpler retrieval
> mechanism?**

### Workflow vs Agent

Ask:

> **Can the sequence be expressed as normal application code without the LLM
> deciding what happens next?**

If yes → **Workflow**

If the LLM needs to select tools:

> **Is the loop termination and execution boundary defined by application code?**

If yes → **Bounded tool-using system**

Use an **Agent** only when dynamic planning/autonomy is genuinely required and
its additional cost, latency, unpredictability, and evaluation burden are
justified.

## 18. Context Engineering

Define:

- System instructions
- User input
- Retrieved knowledge
- Conversation history
- Memory
- Tool results
- Context limits
- Context prioritization
- Context isolation

## 19. Retrieval and Knowledge

> **This section assumes §17's RAG Gate has already justified retrieval; it
> selects the retrieval technique, not whether retrieval is needed.**

Determine:

- Retrieval source
- Retrieval mechanism
- Keyword vs semantic vs hybrid
- Chunking
- Metadata
- Filtering
- Ranking
- Reranking
- Permissions
- Freshness
- Indexing
- Invalidation
- Evaluation

## 20. Memory

First ask:

> **Does this feature need information beyond the current request/session?**

If no → **do not introduce memory.**

If yes, determine:

- What should be remembered?
- Who owns the memory?
- How is it retrieved?
- How is it updated?
- How is it deleted?
- What permissions apply?
- What happens when memory is stale or incorrect?

## 21. Model Strategy

Use the starter evaluation set.

Select models based on:

- Quality
- Capability
- Latency
- Cost
- Reliability
- Context requirements

Introduce model routing only when at least two responsibilities have
**materially different** quality, cost, latency, capability, or reliability
requirements.

## 22. Prompt and Version Management

Define:

- Prompt ownership
- Versioning
- Change management
- Rollback
- Evaluation before promotion
- Model/prompt compatibility

## 23. Structured Outputs and Validation

Use structured outputs when downstream logic depends on model-generated
structure.

Validate:

- Schema
- Required fields
- Types
- Ranges
- Business constraints
- Safety constraints

Never treat model output as inherently trustworthy.

## 24. Tools and Agency

For every tool define:

- Authorization
- Input validation
- Output validation
- Scope
- Timeout
- Failure handling
- Audit requirements

> **The model decides; the system authorizes.**

### Human Approval Gate

If an action is:

- irreversible,
- high-consequence,
- financially/materially significant,
- difficult to undo,

synchronous human confirmation may be required even when the user has generally
authorized the capability.

## 25. Untrusted Content and Prompt Injection

Separate:

**Trusted instructions + authorized context + untrusted content**

Never allow untrusted content to implicitly grant permissions.

Validate authorization **before** sensitive context enters the AI request and
before tools execute.

---

# Phase 4 — Design Production System

## 26. Security Architecture

Define:

- Authentication
- Authorization
- Data isolation
- Context isolation

**Identity invariant**

> **Identity must come from verified authentication/session state, never a
> client-supplied identity field.**

**Third-party identity invariant**

> **Verify the external identity provider assertion/token and bind it to the
> authenticated principal.**

**Within-account isolation**

Correctly enforce persona/sub-identity selection and independently test it.

**Context assembly authorization**

Authorization must occur before sensitive information enters the AI context.

## 27. Safety and Moderation

Define:

- Safety boundaries
- Detection
- Intervention
- Escalation
- Refusal behavior
- Human escalation
- False-positive/false-negative expectations
- Safety evaluation

## 28. Reliability

Design for:

- Timeouts
- Retries
- Idempotency
- Partial failure
- Dependency failure
- Model failure
- Rate limits
- Graceful degradation
- Recovery

Avoid uncontrolled retries. Retry only when the failure is retryable, use
bounded attempts/backoff, and ensure the operation is safe to repeat or
otherwise protected against duplicate effects.

## 29. Observability

Track:

- Logs
- Metrics
- Traces
- AI request/response metadata
- Model version
- Prompt version
- Latency
- Token usage
- Cost
- Failure rates
- Queue/job state

**Coverage matters:**

- % of critical operations traced
- % of model calls with model/prompt version recorded
- % of failed jobs observable
- % of critical workflows covered by telemetry

## 30. Auditability vs Observability

**Observability**

Answers:

> **What is happening in the system?**

Used primarily for engineering and operations.

**Auditability**

Answers:

> **What happened, who/what caused it, and can we prove it later?**

Used for:

- Security
- Compliance
- Legal
- Sensitive actions
- Accountability

Consider:

- Retention
- Immutability
- Access control
- Audit completeness

## 31. Cost Architecture

Perform lightweight cost comparison during pattern selection.

Detailed analysis should include:

- Model cost
- Token usage
- Retrieval cost
- Storage
- Compute
- Queue/workers
- Third-party APIs
- Observability
- Network
- Expected volume

## 32. Scaling and Overload Behavior

Define:

- Expected load
- Peak load
- Scaling boundaries
- Rate limits
- Backpressure
- Load shedding
- Degradation strategy
- Recovery

---

# Phase 5 — Validate

## 33. Failure Mode Analysis

For each major component ask:

- What can fail?
- How does it fail?
- What happens to users?
- Can work be lost?
- Can data become inconsistent?
- Can failure propagate?
- How is it detected?
- How is it recovered?

## 34. Full Evaluation Architecture

Define:

- Offline evaluation
- Regression evaluation
- Safety evaluation
- Retrieval evaluation
- Model evaluation
- End-to-end evaluation
- Production monitoring
- Human evaluation where required

## 35. Evaluation Dataset

Maintain:

- Representative cases
- Edge cases
- Failure cases
- Adversarial cases
- Safety cases
- Regression cases
- Versioning
- Ground truth/expected behavior

## 36. Complexity Review

Whenever introducing:

- Queue
- Vector DB
- RAG
- Agent
- Orchestration framework
- Microservice
- Distributed cache
- Additional model
- Realtime infrastructure

ask:

1. What requirement requires it?
2. What simpler alternative was considered?
3. What measurable benefit does it provide?
4. What new failure modes does it introduce?
5. What operational cost does it add?

## 37. Architecture Review

Review:

- Requirements coverage
- Data flow
- Security
- AI pattern
- State
- Reliability
- Evaluation
- Cost
- Scaling
- Failure modes
- Operational complexity

## 38. Launch Readiness Gate

Before launch verify:

- Empirical model justification
- Persona/context isolation
- Third-party identity verification
- Safety approval
- Backpressure
- Critical evaluations enforced in CI/CD
- Audit requirements
- Rollback
- Recovery

---

# Phase 6 — Implement

## 39. Technology Selection

Only after architecture decisions.

Choose technologies based on:

- Requirements
- Team capability
- Operational burden
- Ecosystem
- Cost
- Reliability
- Vendor constraints

## 40. Interfaces and Contracts

Define:

- API contracts
- Event contracts
- Data contracts
- Tool schemas
- Error contracts
- Authentication/authorization contracts

## 41. Implementation Plan

Break architecture into independently verifiable increments.

Each increment should have:

- Goal
- Components
- Interfaces
- Tests
- Evaluation
- Observability
- Rollback

## 42. AI-Assisted Implementation Contract

AI coding agents should receive:

- Architecture
- Requirements
- Constraints
- Interfaces
- Data model
- Security rules
- Evaluation requirements
- Testing requirements

The agent implements the architecture rather than inventing it.

## 43. ADR

For significant decisions record:

- Context
- Decision
- Alternatives
- Why chosen
- Trade-offs
- Consequences

## 44. Reusable Architecture Canvas

```text
SYSTEM
├── Purpose:
├── Users:
├── Requirements:
├── Quality targets:
│
├── TRUST
│   ├── Trusted actors:
│   ├── Untrusted inputs:
│   ├── Sensitive data:
│   └── Threats:
│
├── DATA
│   ├── Source of truth:
│   ├── Derived data:
│   ├── Lineage:
│   ├── Search/index:
│   ├── Cache:
│   └── Object storage:
│
├── RUNTIME
│   ├── Request path:
│   ├── Async work:
│   ├── Durable jobs:
│   └── State/coordination:
│
├── AI
│   ├── Responsibilities:
│   ├── Pattern:
│   ├── Retrieval:
│   ├── Memory:
│   ├── Model:
│   ├── Model routing:
│   └── Prompt/version:
│
├── TOOLS
│   ├── Tools:
│   ├── Authorization:
│   └── Validation:
│
├── SAFETY
│   ├── Safety boundaries:
│   ├── Moderation:
│   ├── High-consequence actions:
│   └── Approval requirement:
│
├── SECURITY
│   ├── Authentication:
│   ├── Authorization:
│   ├── Data isolation:
│   └── Context isolation:
│
├── RELIABILITY
│   ├── Failure modes:
│   ├── Retry:
│   ├── Idempotency:
│   ├── Degradation:
│   └── Recovery:
│
├── OBSERVABILITY
│   ├── Logs:
│   ├── Metrics:
│   ├── Traces:
│   └── AI telemetry:
│
├── COST
│   ├── Model:
│   ├── Infrastructure:
│   └── Expected cost:
│
└── VALIDATION
    ├── Evaluation set:
    ├── Quality metrics:
    ├── Safety evaluation:
    ├── Regression tests:
    └── Launch gate:
```

## 45. Core Mental Model

> **Requirements → Trust → Data → State → Execution → AI → Security →
> Reliability → Observability → Cost → Evaluation**

## 46. Architecture Loop

> **Requirement → Simpler option → Failure mode → Measurable target → Decision →
> Evaluation**

## 47. Final Rule

> **Do not architect for what AI systems can do. Architect for what the product
> actually requires.**
