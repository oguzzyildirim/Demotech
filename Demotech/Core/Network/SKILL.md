# Core/Network — SKILL.md

## Purpose
`Core/Network` is the layer that encapsulates HTTP communication: endpoint
definition, request building, sending, decoding responses, and mapping network
errors to domain errors. This layer is concerned with *transport only* — business
logic, view state, and persistence do not live here. Converting a DTO to a domain
model sits at the boundary (mapping), but making business decisions does not.

## Layer Boundaries (Core siblings)
Respect the folder split — do not blur it:
- **Models/** — Network defines request/response DTOs here or within itself, but
  MUST NOT make domain Models depend on Network.
- **Services/** — Services *consume* Network, never the reverse. Network MUST NOT
  depend on a Service.
- **Storage/** — Network does not cache or persist; it decodes and returns.
  Persistence is Storage's job.
- **Helpers/**, **Utilities/**, **Extensions/** — Network may use these (e.g. a
  `JSONDecoder` configuration), but MUST NOT leak network logic into them.

Dependency direction points downward only. Network referencing a ViewModel, View,
or Router is a bug.

## Scope & Boundaries
- The Network layer MUST NOT be `@Observable` or `ObservableObject`, and MUST NOT
  hold view state.
- Network MUST NOT import SwiftUI.
- Endpoint definitions MUST be data — side-effect-free and testable (path, method,
  query, body modeled with `enum`/`struct`).
- Sending a request and building it are separate responsibilities: request building
  is pure; sending is `async throws`.
- Raw `URLSession` calls stay behind an abstraction (a `NetworkClient` protocol);
  consumers depend on the protocol, not on concrete `URLSession`.
- All network errors are mapped to a single domain error type (`NetworkError`); raw
  `URLError` is never leaked to the caller.

## Access Control
- Default every type, property, and method to the most restrictive access level.
- Use `private` for implementation details, `fileprivate` only when two types in the
  same file must share, `internal` (implicit) for module API.
- Keep dependencies like `URLSession`, decoder, and base URL as `private let` and
  inject them via init.
- Expose read-only configuration with `private(set)`.

## Type Declaration Rules
- Prefer `struct` or `enum` over `class`; endpoints and DTOs MUST be `struct` or `enum`.
- Every `class` not intended for inheritance MUST be `final class`.
- Model endpoint groups with a case-less `enum` namespace or `enum` cases — never a
  class with a `private init`.
- `NetworkClient` is a protocol; the concrete implementation is `final` and injectable.

## MARK Organization (mandatory in every file you touch)
Reorganize every modified file to this order:
```swift
// MARK: - Endpoint / Request
// MARK: - Response Models (DTO)
// MARK: - Properties / Dependencies
// MARK: - Initialization
// MARK: - Public API
// MARK: - Request Building
// MARK: - Response Handling / Decoding
// MARK: - Error Mapping
// MARK: - Private
```
No exceptions — any file edited in this layer must conform before you finish. (Not
every file contains every section; use only the relevant ones and keep the order.)

## Naming
- Types end in their role: `...Endpoint`, `...Request`, `...Response`, `...DTO`,
  `...Client`, `...Error`.
- DTO fields mirror the API contract; if the Swift-side name differs, map with
  `CodingKeys` instead of distorting the field name.
- Methods read as phrases at the call site; avoid `get`/`do` prefixes.
- No abbreviations except industry-standard (`URL`, `ID`, `JSON`, `HTTP`).

## Swift 6 / iOS 16.6 Best Practices (2026)
- Network calls are `async throws`; do not use completion handlers.
- Use typed throws (`throws(NetworkError)`) where practical, since the error domain
  is closed.
- Mark cross-actor types `Sendable`; `NetworkClient` and DTOs MUST be `Sendable`.
- Network calls MUST NOT be bound to an actor or `@MainActor` — they run in the
  background; the calling layer hops to main with the result.
- No force-unwrap (`!`), force-cast (`as!`), or force-try (`try!`); use `guard`/throw
  even when building a `URL`.
- Use `Result` only at boundaries where `throws` is impractical (e.g. bridging a
  legacy API).

## Clean Code
- One function, one job: request building, sending, decoding, and error mapping are
  separate methods.
- No magic literals — base URL, paths, header keys, and timeout values move to
  `private static let` constants or an enum namespace.
- `JSONDecoder`/`JSONEncoder` configuration is defined once, not rebuilt per call.
- HTTP status-code handling is centralized; each endpoint does not interpret status
  on its own.
- Guard early, return early; avoid nested pyramids.
- Every `public` API carries a `///` doc comment stating its contract, parameters,
  and thrown `NetworkError` cases.

## Testability
- The `NetworkClient` protocol MUST be mockable; tests never hit the real network.
- Endpoint building (URL, method, headers, body) is purely testable without networking.
- Decoding and error mapping are tested against fixed JSON fixtures.
- `URLSession` and base URL are injected, never accessed globally.

## Prohibited
- Catch-all files (a `NetworkManager.swift` god object, an `API.swift` dumping ground).
- Singletons (`.shared`) — prefer an injectable `NetworkClient`.
- References to a View, ViewModel/Store, or Router.
- Business logic, view state, persistence, or navigation.
- Leaking raw `URLError`/`URLSession` types to the caller.
- Rebuilding the decoder/encoder on every request or hardcoding URL string literals.