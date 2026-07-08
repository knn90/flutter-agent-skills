---
name: flutter-widget-expert
description: "Expert Flutter widget guidance — widget composition, state (StatefulWidget / setState / ValueListenable / InheritedWidget), layout, performance (const & rebuilds), navigation, animation, accessibility, Material/Cupertino. Fires when work touches widgets (Widget, build(), setState, etc.)."
---

# Flutter Widget Expert

> **Generated skill** — original wording, consolidated by `flutter-skill-consolidate`; provenance,
> sources, and licenses live in `SOURCES.yaml`. Don't hand-edit — change `SOURCES.yaml` and re-consolidate.

**Source of truth:** Flutter's official documentation (flutter.dev, api.flutter.dev) + the
Material / Cupertino design guidelines are the **primary authority** (see `SOURCES.yaml`) and win on
any conflict, gated to the profile's `flutter_version`/`min_sdk`. Community sources fill in detail.
**Architecture / state:** follow the project's `architecture` + `state_type` from
`.claude/flutter-profile.md`. This skill owns the **widget-tree layer** (composition, rebuilds,
layout, navigation, a11y); the chosen state-management package (Bloc/Riverpod/Provider/…) layers on
top of the framework-native primitives below — don't impose one where the project doesn't ask.
Profile: `.claude/flutter-profile.md` — worktrees don't inherit it; fall back to
`$(git rev-parse --path-format=absolute --git-common-dir)/../.claude/flutter-profile.md`.

---

