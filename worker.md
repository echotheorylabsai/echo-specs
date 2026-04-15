# Worker Service Specification (v3)

## Generic, Implementation-Agnostic

> **Principle:** This spec defines WHAT a worker service must address, not HOW. It separates universal concerns from architectural patterns from implementation choices. Each concern states the requirement, then describes the valid approaches, then optionally illustrates with examples.
>
> **Scope:** This spec applies to any system that accepts units of work and executes them asynchronously — whether that system is a library (BullMQ), a platform (Hatchet), a durable execution engine (Temporal), a managed service (AWS Step Functions), or a custom-built service.

---

## How This Spec Is Structured

Three layers, clearly separated:

| Layer | What It Contains | Changes Between Systems? |
| --- | --- | --- |
| **Concern** | The question every system must answer | No — universal |
| **Patterns** | The known valid approaches to answering it | Rarely — architectural |
| **Implementation** | The specific technology choice | Always — per system |

The spec only contains the first two layers. Implementations fill in the third.

---

## 0. System Classification

Before evaluating any worker service, classify it. Different categories have different expectations.

### 0.1 System Type

| Type | Description | Scope of Responsibility | Examples |
| --- | --- | --- | --- |
| **Library** | Embedded in the application process. Provides primitives. Application owns policy. | Queue, retry, dispatch | BullMQ, Celery, Sidekiq |
| **Platform** | Standalone service with UI, API, and built-in policies. | Queue + observability + multi-tenancy + dashboards | Hatchet, Inngest |
| **Engine** | Provides a fundamentally different execution model (e.g., durable execution). | Workflow state + replay + deterministic execution | Temporal, Restate, DBOS |
| **Managed Service** | Cloud-hosted, fully managed. Abstracts infrastructure entirely. | Everything including infra | AWS Step Functions, Anthropic Managed Agents |

### 0.2 Responsibility Boundary

Concerns in this spec are divided into:

- **Infrastructure concerns** — must be answered by the worker service itself
- **Application concerns** — may be answered by the worker service OR delegated to the application built on top
- **Product concerns** — only relevant when the worker service IS the product (SaaS)

Each concern is tagged accordingly. A library that delegates observability to the application is not deficient — it's operating within its responsibility boundary.

---

## 1. Work Unit Model

**Tag:** Infrastructure concern

**Concern:** How does the system represent, organize, and identify units of work?

### Requirements

| Requirement | Description |
| --- | --- |
| **Granularity is defined** | The system has a clear smallest unit of work, whatever it's called (job, task, activity, message, step, function). |
| **Composition model is explicit** | The system documents whether work units are flat, hierarchical (parent-child), graph-based (DAG), or pipeline-based — or whether composition is not supported. |
| **Instances are distinguishable from definitions** | There is a way to differentiate "what to do" (the template/type) from "a specific execution of it" (the instance/run). This separation may be explicit (registered workflow + run) or implicit (queue name + job). |
| **Payloads have defined boundaries** | Input and output data has a defined format, size constraints, and serialization method. Large payloads have a documented strategy (inline, reference, external storage). |
| **Work is addressable** | Each work unit instance has a unique identifier that allows external systems to query its status, cancel it, or reference it. |

### Valid Patterns

| Pattern | How It Works | Examples |
| --- | --- | --- |
| Flat jobs | Single-level work units, no composition | BullMQ jobs, Celery tasks |
| Parent-child | Work units spawn children, parent waits for children | BullMQ Flows, Hatchet workflows |
| DAG | Steps with explicit dependency edges | Hatchet DAG tasks, Airflow |
| Workflow-as-code | Orchestration logic written as imperative code, framework manages execution | Temporal, Azure Durable Functions |
| Event-driven pipeline | Work units trigger downstream work via events | Kafka consumers, event sourcing |
| Saga | Long-running transaction with compensating actions on failure | Temporal sagas, custom saga orchestrators |

---

## 2. State Management

**Tag:** Infrastructure concern

