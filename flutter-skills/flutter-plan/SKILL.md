---
name: flutter-plan
description: "Create detailed, phased implementation plans for a Flutter project. Use for feature planning, refactors, ticket resolution. Does NOT implement — hands off to flutter-execute. Subcommands: archive, red-team, validate."
argument-hint: "[task | TICKET-ID | archive | red-team | validate]"
model: best
effort: max
---

# Plan — Flutter

**YAGNI + KISS + DRY.** Honest, brutal, concise.

> **CORE PRINCIPLE — plan in VERTICAL SLICES, never horizontal layers.**
> Every phase is ONE end-to-end feature (model → repository → state → widget → tests) that leaves the
> app building & green. Not "all models, then all repositories, then all UI." This is the single most
> important decision in the plan.

## Plan mode (MANDATORY)
All research and design happen inside Claude Code **plan mode**. Enter it **before Step 0** —
Steps 0–4 are read-only, so nothing is written or mutated while planning. When the design is
ready, call **`ExitPlanMode`** (Step 5) with the plan summary: this is the approval gate. Only
**after** the user approves do you leave plan mode and write files (Steps 6–7) + run the handoff
(Step 8). If the host can't enter plan mode, say so, present the plan summary inline, and
require explicit approval before writing anything.

## Step 0 — Load profile
Read `.claude/flutter-profile.md`: `architecture`, `state_type`, `di`, `navigation`,
`networking`, `localization`, `accessibility`, `feature_flags`, `verify_command`,
`high_rigor_domains`, `plans_dir`, `source_roots`, `generated_paths`, `rules_file`,
ticket fields. If missing, read the main checkout's copy —
`$(git rev-parse --path-format=absolute --git-common-dir)/../.claude/flutter-profile.md`
(the profile is usually gitignored, so worktrees don't inherit it). Still missing →
run `flutter-project-init`.

Plans touching `high_rigor_domains` must address security, rollback safety, feature-flag
rollout (if `feature_flags` != none), and analytics impact.

## When to Use
New feature; resolving a ticket; refactor with cascade risk; removing a feature flag;
backend schema change needing coordinated client work.

## When NOT to Use
| Case | Use instead |
|---|---|
| Single-file fix, < 20 LoC | Implement directly |
| Need trade-off analysis | `flutter-brainstorm` |
| Just find files | `flutter-scout` |

## Arguments
- **TICKET-ID** (matches `ticket_pattern`) → fetch via `ticket_fetch`.
- **Free-form** → planning subject.
- **Subcommands:** `archive`, `red-team`, `validate`.
- **Mode flags:** `--fast`, `--hard`, `--two`.

If no args, ask via `AskUserQuestion` (default = create plan; or archive/red-team/validate).

### Mode auto-detect
| Flag | Research | Red Team | Validate |
|---|---|---|---|
| default (auto-detect) | per complexity | per complexity | per complexity |
| `--fast` | skip | skip | skip |
| `--hard` | yes | yes | optional |
| `--two` | 2 researchers | after selection | after selection |

Auto-detect: trivial (1 file, <30 LoC) → `--fast`; single clear feature → default;
cross-package / `high_rigor_domains` → `--hard`; real uncertainty between 2 approaches → `--two`.

---

## Flow
`1` Context → `2` Codebase → `3` Research → `4` Design → **`5` ExitPlanMode (gate)** →
`6` Write → `7` Red team/Validate → `8` Handoff.

### Step 1 — Context
Parse args. **Pre-creation scan:** scan `{plans_dir}` for unfinished plans
(`status != completed/cancelled`) — read frontmatter, compare scope (overlapping files,
shared DI registrations, same feature), set `blockedBy`/`blocks` on BOTH plans when a dependency is
found; `AskUserQuestion` if ambiguous. Load `rules_file` + relevant `docs_root` files +
existing brainstorm report for the same ticket. Fetch ticket via `ticket_fetch`. Scope
challenge (skip if `--fast`/trivial): solving the problem or the symptom? sub-tasks
deferrable? one plan or 2-3 sequential?

### Step 2 — Codebase analysis
**Reuse first (DRY):** if a scout map already exists for this task — e.g. `scope.md` in the
working plan dir, written when `flutter-resolve` ran `flutter-scout` upstream — read it instead of
re-scouting. Otherwise gather it now: scope spanning 3+ packages → `flutter-scout`, else `Glob`/`Grep`.
Capture per file to modify: `path:line` · existing pattern to follow (DRY) · test file + pattern ·
DI registrations touched · route wiring touched · network operations touched (never edit
`generated_paths`) · localization keys + Semantics labels needed.

### Step 3 — Research (skip if `--fast`)
`--hard`/`--two` → `flutter-research <topic>`. Default → lightweight `WebSearch` only if truly
needed. **Don't research patterns already in the codebase.**

### Step 4 — Solution design
Per approach: widget tree / state ({state_type}) / navigation split · DI wiring ({di}) · network
operation + cache strategy ({networking}) · feature flag (`feature_flags`) + default-off
rollout · backwards-compat/migration · testing strategy (unit/widget/golden/integration with Semantics
labels) · rollback plan. `--two` → comparison table.

**Then derive the phase plan:**
- **Slice vertically** per the Core Principle. Extract a thin shared foundation phase only for
  what ≥2 slices genuinely need.
- **Dependency graph → order the slices.** Map what depends on what; order bottom-up — typically
  model/persistence → networking/repository → DI wiring → state/bloc → widget → tests. Foundation slice first.
- **Contract first (team runs).** If slices will be built in parallel (complexity MEDIUM/HIGH),
  shared contracts (abstract classes/interfaces, DI registrations, API surface) land in the foundation slice before slices fork.
- **Checkpoint + fail-fast.** High-risk / uncertain slices first; every slice ends at a verification
  checkpoint ({verify_command} green). Split any step touching >~5 files — or with "and" in its
  title (that's two steps).

Hard multi-layer reasoning (data-flow, state transitions, races) → `flutter-sequential-thinking`.

### Step 5 — ExitPlanMode (approval gate)
Present the plan summary via `ExitPlanMode` (the gate — see Plan mode above).

### Step 6 — Plan documentation (only AFTER approval)
Dir: if a working plan dir already exists (resolve-driven: `{plans_dir}/{slug}/`, holding
`_status.md`/`scope.md`), write into it — do NOT create a second dir. Otherwise create
`{plans_dir}/{YYMMDD-HHMM}-{TICKET|slug}/` — `{YYMMDD-HHMM}` from
`bash -c 'date +%y%m%d-%H%M'`.

```
{plans_dir}/{date-ticket-slug}/
├── plan.md                    # master plan + frontmatter
├── phase-1-foundation.md      # thin shared scaffolding — only if ≥2 slices need it
├── phase-2-{slice-a}.md       # vertical slice: model→repository→state→widget→tests
└── phase-3-{slice-b}.md       # next slice; one phase per end-to-end slice
```

`plan.md` frontmatter:
```yaml
---
title: <short title>
ticket: <id | n/a>
status: pending            # pending | in-progress | completed | cancelled
complexity: MEDIUM         # LOW | MEDIUM | HIGH — drives flutter-execute's dev count
mode: auto                 # auto | fast | hard | two
blockedBy: []
blocks: []
created: <YYYY-MM-DD>
---
```

`plan.md` body:
```markdown
## Overview            <2-4 sentences>
## Acceptance Criteria   <feature-level; every AC must map to ≥1 slice's per-phase Acceptance criteria>
## Approach            <chosen + rationale; link brainstorm report if any>
## Phases              one vertical slice per phase, in dependency order (see Step 4)
## File Changes (Summary Table)  | File | Package | Type | Change | Owner |
       # Owner = dev-1/dev-2/dev-3 for team runs (one owner per file → clean merges); "-" for solo
## Feature Flag        Name / Default off / Rollout  (or "n/a" if feature_flags==none)
## Testing Strategy    Unit / Widget / Golden / Integration (Semantics labels needed)
## Risks & Mitigations | Risk | Likelihood | Mitigation |
## Out of Scope
## Open Questions
```

Per-phase file:
```markdown
## Phase N: <slice name>          # one vertical slice, end-to-end
### Goal           <1-2 sentences — the user-visible capability this slice delivers>
### Acceptance criteria           # this slice's share of the plan's ACs — specific & testable
- [ ] <testable condition>
- [ ] <testable condition>
### Steps
1. **<Step>** (file: `<path>:<line>`, size: XS/S/M/L) — what to change · why · test to add/update
       # size = files touched: XS=1 · S=2 · M=3-5 · L=6+ — L means split it (see Step 4)
### Checkpoint        # all boxes green before the next slice starts
- [ ] `{verify_command}` passes  (if unset → analyze/build-only; state it)
- [ ] Acceptance criteria above all met
- [ ] Manual: <if applicable>
### Depends on        Phase <N>, … | none
```
(Files & owners live in `plan.md`'s File Changes table; per-step `size` covers scope — don't
restate them here.)

### Step 7 — Red team / Validate (optional)
**Red team** (`--hard`/`--two` or subcommand) — `flutter-plan red-team <plan-dir>` spawns an
adversarial reviewer:
> Review this plan as a hostile reviewer. Find missing edge cases, unstated assumptions,
> security gaps, state-transition holes, DI/provider lifecycle issues, networking/cache pitfalls,
> rollback risks, accessibility gaps. Be brutal.

Save to `{plans_dir}/{plan-dir}/red-team.md`; update the plan before proceeding.

**Validate** (optional) — `flutter-plan validate <plan-dir>` via `AskUserQuestion`: ACs mapped to
phases? feature flag confirmed with PM? backend schema timing confirmed? design signed off?
localization strings provided? Semantics labels reviewed? QA bandwidth? Save to `validation.md`.

### Step 8 — Execute handoff (do NOT auto-invoke)
```
Plan ready: {plans_dir}/{plan-dir}/plan.md
- Implement now            → flutter-execute {plans_dir}/{plan-dir}/plan.md
- Grill open decisions     → flutter-grill {plans_dir}/{plan-dir}
- Adversarial review first → flutter-plan red-team <plan-dir>
- Validate with team       → flutter-plan validate <plan-dir>
```
(When `flutter-resolve` drives the flow, it runs this handoff for you.)

---

## Subcommands
- **archive** — move `status: completed|cancelled` plans → `{plans_dir}/_archive/{YYMM}/`, append journal.
- **red-team / validate** — standalone Step 7.

## Greenfield note
First 1-2 features have no prior art — the plan **establishes** the pattern. Relax the
"match existing pattern" DRY checks; be extra explicit about the conventions being set
(they become what later plans DRY against).

## Sources
Methodology adapted from
[planning-and-task-breakdown](https://github.com/addyosmani/agent-skills/blob/main/skills/planning-and-task-breakdown/SKILL.md);
provenance in `SOURCES.yaml` (audit-only — hand-curated, never auto-merged).

## Constraints
- **DO NOT** implement — plans only.
- **DO NOT** create plans outside `{plans_dir}`. **MUST** follow `rules_file`.
- **Sacrifice grammar for concision.**
