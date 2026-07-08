---
name: flutter-research
description: "Strategic technical research for a Flutter project (Dart / widgets / state management / networking / async / security / accessibility). Use for package evaluation, framework pattern questions, migration questions, CVE checks. Outputs a sourced report."
argument-hint: "[topic | TICKET-ID]"
---

# Research — Flutter

Strategic research producing an actionable, sourced report. **Honest, brutal, concise.**
**Bias toward what already works in the repo.** Don't recommend a package the project
doesn't use unless you can show the existing pattern is genuinely insufficient.

## Step 0 — Load profile

Read `.claude/flutter-profile.md`: `min_sdk`/`flutter_version`, `architecture`, `state_type`,
`networking`, `reports_dir`, `ticket_system`/`ticket_pattern`/`ticket_fetch`. If missing, read the main
checkout's copy — `$(git rev-parse --path-format=absolute --git-common-dir)/../.claude/flutter-profile.md`
(the profile is usually gitignored, so worktrees don't inherit it). Still missing → run
`flutter-project-init`.

## When NOT to Use

| Case | Use instead |
|---|---|
| Pattern already exists in repo | `flutter-scout` |
| API doc for a known package | `WebFetch` against the official URL (pub.dev / api.flutter.dev) |
| Trade-off discussion of approaches | `flutter-brainstorm` |
| Step-by-step debug reasoning | `flutter-sequential-thinking` |

---

## Argument Parsing

- **TICKET-ID** (matches `ticket_pattern`) → fetch via `ticket_fetch` for context.
- **Free-form topic** → research as-is. If missing, ask via `AskUserQuestion`.

---

## Process

### Phase 1 — Scope
- Key terms; recency window (< 12 months unless historical); in/out of scope.

### Phase 2 — Ground in the codebase (FIRST)
Before any web search:
- Is there already a working pattern? → `flutter-scout` / `Grep`.
- What's pinned? → `pubspec.yaml`, `pubspec.lock`.
- What's documented? → `rules_file` + `docs_root`.
- **If the codebase already solves it → say so and stop.** That's a successful outcome.

### Phase 3 — External research (≤ 5 search calls)
Preferred order: Flutter/Dart official docs (flutter.dev, dart.dev, api.flutter.dev) →
package docs on pub.dev (score/popularity/maintenance) → Flutter GitHub issues/PRs →
reputable Flutter eng sources (Flutter/Dart team talks, Very Good Ventures, Reso Coder,
Code with Andrea, Robert Brunhage) → recent Stack Overflow / GitHub Discussions.

Query patterns: `<topic> flutter <flutter_version>`, `<topic> dart async stream`,
`<topic> <state-mgmt-lib>`, `<topic> <networking-lib> pub.dev`, `<topic> CVE <year>`.

### Phase 4 — Cross-reference & validate
- 2+ independent sources per claim; flag conflicts.
- Reject APIs above `min_sdk`/`flutter_version` unless the user agrees to bump the SDK constraint.
- Discard anything older than the framework/package major version in use; check pub.dev for latest-stable compatibility.

### Phase 5 — Report
Save to `{reports_dir}/research-{YYMMDD-HHMM}-{TICKET|slug}.md`.
`{YYMMDD-HHMM}` MUST come from `bash -c 'date +%y%m%d-%H%M'`, not model memory.
Create `{reports_dir}` if absent.

---

## Report Template

```markdown
# Research: <topic>

**Ticket**: <id | n/a>   **Date**: <YYYY-MM-DD>   **Branch**: <current>
**Versions**: Flutter <flutter_version>, Dart SDK <min_sdk>, <networking-lib <Z> if any>  ← from repo

## Executive Summary
<3-5 bullets — findings + recommendation>

## Codebase Context
- Existing pattern: <what's there> (path:line)
- Why current approach falls short: <reason | "n/a — greenfield">

## Findings
### Best Practices (current consensus)
### Security / Privacy  (secure storage, platform channels, certificate pinning, permissions — flag affected pinned versions)
### Performance Insights  (numbers > vibes — jank, frame budget, rebuild counts, isolate offloading)
### Comparative Analysis (if multiple options)
| Option | Effort | Risk | Maintenance | Fit |

## Recommendation
**Chosen**: …   **Rationale**: …

## Implementation Sketch
- Affected packages / state shape / DI / networking / localization impact

## Sources
1. [Title](url) — checked YYYY-MM-DD

## Open Questions
- <unresolved — owner: PM / BE / Design>
```

## Constraints

- **DO NOT** implement — research only.
- **MUST** cite sources with check dates.
- **Sacrifice grammar for concision** — bullet > paragraph.
