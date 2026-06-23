# Whole-Codebase Audit Strategy

For `codebase` mode — reviewing an existing project or directory rather than a branch diff. The
challenge is breadth: you can't read everything with equal attention, and a flat file-by-file
crawl produces a sprawling, low-signal report. Work in passes, prioritize, and report at two
levels.

## Step 1 — Map before reading

Build a mental model of the system before judging any file.

1. **Find the structure.** Locate `*.sln` and `*.csproj` (`Glob **/*.csproj`). The project graph
   tells you the layering: which projects reference which. A domain project that references a web
   or EF project is already a finding.
2. **Read the entry points.** `Program.cs` / `Startup.cs` — DI registrations, middleware
   pipeline, configuration. This reveals the architecture, the service lifetimes, and the
   framework version in one place.
3. **Identify the layers/modules.** Group by responsibility: API/controllers, application/handlers,
   domain, infrastructure/persistence, cross-cutting. Note where the boundaries are — and where
   they leak.
4. **Read the README / docs** for intent. You're judging fit-to-purpose; you need the purpose.

State this map briefly at the top of the review so the user can confirm you understood the system.

## Step 2 — Prioritize (don't review uniformly)

Bugs and design debt cluster. Spend attention where risk and value concentrate:

- **Core domain & money/auth paths** — where a defect costs the most. Review these deeply.
- **Largest files** (`wc -l`, or sort by size) — large files correlate with low cohesion and
  hidden complexity; they're where god classes hide.
- **Security-sensitive surfaces** — anything handling untrusted input, auth, data access,
  serialization, file/path, external calls. Apply `csharp-security.md` here.
- **High-churn / recently-changed files** if git history is available
  (`git log --format= --name-only | sort | uniq -c | sort -rn | head`) — churn marks both active
  risk and pain points.
- **Seams to the outside** — DB access, external APIs, queues, the clock — for testability and
  scalability.

Skip or skim: generated code, migrations, designer files, vendored/third-party code, and
straightforward DTOs/POCOs with no logic.

## Step 3 — Review in passes, not file-by-file

For the prioritized set, run the same two passes as a branch review, but scoped to the module:

- **Pass 1 — correctness & safety** (`csharp-correctness.md`, `csharp-security.md`): the concrete
  defects within each high-priority area.
- **Pass 2 — design & modelling** (`design-and-modelling.md`): step up a level. Are the module
  boundaries in the right place? Is the domain model anemic? Where's the change amplification
  across the codebase? This is where whole-codebase review earns its keep — patterns only visible
  across many files (a status duplicated everywhere, a layering rule violated repeatedly, an
  abstraction that leaks system-wide).

Manage context deliberately: read a module's key files together so you can judge their
interaction, summarize what you found, then move on — don't try to hold the entire codebase in
context at once. If the codebase is large, tell the user you're focusing on the prioritized areas
and name what you didn't cover, rather than implying exhaustive coverage.

## Step 4 — Report at two levels

Use the standard report format from `SKILL.md`, with one addition for codebase mode:

- **Per-file/per-module findings** go in the numbered list as usual, with paths and lines, so the
  user can cherry-pick.
- **Systemic findings** — patterns that recur across the codebase — also go in the numbered list,
  but phrase them as the pattern with representative locations (e.g. "#8 · 🟡 Architecture · domain
  entities reference `DbContext` in 6+ places, e.g. `Order.cs:40`, `Customer.cs:55` — the domain
  layer depends on infrastructure"). One numbered finding for the pattern beats fifty identical
  per-site nits.
- The **Design & Architecture Assessment** carries the system-level narrative: the overall
  architecture's fitness, the two or three structural issues that matter most, and what's working
  and should be preserved. This is the part the user most wants from a whole-codebase audit — make
  it the strongest section.

## Scoping note

If the project is large and the user hasn't narrowed it, offer to scope: a specific
project/folder, a layer, or "the highest-risk areas across the whole thing". A focused, deep
review of the parts that matter is more useful than a shallow sweep of everything.