**Concern:** How does the system track where each unit of work is in its lifecycle, and how durable is that tracking?

### Requirements

| Requirement | Description |
| --- | --- |
| **Lifecycle is well-defined** | The system documents all possible states (or equivalent progression markers) a work unit can be in. |
| **Progression is enforced** | The system prevents or detects invalid state changes. Backward movement is either impossible or explicitly modeled (e.g., retry). |
| **Durability guarantees are documented** | The system states what happens to work-in-progress if the system crashes. Can work be recovered? Under what conditions? What is the blast radius? |
| **State is queryable** | External callers can determine the current status of a work unit. The consistency model (strong, eventual) is documented. |
| **History depth is defined** | The system documents whether it preserves only current state, recent history, or full history — and for how long. |

### Valid Patterns

| Pattern | How It Works | Tradeoffs |
| --- | --- | --- |
| Explicit state enum | Work units have named states (queued, running, completed, failed) with defined transitions | Simple to reason about, easy to query. Can become rigid as complexity grows. |
| Implicit state via execution position | State is the current position in executing code. Framework checkpoints automatically. | More expressive, no state machine to maintain. Requires deterministic replay. |
| Event-sourced state | Append-only log of events. Current state derived by replaying events. | Full history by default. Replay cost grows with history length. |
| Hybrid | Explicit states for high-level lifecycle + detailed event log for history | Combines queryability with auditability. More complex to implement. |

### Durability Spectrum

| Level | What Survives | Examples |
| --- | --- | --- |
| **None** | Nothing survives process restart | In-memory queues |
| **Conditional** | Survives restart IF persistence is properly configured | Redis with AOF/RDB (BullMQ) |
| **Guaranteed** | All committed state survives any single-node failure by default | Postgres-backed (Hatchet), Temporal event history |
| **Replicated** | Survives regional failure | Temporal multi-cluster, replicated databases |

A system must document where it falls on this spectrum. "Durable" without qualification is insufficient.

---

## 3. Idempotency & Duplicate Prevention

**Tag:** Infrastructure + Application concern

**Concern:** How does the system prevent the same work from being performed more than once, and who is responsible for ensuring safety?

### Requirements

| Requirement | Description |
| --- | --- |
| **Submission deduplication is addressed** | The system documents what happens when the same work is submitted twice. Either the system deduplicates, or the application must. |
| **Execution safety is addressed** | The system documents what happens when a work unit is retried after a partial completion. Either the system prevents re-execution of completed steps, or the application must make handlers idempotent. |
| **Side-effect responsibility is assigned** | The system documents who is responsible for ensuring external side effects (API calls, DB writes, emails) are safe across retries — the platform or the application developer. |
| **Delivery guarantee is documented** | The system states its guarantee: at-most-once, at-least-once, or effectively-once. The tradeoffs are documented. |

### Responsibility Spectrum

This is the critical insight the previous spec missed. Idempotency responsibility exists on a spectrum:

| Level | Platform Responsibility | Application Responsibility | Examples |
| --- | --- | --- | --- |
| **Platform-guaranteed** | Platform ensures completed work is never re-executed. Replay engine skips completed steps. | Application writes activities that may be called more than once but the platform prevents duplicate orchestration. | Temporal (workflow replay), Restate |
| **Platform-assisted** | Platform provides dedup keys, exactly-once delivery attempts, or idempotency infrastructure. | Application must use the provided mechanisms correctly. | Hatchet (dedup keys), SQS (message dedup ID) |
| **Application-only** | Platform provides at-least-once delivery. No built-in dedup. | Application must make every handler idempotent. | BullMQ, Celery, raw Kafka consumers |

A spec must state which level it targets and document the contract between platform and application.

---

## 4. Failure Handling

**Tag:** Infrastructure concern

**Concern:** How does the system categorize failures, respond to them, and surface permanently failing work?

### Requirements

