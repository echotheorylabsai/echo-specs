# Agent-First Software Engineering: A Primer for Greenfield Projects

*Generalizable principles, patterns, and architectural choices for building software with coding agents.*

**Source**: Distilled from *Harness engineering leveraging Codex in an agent-first world* by Ryan Lopopolo (OpenAI), which describes a five-month experiment building a production SaaS product with **0 lines of manually-written code** (~1M LOC, ~1,500 PRs, 3→7 engineers).

**Audience**: Lead engineers, architects, and CTOs leading greenfield projects with coding agents (Codex, Claude Code, Gemini CLI, Aider, Cursor agent mode, or similar).

**How to read this primer**: Sections 1–7 derive directly from the source. Section 8 is a synthesized implementation sequence, and Section 9 includes two extensions beyond the source. The Appendix maps every section to its source provenance for verification.

---

## Contents

1. The Paradigm Shift
2. First Principles
3. The Substrate: Repository as Agent's World
4. Closing the Feedback Loops
5. Operating at Throughput
6. The Autonomy Progression
7. Anti-Patterns
8. Implementation Sequence
9. Open Questions and Limits
10. Appendix: Source Mapping

---

## 1. The Paradigm Shift

When agents write the code, the engineer's primary job changes. Instead of typing implementation, you **design environments, specify intent, and build feedback loops** the agent operates inside.

**The operating contract**: *Humans steer. Agents execute.*
- Humans prioritize work, translate user feedback into acceptance criteria, and validate outcomes.
- Agents produce everything else: application logic, tests, CI, documentation, observability, and internal tooling.

**The new bottleneck**: human attention. The fixed constraint is human time and attention — not typing speed, not review throughput. Design decisions are best evaluated against that single constraint.

**The key reframing**: When the agent fails, ask *"what capability is missing, and how do we make it both legible and enforceable for the agent?"* — not *"how do I prompt it harder?"* The fix is almost never "try harder." It's almost always to make some piece of context, tooling, or invariant **legible and enforceable** in the repository itself.

> *Note on portability: While the source describes Codex specifically, the principles below apply to any coding agent that can access a repository, use tools, and respond to prompts. The source itself notes that pulling system context into agent-legible form increases leverage "not just for Codex, but for other agents… working on the codebase as well." Specific implementation details — multi-hour autonomous runs, agent-to-agent review chains — depend on the agent's capabilities and your investment.*

---

## 2. First Principles

Seven load-bearing rules. Each is a consolidation of explicit positions in the source; the naming below is this primer's, for memorability.

| # | Principle | Statement | Source section |
|---|---|---|---|
| 1 | **Legibility Principle** | What the agent cannot access in-context while running effectively *does not exist* for it. | "Agent legibility is the goal" |
| 2 | **Map, Not Manual** | Give the agent a map (small, stable entry point + pointers), not a 1,000-page instruction manual. Use progressive disclosure. | "We made repository knowledge the system of record" |
| 3 | **Invariants over Implementations** | Enforce boundaries; allow autonomy within. Don't micromanage *how* — enforce *what must be true*. | "Enforcing architecture and taste" |
| 4 | **Mechanical over Documented** | When documentation falls short, promote the rule into code (custom linters, structural tests, CI gates). | "Enforcing architecture and taste" |
| 5 | **Continuous over Periodic** | Pay down drift continuously, like garbage collection. Daily and automated, not Friday and manual. | "Entropy and garbage collection" |
| 6 | **Capability over Effort** | When the agent fails, fix the environment (tools, abstractions, docs), not the prompt. | "Redefining the role of the engineer" |
| 7 | **Throughput-Aware Tradeoffs** | Conventional norms (gated merges, blocking on flakes, deep human review) become counterproductive when agents far outpace human attention. Match policy to regime. | "Throughput changes the merge philosophy" |

---

## 3. The Substrate: Repository as Agent's World

Everything the agent will ever see lives in the repo. Three substrate decisions are Day-1 work.

### 3.1 Knowledge Architecture

**Don't put everything in `AGENTS.md`.** A monolithic instruction file fails in four predictable ways:

| Failure mode | Why it happens |
|---|---|
| Context is scarce | A giant file crowds out the task, the code, and relevant docs |
| Too much guidance becomes non-guidance | When everything is "important," nothing is — agents pattern-match locally instead of navigating intentionally |
| It rots instantly | Stale rules pile up; humans stop maintaining; the file becomes an attractive nuisance |
| Hard to verify | A single blob resists mechanical checks (coverage, freshness, ownership, cross-links) |

