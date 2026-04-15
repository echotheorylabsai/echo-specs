# Worker Service Evaluation Rubric

## Universal Concerns Every Worker Service Must Address

> **Purpose:** This is an implementation-agnostic checklist. Use it to evaluate any worker service — whether it's built on Temporal, Celery, BullMQ, AWS Step Functions, Kafka consumers, a custom Postgres-backed system, or anything else. Every concern here must have an answer. The answer can be "not applicable" but it can't be "we haven't thought about it."

---

## 1. Work Unit Model

How does the system represent and organize units of work?

| Concern | Questions to Answer |
| --- | --- |
| **Granularity** | What is the smallest unit of work? (job, task, message, event, activity) |
| **Composition** | Can work units compose? (flat, parent-child, DAG, saga, pipeline) |
| **Definition vs Instance** | Is there a separation between the template (workflow definition) and execution (a running instance)? |
| **Payload** | How is input/output data passed to and from work units? Size limits? Serialization format? |
| **Identity** | How is each work unit uniquely identified? Can it be referenced externally? |

---

## 2. State Management

How does the system track the lifecycle of work?

| Concern | Questions to Answer |
| --- | --- |
| **State representation** | What states can a work unit be in? Are transitions explicitly defined? |
| **Transition enforcement** | Can invalid state transitions occur? How are they prevented? |
| **Persistence** | Where is state stored? What are the durability guarantees? |
| **Visibility** | Can external callers query current state? In real-time or eventually consistent? |
| **History** | Is the full state transition history preserved or only current state? |

---

## 3. Idempotency & Exactly-Once Semantics

How does the system prevent duplicate work?

| Concern | Questions to Answer |
| --- | --- |
| **Submission dedup** | If the same work is submitted twice, what happens? Is there a dedup key? |
| **Execution dedup** | If a work unit is retried, how does the handler know it already completed? |
| **Side-effect safety** | How are external side effects (API calls, DB writes, emails) made safe across retries? |
| **Guarantee level** | At-least-once, at-most-once, or effectively-once? What are the tradeoffs? |

---

## 4. Failure Taxonomy & Handling

How does the system categorize and respond to failures?

| Concern | Questions to Answer |
| --- | --- |
| **Error classification** | Does the system distinguish retryable vs fatal vs transient errors? |
| **Retry policy** | What is the retry strategy? (fixed, exponential, custom) Is it configurable per work unit? |
| **Exhaustion** | What happens when retries are exhausted? (DLQ, alert, callback, dead state) |
| **Poison work** | How is permanently failing work identified and isolated? |
| **Partial failure** | If a composed workflow partially completes, what happens? (compensating actions, rollback, partial state) |
| **Dependency failure** | How does the system behave when a downstream dependency is unavailable? |

---

## 5. Scheduling & Ordering

How does the system decide what runs when?

| Concern | Questions to Answer |
| --- | --- |
| **Dispatch model** | Push (broker delivers to worker) or pull (worker claims from queue)? |
| **Ordering guarantees** | FIFO, priority-based, unordered? Per-tenant or global? |
| **Delayed/scheduled work** | Can work be scheduled for future execution? |
| **Rate limiting** | Can execution rate be throttled? Per-tenant? Per-work-type? |
| **Concurrency control** | Max concurrent executions globally? Per tenant? Per work type? |

---

## 6. Backpressure & Flow Control

How does the system behave under load?

| Concern | Questions to Answer |
| --- | --- |
| **Overload signal** | How does the system signal to producers that it's overloaded? |
| **Rejection policy** | Does it reject, queue unboundedly, or drop work under pressure? |
| **Tenant fairness** | Can one tenant's volume starve others? How is fairness enforced? |
| **Worker saturation** | What happens when all workers are busy? How is this detected? |
| **Queue depth management** | Is queue depth monitored? Are there limits? |

---

## 7. Timeout & Liveness

How does the system handle work that takes too long or disappears?

