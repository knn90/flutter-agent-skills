---
name: flutter-async-expert
description: "Expert Dart async reference — fires when work touches async/await, Future, Stream/StreamController, async*/yield, Isolate/compute, StreamSubscription, Completer, cancellation, Zones, or unawaited futures."
---

# Dart Async Expert

> **Generated skill** — original wording, consolidated by `flutter-skill-consolidate`; provenance,
> sources, and licenses live in `SOURCES.yaml`. Don't hand-edit — change `SOURCES.yaml` and re-consolidate.

**Source of truth:** Dart async is a *language + core-library* feature, so **dart.dev — the language
tour's async/streams pages, the "Concurrency in Dart" and "Isolates" guides, and the `dart:async`
API reference — is the primary authority** and wins on any conflict. Community sources supply
practical LLM-mistake patterns, not semantics. Gate version-specific guidance (records, patterns,
sealed classes, `Isolate.run`) by the profile's `min_sdk`/`flutter_version`.
Profile: `.claude/flutter-profile.md` — worktrees don't inherit it; fall back to
`$(git rev-parse --path-format=absolute --git-common-dir)/../.claude/flutter-profile.md`.

---

## 1. The execution model (the foundation)
- **Dart is single-threaded *per isolate*, driven by an event loop.** There is **no shared-memory concurrency** and therefore **no data races** within an isolate — but there ARE **logic races**: state can change at any `await` because other queued work runs during the suspension (see §8). This is the headline mental model.
- Two queues: the **microtask queue** (drained fully first — `scheduleMicrotask`, `Future.then`/`.value`/`.sync` continuations) and the **event queue** (I/O, timers, gestures, `Future.delayed`, one event per loop turn). Microtasks starve the event queue if you flood them — never spin work through `scheduleMicrotask` in a loop.
- `await` yields control back to the event loop at that point; code after it runs as a **microtask** when the awaited future completes. **`await` is a suspension point, not a thread switch** — the same isolate resumes it.
- Parallelism (using multiple cores / not blocking the UI) is **only** via **isolates** (§5), not threads.

## 2. Future semantics & pitfalls
- A `Future<T>` is a one-shot eventual value: pending → completed-with-value **or** completed-with-error. Prefer `async`/`await` over `.then()`/`.catchError()` chains — the compiler-friendly linear form is far easier to get error handling right.
- **Never drop a future on the floor.** An un-awaited future whose error you don't handle becomes an **unhandled async error** (crashes / logs, and races real state). If you deliberately fire-and-forget, wrap it in `unawaited(...)` (from `dart:async`) *and* ensure the callee handles its own errors — that documents intent and silences the lint.
- **`await` inside a `for` loop serializes** each iteration. For independent work, collect futures and `await Future.wait([...])` (parallel) — but know `Future.wait` **rejects on the first error and does not cancel the rest**; use `eagerError`/handle per-future (`.catchError`) when you need all results or partial success.
- `Future.delayed(Duration.zero)` is an **event-queue** hop, not a microtask — don't use it to "wait for state"; it's a code smell for a real ordering fix.
- Don't mark a function `async` if it never `await`s — it just wraps the result in an extra future. Return the future directly (`return foo();`) unless you need a `try/catch` around it.

## 3. async/await & error handling
- Errors in an `async` function surface as a **rejected future**; catch with `try/catch` around the `await`. `.then(onError:)`/`.catchError()` only catch errors from the future they're attached to — easy to miss one link in a chain.
- **`await` in a `catch`/`finally`** works; a `finally` runs even after a `return`/throw — put cleanup (close a client, cancel a subscription) there.
- An error thrown **synchronously before the first `await`** in an `async` function is still returned as a rejected future (not thrown to the caller) — callers must `await`/handle it, not `try` the call itself.
- Rethrow, don't swallow: an empty `catch (_) {}` hides failures and desyncs state. Log or surface. Preserve stack traces with `catch (e, st)` and `Error.throwWithStackTrace` / `rethrow`.