**Use a structured `docs/` tree as the system of record.** `AGENTS.md` becomes a ~100-line table of contents with pointers. From the source:

```
AGENTS.md   ARCHITECTURE.md   DESIGN.md   FRONTEND.md   PLANS.md
PRODUCT_SENSE.md   QUALITY_SCORE.md   RELIABILITY.md   SECURITY.md
docs/
├── design-docs/      (index.md, core-beliefs.md, …)
├── exec-plans/       (active/, completed/, tech-debt-tracker.md)
├── generated/        (db-schema.md, …)
├── product-specs/    (index.md, feature specs, …)
└── references/       (design-system-llms.txt, nixpacks-llms.txt, uv-llms.txt, …)
```

**Plans are first-class artifacts.**
- Lightweight ephemeral plans for small changes.
- Persistent execution plans (with progress and decision logs) for complex work.
- Active plans, completed plans, and known technical debt — all versioned and co-located so the agent operates without external context.
- A quality document grades each product domain and architectural layer, tracking gaps over time.

**Enforce mechanically.** Custom linters and CI jobs validate that the knowledge base is current, cross-linked, and structured correctly. A recurring **"doc-gardening" agent** scans for stale documentation that no longer reflects real code behavior and opens fix-up PRs.

**Why this matters: the agent's knowledge is a bounded bubble.**

```
                ┌──────────────────────┐
                │   Agent's Knowledge  │ ◄── encoded as
                └──────────────────────┘     markdown in repo
                            ▲
                            │  (must be encoded to be visible)
            ┌───────────────┼───────────────┐
            │               │               │
       Google Docs     Slack threads   Tacit knowledge
       (invisible)      (invisible)     (invisible)
```

A Slack discussion that aligned the team on an architectural pattern is invisible to the agent in the same way it would be unknown to a new hire joining three months later. **If it isn't in the repo, it doesn't shape behavior.**

### 3.2 Layered Domain Architecture (Day-1 Prerequisite)

> Architecture you would normally postpone until you have hundreds of engineers becomes a Day-1 prerequisite. Constraints are what allow speed without decay or architectural drift.

Each business domain divides into a fixed set of layers with strictly validated dependency directions:

```
   ┌───────┐
   │ Utils │  (outside boundary; feeds Providers)
   └───┬───┘
       │
       ▼
   ┌─ Business Logic Domain ─────────────────────────────────────┐
   │                                                             │
   │  Types ──► Config ──► Repo ──► Service ──► Runtime ──► UI   │
   │                                   ▲                         │
   │                                   │                         │
   │                            ┌─────────────┐                  │
   │                            │  Providers  │ ──► App Wiring   │
   │                            └─────────────┘     + UI         │
   │                                                             │
   └─────────────────────────────────────────────────────────────┘

Forward dependency chain (within a business domain):
   Types → Config → Repo → Service → Runtime → UI

Cross-cutting concerns (auth, connectors, telemetry, feature flags)
enter the chain ONLY via the Providers seam. Anything else is
disallowed and enforced mechanically.
```

**Constraints are enforced mechanically** via custom linters (often agent-generated themselves) and structural tests. Examples of "taste invariants" encoded as code:
- Structured logging
- Naming conventions for schemas and types
- File size limits
- Platform-specific reliability requirements

**Critical detail: error messages embed remediation.** Because the linters are custom, their failure messages inject *fix instructions directly into the agent's context*. A failing rule becomes an executable repair signal — not just a complaint.

**Where this lands**: enforce boundaries centrally; allow autonomy locally. Care deeply about correctness and reproducibility at the seams. Within the seams, allow agents significant freedom in how solutions are expressed. The resulting code may not match human stylistic preferences — and that is acceptable, as long as it is correct, maintainable, and legible to *future* agent runs.

### 3.3 Tech Selection: Boring Wins

Prefer dependencies and abstractions the agent can fully internalize and reason about in-repo:
- **Composability** over clever abstractions
- **API stability** over novelty
- **Strong representation in training data** over exotic frameworks

