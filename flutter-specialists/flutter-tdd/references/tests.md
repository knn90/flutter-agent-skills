# Good vs bad tests (Dart)

> Reference for [`flutter-tdd`](../SKILL.md). Dart-native rewrite of mattpocock/skills `tdd/tests.md` (MIT).
> The *one rule*: test observable behavior through the **public interface**, never internal structure.

## Good — integration-style, behavior through the public API
Exercises real code paths the way a caller would; survives internal refactors; the name says **what**, not how.

```dart
test('user can checkout with a valid cart', () async {
  final cart = Cart()..add(product);
  final result = await checkout(cart, using: paymentMethod);
  expect(result.status, CheckoutStatus.confirmed);   // outcome the caller cares about
});
```
Characteristics: behavior callers care about · public API only · one logical assertion · survives refactors · describes WHAT.

## Bad — coupled to implementation
Breaks when you refactor even though behavior is unchanged — the signal it was testing *how*, not *what*.

```dart
// BAD: asserts an internal interaction, not an outcome
test('checkout calls paymentService.process', () async {
  final payment = MockPaymentService();
  when(() => payment.process(any())).thenAnswer((_) async => receipt);
  await checkout(cart, using: payment);
  verify(() => payment.process(any())).called(1);    // breaks on any refactor; proves nothing about the result
});
```
Red flags: mocking internal collaborators · testing private methods · asserting call counts/order · name describes HOW · verifying through a back channel instead of the interface.

## Verify through the interface, not a back channel
```dart
// BAD: reaches past the interface into the store
test('createUser writes a row', () async {
  await createUser(name: 'Alice');
  final rows = await db.rawQuery('SELECT * FROM users WHERE name = ?', ['Alice']);
  expect(rows, isNotEmpty);                            // couples the test to the schema
});

// GOOD: round-trips through the public API
test('created user is retrievable', () async {
  final created = await createUser(name: 'Alice');
  final fetched = await userById(created.id);
  expect(fetched.name, 'Alice');
});
```

For a Flutter screen, the "interface" is its **state holder's emitted state** (bloc/cubit/notifier), not the
`Widget` (see `flutter-testing-expert §7,§10`). For `expect`/matchers, widget finders, and doubles, see
`flutter-testing-expert §2–4`.
