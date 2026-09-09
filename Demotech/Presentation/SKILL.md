# Presentation — SKILL.md

## Purpose
`Presentation` owns everything the user sees and interacts with: SwiftUI Views,
their ViewModels, reusable Components, and Routing. This layer renders state and
forwards user intent — it does not fetch data, contain business logic, or talk to
the network. All real work is delegated downward to Services.

## Sub-folder Roles (respect the split)
- **Common/** — shared app-level scaffolding (root views, shared containers). Not a
  dumping ground for one-off feature views.
- **Components/** — reusable, stateless, feature-agnostic UI building blocks
  (buttons, cards, rows). A Component MUST NOT depend on a Service, ViewModel, or
  Router; it is driven purely by its inputs and callbacks.
- **Features/** — one folder per feature, each containing its View(s) + ViewModel.
  A feature MUST NOT reach into another feature's internals; share via Components or
  Core.
- **Routing/** — navigation state and destination definitions. Routing decides
  *where*; Views and ViewModels never construct navigation stacks ad hoc.

## Layer Boundaries (downward only)
- Presentation *consumes* `Core/Services` behind protocols; it MUST NOT import
  `Core/Network` or `Core/Storage` directly.
- Views/ViewModels operate on domain Models — never DTOs.
- Presentation MUST NOT contain business logic, persistence, or transport concerns.
- A ViewModel referencing `URLSession`, a DTO, or a `NetworkClient` is a bug.

## Architecture Rules
- Each feature View is backed by an `@Observable` (Observation framework) ViewModel
  that is `@MainActor`.
- The View is a pure function of state: it reads from the ViewModel and sends intents
  back; it holds no business decisions.
- Local, view-only UI state (animation flags, focus, text-field edits) may live in
  the View via `@State`. Anything that outlives a single view render or is testable
  belongs in the ViewModel.
- The ViewModel depends on Service *protocols*, injected via init — never concrete
  Services, never `.shared`.
- Side effects (loading, saving) are `async` intents on the ViewModel; the View
  triggers them via `.task`/actions and only observes the resulting state.

## Access Control
- Default every type, property, and method to the most restrictive access level.
- ViewModel dependencies are `private let`; mutable state is `private(set) var` where
  the View only reads it.
- Prefer `private` for helper methods and subviews not used elsewhere.

## Type Declaration Rules
- Views are `struct`. ViewModels are `final class` marked `@Observable` and
  `@MainActor`.
- Every `class` not intended for inheritance MUST be `final class`.
- Routing destinations are modeled as an `enum` (`Hashable`/`Identifiable` as needed),
  not stringly-typed.
- Extract repeated view chunks into `Components` or `private var`/`@ViewBuilder`
  subviews — never copy-paste view trees.

## MARK Organization (mandatory in every file you touch)
Use the layout that matches the file's kind, and keep the order.

For a **SwiftUI View**:
```swift
// MARK: - View
// MARK: - Properties (State / ViewModel / Environment)
// MARK: - Body
// MARK: - Subviews
// MARK: - Actions
// MARK: - Private
```

For a **ViewModel**:
```swift
// MARK: - State
// MARK: - Dependencies
// MARK: - Initialization
// MARK: - Intents (Public API)
// MARK: - Business Coordination
// MARK: - Private
```

For a **Routing** type:
```swift
// MARK: - Destination
// MARK: - Router
// MARK: - Navigation State
// MARK: - Public API
// MARK: - Private
```
No exceptions — any file edited in this layer must conform before you finish. (Not
every file contains every section; use only the relevant ones and keep the order.)

## Naming
- Views end in `...View`; view models end in `...ViewModel`.
- Routing types end in `...Route`/`...Destination`/`...Router`.
- Intent methods name the user action as a phrase (`didTapSave()`, `loadProfile()`);
  avoid `get`/`do` prefixes.
- No abbreviations except industry-standard (`URL`, `ID`, `JSON`).

## Swift 6 / iOS 16.6 Best Practices (2026)
- Use the Observation framework (`@Observable`) for ViewModels; avoid legacy
  `ObservableObject`/`@Published` in new code.
- ViewModels are `@MainActor`; `async` intents `await` Service calls and the
  resulting state update lands on main automatically.
- Prefer `.task` over `.onAppear` for async work, and honor cancellation.
- Mark cross-actor value types `Sendable`.
- No force-unwrap (`!`), force-cast (`as!`), or force-try (`try!`) — `guard`,
  `if let`, or handle the error into view state instead.
- Drive lists/navigation with `Identifiable`/`Hashable` model types, not indices or
  strings.

## Clean Code
- The `body` stays readable — decompose into subviews once it grows past a screenful.
- No magic literals in views — spacing, sizes, and durations go to a design-token /
  constants source, not inline numbers scattered across views.
- No business logic in `body` or in computed view properties.
- Error and loading are explicit states the View renders, not silent failures.
- Guard early, return early in ViewModel intents; avoid nested pyramids.
- Every `public`/`internal` ViewModel intent carries a `///` doc comment stating what
  it does and the state it produces.

## Testability
- ViewModels are unit-testable in isolation against mocked Service protocols — no real
  network, no UI.
- Views stay thin so that logic under test lives in the ViewModel, not the `body`.
- Routing decisions are testable as pure state transitions on an `enum`.

## Prohibited
- Importing `Core/Network` or `Core/Storage`, or handling DTOs/`URLSession` in
  Presentation.
- Business logic, persistence, or transport in a View or ViewModel.
- Singletons (`.shared`) or concrete Services inside a ViewModel — inject protocols.
- Massive views: copy-pasted view trees, or a `body` carrying logic and layout for
  the whole screen.
- Components that depend on Services, ViewModels, or Routers.
- Ad-hoc navigation constructed inside views instead of going through Routing.
- Legacy `ObservableObject`/`@Published` in new feature code.