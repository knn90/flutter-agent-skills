# Full-Flow Flutter Workflow — Design & Implementation Plan

**Status:** ported from the ios-* suite (2026-07-08) — decisions locked (see §6). **Amended 2026-07-29 (§9):**
planning delegated to `mattpocock-skills:wayfinder`; `flutter-plan` + `flutter-grill` removed.
**Goal:** One entry point that takes a *ticket or free-form context* and drives it end-to-end:
**wayfinder (spec) → scout → implement (parallel team) → test → review → PR**, project-agnostic (profile-driven).
**Decision taken:** generalized suite (`flutter-skills/`) + a proven team-execution engine.

---

## 1. What already exists (reuse — do NOT rebuild)

The generalized `flutter-skills/` suite already covers most of the flow. Every skill reads `.claude/flutter-profile.md`.

| Stage | Existing skill | Produces |
|---|---|---|
| Scout | `flutter-scout` | a *map* (paths:line) — parallel Explore agents |
| Plan | `mattpocock-skills:wayfinder` | a spec (charted via decision tickets), approval-aware — *external, see §9* |
| Implement | `flutter-execute` | code; plan-first → verify → review gates; solo + team |
| Review | `flutter-code-review` | 3-stage adversarial review + verdict |
| Research / ideate / reason | `flutter-research`, `flutter-brainstorm`, `flutter-sequential-thinking` | reports / reasoning |
| Bootstrap | `flutter-project-init` | the `flutter-profile.md` itself |

---

## 2. The gap (what we build)

1. **An orchestrator** chaining ticket → scout → plan → execute → review → PR.
2. **A team-execution mode** for the implementer (worktree dev agents + peer review + merge + validation gate).

---

## 3. Design

### 3.1 Skill — `flutter-resolve` (orchestrator)

```
flutter-resolve <TICKET-ID | "free-form description">
            [--devs N] [--base BRANCH] [--solo] [--skip-approval] [--no-pr]
```

Pipeline (each step delegates to an existing skill; orchestrator only sequences + gates):

```
0. Load .claude/flutter-profile.md  (missing → flutter-project-init)
1. Resolve input:
     TICKET-ID (matches ticket_pattern) → fetch via ticket_fetch
     free-form                           → use as task description
2. INIT      → create branch {type}/{slug} off {base}; create plans_dir/{slug}/ + _status.md
3. PLAN      → wayfinder  → charts a spec (── APPROVAL GATE unless --skip-approval); to-tickets if sliced
4. SCOUT     → flutter-scout  → writes scope.md
5. IMPLEMENT → flutter-execute (spec/ticket path) [--team N | --solo]  → code + verify + review
6. COMMIT    → /commit (granular; ticket id from branch)
7. PR        → gh pr create --base {base}   (skip if --no-pr or no gh)
8. TICKET    → best-effort transition (e.g. → CODE REVIEW) if ticket_system supports it
9. REPORT    → summary (branch, PR url, files, tests, review verdict)
```

Branch type from ticket issue-type (Bug→`fix`, Story→`feat`, Task→`chore`) or inferred from description; default `feat`.

### 3.2 The implementer — `flutter-execute --team N`

The implementer honors the suite principle *"one implementer."* Phase 3 has three paths:

| Plan size | Path |
|---|---|
| trivial / small | direct Edit/Write |
| medium (1 feature, 5+ files) | one delegated subagent |
| **large / parallelizable** | **`--team N`: N dev agents in isolated worktrees + peer review + merge + validation** |

The team protocol is large, so it lives in **`flutter-execute/references/team-execution.md`** (loaded only for `--team` runs), keeping the main SKILL.md readable. Protocol:

```
Phase A SPAWN   — N Agent(isolation:"worktree"), dev-1..N (parallel, single message)
Phase B CONTEXT — SendMessage each: their slice of scope.md + plan.md + role + rules_file
Phase C BUILD   — devs implement on their branch; TDD per slice; file ownership enforced
Phase D REVIEW  — peer round-robin (1←2, 2←3, 3←1) + edge-case reviewer; fix-loop; lead sign-off → peer-review.md
Phase E MERGE   — .worktrees/{slug}/integrate; git merge --no-ff each dev branch
Phase F VALIDATE— run verify_command in integrate worktree; route failures back; 2-strike → BLOCKED
```

Role table (generalized; modes = feature / deprecation / flag-removal):

