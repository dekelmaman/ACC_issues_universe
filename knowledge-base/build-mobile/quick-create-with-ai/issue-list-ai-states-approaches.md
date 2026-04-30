# Issue List AI Section + Row States — 3 Android UI Approaches

> **Source**: `knowledge-base/build-mobile/quick-create/research-and-poc-suggestions.md` Area 2
> **Goal**: Match iOS visual design for the AI-suggested-issues section on the Issues List screen
> **Date**: 2026-04-16
> **Status**: POC planning — user selects one before implementation

---

## Background

### iOS (shipped reference)

Two relevant files drive the iOS experience:

- `IssuesListFeature.swift:170-205` — splits the list into two arrays: AI issues (where `suggestionsStatusMap[issue.uid] != nil`) and regular issues. Renders an `AISuggestionsHeaderView` with the `issues.list.ai_suggestions_heade` string ("Generate with AI") above the AI group, then a divider, then the regular group.
- `ListSingleIssueView.swift:132-139, 187-242` — when the row is an AI issue, the status circle is replaced by an `AISuggestionsPin` composite:
  - Base: solid dark (Charcoal900) circle (same 48pt footprint as status circle)
  - Foreground: `.aiSparkleFilled` 4-point star glyph in white
  - Overlay (only when `status == .processing`): a ring drawn with `AngularGradient` (purple → blue → transparent) rotated via `.rotationEffect` on a `@State` angle incremented by `Timer` — the "spinning" ring
  - Overlay (only when `status == .error`): a small warning triangle anchored bottom-trailing

### Android (current)

- `IssuesListComposable.kt:74-161` — `LazyIssueList` renders a flat `LazyColumn` over `List<IssueLightListModel>` via `itemsIndexed`, inserting `AlloyHorizontalDivider` between rows (not after the last).
- `IssueItem.kt:51-149` — every row is a `Row` with a 48dp colored status circle (filled with the status color, containing the type pin code text) followed by a two-line Column (title + assignee).
- `IssueLightListModel` — no AI status field today. Data model lives in pgf/ and is currently a "dumb" list model.

### What this doc delivers

Three genuinely different ways to extend the Android list to match iOS, varying along two axes:

1. **List architecture**: how the "Generate with AI" section is expressed in LazyColumn.
2. **Row indicator**: how the dark circle + sparkle + spinning gradient ring is drawn for `processing` state.

Each approach ends with a decision aid (pros/cons, complexity, extensibility).

---

## Shared Elements (all three approaches use these)

Regardless of the chosen approach, these primitives are the same:

| Element | Spec |
|---------|------|
| AI status enum | `sealed class AiSuggestionStatus { Processing, Ready, Error }` (or `enum class`) |
| Section header string | `R.string.issues_list_ai_suggestions_header` → "Generate with AI" |
| Sparkle icon | `ic_star_4_point.xml` (already added in the in-progress AlloyInputField POC at `android/alloycompose/.../res/drawable/ic_star_4_point.xml`) |
| Dark circle color | `AlloyTheme.colors.charcoal_900` (matches iOS Charcoal900) |
| Ring gradient colors | `[AlloyColorPallete.Purple700, AlloyColorPallete.AutodeskBlue700, Color.Transparent]` (reuses the `quickCreateGradient` palette already in `IssuesListComposable.kt:290-292`) |
| Warning icon | `ic_warning_triangle.xml` (already exists for generic error surfaces) |
| Source of status | `aiSuggestionStatus: AiSuggestionStatus?` exposed from Mobius state, keyed by `GGIssueUid` |

**Data flow is the same everywhere**: `IssuesViewModel` exposes a `Map<GGIssueUid, AiSuggestionStatus>` that the list consumes. Whether that status is pushed into the `IssueLightListModel` or kept as a side map is the key differentiator between approaches.

---

## Approach 1 — Typed LazyColumn Items + Overlay Indicator

> **Tagline**: Minimal invasion. Existing `IssueItem` untouched. Indicator is layered on top.

### List architecture

Introduce a sealed class that represents each row in the list:

```kotlin
sealed interface IssueListItem {
    data class AiHeader(val label: String) : IssueListItem
    data class AiIssue(
        val model: IssueLightListModel,
        val status: AiSuggestionStatus,
    ) : IssueListItem
    object SectionDivider : IssueListItem
    data class RegularIssue(val model: IssueLightListModel) : IssueListItem
}
```

