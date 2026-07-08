# When (and where) to mock — Dart

> Reference for [`flutter-tdd`](../SKILL.md). Dart-native rewrite of mattpocock/skills `tdd/mocking.md` (MIT).
> This is the *decision* (when/where) + design-for-substitutability. The **double taxonomy** (dummy/fake/
> stub/spy/mock), mocktail/mockito mechanics, and fixture placement live in `flutter-testing-expert §4` —
> don't duplicate them here.

## Mock only at system boundaries
Substitute the things you don't control or can't make deterministic:
- **Network** — your `http`/`dio` client / API service (never hit the live network in unit tests)
- **Persistence** — a local DB (sqflite/drift/Isar) or a remote store (prefer an **in-memory fake** over a mock when feasible)
- **Time & randomness** — `DateTime.now()`, `Stopwatch`, `Random`, uuid → inject a fixed clock/value (deterministic tests only)
- **Platform channels / device** — `SharedPreferences` (`setMockInitialValues`), secure storage, sensors, path_provider
- **File system** — sometimes; prefer a temp directory

**Don't mock what you own** — your own classes, internal collaborators, value logic. Mocking internals produces
tests coupled to structure that pass while production breaks (see [tests.md](tests.md)). Prefer the real thing:
real > fake (in-memory) > stub > mock.

## Design for substitutability

**1. Inject dependencies — don't construct them inside.**
```dart
// Easy to substitute: the dependency comes in
Future<Receipt> processPayment(Order order, {required PaymentClient client}) {
  return client.charge(order.total);
}

// Hard: builds its own concrete dependency — nothing to swap in a test
Future<Receipt> processPayment(Order order) {
  final client = StripeClient(apiKey: Secrets.stripeKey);   // unmockable
  return client.charge(order.total);
}
```
Inject via the project's `di` (constructor injection, or an abstract interface + test double / a `get_it`
or `ProviderScope` override). A change that's *hard to test* is a design signal — fix the seam, don't skip the test.

**2. Prefer focused interfaces over one generic `request()`.**
One method per operation → each is independently stubbable with a single fixed return; no conditional logic
inside the double, and a test's dependencies are visible from the interface it stubs.
```dart
// GOOD: one method per operation
abstract class OrdersApi {
  Future<User> user(UserId id);
  Future<List<Order>> orders(UserId userId);
  Future<Order> createOrder(OrderDraft draft);
}

// BAD: a single generic entry point forces if/else branching inside every stub
abstract class GenericApi {
  Future<dynamic> request(Endpoint endpoint, RequestOptions options);
}
```
Benefits: each stub returns one concrete shape · no branching in test setup · the interface documents which
operations a unit actually uses · type-safe per operation.
