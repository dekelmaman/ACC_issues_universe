# Identified Collections

## Required Capabilities
- Swift collection manipulation
- Understanding of Identifiable protocol
- O(1) lookup patterns for collections

## Inputs
- Identifiable element types
- Collection operations (lookup, remove, iterate)

## Outputs
- IdentifiedArrayOf declarations
- Efficient collection access patterns

## Patterns & Rules

### Creating an Identified Array

Step 1 -- Ensure the element conforms to `Identifiable`:

```swift
struct User: Identifiable {
  let id: UUID
  var name: String
}
```

Step 2 -- Construct with an array literal:

```swift
let users: IdentifiedArrayOf<User> = [
  User(id: UUID(), name: "Alice"),
  User(id: UUID(), name: "Bob"),
]
```

### O(1) Lookup by ID

```swift
users[id: userID]  // Optional<User>
```

### Removing an Element by ID

Without needing the element:

```swift
users[id: userID] = nil
```

With the removed element:

```swift
if let removedUser = users.remove(id: userID) {
  ...
}
```

### Iterating Over IDs

```swift
for id in users.ids {
  ...
}
```

### Key Rules

- **DO** use `IdentifiedArrayOf<Element>` where `Element: Identifiable` for collections in TCA state
- **DO** use `users[id: userID]` for O(1) lookups instead of `users.first(where:)`
- **DO** use array literal initialization for simplicity
- Use `IdentifiedArrayOf` in TCA parent state when managing collections of child features with `.forEach`