| Requirement | Description |
| --- | --- |
| **Failure categories exist** | The system distinguishes between failures that should be retried and failures that should not. The mechanism may be explicit (typed errors) or implicit (retry count exhaustion). |
| **Retry behavior is configurable** | The system allows configuration of retry count, delay strategy, and maximum backoff. Configuration granularity (global, per-queue, per-work-unit) is documented. |
| **Exhaustion has a defined outcome** | When retries are exhausted, the system has a documented behavior. The work unit must not silently disappear. |
| **Permanently failing work is visible** | Work that cannot succeed is identifiable and inspectable. Operators can find it, understand why it failed, and decide what to do. |
| **Partial completion is addressed** | For composed work (workflows, DAGs), the system documents what happens when some steps succeed and others fail. |
| **Dependency failure is addressed** | The system documents its behavior when downstream systems are unavailable. |

### Valid Exhaustion Patterns

| Pattern | How It Works | Examples |
| --- | --- | --- |
| Dead letter queue/table | Exhausted work moves to a separate holding area | SQS DLQ, custom Postgres DLQ table |
| Failed state retention | Work stays in place with a "failed" status, queryable and replayable | Temporal (failed workflows), Hatchet (failed tasks in dashboard) |
| Callback/webhook | System notifies an external endpoint on exhaustion | Webhook on final failure |
| Error escalation | Failure propagates to parent workflow, which decides the response | Temporal (try/catch in workflow code), Hatchet (workflow-level failure) |

The previous spec assumed DLQ was the universal pattern. It is one valid pattern among several.

---

## 5. Scheduling & Dispatch

**Tag:** Infrastructure concern

**Concern:** How does work get assigned to workers, and in what order?

### Requirements

| Requirement | Description |
| --- | --- |
| **Dispatch mechanism is defined** | The system documents how work reaches workers — push, pull, or hybrid. |
| **Ordering behavior is documented** | The system states whether ordering is guaranteed (FIFO, priority-based) or unordered, and at what scope (global, per-queue, per-tenant). |
| **Deferred execution is supported or explicitly excluded** | The system documents whether work can be scheduled for future execution (delayed jobs, cron, timers). If not supported, this is stated. |
| **Execution rate can be controlled** | The system provides mechanisms to limit how fast work is dispatched — globally, per-queue, per-tenant, or per-key. If not built-in, the application-level approach is documented. |
| **Concurrent execution is bounded** | The system has a mechanism to limit how many work units execute simultaneously. The scope of these limits is documented. |

### Valid Dispatch Patterns

| Pattern | How It Works | Examples |
| --- | --- | --- |
| Worker polling (pull) | Workers long-poll a queue/broker for available work | Temporal task queues, BullMQ Redis polling |
| Broker push | Broker delivers work to connected workers | Hatchet gRPC dispatch, RabbitMQ |
| Event-triggered | Work is triggered by an event (HTTP, message, file change) | Lambda, Cloud Functions, webhook-triggered jobs |
| Polling + reconciliation | Orchestrator polls an external source and dispatches work based on state | Symphony (polls Linear), custom schedulers |

---

## 6. Backpressure & Flow Control

**Tag:** Infrastructure concern (platform) / Application concern (library)

**Concern:** How does the system behave when work is submitted faster than it can be processed?

### Requirements

| Requirement | Description |
| --- | --- |
| **Overload behavior is defined** | The system documents what happens under sustained overload — does it queue unboundedly, reject, drop, or throttle? |
| **Producers can detect overload** | Either the system signals overload to producers (explicit), or producers can detect it via metrics (implicit). The mechanism is documented. |
| **Fairness model is documented** | For multi-tenant or multi-source systems, the system documents whether one source can starve others and what mechanisms prevent it. For single-tenant systems, this is marked N/A with justification. |
| **Worker capacity is bounded** | Workers do not accept more work than they can handle. The mechanism (slot-based, semaphore, connection-based) is documented. |

### Valid Backpressure Patterns

