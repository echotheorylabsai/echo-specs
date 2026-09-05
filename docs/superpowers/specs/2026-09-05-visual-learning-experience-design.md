# Design: Interactive Learning Experience for the Craft & Agent-First Docs

**Date:** 2026-09-05
**Status:** Draft for review

---

## 1. What we are building

A single self-contained HTML page that teaches `software-product-craft.md` (11 principles) and `agent-first-engineering-primer.md` (12 principles) to working engineers.

**Three surfaces in one page:**

- **The Spine** — 10 chapters in a deliberate order. Teaches the load-bearing ideas properly. What a first-time reader follows.
- **The Map** — all 23 principles, terse and expandable, filterable. What a returning reader uses to look something up.
- **The Panels** — the material that is neither a principle nor a chapter: glossary, anti-patterns, mental models, where to start, open questions, provenance, self-test.

**Approved decisions** (from brainstorming): spine + map; interactivity only where a doc describes a decision fork; spine deep / map complete-but-terse with nothing dropped; craft first, then agents, with explicit bridges.

---

## 2. Goals and non-goals

**Goals**

1. A reader who finishes the spine can state the two dials, name the three failure classes, and decide a merge gate without looking anything up.
2. A returning reader finds any of the 23 principles in under 15 seconds.
3. First view is calm — roughly one screen, nothing expanded.
4. Every assertion is traceable to a line in one of the two docs.

**Non-goals**

- Not a replacement for `§0` in each doc, which already serves agents as a checklist.
- No accounts, no server, no persistence beyond the viewer's own browser.
- No content that is not in the docs.

---

## 3. The fidelity rule (hard constraint)

**Every factual claim, example, number, citation, rule, and widget state must appear in one of the two docs.**

Permitted: compressing a paragraph to a phrase; turning a documented table into a widget whose states are exactly that table's rows; turning a documented sequence into a stepper whose steps are exactly the doc's steps; connective narration that asserts nothing new.

Forbidden: new examples, incidents, numbers, or citations; sharpening a hedge; merging two principles into a claim neither makes; widget states the source does not name.

**Provenance must survive.** The primer separates what came from the source essay (§10, lines 561–567) from what it generalized itself (569–573). That split is preserved on the page. These hedges are protected and reproduced without sharpening:

| Location | Hedge |
|---|---|
| primer 14 | "as described, involves no money movement or regulated data" |
| primer 264–265 | semantic staleness "is not a solved problem" |
| primer 375 | "'Loop until clean' is a technique, not a convergence guarantee" |
| primer 409 | no-human-review "defensible only inside a contained, non-regulated blast radius" |
| primer 488 | "should not be assumed to generalize without comparable investment" |
| primer 577 | cross-family review: "the gain is recall, not independence" |
| craft 237 | MCP "is not yet *boring*" |
| craft 312 | a sum type without a checker "is a convention" |

**Verification:** after implementation, a line-by-line pass maps every assertion to a doc line number. The mapping table has three categories — *quoted*, *compressed*, *connective* — and connective lines are checked, not exempted. Anything unmappable is cut.

---

## 4. Coverage matrix

Every source heading has exactly one home. This is what makes "nothing dropped" true.

### software-product-craft.md

| Source | Home |
|---|---|
| §0 proportionality rule | Chapter 1 |
| §0 eleven principles | Map cards (rule line verbatim) |
| §0 operating rules | Self-test panel (checklist) |
| §1 The Standard | Chapter 1 opening frame |
| §2 Four Pillars | Chapter 1, level-3 detail |
| §3 Axes of Variation | Chapter 1 |
| §3.1 Data model | Chapter 2 |
| §3.2 Boundaries | Chapter 5 |
| §3.3 Optimize for change | Chapter 5 |
| §3.4 Reversibility | Chapter 2 |
| §3.5 Simplicity | Chapter 3 |
| §3.6 Boring tech | Chapter 2 |
| §3.7 Illegal states | Chapter 3 |
| §3.8 Failure modes | Chapter 4 |
| §3.9 Pure core | Chapter 3 |
| §3.10 Strangler fig | Chapter 5 |
| §3.11 Adversarial | Chapter 4 |
| §4 ADRs | Chapter 2 |
| §4 Observability | Chapter 4 |
| §4 Migration, flags, small changes, refactor, tests, evals | Chapter 5 |
| §5 Anti-patterns | Anti-patterns panel |
| §6 Mental models | Mental models panel |
| §7 Distillation | Page close |
| §8 Self-test | Self-test panel |

