# flutter-* skill suite

A **project-agnostic Flutter engineering workflow** for Claude Code. Point it at any Flutter app, fill in
**one** file (`.claude/flutter-profile.md`), and you get a full **ticket → PR** pipeline plus
specialist reviewers (used automatically once installed). The skill bodies hardcode **no** app names, paths, architectures, or commands —
every project-specific fact lives in the profile and is read at runtime.

```
flutter-project-init ──→ .claude/flutter-profile.md        # run ONCE per project

flutter-resolve  ── one command: ticket/context → wayfinder → scout → execute → review → PR
      │
      ├─ flutter-scout ────────┐
      ├─ flutter-research ──────┤
      ├─ flutter-brainstorm ────┼─→ wayfinder ─→ flutter-execute ─→ flutter-code-review
      └─ flutter-sequential-thinking  (reasoning aid; plugs in anywhere)
                                          │ apply           │ route
                                          └─── flutter-specialists ───┘
                          (Widget · Async · Testing · TDD · SOLID — Flutter/Dart-primary)

Planning is delegated to mattpocock-skills:wayfinder (charts a spec via its own sub-skills),
then /to-tickets slices it when needed — install mattpocock-skills alongside this suite.
```

## Setup (once per project)

1. **Install the skills** — recommended: install as a Claude Code **plugin** (handles
   install, update, and uninstall natively, across all your projects):
   ```
   /plugin marketplace add knn90/flutter-agent-skills
   /plugin install flutter-skills@flutter-agent-skills
   ```
   Skills are then invocable as `flutter-skills:flutter-resolve`, `flutter-skills:flutter-scout`, … and are
   auto-selected by description as usual. Update with `/plugin marketplace update flutter-agent-skills`;
   remove with `/plugin uninstall flutter-skills@flutter-agent-skills`. (The maintenance-only
   `flutter-skill-consolidate/` is intentionally not part of the plugin.)

   <details><summary>Manual install (no plugin)</summary>

   Copy the skill folders into your app or globally:
   ```bash
   cp -R flutter-skills/*       <project>/.claude/skills/    # the core workflow
   cp -R flutter-specialists/*  <project>/.claude/skills/    # optional specialists you want
   ```
   (Or into `~/.claude/skills/`.) **Don't** ship `flutter-skill-consolidate/` — it's repo-maintenance.
   </details>

   Newly installed skills appear after the Claude Code session is restarted/reloaded.
2. **Create the profile** — run **`flutter-project-init`**. It *detects* an existing app's architecture/paths, or sets conventions for a greenfield app, and writes `.claude/flutter-profile.md`. (Or copy `flutter-skills/flutter-project-init/flutter-profile.template.md` and fill it in by hand.)
3. **Specialists work automatically** once installed (step 1) — `flutter-code-review` routes change-typed
   slices to them and `flutter-execute` applies them while writing. No extra opt-in. The profile's
   `specialists:` field is an **optional override**:
   ```yaml
   # specialists: [flutter-widget-expert, flutter-async-expert]   # restrict to just these
   # specialists: none                                            # turn specialist routing off
   ```
   Leave it unset to auto-use every `flutter-*-expert` you installed.

## How to use it

### The whole loop, one command

```
flutter-resolve ABC-123
flutter-resolve "add pull-to-refresh to the orders list"
flutter-resolve ABC-123 --devs 2          # parallel dev team
flutter-resolve ABC-123 --solo --no-pr    # single agent, stop before the PR
```

`flutter-resolve` is the front door. It runs:

```
fetch ticket/context → create branch → wayfinder (spec) ──(you approve)──► to-tickets (if needed)
    → flutter-scout → flutter-execute (TDD; solo or --team N) → flutter-code-review → /commit → open PR
```

`wayfinder` charts the work as a spec — naming the destination, then resolving its open decisions
one at a time (driving its own sub-skills — prototyping, grilling, research, … — as each demands)
so you approve a **hardened** spec, not one full of unstated assumptions. For a small, clear change
it declines the map and the flow falls straight through to scout → execute.
You approve the spec **before** any code is written, and nothing is pushed without you.

### Or à la carte

Every skill also stands alone:

```
flutter-scout OrderListBloc               # where does this live?
flutter-research "offline sync options"   # sourced technical research
flutter-brainstorm "offline support"      # brutal trade-off analysis → decision report
wayfinder "offline support"                # chart the spec (you approve); a mattpocock-skills skill
flutter-execute <plan-path> --team 2       # implement (TDD; parallel worktree team)
flutter-code-review #42                    # adversarial, multi-lens review of a PR
```

## The skills

**Core workflow — `flutter-skills/`**