When upstream behavior is opaque, **reimplement small subsets in-repo**. The source's example: instead of pulling in a generic `p-limit`-style concurrency package, they implemented their own map-with-concurrency helper — tightly integrated with their OpenTelemetry instrumentation, with 100% test coverage, behaving exactly the way the runtime expects.

---

## 4. Closing the Feedback Loops

For the agent to be autonomous, it must perceive the running system, validate its own work, and recover from failure — without humans copying state into a CLI. Three legibility surfaces matter.

### 4.1 Per-Task Isolation

- App is **bootable per git worktree** — agents launch and drive one instance per task.
- Observability stack is **ephemeral per worktree** — torn down when the task completes.
- Each task gets a fully isolated environment: app + logs + metrics + traces.

### 4.2 Application Legibility (UI)

Drive the running app through Chrome DevTools Protocol (or equivalent). The agent gets skills for DOM snapshots, screenshots, and navigation, then validates by looping until clean:

```
┌────────┐         ┌─────┐         ┌──────────────┐
│ Agent  │─select─►│ App │         │ DevTools/CDP │
│        │ snapshot BEFORE ──────► │              │
│        │ trigger UI path ──────► │              │
│        │◄──── runtime events ────│              │
│        │ snapshot AFTER ───────► │              │
│        │ apply fix + restart ──► │              │
└───┬────┘                          └──────────────┘
    │
    └─── loop until clean (re-run validation)
```

This enables the agent to reproduce bugs, validate fixes, and reason about UI behavior directly — no human in the screenshot loop.

### 4.3 Observability Legibility (Logs / Metrics / Traces)

Logs, metrics, and traces are queryable by the agent over a local stack. The source uses Vector + Victoria{Logs,Metrics,Traces} with LogQL / PromQL / TraceQL, but the *pattern* is what matters: **structured signals plus agent-queryable APIs.**

```
                ┌─────────┐
   App ─OTLP──► │ Vector  │  (local fan-out)
                └────┬────┘
                     │
        ┌────────────┼────────────┐
        ▼            ▼            ▼
    Victoria     Victoria     Victoria
      Logs       Metrics       Traces
        │            │            │
      LogQL       PromQL       TraceQL
        │            │            │
        └────────────┼────────────┘
                     ▼
              ┌────────────┐
              │   Agent    │ ──► query, correlate, reason
              └─────┬──────┘ ──► implement fix
                    │        ──► restart, re-run, verify
                    ▼
              ┌──────────┐
              │ Codebase │
              └──────────┘
```

With this surface available, prompts like *"ensure service startup completes in under 800ms"* or *"no span in these four critical user journeys exceeds two seconds"* become tractable. The source reports single agent runs of upwards of six hours on a single task — often overnight.

### 4.4 Self-Review and Agent-to-Agent Review

After opening a PR, the agent:
1. Reviews its own changes locally.
2. Requests additional specific agent reviews (locally and/or in the cloud).
3. Responds to feedback (human or agent).
4. Iterates in a loop until all reviewers are satisfied.

The agent uses standard development tools (`gh`, local scripts, repo-embedded skills) to gather context. Humans never copy/paste into a CLI.

> Human review is *not required for every PR* — most review moves agent-to-agent. Humans engage when escalation requires judgment (see Section 6, item 10). The agent-to-agent iteration loop is fast (agent-paced, not human-paced), which is why short-lived PRs (Section 5.1) and iterate-until-satisfied review chains coexist without contradiction. This all works only because the substrate (Section 3) makes work mechanically verifiable.

---

## 5. Operating at Throughput

When agents far outpace human review, conventional norms become counterproductive. The system must self-maintain.

### 5.1 Merge Philosophy

- **Minimal blocking merge gates.** Pull requests are short-lived.
- **Test flakes** are addressed with follow-up runs, not by blocking progress indefinitely.
- Logic: when corrections are cheap and waiting is expensive, optimistic merging wins.
- *This would be irresponsible at low throughput. At high agent throughput, it is often the right tradeoff.*

### 5.2 Drift and Garbage Collection

Agents replicate patterns already in the repo — including uneven or suboptimal ones. Without intervention, drift compounds.

**Manual cleanup does not scale.** The source describes a phase where the team spent every Friday (20% of the week) cleaning up "AI slop" — *accumulated low-quality patterns the agent had replicated from prior code (uneven helpers, inconsistent error handling, redundant utilities), which compound if left unchecked*. It did not scale. They replaced it with two mechanisms:

**1. Golden Principles** — opinionated, mechanical rules encoded into the repo. Examples directly from the source:
- *Prefer shared utility packages over hand-rolled helpers* — keeps invariants centralized.
- *Don't probe data "YOLO-style"* — validate at boundaries or rely on typed SDKs so the agent cannot accidentally build on guessed shapes.

**2. Background GC Agents** — recurring agent tasks on a regular cadence that:
- Scan for deviations from golden principles.
- Update quality grades for each domain and layer.
- Open targeted refactoring PRs.
- Most are reviewable in under a minute and automerged.

**Pattern**: technical debt is a high-interest loan. Pay it continuously in small increments rather than letting it compound and tackling it in painful bursts. **Human taste is captured once** (as a rule), then **enforced continuously on every line of code** from then on.

---

## 6. The Autonomy Progression

The source describes a threshold its repository "recently crossed" where the agent can drive a full feature end-to-end from a single prompt.

**The capability checklist (verbatim from source):**

| # | Capability |
|---|---|
| 1 | Validate the current state of the codebase |
| 2 | Reproduce a reported bug |
| 3 | Record a video demonstrating the failure |
| 4 | Implement a fix |
| 5 | Validate the fix by driving the application |
| 6 | Record a second video demonstrating the resolution |
| 7 | Open a pull request |
| 8 | Respond to agent and human feedback |
| 9 | Detect and remediate build failures |
| 10 | Escalate to a human only when judgment is required |
| 11 | Merge the change |

**Where your team sits on this checklist depends on accumulated investment** in the substrate (Section 3), feedback loops (Section 4), and drift management (Section 5).

*Note: this decomposition into substrate / feedback loops / drift management is this primer's framing. The source describes the threshold as resulting from "encoding the development loop directly into the system — testing, validation, review, feedback handling, and recovery."*

> **Source caveat (verbatim)**: *"This behavior depends heavily on the specific structure and tooling of this repository and should not be assumed to generalize without similar investment — at least, not yet."*

---

## 7. Anti-Patterns

Patterns that kill agent-first projects. All are derived from the source — either as explicitly named failure modes (**direct**) or as inversions of stated positive principles (**inversion**).

| Anti-pattern | Why it fails | Provenance |
|---|---|---|
| **Monolithic `AGENTS.md`** | Crowds out context; everything-is-important means nothing is; rots fast; resists mechanical checks | direct |
| **"Try harder" loops** | Agent failure is almost always an environment problem; pushing harder on prompts wastes attention | direct |
| **Knowledge in Slack / Google Docs / heads** | Invisible to the agent — same as not existing. Encode into the repo as markdown | direct |
| **Hand-rolling helpers when shared utilities exist** | De-centralizes invariants; opens drift surface | inversion (of golden principle 1) |
| **"YOLO" data probing** | Agent builds on guessed shapes; enforce typed SDKs or validate at boundaries | direct (golden principle 2) |
| **Optimizing for human stylistic preference** | Agent legibility ≠ human aesthetics. The bar is correct, maintainable, legible to *future agent runs* | inversion (of taste-vs-correctness framing) |
| **Importing libraries the agent cannot reason about** | Opaque upstream behavior creates hidden failure modes; prefer "boring" or reimplement small subsets | inversion (of "boring tech" preference) |
| **Batch / Friday-style cleanup** | Does not scale; drift compounds during the week. Enforce continuously | direct |
| **Gating merges as if humans were the throughput** | When agents are the throughput, optimistic merging plus follow-up correction is cheaper than blocking | inversion (of throughput-aware merge philosophy) |

---

## 8. Implementation Sequence

> *Source mapping: The source does not present an explicit playbook. The sequence below is **synthesized from the source's narrative chronology** — the order in which problems emerged and were solved — and arranged by dependency (each phase compounds on prior phases). Use it as a recommended order, not a verbatim quote from the source.*

