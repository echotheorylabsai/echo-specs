# Software Product Craft — Principles & Practices

> **Scope:** Software product craft only — designing, architecting, building, and maintaining systems. People, team, and organizational dimensions are out of scope.

---

## How to use this document

- **Who it is for:** engineers and coding agents who plan, design, build, and maintain software products.
- **§0 is the checklist.** §1 onward is the lens: for each principle, find where your system sits on the axes in §3, then ask whether your design respects the principle *in that form*.
- **Examples** come from three system classes: multi-platform SaaS, workflow and orchestration platforms (Temporal, Dagster), and agentic AI applications (Claude Code-style agents). SaaS examples name the tier — server, mobile, frontend, API — where it matters.
- The principles apply to other classes too: CLIs, data platforms, embedded, games.
- **Code samples** are Python. The principles are language-neutral.
- **For agents:** load §0 by itself. Retrieve the rest when a rule needs its reasoning or an example.

---

## 0. The rules on one page

Everything below this section is the reasoning and the examples behind each line here.

### Proportionality rule — read first

Apply every principle in proportion to two things: how long the artifact will live (its **half-life**) and how much damage a mistake can do (its **blast radius**). Pick the simplest design that keeps the door open.

**Minimum code that solves the problem. Nothing speculative.**

- No features beyond what was asked.
- No abstractions for single-use code.
- No "flexibility" or "configurability" that wasn't requested.
- No error handling for impossible scenarios.
- If you write 200 lines and it could be 50, rewrite it.

Ask yourself: "Would a senior engineer say this is overcomplicated?" If yes, simplify.

### The eleven principles

1. **Data model is destiny.** Design durable entities and their state transitions before code. Migrating live data is the real lock-in, not editing the schema.
2. **Boundaries follow change rates.** Group what changes together, for the same reason, on the same cadence.
3. **Optimize for change.** Code is read and modified far more than it is written. Evolve formats additively.
4. **Reversibility sets the ceremony.** One-way doors get a written decision and a slow-down. Two-way doors get decided and done.
5. **Simple beats clever.** One concern per unit. Pure data and pure functions first.
6. **Boring substrate, novel surface.** Spend novelty only where it is the product.
7. **Make illegal states unrepresentable** where the set of legal states is settled. Keep the schema open where it is not.
8. **Design the failure modes.** Timeout on every external call. Retry only idempotent calls (safe to repeat), with jitter (random delay) and a cap. Idempotency key on every side-effecting call across a network or process boundary. A defined partial-failure path for every state transition.
9. **Pure core, imperative shell.** Side effects — including model calls — at the edges. Decisions in the middle, deterministic and testable.
10. **Strangler by default.** Replace incrementally behind a stable interface. Rewrite only what is small, has no users or data to preserve, or costs more to wrap than to rewrite — and say which, in writing.
11. **Assume an adversary.** Least privilege (grant only what the task needs), default deny, smallest blast radius. Prompt injection (untrusted text steering the model) is unsolved; defend with architecture, not with prompt wording.

### Operating rules

- Breaking changes to **persisted data, a consumer-facing API, an in-flight workflow, or an eval grader** (a model-evaluation scorer) follow **expand → migrate → contract**: add the new beside the old; move readers and data; remove the old only when nothing reads it.
  - Each step is reversible until the last. Internal changes whose callers all sit in one change set: change them atomically.
- **Small changes.** One concern per change. If it takes more than one sentence to describe, split it.
- **Tests prove behavior**, not implementation. Deterministic and isolated. A flaky test is a bug. No tests for impossible states.
- Every state transition and failure path emits an **observable signal** — a log, metric, or trace.
- **Irreversible or externally visible actions an agent takes** — send, pay, delete, publish — need confirmation or a reversible staging step.
- **Lockfiles committed. Dependencies scanned. Secrets never in code.**
- **If you cannot verify it, do not ship it.**

---

## 1. The Standard

Systems that are **easy to understand, safe to change, cheap to operate, and hard to compromise** — for the lifetime of every artifact, maintained by other people, under conditions no one anticipated.

That is the craft of a distinguished engineer: the strategic taste of an architect combined with the hands-on rigor of a principal engineer. **Design, architecture, build, and maintenance are one continuous activity**, never separate phases.

The standard applies equally to a human engineer and to a coding agent doing the same work.

---

## 2. The Four Pillars

