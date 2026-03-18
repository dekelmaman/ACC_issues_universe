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
