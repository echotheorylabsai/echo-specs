# Software Product Craft — Principles & Practices

> **Scope:** Software product craft only — designing, architecting, building, and maintaining systems. People, team, and org dimensions are explicitly out of scope.

---

> **Who this is for & how to use it.**
>
> These principles cover designing, architecting, building, and maintaining software products — applied without bias toward any single application class. Examples are drawn deliberately from three distinct system classes:
>
> - **Multi-platform SaaS** (web/desktop/mobile — e.g., Facebook, Twitter, WhatsApp)
> - **Workflow & orchestration platforms** (e.g., Temporal, Dagster)
> - **Agentic AI applications** (e.g., OpenClaw, Claude Code-style agents)
>
> The same principles generalize to other classes (CLIs, data platforms, embedded, games) — use the **Axes of Variation** in §3 to translate.
>
> Use it as a **lens, not a checklist.** Read a principle, identify which axis your system sits on, and ask whether your design respects the principle in *that* form. Not every principle weighs equally for every system, and a few flip direction across axes — those are flagged inline.

---

## 1. Definition

A Distinguished Engineer is the highest-leverage individual contributor on the technical track. Their craft is making systems that are **easy to understand, safe to change, cheap to operate, and hard to compromise** — for the lifetime of every artifact they ship, by other people, under conditions no one anticipated.

They combine the strategic taste of an architect with the hands-on rigor of a principal engineer, treating **design, architecture, build, and maintenance as one continuous activity** — never as separate phases.

---

## 2. The Four Pillars

| Pillar | Question they answer | Primary deliverables |
|---|---|---|
| **Design** | What are we actually building, and what are its invariants? | Domain model, interfaces, contracts, edge-case map, threat model |
| **Architect** | How do the pieces fit, and which decisions can we never undo? | ADRs, system diagrams, boundary definitions, tech-stack choices |
| **Build** | How do we encode this so it's correct, clear, changeable, and safe? | Reference implementations and **the parts where being wrong is most expensive** — data models, sync engines, agent loops, payment paths, auth boundaries |
| **Maintain** | How does this stay healthy and evolvable for the artifact's lifetime? | Refactoring patterns, migration playbooks, observability, deprecation paths, eval suites |

---

## 3. Core Principles (the load-bearing eleven)

### Axes of Variation

These principles look the same on paper but manifest differently across system classes. The five dimensions that matter most:

| Axis | Spectrum |
|---|---|
| **State persistence** | Ephemeral context (agent) ↔ replay-durable history (workflow) ↔ persistent (database) |
| **Determinism** | Deterministic (compiler) ↔ replay-deterministic (workflow) ↔ probabilistic (LLM) |
| **Rollout control** | Server-instant ↔ mobile-staged (app-store review) ↔ workflow-in-flight (multi-day executions) ↔ dynamic (prompts/skills loaded at runtime) |
| **Failure taxonomy** | Partial failure / sync conflict / replay divergence / hallucination / prompt injection |
| **Artifact half-life** | Years (data schemas, public APIs) ↔ months (services) ↔ weeks (prompts, skills) |

A principle's emphasis — and occasionally its direction — shifts along these axes. Where the direction reverses, the principle flags it.

---

### 3.1 Data model is destiny

- **What it means:** The shape of your data — entities, relationships, lifecycles — constrains every downstream choice: code structure, performance, what features become possible, and what becomes prohibitively expensive. A wrong data model compounds; a right one quietly enables features for years. The dominant cost is rarely the schema redesign on paper — it is **migrating live data, and every dependent that already relies on the old shape**, which is why a model resists change long after its schema file is trivial to edit (see §4, *Migration is a first-class operation*).
- **In practice:**
  - Spend disproportionate design time on entities and their state transitions before any code is written.
  - Model relationships and cardinalities explicitly; resist "one column for now" shortcuts.
  - Defer denormalization until measurement justifies it.