## 1. State & data flow
- **Framework-native ladder — pick the narrowest:** immutable config in → `final` fields on a `StatelessWidget`; local ephemeral UI state (toggles, controllers, animation) → `StatefulWidget` + `setState`; a single changing value shared without a package → `ValueNotifier` + `ValueListenableBuilder`; shared state passed down the tree → `InheritedWidget`/`InheritedModel` (what Provider/Riverpod/Bloc are built on). Reach for the project's `state_type` for app/feature state; don't hand-roll an `InheritedWidget` when it already provides one.
- **`setState` invalidates the whole `State`'s `build`** — Flutter has no per-property invalidation. Keep the rebuild surface small: push changing state into the smallest subtree (a dedicated `StatefulWidget` or a `ValueListenableBuilder`/`Selector`/`context.select`) rather than calling `setState` at the top of a big screen.
- **Never mutate state without `setState`** (the change won't paint) and **never call `setState` in `build`** (infinite loop). After an `await`, guard with `if (!mounted) return;` before `setState` — a resolved future on a disposed `State` throws.
- **Own vs receive:** a `State` that *owns* a controller/notifier creates it in `initState` and **must** `dispose()` it in `dispose`. A widget that *receives* one takes it as a `final` field and never disposes it (the owner does). Recreating an owned controller in `build` loses its state and leaks.
- **Lift state up** to the lowest common ancestor that needs it; pass values down, pass callbacks up. Don't store derived data in state — compute it in `build` (or memoize if genuinely expensive).
- Don't rebuild from `InheritedWidget` for high-frequency values (scroll offset, animation ticks) read by many widgets — every dependent rebuilds; use a targeted `ValueListenable`/`AnimatedBuilder` instead.

## 2. Widget composition & reuse
- **Extract subtrees into real `Widget` classes, not helper methods returning `Widget`.** A `_buildHeader()` method inlines its subtree into the parent's element, so it re-runs on every parent rebuild (unless it returns a `const` widget) and gets no element boundary of its own; a `HeaderWidget` class with `const` constructor gets its own element and lets Flutter skip rebuilding it when inputs are unchanged. This is the single highest-leverage composition rule.
- **`const` all the way down.** A `const` constructor + `const` call site means the widget instance is canonicalized, so its element is reused and its subtree skipped on rebuild. Make constructors `const` wherever fields allow; enable the `prefer_const_constructors` lint.
- Keep `build` pure and cheap: no I/O, no `Future` starts, no filtering/sorting/formatter allocation inline — do that in `initState`/the model/memoized fields. `build` can be called every frame.
- Prefer **composition over configuration**: many small widgets over one widget with 15 boolean flags. One primary widget per file; split files past ~300 lines; flag a `build` longer than ~one screen.
- Use `Builder` to get a `BuildContext` below an inherited widget you just inserted (e.g. `Scaffold.of(context)` needs a context under the `Scaffold`). Don't reach for a `GlobalKey` where a `Builder`/callback suffices — `GlobalKey`s are expensive and easy to leak.
- Centralize colors/spacing/text styles/durations in `ThemeData` / a design-tokens class; read via `Theme.of(context)` — don't hard-code padding or `Color(0xFF…)` ad hoc.

## 3. Layout
- **Never size from `MediaQuery.of(context).size` as if it were the widget's box** — it's the whole screen and breaks in sheets/dialogs/split views. Use `LayoutBuilder` (parent constraints), `Expanded`/`Flexible` (share space), `FractionallySizedBox`, or intrinsic sizing. Reserve raw pixel math for when layout truly depends on measured size.
- Understand **constraints go down, sizes go up, parent sets position**. Most "unbounded height/width" and overflow errors are a missing `Expanded`/`Flexible` inside a `Row`/`Column`, or a scrollable inside an unbounded parent — fix the constraint chain, don't wrap in a fixed `SizedBox` to paper over it.
- One child that should fill a row → `Expanded`, not `Row(children:[child, Spacer()])` unless you specifically want trailing space. Use `Spacer` for flexible gaps, `SizedBox` for fixed ones.
- Long/unbounded lists → `ListView.builder`/`GridView.builder`/`CustomScrollView` + slivers — **never** a `Column` inside a `SingleChildScrollView` for many children (builds them all). `SliverList`/`SliverAppBar` for collapsing headers.
- Respect text scaling and device sizes: avoid fixed heights on text containers; let content size drive. Empty states → a dedicated widget; loading → skeleton/`CircularProgressIndicator` sized in context.
- **Design polish:** spacing on a grid (4/8/12/16/24/32); hierarchy via `Theme` text styles + weight, not ad-hoc sizes; semantic theme colors (`colorScheme.surface`, `.onSurfaceVariant`) so light/dark + high-contrast adapt for free; `Divider()`/`Gap`-style consistency; consistent corner radii from a token.

## 4. Performance
- **`const` constructors are the primary rebuild firewall** (see §2). A `const` subtree is skipped entirely on parent rebuild.
- **Scope `setState`/rebuilds to the smallest widget.** Wrap only the changing leaf in a `StatefulWidget`/`ValueListenableBuilder`/`AnimatedBuilder` so a frequent change doesn't rebuild a whole screen. In a `ListView.builder`, keep `itemBuilder` cheap.
- **Stable keys for dynamic/reorderable lists:** give list items real domain `ValueKey`s so element/state stays attached across insert/remove/reorder. Don't add keys where identity is positional and stable (they cost). Never key on list index for a mutable list.
- **`RepaintBoundary`** around subtrees that repaint independently (animations, a video/canvas) so their repaints don't dirty the rest. Don't sprinkle them everywhere — each is a layer.
- Pass the **specific values** a widget needs, not whole models, and use `select`/`Selector`/`buildWhen` so a widget rebuilds only when *its* slice changes — narrows the rebuild fan-out.
- Do heavy CPU work (JSON parse, crypto, image decode) **off the UI isolate** via `compute`/`Isolate.run` — deep async/isolate work routes to `flutter-async-expert`. Cache decoded images (`cacheWidth`/`cacheHeight`, `ResizeImage`); never decode full-size images into list thumbnails. Cache `DateFormat`/`NumberFormat` in a `static final` / `late final` — they're expensive to construct.
- Measure with **DevTools** (rebuild counts via "Track widget rebuilds", the performance overlay for jank, the raster/UI thread timeline). `debugPrintRebuildDirtyWidgets` to find over-rebuilding. Prefer evidence over guessing; the frame budget is ~16 ms (60 Hz) / ~8 ms (120 Hz).

## 5. Navigation & presentation
- Follow the project's `navigation`. If it's **declarative** (`go_router`/`auto_route`/`Router`), add routes to the central config — don't sprinkle imperative `Navigator.push` that bypasses deep-linking/URL sync. If it's imperative `Navigator` 1.0, push/pop typed routes (`MaterialPageRoute`) and pass typed arguments, not `dynamic`.
- Return results with `Navigator.pop(context, value)` + `await push<T>()`; type the result. Guard `context` across the await with `context.mounted` before using it.
- **Modals own their own dismissal:** a `showModalBottomSheet`/`showDialog` result comes back via its `Future`; don't prop-drill `onSave`/`onCancel` callbacks when the return value carries the outcome. Give dialogs a clear single action path; `AlertDialog`/`showAdaptiveDialog` for platform-correct chrome.
- Keep route arguments lightweight and serializable (ids/values), never widget instances or live controllers — required for deep links and state restoration. Reset navigation stack on logout.
- Use `Cupertino*` widgets/routes on iOS-styled surfaces and `Material` on Android, or `*.adaptive` constructors where the framework provides them, per the project's design direction.

## 6. Animation & transitions
- **Prefer implicit animations** (`AnimatedContainer`, `AnimatedOpacity`, `AnimatedSwitcher`, `TweenAnimationBuilder`) for simple state→state transitions — they're declarative and cheap. Reach for an explicit `AnimationController` only for coordinated/looping/gesture-driven motion.
- An `AnimationController` needs a `TickerProvider` (`SingleTickerProviderStateMixin`) and **must be disposed**. Drive the smallest subtree with `AnimatedBuilder`/`ListenableBuilder` and its `child:` param (the `child` is built once and reused, not rebuilt each tick).
- **Animate cheap properties** (opacity, transform/scale via `Transform`, color) over relayout (changing `width`/`padding` every frame forces layout). Use `AnimatedSwitcher`/`Hero` for element transitions; `FadeTransition`/`SlideTransition` for enter/exit.
- Respect `MediaQuery.disableAnimations` / reduce-motion: swap large motion for a fade. Never animate every scroll frame with `setState`.

## 7. Accessibility
- **Use real interactive widgets** (`ElevatedButton`, `InkWell`, `IconButton`, `Checkbox`) rather than a bare `GestureDetector` on a `Container` — they bring focus, semantics, and correct traits for free. If you must use `GestureDetector`, wrap in `Semantics(button: true, label: …)`.
- Icon-only controls need a label: `IconButton(tooltip: …)` or a `Semantics` label — a screen reader announces nothing otherwise. Decorative images → `ExcludeSemantics`/`Semantics(excludeSemantics: true)`; informative images → `Semantics(label:)` or `Image(semanticLabel:)`.
- **Respect text scaling:** use `Theme` text styles and let `MediaQuery.textScaler` grow text; avoid fixed-height text boxes and hard pixel font sizes for primary content. Test at large scale factors.
- Meet **tap-target minimums** (48×48 logical px — `MaterialTapTargetSize`), sufficient contrast (check against `ColorScheme`), and don't encode meaning in color alone (add icon/label). Group related nodes with `MergeSemantics`; order with `Semantics(sortKey:)` when visual order ≠ read order.
- Expose stable `Semantics` labels / `Key`s per the profile's `accessibility` convention once the UI is interactive, so widget/integration tests (and screen readers) can find elements.

## 8. Async in widgets
- Never start network/`Future` work directly in `build`. Kick off async work in `initState` (or a state-management event), store the `Future`/`Stream`, and render it with `FutureBuilder`/`StreamBuilder` — and **always handle all of `hasError`/`hasData`/waiting**, not just the happy path.
- Guard every post-`await` use of `context`/`setState` with `mounted`/`context.mounted`. Treat cancellation as normal. Deep async — isolates, stream lifecycles, cancellation, `Completer`, event-loop ordering — routes to `flutter-async-expert`, invoked by the caller.

## 9. Modern API / always-never (currency)
- `Theme.of(context).colorScheme.*` / `textTheme.*` over hard-coded colors and `TextStyle`s. `ColorScheme.fromSeed` for Material 3 theming; `useMaterial3` is the default in current Flutter.
- Prefer `withValues(alpha:)` over the deprecated `withOpacity` (precision loss); `MediaQuery.textScalerOf(context)` over the deprecated `textScaleFactor`; `MediaQuery.sizeOf`/`platformBrightnessOf` (targeted `InheritedModel` aspects) over `MediaQuery.of(context).size` (rebuilds on any MediaQuery change).
- `ListView.builder`/`.separated` over `ListView(children: […])` for non-trivial lists; `SizedBox` over `Container` when you only need sizing; `EdgeInsets.symmetric`/`.only` over recomputed insets. `super.key` in constructors; `Key? key` forwarded. Avoid `!` null-assertion where a null-check/`?.`/`??` reads clearer.

## 10. Testing & previews
- Widget tests (`testWidgets` + `WidgetTester`) for behavior; pump the widget under a `MaterialApp`/`Directionality` as needed; find via `Key`/`Semantics`/text; assert with `find`/`matchesGoldenFile`. Golden tests for pixel-level layout. Deep testing mechanics (pump vs `pumpAndSettle`, fakes, `mocktail`) route to `flutter-testing-expert`.
- Keep logic **out of widgets** (in the state layer/services) so it's unit-testable without pumping a tree; widget tests then cover wiring and rendering only.
- Use `flutter run`/hot-reload and (if used) `widgetbook`/`@Preview` device previews to eyeball each meaningful state (default/empty/error/loading). Previews/tests must be **self-contained** — inject fakes via the project's `di`, never hit live network/disk.

## 11. Material 3 / adaptive & platform design
- **Adopt platform-adaptive UI deliberately, not reflexively** — don't convert a working screen to `.adaptive`/Cupertino unless asked. When targeting both platforms, use `*.adaptive` constructors (`Switch.adaptive`, `showAdaptiveDialog`, `Icon`s per platform) or branch on `Theme.of(context).platform`.
- Drive everything from `ThemeData`/`ColorScheme`/`TextTheme` so dark mode, dynamic color, and high-contrast come free. Don't fight the Material components with heavy custom overlays where a `themeExtension` or component theme (`FilledButtonTheme`, `CardTheme`) does the job.
- Use `SafeArea`, `MediaQuery` padding/insets (keyboard via `viewInsets`), and `Scaffold`'s `resizeToAvoidBottomInset` correctly rather than hard-coding notch/status-bar offsets.

## Review completion criterion
A critique pass is done only when each **bolded** rule in §§1–11 has been **ruled in or out**
against the diff — not when the obvious findings run dry.

## Contested / judgment calls
- **State management choice** is the project's (`state_type`), not this skill's — critique adherence to the chosen approach and the framework-native rules above, don't relitigate Bloc-vs-Riverpod on a small change.
- **`const` everywhere** is near-universally right, but a `const` constructor on a widget whose fields are genuinely dynamic is pointless churn — flag missing `const` only where the call site could actually be `const`.
- **Design-polish rules** (spacing grid, semantic colors, fewer type sizes) are universal; a specific aesthetic (rounded vs sharp, density) is the project's — take the consistency rules, not one look.
