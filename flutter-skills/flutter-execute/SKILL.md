---
name: flutter-execute
description: "ALWAYS activate before implementing any feature, plan, or fix in a Flutter project — except trivial single-file edits (< 20 lines). The only implementer in the suite."
argument-hint: "[task | plan-path | TICKET-ID] [--fast | --auto | --no-test | --team N | --solo]"
---

# Execute — Flutter implementation

**The only implementer** in the suite. **YAGNI + KISS + DRY.** Concise reports — sacrifice
grammar for concision.

## Step 0 — Load profile

Read `.claude/flutter-profile.md`: `architecture`, `state_type`, `di`, `navigation`,
`networking`, `localization`, `accessibility`, `feature_flags`, `crash_reporting`,
`verify_command`, `high_rigor_domains`, `generated_paths`, `plans_dir`, `rules_file`,
ticket fields. If missing, read the main checkout's copy —
`$(git rev-parse --path-format=absolute --git-common-dir)/../.claude/flutter-profile.md`
(the profile is usually gitignored, so worktrees don't inherit it). Still missing →
run `flutter-project-init`.

## When NOT to Use
| Case | Use instead |
|---|---|
| Need trade-off discussion | `flutter-brainstorm` |
| Need a written plan first | `flutter-plan` |
| Just find files | `flutter-scout` |
| Library docs | `WebFetch` against the official URL |
| Trivial fix (< 20 lines, single file) | Direct edit |

---

## HARD-GATE: Plan-First
```
Do NOT write implementation code until a plan exists and has been reviewed.
Exceptions:
- --fast — skips research but still does scout → micro-plan → code
- User explicitly says "just code it" / "skip planning" — respect the override
```

| Tempting | Reality |
|---|---|
| "Too simple to plan" / "I'll plan as I go" | 30-sec plan prevents 30-min rework |

## Argument Parsing
- **Plan path** (`{plans_dir}/.../plan.md`/`phase-*.md`) → execute plan mode.
- **TICKET-ID** → `ticket_fetch` → matching plan, else `flutter-plan TICKET-ID`.
- **Free-form** → interactive.
- **Flags:** `--fast` (scout→micro-plan→code), `--auto` (auto-approve gates, sparingly),
  `--no-test` (skip the final verify *run* only — you'll run it yourself; TDD still drives the code; record in report),
  `--team N` (parallel dev team in worktrees — see Phase 3), `--solo` (force single-agent),
  `--devs N` (alias for `--team N`).

If no args, ask via `AskUserQuestion` (what to implement + mode).

---

## Process Flow
```
1. Parse args
2. Resolve plan (load OR flutter-plan creates OR --fast micro-plan)
3. Plan Review Gate — user approval (skip only with --auto)
4. Domain rigor flag — touches high_rigor_domains? → HIGH-RIGOR
5. Implementation — test-first (TDD); solo (direct edits / one subagent) OR --team N (worktree dev team)
6. Verification — {verify_command} (MANDATORY)
7. Code Review — flutter-code-review --pending (MANDATORY)
8. Finalise — update plan status, ask before commit
```

## Phase 1 — Load context
Resolve the plan (plan path → Read; ticket → fetch + find/create plan; free-form → trivial
inline plan or `flutter-plan`). Load `rules_file` + relevant `docs_root`.

**Domain rigor flag:** if the task touches any `high_rigor_domains` → mark **HIGH-RIGOR**:
- adversarial review **mandatory** at the review gate
- exact money handling — integer minor units or a `Decimal`/`Rational` package, never `double`
- feature flag default-off + gradual rollout (if `feature_flags` != none)
- no PII in logs / analytics / `crash_reporting` breadcrumbs

## Phase 2 — Plan Review Gate
User approval required (skipped only with `--auto`, or when `flutter-resolve` drives the run —
its approval gate on the hardened plan already satisfies this one). Present: files to change (path:line),
phases, test strategy (widget/golden Semantics labels?), feature flag, HIGH-RIGOR flag.
`AskUserQuestion`: Approved → proceed · Revise → regenerate · Abort → stop.
**Do NOT start implementing without approval.**

## Phase 3 — Implementation

**Pick the execution path by plan size** (its complexity / phase count):

| Plan size | Path | How |
|---|---|---|
| trivial / small | **direct** | Edit/Write yourself |
| 1 feature, 5+ files | **delegated** | one `general-purpose` Agent with the plan + `rules_file` |
| large / parallelizable, or `--team N` | **team** | N dev agents in isolated worktrees + peer review + merge (see below) |

**Auto dev count** (when none of `--team N` / `--devs N` / `--solo` is given): read the plan's
`complexity` field (set by flutter-plan) → **LOW = solo · MEDIUM = 2 · HIGH = 3**. `--solo` forces 1; an explicit
`--team N` / `--devs N` always wins. For team runs, **load and follow
`references/team-execution.md`** (spawn → context → build → peer review → merge → validate);
it reuses the same profile + discipline below. The discipline applies to **every** path —
your own edits *and* the dev agents' work.

**Apply specialists (on by default).** When the code touches a domain with an installed `flutter-*-expert`
skill — widgets/UI → `flutter-widget-expert`, async/Future/Stream/isolates → `flutter-async-expert`, tests →
`flutter-testing-expert`, new/altered classes, abstract interfaces, services, or DI wiring → `flutter-solid-expert` —
**load its `SKILL.md` and follow it** as you write. No profile entry needed; the
profile's `specialists:` can restrict the set or `none` to disable. (For `--team`, each dev does this for its slice.)

**Test-First (TDD) — always: solo and team, greenfield and existing.** **Load and follow
`flutter-tdd`** — the canonical home for the RED→GREEN→REFACTOR cycle, the prove-it pattern, and
vertical slicing; the cycle's rules live there, not here. `flutter-tdd`'s non-logic exemptions
must be stated in the report.

**Discipline (enforce `rules_file` + profile conventions while editing):**
- **State** — use `{state_type}`. No ad-hoc one-off state classes when a shared type exists.
- **DI** — inject via `{di}`. Don't reach for global singletons/service-locators from a widget when DI exists.
- **Strings** — if `localization` != none, every user-facing string via the localization
  mechanism. Never hardcode. Never edit generated localization files.
- **Navigation** — follow `{navigation}` (e.g. routes via go_router/router; widgets don't
  `Navigator.push` directly if the convention centralizes routing).
- **Interfaces** — repositories/services have an abstract class/interface for mockability (if that's the convention).
- **Accessibility** — every UI surface used in tests gets a `Semantics` label / stable `Key` per `accessibility`.
- **Generated code** — never edit anything under `generated_paths` (`*.g.dart`, `*.freezed.dart`, …); rerun `build_runner` instead.
- **`const` & rebuilds** — prefer `const` constructors; don't rebuild subtrees needlessly (that's `flutter-widget-expert`'s lens).
- **Comments** — WHY-only. No history, no "added for ticket X", no paraphrasing the code.

**Commit cadence — one concern per commit, via `/commit`** (which extracts the ticket id
from the branch). Commit at each milestone (phase done, self-contained refactor, new
classes/DI wiring, new localization keys, new network operation + codegen, tests, golden
updates). Run relevant tests before each commit — never commit a broken build mid-feature.
Always use `/commit`, not raw `git commit`.

**Tracking:** multi-phase work → `TaskCreate`/`TaskUpdate`; mark `in_progress` before
starting, `completed` when done, don't batch. Update plan checkboxes inline with commit shas.

## Phase 4 — Verification Gate (MANDATORY)
**Iron law: no completion claim without fresh verification — run `{verify_command}`, read the
actual output, claim done only on success.** Thinking "should pass" / "probably fine" is the
red flag: run it instead.

- **`verify_command` unset/empty** → run an **analyze/build-only** check (e.g. `flutter analyze`
  and/or `flutter build <target> --debug`) and state explicitly in the report: "verify_command
  not configured — analyze/build-only, tests not run." Don't pretend tests passed.
- **UI work** → also exercise the change on a device/emulator (`/run` or `/verify` skill) —
  `flutter analyze` verifies code, not feature correctness.
- User said "I'll test myself" → respect, but **say so explicitly** and skip running it.

## Phase 5 — Code Review Gate (MANDATORY)
Run `flutter-code-review --pending`. **Team mode:** the work is committed on `{SLUG}/integrate`, so
review the merged diff instead — `flutter-code-review` on `git diff {BASE}...{SLUG}/integrate`
(`--pending` can't see committed merges). For HIGH-RIGOR tasks, adversarial review is mandatory
(the review skill runs it automatically). Resolve all **Critical** before proceeding;
**Important** → fix or document deferral with a ticket.

## Phase 6 — Finalise (only after 4 + 5 pass)
- Update `plan.md` frontmatter `status: completed`; tick remaining checkboxes.
- Update docs if implementation changed architecture/conventions (light edits OK; rewrites
  need approval).
- **Plan lifecycle:** the plan stays on the branch during dev. If your PR flow removes it
  before opening the PR, that's the PR step's job — not here. Mention `flutter-plan archive`
  if worth keeping as a decision record.
- **Final commit:** most work is already committed via granular `/commit`. If the tree is
  dirty (checkbox/status/doc tweaks), `/commit` once more. **Do NOT push** without explicit
  instruction.

Final report (solo; team runs add peer-review/merge lines per `references/team-execution.md`):
```
✓ Execute complete: <task slug>
- Plan: {plans_dir}/{slug}/plan.md → status: completed
- Files changed: N (paths…)
- Tests: <n> added, all passing   (or "analyze/build-only — verify_command unset")
- Verify: ✅ PASSED / analyze-build-only / user-tested
- Review: passed (X critical resolved, Y important deferred)
- Commit: <sha or pending user action>
- Open follow-ups: <list or none>
```

## Greenfield note
First feature establishes the pattern — there's no prior art to match. Be deliberate about
the conventions you set (they're what every later run inherits). Once a reference feature
and a real `verify_command` exist, full rigor resumes.
