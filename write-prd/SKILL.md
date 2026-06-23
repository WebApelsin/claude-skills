---
name: write-prd
description: >-
  Transform the current conversation into a lean, implementation-ready PRD written to
  .claude/prd/, organized as independently implementable and testable vertical slices.
disable-model-invocation: true
---

# Write PRD

Turn the current discussion into a single lean PRD file that decomposes the feature into
vertical slices. Each slice is sized so it can be picked up by an independent `plan-implement`
run.

## Core principles

- **Synthesize, don't re-interview.** The shared understanding already exists in the
  conversation. Mine it. Crystallize it into a spec — don't re-litigate decisions already made.
- **Ask only blocking questions.** A question is *blocking* only if you cannot draw the module
  boundaries or write a slice's acceptance criteria without the answer. Batch those and ask
  once. Everything else goes under **Assumptions & Open Questions**, and the PRD proceeds.
- **Vertical slices are the unit of delivery.** Every module must be independently
  implementable, independently testable, and readable in isolation (plus the PRD's shared
  Problem/Solution context).
- **Concrete over vague.** Acceptance criteria must be measurable or observable. Ban "fast",
  "easy", "intuitive", "robust", "seamless" unless paired with a number or a behavior.
  "Returns within 200ms for 10k rows" — not "is fast".
- **No file paths or code snippets.** They go stale the moment implementation starts. Describe
  behavior, interfaces, and data contracts at a level that survives refactors.

## Process

1. **Gather context.** Read the conversation. Name the feature. If it touches the codebase,
   briefly explore the repo to ground modules in what exists (patterns, seams) rather than
   inventing a greenfield structure.
2. **Decompose into vertical slices.** Sketch the modules (see *Choosing good slices*). Aim for
   **3–7**. Give each a stable ID (`M1`, `M2`, …), map dependencies, and flag what can run in
   parallel.
3. **Resolve blocking gaps.** If a gap blocks a boundary or a criterion, ask now — one batched,
   minimal set of questions. Otherwise record an assumption and move on.
4. **Write the PRD.** Use the template below. Write to
   `<project-root>/.claude/prd/<feature-slug>.md` (create `.claude/prd/` if missing). Honor an
   explicit location if the user gave one.
5. **Hand off.** Give the file path, the suggested implementation order, what runs in parallel,
   and the critical path.

## Choosing good slices

A good slice is a **deep module behind a narrow interface**: it hides real complexity but
exposes a small, stable surface. Aim for slices that are:

- **Independently testable** — "done" is verifiable through external behavior alone.
- **Low-coupling** — talks to siblings through a clear contract, not shared internals.
- **Deliverable on its own** — mergeable (ideally shippable or flaggable) without its siblings.
- **One plan-implement pass** — if a slice needs its own sub-PRD, split it.

Cut **vertically** (a thin path through all layers delivering one behavior), not horizontally
(all the DB, then all the API, then all the UI).

- **Good:** "Saved-filter persistence" — owns how a filter is stored and retrieved; everything
  else uses a get/set contract. Testable alone.
- **Bad:** "Backend work" — a horizontal layer. Delivers no standalone behavior.

## File location & naming

- **Default:** `<project-root>/.claude/prd/<feature-slug>.md`, where `feature-slug` is the
  kebab-case feature name (e.g. `saved-search-filters`). Create `.claude/prd/` if missing.
- If the user names a different path, use it.
- If that slug already exists, ask whether to overwrite or version (`<slug>-v2.md`) — don't
  silently clobber.

## PRD template

Use this structure. Sections marked *(optional)* may be dropped for small features. Never drop
the module breakdown, acceptance criteria, or test notes.

```markdown
# PRD: <Feature Name>

- **Status:** Draft
- **Created:** <YYYY-MM-DD>
- **Source:** <the conversation/topic this PRD came from>
- **Owner:** <name or TBD>

## Problem
<2–4 sentences: who is hurting, why, and why now.>

## Solution
<2–4 sentences from the user's perspective: what we're building and the outcome.
No implementation detail.>

## Goals & Success Signals  (optional)
- <measurable signal the feature is working>

## Non-Goals
- <explicitly out of scope>

## Assumptions & Open Questions
- ASSUMPTION: <inferred fact not explicitly confirmed in the conversation>
- OPEN (non-blocking): <question safe to resolve during implementation>

## Module Breakdown (Vertical Slices)

Each module is independently implementable and testable. Implement in dependency order;
modules sharing no dependency can run in parallel.

### M1 — <Module name>
- **Purpose:** <one line>
- **Depends on:** <none | M2, M3>
- **Parallelizable with:** <none | M4>
- **Behavior / User stories:**
  - As a <user>, I want <action> so that <benefit>.
- **Acceptance criteria:**
  - [ ] <concrete, measurable, externally observable>
- **Interface / contract:** <inputs, outputs, events, and the data shape this slice owns —
  as behavior, not file paths or code>
- **Tests:** <which external behaviors to test; any prior art to mirror>
- **Out of scope for this slice:** <what a sibling handles instead>

### M2 — <Module name>
<same shape as M1>

## Implementation Order & Parallelization
- **Sequence:** M1 → M2 → (M3 ∥ M4) → M5
- **Parallel groups:** {M3, M4}
- **Critical path:** M1 → M2 → M5

## Risks & Notes  (optional)
- <only if genuinely material; otherwise omit>
```

## Quality bar

**Do**
- Make every acceptance criterion checkable by someone who can't see the code.
- Test **external behavior**, not implementation details.
- Point to prior art (similar existing tests/patterns) when the codebase has it.
- Keep each module section self-contained — `plan-implement` reads it in isolation.

**Don't**
- No file paths, function names, or code snippets — they rot fast.
- Don't invent constraints (tech stack, deadlines, metrics) the conversation didn't establish
  — label them `TBD` or put them under Assumptions.
- Don't pad with an enterprise schema. This is a lean engineering PRD.
- Don't re-run discovery the conversation already covered.

## Handoff to plan-implement

Each module section stands alone. To implement, the user runs `plan-implement` against one
module ID at a time — a single slice, plus the PRD's shared Problem/Solution context, is a
complete brief for one pass. Respect the graph: a slice's `Depends on` modules must be
implemented (or stubbed to their stated contract) first; same-group slices can run concurrently.
