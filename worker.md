# Worker Service Specification

**Version:** 1.0
**Status:** Draft for Public Review

> **Principle:** This specification defines WHAT a worker service must address, not HOW. It separates universal concerns from architectural patterns from implementation choices. Each concern states the requirement, describes valid approaches, and optionally illustrates with examples.
>
> **Scope:** This specification applies to any system that accepts units of work and executes them asynchronously — whether that system is a library, a platform, a durable execution engine, a managed service, or a custom-built service.

---

## How This Specification Is Structured

| Layer | What It Contains | Changes Between Systems? |
| --- | --- | --- |
| **Concern** | The question every system must answer | No — universal |
| **Patterns** | The known valid approaches to answering it | Rarely — architectural |
| **Implementation** | The specific technology choice | Always — per system |

This specification contains the first two layers. Implementations fill in the third.

---

## 0. System Classification

Before evaluating or building any worker service, classify it. Different categories carry different expectations for which concerns the system owns versus delegates.

### 0.1 System Type

| Type | Description | Scope of Responsibility | Examples |
| --- | --- | --- | --- |
| **Library** | Embedded in the application process. Provides primitives. Application owns policy. | Queue, retry, dispatch | BullMQ, Celery, Sidekiq |
| **Platform** | Standalone service with UI, API, and built-in policies. | Queue + observability + isolation + dashboards | Hatchet, Inngest |
| **Engine** | Provides a fundamentally different execution model (e.g., durable execution). | Workflow state + replay + deterministic execution | Temporal, Restate, DBOS |
| **Managed Service** | Cloud-hosted, fully managed. Abstracts infrastructure entirely. | Everything including infrastructure | AWS Step Functions, Azure Durable Functions |

### 0.2 Responsibility Boundary

Each concern is tagged with one of three responsibility levels:

- **Infrastructure** — the worker service itself must answer this
- **Application** — the worker service may answer this or explicitly delegate it to the application layer
- **Product** — only relevant when the worker service directly supports end-user billing, internal chargeback, or regulatory attribution

A library that delegates observability to the application is not deficient — it is operating within its responsibility boundary.

### 0.3 Scoring

When evaluating a system against this specification, score each concern as:

| Score | Meaning |
| --- | --- |
| ✅ Addressed | Clear, documented answer exists |
| ⚠️ Partially addressed | Answer exists but has gaps or known limitations |
| ❌ Not addressed | No answer and not delegated |
| 🔀 Delegated | Explicitly assigned to the application layer (valid for libraries) |
| N/A | Not applicable to this system's use case (must justify) |

**Interpreting Delegated scores:** A high count of Delegated scores is not a deficiency in the system — it is a signal that the application team inherits significant engineering burden. When selecting a system, evaluate whether the team has the capacity and expertise to own those delegated concerns. A library with seven Delegated scores may be the right choice for a team with strong infrastructure skills, and the wrong choice for a team that needs turnkey solutions.

---

## 1. Work Unit Model

**Tag:** Infrastructure

**Concern:** How does the system represent, organize, and identify units of work?

### Requirements

| Requirement | Description |
| --- | --- |
| **Granularity is defined** | The system has a clear, documented smallest unit of work, whatever it is called. |
| **Composition model is explicit** | The system documents whether work units are flat, hierarchical, graph-based, pipeline-based, or whether composition is not supported. |
| **Instances are distinguishable from definitions** | There is a way to differentiate "what to do" from "a specific execution of it." This separation may be explicit or implicit. |
| **Payloads have defined boundaries** | Input and output data has a documented format, size constraints, and serialization method. Large payloads have a documented strategy. |
| **Work is addressable** | Each work unit instance has a unique identifier that allows external systems to query, cancel, or reference it. |

### Valid Patterns

| Pattern | How It Works |
| --- | --- |
| Flat jobs | Single-level work units, no composition |
| Parent-child | Work units spawn children; parent waits for children |
| DAG | Steps with explicit dependency edges |
| Workflow-as-code | Orchestration logic written as imperative code; framework manages execution |
| Event-driven pipeline | Work units trigger downstream work via events |
| Saga | Long-running transaction with compensating actions on failure |