The ViewModel composes this list:

```kotlin
val rows: List<IssueListItem> = buildList {
    if (aiIssues.isNotEmpty()) {
        add(IssueListItem.AiHeader(context.getString(R.string.issues_list_ai_suggestions_header)))
        aiIssues.forEach { add(IssueListItem.AiIssue(it, aiStatus[it.uid]!!)) }
        add(IssueListItem.SectionDivider)
    }
    regularIssues.forEach { add(IssueListItem.RegularIssue(it)) }
}
```

`LazyColumn` dispatches on type:

```kotlin
LazyColumn {
    itemsIndexed(rows, key = { _, it -> it.stableKey() }) { index, item ->
        when (item) {
            is IssueListItem.AiHeader -> AiSectionHeader(item.label)
            is IssueListItem.AiIssue -> Box {
                IssueItem(issue = item.model, ...)                 // untouched
                AiStatusIndicator(                                  // overlaid on the circle
                    status = item.status,
                    modifier = Modifier.align(Alignment.CenterStart).padding(Dimens.SpacingM)
                )
            }
            IssueListItem.SectionDivider -> AlloyHorizontalDivider(thickness = 2.dp)
            is IssueListItem.RegularIssue -> IssueItem(issue = item.model, ...)
        }
    }
}
```

### Row indicator

`AiStatusIndicator` is a `Box` (48dp, same footprint as the status circle) containing:

- **Base layer**: `Canvas` drawing a filled Charcoal900 circle.
- **Glyph layer**: `Icon(painter = painterResource(R.drawable.ic_star_4_point), tint = Color.White)` centered.
- **Ring layer (processing only)**: `Canvas` drawn on top — uses `Brush.sweepGradient` + `rotate` applied via `graphicsLayer { rotationZ = angle }`, where `angle` comes from `rememberInfiniteTransition().animateFloat(0f to 360f, tween(1400, easing = LinearEasing))`.
- **Error layer (error only)**: `Icon(ic_warning_triangle)` positioned bottom-end with a small white ring for contrast.

The indicator **covers** the existing status circle because both live at the same `Modifier.padding(Dimens.SpacingM)` offset from the start of the row. No change to `IssueItem`.

### Data model changes

- `IssueLightListModel`: **unchanged**.
- New pgf or Android-local type: `AiSuggestionStatus` enum.
- View state: `IssuesViewModel.IssueViewState.Data` gains `aiStatus: Map<GGIssueUid, AiSuggestionStatus>` that the Composable converts to the sealed `IssueListItem` list.

### Files to change / create

| Action | File | Purpose |
|--------|------|---------|
| CREATE | `android/issues/.../log/compose/IssueListItem.kt` | Sealed hierarchy + stable keys |
| CREATE | `android/issues/.../log/compose/AiSectionHeader.kt` | "Generate with AI" header composable |
| CREATE | `android/issues/.../log/compose/AiStatusIndicator.kt` | Box with circle + sparkle + animated ring |
| MODIFY | `android/issues/.../log/compose/IssuesListComposable.kt` | `LazyIssueList` consumes `List<IssueListItem>` instead of `List<IssueLightListModel>` |
| MODIFY | `android/issues/.../IssuesViewModel.kt` | Build `rows` from `issuesLightList` + `aiStatus` map |

### Pros

- `IssueItem` stays 100% untouched — zero regression risk for the regular list.
- Indicator is a self-contained composable — easy to preview, easy to iterate on animation.
- Sealed class makes it trivial to add more row types (e.g., "pending upload", "draft") later.
- Indicator overlay means if AI state is ever added mid-render without a restart, we don't need to hunt for where to plug in — just inject an `AiIssue` wrapper.

### Cons

- **Overlay brittleness**: the indicator's position depends on matching `IssueItem`'s padding exactly. If `IssueItem` ever changes its start padding, the overlay desyncs silently. Needs a screenshot test to catch.
- Two composables drawing a 48dp circle in the same pixel footprint is wasteful (the underlying colored status circle is invisible but still allocates).
- Divider logic in `LazyIssueList` (`if (index != issues.size - 1) AlloyDivider`) breaks because the list is no longer homogeneous — dividers need different logic per item type.

### Complexity: **LOW-MEDIUM**
~2 new files, 1 meaningful modification, no data-model churn. Animation is the trickiest part but lives in one composable.