- **Cross-domain instances:**
  - **Multi-platform SaaS:** Many products ship with `user.email` as a single text column. ...
  - **Workflow / orchestration:** A workflow's append-only Event History *is* the data model. ...
  - **Agentic AI (which layer is destiny):** The principle binds the layer with the long half-life, not the one measured in turns — separate the two before applying it. The **durable substrate** (long-term memory schema, the conversation/thread store, persisted tool-result contracts e.g. MCP) *is* destiny: a retrieval layer keyed only on string match cannot later support semantic similarity without redesigning the memory layer. The **ephemeral working memory** (per-turn orchestrator scratch state, the reasoning context assembled for a single request) is deliberately disposable — its entire value is that it can be rebuilt every turn, so applying "destiny" to it produces over-engineering, not safety.

---

### 3.2 Boundaries follow change rates

- **What it means:** A module, file, or service boundary is well-drawn when the things inside it change together — for the same reason, by the same people, on the same cadence. Drawing boundaries any other way creates the very coupling the boundary was supposed to eliminate.
- **In practice:**
  - Group code by what changes together (business domain), not by technical type (controllers, models, utils).
  - A service boundary is correct if teams can deploy independently; if every change touches three services, the boundary is wrong.
  - Re-draw boundaries when change patterns shift; a boundary that fit two years ago may no longer fit.