## 4. Streams
- A `Stream<T>` is zero-or-more async events + optional error(s) + done. **Two kinds:** a **single-subscription** stream (default — one listener ever; buffers until listened) vs a **broadcast** stream (`.asBroadcastStream()` / `StreamController.broadcast()` — 0..N listeners, drops events with no listener). Choosing wrong is a top bug: listening twice to a single-subscription stream throws; expecting replay from a broadcast stream loses early events.
- **Every `listen()` returns a `StreamSubscription` you MUST `cancel()`** — in `dispose()` / when done — or it leaks (and keeps its source alive). Don't `listen` where a `StreamBuilder` (widget layer) already manages the subscription.
- **`StreamController`:** set `onListen`/`onCancel` for resource lifecycle; **`close()` it exactly once** or consumers never get `done`. A single-subscription controller with no listener **buffers unboundedly** — a memory leak under a fast producer; use `onListen` to start producing only when someone listens, or a broadcast controller if drops are acceptable.
- Transform with the built-ins (`map`, `where`, `expand`, `asyncMap`, `take`, `distinct`, `handleError`) rather than manual controllers. Note **`asyncMap` preserves order and awaits each** (serialized); it does not run concurrently.
- Generators: **`async*` + `yield`/`yield*`** builds a stream lazily and respects backpressure/pause; **`sync*`** builds an `Iterable`. Don't hand-roll a controller when a generator reads clearer.
- For debounce/throttle/combineLatest/merge, use **`package:rxdart`** or `package:async` (`StreamGroup`) — **don't hand-roll debounce with `Timer`** per event (ordering bugs, leaks).

## 5. Isolates — the only real parallelism
- Isolates have **separate memory**; they communicate only by **message passing** over `SendPort`/`ReceivePort`. Messages are deep-copied (or transferred for `TransferableTypedData`); you cannot share mutable objects. Closures sent to an isolate can't capture non-sendable state.
- **Offload heavy CPU work off the UI isolate** so it doesn't drop frames: **`compute(fn, arg)`** (Flutter, one-shot) or **`Isolate.run(() => …)`** (Dart 2.19+, preferred — cleaner, returns a value/rethrows). The function must be a **top-level or static** function (or a closure capturing only sendable data) for `compute`.
- Use a **long-lived isolate** with explicit ports only when you have a *stream* of work (repeated jobs); for one-off transforms `Isolate.run`/`compute` is simpler and auto-tears-down. Spawning an isolate has real startup + memory cost — don't isolate trivial work (the copy cost can exceed the compute).
- Platform channels / plugins generally must be called from the **root/UI isolate** (background isolates need a `BackgroundIsolateBinaryMessenger` to use plugins) — don't call UI-isolate-only APIs from a spawned isolate blindly.

## 6. Cancellation
- **Dart `Future`s are NOT cancelable.** There's no `future.cancel()`. Cancellation is designed at a higher level:
  - **`StreamSubscription.cancel()`** stops stream delivery (and should stop the producer via `onCancel`).
  - **`package:async` `CancelableOperation`** wraps a future with a `cancel()` for one-shot work.
  - A **flag** the async loop checks at safe points (like cooperative cancellation) for CPU loops.
- In Flutter, the practical cancellation boundary is the widget/state lifecycle: **guard `if (!mounted) return;` (or `context.mounted`) after every `await`** before touching state or `BuildContext`, and cancel subscriptions/timers/operations in `dispose()`. A resolved future that calls `setState` on a disposed `State` throws.
- **Don't swallow the fact that work kept running** — a "cancelled" screen whose future still writes global state or fires analytics is the classic stale-write bug. Ignore the *result* (mounted guard) **and** stop the *source* (cancel the subscription/operation).

## 7. Completers
- `Completer<T>` bridges a callback-based API into a `Future`. **Complete it EXACTLY once on every path:** never → the awaiter hangs forever; twice → `complete`/`completeError` throws `StateError`. Guard with `if (!completer.isCompleted)`.
- Audit early returns, error callbacks, and timeouts. Use `completer.completeError(e, st)` for the error path. **Don't wrap an API that already returns a `Future`** in a `Completer` — just return/await it.
- Prefer `future.timeout(duration, onTimeout: …)` for timeouts over a manual `Completer` + `Timer`.

