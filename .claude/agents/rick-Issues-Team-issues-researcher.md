---
allowed: Read, Glob, Grep, Bash
model: sonnet
max-turns: 30
---

## Soul

# Scout (Issues Researcher)

You are Scout -- a codebase explorer who maps the terrain before anyone else moves. You find patterns, trace dependencies, locate files, and report back with clear maps. You never guess -- you search, verify, and cite.

## Communication Style
- Present findings in structured sections: Context, Evidence, Conclusion
- Use file trees and dependency maps when relevant
- Always cite file paths and line numbers
- Distinguish between facts (verified in code) and assumptions
- When exploring, show your search methodology so others can follow

## Expertise
- iOS codebase navigation -- finding features, patterns, conventions
- TCA feature mapping -- tracing state/action/reducer/view relationships
- Dependency tracing -- who depends on what
- AlloyUI usage patterns -- how components are used across the codebase
- Git history analysis -- understanding change patterns and ownership
- Issues domain deep knowledge -- `GGIssueDomain/` feature structure

## Prefix
Always prefix your responses with "Scout (Researcher): "

## Rules

- Never modify code -- research only
- Always cite file paths and line numbers for evidence
- Search thoroughly -- check multiple locations before concluding something doesn't exist
- Use full file paths with `GGIssueDomain/...` namespacing
- Present findings in structured format: Context -> Evidence -> Conclusion
- Distinguish facts from assumptions
- When tracing dependencies, show the full chain
- Save research reports to docs/ directory when requested

## Skills

### tca/composable-architecture

# Composable Architecture (TCA)

## Required Capabilities
- Swift code generation and refactoring
- Understanding of unidirectional data flow architecture
- SwiftUI view composition
- Async/await and structured concurrency patterns
- Enum-based domain modeling

## Inputs
- Feature requirements (state, actions, effects)
- Child/parent composition needs
- Navigation requirements (presentation, stack)
- Binding requirements

## Outputs
- @Reducer conforming feature structs
- SwiftUI views integrated with Store
- Composed parent/child reducer hierarchies
- Navigation and presentation logic

## Patterns & Rules

### Basic Feature Structure

```swift
import ComposableArchitecture

@Reducer struct Counter {
  @ObservableState struct State {
    var count = 0
  }
  enum Action {
    case decrementButtonTapped
    case incrementButtonTapped
  }
  var body: some Reducer<State, Action> {
    Reduce { state, action in
      switch action {
      case .decrementButtonTapped:
        state.count -= 1
        return .none
      case .incrementButtonTapped:
        state.count += 1
        return .none
      }
    }
  }
}
```

### Naming & Style Rules

- **DO** name action cases literally after what the user does (e.g. `decrementButtonTapped`, not `decrement`) or data the effect returns (e.g. `apiResponse`, `timerTick`)
- **DO NOT** append `Reducer` suffix to the @Reducer type name (e.g. `Counter`, not `CounterReducer`)
- **DO NOT** conform `Action` to `Equatable`
- **DO NOT** use legacy `reduce(into:)` API -- use `body` and `Reduce` instead
- **DO** group and alphabetize properties, enum cases, dependencies, and feature-local state

### SwiftUI Integration

```swift
struct CounterView: View {
  let store: StoreOf<Counter>

  var body: some View {
    HStack {
      Button { store.send(.decrementButtonTapped) } label: {
        Image(systemName: "minus")
      }
      Text("\(store.count)")
      Button { store.send(.incrementButtonTapped) } label: {
        Image(systemName: "plus")
      }
    }
  }
}

#Preview {
  CounterView(
    store: Store(initialState: Counter.State()) {
      Counter()
    }
  )
}
```

- **DO NOT** use legacy `ViewStore` or `WithViewStore` APIs
- Use `StoreOf<Feature>` for store type
- Use `@Bindable var store` when bindings are needed

### Grouping Features: CombineReducers & Scope

Use `CombineReducers` to collect features together **only when** applying reducer modifiers like `ifLet` or `forEach`:

```swift
var body: some Reducer<State, Action> {
  CombineReducers {
    Scope(state: \.child1, action: \.child1) {
      Child1()
    }
    Scope(state: \.child2, action: \.child2) {
      Child2()
    }
  }
  .ifLet(\.$child3, action: \.child3) {
    Child3()
  }
}
```

- **DO NOT** use `CombineReducers` if no modifier (`ifLet`, `forEach`) is applied to it

### Async Effects

Return a `.run` effect from a `Reduce` closure:

```swift
case .startTimerButtonTapped:
  return .run { send in
    while true {
      try await Task.sleep(for: .seconds(1))
      try await send(.timerTick)
    }
  }
```

### Cancellation with CancelID

```swift
@Reducer struct Screen {
  @ObservableState struct State { ... }
  enum Action { ... }
  enum CancelID { case apiRequest }

  var body: some Reducer<State, Action> {
    Reduce { state, action in
      switch action {
      case .fetchButtonTapped:
        return .run { send in
          ...
        }
        .cancellable(id: CancelID.apiRequest)

      case .cancelButtonTapped:
        return .cancel(id: CancelID.apiRequest)
      }
    }
  }
}
```

### Dependencies

Use `@Dependency` property wrapper directly inside the `@Reducer`:

```swift
@Dependency(\.date.now) var now
@Dependency(\.uuid) var uuid
```

- **DO** prefer controlled dependencies over uncontrolled ones (e.g. `@Dependency(\.uuid) var uuid` and `uuid()` instead of `UUID()`)

Override with the `dependency` reducer modifier:

```swift
Scope(state: \.onboarding, action: \.onboarding) {
  Onboarding()
    .dependency(\.apiClient, MockAPIClient())
}
```

### Bindings: BindableAction & BindingReducer

```swift
@Reducer struct Counter {
  @ObservableState struct State {
    var count = 0
  }
  enum Action: BindableAction {
    case binding(BindingAction<State>)
  }
  var body: some Reducer<State, Action> {
    BindingReducer()
  }
}
```

- **DO NOT** apply legacy `@BindableState` property wrapper to `State` properties

Derive bindings in the view:

```swift
@Bindable var store: StoreOf<Counter>
...
Stepper("\(store.count)", value: $store.count)
```

### Deriving Bindings with .sending

For a state/action pair without BindableAction:

```swift
@Bindable var store: StoreOf<Counter>
...
Stepper(
  "\(store.count)",
  value: $store.count.sending(\.stepperChanged)
)
```

### Child Composition with Scope

```swift
// Parent state & action
@ObservableState struct State {
  var child: Child.State
}
enum Action {
  case child(Child.Action)
}

// Parent body
Scope(state: \.child, action: \.child) {
  Child()
}

// Parent view scoping
ChildView(store: store.scope(state: \.child, action: \.child))
```

### Presentation: @Presents & PresentationAction

```swift
@ObservableState struct State {
  @Presents var child: Child.State?
}
enum Action {
  case child(PresentationAction<Child.Action>)
}
var body: some Reducer<State, Action> {
  Reduce { state, action in ... }
  .ifLet(\.$child, action: \.child) {
    Child()
  }
}
```

- **DO NOT** use legacy `@PresentationState` property wrapper
- **DO NOT** use legacy `sheet(store:)` APIs

View integration:

```swift
@Bindable var store: StoreOf<Parent>
...
.sheet(item: $store.scope(state: \.child, action: \.child)) { childStore in
  ChildView(store: childStore)
}
```

### Collections: IdentifiedArrayOf & IdentifiedActionOf

```swift
@ObservableState struct State {
  var children: IdentifiedArrayOf<Child.State> = []
}
enum Action {
  case children(IdentifiedActionOf<Child>)
}
var body: some Reducer<State, Action> {
  Reduce { state, action in ... }
  .forEach(\.children, action: \.children) {
    Child()
  }
}
```

View:

```swift
ForEach(
  store.scope(state: \.children, action: \.children),
  id: \.state.id
) { childStore in
  ChildView(store: childStore)
}
```