---

## 2. State Management

**Tag:** Infrastructure

**Concern:** How does the system track where each unit of work is in its lifecycle, and how durable is that tracking?

### Requirements

| Requirement | Description |
| --- | --- |
| **Lifecycle is well-defined** | The system documents all possible states or equivalent progression markers a work unit can be in. |
| **Progression is enforced** | The system prevents or detects invalid state changes. Backward movement is either impossible or explicitly modeled. |
| **Durability guarantees are documented** | The system states what happens to work-in-progress if the system crashes. The durability level, conditions, and blast radius are documented. |
| **State is queryable** | External callers can determine the current status of a work unit. The consistency model is documented. |
| **History depth is defined** | The system documents whether it preserves only current state, recent history, or full history — and for how long. |

### Valid State Representation Patterns

| Pattern | How It Works | Tradeoffs |
| --- | --- | --- |
| Explicit state enum | Named states with defined transitions | Simple to reason about and query. Can become rigid as complexity grows. |
| Implicit state via execution position | State is the current position in executing code. Framework checkpoints automatically. | More expressive. Requires deterministic replay. |
| Event-sourced | Append-only log of events. Current state derived by replaying events. | Full history by default. Replay cost grows with history length. |
| Hybrid | Explicit high-level lifecycle states combined with a detailed event log | Combines queryability with auditability. |

### Durability Spectrum

Systems must document where they fall on this spectrum. "Durable" without qualification is insufficient.

| Level | What Survives | Conditions |
| --- | --- | --- |
| **None** | Nothing survives process restart | In-memory only |
| **Conditional** | Survives restart if persistence is properly configured | Requires specific configuration |
| **Guaranteed** | All committed state survives any single-node failure by default | No additional configuration required |
| **Replicated** | Survives regional or multi-node failure | Multi-cluster or replicated storage |

---

## 3. Idempotency & Duplicate Prevention

**Tag:** Infrastructure + Application

**Concern:** How does the system prevent the same work from being performed more than once, and who is responsible for ensuring safety?

### Requirements

| Requirement | Description |
| --- | --- |
| **Submission deduplication is addressed** | The system documents what happens when identical work is submitted twice. |
| **Execution safety is addressed** | The system documents what happens when a work unit is retried after partial completion. |
| **Side-effect responsibility is assigned** | The system documents who is responsible for ensuring external side effects are safe across retries — the platform or the application developer. |
| **Delivery guarantee is documented** | The system states its guarantee level and the associated tradeoffs. |

### Responsibility Spectrum

| Level | Platform Responsibility | Application Responsibility |
| --- | --- | --- |
| **Platform-guaranteed** | Platform ensures completed work is never re-executed via replay or deduplication | Application writes handlers that may be invoked more than once, but the platform prevents duplicate orchestration |
| **Platform-assisted** | Platform provides deduplication keys, exactly-once delivery attempts, or idempotency infrastructure | Application must use the provided mechanisms correctly |
| **Application-only** | Platform provides at-least-once delivery with no built-in deduplication | Application must make every handler idempotent |

A specification must state which level it targets and document the contract between platform and application.

---

## 4. Failure Handling

**Tag:** Infrastructure

**Concern:** How does the system categorize failures, respond to them, and surface permanently failing work?

### Requirements

| Requirement | Description |
| --- | --- |
| **Failure categories exist** | The system distinguishes between failures that should be retried and failures that should not. The mechanism may be explicit or implicit. |
| **Retry behavior is configurable** | Retry count, delay strategy, and maximum backoff are configurable. Configuration granularity is documented. |
| **Exhaustion has a defined outcome** | When retries are exhausted, the system has a documented behavior. Work must not silently disappear. |
| **Permanently failing work is visible** | Work that cannot succeed is identifiable, inspectable, and actionable by operators. |
| **Partial completion is addressed** | For composed work, the system documents what happens when some steps succeed and others fail. |
| **Dependency failure is addressed** | The system documents its behavior when downstream systems are unavailable. |