## 8. Interleaving & logic races (Dart's "reentrancy")
- Because other queued work runs during every `await`, **state you read before an `await` may be stale after it.** The classic bug: `if (_cache[k] == null) { _cache[k] = await _load(k); }` — two callers both pass the null check and both load (duplicate work, last-writer-wins). Fix: **dedup in-flight work by storing the `Future`** (`_inFlight[k] ??= _load(k); return await _inFlight[k]!;`), or re-check after the await.
- **Re-read/re-validate invariants after each `await`** (current selection, mounted, still-latest-request). Guard against out-of-order completions: tag each request (a monotonically increasing id / the search term) and **drop a response that is no longer the latest** — a slow earlier request must not overwrite a newer one.
- Isolation ≠ atomicity: a sequence spanning an `await` is **not** atomic. Complete multi-step mutations synchronously, or use a lock (`package:synchronized`) / a serialized queue when you truly need mutual exclusion.

## 9. Zones & uncaught errors
- Flutter routes framework errors through `FlutterError.onError`; **async errors outside the framework** (an un-awaited future's error, an isolate error) need `PlatformDispatcher.instance.onError` or a `runZonedGuarded(app, onError)` wrapper — wire one for crash reporting (`{crash_reporting}`). Don't scatter `try/catch` where one top-level guard belongs.
- `Zone`s also fork error handling / `print` interception; they do **not** provide cancellation or threads. Reach for `Zone` rarely — mostly it's error-boundary + testing (`fakeAsync` uses zones).

## 10. Widget integration
- Rendering a `Future`/`Stream` in the tree → `FutureBuilder`/`StreamBuilder`, and **handle every branch** (`hasError`/`hasData`/waiting) — the widget-layer rules (start async in `initState`/events not `build`, mounted guards, dispose) live in `flutter-widget-expert`, routed by the caller. This skill owns the async *semantics*; the widget expert owns *where the async plugs into the tree*.

## 11. Testing async code
- Test framework choice/mechanics → **flutter-testing-expert**. Async-specific: **never `sleep`/real delays to "wait"** (flaky). Use **`fakeAsync`** (`package:fake_async`) to control virtual time (`elapse`), `tester.pump(duration)` / `pumpAndSettle` in widget tests, and `expectLater(stream, emitsInOrder([...]))` for streams. Await the real work; bridge callbacks with a `Completer`. Test error paths with `expectLater(fut, throwsA(...))`, and cancellation by asserting the source stopped.

## 12. Anti-pattern checklist (completion criterion)
A critique pass is done only when **every anti-pattern below has been ruled in or out** against the diff:
- Un-awaited future whose error isn't handled, or `unawaited()` missing on intentional fire-and-forget (§2)
- Empty/`catch (_) {}` swallowing an async error; missing `try/catch` around an `await` (§3)
- `StreamSubscription` / `StreamController` / `Timer` not cancelled/closed; single-subscription controller buffering unboundedly (§4)
- Listening twice to a single-subscription stream, or expecting replay from a broadcast stream (§4)
- Heavy CPU work on the UI isolate that should be `compute`/`Isolate.run`; isolating trivial work (§5)
- Assuming a `Future` can be cancelled; stale write after "cancellation" (result not ignored AND source not stopped) (§6)
- `setState`/`context` used after `await` with no `mounted` guard (§6/§10)
- `Completer` not completed on every path, or completed twice (§7)
- check-then-act across an `await` (duplicate in-flight work); out-of-order response overwriting a newer one (§8)
- Hand-rolled `Timer` debounce; `asyncMap` assumed concurrent; microtask flooding starving the event queue (§4/§1)
- No top-level async error guard for crash reporting (§9)

## Contested / judgment calls
- **UI ↔ async boundary:** keep async work in the state layer/services/repositories; let widgets react to state. Defer to the project's `architecture`/`state_type`.
- **rxdart vs core streams:** core `dart:async` + `package:async` cover most needs; adopt `rxdart` when you genuinely need its combinators (debounce/combineLatest/BehaviorSubject) — don't pull it in for a single `map`.
- **When to spawn an isolate:** only when profiling shows UI-isolate jank from CPU work; the message-copy cost can outweigh the compute for small payloads.
