# Trinity's Memory

## Platform Patterns Discovered

### Mobius: State vs RenderData separation (Android)

In the MemberPicker Mobius architecture, there are two distinct layers:
- **`State`** — internal truth, holds raw data and configuration. Lives inside the Mobius loop.
- **`RenderData`** — what the UI actually needs. Produced by `StateMapper` from `State`.

**Rule:** Never add UI-behavioral fields (like `isEnabled`, `isVisible`) to sealed classes inside `RenderData` sub-models (e.g., `TopBarActionButton`). Instead, add them as **top-level fields on `RenderData`** itself.

**Example — correct pattern:**
```kotlin
data class RenderData(
    val isMembersEnabled: Boolean = true,   // ✅ top-level on RenderData
    val isClearEnabled: Boolean = true,      // ✅ top-level on RenderData
    val topBarActionButton: TopBarActionButton = TopBarActionButton.Hidden,  // clean render model
)
```

**Anti-pattern:**
```kotlin
data class Clear(
    val selectedMembersSize: Int,
    val isEnabled: Boolean = true  // ❌ behavioral state inside a render sub-model
) : Visible(...)
```

**Why:** Keeps render sub-models (like `TopBarActionButton`) as pure display descriptors. Behavioral state that the UI needs to gate interactions belongs at the `RenderData` level, consistent with existing patterns like `isMembersEnabled`.

*Source: SCCOM-32891 code review feedback from Ronny.*

## Skill Gaps Encountered

## Learnings
