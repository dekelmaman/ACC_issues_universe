# Case Paths

## Required Capabilities
- Swift enum manipulation and pattern matching
- Macro-based code generation understanding
- Generic programming over enum types
- Dynamic member lookup patterns

## Inputs
- Enum types needing ergonomic access
- Pattern matching requirements
- Generic algorithm needs over enum cases

## Outputs
- @CasePathable annotated enums
- Ergonomic enum access patterns
- Generic embed/extract operations

## Patterns & Rules

### Adding Key Paths to an Enum

Annotate with `@CasePathable`:

```swift
@CasePathable enum LoadState<Value> {
  case idle
  case loading(progress: Float)
  case loaded(Value)
  case failed(any Error)
}
```

### Checking if an Enum is a Certain Case

Use `.is(\.case)`:

```swift
loadState.is(\.idle)      // Bool
loadState.is(\.loading)   // Bool
```

### Modifying an Associated Value

Use `.modify(\.case)`:

```swift
loadState.modify(\.progress) { $0 += 0.1 }
```

### Ergonomic Associated Value Access with @dynamicMemberLookup

Step 1 -- Apply `@dynamicMemberLookup`:

```swift
@CasePathable
@dynamicMemberLookup
enum LoadState<Value> {
  case idle
  case loading(progress: Float)
  case loaded(Value)
  case failed(any Error)
}
```

Step 2 -- Use dot-syntax to access associated values as optionals:

```swift
loadState.loading  // Optional<Float>
loadState.loaded   // Optional<Value>
```

### compactMap on Arrays

Use `compactMap` with case key paths to extract associated values from collections:

```swift
loadStates.compactMap(\.loaded)  // [Value]
```

### Iterating All Case Key Paths

Use `allCasePaths`:

```swift
for caseKeyPath in LoadState<String>.allCasePaths {
  _: PartialCaseKeyPath<LoadState<String>> = caseKeyPath
}
```

### Getting the Case Key Path of an Instance

```swift
LoadState.allCasePaths[loadState]  // PartialCaseKeyPath<LoadState<String>>
```

### Checking if Two Enums are the Same Case

```swift
LoadState.allCasePaths[lhs] == LoadState.allCasePaths[rhs]
```

### Generically Embedding a Value

Invoke a case key path with a value:

```swift
let caseKeyPath: CaseKeyPath<Root, Value>
caseKeyPath(value)  // Root
```

### Generically Extracting a Value

Use `[case:]` subscript:

```swift
let caseKeyPath: CaseKeyPath<Root, Value>
root[case: caseKeyPath]  // Value?
```

### Fix: "inaccessible due to 'private' protection level"

- **DO NOT** define `private` case-pathable enums in a nested scope
- **DO** use `fileprivate` instead:

```swift
struct Feature {
  @CasePathable fileprivate enum Action {
    ...
  }
}
```

### Adding Case Key Paths to External Types

Manually extend the type with `CasePathable` and `CaseIterable`, defining a nested `AllCasePaths` struct with a property per case using `AnyCasePath`:

```swift
extension LoadState: CasePathable, CaseIterable {
  struct AllCasePaths: CasePathReflectable, Sendable {
    var idle: AnyCasePath<LoadState, Void> {
      AnyCasePath { .idle } extract: {
        guard case .idle = $0 else { return nil }
        return
      }
    }
    var loading: AnyCasePath<LoadState, Float> {
      AnyCasePath { .loading(progress: $0) } extract: {
        guard case .loading(let progress) = $0 else { return nil }
        return progress
      }
    }
    // ... one property per case
  }
  static var allCasePaths: AllCasePaths { AllCasePaths() }
}
```