| Mode | dev-1 | dev-2 | dev-3 (if N≥3) |
|---|---|---|---|
| feature | a vertical slice + its tests | another slice + its tests | integration / DI / edge cases + tests |
| deprecation | remove tests + mocks | remove implementation + DI | remove references (imports, route registrations, exports) |
| flag-removal | simplify kept path | remove flag + dead code | test cleanup |

### 3.3 Profile changes — **minimal** (push back on over-spec)

Almost every project-specific fact is already covered by the generalized model — adding fields would violate the suite's "facts that vary, nothing else" rule.

| Project hardcode (example) | Generalized source — **no new field** |
|---|---|
| `flutter test` / `flutter analyze` / `melos run test` (build + test gates) | **`verify_command`** (the single source of truth) |
| a design-system package, logging wrapper, DI container, bloc/notifier base, crash reporter | **`rules_file`** + existing `architecture`/`state_type`/`di`/`networking`/`crash_reporting` + profile-gated checks |
| hardcoded source + test directories | **`source_roots`** / **`test_roots`** |
| a feature-flag config + defaults | **`feature_flags`** |
| `.worktrees/`, `_status.md`, `dev-N.md`, `dev-N` naming | **engine internals** — sane defaults, not project facts |
| deprecation reference check via `rg` over source_roots | **universal** — bake into engine, not profile |
| project-specific commit scopes | delegate to `/commit` + note in `rules_file` |

**Genuinely new — both OPTIONAL, with safe defaults:**

```yaml
default_base_branch: main      # branch/PR target; overridable by --base. (default: main)
pr_tool: gh                    # gh | none. none → stop after push, print manual PR steps.
```

Task workspace reuses **`plans_dir/{slug}/`** (where `plan.md` already lands) for `scope.md`, `_status.md`, `dev-N.md`, `peer-review.md`. No new "tasks dir" concept.

### 3.4 Where testing lives

Testing is woven through, not bolted on:

1. **Plan** — the `wayfinder` spec carries a *Testing Strategy* (unit / widget / golden / integration + Semantics labels).
2. **Implement** — each team dev writes tests **first** for their slice (TDD); solo mode writes tests alongside.
3. **Verify gate** — `verify_command` (analyze + test) must pass; output is read, not assumed. Unset → analyze/build-only, stated explicitly.
4. **Review gate** — `flutter-code-review` checks test coverage of changed logic + transition holes.

---

## 4. End-to-end data flow

```
TICKET-ID / context
      │
      ▼
flutter-resolve ──INIT──► branch + plans_dir/{slug}/_status.md
      │
      ├─► wayfinder ───────────► spec ──[APPROVAL GATE]── to-tickets (if sliced)
      │
      ├─► flutter-scout ───────────► scope.md
      │
      ├─► flutter-execute --team N ► .worktrees/{slug}/dev-1..N ──peer review──► integrate
      │        │                                                                  │
      │        └─ VERIFY: verify_command ◄───────────────────────────────────────┘
      │
      ├─► flutter-code-review ─────► verdict (Critical must clear)
      │
      ├─► /commit ─────────────────► granular commits
      │
      └─► gh pr create ────────────► PR url ──► (best-effort ticket transition)
```

Artifacts under `plans_dir/{slug}/`: `_status.md`, `scope.md`, spec/ticket pointers (the spec itself lives on the wayfinder tracker), `dev-1..N.md`, `peer-review.md`.

---

## 5. Implementation plan (phases)

| Phase | Deliverable | Verify |
|---|---|---|
| **1. Profile + scaffold** | Add `default_base_branch`, `pr_tool` to `flutter-profile.template.md`; create `flutter-resolve/` folder; add `flutter-execute/references/` | template parses; folders exist |
| **2. Team engine** | `flutter-execute/references/team-execution.md` (spawn→context→build→review→merge→validate, generalized prompts/roles/gates); wire `--team N` + `--solo` flags into `flutter-execute` Phase 3 | dry-read: every project hardcode replaced by a profile ref or documented default |
| **3. Orchestrator** | `flutter-resolve/SKILL.md` (full pipeline §3.1, gates, ticket fetch, branch logic, PR, report) | walk-through on a sample ticket id + a free-form string (no code run) |
| **4. Fallbacks** | Handle: no ticket system, `ticket_fetch: none`, `pr_tool: none`, `verify_command` unset, `gh` not authed, merge conflict → BLOCKED | each path prints actionable guidance |
| **5. Docs** | Update root `README.md` pipeline diagram to include `flutter-resolve`; link this design | README shows the entry point |
| **6. Dry-run** | Exercise the chain end-to-end on one real Flutter app with a filled profile (small ticket) | branch + scope.md + plan.md + PR produced; gates pass |

