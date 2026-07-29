---
name: flutter-resolve
description: "Resolve a ticket or a free-form request end-to-end for a Flutter project — the single front door from input to open PR. Use when given a ticket id or told to 'resolve/ship/work on' something."
argument-hint: "[TICKET-ID | \"description\"] [--devs N | --solo] [--base BRANCH] [--skip-approval] [--no-pr]"
---

# Resolve — Flutter end-to-end workflow

One command takes a **ticket or free-form context** all the way to an open PR. This skill only
**sequences and gates** — the real work is done by `wayfinder` (planning) and the other `flutter-*`
skills (implementation): never re-do what a sub-skill already does, never skip a gate.
Project-agnostic: every fact comes from `.claude/flutter-profile.md` — never hardcode app specifics.

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
| "Resolve / ship / work on X" end-to-end | Just chart the spec → `wayfinder` |
| A described change spanning spec→code→PR | Implement an existing spec/tracer-bullet ticket → `flutter-execute <path>` |
| | Trade-off discussion → `flutter-brainstorm` · review only → `flutter-code-review` |

**Sizing** (referenced below):
- **trivial** — single file, < ~20 lines → skip this skill; edit directly (or `flutter-execute --fast`).
- **small** — fits one session, no open decisions → wayfinder declines the map (Step 3); scout may be skipped (Step 5).

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
- `--skip-approval` — skip the spec approval gate (trivial/well-understood only).
- `--no-pr` — stop after commit; print push/PR steps for the user.

---

## Pipeline

```
0. Load profile
1. RESOLVE INPUT   ticket → fetch context · free-form → use as description
2. INIT            branch {type}/{slug} off {PR_BASE}; create {plans_dir}/{slug}/_status.md
3. WAYFINDER       /mattpocock-skills:wayfinder → chart map · answer questions live · land the spec
                   (single session, human-present; drives its own sub-skills internally)
    GATE           ──► APPROVAL on the spec
4. SLICE           /mattpocock-skills:to-tickets → break the spec into tracer-bullet tickets (if needed)
5. SCOUT           flutter-scout → scope.md
6. IMPLEMENT       flutter-execute (owns the verify + review gates)
7. COMMIT          /commit anything left
8. PR              push + open PR → {PR_BASE}
9. INPUT TICKET    best-effort transition (e.g. → In Review)
10. REPORT         branch · PR url · files · tests · review verdict
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

### Step 3 — Wayfinder (chart the map → spec)
Invoke `/mattpocock-skills:wayfinder` with the resolved input from Step 1 (the input ticket context)
as the loose idea, plus a **Note that this is a single-session, human-present effort: chart, then
drive to a spec now** — overriding wayfinder's default one-ticket-per-session / stop-after-charting
cadence. Then, in this one session, wayfinder:
1. **Charts the map** — names the destination (a **spec** to hand to implementation) and surfaces the open **decision tickets**.
2. **Works them live** — the user answers each clarification question in turn (wayfinder grills / prototypes / researches per ticket). **Never answer for the user.**
3. **Synthesises the spec** once the decisions are made.

Let wayfinder own all decision-hardening — **do not** grill or plan separately. Record the spec's
location (file path or issue URL) in `_status.md` so later steps find it.

**Small-task fall-through:** for a **small** change (Sizing above — wayfinder declines the map),
skip the map: proceed straight to Step 5 (Scout) and Step 6 (Execute) with the input as the spec.
Note this in `_status.md`.

### Approval gate
Unless `--skip-approval`, present a summary of the spec and **wait**:
```
Spec ready: {spec location}
- Destination: {one line}     Files touched (est.): {n}   Feature flag: {name|n/a}   HIGH-RIGOR: {yes/no}
- Key decisions: {3-5 bullets from the wayfinder map's Decisions-so-far}
Reply: approved · revise: {feedback} · abort
```
`revise:` → re-enter wayfinder with the feedback. `abort` → set `_status.md` CANCELLED, stop.

### Step 4 — Slice into tracer-bullet tickets (if needed)
If the spec spans more than one vertical slice, invoke `/mattpocock-skills:to-tickets` to break it
into **tracer-bullet tickets** (GitHub issues / Jira) with blocking edges on the tracker. Skip for a
single-slice change — the spec goes straight to Scout + Execute.

### Step 5 — Scout
Invoke `flutter-scout {target}` (target = main component/feature from the spec or input ticket).
Write its map to `{plans_dir}/{slug}/scope.md`. Skip only for a **trivial** change (Sizing above).

### Step 6 — Implement
Hand the approved spec (and **tracer-bullet tickets**, if sliced) to the implementer **from the task
branch** (so the team engine's `BASE` = `{type}/{slug}` and the merge stays in-lineage):
```
git checkout {type}/{slug}
flutter-execute {spec path | tracer-bullet ticket} [--team N | --solo]
```
- Pass `--devs`/`--solo` through if given; otherwise `flutter-execute` auto-picks from the spec's scope.
- **Team mode** returns `{slug}/integrate` (already verified + reviewed inside `flutter-execute`).
  Fast-forward it onto the task branch: `git checkout {type}/{slug} && git merge --ff-only {slug}/integrate`.
  Solo mode commits onto `{type}/{slug}` directly — nothing to merge.
- If `flutter-execute` returns BLOCKED (e.g. team merge conflict — it reports files/owners) →
  surface its report and stop.

### Step 7 — Commit
`flutter-execute` commits granularly via `/commit`. If anything is still uncommitted (status/doc
tweaks, the integrate merge), run `/commit` once more. Never `git commit --no-verify`.

### Step 8 — PR
If `--no-pr` or `pr_tool: none` → push nothing; print the manual `git push` + PR command and stop at Step 10.
Otherwise (`pr_tool: gh`):
```bash
git push -u origin {type}/{slug}
gh pr create --base {PR_BASE} --fill   # or build a body from the spec + ticket link
```
If a `create-pr-description` / PR-template skill is available, use it for the body. If `gh` is
absent/unauthed → print the manual steps (don't fail the whole run).

### Step 9 — Input-ticket transition (best-effort)
If an **input ticket** was fetched and `ticket_system` supports transitions, move it to the review
state (e.g. "In Review" / "Code Review"). On failure → log a warning, continue. Skip for free-form runs.

### Step 10 — Report
```
## Resolved: {input ticket id | description}
Branch: {type}/{slug} → {PR_BASE}     PR: {url | "not created (--no-pr)"}
Mode: {solo | team N}                 Ticket: {new state | n/a}
Files: {n}   Tests: {n} ({passing | analyze/build-only})   Review: {APPROVED | issues}
Artifacts: {plans_dir}/{slug}/ (scope.md, _status.md[, spec / tracer-bullet ticket links, dev-*.md, peer-review.md])
Next: review the PR · add reviewers · merge when green
```

---

## Gates & constraints (never skip)
1. **Approval** after wayfinder produces the spec (unless `--skip-approval`).
2. **Verify** — owned by `flutter-execute`.
3. **Review** — owned by `flutter-execute`.

- **DO NOT** implement code yourself — `flutter-execute` is the only implementer.
- **DO NOT** plan or grill separately — `wayfinder` owns planning and decision-hardening.
- **DO NOT** auto-merge to `{PR_BASE}`; never push before Step 8.
- **Sacrifice grammar for concision** in reports.
