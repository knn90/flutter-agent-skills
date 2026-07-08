---
name: flutter-sequential-thinking
description: "Step-by-step analysis with revision/branching for complex Flutter problems — multi-layer data-flow bugs, cache/normalisation issues, navigation routing tangles, state-transition holes, async/Future/Stream races, money/PII correctness audits. Internal reasoning aid, not a planner."
argument-hint: "[problem to analyse step-by-step]"
---

# Sequential Thinking — Flutter

## Step 0 — Load profile (light)
Skim `.claude/flutter-profile.md` for `architecture`, `state_type`, `navigation`,
`networking`, `high_rigor_domains` so the patterns below map onto this app's vocabulary.
If missing, skim the main checkout's copy —
`$(git rev-parse --path-format=absolute --git-common-dir)/../.claude/flutter-profile.md`
(the profile is usually gitignored, so worktrees don't inherit it).

## When to Apply
At least one of:
- **Multi-step data flow** — Repository → Bloc/Notifier → Widget; trace where state diverges.
- **Cache / normalisation** (if `networking` uses a cache) — query returns right data, UI shows stale.
- **Navigation routing** — push/pop/dialog ordering, stale `BuildContext` after `await`, redirect races.
- **State transitions** (`state_type`) — loading→loaded→error edges; empty vs error vs initial; pagination handoff.
- **Async** — `Future`/`Stream` races, unawaited futures, `StreamSubscription` leaks, event-loop reentrancy, isolate boundaries.
- **Hypothesis-driven debugging** — symptom is N layers from cause.
- **`high_rigor_domains` correctness audits** — wrong logic ships real-money / PII bugs.

## When NOT to Use
| Case | Use instead |
|---|---|
| Simple one-step answer | Just answer |
| Brutal trade-off comparison | `flutter-brainstorm` |
| Plan + phases | `flutter-plan` |
| Bug needs log/CI investigation | Direct `Bash`/`Read` |

---

## Core Process

1. **Loose estimate** — `Thought 1/N: [framing]`.
2. **Structure each thought** — build on prior context, one aspect each, state assumptions/
   uncertainties, signal what the next thought addresses.
3. **Adjust N as understanding changes** — expand or contract.
4. **Revision**
   ```
   Thought 5/8 [REVISION of Thought 2]: <corrected understanding>
   - Original: … - Why revised: … - Impact: …
   ```
5. **Branching**
   ```
   Thought 4/7 [BRANCH A from 2]: <approach A>
   Thought 4/7 [BRANCH B from 2]: <approach B>
   ```
   Compare, converge with rationale.
6. **Hypothesis & verification**
   ```
   Thought 6/9 [HYPOTHESIS]: <cause/solution>
   Thought 7/9 [VERIFICATION]: <checked file:line — found …>
   ```
   Every [HYPOTHESIS] must be paired with a [VERIFICATION] carrying a receipt — a real
   `path:line`, not abstract reasoning.
7. **Finish** — `Thought N/N [FINAL]` names the root cause (`path:line`), the fix, and the
   test gap that would have caught it (or the pattern's own FINAL shape). Stop there —
   don't pad to N.

---

## Reusable Patterns (adapt names to `state_type`/`navigation`/`networking`)

### State-transition audit
```
1: Define expected sequence (initial → loading → loaded/empty/error)
2: Locate the state holder — Bloc/Cubit/Notifier/ChangeNotifier (path:line)
3: Trace each emit/notifyListeners point — does each branch exit cleanly?
4: Pagination — cursor handoff between pages
5 [HYPOTHESIS]: skipped state / lost cursor / double-emit
6 [VERIFICATION]: read tests — is this edge covered?
N [FINAL]: root cause + fix + test gap
```

### Cache / normalisation debug  (only if networking caches)
```
1: Identify query/request + cache key policy
2: What writes that key? (mutation, manual cache write, repository refresh)
3: Listeners — subscription lifecycle tied to which widget/provider?
4 [HYPOTHESIS]: stale entity after partial update
5 [VERIFICATION]: read generated request + key fn (never edit generated_paths)
N [FINAL]: cache fix + invalidation strategy
```

### Async race / cancellation / leak
```
1: Identify Future/Stream boundaries (created / awaited / listened / cancelled)
2: main isolate vs background isolate (compute/Isolate.run) — any heavy work blocking the main isolate?
3: Cancellation — is the StreamSubscription cancelled / CancelableOperation honoured in dispose()?
4: Reentrancy — re-callable while a previous await is suspended? unawaited future racing state?
5 [HYPOTHESIS]: e.g. cursor advanced while previous fetch in-flight; setState after dispose
6 [VERIFICATION]: read the class + service; confirm mounted check / cancel / guard
N [FINAL]: race surface + guard/cancellation fix
```

### Navigation routing knot
```
1: Diagram screens + desired transitions (routes / go_router config)
2: Locate the router — what data/callbacks pass to children? redirect logic?
3: BuildContext after await — is `mounted` / `context.mounted` checked before use?
4: Presentation order — dialog over sheet? pop during a push transition? redirect loop?
5 [HYPOTHESIS]: missed pop / stale context / redirect race / state mismatch
6 [VERIFICATION]: trace one path end-to-end
N [FINAL]: routing fix + invariant a test must protect
```

### Money / PII correctness audit  (high_rigor_domains)
```
1: What value flows here? (Decimal package? double? string/int minor-units from backend?)
2: Precision boundaries — where formatting/rounding happens (double math = danger)
3: Sign / direction — refund vs charge, credit vs debit
4: Edge cases — zero, negative, very large, multi-currency
5: PII — what's logged? crosses analytics / crash reporting?
6 [HYPOTHESIS]: precision loss / leaked PII / wrong sign
7 [VERIFICATION]: run the math on paper + grep log emissions on the value
N [FINAL]: pass/fail per case + remediation
```

## Constraints
- **DO NOT** implement during thinking — just reason.
- **Collapse the chain to conclusion + key reasoning** — surface the full trace only when
  the user asks for it or `high_rigor_domains` correctness reasoning needs an audit trail.