- **Cross-domain instances:**
  - **Server SaaS:** The "distributed monolith" anti-pattern (Sam Newman, *Building Microservices*, O'Reilly) appears when companies split a monolith into microservices drawn by data type — `UserService`, `OrderService`, `ProductService` — only to find every product change requires coordinated deployments across all three. Shopify took the inverse path: a single Rails deployable, but strict internal module boundaries around business capabilities like Cart, Checkout, and Inventory (Kirsten Westeinde, *Deconstructing the Monolith*, RailsConf 2019). Decoupling without distribution.
  - **Workflow / orchestration:** Temporal's Workflow vs. Activity boundary is a textbook change-rate boundary: deterministic orchestration logic (changes slowly, must be replay-safe) is separated from non-deterministic I/O (changes freely, runs as Activities).
  - **Agentic AI:** Claude Code separates Skills (auto-loaded capability packages), MCP servers (external tool/data integrations), subagents (isolated context windows), and hooks (deterministic shell triggers) — each a boundary chosen because the artifacts inside change for different reasons and at different rates (docs.claude.com).

---

### 3.3 Optimize for change, not current correctness

- **What it means:** The dominant lifetime cost of software is not writing it but reading and modifying it later — usually by people who didn't write it. A design slightly less elegant today but easier to evolve tomorrow is the right trade.
- **In practice:**
  - Make modification surfaces explicit: pure functions, clear interfaces, documented invariants.
  - Choose data formats and APIs that can evolve additively (optional fields, new variants).
  - Prefer code that five future engineers can change confidently over code one current engineer finds clever.
- **Cross-domain instances:**
  - **Server SaaS:** Stripe's API versioning pins every customer to the version they integrated against; internally, requests pass through version-translation layers that map old shapes to current internal models (Brandur Leach, *APIs as infrastructure: future-proofing Stripe with versioning*, Stripe Engineering Blog, August 2017).
  - **Multi-platform SaaS:** The mobile case is harder still — app-store review and slow user adoption mean a breaking client/server contract change can take 6+ months to retire. Versioning is not optional.
  - **Workflow / orchestration:** Temporal provides a Workflow Versioning (Patching) API — `workflow.GetVersion` (Go/Java) or `workflow.patched` (Python/TypeScript/.NET) — that lets developers safely evolve workflow code without breaking determinism for in-flight executions, by recording version markers in Event History so replays remain consistent (docs.temporal.io). Dagster offers an analogous mechanism via `code_version` and `DataVersion` for asset materializations (docs.dagster.io).
  - **Agentic AI:** Prompts, skills, and tool schemas evolve weekly. Treating them as code — versioned, reviewed, evaluated — is the equivalent discipline.

---

### 3.4 Reversibility-aware decisions

- **What it means:** Jeff Bezos's framing (2015 Amazon shareholder letter, published April 2016): some decisions are one-way doors (Type-1) — undoing them is prohibitively expensive. Others are two-way doors (Type-2) — undoing them is cheap. Ceremony, debate, and writing should match irreversibility, not perceived importance.
- **In practice:**
  - **Type-1** — public API contracts, identifier schemes, persisted data formats, identity systems → write the ADR, surface disagreement, slow down.
  - **Type-2** — frameworks inside a single service, internal class names, build tooling → decide and move.
  - The framing forces honesty about what is actually expensive to undo.
- **Cross-domain instances:**
  - **Server SaaS:** Choice of identifier strategy is a Type-1 classic — once an integer auto-increment ID is published in URLs, foreign keys, customer webhooks, and exported reports, switching to UUIDs is effectively impossible without breaking external systems.
  - **Multi-platform SaaS:** Mobile rollback extends Type-1 windows in a way backend engineers underestimate. A bad iOS or Android build sits in users' hands for days even after staged-rollout pause, because there is no way to force-uninstall a binary.
  - **Workflow / orchestration:** Deploying a breaking workflow change is Type-1 — in-flight workflows may run for weeks or months and will replay against the new code if you don't use the Patching API.
  - **Agentic AI:** Agent **actions** (sent emails, executed payments, posted comments) are runtime-irreversible even when the agent's design is freely revisable. The Type-1/Type-2 frame applies to the agent's action policy, not just its code.

---

### 3.5 Simplicity > cleverness

- **What it means:** Rich Hickey's distinction (*Simple Made Easy*, Strange Loop 2011): *simple* means one concern, no interleaving — it is objective. *Easy* means familiar, close to hand — it is subjective. Many systems that feel "easy" to start with are not simple, and the interleaved concerns surface later as compounding cost.
- **In practice:**
  - Prefer pure data and pure functions over objects with hidden state.
  - Prefer composition over inheritance.
  - Question abstractions that bundle concerns (e.g., one object handling caching + persistence + identity).
- **Cross-domain instances:**
  - **Server SaaS:** ORMs feel easier than raw SQL — `User.find(id).orders.recent` is one line. But beneath it live caching, identity maps, lazy loading, eager-loading hints, query interpretation, and transaction management. Teams routinely lose weeks to N+1 queries, stale caches, and surprising transaction boundaries that surface only at scale (Ted Neward, *The Vietnam of Computer Science*, 2006).
  - **Agentic AI:** A "god skill" that bundles context retrieval + reasoning + tool selection + persistence is the agentic equivalent — easy to ship, painful to debug. Decomposing into focused skills with explicit interfaces is harder to start with but simpler to evolve.

---

### 3.6 Boring tech is a feature — boring substrate, novel surface

- **What it means:** Every novel technology in your stack carries hidden, ongoing cost: hiring, on-call expertise, debugging unknown failure modes, upgrade paths, integration with the rest of the stack. You have a small budget of "innovation tokens" (Dan McKinley, *Choose Boring Technology*, 2015). Spend them where novelty is the strategic point. **For products whose entire value proposition is novel tech (e.g., AI agents), the rule reframes: boring substrate, novel surface.** Storage, queues, and observability stay boring; the differentiating layer is where novelty earns its keep.
- **In practice:**
  - Default to PostgreSQL, a well-understood queue (Redis, SQS), one of the major clouds, and the language your team knows best.
  - Justify any deviation in writing: what does this novel tech buy that boring tech can't?
  - Reach for novelty only where it is the strategic moat; never out of taste.
- **Cross-domain instances:**
  - **Server SaaS:** Stack Overflow has, for years, served some of the highest-traffic sites on the internet from a remarkably small server fleet running .NET, SQL Server, IIS, Redis, and a monolith — deliberately boring choices, documented publicly by Nick Craver in *Stack Overflow: The Architecture — 2016 Edition* (nickcraver.com, Feb 2016).
  - **Multi-platform SaaS:** WhatsApp's messaging backend is built primarily in Erlang/OTP — exotic to outsiders, but the canonical fault-tolerant message-passing runtime, publicly documented by Rick Reed (Erlang Factory 2012, 2014). "Boring" is defined by the problem domain.
  - **Agentic AI:** Anthropic's Model Context Protocol (MCP), announced November 25, 2024, is itself an exercise in standardizing a boring substrate — a single open protocol for tool/data integrations — so each agent product doesn't reinvent its own bespoke wiring. Novelty stays in the inference layer.

---

### 3.7 Make illegal states unrepresentable

- **What it means:** Encode invariants in the type system or schema so that invalid combinations of state cannot be constructed at all. The compiler (or schema validator) enforces correctness, replacing scattered runtime guards. Closely related: Alexis King's *Parse, don't validate* (lexi-lambda.github.io, Nov 5, 2019) — convert untrusted input into a typed value once at the boundary, then trust the type from then on.
- **In practice:**
  - Replace nullable fields and parallel boolean flags with discriminated unions / sum types.
  - Use branded types where validation matters: `EmailAddress` and `UserId` instead of `string` and `string`.
  - Make state machines explicit; each state carries only its valid fields.
- **Cross-domain instances:**
  - **Frontend SaaS:** The familiar React data-fetching pattern uses three independent booleans: `loading`, `error`, `data`. This permits illegal combinations — `loading=true, error=set, data=set`. A discriminated union — `{ status: 'idle' } | { status: 'loading' } | { status: 'error', error: E } | { status: 'success', data: T }` — makes them impossible to construct.
  - **Backend SaaS / API design:** Stripe's PaymentIntent API exposes a `status` field (`requires_payment_method`, `requires_confirmation`, `processing`, `succeeded`, etc.); each state exposes only the actions valid in that state.
  - **Workflow / orchestration:** A Temporal workflow definition *is* a state machine — states the workflow code cannot construct cannot occur in the Event History.
  - **Agentic AI:** Tool input JSON Schemas (MCP, OpenAI function-calling, Anthropic tool-use) and structured-output constraints make invalid tool invocations rejectable at the boundary, before they reach the agent's reasoning loop.

- **Tension with §3.3 (optimize for change):** These two principles pull in opposite directions. A sum type makes illegal states impossible, but **adding a variant forces an update at every site that branches on it** — the opposite of the additive, optional-field evolution §3.3 prefers. Whether that is a *safe* change or a *silent* one depends entirely on how much exhaustiveness your toolchain enforces:

  | Enforcement level | Languages / Tools | On new variant | Guarantee |
  |---|---|---|---|
  | **Compiler-enforced** | Rust, Swift `switch`; OCaml/Haskell/Scala sealed | Build fails — missed case is a compile error | Automatic; nearly all upside |
  | **Opt-in static** | TypeScript `never`-assignment; Python `typing.assert_never()` (3.11+, or `typing_extensions`) | Check fires *only if wired up* — add mypy `exhaustive-match` or pyright `reportMatchNotExhaustive` to CI, or the check is decorative | Exists only when configured |
  | **Unenforced at runtime** | Python `match` with no `case _`, plain `if/elif`, no type checker | Silently falls through — worst case: sum-type rigidity with zero safety, because nothing reports the missing handler | None |
  | **Boundary ≠ handling (Pydantic)** | `Field(discriminator=...)` / `Annotated[Union[...], Discriminator(...)]` | Rejects bad *input* at parse time; handling exhaustiveness is unguarded | Input boundary only — the type checker stops a missing *branch*, not Pydantic |

- **Resolution (the artifact-half-life axis, §3):** Apply the principle where the legal state set is *settled* — long-half-life contracts: persisted formats, public APIs, identity. Stay additively loose on short-half-life surfaces — weekly-changing prompts, skills, internal DTOs — where new variants land often and the set of legal states is still moving. Make illegal states unrepresentable where the legal set is settled; keep the schema open where it is not.

  > **Python caveat:** A sealed sum type without `assert_never` + a type checker in CI is a convention, not an invariant. In Python, the discipline that *enforces* §3.7 lives in the checker config (`pyright --strict` or mypy with `exhaustive-match`), not the type definition itself.

---

### 3.8 Design the failure modes, not the happy path

- **What it means:** The happy path nearly designs itself. Real design effort lives in: what can fail, how it is detected, what recovers automatically, what data must survive a failure, and what is communicated to humans. **Failure taxonomies differ by system class** — design must address the taxonomy that actually applies.
- **Failure taxonomy by domain:**

  | Domain | Failure modes to design for |
  |---|---|
  | **Distributed services** | Partial failure, network timeout, retry storms, cascading failure |
  | **Multi-client SaaS** | Offline writes, sync conflict, push delivery loss, app crash mid-write |
  | **Workflow / orchestration** | Replay divergence, non-deterministic activity, version skew between worker and history |
  | **Agentic AI** | Hallucination, context overflow, infinite tool loop, runaway cost, indirect prompt injection |

- **In practice:**
  - For every external call: timeout, retry policy with jitter, idempotency strategy, fallback.
  - For every state transition: partial-failure behavior, recovery path, observable signal.
  - Run failure-injection (chaos engineering) on the critical paths.
- **Cross-domain instances:**
  - **Distributed (positive):** AWS API design treats failure as a first-class input. Mutating APIs accept idempotency tokens (e.g., EC2 `RunInstances`'s `ClientToken`); SDKs implement exponential backoff with jitter by default — codified in Marc Brooker, *Exponential Backoff and Jitter* (AWS Architecture Blog, March 2015).
  - **Distributed (cautionary):** Knight Capital lost approximately $440M in roughly 45 minutes on August 1, 2012: a deployment left an obsolete code path ("Power Peg") on one of eight servers, where a re-purposed feature flag activated it and began placing erroneous orders. There was no kill switch, no anomaly detector, no automatic safe-stop (SEC Release No. 70694, October 2013). Knight ceased to exist as an independent firm within months.
  - **Agentic AI (cautionary):** In *Moffatt v. Air Canada* (2024 BCCRT 149, decided February 14, 2024), the British Columbia Civil Resolution Tribunal found Air Canada liable for negligent misrepresentation after its chatbot fabricated a bereavement-fare refund policy. Damages were modest (CAD $812.02), but the principle is load-bearing: organizations are accountable for the outputs of agents they deploy. The hallucination failure mode was undesigned.
  - **Agentic AI (research):** Indirect prompt injection — untrusted content (an email, a webpage, a document) reaching an LLM with tool access can hijack intent (Greshake et al., *Not what you've signed up for: Compromising Real-World LLM-Integrated Applications with Indirect Prompt Injection*, arXiv:2302.12173, February 2023).

---

### 3.9 Pure core, imperative shell

- **What it means:** Gary Bernhardt's pattern from *Boundaries* (SCNA 2012). Push side effects — I/O, time, randomness, mutation — to the edges. The core, where the business logic lives, takes data in and returns data out, deterministically. The shell orchestrates I/O around the pure core.
- **In practice:**
  - A request handler should be: load required data → call a pure function → write the result. Not: interleave business decisions with database calls.
  - Pure functions are trivially testable, parallelizable, and easy to reason about.
- **Cross-domain instances:**
  - **Frontend SaaS:** Redux reducers are pure functions of the form `(state, action) => newState`. Side effects are confined to middleware (thunks, sagas, RTK Query).
  - **Workflow / orchestration:** Temporal achieves durable execution by persisting an append-only Event History per workflow execution; on worker crash, the workflow code is replayed against this history to deterministically reconstruct in-memory state. Workflow code is the pure core; Activities are the imperative shell. The pattern maps perfectly.
  - **Agentic AI (the inversion):** LLM calls are inherently non-deterministic — they cannot be the pure core. The pattern flips: **the LLM call is the impure shell; the orchestration around it (prompt assembly, tool dispatch, output parsing, eval graders) should be the deterministic core.** Treating the LLM as a pure function of its inputs is one of the most common architectural mistakes in this domain.

---

### 3.10 Strangler fig, never big-bang

- **What it means:** Martin Fowler's pattern (*StranglerFigApplication*, martinfowler.com, June 2004), named after the strangler fig tree that grows around its host. Replace a system by routing traffic incrementally to the new implementation while the old keeps serving — until the new carries all traffic and the old can be removed. Big-bang rewrites fail at very high rates because they require freezing a moving target and assume perfect knowledge of the old system's behavior.
- **In practice:**
  - Put the legacy system behind a stable interface (router, proxy, or facade).
  - Build the new implementation behind the same interface; route a small slice of traffic to it.
  - Run shadow mode (compare outputs without serving), then gradually expand traffic, then remove the old.
- **Cross-domain instances:**
  - **Server SaaS (cautionary):** Joel Spolsky's *Things You Should Never Do, Part I* (joelonsoftware.com, April 2000) describes Netscape's decision to rewrite Communicator from scratch. The rewrite consumed years during which Netscape shipped no competitive browser, and Internet Explorer captured the market.
  - **Multi-platform SaaS (positive):** Facebook's Hack/HHVM transition (Julien Verlaguet & Alok Menghrajani, *Hack: a new programming language for HHVM*, Engineering at Meta, March 20, 2014) introduced a gradually-typed PHP dialect that interoperates file-by-file with existing PHP — letting Meta migrate nearly its entire codebase incrementally rather than rewrite it.
  - **Agentic AI:** Routing a fraction of traffic to a new prompt or model behind a stable agent interface — with output comparison in shadow mode before promotion — is the same shape as classical traffic-routed migration.

---

### 3.11 Adversarial-aware design

- **What it means:** Failure-mode design (§3.8) asks *"what breaks?"* Adversarial design asks *"what can be made to break by someone who wants it to?"* The two produce different mitigations: retries help with the former; threat modeling, least privilege, sandboxing, and blast-radius minimization help with the latter. Distinguished engineers treat security as a design input, not a security-team handoff.
- **In practice:**
  - Threat-model at the design stage (STRIDE, attack trees, or trust-boundary diagram on the whiteboard).
  - Default to deny; require justification for each granted capability.
  - Minimize blast radius: a compromised component should compromise the smallest possible adjacent surface.
  - Treat secrets, identity, and authorization boundaries as Type-1 decisions (§3.4).
- **Cross-domain instances:**
  - **Server SaaS (cautionary):** The Equifax 2017 breach exposed personal information of approximately 145.5 million U.S. consumers (GAO-18-559, *Data Protection: Actions Taken by Equifax and Federal Agencies in Response to the 2017 Breach*, September 2018). Attackers exploited Apache Struts CVE-2017-5638 — a vulnerability the Apache Software Foundation patched on March 7, 2017. Attackers first accessed Equifax systems on May 13, 2017; the intrusion was not detected until July 29, 2017. The patch was available for two months before exploitation; the system that should have applied it wasn't.
  - **Supply chain (cautionary):** SolarWinds SUNBURST (disclosed by FireEye/Mandiant, December 13, 2020; CISA Alert AA20-352A) compromised the build pipeline for Orion software updates released between March and June 2020. SolarWinds' SEC filing stated up to 18,000 customers may have downloaded the trojanized update. The threat model assumed attackers attack the application; this attack went a layer below.
  - **Agentic AI:** Prompt injection is the new SQL injection. The term was coined by Simon Willison (*Prompt injection attacks against GPT-3*, simonwillison.net, September 12, 2022), building on an exploit first surfaced by Riley Goodside. Stanford student Kevin Liu used a direct prompt injection on February 8, 2023 to extract Bing Chat's hidden system prompt, exposing its internal codename "Sydney." The defense pattern is identical to SQL injection — never trust input reaching an interpreter; isolate untrusted content from privileged operations.

---

## 4. Operating Practices

- **ADRs for irreversible decisions** — surface disagreement on the record before lock-in. Universal across SaaS, workflow, and agentic systems.

- **Observability from day one** — adapted to the domain:
  - Backends: logs, metrics, distributed traces
  - Client apps: crash reporting, RUM (real user monitoring)
  - Workflow / orchestration: Event History timelines, worker dashboards
  - Agents: trace replay, token-cost telemetry, eval scores

- **Feature flags decouple deploy from release** — ship dark, enable gradually, kill-switch instantly. For mobile clients, server-side flag checks are required because binaries cannot be force-updated.

- **Migration is a first-class operation** — schema migrations (SQL), workflow versioning (e.g., Temporal Patching), and prompt/skill version rollouts each have a playbook, not improvisation.

- **Continuous refactor (Boy Scout Rule)** — leave each file slightly better; daily compounds beat scheduled rewrites.

- **Tests *and* evals as design tools** — if it's hard to test, the design is wrong. For probabilistic surfaces (LLMs, ML), eval suites complement unit tests by capturing regressions tests cannot — see Hamel Husain, *Your AI Product Needs Evals* (hamel.dev, March 29, 2024), who frames evals as a superset of assertion-style tests, not a replacement.

- **Eval suites are first-class artifacts, not scripts** — the principles above apply to the eval harness itself, not just to the system under test:
  - **Data model is destiny (§3.1):** the eval dataset and the grader/rubric schema decide what any score can *mean*. A rubric that collapses "wrong" and "unhelpful" into one pass/fail can never later distinguish them without re-grading the whole set.
  - **Migration is first-class (above):** changing a grader, rubric, or judge model **breaks comparability with every prior score** — 0.82 today is not 0.82 from last month. That is a migration with a versioned dataset and a re-baseline step, not a silent edit; pin which dataset and grader version produced which number.
  - **Boundaries follow change rates (§3.2):** keep weekly-changing prompts on one side of a boundary and slow-moving golden datasets and grader logic on the other; co-locating them couples a stable asset to a volatile one and corrupts your trend line.

---

## 5. Anti-Patterns (what they explicitly do NOT do)

- ❌ Introduce new technology without an ADR explaining what existing tech failed.
- ❌ Add an abstraction before the third concrete case (Rule of Three) — *caveat:* design systems and shared agent capabilities (skills, tools) may justify abstraction earlier when consistency itself is the value.
- ❌ Ship code without considering failure modes, observability, **and threat model**.
- ❌ Design for hypothetical future requirements (YAGNI).
- ❌ Perform big-bang rewrites.
- ❌ Optimize before measuring — including token-cost optimization in agentic systems.
- ❌ Leave dead code, dead flags, dead services — and for agents, orphaned skills, unused tools, stale memory entries.
- ❌ Accept "magic" — every behavior must have a traceable origin. For agents, prompt-driven behavior must be explainable; black-box "it just works" agent loops are anti-pattern.

---

## 6. Mental Models Toolkit

A library of patterns the engineer recognizes within minutes of seeing a problem — spanning all four system classes, not just distributed backends.

#### Distributed & Concurrent

- Backpressure — producer/consumer rate mismatch
- CAP & PACELC — consistency vs. availability under partition
- Idempotency & at-least-once delivery — any distributed call
- Race conditions, deadlocks, livelocks — any concurrent code
- Two-generals / consensus — distributed agreement
- Cache invalidation & cache stampede — read-heavy systems

#### Stateful & Evolving

- State machines — anything with a status field
- Bounded contexts (DDD) — where one model ends and another begins
- Eventual vs. strong consistency — replicated state
- Conway's Law — org shape ⇒ system shape

#### Workflow & Multi-Client

- **Determinism & replay** — workflows, simulations, deterministic builds
- **Saga / compensation** — distributed transactions decomposed into reversible steps with compensating actions on failure (Garcia-Molina & Salem, *Sagas*, ACM SIGMOD 1987 — original scope was long-lived transactions in a single database; the microservices-saga framing is a later reinterpretation)
- **Offline-first & conflict resolution** — multi-client sync (Shapiro, Preguiça, Baquero, Zawirski, *Conflict-free Replicated Data Types*, SSS 2011 — formalized and named CRDTs, building on prior eventual-consistency work)
- **Operator UX as product** — the dashboard, retry button, and timeline view *are* the product for ops users (Temporal, Dagster, Airflow)

#### Probabilistic Systems

- **Probabilistic correctness & evals** — output is "right enough most of the time," not "right"; eval suites capture regressions tests cannot
- **Token / context budget** — the LLM-era analog of memory budget in embedded systems
- **Prompt injection (direct & indirect)** — untrusted input reaching an interpreter; treat exactly like SQL injection (Willison 2022; Greshake et al. 2023)

---

## 7. The One-Line Distillation

> **A Distinguished Engineer's job is to make the right things easier to build and the wrong things harder to build — across the whole product, for the lifetime of every artifact, against accidents and adversaries — through strategic taste, architectural rigor, and hands-on craft.**

---

## 8. Self-Test (am I operating at this level?)

- Can I name the **invariants** of every system I own?
- Have I written an **ADR** for every irreversible decision in my domain?
- Is my system **observable** end-to-end, in the form appropriate to its class — server traces, client crash reports, workflow timelines, agent eval scores?
- Could a new engineer **understand and change** my code without my help?
- When something breaks at 3am, does the **playbook** already exist?
- Am I **deleting** as much code as I'm writing?
- Have I **threat-modeled** this — assuming an adversary, not just an accident?
- Do I know the **unit economics** ($ per request, $ per active user, $ per agent action) of this system?
- For probabilistic surfaces, do my **evals** catch regressions my tests can't?

If any answer is "no," that's the next thing to fix.