| Pattern | How It Works | Examples |
| --- | --- | --- |
| Explicit rejection | System returns an error (429, NACK) when capacity is exceeded | API rate limiting, RabbitMQ NACK |
| Implicit queueing | Work queues in a buffer. Backpressure manifests as increased latency. | Temporal (schedule-to-start latency), BullMQ (Redis queue growth) |
| Worker-controlled intake | Workers only pull work they can handle. Producers are unaware of load. | Temporal workers, BullMQ concurrency setting |
| Tenant-scoped throttling | Per-tenant rate limits prevent any single tenant from dominating | Hatchet concurrency keys, custom rate limiters |

---

## 7. Timeout & Liveness

**Tag:** Infrastructure concern

**Concern:** How does the system detect and recover from work that takes too long or workers that disappear?

### Requirements

| Requirement | Description |
| --- | --- |
| **Maximum execution duration is enforceable** | Work units have a configurable time limit. Exceeding it triggers a defined response (failure, retry, alert). |
| **Abandoned work is recoverable** | If a worker crashes or disconnects mid-execution, the system detects this and makes the work available again within a bounded time. |
| **Long-running work can signal liveness** | For work that legitimately takes a long time, there is a mechanism for the worker to signal that it is still making progress. Absence of this signal is treated as a failure. |
| **Cascading deadlines are addressed** | For composed work, the system documents whether a top-level deadline applies to child work units. If not supported, this is stated. |

---

## 8. Observability

**Tag:** Infrastructure concern (platform) / Application concern (library)

**Concern:** How do operators understand what the system is doing, detect problems, and debug failures?

### Requirements

| Requirement | Description |
| --- | --- |
| **Work is traceable** | Each work unit's execution can be followed from submission to completion, including retries, errors, and state changes. The mechanism may be built-in (dashboard) or delegated to the application (structured log fields). |
| **System health is measurable** | Key health indicators (throughput, latency, error rate, queue depth/backlog) are either emitted by the system or derivable from its APIs. |
| **Failures are surfaceable** | When work fails, operators can discover it without polling. The mechanism may be alerts, dashboards, log patterns, or callbacks. |
| **Debugging is possible** | For a specific failed work unit, an operator can inspect its inputs, outputs, errors, retry history, and execution timeline. |

### Responsibility by System Type

| System Type | Expected Observability |
| --- | --- |
| Library | Emits events/hooks. Application integrates with its own monitoring. |
| Platform | Built-in dashboard, alerting, and searchable execution history. |
| Engine | Full execution replay, event history browser, query APIs. |
| Managed Service | Cloud console, integrated metrics, managed alerting. |

A library that emits structured events but has no dashboard is fulfilling its observability contract. A platform without a dashboard is not.

---

## 9. Isolation

**Tag:** Application concern (may be delegated to the application layer)

**Concern:** How does the system prevent one workload from affecting another — whether those workloads represent different tenants, different priorities, or different environments?

### Requirements

| Requirement | Description |
| --- | --- |
| **Data separation is addressed** | The system documents how data from different workloads is separated — logically (row-level, key prefix), structurally (namespace, queue), or physically (database, cluster). |
| **Resource contention is addressed** | The system documents whether one workload can consume resources (CPU, memory, queue capacity, worker slots) at the expense of another, and what prevents it. |
| **Workload identity is associable** | Work units can carry metadata (tenant ID, environment, priority class) that enables filtering, querying, and policy enforcement. |
| **Lifecycle independence is addressed** | The system documents what happens to in-flight work when a workload context is modified (tenant suspended, environment scaled down, priority changed). |

### Why "Isolation" Instead of "Multi-Tenancy"

The previous spec used "Multi-Tenancy" which presumes a SaaS product with tenants. The actual concern is broader — any system needs isolation between workloads, whether those are tenants, environments (prod vs staging), priority classes, or customer segments. Single-tenant systems still need isolation between workload types.

---

## 10. Security

**Tag:** Infrastructure concern + Application concern

**Concern:** How is the system protected from unauthorized access, data exposure, and malicious use?

### Requirements

