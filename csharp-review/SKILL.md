---
name: csharp-review
description: >-
  Comprehensive C# / .NET code review. Produces a numbered, cherry-pickable list of findings
  (tagged with severity, lens, and confidence) covering correctness, async/concurrency, EF
  Core/data, security, performance, and design — plus an honest architecture / maintainability /
  scalability critique.
disable-model-invocation: true
allowed-tools:
  - Read
  - Grep
  - Glob
  - Bash
---

# C# / .NET Code Review

Do a senior-level review of C# code and return findings the user can act on selectively. The
output is a flat, numbered list so the user can reply "fix #2, #5, #11" and you know exactly what
they mean. Alongside concrete defects, give an honest read on design: cognitive load, domain
modelling, architecture, maintainability, and scalability — including saying plainly when the
approach doesn't fit the problem.

## Two review modes

Pick the mode from what the user asks for. If it's ambiguous, ask once.

**`branch` — review the changes on the current branch.** This is the default for "review my
changes / my branch / this PR".
1. Find the base: try `git merge-base HEAD origin/main`, falling back to `origin/master`,
   `origin/develop`, or whatever the repo's default branch is. If unsure, ask which base to diff
   against.
2. Get the changeset: `git diff <base>...HEAD` for the diff, and `git diff --name-only <base>...HEAD`
   for the file list. Use `git log <base>..HEAD --oneline` to understand intent.
3. Review the changed lines, but **read the surrounding files** for context — a change is only
   correct relative to the code around it. Don't review a hunk in isolation.

**`codebase` — review an existing project or directory.** For "audit project X / review this
codebase". This needs a strategy so it doesn't drown in context or in trivia — read
`references/codebase-audit.md` and follow it. In short: map the solution, prioritize the
high-risk / high-value areas, review in passes, and report at both the module and the file level.

## Review process

1. **Understand intent first.** What is this code supposed to do? For `branch` mode, read the
   commit messages / PR description. For `codebase` mode, read the README and project structure.
   You cannot judge whether an approach fits without knowing the goal.
2. **Map the stack.** Check `.csproj` for the target framework (`net8.0`, `net9.0`…) and key
   packages (EF Core, ASP.NET Core, MediatR, etc.). Tailor findings to the actual framework
   version — don't suggest C# 12 syntax on a `net6.0` project, and do flag patterns that newer
   versions made obsolete.
3. **Read the relevant reference modules** (below) for the dimensions in scope. Load only what's
   relevant — security review of a pure-domain library doesn't need the EF Core sections.
4. **Pass 1 — correctness & safety.** Bugs, async/concurrency hazards, EF Core/data issues,
   security holes, resource leaks. These are usually the blocking findings.
5. **Pass 2 — design & modelling.** Step back from the lines. Is this the right shape? Are the
   modules deep or shallow? Are illegal states representable? Where will this hurt to change or
   scale? This is the pass most reviewers skip and the one the user explicitly wants.
6. **Verify before asserting.** If a build or tests are available and cheap to run, prefer
   confirming a suspected bug over guessing. Lower the confidence tag when you can't verify.
7. **Assemble the report** in the exact format below.

## What to review — and what to leave to the toolchain

Spend the review budget on judgment, not on things tooling already enforces. **Do not** raise
findings for: formatting / whitespace / brace style, `using` ordering, naming-convention
casing, or anything a Roslyn analyzer, `.editorconfig`, nullable warnings-as-errors, or
`dotnet format` would catch on its own. If you notice the project lacks that enforcement, raise
*one* finding recommending it, rather than dozens of style nits.

**Do** review: logic and edge cases, async correctness, concurrency/thread-safety, EF
Core/data access, security, resource lifetime, public API design, error handling, test coverage
and quality, and the whole design/modelling dimension. That's where a human (or a careful model)
adds value over a linter.

## Reference modules

Read these as needed — they hold the detailed C#-specific patterns with ❌/✅ examples:

| Module | Read it when reviewing… |
|--------|--------------------------|
| `references/csharp-correctness.md` | async/concurrency, nullability, exceptions, EF Core & data, LINQ, resource lifetime, correctness pitfalls (date/decimal/string), allocation/perf |
| `references/csharp-security.md` | anything handling untrusted input, auth, data access, serialization, secrets, or logging of user data |
| `references/design-and-modelling.md` | the design pass — cognitive load (deep modules), domain modelling (illegal states unrepresentable), architecture, maintainability, scalability. **The differentiator — read it on every non-trivial review.** |
| `references/codebase-audit.md` | `codebase` mode — how to map, prioritize, and chunk a whole-project review |

