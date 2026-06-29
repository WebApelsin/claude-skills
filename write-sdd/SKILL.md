---
name: write-sdd
description: >-
  Transform the current conversation — usually a /grill-me discovery session, optionally an
  existing PRD — into a lean, readable, modularized Software Design Document written to
  .claude/sdd/. The SDD is the *design* layer (the how): it elaborates each vertical slice
  into a binding contract, data model, interactions, failure behavior, and key decisions,
  grounded in the actual codebase, so each module can be handed to a separate plan-implement
  run one-by-one or in parallel waves. It carries a live task/status list and supports an
  UPDATE mode that marks modules done and reconciles design drift after each module ships.
  Use this whenever the user says "write an SDD", "write a design doc", "turn this into a
  design", "build the modularized spec", "design doc for this feature" — or "update the SDD",
  "mark M3 done", "M2 is implemented" — even if they never say "SDD". Prefer this over
  write-prd when the feature has real architectural surface (contracts that cross modules,
  non-trivial data flow, decisions worth recording). Synthesize from the conversation; do not
  re-run discovery.
---

# Write SDD

Turn a discussion (usually a `/grill-me` session, optionally an existing PRD) into a single
**Software Design Document**: a lean, readable, modularized design that decomposes the feature
into vertical slices and elaborates each into a contract an implementer can build against. It
sits between requirements and `plan-implement`, and it is a **living document** — it tracks
which modules are done and absorbs design drift as they ship.

This is the richer alternative to `write-prd`. Reach for it when the feature has real design
surface: contracts that cross module boundaries, non-trivial data flow, or decisions worth
recording so nobody silently re-decides them. For a thin feature with no cross-module design,
`write-prd` is enough.

## Two modes

Detect which mode applies before doing anything else.

- **GENERATE** — the default. The user wants a new SDD from the conversation (and/or an
  existing PRD). Most of this skill is about generate mode.