### agent-first-engineering-primer.md

| Source | Home |
|---|---|
| §0 Terms | Glossary panel (verbatim) |
| §0 Three reframes | Chapter 6 (three numbered lines, verbatim) |
| §0 Proportionality | Chapter 6 |
| §0 Twelve principles | Map cards (rule line verbatim) |
| §0 Operating rules | Self-test panel (checklist) |
| §1 Paradigm shift, failure diagnostic | Chapter 6 |
| §2 P0 | Chapter 7 |
| §2 P1 | Chapter 9 |
| §2 P2 | Chapter 6 |
| §2 P3, P4, P5, P6 | Chapter 8 |
| §2 P7, P8 | Chapter 10 |
| §2 P9, P10 | Chapter 9 |
| §2 P11 | Chapter 7 |
| §3.1 Knowledge architecture | Chapter 8 |
| §3.2 Layers and seam | Chapter 8 |
| §3.3 Tech selection | Chapter 8 |
| §4 Verification tiers, build tiers | Chapter 7 |
| §4.1–4.3 Isolation, UI, observability | Chapter 7 |
| §4.4 Review and trust chain | Chapter 9 |
| §5.1 Merge philosophy | Chapter 10 |
| §5.2 Drift and cleanup | Chapter 10 |
| §6 Autonomy progression | Chapter 10 |
| §7 Anti-patterns | Anti-patterns panel |
| §8 Implementation sequence | "Where to start" panel |
| §9 Open questions | Open questions panel |
| §10 Sources and uncertainties | Provenance panel |

---

## 5. Structure: the Spine

Ten chapters, five per act. Titles are imperative and *are* the rule, because that is what gets recalled.

### Act I — Craft: what good software is

| # | Chapter | Source | Widget |
|---|---|---|---|
| 1 | **Size the effort to the stakes** | §1, §0 proportionality, §3 axes, §2 pillars | Two dials |
| 2 | **Decide what you cannot undo** | §3.4, §3.1, §3.6, §4 ADRs | One-way / two-way sorter |
| 3 | **Keep the core simple, typed, and pure** | §3.5, §3.7, §3.9 | Exhaustiveness ladder |
| 4 | **Design the failure, and the adversary** | §3.8, §3.11, §4 observability | — |
| 5 | **Change it safely, forever** | §3.2, §3.3, §3.10, §4 migration/tests/flags/small changes/evals | Expand → migrate → contract stepper |

### Act II — Agents: what changes when agents write the code

| # | Chapter | Source | Widget |
|---|---|---|---|
| 6 | **What actually changes** | §1, §0 three reframes, §0 proportionality, P2 | Failure diagnostic |
| 7 | **Close the loop** | P0, P11, §4 tiers, §4.1–4.3 | Build-tier trigger |
| 8 | **Build the agent's world** | P3, P4, P5, P6, §3.1, §3.2, §3.3 | — |
| 9 | **Who says it is correct?** | P1, P9, P10, §4.4 | — |
| 10 | **Gate on blast radius, not speed** | P7, §5.1, P8, §5.2, §6 | Merge gate |

### Why this order

- **Chapter 1 teaches proportionality**, the one section both docs mark "read first." Craft measures it by half-life × blast radius; the primer measures it by pain removed. The remaining 22 principles are instances of "how much of this applies here?"
- **Chapter 1 does not claim the two docs share one dial.** They do not: "half-life" appears nowhere in the primer, and "attention" appears nowhere in craft. Chapter 6 introduces the primer's own form and bridges back.
- Act I precedes Act II because the primer repeatedly points back to craft rather than restating it.
- Autonomy sits last because the primer's own §8 places it at phase 8, emergent only once phases 1–7 are mature.

### Chapter shape

**Documented example → diagram → rule → widget → caveat.**