### Valid Exhaustion Patterns

| Pattern | How It Works |
| --- | --- |
| Dead letter queue/table | Exhausted work moves to a separate holding area for inspection and replay |
| Failed state retention | Work stays in place with a terminal failure status, queryable and replayable |
| Callback/webhook | System notifies an external endpoint on exhaustion |
| Error escalation | Failure propagates to a parent or orchestrator, which decides the response |

---

## 5. Scheduling & Dispatch

**Tag:** Infrastructure

**Concern:** How does work get assigned to workers, and in what order?

### Requirements

| Requirement | Description |
| --- | --- |
| **Dispatch mechanism is defined** | The system documents how work reaches workers. |
| **Ordering behavior is documented** | The system states whether ordering is guaranteed and at what scope. |
| **Deferred execution is supported or explicitly excluded** | The system documents whether work can be scheduled for future execution. |
| **Execution rate can be controlled** | The system provides mechanisms to limit dispatch rate, or documents how the application should implement rate control. |
| **Concurrent execution is bounded** | The system has a mechanism to limit simultaneous work unit execution. The scope of these limits is documented. |
| **Dependency ordering is addressed** | When work units have causal dependencies (B requires the result of A), the system documents how it guarantees A completes before B starts. If the system does not manage dependencies, this is stated. |

### Valid Dispatch Patterns

| Pattern | How It Works |
| --- | --- |
| Worker polling (pull) | Workers request available work from a queue or broker |
| Broker push | Broker delivers work to connected workers |
| Event-triggered | Work is triggered by an external event |
| Polling + reconciliation | Orchestrator polls an external source and dispatches work based on observed state |

---

## 6. Backpressure & Flow Control

**Tag:** Infrastructure (platform/engine) · Application (library)

**Concern:** How does the system behave when work is submitted faster than it can be processed?

### Requirements

| Requirement | Description |
| --- | --- |
| **Overload behavior is defined** | The system documents what happens under sustained overload. |
| **Producers can detect overload** | Either the system signals overload to producers explicitly, or producers can detect it via observable indicators. The mechanism is documented. |
| **Fairness model is documented** | For systems serving multiple workload sources, the system documents whether one source can starve others and what mechanisms prevent it. For single-source systems, this is marked N/A with justification. |
| **Worker capacity is bounded** | Workers do not accept more work than they can handle. The mechanism is documented. |

### Valid Backpressure Patterns

| Pattern | How It Works |
| --- | --- |
| Explicit rejection | System returns an error when capacity is exceeded |
| Implicit queueing | Work buffers in a queue; backpressure manifests as increased latency |
| Worker-controlled intake | Workers only pull work they can handle; producers are unaware of load |
| Tenant-scoped throttling | Per-source or per-tenant rate limits prevent any single source from dominating |

---

## 7. Timeout & Liveness

**Tag:** Infrastructure

**Concern:** How does the system detect and recover from work that takes too long or workers that disappear?

### Requirements

| Requirement | Description |
| --- | --- |
| **Maximum execution duration is enforceable** | Work units have a configurable time limit. Exceeding it triggers a defined response. |
| **Abandoned work is recoverable** | If a worker crashes or disconnects mid-execution, the system detects this and makes the work available again within a bounded time. |
| **Long-running work can signal liveness** | For work that legitimately takes a long time, there is a mechanism for the worker to signal progress. Absence of this signal is treated as a failure. |
| **Cascading deadlines are addressed** | For composed work, the system documents whether a top-level deadline applies to child work units. |

---

## 8. Observability

**Tag:** Infrastructure (platform/engine) · Application (library)

**Concern:** How do operators understand what the system is doing, detect problems, and debug failures?

### Requirements

