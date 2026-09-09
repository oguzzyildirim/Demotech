# Core/Helpers — SKILL.md

## Purpose
`Core/Helpers` holds stateless, reusable helper types that perform a focused
operation on behalf of other layers (formatting, validation, mapping, encoding,
keychain access, etc.). A helper supports a flow — it never owns one. If a type
holds observable state, drives navigation, or contains feature business logic, it
does not belong here.

## Layer Boundaries (Core siblings)
Respect the folder split — do not blur it:
- **Extensions/** — protocol conformances and extensions on existing types
  (`String+Validation.swift`). Never put an extension in Helpers.
- **Utilities/** — pure, dependency-free free-standing functions, constants, and
  type-agnostic primitives. If it has zero dependencies and no domain meaning, it's
  a Utility, not a Helper.
- **Helpers/** — small, single-purpose types that *do* something with a dependency
  or a domain concept (`DateFormattingHelper`, `KeychainHelper`, `DTOMapper`).
- **Services/**, **Network/**, **Storage/**, **Models/** — Helpers may consume
  Models, but MUST NOT depend on Services, Network, or Storage.

Dependency direction points downward only. A helper importing a Service is a bug.

## Scope & Boundaries
- Helpers MUST be stateless and side-effect-free unless the side effect is their
  single explicit purpose (e.g. `KeychainHelper`).
- Helpers MUST NOT be `@Observable`, `ObservableObject`, or hold
  `@State` / `@Binding` / `@Environment`.
- Helpers MUST NOT import SwiftUI unless the helper's sole responsibility is a
  SwiftUI-value concern (a `Color`/`Font` mapper, a `ViewModifier`). Pure-logic
  helpers stay SwiftUI-free.
- Helpers MUST NOT reference a View, ViewModel/Store, or Router/Coordinator.

## Access Control
- Default every type, property, and method to the most restrictive access level.
- Use `private` for implementation details, `fileprivate` only when two types in the
  same file must share, `internal` (implicit) for module API, `public` only for
  types shared across a package/framework boundary.
- Expose read-only state with `private(set)`.
- Prefer `private let` over `private var`; every `var` must be justified.

## Type Declaration Rules
- Prefer `struct` or a case-less `enum` namespace over `class`.
- Every `class` that is not intended for inheritance MUST be `final class`.
- For grouping static-only helpers, use a case-less `enum` (uninstantiable) — never
  a `struct`/`class` with a `private init`:
```swift
  enum DateHelper {
      static func iso8601String(from date: Date) -> String { ... }
  }
```

## SOLID
- **S** — one helper, one responsibility. No `Helpers.swift` / `Common.swift` god files.
- **O** — extend via new conformances/extensions, not by editing existing helpers.
- **L** — a protocol-typed helper must be substitutable with no special-casing.
- **I** — many small protocols (`Validating`, `Formatting`) over one fat protocol.
- **D** — consumers depend on helper protocols and inject concretes where testability
  matters; concrete helpers are leaves.

## MARK Organization (mandatory in every file you touch)
Reorganize every modified file to this order:
```swift
// MARK: - Type Definition
// MARK: - Properties
// MARK: - Initialization
// MARK: - Public API
// MARK: - Helpers
// MARK: - Private
```
No exceptions — any file edited in this layer must conform before you finish.

## Naming
- Helper names end in their role: `...Helper`, `...Formatter`, `...Validator`,
  `...Mapper`, `...Encoder`.
- Methods read as phrases at the call site; avoid `get`/`do` prefixes.
- No abbreviations except industry-standard (`URL`, `ID`, `JSON`).

## Swift 6 / iOS 16.6 Best Practices (2026)
- Adopt strict concurrency: mark cross-actor helpers `Sendable`; annotate any
  main-thread-bound helper `@MainActor`.
- Prefer `async`/`await` over completion handlers; use `throws` (typed throws where
  the error domain is closed) for fallible work.
- Use `Result` only at boundaries where `throws` is impractical.
- No force-unwrap (`!`), force-cast (`as!`), or force-try (`try!`) — `guard`,
  `if let`, or throw instead.
- Use `some`/`any` deliberately: `some` for opaque returns, `any` only for genuine
  heterogeneity.

## Clean Code
- One function, one job; keep functions short and complexity low.
- No magic literals — hoist to `private static let` constants or an enum namespace.
- Guard early, return early; avoid nested pyramids.
- Every `public` helper carries a `///` doc comment stating contract, params, and
  thrown errors.

## Testability
- Pure helpers unit-testable with no mocks.
- Time, randomness, locale, and I/O are injected, never accessed globally inside a
  helper.

## Prohibited
- Catch-all files (`Helpers.swift`, `Common.swift`, `Utils.swift`).
- Extensions living in Helpers (they belong in `Core/Extensions`).
- Singletons (`.shared`) unless wrapping a genuinely global system resource — then
  `final` with `private init`.
- Business logic, persistence orchestration, navigation, or observable state.