The opening example must be one the source names — Knight Capital, Air Canada, Equifax, SolarWinds, Netscape, Shopify, Stripe, WhatsApp, Stack Overflow, the Friday cleanup, the `p-limit` reimplementation, the `user.email` column. Where a chapter has no documented incident, it opens with the rule instead. **There is no "story" slot**; that word invites invention.

Word budget 400–700 per chapter. Overflow moves to level-3 detail — but only *supporting* material, never the chapter's teaching.

**Two chapters get a raised budget of 700–900**, because their source material is roughly 1,500 words each and compressing 2× would push teaching into level 3. Their internal structure is named now, not discovered mid-build:

- **Chapter 5** — one shape (expand → migrate → contract), then the practices around it: boundaries, additive evolution, strangler, small changes, tests, flags, evals.
- **Chapter 8** — four rules: retrievable, layered, boring, mechanized (P3, P4, P5, P6).

---

## 6. Isolation vs. blending — the rule

The answer is structural, not editorial.

**Origin is always visible.** Every rule, example, and expandable carries a badge: `CRAFT §3.4` or `AGENTS P7`. The reader never guesses which body of knowledge a claim comes from. This is what makes blending safe.

**Blend only at the nine points the primer itself declares.** The unit is a *declared reference*, not a target section — the primer declares craft §4 for two different rules. Each is a visually distinct **Bridge** card naming both sides. It appears once, in the Act II chapter where the primer makes the reference, with a backlink from the Act I chapter.

| # | Craft side | Agents side | Declared at |
|---|---|---|---|
| 1 | §0 simplicity rules | Harness code is code too | primer 47 |
| 2 | §3.2 boundaries follow change rates | P4 movable boundaries | primer 147 |
| 3 | §3.4 Type-1 / Type-2 | P7 gate on blast radius | primer 164 |
| 4 | §3.6 innovation tokens | P5 boring by default | primer 149 |
| 5 | §3.7 parse, don't validate | §5.2 "no YOLO data probing" | primer 441 |
| 6 | §3.11 prompt injection unsolved | §4.4 poisoned rule files; security needs its own threat model | primer 75, 400, 551 |
| 7 | §4 flaky test is a bug | §5.1 quarantine, never re-run to green | primer 69, 428 |
| 8 | §5 chase every review finding | §4.4 cap review rounds | primer 76, 407 |
| 9 | §4 continuous refactor, in its own small change | P8 / §5.2 cleanup, never inside a feature change | primer 59, 171, 173 |

**One juxtaposition, labelled as such** (not source-declared, so it is presented as a comparison rather than a cross-reference): craft's *"If you cannot verify it, do not ship it"* against the primer's *"Verifiable is not correct."* Necessary versus sufficient. Chapter 9.

**One deliberate duplication, badged apart.** Chapter 6 teaches primer §1's operating contract (lines 84–91) badged `AGENTS §1`; chapter 9 teaches P1 badged `AGENTS P1`. This mirrors the primer's own structure — the accuracy pass should not flag it as a repeat.

**One disambiguation.** Craft's agentic examples are about agents **as the product being built**; the primer is about agents **as the builder**. Badges alone will not prevent this conflation — chapter 6 names it in one sentence.

**Everything else stays isolated.** Craft-only ideas (data model, pure core, strangler fig, migration, mental models) live in Act I and are never re-explained in Act II. Agent-only ideas (close the loop, attention as budget, retrievable-not-resident, harness tiers, autonomy, implementation sequence) live in Act II and are never back-ported.

---

## 7. Structure: the Map

All 23 principles in one filterable grid. Collapsed, each card shows its origin badge, the rule line **verbatim from that doc's §0** (identical to the spine's rule line — same words in both places, so repetition reinforces), and one documented example in a phrase.

Expanded: what it means, in practice (2–4 bullets), the caveat or tension, and a link to the spine chapter that teaches it.

**Filters use the docs' own taxonomies, not an invented one.** Origin (All / Craft / Agents), plus — for primer cards only — the doc's own six groupings: **Foundation** (§2 "The foundation", P0) and **Themes A–E** (Intent and ownership, The substrate, Enforcement, Throughput and risk, Verification and trust).

