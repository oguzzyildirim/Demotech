# Core/Services — SKILL.md

## Purpose
`Core/Services` is the layer that owns application-level business logic and
orchestration. A Service coordinates lower layers (Network, Storage, Helpers) to
fulfill a domain use case, exposes a clean protocol-based API to the Presentation
layer, and returns domain Models — never DTOs. A Service *decides and orchestrates*;
it does not render, navigate, or hold view state.

## Layer Boundaries (Core siblings)
Respect the folder split — do not blur it:
- **Network/** — Services *consume* Network to fetch/send data, then map DTOs to
  domain Models. A Service depends on a `NetworkClient` abstraction, not on
  `URLSession`.
- **Storage/** — Services *consume* Storage for persistence/caching, behind a
  protocol. A Service decides *when* to cache; Storage decides *how*.
- **Models/** — Services return and operate on domain Models; this is their currency.
- **Helpers/**, **Utilities/**, **Extensions/** — Services may use these, but MUST NOT
  push business logic down into them.
- **Presentation/** — Services are *consumed by* ViewModels/Stores. A Service MUST
  NOT reference a View, ViewModel/Store, or Router.

Dependency direction points downward only. A Service importing SwiftUI or referencing
UI is a bug.

## Scope & Boundaries
- Every Service MUST be defined behind a protocol; consumers depend on the protocol,
  the concrete type is injected.
- Services MUST NOT be `@Observable` or `ObservableObject`, and MUST NOT hold view
  state (`@State`/`@Binding`/`@Published`). Transient in-memory caches are allowed
  only when that is the Service's explicit responsibility.
- Services MUST NOT import SwiftUI.
- Services MUST NOT construct or leak DTOs to the caller — they map to domain Models
  at the boundary.
- Services MUST NOT perform navigation or routing.
- A Service that only forwards calls with no logic is a smell — either add the
  orchestration that justifies it or let the consumer use the lower layer directly.

## Access Control
- Default every type, property, and method to the most restrictive access level.
- Use `private` for implementation details, `fileprivate` only when two types in the
  same file must share, `internal` (implicit) for module API.
- Keep injected dependencies (`NetworkClient`, storage, other services) as
  `private let`.
- Expose read-only state with `private(set)`; prefer `private let` over `private var`.

## Type Declaration Rules
- Each Service is a protocol plus a `final class` (or `struct`/`actor`) concrete
  implementation.
- Every `class` not intended for inheritance MUST be `final class`.
- Use an `actor` when the Service owns mutable state accessed concurrently; otherwise
  prefer a stateless `final class`/`struct`.
- Never use a `class`/`struct` with a `private init` as a namespace — use a case-less
  `enum` for pure static grouping.

## MARK Organization (mandatory in every file you touch)
Reorganize every modified file to this order:
```swift
// MARK: - Protocol
// MARK: - Dependencies
// MARK: - Initialization
// MARK: - Public API
// MARK: - Business Logic
// MARK: - Mapping
// MARK: - Private
```
No exceptions — any file edited in this layer must conform before you finish. (Not
every file contains every section; use only the relevant ones and keep the order.)

## Naming
- Types end in their role: `...Service` (protocol and implementation, e.g.
  `AuthService` / `DefaultAuthService`), `...Repository` where the type is
  persistence-centric.
- Methods name the use case as a phrase (`refreshSession()`, `loadProfile(for:)`);
  avoid `get`/`do` prefixes.
- No abbreviations except industry-standard (`URL`, `ID`, `JSON`).

## Swift 6 / iOS 16.6 Best Practices (2026)
- Service APIs are `async throws`; do not use completion handlers.
- Use typed throws (`throws(SomeServiceError)`) where the error domain is closed.
- Mark Services and their domain Models `Sendable`; use `actor` for shared mutable state.
- Services MUST NOT be `@MainActor` by default — they run off the main actor; the
  consuming ViewModel/Store hops to main. Annotate `@MainActor` only when a Service
  genuinely must touch main-actor state, and justify it.
- No force-unwrap (`!`), force-cast (`as!`), or force-try (`try!`) — `guard`, `if let`,
  or throw instead.
- Use `Result` only at boundaries where `throws` is impractical.

## Clean Code
- One Service, one bounded responsibility (auth, profile, catalog...). No `AppService`
  god object.
- One method, one use case; keep orchestration readable, extract steps into private
  methods.
- No magic literals — hoist thresholds, keys, and defaults to `private static let`
  constants.
- Map network/storage errors into a Service-owned error type; never leak
  `NetworkError` or raw storage errors to the Presentation layer unchanged unless
  that is deliberate.
- Guard early, return early; avoid nested pyramids.
- Every `public` API carries a `///` doc comment stating its contract, parameters,
  and thrown error cases.

## Testability
- The Service protocol MUST be mockable; ViewModels are tested against a mock Service.
- The concrete Service is tested with mocked `NetworkClient` and storage — no real
  network or disk.
- All dependencies are injected via init; nothing is accessed through a global
  singleton.

## Prohibited
- Catch-all files (`AppService.swift`, `Manager.swift` god objects).
- Singletons (`.shared`) — prefer an injectable, protocol-typed Service.
- Concrete dependencies hardcoded inside the Service instead of injected.
- References to a View, ViewModel/Store, or Router; any SwiftUI import.
- Leaking DTOs or raw lower-layer error types to the caller.
- View state, navigation, or persistence *mechanics* (the how belongs in Storage).