| Requirement | Description |
| --- | --- |
| **Work is traceable** | Each work unit's execution can be followed from submission to completion, including retries and errors. |
| **System health is measurable** | Key health indicators (throughput, latency, error rate, backlog) are either emitted by the system or derivable from its APIs. |
| **Failures are surfaceable** | When work fails, operators can discover it without polling. |
| **Debugging is possible** | For a specific failed work unit, an operator can inspect its inputs, outputs, errors, retry history, and execution timeline. |

### Expectations by System Type

| System Type | Expected Observability |
| --- | --- |
| Library | Emits events and hooks. Application integrates with its own monitoring stack. |
| Platform | Built-in dashboard, alerting, and searchable execution history. |
| Engine | Full execution replay, event history inspection, and query APIs. |
| Managed Service | Cloud console, integrated metrics, and managed alerting. |

---

## 9. Isolation

**Tag:** Infrastructure + Application

**Concern:** How does the system prevent one workload from affecting another?

This applies to any dimension of separation: tenants, environments, priority classes, customer segments, or workload types. Single-tenant systems still require isolation between workload categories. The system must address isolation at the infrastructure level; additional application-level isolation may be layered on top.

### Requirements

| Requirement | Description |
| --- | --- |
| **Data separation is addressed** | The system documents how data from different workloads is separated. |
| **Resource contention is addressed** | The system documents whether one workload can consume resources at the expense of another, and what prevents it. |
| **Workload identity is associable** | Work units can carry metadata that enables filtering, querying, and policy enforcement. |
| **Lifecycle independence is addressed** | The system documents what happens to in-flight work when a workload context is modified. |

---

## 10. Security

**Tag:** Infrastructure + Application

**Concern:** How is the system protected from unauthorized access, data exposure, and malicious use?

### Requirements

| Requirement | Description |
| --- | --- |
| **Callers are authenticated** | The system documents how producers and consumers prove their identity. |
| **Actions are authorized** | The system documents what permissions exist and how they are enforced. At minimum: who can submit, cancel, and inspect work. |
| **Sensitive data is protected** | The system documents how sensitive data in payloads is handled. |
| **Credentials are managed** | The system documents how workers access credentials for downstream services. Credentials must not be exposed to untrusted code executing within the worker. |
| **Administrative actions are auditable** | Significant actions are logged with actor identity and timestamp. |

---

## 11. Data Lifecycle

**Tag:** Application

**Concern:** How does the system manage data growth, retention, and deletion over time?

### Requirements

| Requirement | Description |
| --- | --- |
| **Retention policy exists** | Completed work and its associated data have a defined lifespan. |
| **Growth is bounded** | The system documents how it prevents unbounded data growth. |
| **Data can be removed** | Work history and payloads can be deleted. The completeness of deletion is documented. |
| **Regulatory requirements are addressable** | For systems handling user data, the specification documents whether data subject requests (deletion, export) can be fulfilled. If delegated, that boundary is documented. |

---

## 12. Usage Accounting

**Tag:** Product

**Concern:** How is system usage tracked, attributed, and reported?

### Applicability

This concern applies when:

- The worker service is offered as a product or platform
- Usage-based billing is required for end customers
- Internal cost attribution or chargeback is required between teams, departments, or workload owners
- Regulatory or compliance requirements demand usage attribution

For purely internal systems without cost attribution requirements, this concern is N/A.

### Requirements (When Applicable)

| Requirement | Description |
| --- | --- |
| **Usage dimensions are defined** | What is measured — executions, compute time, data volume, API calls, or other metrics. |
| **Attribution is accurate** | Usage can be attributed to a specific workload context with documented accuracy. |
| **Metering does not block execution** | Usage tracking must not be in the critical path of work execution. Acceptable loss rate is documented. |
| **Usage limits are enforceable** | If the system enforces quotas, it documents whether enforcement happens before execution or after. |

---

## 13. Operational Lifecycle

**Tag:** Infrastructure

**Concern:** How is the system deployed, updated, scaled, and recovered?

### Requirements

