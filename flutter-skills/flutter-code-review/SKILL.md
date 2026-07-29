---
name: flutter-code-review
description: "Adversarial, multi-lens code review for a Flutter project (Dart / widgets / state management); routes slices of the diff to installed flutter-* specialist skills. Use for PR, commit, pending-diff, or whole-codebase review."
argument-hint: "[#PR | COMMIT | --pending | codebase]"
effort: xhigh
---

# Code Review — Flutter

Adversarial, profile-driven review for Flutter/Dart. **Precision over recall** — a focused list
of real, high-confidence findings beats a wall of nits.

**Principles:** YAGNI + KISS + DRY. Technical correctness over social comfort. **Honest, brutal, concise.**

## Step 0 — Load profile
Read `.claude/flutter-profile.md`: `architecture`, `state_type`, `di`, `navigation`, `networking`,
`localization`, `accessibility`, `crash_reporting`, `verify_command`, `high_rigor_domains`,
`generated_paths`, **`specialists`**, `rules_file`, `test_roots`, `plans_dir`. If missing, read the main
checkout's copy — `$(git rev-parse --path-format=absolute --git-common-dir)/../.claude/flutter-profile.md`
(the profile is usually gitignored, so worktrees don't inherit it). Still missing → run `flutter-project-init`.

---

## Phase 0 — Resolve scope (do this FIRST; everything keys off it)

Build ONE canonical `scope` and **print a banner before any finding** — if the scope is wrong, every
finding is suspect.

| Input | Mode | Diff |
|---|---|---|
| `#123` / PR URL | PR | `gh pr diff 123` (base via `gh pr view --json baseRefName`) |
| `abc1234` (7+ hex) | Commit | `git show <sha>` |
| `--pending` | Pending | `git diff` + `git diff --cached` |
| *(no args, recent edits)* | Default | files edited this session |
| `codebase` | Codebase | full scan — adapt the stages per [codebase-mode.md](codebase-mode.md) |

- **Base-branch fallback:** `gh pr view --json baseRefName` → `git rev-parse origin/HEAD` → `origin/main`|`origin/master`.
- **Buckets:** `modified` (review in full, all severities) · `tests-for-modified` (coverage → main report) · `related/context` (read for context only) · `deleted` (spec reasoning only).
- **Exclude always:** `generated_paths`, `.dart_tool/`, `build/`, `.pub-cache/`, `*.g.dart`/`*.freezed.dart`/`*.gr.dart`/`*.config.dart`/`*.mocks.dart`.
- **Adjacent quarantine:** issues in context-only files go in an "Adjacent observations" section and **do NOT count** toward the verdict.
- **Banner:** `Scope: PR #123 · base: main · modified: 7 · tests: 2 · HIGH-RIGOR: yes`.
- **Trivial:** tiny diff (≤2 files, ≤30 lines, not HIGH-RIGOR) → review locally: skip the parallel fan-out and Stage 3.

**HIGH-RIGOR:** diff touches any domain listed in the profile's `high_rigor_domains` — never trivial; log `[HIGH-RIGOR]`, and alongside Stage 3 run the **money/PII correctness audit** from `flutter-sequential-thinking`. (The project defines its own domains via the profile; the skill assumes none.)

---

## Stage 1 — Spec compliance (skip if no plan/spec)
Read the plan (`{plans_dir}/{slug}/plan.md`) and/or the PR/issue (`gh pr view --json title,body,closingIssuesReferences`). Emit a **requirement-coverage table**:

| Requirement / AC | Status | Where |
|---|---|---|
| <criterion> | ✅ met / ⚠️ partial / ❌ missing | `file:line` |

Flag **scope creep** (unrelated refactors bundled in) and **missing work** (`TODO`, `throw UnimplementedError()`, empty bodies, stubbed fakes). No spec available → mark "not assessed" (don't invent one). **Stage 1 FAIL → stop, report, ask.**

---

## Stage 2 — Multi-lens review (run the lenses in parallel)

### 2.0 — Specialist routing (FIRST — this is the point)
For each domain the diff touches, route it to the matching specialist below **if that specialist skill
is installed** in this project — **on by default, no profile entry needed**. Spawn a **read-only
`general-purpose` Agent** that loads the specialist's `SKILL.md` and reviews **only its slice** of the
diff; the general lens (2.1) then skips that domain. The profile's `specialists:` is an **optional
override**: if set, restrict routing to that list; `specialists: none` turns routing off.

| Diff signal | Route to (if installed) | Reviews |
|---|---|---|
| `Widget`, `build(BuildContext)`, `StatefulWidget`/`setState`, `InheritedWidget`, layout/`const` widgets | **flutter-widget-expert** | widget composition, state ownership, rebuild perf, navigation, a11y, Material/Cupertino |
| `async`/`await`, `Future`, `Stream`/`StreamController`, `Isolate`/`compute`, `StreamSubscription`, `Completer` | **flutter-async-expert** | unawaited futures, stream/subscription leaks, isolate boundaries, cancellation, UI-isolate blocking |
| `test(`, `testWidgets(`, `expect(`, `group(`, `WidgetTester`, `mocktail`/`mockito`, files under `test_roots` | **flutter-testing-expert** | flutter_test/widget/golden/integration idioms, coverage, flakiness, pump vs pumpAndSettle |
| new/changed classes, abstract interfaces, services, blocs/notifiers; DI / provider wiring; large classes; cross-layer changes | **flutter-solid-expert** | SOLID (SRP/OCP/LSP/ISP/DIP), decoupling (Decorator/Composite/Adapter), constructor DI, framework isolation |
| bug-fix diff, or behavior changes landing with their tests in one diff | **flutter-tdd** | TDD discipline: reproduction test exists for the bug, tests exercise public behavior not internals, no test-after backfill smells |
| *(future specialists, e.g. networking/persistence)* | **flutter-`<domain>`-expert** | its domain |

Specialist agent prompt:
```
Load .claude/skills/<specialist>/SKILL.md and check this slice against every section of it — report
violations found, and end with one line naming the sections that came up clean, so every section is
accounted for. Review ONLY the <domain> aspects of this diff: <the relevant hunks>.
Skip what a general reviewer covers (naming, layering, localization).
Output each finding as { severity, file:line, problem, fix, confidence: high|med|low }. Be specific; cite the rule.
```
No specialists installed / none match → skip 2.0 (the general lens still covers the floor).

### 2.1 — General Flutter lens (always; the floor when a specialist isn't installed)
Spawn a `general-purpose` Agent (fill `{…}` from profile). **Authoritative: `rules_file` + Dart's Effective Dart guidelines** — read them.

```
Review this Flutter/Dart diff with senior rigor. Flag { severity, file:line, problem, fix, confidence }.

ASYNC & LIFECYCLE (highest-value bugs — unless flutter-async-expert handled it):
- BuildContext used across an await without a `mounted`/`context.mounted` guard; setState after dispose.
- unawaited futures that race state; forgotten `await`; Future.then chains swallowing errors.
- StreamSubscription / AnimationController / TextEditingController / FocusNode not disposed; listeners not removed.
- heavy sync work on the UI isolate (parse/crypto/image decode) → should be compute()/Isolate.run.
MEMORY & LIFETIME: leaked subscriptions/controllers/timers, unbounded caches, global singletons holding state.
SECURITY & PRIVACY: token leakage; flutter_secure_storage (not SharedPreferences) for credentials; missing HTTPS/cert pinning; PII in logs/analytics/{crash_reporting}/print/debugPrint.
MONEY (if a money/finance domain is configured in high_rigor_domains): integer minor units or a Decimal package, never double; rounding; sign errors; currency parsing.
ACCESSIBILITY: Semantics labels/order, text scaling (MediaQuery.textScaler), contrast, tap-target size (unless flutter-widget-expert handled it).
HYGIENE: comments WHY-only; no dead/commented code; no back-compat shims for code this diff removed; no stray print/debugPrint.

PROFILE-GATED (run only those that apply):
- state_type != none → uses {state_type}; transition holes (loading→loaded/empty/error) covered.
- di != manual-constructor → injected via {di}; no service-locator reach-in from widgets.
- navigation centralized → routes via {navigation}; widgets don't Navigator.push directly if the convention forbids it.
- networking != none → DTO→domain mapping at the boundary; UI doesn't import generated/network types.
- localization != none → no hardcoded user-facing strings; new keys added; generated l10n untouched.
- accessibility != none → tested UI exposes a Semantics label / stable Key per {accessibility}.
- Always: never edit generated_paths.
```

### 2.2 — Architecture lens (profile-gated)
Check against the project's `architecture`: **dependency direction** (Widget → Bloc/Notifier → UseCase/Repo; Model independent), **single source of truth** (no widget `setState` mirroring bloc/notifier state), **God bloc / massive notifier** (extract UseCases), **presentation isolation** (no `flutter/material` import or `BuildContext` in a bloc/repository; navigation intents as values/events), **stale-async overwrite / missing cancellation**, business logic living in widgets. Align to the *existing* pattern — don't propose an architecture switch for a small change. (When `flutter-solid-expert` is installed, 2.0 already covered the SOLID/decoupling depth; this lens then focuses on pattern-fit + the project's `architecture`, not re-deriving the principles.)

### 2.3 — Reuse & simplification lens
Flag both directions: **over-engineering** (duplicated logic that an existing helper covers; redundant/derived state stored; parameter sprawl; needless abstraction/indirection; stringly-typed where an enum/sealed class exists) **and over-simplification** (distinct concerns collapsed into one unclear unit). Only findings that materially improve maintainability/correctness/cost — never churn for style. (If the repo has a `simplify` skill, this lens can defer the *applying* to it; here it only flags.)

---

## Stage 3 — Adversarial red-team
Try to **break** the change across four lenses (parallel agents for large diffs). Define regression relative to the **stated intent** (what should change vs what must stay unchanged):

1. **Intent & regression** — behavior outside stated scope; broken edge/fallback paths; contract drift between callers & callees; adjacent flows that should have changed together but didn't.
2. **Security & privacy** — authn/authz gaps, unsafe input, injection, secret/token exposure, risky defaults, trust of unverified data, PII leakage.
3. **Performance & reliability** — duplicate work / redundant I/O; new work on hot paths (build/layout/frame/request); rebuild storms (missing `const`, whole-tree `setState`); leaks, retry storms, subscription drift; ordering/race/cancellation brittleness; image decode / JSON parse on the UI isolate.
4. **Contracts & coverage** — API/schema/type/flag mismatches; migration & back-compat fallout; missing/weak tests for changed behavior; **detectability** (would a future regression even be observable — logs/metrics/assertions?).

Each finding: `{ severity, file:line, problem, exploit/scenario, fix, confidence }`.

---

## Synthesis (precision over recall)
Treat all lens output as raw input, then:
- **Dedup** across lenses; **drop** weak/speculative/style-only items and anything that conflicts with the stated intent — each dropped finding gets a one-line drop reason.
- "May be wrong but intent unclear" → a **question**, not a finding.
- **Normalize:** `{ file:line, category (spec|async|widget|testing|architecture|security|perf|contracts|simplification|hygiene), severity, why it matters, fix, confidence }`.
- **Order:** high-severity high-confidence first. If nothing material, **say so** — don't manufacture feedback.

## Agent-loop feedback (self-improving rules)
Group findings by rule. A rule firing **≥2×** across the diff = a recurring pattern → propose a one-line **directive** for the project's `rules_file` (e.g. "Always guard BuildContext with `context.mounted` after an await"). Especially useful for AI-generated diffs — it turns repeated misses into standing rules. (Propose only; don't edit `rules_file` without approval.)

## Verification gate
**Iron law: no "review passed" without fresh verification.** Tests/build → run `{verify_command}` (unset → analyze/build-only, say so). Bug-fix → the original reproducer no longer reproduces. Stop if you catch yourself thinking "should pass" — run it.

## Severity model
- **Critical** — crash · setState-after-dispose · state corruption · money precision loss (double) · credential leak · PII in logs → **must fix before merge** (verdict cannot be APPROVED while one remains).
- **Important** — state-transition hole · layer/dependency violation · leaked subscription/controller · missing test for changed behavior · missing Semantics label on tested UI · missing localization key → **fix before merge or file a ticket**.
- **Nit** — style, naming, API cleanliness → author's call.

## Report format
```markdown
Scope: <banner>
# Code Review — <mode>   HIGH-RIGOR: yes/no   Verdict: APPROVED / CHANGES REQUESTED / BLOCKED

## Summary   Critical: N · Important: N · Nit: N
## Stage 1 — Spec       <coverage table> · scope-creep: <…>
## Findings (by severity)   [Sev] category — file:line — problem → fix (confidence)
   - Specialist findings grouped under their sub-heading (flutter-widget-expert / …)
## Agent-loop feedback   <recurring pattern → proposed rules_file directive>  (or none)
## Adjacent observations (out of scope)
## Critical open items · Recommended follow-ups
```

## Constraints
- **Read-only review** — find and recommend; do NOT apply fixes (that's `flutter-execute` / a `simplify` skill).
- **DO NOT** rely on memory — read the actual diff + `rules_file`. Specialists win on their domain.
