---
name: flutter-resolve
description: "Resolve a ticket or a free-form request end-to-end for a Flutter project — the single front door from input to open PR. Use when given a ticket id or told to 'resolve/ship/work on' something."
argument-hint: "[TICKET-ID | \"description\"] [--devs N | --solo] [--base BRANCH] [--skip-approval] [--no-grill] [--no-pr]"
---

# Resolve — Flutter end-to-end workflow

One command takes a **ticket or free-form context** all the way to an open PR. This skill only
**sequences and gates** — the real work is done by the other `flutter-*` skills: never re-do what a
sub-skill already does, never skip a gate. Project-agnostic: every fact comes from
`.claude/flutter-profile.md` — never hardcode app specifics.

## Step 0 — Load profile

Read `.claude/flutter-profile.md`: `source_roots`, `plans_dir`, `high_rigor_domains`,
`feature_flags`, `ticket_system`, `ticket_pattern`, `ticket_fetch`,
`default_base_branch`, `pr_tool` (sub-skills load the rest themselves).
If missing, read the main checkout's copy —
`$(git rev-parse --path-format=absolute --git-common-dir)/../.claude/flutter-profile.md`
(the profile is usually gitignored, so worktrees don't inherit it). Still missing →
run `flutter-project-init` first. If `source_roots` is empty (greenfield, nothing to
scout/implement against) → tell the user to bootstrap the app first.

## When to Use / NOT

| Use | Don't — use instead |
|---|---|
| A ticket id to take to PR | Just find files → `flutter-scout` |
| "Resolve / ship / work on X" end-to-end | Just a plan, no code → `flutter-plan` |
| A described change spanning scope→plan→code→PR | Implement an existing plan → `flutter-execute <plan-path>` |
| | Trade-off discussion → `flutter-brainstorm` · review only → `flutter-code-review` |

For a trivial one-file fix, skip this — edit directly (or `flutter-execute --fast`).

---

## Syntax
```
flutter-resolve ABC-123
flutter-resolve "add pull-to-refresh on the orders list"
flutter-resolve ABC-123 --devs 3 --base release/1.2
flutter-resolve ABC-123 --solo --skip-approval
flutter-resolve "fix crash on empty cart" --no-pr
```

## Argument parsing
- **TICKET-ID** — matches `ticket_pattern` → fetch via `ticket_fetch` (Step 1).
- **"free-form"** — used as the task description (no ticket fetch).
- `--devs N` / `--solo` — passed through to `flutter-execute`.
- `--base BRANCH` — PR target + branch-from point (default `default_base_branch`, else `main`).
- `--skip-approval` — skip the plan approval gate (trivial/well-understood only). Also skips grill.
- `--no-grill` — skip the plan stress-test (Step 4.5) but keep the approval gate.
- `--no-pr` — stop after commit; print push/PR steps for the user.

---

## Pipeline

```
0. Load profile
1. RESOLVE INPUT   ticket → fetch context · free-form → use as description
2. INIT            branch {type}/{slug} off {PR_BASE}; create {plans_dir}/{slug}/_status.md
3. SCOUT           flutter-scout → scope.md
4. PLAN            flutter-plan  → plan.md
4.5 GRILL          flutter-grill → hardened plan.md + grill.md
    GATE           ──► APPROVAL on the hardened plan
5. IMPLEMENT       flutter-execute (owns the verify + review gates)
6. COMMIT          /commit anything left
7. PR              push + open PR → {PR_BASE}
8. TICKET          best-effort transition (e.g. → In Review)
9. REPORT          branch · PR url · files · tests · review verdict
```

### Step 1 — Resolve input
- **TICKET-ID:** fetch via `ticket_fetch` (an MCP tool or CLI named in the profile — discover
  with `ToolSearch` if it's an MCP server). Extract: summary, description, acceptance criteria,
  issue type, labels. If `ticket_fetch: none` or the fetch fails → ask the user to paste the
  ticket text, then continue.
- **Free-form:** the argument *is* the task; no fetch.
- Pick **branch type** from issue type (`Bug`→`fix` · `Story`/feature→`feat` · `Task`/chore→`chore`),
  else infer from the description; default `feat`.

### Step 2 — Init
- `PR_BASE` = `--base` / `default_base_branch` / `main`.
- `slug` = ticket id (if any) + short kebab summary (≤ 50 chars).
- `git fetch && git checkout -b {type}/{slug} origin/{PR_BASE}` (resume if it already exists).
- Create `{plans_dir}/{slug}/_status.md` (task title, ticket, base, branch, phase checklist).

### Step 3 — Scout
Invoke `flutter-scout {target}` (target = main component/feature from the ticket, or the description).
Write its map to `{plans_dir}/{slug}/scope.md`. Skip only for a truly trivial single-file change.

### Step 4 — Plan
Invoke `flutter-plan` for this task (point it at the Step 3 `scope.md` so it reuses that map instead
of re-scouting) → writes `{plans_dir}/{slug}/plan.md` (phased, with a Testing
Strategy, a `complexity` field, and an Owner column on the File Changes table for team runs).
**Read the plan's `complexity`** — surface it at the approval gate.

### Step 4.5 — Grill (harden the plan)
Unless skipped (`--no-grill` / `--skip-approval`), invoke `flutter-grill {plans_dir}/{slug}` to
interrogate the open decisions in `plan.md` one at a time (each with a recommended answer,
codebase checked first) → writes the resolved decisions back into `plan.md` + a `grill.md` log.
It auto-skips (no questions) when the plan is already unambiguous. The approval gate below then
runs on the **hardened** plan.

### Approval gate
Unless `--skip-approval`, present a summary of the hardened plan and **wait**:
```
Plan ready: {plans_dir}/{slug}/plan.md
- Complexity: {…} (dev count auto-picked by flutter-execute)   Files: {n}   Feature flag: {name|n/a}   HIGH-RIGOR: {yes/no}
- Key changes: {3-5 bullets}
Reply: approved · revise: {feedback} · abort
```
`revise:` → re-run `flutter-plan` with feedback. `abort` → set `_status.md` CANCELLED, stop.

### Step 5 — Implement
Hand the approved plan to the implementer **from the task branch** (so the team engine's
`BASE` = `{type}/{slug}` and the merge stays in-lineage):
```
git checkout {type}/{slug}
flutter-execute {plans_dir}/{slug}/plan.md [--team N | --solo]
```
- Pass `--devs`/`--solo` through if given; otherwise `flutter-execute` auto-picks from the plan's `complexity`.
- **Team mode** returns `{slug}/integrate` (already verified + reviewed inside `flutter-execute`).
  Fast-forward it onto the task branch: `git checkout {type}/{slug} && git merge --ff-only {slug}/integrate`.
  Solo mode commits onto `{type}/{slug}` directly — nothing to merge.
- If `flutter-execute` returns BLOCKED (e.g. team merge conflict — it reports files/owners) →
  surface its report and stop.

### Step 6 — Commit
`flutter-execute` commits granularly via `/commit`. If anything is still uncommitted (status/doc
tweaks, the integrate merge), run `/commit` once more. Never `git commit --no-verify`.

### Step 7 — PR
If `--no-pr` or `pr_tool: none` → push nothing; print the manual `git push` + PR command and stop at Step 9.
Otherwise (`pr_tool: gh`):
```bash
git push -u origin {type}/{slug}
gh pr create --base {PR_BASE} --fill   # or build a body from plan.md + ticket link
```
If a `create-pr-description` / PR-template skill is available, use it for the body. If `gh` is
absent/unauthed → print the manual steps (don't fail the whole run).

### Step 8 — Ticket transition (best-effort)
If a ticket was fetched and `ticket_system` supports transitions, move it to the review state
(e.g. "In Review" / "Code Review"). On failure → log a warning, continue. Skip for free-form runs.

### Step 9 — Report
```
## Resolved: {ticket id | description}
Branch: {type}/{slug} → {PR_BASE}     PR: {url | "not created (--no-pr)"}
Mode: {solo | team N}                 Ticket: {new state | n/a}
Files: {n}   Tests: {n} ({passing | analyze/build-only})   Review: {APPROVED | issues}
Artifacts: {plans_dir}/{slug}/ (scope.md, plan.md, _status.md[, dev-*.md, peer-review.md])
Next: review the PR · add reviewers · merge when green
```

---

## Gates & constraints (never skip)
1. **Approval** after planning (unless `--skip-approval`).
2. **Verify** — owned by `flutter-execute`.
3. **Review** — owned by `flutter-execute`.

- **DO NOT** implement code yourself — `flutter-execute` is the only implementer.
- **DO NOT** auto-merge to `{PR_BASE}`; never push before Step 7.
- **Sacrifice grammar for concision** in reports.
