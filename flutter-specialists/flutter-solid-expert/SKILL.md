---
name: flutter-solid-expert
description: "Expert SOLID + clean-decoupling review for Flutter/Dart — the 5 principles by name (SRP, OCP, LSP, ISP, DIP), composition-root DI, decoupling patterns (Decorator / Composite / Adapter / Facade), and framework isolation. Fires when a change adds/alters classes, interfaces, services, blocs/notifiers, or DI/composition wiring. Delegates *which architecture pattern* to the architecture lens."
---

# SOLID / Decoupling Expert (Flutter / Dart)

> **Generated skill** — original wording, consolidated by `flutter-skill-consolidate`; provenance,
> sources, and licenses live in `SOURCES.yaml`. Don't hand-edit — change `SOURCES.yaml` and re-consolidate.

**Source of truth:** SOLID is **general** software engineering (Robert C. Martin) — Flutter takes no
official app-architecture stance. The Dart-idiomatic translation is **implicit interfaces + abstract
classes + mixins + composition** (dart.dev: Classes, Abstract & interface classes, Mixins). Gate to the
project's `architecture` + conventions in `rules_file`.
Profile: `.claude/flutter-profile.md` — worktrees don't inherit it; fall back to
`$(git rev-parse --path-format=absolute --git-common-dir)/../.claude/flutter-profile.md`.

**North star (YAGNI):** SOLID serves *change*, not ceremony. An abstraction earns its keep with
≥2 implementations **or** a real test seam; otherwise it's **speculative generality**.

---

## The five principles (rule · Dart idiom · smell → fix)

**1. SRP — Single Responsibility.** One reason to change / one actor a type answers to. *Smell:* a
bloc/notifier that fetches **and** parses **and** caches **and** formats; a `Manager`/`Helper`/`Utils`
doing everything. *Fix:* one use-case/repository/service per responsibility; the bloc/notifier only orchestrates.

**2. OCP — Open/Closed.** Add behavior by adding a type, not by editing existing code. *Dart:* an
abstract interface + a new implementer / injected strategy. *Smell:* a `switch (kind) { … }` you must edit for every
new case (a `sealed` switch that's *meant* to be exhaustive is fine — that's the opposite intent). *Fix:* polymorphism / strategy / a registry.

**3. LSP — Liskov Substitution.** An implementer must be usable everywhere the abstraction is, with no
surprises. *Dart:* prefer `implements`/composition + mixins over deep `extends` inheritance. *Smell:* an
implementer that `throw UnimplementedError()`s or no-ops part of the interface ("unsupported"). *Fix:* split the interface (→ ISP).

**4. ISP — Interface Segregation.** Many small, client-focused interfaces over one fat one. *Dart:*
`FeedLoader`, `ImageCache` as separate `abstract interface class`es (compose by implementing several).
*Smell:* a god interface with 15 methods; implementers stubbing half of them. *Fix:* split by what each client actually uses.

**5. DIP — Dependency Inversion.** Policy depends on **abstractions it owns**, not on concrete
details. *Dart:* the **domain declares** `abstract interface class FeedRepository`; the data/network module
*implements* it; the composition root wires them. *Smell:* a bloc/notifier that `import`s `dio`/`sqflite`
or constructs a concrete client. *Fix:* invert — depend on a domain interface, inject the impl.

## Dependency injection & the composition root
- **Constructor injection.** The **composition root** (`main.dart` / the DI setup — `get_it`/`injectable`
  registration, a `ProviderScope`/`MultiProvider` at the app root, or a `*Factory`) is the *only* place that
  knows concrete types and assembles the graph. Everything else takes abstractions.
- *Smells:* the service locator reached from policy code (`GetIt.I<Foo>()` called *inside* a bloc/use-case
  instead of injected via constructor); singletons; **constructor over-injection** (≥4–5 deps ⇒ the type
  does too much → split it, or hide a subsystem behind a Facade).
- Constructor or interface injection gives testability without ceremony (no live `http`/`dio` client,
  `SharedPreferences`, `DateTime.now()` reached from inside — inject them).

## Decoupling patterns (the Essential-Developer toolkit)
- **Decorator** — add a cross-cutting concern (logging, analytics, retry, caching) with the *same*
  interface in and out, without touching the decorated type.
- **Composite** — combine implementations behind one interface (e.g. remote-**with-fallback-to**-cache `FeedRepository`).
- **Adapter** — bridge a concrete/SDK type to the domain's interface, keeping the SDK (dio, Firebase, sqflite) at the edge.
- **Facade** — hide a multi-step subsystem behind a simple interface for the composition root.

## Framework isolation
- The **domain is pure Dart** — no `import 'package:flutter/…'`, no `dio`/`sqflite`/`firebase_*`,
  no `BuildContext` in domain/use-case/entity types. Frameworks are *replaceable details* behind interfaces
  the domain owns; map **DTO → domain entity at the boundary**. *Smell:* `import 'package:flutter/material.dart'`
  in a model; a `Response`/`DocumentSnapshot`/JSON `Map` leaking into the domain or UI. (Pairs with the
  testing skill's mockability + the async skill's isolate/boundary rules.)

## Idiomatic Dart translation
- Prefer **implicit interfaces** (every class induces an interface — `implements SomeClass`) and
  `abstract interface class` (Dart 3, for pure contracts) over deep inheritance. Use **mixins** for shared
  behavior across unrelated types, **composition** over inheritance, `sealed`/`final` classes to constrain
  hierarchies, and extension types / value objects for domain primitives. This is how SOLID lands idiomatically in Dart.

## Review checklist (what this skill flags)
Per changed/added type: single responsibility? depends on abstractions or concretes? a framework
leaking into the domain? interface the right size (ISP) / honestly substitutable (LSP)? extended vs
modified (OCP)? wired at the composition root, not reached as a service locator/singleton? Output each finding as
`{ severity, file:line, principle, problem, fix }`. Severity: **Critical** (framework leak into domain,
concrete dependency that blocks testing a high-value path) · **Important** (God type, fat interface,
over-injection, OCP switch-edit) · **Nit** (naming, minor seam). Don't invent abstraction needs —
flag *speculative* abstractions too.

## Boundaries (avoid duplication)
- **Pattern selection** (Clean vs MVVM vs feature-first, Bloc vs Riverpod) → the architecture lens, not here.
- **Async isolation / isolates** → `flutter-async-expert`. **Widget state ownership / rebuilds** →
  `flutter-widget-expert`. **Test seams / doubles** → `flutter-testing-expert`. This skill is the
  principle-level "is it decoupled and substitutable?" lens that sits beneath all of them.