| Concern | Questions to Answer |
| --- | --- |
| **Execution timeout** | Is there a max duration per work unit? What happens on timeout? |
| **Claim/lease expiry** | If a worker claims work and dies, how is the work reclaimed? |
| **Heartbeat** | Do workers signal liveness during long-running work? What's the failure detection latency? |
| **Deadline propagation** | Can a top-level deadline cascade down to child work units? |

---

## 8. Observability

How do you know what the system is doing?

| Concern | Questions to Answer |
| --- | --- |
| **Logging** | Are logs structured? Do they include correlation IDs (work unit ID, tenant ID, worker ID)? |
| **Metrics** | Are throughput, latency, error rate, queue depth measured? |
| **Tracing** | Can a single work unit's execution be traced end-to-end across components? |
| **Alerting** | What conditions trigger alerts? Are there SLA-based alerts? |
| **Debugging** | Can you inspect a specific work unit's full history, inputs, outputs, and errors? |
| **Dashboards** | Is the system's health visible at a glance? |

---

## 9. Multi-Tenancy

How does the system isolate and manage tenant workloads?

| Concern | Questions to Answer |
| --- | --- |
| **Data isolation** | Is tenant data logically or physically separated? |
| **Resource isolation** | Can one tenant's workload impact another's performance? |
| **Tenant context** | How is tenant identity associated with work? Metadata? Separate queues? |
| **Tenant-scoped queries** | Can you query all work for a specific tenant? |
| **Tenant lifecycle** | What happens to in-flight work when a tenant is suspended or deleted? |

---

## 10. Security

How is the system protected?

| Concern | Questions to Answer |
| --- | --- |
| **Authentication** | How do callers authenticate? (API keys, mTLS, OAuth2, network-level) |
| **Authorization** | Are permissions scoped? (submit, read, cancel, admin) |
| **Payload security** | Is sensitive data in payloads encrypted at rest? In transit? |
| **Network boundary** | Are workers and storage accessible only from within the trusted network? |
| **Secret management** | How do workers access credentials for downstream services? |
| **Audit trail** | Are submission, cancellation, and administrative actions logged with actor identity? |

---

## 11. Data Lifecycle

How does the system handle data over time?

| Concern | Questions to Answer |
| --- | --- |
| **Retention** | How long are completed work units and their payloads retained? |
| **Archival** | Is there a tiered storage strategy (hot → warm → cold)? |
| **Export** | Can work history be exported per tenant? |
| **Deletion** | Can data be purged per tenant? Is deletion cascading and complete? |
| **Compliance** | Does retention/deletion meet regulatory requirements (GDPR, SOC2)? |

---

## 12. Billing & Metering

How is usage tracked and charged?

| Concern | Questions to Answer |
| --- | --- |
| **Metering dimensions** | What is measured? (executions, compute time, data volume, API calls) |
| **Metering reliability** | Can metering data be lost? What's the acceptable margin of error? |
| **Metering impact** | Does metering affect execution performance? |
| **Entitlements** | Are usage limits enforced before or after execution? |
| **Reporting** | Can tenants see their own usage? In real-time or batched? |

---

## 13. Operational Concerns

How is the system deployed and maintained?

| Concern | Questions to Answer |
| --- | --- |
| **Deployment** | Can the system be updated without losing in-flight work? |
| **Scaling** | How are workers scaled? Manually, auto-scaled, serverless? |
| **Migration** | How are schema/state changes applied to in-flight work? |
| **Disaster recovery** | What's the recovery plan for total system failure? RPO/RTO? |
| **Rollback** | Can a bad deployment be rolled back without data loss? |

---

## How to Use This Rubric

**For evaluation:** Walk through each concern for the system under review. Mark each as:
- ✅ **Addressed** — clear answer with implementation details
- ⚠️ **Partially addressed** — answer exists but has gaps or known limitations
- ❌ **Not addressed** — no answer or not considered
- **N/A** — genuinely not applicable to this system's use case (must justify)

**For comparison:** Fill out the rubric for each system side-by-side. Differences in answers reveal architectural tradeoffs, not right/wrong — but missing answers reveal blind spots.

**For spec writing:** Every ✅ becomes a section in your implementation spec. Every ⚠️ becomes a known limitation or TODO. Every ❌ becomes a risk.