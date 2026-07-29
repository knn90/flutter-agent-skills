---
name: flutter-scout
description: "Codebase scouting for a Flutter project via parallel Explore subagents. Use to locate where a feature/symbol lives before edits, or to gather codebase context for a task."
argument-hint: "[search-target] [quick|full]"
---

# Scout — Flutter

Find things fast using parallel `Explore` subagents — they do the reading; never pull
full files into main context. Output is a **map**, not analysis.

## Step 0 — Load profile

Read `.claude/flutter-profile.md` for `source_roots`, `test_roots`, `generated_paths`,
`state_type`, `navigation`, `networking`. If absent, read the main checkout's copy —
`$(git rev-parse --path-format=absolute --git-common-dir)/../.claude/flutter-profile.md`
(the profile is usually gitignored, so worktrees don't inherit it). Still absent → tell the
user to run `flutter-project-init` first (greenfield repos have nothing to scout).

## When to Use

- A feature spans multiple packages / `source_roots`
- Before cross-cutting edits (shared state, DI registrations, repositories, routes)
- DRY check — confirm a pattern exists before adding a new one

## When NOT to Use

| Case | Use instead |
|---|---|
| Single file, known path | `Read` |
| One symbol grep | `Grep` |
| Trade-off discussion | `flutter-brainstorm` |
| Full implementation plan | `wayfinder` |
| Pattern outside the repo | `flutter-research` |

---

## Argument Parsing

- **TARGET** (required) — class/widget name, feature area, function, provider/bloc, or NL description.
- **Depth** — omitted → Step 1's estimate decides the agent count · `quick` → force 1 agent ·
  `full` → force 2-3.

If TARGET missing, ask via `AskUserQuestion`.

---

## Workflow

### 1. Estimate scale (cheap probes)

Use profile paths:
```
Glob {source_roots}/**/<term>*.dart
Glob {test_roots}/**/<term>*.dart
Grep <term> across the narrowed dirs
```

Agent count:
- **1** — < 50 matched files, single package
- **2** — feature touches the app package + a local package (`packages/*`)
- **3** — split: app package / local packages / tests

Don't spawn agents for trivial single-file lookups (overhead > benefit).

### 2. Spawn parallel `Explore` agents (single message → concurrent)

Each gets a tight scope. Prompt template:

```
Scope: {absolute path under a source_root}
Search target: {TARGET}

Find:
1. Primary implementation files (Widgets/Screens, Blocs/Cubits/Notifiers/ViewModels, Repositories, Services)
2. Tests covering them ({test_roots} — unit, widget, integration)
3. Direct usages: callers, route wiring, DI registrations
4. Related models / DTOs / network operations ({networking})
5. Semantics-label / Key usages (if a UI surface and accessibility != none)

Return brief markdown: `file_path:line — one-line description`.
Do NOT paste file contents. Cap at 25 most relevant matches.
```

**Never** scout into `generated_paths` from the profile (`*.g.dart`, `*.freezed.dart`, etc.).

### 3. Aggregate

Combine, dedupe, sort by relevance. **Verify, don't guess** — drop any returned path
that doesn't exist. Note any agent that timed out and continue.

---

## Report Format

```markdown
# Scout Report: <TARGET>

**Branch**: <current>   **Agents**: <N>

## Relevant Files
### App package
- `<path>:<line>` — <desc>
### Local packages
- `<path>:<line>` — <desc>
### Tests
- `<path>:<line>` — <desc>

## Patterns Observed
- <state pattern, DI mechanism, route wiring actually seen>

## Unresolved Questions
- <gap or ambiguity>
```

If invoked inline (no report needed), output to the user directly.