Phases 2 and 3 are the bulk. 1 is quick. 4–6 are hardening.

---

## 6. Decisions (locked)

1. **Orchestrator name** — **`flutter-resolve`**.
2. **Engine placement** — **folded into the implementer** (`flutter-execute`); the `--team N` protocol lives in `flutter-execute/references/team-execution.md`.
3. **Dedicated test skills** — **deferred**. Testing = TDD in team mode + `verify_command` gate + review coverage check.
4. **Default dev count** — **auto from plan complexity**: LOW→solo, MEDIUM→2, HIGH→3; `--devs N` / `--solo` always override.

---

## 7. Risks

- **Worktree overhead** on consuming repos (disk + setup); mitigate by defaulting to solo for small plans, team only when warranted.
- **Device/test contention** when several dev agents run `verify_command` concurrently — keep verification at the *integrate* step (post-merge), not per-dev, to serialize it.
- **Profile drift** — engine must fail loud if `verify_command` is empty (analyze/build-only) rather than silently "passing".
- **Over-generalization** — resist re-adding project-specific fields; `verify_command` + `rules_file` are the pressure-relief valves.

---

## 8. Hardening decisions

- **Dev worktree model** — native harness `isolation:"worktree"`; each dev commits to branch `{slug}/dev-N`; merge + peer-review happen by **branch name** (`git diff {base}...{slug}/dev-N`), so paths don't matter; the harness auto-cleans dev worktrees; only the integrate worktree is skill-managed.
- **Plan inputs** — the `complexity` signal (drives the auto dev-count) and per-file **Owner** assignment now come from the `wayfinder` spec / `to-tickets` slices; `flutter-execute` estimates them itself when charting a micro-plan for a small change. *(Originally emitted by `flutter-plan` — see §9.)*
- **Test split** — **vertical slices**: each dev owns code **and** tests for their slice (real local TDD, clean per-file ownership), no dedicated test dev. Independent edge-case coverage is a dedicated adversarial **edge-case / logic-gap reviewer agent** (Phase D) that drives missing tests *pre-merge*; the post-merge `flutter-code-review` adversarial pass stays the final gate.
- **Bug fixes** — (1) the team engine builds on the **task branch** (`BASE`), ff-merged back into it; (2) team-mode review uses `git diff {base}...{slug}/integrate`, not `--pending`; (3) devs must compile (`flutter analyze` clean) their worktree before reporting done (full test gate stays serialized at integrate).

**Known soft spot (for the dry-run):** the whole team engine is unexercised, and the spec reliably carrying `complexity` / `Owner` is LLM-dependent. The dry-run (§5 phase 6) is where these get validated.

---

## 9. Amendment — wayfinder-driven planning (2026-07-29)

Planning moved out of this suite. The bespoke `flutter-plan` (phased `plan.md`) and `flutter-grill`
(decision-hardening) skills are **removed**; their role is now filled by
[`mattpocock-skills:wayfinder`](https://github.com/mattpocock), which charts the work as a **spec**
through decision tickets — driving its own sub-skills (prototyping, grilling, research, …)
internally, then `/to-tickets` to slice the spec when the work spans more than one session.

**New pipeline:** `input → init → wayfinder (spec) ──approval──► to-tickets (if needed) → flutter-scout → flutter-execute → commit → PR`.

**Rationale / trade-offs:**
- **Grilling doesn't disappear** — it moves *inside* wayfinder (which grills to name the destination and map the frontier). We drop the separate `flutter-grill` pass on a `plan.md`.
- **`flutter-execute` decoupled from `plan.md`** — it now consumes a spec/tickets (or micro-plans a trivial change itself); `complexity` / `Owner` come from the spec/scope rather than a plan frontmatter field.
- **External dependency + tracker** — wayfinder is a `mattpocock-skills` plugin and its ticket ops assume `/setup-matt-pocock-skills` has configured an issue tracker (defaults to local-markdown). Install it alongside this suite.
- **Small-task escape hatch** — if wayfinder finds no fog (the change fits one session), it declines the map and `flutter-resolve` falls straight through to scout → execute.

Sections §1–§8 above are the original v1 record; where they still name `flutter-plan` / `flutter-grill`, this section supersedes them.
