# TCA Testing

## Required Capabilities
- TCA TestStore usage
- Dependency injection for deterministic tests
- CustomDump assertions (expectNoDifference, expectDifference)
- Snapshot testing with record mode
- Swift Testing framework (@Test)

## Inputs
- TCA features (Reducers) to test
- Expected state mutations and effects
- Snapshot subjects (views, data structures)

## Outputs
- Deterministic, exhaustive TCA tests
- Dependency-controlled test environments
- Snapshot test suites

## Patterns & Rules

### TestStore Patterns

Use `TestStore` to exhaustively test TCA features by sending actions and asserting state changes:

```swift
@Test func incrementDecrement() async {
  let store = TestStore(initialState: Counter.State()) {
    Counter()
  }

  await store.send(.incrementButtonTapped) {
    $0.count = 1
  }
  await store.send(.decrementButtonTapped) {
    $0.count = 0
  }
}
```

### Controlling Dependencies for Deterministic Tests

Override dependencies using the `.dependencies` test trait:

```swift
@Test(
  .dependencies {
    $0.date.now = Date(timeIntervalSince1970: 1234567890)
    $0.uuid = .incrementing
  }
)
func example() async {
  let store = TestStore(initialState: Feature.State()) {
    Feature()
  }
  // Test with deterministic date and UUID
}
```

Or override on the store directly:

```swift
let store = TestStore(initialState: Feature.State()) {
  Feature()
} withDependencies: {
  $0.apiClient = MockAPIClient()
}
```

### Assertion Rules: expectNoDifference

Use `expectNoDifference` for static equality checks -- NOT `XCTAssertEqual` or `#expect(lhs == rhs)`:

```swift
expectNoDifference(actualValue, expectedValue)
```

- **DO NOT** use legacy `XCTAssertNoDifference`
- Best for static, expected-state tests (not action/behavior tests)

### Assertion Rules: expectDifference

Use `expectDifference` for mutation/action tests:

```swift
expectDifference(model.counters) {
  model.incrementButtonTapped(counter: model.counters[0])
} changes: {
  $0[0].count = 1
}
```

- **DO NOT** use legacy `XCTAssertDifference`
- **ONLY** use `expectDifference` to assert against value types
- **DO** assert against as much feature state as possible with no transformations
- **ALWAYS** mutate the smallest changes possible (e.g. `$0[2].count = 42`, NOT replacing the entire array)
- **DO NOT** replace entire data types or array elements unless ALL properties changed
- **ALWAYS** reassign simple values (`$0.count = 2`, NOT `$0.count += 1`) -- avoid logic in assertions
- **ALWAYS** insert/remove complex values rather than recreating entire collections
- **DO** prefer `expectDifference` over `expectNoDifference` when asserting mutations caused by an action

### customDump for Debugging

Print values for debugging:

```swift
customDump(value)              // to console
String(customDumping: value)   // to string
```

Diff two values:

```swift
if let difference = diff(lhs, rhs) {
  print(difference)
}
```

### Snapshot Testing Rules

- **DO NOT** write the content of inline snapshot trailing closures manually -- run tests in record mode to capture snapshots
- **DO NOT** update inline snapshot content manually -- use record mode
- **DO NOT** interpolate values into trailing closure strings
- Use record mode to generate fresh snapshots, then verify they match on subsequent runs

### Customizing Dump Format

Conform to one of:
- `CustomDumpRepresentable` -- override dump to another value
- `CustomDumpStringConvertible` -- override dump with a raw string
- `CustomDumpReflectable` -- override dump with a custom tree structure

```swift
extension ID: CustomDumpRepresentable {
  var customDumpValue: Any { rawValue }
}
```
