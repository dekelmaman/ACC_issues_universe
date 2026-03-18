# Android Technical Requirements Generator

Generate an implementation-quality technical requirements document from a PRD or feature spec. This skill produces a structured `requirements.md` that an Android developer can implement directly.

## When to Use
- A PRD, spec, or feature description has been provided
- The team needs a technical requirements document before implementation
- Translating product requirements into Android-specific architecture decisions

## How to Use
1. Read the provided PRD/spec
2. Fill in each section below using the Rules and Examples as guides
3. Output a complete `requirements.md` file

---

## Template Sections

### 1. Overview & Scope

```markdown
## Overview
<1-2 paragraph summary of the feature, what it does, and why>

## Platform & Scope
| Attribute       | Value |
|----------------|-------|
| Platform        | Android only |
| UI Framework    | Jetpack Compose |
| Component Library | Alloy (compose variants only) |
| Module Location | `android/<module>/src/main/java/com/plangrid/android/<module>/<feature>/` |
| Navigation      | Navigation 3 (`com.plangrid.android.components.compose.navigation`) |
| Out of Scope    | <list what's explicitly excluded> |
```

**Rules:**
- Always specify the exact module path where new files will live
- Always list what's out of scope to prevent scope creep
- Reference the source PRD/spec document

---

### 2. Architecture & State Management

Define the Mobius MVI types for this feature:

```kotlin
// <Feature>Mobius.kt — all types in one file
class <Feature>Mobius(...) : MobiusViewModel<Action, Effect, State, RenderData, Event>(...) {
    data class State(...)
    sealed class Action {
        sealed class View : Action()      // From UI
        sealed class Internal : Action()  // From Processor
    }
    sealed class Effect { ... }
    sealed class Event { ... }
    data class RenderData(...)
    class Factory @Inject constructor(...) : ViewModelProvider.Factory
}
```

**For each feature, enumerate:**
- All State fields with types and defaults
- All Action.View variants (user interactions)
- All Action.Internal variants (system responses)
- All Effects (async operations to perform)
- All Events (one-shot UI events: navigation, toasts, dialogs)
- All RenderData fields (what Compose consumes)

**Example (Quick Create):**
```
QuickCreateMobius : MobiusViewModel<Action, Effect, State, RenderData, Event>
+-- State           -- photos, selectedIndex, issueUid, type, description, placement, status
+-- Action.View     -- ScreenOpened, PhotoSelected, TypeSelected, ReviewTapped, CloseTapped...
+-- Action.Internal -- DefaultTypeResolved, IssueCreated, IssueDeleted, Error...
+-- Effect          -- CreateIssue, DeleteIssue, SaveDescription, ImportPhotos, TrackEvent...
+-- Event           -- NavigateToIssueDetails, DismissScreen, LaunchCamera, ShowError...
+-- RenderData      -- What Compose UI consumes. No business logic.
```

---

### 3. Package Structure

```
<feature_name>/
+-- <Feature>Fragment.kt                    # Fragment entry point (Anvil DI)
+-- <Feature>StickyDefaults.kt              # SharedPreferences wrapper (if applicable)
+-- <Feature>PlacementType.kt               # Enums/sealed classes (if applicable)
|
+-- analytics/
|   +-- <Feature>Analytics.kt              # Dedicated analytics tracker
|
+-- composables/
|   +-- <Feature>Screen.kt                 # Main composable container
|   +-- <Feature>ScreenHost.kt             # Compose navigation host
|   +-- <Feature>Toolbar.kt                # Top app bar
|   +-- <Feature><Section>.kt              # Per-section composables
|   +-- <Feature>BottomSheet.kt            # Bottom sheets (if applicable)
|
+-- mobius/
|   +-- <Feature>Mobius.kt                 # State, Action, Effect, Event, RenderData, Factory
|   +-- <Feature>Updater.kt               # Pure state mutations
|   +-- <Feature>Processor.kt             # Async effect handlers (RxJava)
|   +-- <Feature>StateMapper.kt           # State -> RenderData
|
+-- <domain_concern>/                       # e.g., photos/, attachments/
    +-- <Feature><Concern>Storage.kt
    +-- <Feature><Concern>Importer.kt
```

