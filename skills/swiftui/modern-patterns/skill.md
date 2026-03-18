# Modern SwiftUI Patterns

## Required Capabilities
- SwiftUI view composition
- @Observable macro usage
- Binding derivation patterns
- Async/await in SwiftUI context

## Inputs
- Feature UI requirements
- Model/state management needs
- Binding requirements

## Outputs
- Modern SwiftUI views following best practices
- @Observable model classes
- Properly derived bindings

## Patterns & Rules

### Action Closures in Views

```swift
struct CounterView: View {
  @State var count = 0
  var body: some View {
    Form {
      Button("Refresh") { refreshButtonTapped() }
    }
  }

  private func refreshButtonTapped() {
    ...
  }
}
```

- **DO** move multiline logic out of action closures into private methods on the view
- **DO** name methods after the user action (e.g. `refreshButtonTapped`, NOT `fetchNewData`)
- **DO NOT** define private view methods for action closures that are a single line

### Custom Binding Rules

- **ALWAYS** define helpers on a binding's `Value` type to derive new bindings using dynamic member lookup
- **NEVER** use `Binding.init(get:set:)` to derive bindings

Correct approach:

```swift
extension Optional {
  var isPresented: Bool {
    get { self != nil }
    set {
      guard !newValue else { return }
      self = nil
    }
  }
}

@Binding var item: Item?

MyView()
  .sheet(isPresented: $item.isPresented) { ... }
```

### @Observable Model Patterns

Use the `@Observable` macro for model classes:

```swift
@Observable
@MainActor
final class CounterModel {
  var count = 0
  @ObservationIgnored
  @Dependency(\.apiClient) var apiClient

  func incrementButtonTapped() {
    count += 1
  }

  func fetchData() async throws {
    let data = try await apiClient.fetch()
    // update state
  }
}
```

#### Observable Model Rules

- **DO** use `@Observable` macro on classes (not structs)
- **DO** annotate with `@MainActor`
- **DO** focus each model on one feature/screen
- **DO** name methods after the user action (e.g. `refreshButtonTapped`)
- **DO** make async methods `async` -- call them with `Task` in the view, NOT in the model
- **DO** use `@Dependency` with `@ObservationIgnored` for dependencies
- **DO** present optional models with `.sheet(item:)` for state-driven navigation

#### Task in View, Not Model

```swift
struct MyView: View {
  @State var model = MyModel()

  var body: some View {
    Button("Fetch") {
      Task { try await model.fetchData() }
    }
  }
}
```

- **DO NOT** create `Task` inside the model -- let the view own the task lifecycle

#### Presenting Optional Models

```swift
@Observable class ParentModel {
  var detail: DetailModel?

  func detailButtonTapped() {
    detail = DetailModel()
  }
}

// In view:
.sheet(item: $model.detail) { detailModel in
  DetailView(model: detailModel)
}
```