| Pillar | Question it answers | Primary deliverables |
|---|---|---|
| **Design** | What are we building, and what are its invariants? | Domain model, interfaces, contracts, edge-case map, threat model |
| **Architect** | How do the pieces fit, and which decisions can we never undo? | Decision records, system diagrams, boundary definitions, tech-stack choices |
| **Build** | How do we encode this so it is correct, clear, changeable, and safe? | Reference implementations — especially for the parts where being wrong is most expensive: data models, sync engines, agent loops, payment paths, auth boundaries |
| **Maintain** | How does this stay healthy for the artifact's lifetime? | Refactoring patterns, migration playbooks, observability, deprecation paths, eval suites |

---

## 3. Core Principles

### Axes of Variation

The eleven principles look the same on paper but land differently across system classes. Five dimensions explain most of the difference:

| Axis | Spectrum |
|---|---|
| **State persistence** | Ephemeral context (agent) ↔ replay-durable history (workflow) ↔ persistent (database) |
| **Determinism** | Deterministic (compiler) ↔ replay-deterministic (workflow) ↔ probabilistic (LLM) |
| **Rollout control** | Server-instant ↔ mobile-staged (app-store review) ↔ workflow-in-flight (multi-day runs) ↔ dynamic (prompts and skills loaded at runtime) |
| **Failure taxonomy** | Partial failure / sync conflict / replay divergence / hallucination / prompt injection |
| **Artifact half-life** | Years (data schemas, public APIs) ↔ months (services) ↔ weeks (prompts, skills) |

A principle's emphasis — and occasionally its direction — shifts along these axes. Where the direction reverses, the principle says so.

The proportionality rule (§0) is these axes applied. **Half-life** says how much design a thing deserves. **Blast radius** — set by reversibility (§3.4) and the failure taxonomy — says how much caution.

---

### 3.1 Data model is destiny

**What it means.** The shape of your data — entities, relationships, lifecycles — constrains every later choice: code structure, performance, which features are cheap and which become prohibitively expensive.

- A wrong data model compounds. A right one quietly enables features for years.
- The real cost is rarely the schema redesign. It is **migrating live data and every dependent that relies on the old shape**.
- That is why a model stays hard to change long after its schema file is trivial to edit (see §4, *Migration*).

**In practice.**

- For durable data, spend design time on entities and their state transitions before writing code. For throwaway or ephemeral state, do not.
- Model relationships and cardinalities explicitly. Resist "one column for now" shortcuts for anything that will outlive the sprint.
- Defer denormalization until measurement justifies it.

**Across system classes.**

- **SaaS (server).** Many products ship `user.email` as a single text column. Later they need several addresses per user, a verified flag, and a primary marker.
  - The fix is a new `email_addresses` table — plus a backfill of every existing user and a change to every query and client that read `user.email`.
  - The schema edit takes a minute. The migration takes a quarter.
- **Workflow / orchestration.** A workflow's append-only event history *is* the data model. Change what an event means and every in-flight workflow replays against the new meaning. Design the event schema with the same care as a public API.
- **Agentic AI — which layer is destiny.** The principle binds the layer with the long half-life, not the one measured in turns:
  - **Durable substrate** — long-term memory schema, conversation store, tool and result schemas — for example, those a Model Context Protocol (MCP) server declares. *This* is destiny. A memory layer keyed only on string match cannot later support semantic similarity without a redesign.
  - **Ephemeral working memory** — per-turn scratch state, reasoning context for a single request. Deliberately disposable. Applying "destiny" here produces over-engineering, not safety.

---

### 3.2 Boundaries follow change rates

**What it means.** A module, file, or service boundary is well drawn when the things inside it change together — for the same reason, by the same people, on the same cadence. Boundaries drawn any other way create the coupling they were meant to remove.

**In practice.**

- Group code by business domain (what changes together), not by technical type (controllers, models, utils).
- A service boundary is correct only if the services can deploy independently. If every change touches three services, the boundary is wrong.
- At small scale, one deployable with clear internal modules beats an early service split.
- Re-draw boundaries when change patterns shift. A boundary that fit two years ago may not fit now.

**Across system classes.**