**No theme filter for craft cards.** Its system-class axis does not discriminate: SaaS and Agentic examples appear in all 11 principles, Workflow in 6. Two of three chips would change nothing, which reads as broken. Eleven cards plus search already meets the 15-second goal.

**Search:** one input filtering on rule text and principle name. No fuzzy matching.

---

## 8. The panels

| Panel | Content | Source |
|---|---|---|
| **Glossary** | 8 terms, verbatim | primer §0 Terms |
| **Anti-patterns** | Two badged lists, side by side | craft §5 (9 items), primer §7 (13 items) |
| **Mental models** | Four groups | craft §6 |
| **Where to start** | 8 phases, why the order compounds | primer §8 |
| **Open questions** | 9 unresolved items | primer §9 |
| **Provenance** | From the essay / generalized here / standing uncertainties | primer §10 |
| **Operating rules** | Two badged checklists, side by side | craft §0 (7 rules), primer §0 (9 rules) |
| **Self-test** | 10 questions, ending on craft 587 | craft §8 |

**The close, in order:** the self-test's ten questions → craft line 587, *"If any answer is 'no,' that is the next thing to fix"* → craft §7's one-line distillation (line 570) as the literal last line on the page.

The self-test is the docs' own retrieval-practice instrument, and retrieval practice is what converts reading into memory. The operating rules are for *doing*, not recall, so they get their own panel rather than diluting the close.

---

## 9. The widgets

Seven. Each maps to a decision a doc explicitly describes. **No widget invents a state the source does not name.** Each shows its source badge and collapses to a static readable equivalent.

| Widget | Ch. | Interaction | Output | Source |
|---|---|---|---|---|
| **Two dials** | 1 | Two *independent* axes, no combined verdict. Half-life has the doc's three positions (years / months / weeks, line 96). Blast radius has two: **Type-2** and **Type-1** (180–181) — craft sets blast radius by reversibility (line 100) | Half-life → how much design this deserves (§3.7 resolution, 308–309). Blast radius → how much caution, labelled with line 468's "a prototype needs logs / a payment path needs traces" | craft §3 line 96; §3.4 lines 180–181, 100; §4 line 468; §3.7 lines 308–309 |
| **One-way / two-way sorter** | 2 | Assign these six documented decisions: public API contract (180), identifier scheme in URLs and webhooks (186), a mobile build in users' hands (187), a breaking workflow change with runs in flight (188), internal class names (181), build tooling (181). A seventh — a framework or ORM (182) — carries the third state | Correct bin plus the doc's own reasoning | craft §3.4. The framework/ORM item is the trap: Type-2 to decide, Type-1 to undo |
| **Exhaustiveness ladder** | 3 | Select a toolchain row; Python selected by default | What happens when you add a variant | craft §3.7 table, five rows verbatim; the payoff is line 312's "it is a convention" |
| **Expand → migrate → contract** | 5 | Step through 3 stages; pick one of 3 change types | What happens at each stage; reversible until the last | craft §4 lines 482–486 and the table at 490–492 |
| **Failure diagnostic** | 6 | Pick the symptom | The response for that class, plus line 105's payoff: "Only the middle row is solved by 'fix the environment, not the prompt'" | primer §1 table, 3 rows, and line 105 |
| **Build-tier trigger** | 7 | "What is happening in your repo?" | Which tier to build, and not to build the next one yet | primer §4 tier table lines 357–361. The trust ranking (349–353) renders statically — it is a ranking, not a fork |
| **Merge gate** | 10 | **One** question, as in the source | Optimistic merge, or gate — with the four gated domains as examples of "no" | primer §5.1 lines 423–425 |

**Rendered statically, not as widgets:** the verification trust ranking, the autonomy map table (primer §6), the P9 gap/owner table, the failure-taxonomy table.

**Why two independent dials and not a matrix.** Craft gives half-life three values and ties blast radius to reversibility and the failure taxonomy. It never defines a blast-radius scale or a combined verdict. Nine synthesized cells would be invention. The reader combining the two dials mentally *is* the lesson.

---

## 10. Interaction and disclosure