| Phase | Investment | Source signal |
|---|---|---|
| **1. Bootstrap** | Empty repo; agent-generated initial scaffold (CI, formatting, package manager, framework, initial AGENTS.md) | "First commit … late August 2025. The initial scaffold … was generated by Codex CLI" |
| **2. Architecture as Day-1** | Layered domain model with explicit dependency directions; cross-cutting only via Providers | "With coding agents, it's an early prerequisite" |
| **3. Mechanical Enforcement** | Custom linters with **remediation messages embedded in errors**; structural tests; CI gates | "Custom linters … error messages to inject remediation instructions into agent context" |
| **4. Knowledge Architecture** | Move from monolithic to docs/ tree; AGENTS.md as ~100-line TOC; plans as first-class artifacts; doc-gardening agent | "Instead of treating AGENTS.md as the encyclopedia, we treat it as the table of contents" |
| **5. Application Legibility** | App bootable per worktree; UI driving via DevTools/CDP; ephemeral observability stack with queryable logs/metrics/traces | "As code throughput increased, our bottleneck became human QA capacity" → built UI + observability legibility |
| **6. Self-Review Loops** | Agent self-review; agent-to-agent review chains; iterate until reviewers satisfied | "Codex to review its own changes locally, request additional specific agent reviews … iterate in a loop" |
| **7. Drift Management** | Golden principles encoded in code; recurring GC agents that grade quality and open refactor PRs | "Initially, humans addressed this manually … that didn't scale … we started encoding 'golden principles'" |
| **8. End-to-End Autonomy** | Agent drives full feature lifecycle from a single prompt; human escalation only when judgment is required | "the repository recently crossed a meaningful threshold" |

**Why the order matters.** Each phase compounds:
- Phase 5 has limited value without Phase 2 — there is nothing stable to validate against.
- Phase 7 has limited value without Phase 4 — there is no map of what "correct" looks like.
- Phase 8 is emergent; it appears once Phases 1–7 are mature.

Do not skip ahead.

---

## 9. Open Questions and Limits

The source is candid about what is not yet known. These frontiers are worth tracking.

### From the source

- **Long-term architectural coherence.** How does coherence evolve over *years* in a fully agent-generated system? The source has months of data, not years.
- **Where human judgment compounds vs. decays.** "We're still learning where human judgment adds the most leverage and how to encode that judgment so it compounds."
- **Model capability evolution.** As models become more capable, which scaffolding becomes obsolete versus more important?

### Extensions added by this primer (not in the source)

- **Cost economics.** The source does not discuss compute cost. Per-task ephemeral observability stacks, agent-to-agent review chains, multi-hour autonomous runs, and continuous GC agents all consume budget. Engineering teams adopting these patterns should track $-per-merged-PR as a first-class metric. *Open question: at what scale does the economics break down, and which scaffolding components have the worst cost-to-leverage ratio?*
- **Brownfield generalization.** The source explicitly limits its claims to its specific repository, which was greenfield from commit zero. The hardest case — applying these patterns to a 5–10 year old enterprise codebase with messy invariants and partial test coverage — is not addressed. *Open question: which patterns transfer to legacy code with no agent-generated greenfield substrate to bootstrap from?*

---

## 10. Appendix: Source Mapping

For verification, here is which sections of this primer derive directly from the source versus which are extensions added by this primer.

| Section | Provenance |
|---|---|
| 1. The Paradigm Shift | **Direct** — consolidated framing of source's opening |
| 2. First Principles | **Derived** — every principle traceable to a specific source section; the consolidation, naming, and tabular framing are this primer's |
| 3. The Substrate | **Direct** — knowledge tree, layered architecture, mechanical enforcement, knowledge-bubble framing, and "boring tech" preference are all in the source (with their diagrams) |
| 4. Feedback Loops | **Direct** — worktree isolation, UI driving via Chrome DevTools, Vector + Victoria observability stack, self-review and agent-to-agent loops are all in the source |
| 5. Operating at Throughput | **Direct** — merge philosophy, golden principles, GC agents are all in the source |
| 6. Autonomy Progression | **Direct** — 11-item capability list and source caveat are verbatim. The "function of accumulated investment in Sections 3/4/5" framing is this primer's decomposition |
| 7. Anti-Patterns | **Direct + inversions** — explicit anti-patterns and inversions of positive recommendations from the source. Consolidation into one table is this primer's |
| 8. Implementation Sequence | **Synthesis** — phases derived from source's narrative chronology; the source does not present an explicit playbook |
| 9. Open Questions | **Source items + 2 labeled extensions** (cost economics, brownfield generalization) |

**Primary source**: Lopopolo, R. *Harness engineering leveraging Codex in an agent-first world*. OpenAI. PDF in working directory; describes work from late August 2025 through approximately early 2026.