| Skill | Role |
|---|---|
| `flutter-project-init` | One-time: writes `.claude/flutter-profile.md` (detect existing app / set greenfield conventions). |
| `flutter-resolve` | **The front door.** ticket/context → wayfinder → scout → execute → review → PR. Orchestrates; never implements directly. |
| `flutter-scout` | Fast, token-efficient parallel code discovery — returns a *map*, not analysis. |
| `flutter-research` | Sourced technical research grounded in the codebase. |
| `flutter-brainstorm` | Brutally honest trade-off analysis → decision report. |
| `flutter-sequential-thinking` | Step-by-step reasoning aid; plugs in anywhere. |
| `wayfinder` *(mattpocock-skills)* | Planning front-end: charts the work as a spec through decision tickets, driving its own sub-skills (prototyping, grilling, research, …); `/to-tickets` then slices the spec. Approval gate on the spec. Not part of this suite — install `mattpocock-skills` alongside it. |
| `flutter-execute` | **The only implementer.** spec → code → verify → review. **TDD always**; solo, or `--team N` (parallel worktree devs + peer review + a dedicated edge-case reviewer + merge). |
| `flutter-code-review` | Multi-lens adversarial review; **routes change-typed slices to the specialists**; precision-over-recall findings. |

**Specialists — `flutter-specialists/` (optional add-ons; auto-used once installed)**

Deep domain reviewers, consolidated from curated community sources with **Flutter / Dart official docs as the source of truth**. `flutter-code-review` triggers them per change type; `flutter-execute` applies them while writing.

| Specialist | Domain |
|---|---|
| `flutter-widget-expert` | Widgets — composition, state, rebuild performance, layout, navigation, accessibility, Material/Cupertino |
| `flutter-async-expert` | Dart async — Futures, Streams, async/await, isolates, cancellation, Zones |
| `flutter-testing-expert` | Testing — unit / widget / golden / integration, mocktail/mockito, bloc_test, flakiness |
| `flutter-tdd` | TDD discipline — red→green→refactor, prove-it bug repro, vertical slices (process, pairs with `flutter-testing-expert`) |
| `flutter-solid-expert` | SOLID + decoupling — SRP/OCP/LSP/ISP/DIP, composition-root DI, framework isolation |

**Maintenance — `flutter-skill-consolidate/` (repo-only; not shipped)**

Rebuilds any specialist (or a core skill like `flutter-code-review`) from its `SOURCES.yaml`: a staleness audit + **web discovery** for newer/better sources + fan-out extraction + synthesis. Run it with no argument to pick a skill from a menu (or `all`). Keeps skills from rotting; Flutter/Dart official sources always win on conflict.

## The profile — the one file that matters

`.claude/flutter-profile.md` is the single source of every project-specific fact: `source_roots`,
`architecture`, `state_type`, `di`, `navigation`, `networking`, `verify_command`,
`high_rigor_domains`, `specialists`, `ticket_fetch`, `default_base_branch`, `pr_tool`, … Every skill
reads it first, so the skill bodies stay generic. Start from
`flutter-skills/flutter-project-init/flutter-profile.template.md`.

The profile is usually **gitignored**, so git worktrees don't inherit it. Every skill falls
back to the main checkout's copy via
`$(git rev-parse --path-format=absolute --git-common-dir)/../.claude/flutter-profile.md`.

## Layout

```
flutter-agent-skills/
├── .claude-plugin/              # plugin + marketplace manifests (the one-command install)
│   ├── plugin.json              #   bundles flutter-skills/ + flutter-specialists/ as the plugin
│   └── marketplace.json         #   self-marketplace listing the plugin
├── README.md
├── flutter-skills/              # core workflow (8 skills)        — shipped by the plugin
│   └── flutter-project-init/flutter-profile.template.md   # copy to <project>/.claude/flutter-profile.md
├── flutter-specialists/         # optional specialist reviewers   — shipped by the plugin
├── flutter-skill-consolidate/   # repo-maintenance (rebuilds skills from sources) — NOT shipped
└── docs/                        # design notes
```

## Core principles (every skill)

- **Profile-driven** — read `.claude/flutter-profile.md` first; never hardcode project facts.
- **Plan-first** — no implementation before an approved spec/plan (hard gate in `flutter-execute`).
- **TDD always** — `flutter-execute` writes a failing test before the code.
- **Verify before claiming** — run `verify_command`, read the output, *then* claim done (empty → analyze/build-only, stated).
- **Flutter / Dart official = source of truth** — specialists and review defer to flutter.dev / dart.dev over community sources.
- **HIGH-RIGOR escalation** — diffs touching the profile's `high_rigor_domains` get a mandatory adversarial pass + correctness audit.
- **Precision over recall** — reviews surface a focused list of real findings, not a wall of nits.