| Requirement | Description |
| --- | --- |
| **Callers are authenticated** | The system documents how producers and consumers prove their identity. The mechanism matches the system type (API keys for platforms, network-level for libraries, IAM for cloud services). |
| **Actions are authorized** | The system documents what permissions exist and how they are enforced. At minimum: who can submit work, who can cancel it, who can inspect it. |
| **Sensitive data is protected** | The system documents how sensitive data in payloads is handled — encryption at rest, in transit, or delegated to the application. |
| **Credentials are managed** | The system documents how workers access credentials for downstream services. Credentials must not be exposed to untrusted code executing within the worker. |
| **Administrative actions are auditable** | Significant actions (work submission, cancellation, configuration changes) are logged with actor identity and timestamp. |

---

## 11. Data Lifecycle

**Tag:** Application concern (product-level when the worker service is a SaaS)

**Concern:** How does the system manage data growth, retention, and deletion over time?

### Requirements

| Requirement | Description |
| --- | --- |
| **Retention policy exists** | Completed work and its associated data have a defined lifespan. The system either enforces it automatically or provides mechanisms for the application to enforce it. |
| **Growth is bounded** | The system documents how it prevents unbounded data growth. Mechanisms may include TTL, max history size, or archival. |
| **Data can be removed** | Work history and payloads can be deleted — either per-workload-context, per-time-range, or globally. The completeness of deletion (including derived data, indexes, logs) is documented. |
| **Regulatory requirements are addressable** | For systems handling user data, the spec documents whether GDPR-style deletion requests and data export can be fulfilled. If the system delegates this to the application, that boundary is documented. |

---

## 12. Usage Accounting

**Tag:** Product concern (only when the worker service IS the product or directly supports billing)

**Concern:** How is system usage tracked, reported, and optionally billed?

### Applicability

This concern is only relevant when:
- The worker service is offered as a product/platform (SaaS)
- The worker service directly supports usage-based billing for end customers
- Regulatory or internal chargeback requires usage attribution

For infrastructure libraries (BullMQ, Celery) and internal-only services, this concern is **N/A**.

### Requirements (When Applicable)

| Requirement | Description |
| --- | --- |
| **Usage dimensions are defined** | What is measured — executions, compute time, data volume, API calls, or other metrics. |
| **Attribution is accurate** | Usage can be attributed to a specific workload context (tenant, team, project) with documented accuracy. |
| **Metering does not block execution** | Usage tracking must not be in the critical path of work execution. Acceptable loss rate is documented. |
| **Usage limits are enforceable** | If the system enforces quotas, it documents whether enforcement happens before execution (reject at submission) or after (alert/throttle). |

---

## 13. Operational Lifecycle

**Tag:** Infrastructure concern

**Concern:** How is the system deployed, updated, scaled, and recovered?

### Requirements

| Requirement | Description |
| --- | --- |
| **Updates preserve in-flight work** | The system documents how deployments affect running work. Zero-downtime update paths are documented, or the expected disruption is stated. |
| **Capacity can scale** | The system documents how to increase processing capacity — adding workers, scaling infrastructure, or adjusting configuration. The scaling model (horizontal, vertical, serverless) is documented. |
| **Failures can be recovered** | The system documents its recovery plan for total failure. Recovery Point Objective (how much data can be lost) and Recovery Time Objective (how long until recovery) are stated or estimable. |
| **Changes can be reversed** | The system documents how to roll back a bad deployment. The impact on in-flight and recently completed work is documented. |

---

## 14. Definition Versioning & Evolution

**Tag:** Infrastructure concern

**Concern:** How do work definitions (handlers, workflows, task logic) change over time without breaking in-flight work?

This was a missing concern in the previous spec, identified during evaluation.

### Requirements

| Requirement | Description |
| --- | --- |
| **Version coexistence is addressed** | The system documents whether old and new versions of work definitions can run simultaneously. If not, the migration path is documented. |
| **In-flight work compatibility is addressed** | The system documents what happens to work that was submitted under an old definition when a new definition is deployed. Does it complete under old logic, switch to new logic, or fail? |
| **Schema evolution is addressed** | The system documents how changes to input/output schemas (new fields, removed fields, type changes) affect existing work and historical data. |

