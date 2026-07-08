---
name: flutter-testing-expert
description: "Expert Flutter testing guidance — unit / widget / golden / integration layers, flutter_test & package:test mechanics, test doubles/DI (mocktail/mockito), bloc_test, async testing. Fires when work touches tests (test(, testWidgets(, expect(, WidgetTester, mocktail, files under test/)."
---

# Flutter Testing Expert

> **Generated skill** — original wording, consolidated by `flutter-skill-consolidate`; provenance,
> sources, and licenses live in `SOURCES.yaml`. Don't hand-edit — change `SOURCES.yaml` and re-consolidate.

**Source of truth:** **Flutter/Dart official docs** — [Testing Flutter apps](https://docs.flutter.dev/testing),
the [`flutter_test`](https://api.flutter.dev/flutter/flutter_test/flutter_test-library.html) /
[`package:test`](https://pub.dev/packages/test) APIs, and the `integration_test` package — win on any
conflict. Community sources supply idioms and mistake patterns, not semantics. Package-specific rules
(mocktail/mockito/bloc_test/golden tooling) follow each package's official docs, gated to the versions
pinned in `pubspec.lock`.
Profile: `.claude/flutter-profile.md` — worktrees don't inherit it; fall back to
`$(git rev-parse --path-format=absolute --git-common-dir)/../.claude/flutter-profile.md`.

---

## 1. The layers — pick the cheapest that proves the behavior (test pyramid)
- **Unit** (`test/`, `package:test`) — pure Dart logic: models, use-cases, repositories, blocs/notifiers, mappers, extensions. Fast, no `WidgetTester`. **The base of the pyramid — most tests live here.** Keep logic out of widgets so it's reachable by a unit test.
- **Widget** (`testWidgets` + `WidgetTester`, `flutter_test`) — a single widget/screen's rendering + interaction in a headless test environment. Covers wiring, conditional UI, tap→state→rebuild. Mid-tier: more than unit, far cheaper than integration.
- **Golden** (`matchesGoldenFile`) — pixel-level snapshot of a widget's rendered appearance. Use for design-critical/regression-prone UI; **not** for logic. Needs determinism (§5).
- **Integration** (`integration_test` package, on a real device/emulator) — full-app end-to-end flows across screens, real (or near-real) plugins. **Top of the pyramid — few, high-value flows** (login, checkout). `patrol` extends it for native dialogs/permissions.
- Rule: don't reach for a widget test where a unit test proves it, or integration where a widget test does. Each layer up is slower and flakier.

## 2. Basics — structure & matchers
- `test('description', () { … })`, `group('feature', () { … })`, `setUp`/`tearDown` (+ `setUpAll`/`tearDownAll` — use the All variants sparingly; per-test setUp keeps isolation). Each `test` should be independent — **no shared mutable top-level state**, no ordering assumptions.
- **`expect(actual, matcher)`** — prefer real matchers over booleans: `equals`, `isA<T>()`, `isNull`, `throwsA(isA<FooException>())`, `contains`, `hasLength`, `closeTo(x, epsilon)` for doubles, `predicate((v) => …, 'desc')`. A precise matcher gives a readable failure; `expect(x == y, isTrue)` gives none.
- Async assertions: `expectLater(future, completes / throwsA(...))`, `expectLater(stream, emitsInOrder([a, b, emitsDone]))`, `emits`, `neverEmits`, `emitsError`. Mark the test `async` and `await` the real work — never `sleep`.
- Arrange–Act–Assert; one behavior per test; name the test after the behavior (`emits [Loading, Loaded] when fetch succeeds`), not the method.

## 3. Widget tests — `WidgetTester`
- `await tester.pumpWidget(widget)` mounts the tree. Wrap the unit under test in the **minimum** needed context: `MaterialApp`/`CupertinoApp` (for `Directionality`, theme, `Navigator`, `MediaQuery`), or a bare `Directionality`+`MediaQuery` for a leaf. Inject fakes via the project's `di` (a `Provider`/`ProviderScope`/`get_it` override), not real services.
- **`pump` vs `pumpAndSettle`:** `await tester.pump()` advances **one frame**; `pump(Duration)` advances virtual time by that much; **`pumpAndSettle()` pumps until no frames are scheduled** — great after a tap that animates, but it **hangs on an infinite animation** (a looping spinner/`repeat()`). For a perpetual animation, use explicit `pump(duration)` steps, not `pumpAndSettle`.
- **Finders:** prefer `find.byKey(const ValueKey('…'))` and `find.text('…')` / `find.bySemanticsLabel('…')` (accessible + stable) over `find.byType` (brittle to refactors) and never over widget-tree indices. Interact: `tester.tap`, `enterText`, `drag`, `fling`, `longPress` — **each is followed by a `pump`** to apply the resulting rebuild.
- Assert with `expect(find.text('…'), findsOneWidget / findsNothing / findsNWidgets(n) / findsWidgets)`. Read widget state via `tester.widget<T>(finder)`.
- **Pump an image/network-free tree:** real `Image.network`/HTTP fails in tests (mocked `HttpClient` returns 400) — inject fake image providers or use `mockNetworkImagesFor`. Set a fixed surface size with `tester.view.physicalSize`/`devicePixelRatio` and reset in `tearDown`.

## 4. Test doubles & DI
- **Prefer `mocktail`** (no codegen — `class MockRepo extends Mock implements Repo {}`) for most projects; **`mockito`** when the team already uses it (needs `@GenerateMocks([Repo])` + `build_runner`). A **fake** (hand-written working shortcut, `extends Fake`) beats a mock when you need behavior, not interaction assertions.
- **Prefer state/output verification over interaction verification** — `verify(() => repo.save(any())).called(1)` is brittle; assert the resulting state/return where you can. Reserve `verify`/`verifyNever` for genuine contracts (analytics fired, network NOT called).
- Stub with `when(() => repo.fetch()).thenAnswer((_) async => tItems)` (use `thenAnswer` for futures/streams, `thenReturn` for sync). **Register fallback values** for custom types used with `any()` via `registerFallbackValue` in `setUpAll` (mocktail).
- **Inject every hidden dependency** — clock/`DateTime.now`, random, `http`/`dio` client, `SharedPreferences` (`SharedPreferences.setMockInitialValues({})`), platform channels (`TestDefaultBinaryMessengerBinding` handlers) — never hit live network/disk/clock. **Deterministic data only.** Build fixtures with factory helpers (`Item fixture({int id = 1})`) so tests set only what matters.

## 5. Golden tests
- `await expectLater(find.byType(MyCard), matchesGoldenFile('goldens/my_card.png'))`. Regenerate with `flutter test --update-goldens`; **review the PNG diff in review** — a golden is only as trustworthy as the eyes that approved it.
- **Determinism is everything** — goldens flake across platforms/font rendering. Load real fonts (or use `golden_toolkit`/`alchemist` which bundle Ahem/roboto and pump fonts), pin surface size + `devicePixelRatio`, disable animations, freeze clock/random. Run goldens in **one canonical environment (CI)**; consider a separate CI job so local font differences don't churn them.
- Tools: **`alchemist`** (CI/local split, no font hassles) or **`golden_toolkit`** (multi-device, `multiScreenGolden`). Keep goldens for design-stable, high-value screens; they're a maintenance cost on churning UI.

## 6. Async & time
- **Never use real delays.** Control virtual time with **`fakeAsync`** (`package:fake_async`) for pure Dart (`fakeAsync((async) { … async.elapse(Duration(seconds: 5)); })`), and **`tester.pump(duration)`** in widget tests. For real async that must run (a real `Future` off the test's fake clock), wrap in **`await tester.runAsync(() async => …)`**.
- Streams: `expectLater(bloc.stream, emitsInOrder([...]))`; drain a `StreamController` you own and `close()` it in `tearDown`. Timers must be cancelled — assert `async.pendingTimers` is empty (the `testWidgets` binding fails on a leaked timer; bare `fakeAsync` does not).
- Bridge a callback into an awaitable with a `Completer` in the test. Test error/cancellation paths explicitly (`throwsA`, assert the source stopped).

## 7. State-management testing
- **Bloc/Cubit** → **`bloc_test`**: `blocTest<CartBloc, CartState>('…', build: () => bloc, act: (b) => b.add(Load()), expect: () => [Loading(), Loaded(items)])`. Seed with `seed:`, stub deps in `build:`, assert side effects in `verify:`. Assert **the sequence of emitted states**, and make states `Equatable`/`freezed` so `expect` compares by value.
- **Riverpod** → build a `ProviderContainer(overrides: [...])` (or `ProviderScope` in widget tests), `container.read`/`listen`, `addTearDown(container.dispose)`. **Provider/ChangeNotifier** → pump under a `ChangeNotifierProvider` override, or test the notifier directly by listening.
- Test the **state holder** directly at unit level; use widget tests only for the widget↔state wiring, not to re-prove the logic.

## 8. Integration & E2E
- `integration_test` package + `IntegrationTestWidgetsFlutterBinding.ensureInitialized()`; drive with the same `WidgetTester` API but against the real app on a device/emulator (`flutter test integration_test`). Use real or high-fidelity fakes for backends; keep these few and focused on critical journeys.
- **`patrol`** when a flow needs native interaction (system permission dialogs, notifications, webviews) that `integration_test` can't reach. Screenshots/perf traces attach here.

## 9. Coverage, flakiness & CI
- `flutter test --coverage` → `coverage/lcov.info`; treat coverage as a *gap-finder*, not a target (100% of getters proves nothing). Exclude `generated_paths` from coverage.
- **Flakiness sources** are enumerated as the §10 completion criterion — the usual culprits are un-reset globals/singletons (`get_it` — `reset()` in tearDown), real `sleep`/network/clock/random, leaked timers/subscriptions, unpinned goldens, and ordering reliance.
- Keep tests **fast and hermetic**; a flaky test is worse than no test — quarantine (`skip: 'reason'`) with a tracking issue rather than let it erode trust, and fix or delete it.

## 10. Common mistakes / anti-patterns (completion criterion)
A critique pass is done only when the bolded rules in §§1–9 and the items below have each been
**ruled in or out** against the diff:
- Wrong layer — a widget/integration test proving logic a unit test should own; over-mocking a pure function.
- `pumpAndSettle` on an infinite animation (hang); missing `pump` after an interaction; asserting before the frame that renders the change.
- Real `sleep`/`Future.delayed` to "wait"; real network/disk/clock/random in a unit or widget test.
- Interaction verification (`verify().called`) where state assertion is available and less brittle; missing `registerFallbackValue` for `any()` custom types.
- Brittle finders (`find.byType`/tree index) where a `Key`/text/semantics finder is stable.
- Golden without pinned fonts/size/animations (flake); golden used for logic.
- `get_it`/singletons not reset between tests; `StreamController`/timer not closed; `ProviderContainer` not disposed.
- Testing a whole widget tree to prove one branch instead of testing the state holder directly.

## Currency
`flutter_test`/`package:test` matchers are stable; per-package APIs (`mocktail`, `bloc_test`, `integration_test`/`patrol`, `alchemist`/`golden_toolkit`) vary by version. Maintainer notes + package refs live in `SOURCES.yaml`.
