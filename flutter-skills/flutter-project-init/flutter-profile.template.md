---
# flutter-profile.md — single source of project-specific facts for the flutter-* skill suite.
# Every flutter-* skill reads this FIRST. Copy this file to `.claude/flutter-profile.md`
# at the project root and fill it in (or let `flutter-project-init` generate it).
#
# Rule of thumb: if a value differs between two Flutter apps, it belongs HERE, not in a skill body.

app_name:                 # e.g. Acme
state: existing           # existing | greenfield

# ── Repo layout ────────────────────────────────────────────────
pubspec: pubspec.yaml     # path to the app's pubspec.yaml (root, or packages/app/ in a monorepo)
monorepo:                 # melos | none
source_roots: []          # e.g. [lib/, packages/*/lib]
test_roots: []            # e.g. [test/, integration_test/]
generated_paths: []       # NEVER edit — e.g. [*.g.dart, *.freezed.dart, *.gr.dart, *.config.dart, *.mocks.dart, l10n/]
docs_root:                # docs/ | none
plans_dir:                # docs/plans | .claude/plans
reports_dir:              # docs/reports | .claude/reports
rules_file:               # CLAUDE.md | docs/architecture.md | none  (always/never rules)

# ── Architecture ───────────────────────────────────────────────
architecture:             # Clean-Architecture | feature-first | layer-first | MVVM | other
state_type:               # BLoC | Cubit | Riverpod | Provider/ChangeNotifier | GetX | setState | freezed-union | none
di:                       # get_it | injectable | riverpod-providers | provider | manual-constructor
navigation:               # go_router | auto_route | Navigator-1.0 | Beamer
min_sdk:                  # Dart SDK constraint, e.g. ">=3.5.0 <4.0.0" — gates version-specific specialist guidance (records, patterns, sealed classes)
flutter_version:          # e.g. 3.24 — gates version-specific guidance

# ── Integrations (set to `none` if unused — skills skip the related checks) ──
networking:               # dio | http | chopper | retrofit | graphql_flutter | ferry | none
localization:             # intl+gen-l10n(.arb) | slang | easy_localization | none
accessibility:            # Semantics-label/Key convention name | none
feature_flags:            # Firebase-RemoteConfig | LaunchDarkly | flavors | none
crash_reporting:          # firebase_crashlytics | sentry_flutter | none
ticket_system:            # Jira | GitHub | Linear | none
ticket_pattern:           # regex, e.g. ABC-\d+  (used to detect a ticket id in args/branch)
ticket_fetch:             # MCP tool or CLI to fetch a ticket | none

# ── Workflow / PR (used by flutter-resolve + flutter-execute --team) ───
default_base_branch: main # branch-from point + PR target; overridable with --base
pr_tool:                  # gh | none   (none → flutter-resolve prints manual PR steps)

# ── Verification ───────────────────────────────────────────────
# Whatever this project actually runs to prove a change is correct.
# Free-form: flutter test, flutter analyze, very_good test, melos run test, dart test, ./scripts/foo.sh …
# Empty/unset → skills fall back to ANALYZE/BUILD-ONLY and say so explicitly in the report.
verify_command: |

# ── Rigor ──────────────────────────────────────────────────────
# Domains that force adversarial review + correctness audit every time.
high_rigor_domains: [checkout, payment, auth, PII, money]

# ── Specialist skills (optional add-ons) ───────────────────────
# ON BY DEFAULT: any installed flutter-*-expert is auto-used — flutter-code-review routes to it per change
# type, flutter-execute applies it while writing. This field is an OPTIONAL OVERRIDE.
specialists:              # unset = auto-use all installed · [flutter-widget-expert, …] = restrict · none = off
---

# Project Notes (free text)

Anything a skill should know that doesn't fit a field above — naming conventions,
forbidden patterns, "always do X / never do Y" rules specific to this app.