- **UPDATE** — the user reports that a module is implemented ("M3 is done", "mark M2
  in-progress", "I finished the resolver, update the SDD") and an SDD already exists. Jump to
  *Update mode* near the end. Do **not** regenerate the document.

## Core principles

- **Synthesize, don't re-interview.** The shared understanding already exists in the
  conversation (and the PRD, if there is one). Mine it. A `/grill-me` session has already
  pushed on the hard questions — crystallize, don't re-litigate.
- **Bind only what's expensive to reverse or crosses a boundary.** The SDD is not a place to
  pre-specify everything. Apply the **reversibility-and-boundary test**: *if two competent
  implementers could choose differently and it wouldn't affect any other module or any
  acceptance criterion, it does not belong in the SDD.* That decision is for `plan-implement`.
- **Readable by default.** Most design docs fail not on missing detail but on being an
  undifferentiated wall of identical bullet blocks. Lead with the gist everywhere; push
  precision down; let diagrams carry interaction load. (See *Readability*.)
- **Living document.** The SDD tracks module status and reconciles drift. It is never a stone
  tablet that goes stale the moment code starts.
- **Module IDs are planning artifacts, not code.** `M1…Mn` are for routing implementation
  runs. They must never leak into code, comments, tests, or commit messages.
- **Ask only blocking questions.** A question is blocking only if you cannot draw a module
  boundary or write a binding contract without the answer. Batch those, ask once. Everything
  else goes under *Assumptions & Open Questions* and the SDD proceeds.

## Process (GENERATE mode)

1. **Gather context.** Read the conversation. If a PRD exists (`.claude/prd/<slug>.md` or one
   the user points to), read it and **reuse its module IDs and names** so the two stay in
   lockstep. Identify the feature and a short slug.

2. **Ground in the codebase.** If the discussion references the repo, briefly explore it:
   existing patterns, where the seams are, what to extend vs. create. The SDD's job is to be
   grounded in what actually exists — but describe behavior and contracts, not file paths (see
   the binding/leaves-alone split). Grounding informs the design; it does not put line numbers
   in the doc.

3. **Decompose into vertical slices.** If you inherited modules from a PRD, keep them. If not,
   sketch **3–7** deep modules behind narrow interfaces (a good slice is independently
   testable, low-coupling, deliverable on its own, and buildable in one `plan-implement` pass).
   Cut vertically (a thin path through all layers delivering one behavior), not horizontally
   (all DB, then all API, then all UI). Assign stable IDs `M1, M2, …`.

4. **Map dependencies and compute waves.** For each module record `Depends on` and `Blocks`.
   Then compute parallel **waves**: Wave 1 = modules with no dependencies; Wave N = modules
   whose dependencies are all satisfied by waves < N. Modules in the same wave can run
   concurrently. Reject cycles — if you find one, the boundaries are wrong; re-cut.

5. **Resolve blocking gaps.** Only if a gap blocks a boundary or a binding contract, ask one
   batched, minimal set of questions. Otherwise record assumptions and proceed.

6. **Write the SDD** using the template below. Write to
   `<project-root>/.claude/sdd/<feature-slug>.md` (create `.claude/sdd/` if absent). Honor an
   explicit path if the user gave one.

7. **Self-check** against the checklist near the end before presenting. Fix anything it flags.

8. **Hand off.** Give the file path, the wave schedule (which modules run in parallel, what the
   critical path is), and the suggested first wave to implement.

## What the SDD binds vs. leaves alone

This is the most important judgment call. Get it right and the doc stays lean and survives
refactors; get it wrong and it either rots or strangles the implementer.

**The SDD BINDS** (these are contracts; `plan-implement` must honor them):
- Cross-module interface contracts — inputs, outputs, events, error shapes, idempotency.
- The shared data model — anything more than one module reads or writes.
- Architecturally significant decisions **and their rejected alternatives** (ADR content), so
  nobody silently re-decides them.
- Externally observable failure behavior — what a caller sees when things go wrong.

**The SDD LEAVES ALONE** (inferred at `plan-implement` time):
- Internal algorithms and data structures local to one module.
- File and function layout, local naming.
- Library micro-choices with no cross-module effect.

**Advisory (a third category, optional, must be visually distinct).** Hard-won knowledge you
don't want re-derived — prior art to mirror, a known gotcha, "we tried X, it deadlocks." These
are **non-binding hints**, not specifications. Mark them clearly as advisory or `plan-implement`
will treat a hint as a spec.

## Readability

Voice is the smaller lever; **structure is the bigger one.** In priority order:

- **BLUF — bottom line up front, everywhere.** The doc opens with three plain sentences a tired
  engineer reads in ten seconds. Every module opens with a one-line **the gist** before any
  contract. This is the highest-impact rule in the skill.
- **Worked example before abstraction.** Show one real input→output or scenario before the
  formal contract. People build the mental model from the concrete instance.
- **Diagrams carry the interaction load.** Replace "then it calls… which returns… which the
  router then…" with a mermaid sequence diagram. Use mermaid for context, sequence, and state.
- **Progressive disclosure within each module.** Two layers: a prose *gist* (human voice,
  analogies welcome) then a tight *contract* block (precise, EARS, no fluff). The reader stops
  at the depth they need.
- **Analogies live in the gist/rationale only — never in a contract or acceptance criterion.**
  An analogy explaining *why* the cache exists is great; one standing in for *what is returned
  on error* will mislead someone at 2am. Precision and warmth live in different zones.

## Keeping module IDs out of the code

Module IDs leak into code comments because `Mn` ends up the most prominent label on a section
and the implementing agent grabs it as a traceability breadcrumb. Defend at three levels:

1. **Lead with the human name, demote the ID.** Header reads **Variation Resolver** with `M3`
   as a small tag — never `## M3`. The salient token becomes the domain concept, which is what
   belongs in code anyway.
2. **Explicit guard in the handoff section** (it's in the template): module IDs are planning
   artifacts; never reference them in code, comments, test names, or commit messages.
3. **Suggest a systemic backstop** once: a single line in the project's `CLAUDE.md` stating that
   spec/module identifiers are not code artifacts. Catches it across every feature with no
   per-doc effort. Offer this; don't edit `CLAUDE.md` without asking.

## File location & naming

- **Default:** `<project-root>/.claude/sdd/<feature-slug>.md`
- `feature-slug` is the kebab-case feature name; match the PRD slug if one exists.
- Create `.claude/sdd/` if missing.
- If an SDD with that slug exists and you're in GENERATE mode, ask whether to overwrite or
  version (`<slug>-v2.md`) — don't silently clobber. (If the user is reporting progress, that's
  UPDATE mode, not a clobber.)

## SDD template

Use this structure. Sections marked *(optional)* may be dropped for smaller features — keep it
lean. Never drop the Status table, the module contracts, acceptance criteria, or the handoff.

```markdown
# SDD: <Feature Name>

- **Status:** Active
- **Created:** <YYYY-MM-DD>  ·  **Updated:** <YYYY-MM-DD>
- **Source:** <PRD slug, and/or /grill-me session on <topic>>

## Status

Legend: ⬜ Not started · ◐ In progress · ✅ Done · ⚠ Needs revision

| Module | Name | Wave | Status |
|--------|------|------|--------|
| M1 | <name> | 1 | ⬜ |
| M2 | <name> | 1 | ⬜ |
| M3 | <name> | 2 | ⬜ |

**Wave schedule:** Wave 1 = {M1, M2} (parallel) → Wave 2 = {M3}.
**Critical path:** M1 → M3.

## The Big Picture  (BLUF)

<Three plain sentences. What is this feature, who is it for, and the shape of the design —
readable in ten seconds. No jargon, no module IDs.>

## Architecture  (optional for small features)

<One context paragraph: how this feature sits in the existing system. Then a mermaid diagram
if interactions are non-trivial.>

```mermaid
%% context or component diagram
```

## Cross-Cutting Decisions

<Decisions every module inherits: error-handling philosophy, observability/telemetry hooks,
security/PII handling, feature-flag & rollout strategy. Each is one line. Omit any that don't
apply.>

## Shared Data Model & Contracts  (optional)

<Only the data and contracts that more than one module touches. Describe shape and meaning,
not storage mechanics.>

## Key Decisions (ADR-lite)

- **Decision:** <what was chosen> — **Because:** <one line> — **Rejected:** <alternative + why
  not>. <Repeat for each architecturally significant choice. This is what stops silent
  re-deciding during implementation.>

## Assumptions & Open Questions

- ASSUMPTION: <inferred, not explicitly confirmed>
- OPEN (non-blocking): <safely resolved during implementation>

## Modules

Each module is independently implementable and testable, and is written to stand alone — a
single module plus *The Big Picture* and *Cross-Cutting Decisions* is a complete brief for one
plan-implement run.

### <Module Name> · `M1` · Wave 1 · ⬜

**The gist:** <One or two plain sentences. Analogy welcome here. What does this module do, in
human terms, and what does the rest of the system get from it?>

**Worked example:**  *(optional but recommended)*
<One concrete scenario or input→output before the formal contract.>

**Contract**  *(binding)*
- In: `<shape>` → Out: `<shape>` / `<error shape>`
- WHEN <condition/event> THE SYSTEM SHALL <observable behavior>
- WHEN <failure condition> THE SYSTEM SHALL <observable failure behavior>

**Data model delta:** <what this module owns; omit if none>

**Interactions:**  *(mermaid sequence if it earns its place; omit for trivial modules)*

**Acceptance**
- [ ] <concrete, externally observable, traceable to a requirement>
- [ ] <edge case>
- [ ] <failure case>

**Advisory (non-binding):** <prior art to mirror, gotcha, dead end to avoid. Omit if none.>

**Depends on:** <none | M2> · **Blocks:** <none | M3>

### <Module Name> · `M2` · Wave 1 · ⬜
<same shape>

## Implementation Order
- **Waves:** Wave 1 {M1, M2} → Wave 2 {M3}
- **Critical path:** M1 → M3
- Implement a wave's modules concurrently; a module may start once everything it *Depends on*
  is done or stubbed to its stated contract.

## Handoff to plan-implement

Run `plan-implement` against one module at a time, in wave order. Each module section above
plus *The Big Picture* and *Cross-Cutting Decisions* is a complete, self-contained brief.

> **Module IDs (M1…Mn) are planning artifacts only.** Never reference them in code, comments,
> test names, branch names, or commit messages. Name things by their domain concept (the
> module's human name), not its planning ID.

## Change Log
- <YYYY-MM-DD> — Created.
```

## EARS quick reference

Write acceptance criteria and contract behavior in EARS (Easy Approach to Requirements Syntax)
so they're unambiguous and testable:

- **Ubiquitous:** THE SYSTEM SHALL <behavior>.
- **Event:** WHEN <trigger> THE SYSTEM SHALL <behavior>.
- **State:** WHILE <state> THE SYSTEM SHALL <behavior>.
- **Conditional:** IF <condition> THEN THE SYSTEM SHALL <behavior>.
- **Feature-scoped:** WHERE <feature is present> THE SYSTEM SHALL <behavior>.

Cover happy path, edge cases, and failure modes. For AI/LLM modules (prompts, bots, guardrails),
"acceptance" can be a passing **eval suite** (e.g. a promptfoo config) rather than unit tests —
state which, and what the suite must assert.

## UPDATE mode

Triggered when the user reports progress and an SDD already exists. **Do not regenerate.** Read
the existing SDD, then:

1. **Flip the status** of the named module(s) in the Status table (⬜ → ◐ → ✅, or ⚠ if the
   implementation revealed the design was wrong).
2. **Check off** the acceptance criteria that are now satisfied. Leave unmet ones unchecked.
3. **Reconcile drift.** If implementation diverged from the design:
   - If the divergence is **internal** to the module (algorithm, naming, local choice): no
     action — the SDD never specified it. Don't record it.
   - If it touched a **binding contract** (a cross-module interface, the shared data model, a
     recorded decision): **flag it loudly.** Add a dated entry to the Change Log, update the
     contract, and explicitly warn the user which sibling modules depend on it and may need
     revisiting. Never silently rewrite a contract — a quiet contract change is exactly what
     breaks a parallel sibling.
4. **Bump the Updated date** and append to the Change Log: `<date> — Mn implemented; <one line>`.
5. **Report** what changed, especially any contract change and its blast radius, and which wave
   is now unblocked.

The durable contract is the one thing update mode treats as sacred: editable only deliberately,
visibly, and with the downstream impact called out.

## Quality bar

**Do**
- Open the doc and every module with the gist (BLUF). Make a tired engineer's ten seconds count.
- Make every acceptance criterion externally observable and traceable to a requirement.
- Record rejected alternatives for significant decisions.
- Keep each module self-contained — it's read in isolation by `plan-implement`.
- Use diagrams for interactions; reserve prose for intent.

**Don't**
- Don't bind anything that passes the reversibility-and-boundary test (it belongs to
  `plan-implement`).
- Don't put file paths, function names, or code snippets in the doc — they rot.
- Don't let analogies into contracts or acceptance criteria.
- Don't invent constraints (stack, deadlines, metrics) the conversation didn't establish —
  label them `TBD` or Assumptions.
- Don't use `## M3` style headers — lead with the human name, ID as a tag.
- Don't pad into an enterprise schema. Lean beats complete.

## Self-check before presenting (GENERATE mode)

- [ ] Every requirement/PRD item maps to at least one module (no orphaned requirements).
- [ ] Every module has a gist, a binding contract, and acceptance criteria.
- [ ] No dependency cycles; every wave is correctly computed.
- [ ] No module binds something that should be left to `plan-implement`.
- [ ] No file paths, code snippets, or `## Mn` headers; advisory clearly marked as non-binding.
- [ ] Status table present and consistent with the modules below it.
- [ ] No vague words ("fast", "robust", "seamless") without a number or observable behavior.