## Finding taxonomy

Tag every finding on three axes.

**Severity** — how much it should block:
- 🔴 `blocking` — a real defect or risk that should be fixed before this ships (bug, security hole, deadlock, data loss/corruption).
- 🟡 `important` — should be fixed; matters for correctness-under-load, maintainability, or design integrity, but isn't an outright defect.
- 🟢 `minor` — worth doing, not worth blocking on (local simplification, small clarity win).

**Lens** — what kind of issue it is: `Correctness` · `Async` · `Concurrency` · `Security` ·
`Data/EF` · `Performance` · `Architecture` · `Maintainability` · `Domain Modelling` ·
`Simplification` · `Tests`. Optimizations and simplifications are findings too — tag them
`Simplification` or `Domain Modelling` so they share the numbering and can be cherry-picked like
anything else.

**Confidence** — how sure you are: `High` (verified or unambiguous) · `Medium` (very likely,
based on visible code) · `Low` (depends on context you can't see — call out the assumption). Be
honest with this; a `Low`-confidence flag the user can quickly confirm is more useful than false
certainty.

## Report format

ALWAYS use this structure. It is built for scannability: a strong `---` rule between findings,
a thin `──────────` rule inside each finding, and real paragraphs — never single walls of text —
in the prose blocks.

### Verdict

Lead with the `## Verdict` heading, then **two short paragraphs** (3–5 sentences total), separated
by a blank line:

- **Paragraph 1 — the read.** Does the chosen approach actually fit the feature? Give the honest
  overall verdict: if the design is sound, say so; if it's the wrong shape, say that plainly.
- **Paragraph 2 — the risks.** The headline risks, each pointing at the finding number that
  carries it (e.g. "the deadlock in #5 is the one blocker").

### Findings

Use the `## Findings` heading, then a single, flat, globally numbered list ordered by severity
(🔴 → 🟡 → 🟢) and numbered continuously so every finding has a unique number. **Separate each
finding from the next with a `---` rule.** Inside each finding, split the three bands — header,
body, footer — with a thin `──────────` line and surround every rule with blank lines:

```
---

**#<n> · <severity emoji> <Lens> · `<relative/path/File.cs:line>`**

──────────

<Body. State the problem in a sentence or two — what's wrong and why it bites. Then the
recommendation as its own paragraph, prefixed `→`. Show the fix in a fenced code block when it's
short and clarifying. Keep prose in paragraphs; never let it run on into one dense line.>

──────────

Effort: S|M|L · Risk if unfixed: High|Medium|Low · Confidence: High|Medium|Low
```

### Design & Architecture Assessment

Use the `## Design & Architecture Assessment` heading, then **narrative prose broken into a few
short paragraphs** — one idea per paragraph, never a list, and never one giant block. Cover, in
roughly this order: approach suitability; architecture gaps and where change amplifies; scalability
limits; and domain-modelling smells. Reference finding numbers (e.g. "the coupling in #7 and #12
is the same root cause") instead of restating them.

This section is allowed — expected — to challenge the approach, not just the lines. If the design
is genuinely good, keep it to one paragraph saying what works and why.

Example of one finding, for calibration:

```
---

**#5 · 🔴 Async · `Services/OrderService.cs:142`**

──────────

Blocks on `.Result` inside a request path — risks deadlock under load and starves the thread pool.

→ Make the call chain async end-to-end and `await` it; thread `CancellationToken` through.

──────────

Effort: S · Risk if unfixed: High · Confidence: High
```

## Tone and calibration

- Be direct and specific. Findings are about the code, never the author.
- Don't pad the list. Ten real findings beat forty with thirty nits — noise trains people to
  ignore reviews. If something is genuinely fine, don't manufacture a problem.
- It is part of the job to say "this approach won't hold up" when it's true, and to back it with
  the specific reason (which invariant leaks, which change will be expensive, where it bottlenecks).
  Vague disapproval is useless; a named failure mode is actionable.
- When you praise, be specific too — it tells the author what to keep doing. Fold praise into the
  Verdict or Assessment rather than as numbered findings.
- Prefer the simpler shape. If a finding's recommendation is "add more structure", make sure the
  structure earns its complexity (see the deep-modules material in `design-and-modelling.md`).