- **SaaS (server).** The "distributed monolith" appears when services are split by data type — `UserService`, `OrderService`, `ProductService` — and every product change needs a coordinated deploy across all three (Sam Newman, *Building Microservices*, O'Reilly).
  - Shopify took the inverse path: one Rails deployable with strict internal module boundaries around business capabilities — Cart, Checkout, Inventory. Decoupling without distribution (Kirsten Westeinde, *Deconstructing the Monolith*, RailsConf 2019).
- **Workflow / orchestration.** Temporal separates Workflows (orchestration logic, which must replay deterministically) from Activities (I/O). The primary driver is determinism (§3.9). A change-rate benefit follows: activity code changes per integration; workflow code changes per business process.
- **Agentic AI.** Claude Code separates skills (capability packages), MCP servers (external tool and data integrations), subagents (isolated context windows), and hooks (deterministic shell triggers).
  - Each is its own boundary because the artifacts inside change for different reasons and at different rates (docs.claude.com).

---

### 3.3 Optimize for change, not for today's elegance

**What it means.** The dominant lifetime cost of software is not writing it but reading and modifying it later, usually by people who did not write it. A design slightly less elegant today but easier to evolve tomorrow is the right trade.

**In practice.**

- Make modification surfaces explicit: pure functions, clear interfaces, documented invariants.
- Choose data formats and APIs that evolve additively — optional fields, new variants.
- Prefer code that five future engineers can change confidently over code one current engineer finds clever.

**Across system classes.**

- **SaaS (API).** Stripe pins every customer to the API version they integrated against. Internally, requests pass through version-translation layers that map old shapes to current models (Brandur Leach, *APIs as infrastructure*, Stripe Engineering Blog, August 2017).
- **SaaS (mobile).** Mobile is harder. App-store review and slow user adoption mean a breaking client/server change can take six months or more to retire. Versioning is not optional.
- **Workflow / orchestration.** Both major platforms version explicitly to protect in-flight executions.
  - Temporal's Patching API (`GetVersion` in Go and Java; `patched` in Python, TypeScript, and .NET) records version markers in event history so replays stay deterministic as code evolves (docs.temporal.io).
  - Dagster provides `code_version` and `DataVersion` for asset materializations (docs.dagster.io).
- **Agentic AI.** Prompts, skills, and tool schemas change weekly. Treat them as code: versioned, reviewed, evaluated.

---

### 3.4 Reversibility-aware decisions

**What it means.** Some decisions are one-way doors (Type-1): undoing them is prohibitively expensive. Others are two-way doors (Type-2): undoing them is cheap.

Ceremony, debate, and writing should match irreversibility, not perceived importance (Jeff Bezos, 2015 Amazon shareholder letter, published April 2016).

**In practice.**

- **Type-1** — public API contracts, identifier schemes, persisted data formats, identity and authorization systems. Write the decision record (§4), surface disagreement, slow down.
- **Type-2** — internal class names, build tooling, a library used in one module. Decide and move.
- **Watch the drift.** A framework or ORM is Type-2 to *decide* and often Type-1 to *undo* once thousands of files depend on it. Classify by the cost of undoing at the scale you expect, not the cost of choosing today.

**Across system classes.**

- **SaaS (server).** Identifier strategy is a Type-1 classic. Once an auto-increment integer ID appears in URLs, foreign keys, customer webhooks, and exported reports, switching to UUIDs is prohibitively expensive — or it breaks external systems.
- **SaaS (mobile).** Mobile rollback extends Type-1 windows in a way backend engineers underestimate. A bad iOS or Android build sits in users' hands for days even after a staged-rollout pause, because a binary cannot be force-uninstalled.
- **Workflow / orchestration.** Deploying a breaking workflow change is Type-1. In-flight workflows may run for weeks and will replay against the new code unless the Patching API is used.
- **Agentic AI.** Agent **actions** — sent emails, executed payments, posted comments — are irreversible at runtime even when the agent's design is freely revisable. Apply Type-1 discipline to the action policy, not just the code. §3.8 gives the practice.

---

### 3.5 Simplicity > cleverness

**What it means.** Rich Hickey's distinction (*Simple Made Easy*, Strange Loop 2011):

- *Simple* means one concern, no interleaving — an objective property of the code.
- *Easy* means familiar, close to hand — a subjective feeling.

Many systems feel easy at first but are not simple. The interleaved concerns surface later as compounding cost.

**In practice.**

- Prefer pure data and pure functions over objects with hidden state.
- Prefer composition over inheritance.
- Question abstractions that bundle concerns — one object handling caching, persistence, and identity.

**Across system classes.**

- **SaaS (server).** An ORM makes `User.find(id).orders.recent` one line. For routine reads and writes, that is the right trade.
  - Beneath it live caching, lazy loading, and transaction management. On hot or complex paths those hidden concerns are what bite: N+1 queries (one query per row instead of one per set), stale caches, surprising transaction boundaries (Ted Neward, *The Vietnam of Computer Science*, 2006).
  - Use the ORM for the routine. Write explicit SQL where the query matters.
- **Agentic AI.** A "god skill" that bundles context retrieval, reasoning, tool selection, and persistence is the agentic equivalent — easy to ship, painful to debug. Focused skills with explicit interfaces are harder to start and simpler to evolve.

---

### 3.6 Boring tech is a feature — boring substrate, novel surface

**What it means.** Every novel technology carries hidden, ongoing cost: hiring, on-call expertise, unknown failure modes, upgrade paths, integration with the rest of the stack.

You have a small budget of "innovation tokens" (Dan McKinley, *Choose Boring Technology*, 2015). Spend them where novelty is the strategic point.

For products whose value *is* novel tech — AI agents, for instance — the rule becomes **boring substrate, novel surface**. Storage, queues, and observability stay boring. The differentiating layer is where novelty earns its keep.

**In practice.**

- For a server that needs a database, default to PostgreSQL. For local or embedded state, SQLite.
- Add a queue (Redis, SQS) only when you need one. Use one of the major clouds and the language your team knows best.
- Justify any deviation in writing: what does this buy that boring tech cannot?
- Reach for novelty only where it is the moat — never out of taste.

**Across system classes.**

- **SaaS (server).** Stack Overflow served some of the highest-traffic sites on the internet from a small server fleet running .NET, SQL Server, IIS, Redis, and a monolith — deliberately boring (Nick Craver, *Stack Overflow: The Architecture — 2016 Edition*, February 2016).
- **SaaS (mobile).** WhatsApp's messaging backend is built primarily in Erlang/OTP — exotic to outsiders, but the canonical fault-tolerant message-passing runtime (Rick Reed, Erlang Factory 2012 and 2014). "Boring" is defined by the problem domain.
- **Agentic AI.** MCP (announced November 25, 2024) standardizes tool and data integrations so each product does not invent its own wiring.
  - It is not yet *boring*: the specification has five published versions between November 2024 and July 2026, and the July 2026 version changed the protocol layer itself.
  - Treat it as a boundary to isolate behind an adapter (§3.2), not as stable ground.

---

### 3.7 Make illegal states unrepresentable

**What it means.** Encode invariants in the type system or schema so that invalid combinations of state cannot be constructed. The compiler or schema validator enforces correctness, replacing scattered runtime guards.

Closely related: *Parse, don't validate* (Alexis King, lexi-lambda.github.io, November 5, 2019). Convert untrusted input into a typed value once, at the boundary. Trust the type from then on.

**In practice.**

- Replace nullable fields and parallel boolean flags with discriminated unions (sum types).
- Use distinct types where validation matters: `EmailAddress` and `UserId`, not `str` and `str`.
- Make state machines explicit. Each state carries only its valid fields.

**Example.** Three independent flags — `loading`, `error`, `data` — permit `loading=True, error=set, data=set`. A union makes that impossible:

```python
# Python 3.11+ (assert_never is in typing_extensions for older versions)
from dataclasses import dataclass
from typing import assert_never

@dataclass(frozen=True)
class Loading: ...

@dataclass(frozen=True)
class Failed:
    error: str

@dataclass(frozen=True)
class Loaded:
    data: dict

Fetch = Loading | Failed | Loaded

def render(state: Fetch) -> str:
    match state:
        case Loading():
            return "spinner"
        case Failed(error=e):
            return f"error: {e}"
        case Loaded(data=d):
            return str(d)
        case _:
            assert_never(state)  # type checker fails if a variant is unhandled
```

**Across system classes.**

- **SaaS (API).** Stripe's PaymentIntent exposes a `status` field (`requires_payment_method`, `requires_confirmation`, `processing`, `succeeded`, …). Each state exposes only the actions valid in that state.
- **Workflow / orchestration.** A Temporal workflow definition *is* a state machine. States the code cannot construct cannot appear in the event history.
- **Agentic AI.** Tool input schemas (MCP, function calling, structured output) reject invalid tool invocations at the boundary, before they reach the reasoning loop.

**Tension with §3.3 (optimize for change).** A sum type makes illegal states impossible, but **adding a variant forces an update at every site that branches on it** — the opposite of the additive evolution §3.3 prefers.

Whether that update is *safe* or *silent* depends on how much exhaustiveness your toolchain enforces:

| Enforcement | Languages / tools | On a new variant |
|---|---|---|
| **Hard error** | Rust; Swift `switch` | Build fails. Automatic. |
| **Warning only** | OCaml (default); Scala sealed types (default); Haskell (under `-W` / `-Wall`) | Compiler warns. Build fails only with warnings-as-errors on. |
| **Type checker** | TypeScript `never` assignment; Python `assert_never` | Fails whenever the type checker runs. Decorative if no checker runs in CI. |
| **Unenforced** | Python `match` with no checker; plain `if/elif` | Silently falls through. Sum-type rigidity with zero safety. |
| **Boundary only** | Pydantic discriminated unions | Rejects bad *input* at parse time. Does not check that every branch is handled. |

A `match` with no catch-all `assert_never` needs the checker's own exhaustiveness option instead: pyright `reportMatchNotExhaustive` or mypy `--enable-error-code exhaustive-match`.

**Resolution (artifact half-life, §3).**

- Encode invariants for long-half-life contracts where the legal state set is *settled*: persisted formats, public APIs, identity.
- Stay additively loose on short-half-life surfaces — weekly-changing prompts, skills, internal data-transfer objects — where new variants land often.
- Rule of thumb: **make illegal states unrepresentable where the legal set is settled; keep the schema open where it is not.**

> **Python.** A sum type is an invariant only when a type checker (pyright or mypy) runs in CI and every `match` ends in `assert_never` — or the checker's exhaustiveness option is on. Without that, it is a convention.

---

### 3.8 Design the failure modes, not the happy path

**What it means.** The happy path nearly designs itself. Real design effort goes into what can fail, how it is detected, what recovers automatically, what data must survive, and what humans are told.

**Failure taxonomies differ by system class.** Design for the one that applies:

| Domain | Failure modes to design for |
|---|---|
| **Distributed services** | Partial failure, network timeout, retry storms, cascading failure |
| **Multi-client SaaS** | Offline writes, sync conflict, push delivery loss, app crash mid-write |
| **Workflow / orchestration** | Replay divergence, non-deterministic activity, version skew between worker and history |
| **Agentic AI** | Hallucination, context overflow, infinite tool loop, runaway cost, indirect prompt injection |

**In practice.**

- Every external call gets a timeout.
- Retry only calls that are idempotent — with jitter, a cap, and at one layer, so retries do not multiply across layers.
- Every side-effecting call that crosses a network or process boundary and may be retried carries an idempotency key. Where the callee has no key support, dedupe on your side with a stored request id.
- A fallback where one is meaningful — a cached value, a degraded mode. Not everywhere.
- Every state transition: partial-failure behavior, recovery path, observable signal.
- Failure injection (chaos testing) on the critical paths of systems with real users.

```python
# One key per logical operation. Create and store it once, before the first
# call; every retry reuses it. A new charge decision gets a new key.
payments.charge(customer_id, amount_cents, idempotency_key=charge_id)
```

**Agent actions.** Because agent actions are irreversible at runtime (§3.4):

- Classify every tool by reversibility before the agent can call it.
- Irreversible or externally visible actions — send, pay, delete, publish — require explicit confirmation or a reversible staging step: a draft, a dry run, a soft delete.
- Idempotency key on every side-effecting call that crosses a network or process boundary, so it is safe to retry.
- Grant each task only the tools it needs.
- Set stopping conditions: an iteration cap and a cost budget.

**Across system classes.**

- **SaaS (distributed backend), positive.** AWS treats failure as a first-class input.
  - Mutating APIs accept idempotency tokens (EC2 `RunInstances` has `ClientToken`). SDKs implement exponential backoff with jitter by default (Marc Brooker, *Exponential Backoff and Jitter*, AWS Architecture Blog, March 2015).
- **SaaS (distributed backend), cautionary — Knight Capital, August 1, 2012.**
  - What happened: a deployment left an obsolete code path on one of eight servers; a repurposed feature flag activated it and it began placing erroneous orders. About $440M lost in roughly 45 minutes (SEC Release No. 70694, October 2013).
  - What was missing: a kill switch, an anomaly detector, an automatic safe-stop. Knight ceased to exist as an independent firm within months.
- **Agentic AI, cautionary — *Moffatt v. Air Canada* (2024 BCCRT 149, February 14, 2024).**
  - What happened: Air Canada's chatbot invented a bereavement-fare refund policy. The tribunal held the airline liable.
  - What it means: the award was small (CAD $812.02 in total); the principle is not. Organizations are accountable for the outputs of the agents they deploy. The hallucination failure mode had not been designed.
- **Agentic AI, research.** Indirect prompt injection: untrusted content — an email, a web page, a document — reaching a model with tool access can hijack intent (Greshake et al., arXiv:2302.12173, February 2023).

---

### 3.9 Pure core, imperative shell

**What it means.** Gary Bernhardt's pattern (*Boundaries*, SCNA 2012):

- Push side effects — I/O, time, randomness, mutation — to the edges.
- The core takes data in and returns data out, deterministically. Business logic only.
- The shell orchestrates I/O around the pure core.

**In practice.**

- A request handler should be: load the data → call a pure function → write the result. Not: interleave business decisions with database calls.
- Pure functions are trivially testable, parallelizable, and easy to reason about.

**Across system classes.**

- **SaaS (frontend).** Redux reducers are pure functions `(state, action) → newState`. Side effects are confined to middleware.
- **Workflow / orchestration.** Temporal persists an append-only event history per workflow. On worker crash, the code replays against that history to rebuild in-memory state deterministically. Workflow code is the pure core; Activities are the imperative shell.
- **Agentic AI.** A model call is non-deterministic, so it cannot be part of the core. Treat it as one more external call at the edge — like a database or an HTTP request.
  - Prompt assembly, output parsing, and **deciding** which tool to call stay pure and unit-testable without a model.
  - **Calling** the tool is I/O and belongs in the shell.

```python
# Sketch — names are illustrative.
# Pure: decide. Testable with no model and no network.
def next_action(state: AgentState, model_reply: str) -> Action: ...

# Shell: do.
reply = llm.complete(build_prompt(state))   # I/O
action = next_action(state, reply)          # pure
result = tools.run(action)                  # I/O
```

Treating the model as a pure function of its inputs is one of the most common architectural mistakes in this domain.

---

### 3.10 Strangler fig by default, big-bang by exception

**What it means.** Martin Fowler's pattern (*StranglerFigApplication*, martinfowler.com, June 2004), named after the fig that grows around its host tree. Route traffic incrementally to the new implementation while the old keeps serving, until the new carries everything and the old can be removed.

Big-bang rewrites fail at a high rate: they freeze a moving target and demand perfect knowledge of the old system's behavior.

**In practice.**

- Put the legacy system behind a stable interface — a router, proxy, or facade.
- Build the new implementation behind the same interface. Route a small slice of traffic to it.
- Where old and new outputs can be compared, run shadow mode (compare without serving). Expand traffic gradually, then remove the old.
- **No routable seam?** Create one first: introduce an abstraction over the old implementation, move callers to it, then swap what sits behind it (*Branch by Abstraction* — Jez Humble, continuousdelivery.com, 2011; Fowler, January 2014).

**When a rewrite is acceptable.** The system is small; it has no users or data to preserve; or building the seam would cost more than the rewrite. Say which one applies, in writing, before starting.

**Across system classes.**

- **SaaS (server), cautionary.** Netscape rewrote Communicator from scratch. The rewrite consumed years during which Netscape shipped no competitive browser, and Internet Explorer took the market (Joel Spolsky, *Things You Should Never Do, Part I*, April 2000).
- **SaaS (server), positive.** Facebook's Hack/HHVM transition introduced a gradually-typed PHP dialect that interoperates file by file with existing PHP, letting Facebook migrate nearly its whole codebase incrementally (Verlaguet & Menghrajani, Facebook Engineering, March 20, 2014).
- **Agentic AI.** Route a fraction of traffic to a new prompt or model behind a stable agent interface, compare outputs in shadow mode, then promote. Same shape.

---

### 3.11 Adversarial-aware design

**What it means.** Failure-mode design (§3.8) asks *"what breaks?"* Adversarial design asks *"what can be made to break by someone who wants it to?"* The two produce different mitigations:

- Failure modes → retries, circuit breakers, fallbacks.
- Adversarial attacks → threat modeling, least privilege, sandboxing, blast-radius minimization.

Security is a design input, not a hand-off to a security team.

**In practice.**

- Threat-model at design time for anything that handles untrusted input, secrets, identity, money, or irreversible actions: STRIDE (a six-category threat checklist), attack trees, or a trust-boundary diagram on a whiteboard.
- Default to deny. Require justification for each granted capability.
- Minimize blast radius: a compromised component should reach the smallest possible adjacent surface.
- Treat secrets, identity, and authorization boundaries as Type-1 decisions (§3.4).
- **Dependencies:** commit lockfiles; run automated dependency updates and vulnerability scanning in CI.
- **Secrets:** never in code or the repository. Environment variables or a secret manager; rotate on exposure.

**Across system classes.**

- **SaaS (server), cautionary — Equifax, 2017.**
  - What happened: Apache Struts CVE-2017-5638 was patched on March 7, 2017. Attackers first entered on May 13; the breach was detected on July 29. Personal data of about 145.5 million U.S. consumers was exposed (GAO-18-559, August 2018).
  - What was missing: the process that should have applied a two-month-old patch.
- **Any class, cautionary — SolarWinds SUNBURST, disclosed December 13, 2020.**
  - What happened: the build pipeline for Orion updates was compromised; trojanized updates shipped March–June 2020 to up to 18,000 customers (CISA Alert AA20-352A).
  - What was missing: a threat model that included the build pipeline. This attack went a layer below the application.
- **Agentic AI — prompt injection.** The term was coined by Simon Willison (September 12, 2022), building on an exploit surfaced by Riley Goodside. In February 2023, Kevin Liu extracted Bing Chat's hidden system prompt with a direct injection.
  - Prompt injection resembles SQL injection — untrusted input reaching an interpreter — but **there is no equivalent of the parameterized query**.
  - SQL has a grammar, so a parser can separate code from data. A language model has no such separation; it follows instructions wherever they appear.
  - Treat prompt injection as unsolved (UK NCSC, *Prompt injection is not SQL injection*; OWASP Top 10 for LLM Applications 2025, LLM01).

  Defend with architecture, not with prompt wording:
  - Least privilege per tool, per task (OWASP LLM06, *Excessive Agency*).
  - Keep untrusted content away from any context that can call tools. Where it must be read, read it with a model that has no tool access and pass only structured results onward (the *CaMeL* pattern — Google DeepMind, arXiv:2503.18813, 2025).
  - Human confirmation for irreversible or externally visible actions (§3.8).
  - Adversarial testing as a standing practice, not a one-off audit.

---

## 4. Operating Practices

**Architecture decision records (ADRs) for irreversible choices.** One page: context, options, decision, consequences. Written before lock-in, so disagreement is on the record.

**Observability, sized to blast radius.** A prototype needs logs. A payment path needs traces. The form depends on the domain:

- Backends: logs, metrics, distributed traces.
- Client apps: crash reporting, real-user monitoring.
- Workflow / orchestration: event-history timelines, worker dashboards.
- Agents: trace replay, token-cost telemetry, eval scores.

**Feature flags decouple deploy from release** for user-visible or risky changes. Ship dark, enable gradually, kill-switch instantly. Skip them for prototypes and internal tools.

- Mobile clients need server-side flag checks because binaries cannot be force-updated.
- A flag whose rollout is complete is dead code — remove it.

**Migration is one shape: expand → migrate → contract** (Danilo Sato, *ParallelChange*, martinfowler.com, May 2014). Every breaking change to persisted data, a consumer-facing API, an in-flight workflow, or an eval grader follows it:

1. **Expand.** Add the new alongside the old. Nothing breaks.
2. **Migrate.** Move consumers — and data — from old to new. Both work throughout.
3. **Contract.** Remove the old, only once nothing reads it.

Every step is reversible until contract. Never rename or drop in one step on live data. Internal changes whose callers all sit in one change set do not need the three steps — change them atomically.

| Change | Expand | Migrate | Contract |
|---|---|---|---|
| Rename a column | Add the new column; write to both | Backfill; switch reads to the new column | Drop the old column |
| Change an API field | Add the new field; keep the old | Move clients; log who still reads the old one | Remove the old field |
| Change an eval grader | Version the grader; score with both | Re-baseline; pin dataset and grader version to every score | Retire the old grader |

**Small, reversible changes.** One concern per change. If describing it takes more than one sentence, split it. Large migrations become a sequence of small changes through the shape above (Google Engineering Practices, *Small CLs*).

**Continuous refactor.** Leave code better in its own small change, never folded into a feature change. Agents: propose the refactors you notice; do them only when asked or when the refactor *is* the task.

**Tests prove behavior.** Six rules:

1. Test through public interfaces. A refactor that keeps behavior should not break a test (Google Testing Blog, *Test Behavior, Not Implementation*, August 2013).
2. Unit tests use no real clock, network, or shared state. Integration tests use real dependencies under test control — a test database, a container — and stay deterministic and isolated from each other.
3. **A flaky test is a bug.** Fix it or delete it. Never retry to green.
4. Most unit tests target the pure core (§3.9). A few integration tests cover the shell.
5. Property-based or fuzz tests for parsers and state machines at boundaries (§3.7).
6. No tests for impossible states. Coverage is a signal, not a goal.

If it is hard to test, the design is wrong. **If you cannot verify it, do not ship it.**

**Evals complement tests on probabilistic surfaces.** For models and ML, eval suites catch regressions tests cannot (Hamel Husain, *Your AI Product Needs Evals*, hamel.dev, March 2024). The principles above apply to the eval harness itself:

- **Data model is destiny (§3.1).** The eval dataset and grader schema decide what a score can *mean*. A rubric that collapses "wrong" and "unhelpful" into one pass/fail can never later tell them apart without re-grading everything.
- **Migration is first-class.** Changing a grader, rubric, or judge model breaks comparability with every prior score — 0.82 today is not 0.82 last month. Run it through expand → migrate → contract with a versioned dataset and a re-baseline step.
- **Boundaries follow change rates (§3.2).** Keep weekly-changing prompts on one side of a boundary and slow-moving golden datasets and grader logic on the other.

---

## 5. Anti-Patterns

What a distinguished engineer — or a well-run agent — does not do:

- ❌ Introduce new technology without a decision record explaining what the existing tech failed at.
- ❌ Add an abstraction before the third concrete case. *Exceptions:* a replacement seam (§3.10); design systems and shared agent capabilities (skills, tools) when consistency itself is the value.
- ❌ Ship code without considering failure modes and observability — and, wherever it touches untrusted input, secrets, identity, money, or irreversible actions, **the threat model**.
- ❌ Design for hypothetical future requirements.
- ❌ Perform a big-bang rewrite without stating which §3.10 exception applies.
- ❌ Optimize before measuring — including token-cost optimization in agentic systems.
- ❌ Leave dead code, dead flags, dead services — and, for agents, orphaned skills, unused tools, stale memory entries. (Agents: propose the removal; do it when asked — §4.)
- ❌ Accept "magic." Every behavior has a traceable origin. Prompt-driven behavior must be explainable; a black-box "it just works" agent loop is an anti-pattern.
- ❌ Chase every review finding. A reviewer asked to find gaps will find some. Act on findings that affect correctness, safety, or stated requirements; the rest produce extra layers, defensive code, and tests for cases that cannot happen.

---

## 6. Mental Models Toolkit

Patterns a distinguished engineer recognizes within minutes of seeing a problem.

**Distributed & concurrent**

- Backpressure — producer/consumer rate mismatch
- CAP and PACELC — consistency vs. availability under partition; latency vs. consistency otherwise
- Idempotency and at-least-once delivery — any distributed call
- Race conditions, deadlocks, livelocks — any concurrent code
- Two-generals / consensus — distributed agreement
- Cache invalidation and cache stampede — read-heavy systems

**Stateful & evolving**

- State machines — anything with a status field
- Bounded contexts (domain-driven design) — where one model ends and another begins
- Eventual vs. strong consistency — replicated state
- Conway's Law — team structure shapes system structure; draw boundaries knowing this

**Workflow & multi-client**

- Determinism and replay — workflows, simulations, reproducible builds
- Saga / compensation — a distributed transaction as reversible steps with compensating actions on failure (Garcia-Molina & Salem, *Sagas*, ACM SIGMOD 1987 — originally for long-lived transactions in one database; the microservices framing came later)
- Offline-first and conflict resolution — multi-client sync (Shapiro et al., *Conflict-free Replicated Data Types*, SSS 2011)
- Operator UX as product — the dashboard, retry button, and timeline view *are* the product for ops users (Temporal, Dagster, Airflow)

**Probabilistic systems**

- Probabilistic correctness and evals — output is "right enough most of the time," never "right"
- Token / context budget — the LLM-era analog of memory budget in embedded systems
- Prompt injection, direct and indirect — untrusted input reaching an interpreter, with no parameterized-query fix (§3.11)

---

## 7. The One-Line Distillation

> **Make the right things easier to build and the wrong things harder to build — across the whole product, for the lifetime of every artifact, against accidents and adversaries — through strategic taste, architectural rigor, and hands-on craft.**

---

## 8. Self-Test

- Can I name the **invariants** of every system I own?
- Have I written a **decision record** for every irreversible decision in my domain?
- Is my system **observable** end to end, in the form appropriate to its class?
- Could a new engineer — or a fresh agent session — **understand and change** my code without my help?
- When something breaks at 3 a.m., does the **playbook** already exist?
- Am I **deleting** as much code as I am writing?
- Have I **threat-modeled** this — assuming an adversary, not just an accident?
- Do I know the **unit economics** — $ per request, per active user, per agent action?
- For probabilistic surfaces, do my **evals** catch regressions my tests cannot?
- Was every change I shipped this week **describable in one sentence**?

If any answer is "no," that is the next thing to fix.