### Extensibility: **MEDIUM**
Easy to add new row types. Harder to add new AI states because the overlay pattern hides the real row — any AI state that changes title/subtitle requires abandoning the overlay approach.

---

## Approach 2 — stickyHeader LazyColumn + Extended IssueItem

> **Tagline**: Use Compose's native section primitive. Bake AI state directly into the row.

### List architecture

Use `LazyColumn.stickyHeader {}` for the "Generate with AI" title so it pins to the top while the AI group is on-screen (mirrors the iOS visual weight of a section header even without stickiness — but Android gets stickiness for free).

```kotlin
LazyColumn {
    if (aiIssues.isNotEmpty()) {
        stickyHeader(key = "ai_header") { AiSectionHeader(...) }
        items(aiIssues, key = { it.uid.uidString }) { issue ->
            IssueItem(
                issue = issue,
                aiSuggestionStatus = aiStatus[issue.uid],  // <-- new param
                callbacks = ...,
            )
        }
        item(key = "section_divider") { AlloyHorizontalDivider(thickness = 2.dp) }
    }
    itemsIndexed(regularIssues, key = { _, it -> it.uid.uidString }) { index, issue ->
        IssueItem(issue = issue, aiSuggestionStatus = null, callbacks = ...)
        if (index != regularIssues.lastIndex) AlloyHorizontalDivider(...)
    }
}
```

### Row indicator

`IssueItem` gains an `aiSuggestionStatus: AiSuggestionStatus? = null` parameter. When non-null, the status-color circle at lines 96-120 is **replaced** (not overlaid) by `AiStatusIndicator`:

```kotlin
if (aiSuggestionStatus != null) {
    AiStatusIndicator(
        status = aiSuggestionStatus,
        modifier = Modifier.padding(Dimens.SpacingM).size(STATUS_CIRCLE_SIZE.dp)
    )
} else {
    Box(...existing status circle...)
}
```

The indicator itself uses the same `Canvas` + `rememberInfiniteTransition` technique as Approach 1 — the change is **where it plugs in**, not how it's drawn.

### Data model changes

- `IssueLightListModel`: **unchanged** (AI status passed as separate param, not baked into the model).
- `IssueItem`: gains `aiSuggestionStatus` param with default `null` (backward compatible for every other call site).
- View state: `IssuesViewModel.IssueViewState.Data` adds `aiStatus: Map<GGIssueUid, AiSuggestionStatus>`; the Composable splits issues into AI/regular groups locally.

### Files to change / create

| Action | File | Purpose |
|--------|------|---------|
| CREATE | `android/issues/.../log/compose/AiSectionHeader.kt` | "Generate with AI" header |
| CREATE | `android/issues/.../log/compose/AiStatusIndicator.kt` | Circle + sparkle + animated ring (same as Approach 1) |
| MODIFY | `android/issues/.../log/compose/IssueItem.kt` | Add `aiSuggestionStatus` param; swap circle when non-null |
| MODIFY | `android/issues/.../log/compose/IssuesListComposable.kt` | `LazyIssueList` split into AI + regular groups, use `stickyHeader` |
| MODIFY | `android/issues/.../IssuesViewModel.kt` | Expose `aiStatus` map in `Data` state |

### Pros

