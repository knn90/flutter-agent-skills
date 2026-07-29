---
name: flutter-project-init
description: "One-time bootstrap for the flutter-* skill suite. Creates .claude/flutter-profile.md — the single source of project-specific facts every other flutter-* skill reads. Use before first running any flutter-* skill on a project, or whenever the profile is missing or stale."
argument-hint: "[--greenfield | --detect]"
---

# Project Init — flutter-* suite bootstrap

Produce `.claude/flutter-profile.md`: the one file that makes the generic `flutter-*` skills
concrete for THIS project. This skill only writes the profile (+ optional docs scaffold) —
never invoke implementation skills from it.

The profile holds **facts, not opinions**: if a value differs between two Flutter apps it
belongs in the profile; everything else stays in the skill bodies. Keep it minimal.

---

## Two modes

| Repo state | Mode | Profile is… |
|---|---|---|
| Empty / no source files yet | **greenfield** (prescriptive) | a record of decisions you make up front |
| Has `pubspec.yaml` + `lib/*.dart` source | **detect** (descriptive) | a record of reality read from the code |

Auto-detect the mode:
```
Glob **/pubspec.yaml
Glob lib/**/*.dart, **/*.dart   (any real source?)
```
- Source present → `--detect`. Empty/scaffold only → `--greenfield`.
- `$ARGUMENTS` may force `--greenfield` / `--detect`.

If a profile already exists, show it and ask (via `AskUserQuestion`): keep / refresh / overwrite.

---

## Mode A — Detect (existing app)

Read the codebase, fill the profile from evidence. **Verify every value — never guess.**

| Profile field | How to detect |
|---|---|
| `pubspec` / `monorepo` | the app's `pubspec.yaml`; `melos.yaml` or `packages/*/pubspec.yaml` → monorepo (melos) |
| `source_roots` / `test_roots` | top-level dirs containing `*.dart` (`lib/`, `packages/*/lib`) / `test/`, `integration_test/` |
| `architecture` | folder shape: `features/*/{data,domain,presentation}`→Clean/feature-first · `lib/{data,domain,presentation}`→layer-first · else grep imports below |
| `state_type` | pubspec deps: `flutter_bloc`→BLoC/Cubit · `riverpod`/`hooks_riverpod`→Riverpod · `provider`→Provider/ChangeNotifier · `get`→GetX · none→setState |
| `di` | `get_it`/`injectable` · riverpod providers · `provider` · else manual-constructor |
| `navigation` | `go_router` · `auto_route` · `beamer` · else Navigator-1.0 (`Navigator.push`) |
| `networking` | `dio`/`http`/`chopper`/`retrofit`→REST · `graphql_flutter`/`ferry` + `*.graphql`→GraphQL · none |
| `localization` | `flutter_localizations`+`*.arb`+`l10n.yaml`→intl/gen-l10n · `slang` · `easy_localization` · none |
| `accessibility` | grep for a shared `Semantics` label / `Key` convention module, else none |
| `feature_flags` / `crash_reporting` | pubspec: `firebase_remote_config`/`launchdarkly`, `firebase_crashlytics`/`sentry_flutter` |
| `generated_paths` | `*.g.dart`, `*.freezed.dart`, `*.gr.dart`, `*.config.dart`, `*.mocks.dart`, generated `l10n/` (build_runner output — never edit) |
| `verify_command` | existing `scripts/*verify*`, `Makefile`/`melos run test`, CI workflow, else propose `flutter test` (+ `flutter analyze`) |
| `rules_file` | `CLAUDE.md` else `docs/architecture.md` else none |
| `ticket_system`/`ticket_pattern` | branch names + `git log` (e.g. `ABC-123`), or ask |
| `high_rigor_domains` | grep for `checkout`/`payment`/`auth`/`profile`; default `[auth, PII]` if none |
| `default_base_branch` | `git symbolic-ref refs/remotes/origin/HEAD` (repo default), else `main` |
| `pr_tool` | `gh` if the GitHub CLI is installed + authed, else `none` |
| `min_sdk` / `flutter_version` | `environment:` in pubspec.yaml (`sdk:`/`flutter:`); else `flutter --version` |
| everything else (`app_name`, `docs_root`/`plans_dir`/`reports_dir`, …) | pubspec `name:` / obvious repo layout |

Use `Explore`/`Grep`/`Glob` (or invoke `flutter-scout` if the repo is large).
Done = every template field filled or explicitly `none` — no silent blanks. Confirm the
detected `architecture`, `verify_command`, and `high_rigor_domains` with the user before
writing — these three drive the most downstream behaviour.

---

## Mode B — Greenfield (new app)

Nothing to detect. **Decide** the conventions via `AskUserQuestion`, then record them.

Ask (bundle into 2-4 `AskUserQuestion` groups):

1. **State management + architecture** — BLoC/Cubit · Riverpod · Provider/ChangeNotifier ·
   plain setState; feature-first vs layer-first foldering.
   For each, state the trade-off honestly (testability, boilerplate, team familiarity,
   ecosystem maturity). If the user is unsure, recommend the boring proven default for their
   stated team size and push back on novelty. *(This is `flutter-brainstorm` applied to
   foundations — borrow its brutal-honesty stance.)*
2. **Navigation + min SDK** — go_router / auto_route / Navigator 1.0; Dart/Flutter SDK constraint.
3. **Networking** — dio/http REST · GraphQL(graphql_flutter/ferry) · none yet.
4. **DI · localization** — pick or defer (`none` is valid early).
5. **Verify command** — default suggestion `flutter analyze` (no tests yet) or
   `flutter test` once a test dir exists. May be left **empty** → analyze/build-only.
6. **Ticket system** + pattern, or `none`.
7. **HIGH-RIGOR domains** — which of checkout/payment/auth/PII this app will have.

Do **NOT** generate a `verify.sh` or any wrapper script. The verify mechanism is just the
`verify_command` string — keep it as configuration, not a committed artifact.

Optionally (ask first) create the docs scaffold the other skills expect:
`{docs_root}/plans/` and `{docs_root}/reports/`, and a starter `rules_file` (CLAUDE.md)
capturing the always/never rules implied by the chosen architecture.

---

## Output

Write `.claude/flutter-profile.md` from `flutter-profile.template.md` (sibling of this SKILL.md
in the skill's base directory), filled in. If the sibling is missing (untracked skill
files don't appear in git worktrees), read the main checkout's copy — same relative path
under `$(git rev-parse --path-format=absolute --git-common-dir)/..`. Then:

```
✓ Profile written: .claude/flutter-profile.md
- Mode: detect | greenfield
- Architecture: <…>   State: <…>   Networking: <…>   Verify: <command or "analyze-only (unset)">
- HIGH-RIGOR domains: <…>
- Specialists available: <installed flutter-*-expert skills, or none>

Next: the flutter-* skills are now live for this project.
- Find code            → flutter-scout
- Plan a feature       → wayfinder
- Implement            → flutter-execute
- Resolve end-to-end   → flutter-resolve <ticket | "description">  (wayfinder→scout→execute→review→PR)
```
