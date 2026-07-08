---
name: flutter-tdd
description: "The test-driven-development DISCIPLINE for Flutter — use when implementing any logic or fixing any bug: red→green→refactor, Prove-It reproduction tests, vertical tracer-bullet slices. Process, not framework mechanics — test()/testWidgets/expect syntax, doubles, and golden/integration setup live in flutter-testing-expert."
---

# Flutter TDD — the discipline

> **Generated skill** — original wording, consolidated by `flutter-skill-consolidate`; provenance,
> sources, and licenses live in `SOURCES.yaml`. Don't hand-edit — change `SOURCES.yaml` and re-consolidate.

This skill owns the **process**: how to drive code with tests. Framework syntax — `test`/`testWidgets`,
`expect`/matchers, `WidgetTester`, doubles, golden/integration setup — lives in
**`flutter-testing-expert`**.

## Step 0 — Load profile
Read `.claude/flutter-profile.md`: `verify_command` (how to run the suite), `test_roots`, `source_roots`.
If missing, read the main checkout's copy —
`$(git rev-parse --path-format=absolute --git-common-dir)/../.claude/flutter-profile.md`
(the profile is usually gitignored, so worktrees don't inherit it).
You will **run** tests here, not just write them — know the command before you start. If
`verify_command` is unset, find the targeted run (e.g. `flutter test test/cart/cart_total_test.dart`
or `flutter test --name 'empty cart'`) so you can observe one test going red then green without the full suite.

## When to Use
Implementing any logic/behavior · fixing any bug (the **Prove-It** pattern) · modifying existing
behavior · adding edge-case handling · any change that could break existing behavior.

**When NOT to use:** pure non-logic changes — copy, assets, config, formatting — say so in the report.

---

## The cycle — RED → GREEN → REFACTOR (run the suite at each step)

```
   RED                  GREEN                 REFACTOR
 write a test      minimum code to       clean up; behavior
 that FAILS   ──→  make it PASS    ──→    unchanged          ──→ (repeat)
     │                   │                     │
 RUN it →           RUN it →              RUN the suite →
 see it fail        see it pass           still green
```

The runs are **not optional**. "Looks right" / "should pass" is not done — **execute the suite and read
the output.**

1. **RED — write one failing test, run it, watch it fail.** A test that passes the first time proves
   nothing (it isn't exercising new behavior, or the assert is wrong). Confirm it fails **for the
   reason you expect** (the missing behavior) — not a typo, missing import, or compile error.
   ```dart
   // RED — CartTotal doesn't exist yet, so this fails to compile/run. That's the point.
   test('empty cart totals to zero', () {
     expect(CartTotal(items: const []).amountMinor, 0);
   });
   ```
   Run it (`verify_command` or a targeted `flutter test <file> --name`) and see RED before writing any production code.

2. **GREEN — minimum code to pass, then run.** Don't over-build; just satisfy this test. Run the suite
   and confirm it now passes (and nothing else broke).

3. **REFACTOR — tidy with tests green, re-run after each step.** Extract duplication, deepen modules,
   improve names. **Never refactor while RED** — get to green first. Run the suite after every refactor
   step to prove behavior is unchanged. (Candidates: [`references/refactoring.md`](references/refactoring.md).)

## Vertical slices, not horizontal — the cardinal anti-pattern
Go **vertical** — tracer bullets. One test → one minimal impl → repeat. Each cycle uses what you
learned from the last.
```
WRONG (horizontal):  RED: test1..test5     then  GREEN: impl1..impl5
RIGHT (vertical):    test1→impl1 · test2→impl2 · test3→impl3 · …
```

## The Prove-It pattern (bug fixes)
When a bug is reported, **do not start by fixing it.** First write a test that **reproduces** it, run it,
and watch it **fail** — that failing run confirms you've actually found the bug. Then fix; the test goes
green and stays as a regression guard. Finally run the **full suite** for no regressions.
```
bug report → write reproduction test → RUN: fails (bug confirmed)
           → implement fix → RUN: passes → RUN full suite: no regressions
```

## Plan before you write tests
- Confirm the **public interface** the change needs, and **which behaviors matter most** — you can't
  test everything; focus on critical paths and complex logic, not every edge.
- List **behaviors** to test (observable outcomes), not implementation steps. If a profile `rules_file`
  names the domain vocabulary, use it so test names match the domain.
- For a non-trivial change, get the behavior list approved before coding (this is the same approval the
  `flutter-execute` plan gate covers — don't re-ask if it already happened).

## Writing good tests (Dart)
- **Test behavior through the public interface — not implementation.** Assert on the *outcome* (state),
  not which internal methods were called. A test that breaks when you rename a private helper, with
  behavior unchanged, was testing the wrong thing. (Good/bad examples: [`references/tests.md`](references/tests.md);
  for state holders: test the bloc/cubit/notifier's emitted state, not the widget — see `flutter-testing-expert §7,§10`.)
  ```dart
  // Good — observable outcome
  expect(sut.sortedByDateDescending.first.createdAt.isAfter(sut.sortedByDateDescending.last.createdAt), isTrue);
  // Bad — couples to internals; breaks on refactor though behavior is identical
  verify(() => mockDb.query('ORDER BY created_at DESC')).called(1);
  ```
- **DAMP over DRY in tests.** Each test reads like a self-contained specification; some duplication is
  fine if it makes the test independently understandable. Don't hide the inputs behind shared helpers.
- **Prefer real implementations over doubles:** real > fake (in-memory) > stub > mock. Use a double only
  when the real thing is slow, non-deterministic, or has side effects you can't control (network, clock,
  random). Over-mocking gives tests that pass while production breaks. (When/where to mock + design-for-
  substitutability: [`references/mocking.md`](references/mocking.md); double taxonomy + fixture
  placement: `flutter-testing-expert §4`.)
- **Arrange-Act-Assert** structure; **one concept per test** (`rejects empty title`, `trims whitespace`
  — not one `validates correctly`); **descriptive names** that read as a spec.
- **Deterministic only** — no `DateTime.now()`/random in tests; inject a fixed clock/value.

## The test pyramid
```
   E2E / Integration (~5%)   full flows on a device (integration_test/patrol) — critical paths only
 Widget / Integration (~15%) crosses a boundary (widget tree, network, disk) with a test seam
   Unit (~80%)               pure logic, isolated, milliseconds
```
(Layer mechanics — unit vs widget vs golden vs integration — live in `flutter-testing-expert §1`.)

**The Beyoncé rule:** if you liked it, you should have put a test on it. A refactor or migration isn't
responsible for catching your bug — your tests are.

## Anti-patterns (beyond the cycle rules above)
| Anti-pattern | Fix |
|---|---|
| Flaky (timing / order-dependent) | Deterministic asserts, isolated state, no `sleep`/real timers (see `flutter-testing-expert §9`) |
| Golden abuse | sparingly; review every change (`flutter-testing-expert §5`) |
| Skipping/disabling tests to go green | fix the code or the test, never mute it |
| Re-running unchanged code as reassurance | run after a *change*, not for comfort |

## Per-cycle checklist
```
[ ] One behavior, one test — vertical slice (not all-tests-then-all-code)
[ ] Test describes behavior via the public interface; survives an internal refactor
[ ] RED observed: ran it, saw it fail for the expected reason
[ ] GREEN: minimal code; ran it, saw it pass — no speculative features
[ ] Refactored only while green; re-ran the suite after each step
[ ] Bug fix? a reproduction test failed before the fix
[ ] Full suite run before claiming done — no regressions
```

## References
- [`references/tests.md`](references/tests.md) — good vs bad tests; behavior-through-the-interface examples.
- [`references/mocking.md`](references/mocking.md) — when/where to mock (boundaries only) + design-for-substitutability (DI, focused interfaces).
- [`references/refactoring.md`](references/refactoring.md) — refactor candidates for the REFACTOR step.