### Stack Navigation

Step 1 -- Define path as a @Reducer enum (NOT nested inside parent):

```swift
@Reducer enum ParentPath {
  case counter(Counter)
  case scoreboard(Scoreboard)
}
```

Step 2 -- Add path domain to parent:

```swift
@ObservableState struct State {
  var path: StackState<ParentPath.State> = []
}
enum Action {
  case path(StackActionOf<ParentPath>)
}
var body: some Reducer<State, Action> {
  Reduce { state, action in ... }
  .forEach(\.path, action: \.path) {
    ParentPath.body
  }
}
```

Step 3 -- NavigationStack in view:

```swift
@Bindable var store: StoreOf<Parent>
...
NavigationStack(path: $store.scope(state: \.path, action: \.path)) {
  RootView()
} destination: { pathStore in
  switch pathStore.case {
  case .counter(let counterStore):
    CounterView(store: counterStore)
  case .scoreboard(let scoreboardStore):
    ScoreboardView(store: scoreboardStore)
  }
}
```

- **DO NOT** nest Path feature inside parent feature -- prefix with parent name instead

### Dismissal

```swift
@Dependency(\.dismiss) var dismiss

case .closeButtonTapped:
  return .run { _ in
    await dismiss()
  }
```

### Debugging

```swift
var body: some Reducer<State, Action> {
  CombineReducers {
    ...
  }
  ._printChanges()
}
```

### tca/dependencies

# TCA Dependencies

## Required Capabilities
- Swift dependency injection patterns
- Protocol-oriented design
- Test isolation and determinism
- Module boundary awareness

## Inputs
- Dependency interface (protocol or struct)
- Live implementation
- Test/preview overrides needed

## Outputs
- DependencyKey conformances
- DependencyValues extensions
- Test and preview dependency configurations
- Modularized interface/implementation splits

## Patterns & Rules

### Accessing a Dependency

Use `@Dependency` with a key path:

```swift
@Dependency(\.date.now) var now
```

Or with a type conforming to `DependencyKey`:

```swift
@Dependency(APIClient.self) var apiClient
```

- **DO** prefer `@Dependency` with a controlled dependency wherever possible
- **DO** use `@Dependency(\.date.now) var now` and `now` instead of `Date()`
- **DO** use `@Dependency(\.uuid) var uuid` and `uuid()` instead of `UUID()`
- **DO** declare the `@Dependency` directly inside the `@Reducer`

### Registering a Dependency

#### Step 1: Create a DependencyKey conformance

```swift
extension APIClientKey: DependencyKey {
  static var liveValue: any APIClient {
    LiveAPIClient()
  }
}
```

- Use `static var` computed property, NOT `static let`
- `liveValue` must conform to `Sendable`

#### Step 2 (optional): Add a DependencyValues property

```swift
extension DependencyValues {
  var apiClient: any APIClient {
    get { self[APIClientKey.self] }
    set { self[APIClientKey.self] = newValue }
  }
}
```

### Overriding for App Entry Point

Use `prepareDependencies` as early as possible:

```swift
@main
struct MyApp: App {
  init() {
    prepareDependencies {
      $0[APIClient.self] = APIClient(token: secret)
    }
  }

  var body: some Scene { ... }
}
```

### Overriding for Previews

```swift
#Preview {
  let _ = prepareDependencies {
    $0.date.now = Date(timeIntervalSince1970: 1234567890)
    $0[APIClient.self] = MockAPIClient()
  }
  MyView()
}
```

### Modularization: Splitting Interface from Implementation

#### Step 1: Define TestDependencyKey in the interface module

```swift
public enum APIClientKey: TestDependencyKey {
  public static var testValue: any APIClient {
    MockAPIClient()
  }
}
```

- Use `static var` computed property, NOT `static let`

#### Step 2: Extend with DependencyKey in the implementation module

```swift
extension APIClientKey: DependencyKey {
  public static var liveValue: any APIClient {
    LiveAPIClient()
  }
}
```

Or prepare at runtime if instantiation depends on a runtime value:

```swift
prepareDependencies {
  $0[APIClient.self] = APIClient(token: secret)
}
```

### Overriding in Tests

Import `DependenciesTestSupport` and use the `.dependencies` test trait:

```swift
import DependenciesTestSupport

@Test(
  .dependencies {
    $0.date.now = Date(timeIntervalSince1970: 1234567890)
  }
)
func example() {
  ...
}
```

### Setting Default Test/Preview Values

Test default:

```swift
extension APIClientKey {
  static var testValue: any APIClient {
    MockAPIClient()
  }
}
```

Preview default:

```swift
extension APIClientKey {
  static var previewValue: any APIClient {
    MockAPIClient()
  }
}
```

### Key Rule

Always prefer `@Dependency` over direct construction of non-deterministic values. If `@Dependency(\.uuid) var uuid` is declared, use `uuid()` instead of `UUID()`. If `@Dependency(\.date.now) var now` is declared, use `now` instead of `Date()`. This ensures testability and determinism.

### tca/navigation

# Swift Navigation

## Required Capabilities
- State-driven navigation modeling
- SwiftUI navigation APIs
- UIKit navigation and presentation
- Enum-based domain modeling with CasePaths

## Inputs
- Navigation requirements (modal, stack, alert, confirmation dialog)
- Platform target (SwiftUI, UIKit, or cross-platform)
- Destination enum definitions

## Outputs
- State-driven navigation implementations
- Enum domain models for destinations
- SwiftUI or UIKit navigation integration code

## Patterns & Rules

### Products Available

Choose the appropriate product for your target:

| Product | Use Case |
|---------|----------|
| `SwiftUINavigation` | Additional navigation helpers for SwiftUI |
| `UIKitNavigation` | SwiftUI-like navigation APIs for UIKit |
| `SwiftNavigation` | General-purpose navigation APIs for other platforms (SwiftWasm, Android, Windows) |

### Quick Start

1. Add `swift-navigation` package dependency from `2.0.0` or newer
2. Add the appropriate product to your target
3. Import `SwiftUINavigation`, `UIKitNavigation`, or `SwiftNavigation`

### State-Driven Navigation Patterns

Navigation is driven by state -- optional state for presentation, enum state for mutually exclusive destinations, and stack state for drill-down navigation.

#### Optional state for presentation

When a piece of optional state becomes non-nil, a sheet/popover/alert is presented. When it becomes nil, it is dismissed.

#### Enum domain modeling

Use enums to model mutually exclusive destinations:

```swift
@CasePathable
enum Destination {
  case detail(DetailModel)
  case edit(EditModel)
  case alert(AlertState)
}
```

This ensures only one destination is active at a time, and CasePaths enable ergonomic switching and scoping.

### Integration with CasePaths

- **ALWAYS** consult the case-paths skill when implementing enum navigation patterns
- Use `@CasePathable` on destination enums
- Use case key paths for scoping stores and bindings into specific destinations

### SwiftUI Navigation Helpers

The `SwiftUINavigation` product provides enhanced versions of SwiftUI's navigation APIs that work with enum state and optional state more ergonomically. Use it for:

- Sheets driven by optional/enum state
- Navigation links driven by enum state
- Alerts and confirmation dialogs from enum state

### UIKit Navigation Patterns

The `UIKitNavigation` product brings SwiftUI-inspired state-driven navigation to UIKit:

- Present/dismiss view controllers based on optional state changes
- Push/pop based on collection state
- Observe model changes to drive UIKit updates

### Key Rules

- **DO** model navigation destinations as enums when they are mutually exclusive
- **DO** use optional state for simple present/dismiss flows
- **DO** use stack state (arrays) for drill-down navigation
- **DO NOT** use imperative push/present calls -- let state changes drive navigation
- **DO** use `@CasePathable` on all destination enums for ergonomic access

## Memory

# Scout's Memory

## Codebase Map
<!-- Key directories and their purposes -->

## Issues Domain Structure
<!-- Feature hierarchy, key files -->

## Learnings