**First paint:** title, two-sentence framing, the two acts as 10 collapsed chapter rows, and links to Map and Panels. Roughly one screen. Nothing expanded.

**Three levels:**

1. Chapter row — number, imperative title, one-line promise, act colour.
2. Chapter open — the teaching. 400–700 words.
3. `▸ detail` — cross-domain examples, full tables, code, citations, layer names, the eleven capabilities.

**Kept out of any chapter's level 2** (high detail, low retention value): the six test rules, the four `AGENTS.md` failure modes, the eleven capabilities, the specific layer names — which the primer itself calls "not a law" (line 321) — and all citation details. The eight phases are not level-3 material; their home is the "Where to start" panel.

**State:** open chapter, filters, theme, and last-read position persist in `localStorage`, wrapped in try/catch, rendering correctly when storage is unavailable.

**Navigation:** slim persistent rail — 10 chapters, Map, Panels. Deep links via hash (`#ch-4`, `#map`, `#p-craft-3-7`).

---

## 11. Visual system

- **Single page, no framework, no CDN.** Hand-written HTML, CSS, vanilla JS.
- **Two act identities.** Distinct accent hues for Craft and Agents, on badges, rails, and bridge cards. Everything else neutral, so accent carries meaning rather than decoration.
- **Diagrams are inline SVG,** hand-built, theme-aware. The primer's ten diagrams are redrawn inline rather than embedded as files, so they scale, theme, and carry accessible text. **Every label comes from the text equivalent already in the primer; nothing is added.** Act I diagrams — craft has none — must each render a documented list, table, or sequence with its own label set: the two dials, the Type-1/Type-2 bins, failure versus adversarial mitigations (craft 429–430), expand → migrate → contract, pure core / imperative shell.
- **Theme:** light and dark via `prefers-color-scheme` plus a toggle; full token set on `:root`, redefined under both the media query and `[data-theme]`.
- **Type:** system stack; monospace for code and diagram labels.
- **Motion:** minimal; disabled under `prefers-reduced-motion`.
- **Responsive:** single column under 768px; rail becomes a top bar; the sorter switches from drag to tap-to-assign. Tables and code scroll in their own containers; the body never scrolls horizontally.
- **Accessibility:** `<details>`/`<summary>` where it fits, real buttons for widgets, visible focus rings, `aria-live` on widget verdicts, contrast ≥ 4.5:1 in both themes.

---

## 12. Deliverable and validation

**Artifact:** one file, `learning/index.html`, self-contained.

**Validation before this is called done:**

1. **Accuracy pass** — every assertion mapped to a doc line, categorised quoted / compressed / connective. Unmappable content cut. Protected hedges (§3) confirmed present and unsharpened.
   - **Write the mapping before the prose.** For each chapter paragraph, record the line range it draws from, then derive the prose from that. Mapping afterward finds problems only after the text has been written to flow — the highest-risk failure mode in this build.
2. **Widget-state audit** — for each widget, every state listed against the source row it comes from. No state without a row.
3. **Coverage pass** — every row of §4's matrix confirmed to have a home on the page.
4. **Live validation** — served locally, walked end to end as a first-time reader: every chapter, widget, filter, both themes, mobile width. Console clean.
5. **Critical review from the audience's lens** — is the first screen calm? Does chapter 1 make the rest feel like instances rather than a list? Can a returning reader find a principle in seconds?
6. **Independent review** of the built result.

---

## 13. Risks

| Risk | Mitigation |
|---|---|
| Wall of text with expanders bolted on | 400–700 word budget; overflow moves to level 3, but teaching never does |
| Widgets feel like toys | Each maps to a documented decision, shows its source, and collapses to a static equivalent |
| Blending blurs origin | Badge on every rule; blending confined to eight declared bridges plus one labelled juxtaposition |
| Invention while compressing | Accuracy pass with line mapping; protected-hedge list; documented-example rule; no "story" slot |
| Act I diagrams invent structure | Each must render a documented list, table, or sequence |
| Agent-as-product vs. agent-as-builder conflation | Named explicitly in chapter 6 |
| Single file grows unmaintainable | Delimited sections with a table-of-contents comment at the top |
