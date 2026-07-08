---
name: flutter-brainstorm
description: "Brainstorm solutions for a Flutter app. Use for ideation, architecture decisions, or feasibility checks. Does NOT implement — hands off to flutter-plan."
argument-hint: "[topic | TICKET-ID | description]"
model: best
effort: xhigh
---

# Brainstorm — Flutter

Collaborate to find the best solution while being **brutally honest** about feasibility,
trade-offs, and over-engineering.

**YAGNI + KISS + DRY.** Prefer the boring, proven approach already used in the codebase.

## Step 0 — Load profile

Read `.claude/flutter-profile.md` — keys are referenced at point of use below.
If missing, read the main checkout's copy —
`$(git rev-parse --path-format=absolute --git-common-dir)/../.claude/flutter-profile.md`
(the profile is usually gitignored, so worktrees don't inherit it). Still missing →
run `flutter-project-init`.

<HARD-GATE>
Brainstorm and advise only. Do NOT invoke any implementation skill, write code, write
plans, or scaffold anything until you have presented a design AND the user has approved
it — regardless of perceived simplicity. If you think you already know the answer,
writing it down takes 30 seconds.
</HARD-GATE>

---

## Phase 1 — Context
- Parse args: matches `ticket_pattern` → fetch via `ticket_fetch`; extract title,
  description, ACs, links. Otherwise treat as a free-form topic.
- Load `rules_file` (honor it throughout), relevant `docs_root` files, branch name +
  `git log -5 --oneline`.
- Scout for non-trivial topics (`flutter-scout` if scope spans 3+ packages/features). **Never recommend
  something whose existence you haven't verified.**

## Phase 2 — Clarifying questions
`AskUserQuestion`, bundle 2-4. Typical: feature flag needed (`feature_flags`)? deadline?
`min_sdk`/`flutter_version` floor? backend/schema change? design sign-off? localization keys? success/error/
empty/loading states (`state_type` shape)? Semantics labels needed for widget/integration tests?
Skip anything already answered by the ticket or scout.

## Phase 3 — Scope assessment
| Signal | Action |
|---|---|
| Spans 3+ independent subsystems | **Decompose** — split into sub-topics |
| 1-line copy/colour change | **Skip brainstorm** — point at implementation |
| Touches `high_rigor_domains` | **HIGH-RIGOR** — extra scrutiny, security Qs, rollback discussion |
| Greenfield, no prior art | First feature **establishes** the pattern — relax DRY, focus on getting the foundation right |

## Phase 4 — Propose approaches (2-3)

```markdown
### Approach N: <name>
**Summary**: <1-2 sentences>
**How it works here**:
- Widgets / UI:
- State ({state_type}):
- Navigation ({navigation}):
- Dependencies / DI ({di}):
- Networking ({networking}, if any):
- Localization + accessibility (Semantics) touched:
- Tests (unit / widget / golden / integration):
**Pros / Cons** · **Effort** S/M/L · **Risk** LOW/MED/HIGH · **Maintainability**
**Recommended? Yes/No — because <reason>**
```

### Brutal-honesty checklist
- [ ] Matches existing patterns, or introduces a new abstraction / package?
- [ ] Hidden costs (build time, app size, runtime jank, CI, extra dependency)?
- [ ] Migration burden on unrelated screens?
- [ ] Is the user solving the right problem?
- [ ] Violates any `rules_file` rule?

If the idea is wrong / over-engineered / regression-prone — **say so directly.**

## Phase 5 — Decision & report
Path: `{reports_dir}/brainstorm-{YYMMDD-HHMM}-{TICKET|slug}.md`
(`{YYMMDD-HHMM}` from `bash -c 'date +%y%m%d-%H%M'`). Sacrifice grammar for concision.

```markdown
# Brainstorm: <topic>
**Ticket**: <id|n/a>  **Date**: <YYYY-MM-DD>  **Branch**: <current>
## Problem Statement
## Constraints & Requirements
## Approaches Evaluated  (1 / 2 / 3 — summary, pros, cons, effort/risk)
## Recommendation  (chosen + rationale)
## Implementation Considerations
- state shape / DI wiring / navigation / networking / localization + a11y
- feature flag (if feature_flags): <yes/no, name, default-off?>
- migration / rollback / testing strategy
## Risks & Mitigations  | Risk | Likelihood | Mitigation |
## Success Metrics
## Out of Scope
## Open Questions  (owner: PM / BE / Design)
```

## Phase 6 — Hand-off (menu, do NOT auto-invoke)
```
Brainstorm complete → {reports_dir}/<file>.md
- Create plan      → flutter-plan <ticket-or-topic>
- More research    → flutter-research <sub-topic>
- Deep reasoning   → flutter-sequential-thinking <problem>
- Stop here        → keep report as decision record
```
