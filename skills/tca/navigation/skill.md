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