| Requirement | Description |
| --- | --- |
| **Updates preserve in-flight work** | The system documents how deployments affect running work. |
| **Capacity can scale** | The system documents how to increase or decrease processing capacity. |
| **Failures can be recovered** | The system documents its recovery plan for total failure. Recovery Point Objective and Recovery Time Objective are stated or estimable. |
| **Changes can be reversed** | The system documents how to roll back a bad deployment and the impact on in-flight work. |

---

## 14. Definition Versioning & Evolution

**Tag:** Infrastructure

**Concern:** How do work definitions change over time without breaking in-flight work?

### Requirements

| Requirement | Description |
| --- | --- |
| **Version coexistence is addressed** | The system documents whether old and new versions of work definitions can run simultaneously. If not, the migration path is documented. |
| **In-flight work compatibility is addressed** | The system documents what happens to work submitted under an old definition when a new definition is deployed. |
| **Schema evolution is addressed** | The system documents how changes to input/output schemas affect existing work and historical data. |

### Valid Patterns

| Pattern | How It Works |
| --- | --- |
| Workflow versioning | Platform routes work to the correct handler version based on when it was created |
| Queue versioning | Different versions use different queues, drained separately |
| Handler compatibility | Handlers are written to accept old and new schemas simultaneously |
| Blue-green deployment | Old and new versions run simultaneously; traffic shifts gradually |

---

## 15. Composability & Extensibility

**Tag:** Infrastructure

**Concern:** How does the system integrate with other systems and adapt to new requirements?

### Requirements

| Requirement | Description |
| --- | --- |
| **External systems can trigger work** | Work can be submitted via documented APIs, events, webhooks, or other integration points. |
| **Work completion can notify external systems** | When work completes or fails, external systems can be notified via callbacks, events, webhooks, or polling. |
| **The system can be extended** | Custom behavior (hooks, middleware, custom retry logic) can be added without modifying the core system. |
| **The system can be replaced** | The interfaces are documented well enough that a different implementation could be substituted behind the same contracts. |

---

## 16. Performance Envelope

**Tag:** Infrastructure

**Concern:** Are the system's performance characteristics documented so that teams can make informed selection and capacity planning decisions?

### Requirements

| Requirement | Description |
| --- | --- |
| **Throughput limits are documented** | The system states its expected throughput range under normal operation, including any known ceilings or scaling inflection points. |
| **Latency profiles are documented** | The system documents expected latency for key operations (submission, dispatch, completion notification) under normal and peak load. |
| **Resource scaling characteristics are documented** | The system documents how performance changes as load increases — linear, sublinear, or degrading — and what the primary bottlenecks are. |
| **Known limitations are stated** | The system documents scenarios where it is not a good fit, including workload types, scale thresholds, or architectural constraints that would require a different tool. |

### Valid Documentation Approaches

| Approach | How It Works |
| --- | --- |
| Benchmark suites | Published, reproducible benchmarks with defined workload profiles and hardware specifications |
| Load test results | Results from realistic load tests showing behavior at various concurrency and throughput levels |
| Theoretical models | Documented throughput ceilings derived from architectural constraints (e.g., single-writer, network-bound) |
| Anti-pattern documentation | Explicit guidance on workload types or scale thresholds where the system is not a good fit |

---

## Using This Specification

### For Building a New System

1. Classify your system (Section 0).
2. Walk through each concern and decide: will you address it, delegate it, or mark it N/A?
3. For each addressed concern, select an approach from the valid patterns.
4. Document your choices as an implementation specification.
5. Revisit when your system type changes (e.g., graduating from library to platform).

### For Evaluating an Existing System

1. Classify the system (Section 0).
2. Walk through each concern. Score using the scale in Section 0.3.
3. Compare scores, noting that different system types have different expectations.

### For Comparing Systems

1. Evaluate each system independently.
2. Differences reveal architectural tradeoffs, not deficiencies.
3. ❌ scores reveal blind spots.
4. 🔀 Delegated scores reveal responsibility boundaries — evaluate whether your team can absorb those responsibilities.