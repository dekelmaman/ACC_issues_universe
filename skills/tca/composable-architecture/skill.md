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