- **Uses Compose idioms natively** — `stickyHeader` is the canonical way to do sectioned lists in LazyColumn. Less custom infrastructure.
- **Sticky title** is a real UX win on long AI lists (you always see what section you're in).
- `IssueItem` becomes a single source of truth for "what does a row look like" — easier to reason about, one place to change if AI state needs to affect title or subtitle later.
- Sibling-safe: the header and rows are separate lazy items, so scrolling performance is identical to the flat list.
- Row height stays consistent — no overlay painting wastage.

### Cons

- **`IssueItem` now knows about AI** — a leaky concern that previously lived purely in the list screen. Future AI features will keep accreting params on `IssueItem`.
- `stickyHeader` experimental API requires `@OptIn(ExperimentalFoundationApi::class)` — already used elsewhere in the file, so low cost, but still an opt-in.
- Splitting `aiIssues` / `regularIssues` lives in the Composable or ViewModel — duplicated logic unless a shared helper is introduced.
- Divider rules still need per-group handling (divider after last AI row uses `SectionDivider` width, dividers within regular list are slim horizontal). Minor, but a gotcha.

### Complexity: **MEDIUM**
2 new files, 3 meaningful modifications. `IssueItem` touching is the most invasive change — every existing call site continues to work via the default param, but it's still a behavior-critical file.

### Extensibility: **LOW-MEDIUM**
Adding a new AI state (say, "partial result") means editing `IssueItem` and `AiStatusIndicator`. Adding a non-AI row type (e.g., "pending upload" badge) means a second bolt-on param. Pressure accumulates on `IssueItem`.

---

## Approach 3 — Dedicated AiIssueItem + Section Container

> **Tagline**: Full separation. AI rows are a new composable. Regular list is completely untouched.

### List architecture

Introduce a `GenerateWithAiSection` composable that owns the entire AI block (header + rows + trailing divider), and keep the existing `LazyIssueList` logic for the regular section:

```kotlin
LazyColumn {
    if (aiIssues.isNotEmpty()) {
        generateWithAiSection(aiIssues, aiStatus, callbacks)  // extension on LazyListScope
    }
    regularIssuesList(regularIssues, callbacks)                // extension on LazyListScope
}
```

Where:

```kotlin
fun LazyListScope.generateWithAiSection(
    issues: List<IssueLightListModel>,
    status: Map<GGIssueUid, AiSuggestionStatus>,
    callbacks: IssueItemCallbacks,
) {
    item(key = "ai_header") { AiSectionHeader(...) }
    items(issues, key = { it.uid.uidString }) { issue ->
        AiIssueItem(
            issue = issue,
            status = status[issue.uid] ?: AiSuggestionStatus.Processing,
            callbacks = callbacks,
        )
    }
    item(key = "ai_section_divider") { ThickDivider() }
}
```

`AiIssueItem` is a **brand new** composable — it reimplements the row layout (Row + dark circle indicator + title column) in ~60 lines of code. It's not a fork of `IssueItem`; it's a purpose-built sibling.

### Row indicator

The indicator lives **inside** `AiIssueItem` as an inline `Box` — it's not a separate reusable composable, it's the row's natural state symbol. Same drawing technique as Approach 1/2 (`Canvas` + `rememberInfiniteTransition` for the ring) but co-located with the row it belongs to.

This matches the iOS pattern where `AISuggestionsPin` is a dedicated view that only `ListSingleIssueView` knows about.

### Data model changes

- `IssueLightListModel`: **unchanged**.
- `IssueItem`: **unchanged**.
- View state: same `aiStatus: Map<GGIssueUid, AiSuggestionStatus>` as other approaches.
- New composable `AiIssueItem` is structurally similar to `IssueItem` but swaps the status circle for the AI indicator and does not support multi-select (AI rows are never multi-selectable on iOS either).

### Files to change / create

| Action | File | Purpose |
|--------|------|---------|
| CREATE | `android/issues/.../log/compose/AiSectionHeader.kt` | "Generate with AI" header |
| CREATE | `android/issues/.../log/compose/AiIssueItem.kt` | Full row composable dedicated to AI issues (includes indicator inline) |
| CREATE | `android/issues/.../log/compose/IssuesListSections.kt` | `LazyListScope` extensions `generateWithAiSection` / `regularIssuesList` |
| MODIFY | `android/issues/.../log/compose/IssuesListComposable.kt` | `LazyIssueList` delegates to the two section extensions |
| MODIFY | `android/issues/.../IssuesViewModel.kt` | Expose `aiStatus` map in `Data` state |

### Pros

- **Zero regression risk for the regular list** — `IssueItem` and all its paths are not touched.
- AI concerns live entirely in AI-named files (`AiIssueItem`, `AiSectionHeader`, `GenerateWithAiSection`). A developer searching for "AI" finds them all in one place.
- **Mirrors the iOS code structure** (`ListSingleIssueView` has a dedicated AI branch; `AISuggestionsPin` is a separate view). Easier spec-to-code traceability during the drift checks.
- `LazyListScope` extension pattern is idiomatic Compose and composes trivially — easy to add a third section ("drafts", "recent") without touching anything.
- AI rows can safely diverge from regular rows over time (e.g., swipeable actions, different click behavior for `error` state) without affecting regular rows.

### Cons

- **Code duplication**: `AiIssueItem` and `IssueItem` share ~70% of their layout code (Row, title, assignee). Drift risk — changes to typography or spacing need to be made in both.
- Largest surface area — 3 new files vs 2 in other approaches.
- If multi-select is ever needed on AI rows (spec says no, but spec can change), a feature bolt-on requires a second implementation or a refactor.

### Complexity: **MEDIUM-HIGH**
3 new files, 2 modifications. The new code is straightforward but there's more of it. The duplication discipline (keep AiIssueItem and IssueItem visually identical except for the indicator) requires a preview-driven review.

### Extensibility: **HIGH**
New row types plug in as new `LazyListScope` extensions. New AI states plug into `AiIssueItem` alone. Zero pressure on existing `IssueItem`. Easiest approach to evolve.

---

## Decision Aid

| Dimension | Approach 1 (Typed + Overlay) | Approach 2 (stickyHeader + Extended) | Approach 3 (Dedicated + Section) |
|-----------|------------------------------|--------------------------------------|-----------------------------------|
| **Data model impact** | None | None | None |
| **`IssueItem` modified?** | No | **Yes** | No |
| **Regression risk for regular list** | Low | Medium | **None** |
| **Code duplication** | None | None | **~70% between `AiIssueItem` and `IssueItem`** |
| **Sticky header** | No | **Yes** (free from `stickyHeader`) | No (could be added trivially) |
| **Indicator reusability** | High (standalone composable) | High (standalone composable) | Low (inlined in row) |
| **iOS code structure match** | Low | Medium | **High** |
| **Lines of new code (estimate)** | ~180 | ~150 | ~240 |
| **Number of new files** | 3 | 2 | 3 |
| **Easiest to add 4th section type later** | Medium (add sealed case) | Hard (more params on rows) | **Easy (new LazyListScope extension)** |
| **Hardest failure mode** | Overlay desyncs if `IssueItem` padding changes | `IssueItem` grows AI-specific params over time | Visual drift between `IssueItem` and `AiIssueItem` |

### My recommendation for the spec-forge pipeline

**Approach 3 (Dedicated AiIssueItem + Section Container)** is the best match for this codebase because:

1. **Spec traceability**: when Rev runs drift checks, a 1:1 mapping between iOS `AISuggestionsPin`/`ListSingleIssueView` AI branch and Android `AiIssueItem`/`generateWithAiSection` is the clearest possible alignment. Spec reviewers don't have to trace through shared code to know which path the AI state follows.
2. **Zero risk to the regular list**: the current Issues List is mature, well-tested, and used by every user every day. Adding AI by not touching it is the safest move.
3. **Duplication is bounded and catchable**: the ~70% shared layout can be caught by a screenshot-based Paparazzi test that renders `IssueItem` and `AiIssueItem` side by side and asserts identical title/assignee positions. That test is cheap to write and catches drift early.
4. **Future-friendly**: quick-create AI will grow — error-retry affordances, drafts, "re-run with new photos". Each of those lands cleaner when AI code has its own home.

**Approach 2** is a reasonable second choice if the team strongly values `stickyHeader` stickiness and is willing to accept `IssueItem` growing an AI concern.

**Approach 1** is only right if the AI feature is expected to be temporary or experimental — the overlay trick is clever but fragile, and the data model fits sealed-class routing more naturally once you have more than one special row type.

---

## Next Steps (regardless of approach)

1. Add `AiSuggestionStatus` type to pgf/ shared (or keep Android-local if the iOS TCA reducer doesn't need symmetry).
2. Wire `IssuesViewModel.Data.aiStatus` from the existing `issueSuggestionRepository` observer (pgf already has the subscription, Area 3 of the research doc).
3. Write a Paparazzi preview for each of: AI section header, an AI row in each of the three states (processing / ready / error), a regular row immediately below.
4. Add screenshot tests against the preview.
5. Only then wire into the live `IssuesListComposable`.

## Usage in the Android ADR

When writing the quick-create Android ADR, map these spec requirements to the chosen approach:

- "List shows an AI-suggested issues section above the regular list" → section architecture choice
- "AI rows show a processing spinner while enrichment is in-flight" → indicator Canvas + `rememberInfiniteTransition` technique
- "AI rows show an error badge when enrichment fails" → error overlay
- EARS: WHEN the issue suggestions status map contains an entry for an issue UID, THE list SHALL render that issue under a "Generate with AI" header above the regular list with a dedicated AI status indicator in place of the status circle.