### Valid Patterns

| Pattern | How It Works | Examples |
| --- | --- | --- |
| Workflow versioning | Platform routes work to the correct handler version based on when it was created | Temporal workflow versioning |
| Queue versioning | Different versions use different queues, drained separately | BullMQ queue-per-version |
| Handler compatibility | Handlers are written to accept old and new schemas simultaneously | Backward-compatible handlers, additive-only schema changes |
| Blue-green deployment | Old and new versions run simultaneously, traffic shifts gradually | Kubernetes rolling deployments |

---

## 15. Composability & Extensibility

**Tag:** Infrastructure concern

**Concern:** How does the system integrate with other systems and adapt to requirements it wasn't originally designed for?

This was implicit but never explicitly stated in the previous spec.

### Requirements

| Requirement | Description |
| --- | --- |
| **External systems can trigger work** | Work can be submitted via documented APIs, events, webhooks, or other integration points. |
| **Work completion can notify external systems** | When work completes (or fails), external systems can be notified via callbacks, events, webhooks, or polling. |
| **The system can be extended** | Custom behavior (pre/post-processing hooks, custom retry logic, middleware) can be added without modifying the core system. |
| **The system can be replaced** | The interfaces are documented well enough that a different implementation could be substituted behind the same contracts. |

---

## How to Use This Spec

### For Building a New System

1. Walk through each concern and decide: **will we address this, delegate it, or mark it N/A?**
2. For each addressed concern, pick a pattern from the valid patterns.
3. Document your choices as an implementation spec (Layer 3).
4. Revisit when your system type changes (e.g., graduating from library to platform).

### For Evaluating an Existing System

1. Classify the system (Section 0).
2. Walk through each concern. Score as:
   - ✅ **Addressed** — clear answer exists
   - ⚠️ **Partially addressed** — answer exists but with gaps
   - ❌ **Not addressed** — no answer and not delegated
   - **Delegated** — explicitly assigned to the application layer (valid for libraries)
   - **N/A** — not applicable (must justify)
3. Compare scores, noting that different system types have different expectations.

### For Comparing Systems

1. Fill out the evaluation for each system independently.
2. Differences reveal **architectural tradeoffs**, not deficiencies.
3. ❌ scores reveal **blind spots**.
4. Delegated scores reveal **responsibility boundaries** — a library that delegates observability is making a different tradeoff than a platform that owns it.

---

## Changelog From v2

| Change | Reason |
| --- | --- |
| Added Section 0 (System Classification) | Previous spec penalized libraries for not being platforms |
| Tagged each concern with responsibility boundary | Clarifies what the system must own vs may delegate |
| Reframed State Management to include implicit state models | Previous spec assumed explicit state enums, missed Temporal's workflow-as-code |
| Replaced "Multi-Tenancy" with "Isolation" | Broader concern — applies to environments, priority classes, not just SaaS tenants |
| Reframed idempotency as a responsibility spectrum | Previous spec assumed application-only idempotency, missed platform-guaranteed models |
| Generalized exhaustion patterns beyond DLQ | DLQ is one pattern, not the universal answer |
| Reframed Billing/Metering as "Usage Accounting" with applicability gate | All three evaluated systems scored ❌ because it's a product concern, not infrastructure |
| Added Section 14 (Definition Versioning) | Identified as missing during Temporal/Hatchet/BullMQ evaluation |
| Added Section 15 (Composability & Extensibility) | Was implicit but never stated |
| Removed all SQL schemas, specific technology names, and implementation patterns from requirements | Requirements now state WHAT, patterns state HOW, implementations state WITH WHAT |
| Added "Delegated" as a valid evaluation score | Libraries intentionally delegate concerns — this isn't a gap |
| Added Durability Spectrum to State Management | "Durable" without qualification is insufficient — conditional vs guaranteed vs replicated |