---

### 4. UI Components

For each screen/section, specify:
- Which Alloy component to use
- Key props and configuration
- Tap handlers and their corresponding Actions

```markdown
### <Screen/Section Name>

| Element | Alloy Component | Props | Action |
|---------|----------------|-------|--------|
| Title input | AlloyInputField.AlloyInputField | label, onTap | Action.View.TitleTapped |
| Submit | AlloyButton.AlloyButtonSolid | text, onClick, testTag | Action.View.SubmitTapped |
```

---

### 5. Strings

List ALL user-facing strings with their resource names:

```xml
<!-- strings.xml -->
<!-- Feature: <Feature Name> -->
<string name="<feature>_title">Screen Title</string>
<string name="<feature>_<field>_label">Field Label</string>
<string name="<feature>_<field>_hint">Placeholder text</string>
<string name="<feature>_<button>_action">Button Text</string>

<!-- Plurals -->
<plurals name="<feature>_<item>_count">
    <item quantity="one">%d item</item>
    <item quantity="other">%d items</item>
</plurals>
```

---

### 6. Navigation

```xml
<!-- navigation_<module>.xml -->
<fragment
    android:id="@+id/fragment_<feature>"
    android:name="com.plangrid.android.<module>.<feature>.<Feature>Fragment">
    <argument android:name="argName" app:argType="string" />
</fragment>

<action
    android:id="@+id/action_<source>_to_<feature>"
    app:destination="@id/fragment_<feature>" />
```

Specify:
- Entry points (where users navigate FROM)
- Arguments passed to the fragment
- Internal navigation routes (if multi-screen feature)
- Back press behavior

---

### 7. Feature Flags

```kotlin
val <featureName>: Boolean by lazy {
    projectFlags.isEnabled(<FLAG_KEY>)
}
```

Specify:
- Flag name and key
- Visibility conditions (if multi-condition, define a VisibilityHelper)
- Default value when flag is off

---

### 8. Analytics Events

| Event Name | Trigger | Parameters |
|-----------|---------|------------|
| `<feature>.opened` | Screen opens | common params |
| `<feature>.action_clicked` | Action button tapped | issueUid, actionType |

Include ALL events. Every event missed here will be missed in implementation.

---

### 9. Responsive Design

Specify phone vs tablet differences:

| Aspect | Phone | Tablet |
|--------|-------|--------|
| Thumbnail size | 80dp | 108dp |
| Action label | "Short" | "Longer descriptive" |
| Layout | Single column | Side-by-side |

---

### 10. Error Handling

| Scenario | Handling | User Impact |
|----------|----------|-------------|
| Network unavailable | Local-first (SQLDelight) | Transparent; syncs later |
| Repository call fails | `Internal.Error` -> `ShowError` toast | User can retry |
| Permission denied | System permission dialog | Show settings redirect |
| Input exceeds limit | Alloy `maxCharacters` | Input capped in UI |

---

### 11. Automation Test Tags

```kotlin
data object <FeatureName> {
    const val SCREEN = "<feature>_screen"
    const val TOOLBAR = "<feature>_toolbar"
    const val CLOSE_BUTTON = "<feature>_close_button"
    const val <ELEMENT> = "<feature>_<element>"
    fun indexed<Item>(index: Int) = "$<ITEM_PREFIX>$index"
    private const val <ITEM_PREFIX> = "<feature>_<item>_"
}
```

List EVERY interactive element that needs a tag.

---

### 12. Testing Plan

For each Mobius component, list what to test:

**Updater Tests:**
- `<Action> sets <state field> and emits <effects>`
- `<Action> when <condition> transitions to <state>`

**Processor Tests:**
- `<Effect> calls <repository method> and emits <Action.Internal>`
- `<Effect> on error emits Internal.Error`

**StateMapper Tests:**
- Phone vs tablet output differences
- Edge cases (empty state, loading state)

**Integration Tests:**
- Full flow: ScreenOpened -> data loads -> user acts -> result
