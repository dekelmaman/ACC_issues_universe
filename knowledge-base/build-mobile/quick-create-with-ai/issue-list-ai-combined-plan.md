# Issue List AI — Combined Implementation Plan (Approach 2 + 3)

> **Status**: Plan only. No Kotlin implementation is written here — execution belongs to Trinity.
> **Scope**: Android Issues List UI for the "Generate with AI" section (AI-suggested issues grouped above the regular list).
> **Source of decision**: User chose to combine Approach 2 (stickyHeader) + Approach 3 (dedicated AiIssueItem + section extensions). See `issue-list-ai-states-approaches.md`.

---

## 1. Summary

This approach renders AI-suggested issues and regular issues inside **one `LazyColumn`** so scrolling is unified, but keeps the two code paths physically separate on disk. A `LazyColumn.stickyHeader { AiSectionHeader(...) }` pins the "Generate with AI" title at the top while any AI row is on-screen (Approach 2). Each AI row is drawn by a **dedicated `AiIssueItem` composable** living in AI-named files, with the status indicator (ready star / processing gradient ring / error triangle) rendered inline inside the row's leading circle — the existing `IssueItem` is **not modified** (Approach 3). `LazyListScope` extensions (`generateWithAiSection`, `regularIssuesList`) keep `LazyIssueList`'s body readable and make it obvious that the AI section is additive. A shared `IssueRowBody` private composable holds the title + assignee column so the regular and AI rows cannot visually drift.

---

## 2. LazyColumn structure (pseudocode)

Single `LazyColumn` body inside `LazyIssueList`:

```kotlin
LazyColumn(state = listState, modifier = Modifier.testTag(AutomationTestTags.IssuesList.CONTENT)) {

    // --- AI section (Approach 2 + 3) ---
    if (aiIssues.isNotEmpty()) {
        generateWithAiSection(
            aiIssues = aiIssues,
            aiStatus = aiStatus,
            onIssueClicked = onIssueClicked,
        )
        // Thicker visual separator between AI group and regular group
        item(key = "section_divider") { AiRegularSectionDivider() }
    }

    // --- Regular section (unchanged behavior) ---
    regularIssuesList(
        issues = regularIssues,
        selectedIssueUid = selectedIssueUid,
        selectedIssuesUids = selectedIssuesUids,
        isMultiSelectionModeOn = isMultiSelectionModeOn,
        issuesViewModel = issuesViewModel,
        masterDetailViewModel = masterDetailViewModel,
        onIssueClicked = onIssueClicked,
    )
}
```

Where `generateWithAiSection` (a `LazyListScope.() -> Unit` extension) expands to:

```kotlin
fun LazyListScope.generateWithAiSection(
    aiIssues: List<IssueLightListModel>,
    aiStatus: Map<GGIssueUid, AiSuggestionStatus>,
    onIssueClicked: (String) -> Unit,
) {
    stickyHeader(key = "ai_header") {
        AiSectionHeader(/* title, optional AI icon */)
    }
    itemsIndexed(aiIssues, key = { _, issue -> "ai_${issue.uid.uidString}" }) { index, issue ->
        AiIssueItem(
            issue = issue,
            status = aiStatus[issue.uid] ?: AiSuggestionStatus.Ready,
            itemIndex = index,
            onClick = onIssueClicked,
        )
        if (index != aiIssues.lastIndex) {
            AlloyDivider.AlloyHorizontalDivider(modifier = Modifier.padding(horizontal = 16.dp))
        }
    }
}
```

And `regularIssuesList` is the existing `itemsIndexed(issues) { ... IssueItem ... + divider between }` body extracted verbatim — no behavior change.

**Single LazyColumn guarantee**: there is exactly one `LazyColumn` in `LazyIssueList`. Both sections are contributed through `LazyListScope` extensions so scroll position, sticky-header mechanics, and `snapshotFlow` pagination continue to work as one list.

---

## 3. File plan (CREATE / MODIFY)

All Android paths are relative to `android/issues/src/main/java/com/plangrid/android/issues/` unless noted.

| # | Action | Path | Purpose |
|---|--------|------|---------|
| ~~F1~~ | ~~CREATE~~ | ~~`AiSuggestionStatus.kt`~~ | **DELETED** — see §10. Reuse `com.plangrid.pgfoundation.feature.issues.models.IssueSuggestion` + `SuggestionStatus` from pgf. |
| F2 | CREATE | `log/compose/ai/AiSectionHeader.kt` | Sticky header composable: "Generate with AI" text-only title over opaque Alloy surface background. No leading icon (match iOS). |
| F3 | CREATE | `log/compose/ai/AiIssueItem.kt` | Dedicated AI row. Leading `Box(STATUS_CIRCLE_SIZE.dp)` with Charcoal900 fill + white 4-point star icon; `Canvas` sweep-gradient ring when `PROCESSING`; small non-interactive warning triangle overlay when `ERROR`. Body delegates to `IssueRowBody`. Tap wires to the same `onIssueClicked(uid)` as `IssueItem` — no special-casing. |
| F4 | CREATE | `log/compose/ai/IssueRowBody.kt` | `internal` shared composable — the title + assignee column used by BOTH `IssueItem` and `AiIssueItem`. Single source of truth for row typography and padding. |
| F5 | CREATE | `log/compose/ai/IssuesListSections.kt` | `LazyListScope.generateWithAiSection(...)` and `LazyListScope.regularIssuesList(...)` extension functions. Also holds the private `AiRegularSectionDivider()` composable. |
| F6 | CREATE | `log/compose/ai/AiRowPreview.kt` | Paparazzi-style `@Preview` that stacks one `AiIssueItem` (PENDING_SUGGESTIONS), one `AiIssueItem` (PROCESSING), one `AiIssueItem` (ERROR) and one regular `IssueItem` in a column so typography/spacing drift is caught visually. |
| F7 | MODIFY | `log/compose/IssuesListComposable.kt` (lines 74–161 `LazyIssueList`) | Change signature: add `aiIssues: List<IssueLightListModel>`, `regularIssues: List<IssueLightListModel>`, `aiStatus: Map<GGIssueUid, IssueSuggestion>`. Replace the current `itemsIndexed(issues) { ... }` body with the two `LazyListScope` extension calls from §2. Keep `listState`, `LaunchedEffect(dataVersion)`, infinite-scroll `snapshotFlow` — those stay exactly as they are. |
| F8 | MODIFY | `log/compose/IssueItem.kt` | Replace the inlined title + assignee `Column(...)` block (currently lines 121–147) with a call to `IssueRowBody(title, assignee, itemIndex)`. No visual change. Keep `STATUS_CIRCLE_SIZE` constant and type-pin circle rendering here. |
| F9 | MODIFY | `IssuesViewModel.kt` (state at lines 160–166, emission at line 378) | Extend `IssueViewState.Data` with `aiStatus: Map<GGIssueUid, IssueSuggestion> = emptyMap()`. Populate by collecting `IssueSuggestionRepository.statusMap: Flow<Map<GGIssueUid, IssueSuggestion>>` and combining with the existing issues flow. No partition logic in the ViewModel — the composable partitions locally. |

**Nothing else moves.** `IssuesListComposable` at the `Data` branch (line 226) partitions `issueState.issuesLightList` into `(aiIssues, regularIssues)` right before calling `LazyIssueList` and passes all three lists down.

---

## 4. Data flow

The pgf AI suggestion repository emits a status-per-issue stream (keyed by `GGIssueUid`). `IssuesViewModel` combines that stream with the existing `issuesLightList` flow and emits `IssueViewState.Data(issuesLightList, aiStatus, isUnresolvedIssuesBannerVisible, dataVersion)`. `IssuesListComposable` reads `issueState.issuesLightList` and `issueState.aiStatus`, runs the local partition (§5) to produce `(aiIssues, regularIssues)`, and passes them plus `aiStatus` into `LazyIssueList`. The partition logic is intentionally **co-located with the list UI** — it is a view concern, not a ViewModel concern, and keeping it there means the ViewModel does not need to know the AI section exists; it just publishes "which issues are AI-flagged with what status".

---

## 5. Partition logic (code sketch)

Inside `IssuesListComposable`, just before calling `LazyIssueList`:

```kotlin
val (aiIssues, regularIssues) = remember(issueState.issuesLightList, issueState.aiStatus) {
    issueState.issuesLightList.partition { issueState.aiStatus.containsKey(it.uid) }
}
```

Ordering guarantee: `List.partition` preserves relative order for both output lists, so AI issues keep their original list order (first-seen AI issue stays first) and regular issues do too. No sort is introduced here — any AI-specific ordering (e.g., newest Processing first) is the repository's responsibility.

---

## 6. Sticky header visual contract

**Question**: Does `AiSectionHeader` stay pinned once the user has scrolled past all AI rows and only regular rows are visible?

**Answer**: **No.** Compose's `LazyListScope.stickyHeader` pins the header only while at least one item from the same "sticky group" is on screen. Because the AI items are the only items declared between the `stickyHeader("ai_header")` call and the next section (the `item("section_divider")` and then the `regularIssuesList` items), the header unsticks and scrolls away as soon as the last AI row leaves the viewport. This matches the iOS `List` section-header behavior exactly: the section header scrolls off with its section. The `AiRegularSectionDivider` item that immediately follows the AI group provides a clear visual boundary once the header is gone.

Implementation consequence: `AiSectionHeader` **must** render an opaque background (Alloy surface color) — otherwise rows scrolling underneath the pinned header bleed through.

---

## 7. AiIssueItem anatomy

Leading status indicator is a `Box(Modifier.size(STATUS_CIRCLE_SIZE.dp))` (same 48.dp as `IssueItem`), composed of these layers:

1. **Background**: `Modifier.clip(CircleShape).background(AlloyColorPallete.Charcoal900)` — filled dark circle.
2. **Center icon**: `Icon(painter = painterResource(R.drawable.ic_star_4_point), tint = Color.White)` centered via `contentAlignment = Alignment.Center`. The 4-point star asset already exists (`android/alloycompose/src/main/res/drawable/ic_star_4_point.xml`, untracked in current git status).
3. **Processing ring overlay** — rendered only when `status is Processing`:
   - `rememberInfiniteTransition()` → `animateFloat(initialValue = 0f, targetValue = 360f, animationSpec = infiniteRepeatable(animation = tween(durationMillis = 1400, easing = LinearEasing)))` → `angle`.
   - `Canvas(Modifier.matchParentSize().graphicsLayer { rotationZ = angle })`.
   - Brush: `Brush.sweepGradient(colors = listOf(AlloyColorPallete.Purple700, AlloyColorPallete.AutodeskBlue700, Color.Transparent))`.
   - `drawCircle(brush = brush, style = Stroke(width = 3.dp.toPx()))`.
4. **Error overlay** — rendered only when `status is Error`:
   - Small warning triangle icon (`ic_warning_triangle` or equivalent Alloy asset — confirm in §10) positioned bottom-end with `Modifier.align(Alignment.BottomEnd).size(16.dp).offset(x = 2.dp, y = 2.dp)`.

Row body: `Row { StatusIndicator(); IssueRowBody(title, assignee, itemIndex) }` — exactly mirrors the layout of `IssueItem` so multi-selection checkbox padding and click target stay consistent. Click handling for AI rows is a single `Modifier.clickable { onClick(issue.uid.uidString) }` on the outer `Row` — no long-press / multi-select semantics for AI rows (see §10 open question on Error clickability).

**Shared typography**: `IssueRowBody` (F4) is the single place the title uses `AlloyTheme.typography.body1.copy(fontWeight = FontWeight.SemiBold)` and the assignee uses `AlloyTextColor.Secondary + AlloyTextStyle.Body`. `IssueItem` and `AiIssueItem` both call it. Any designer-driven typography change is a one-file edit.

---

## 8. Drift-prevention strategy

**Risk**: duplicating the row into `IssueItem` and `AiIssueItem` means a padding / font-weight / ellipsis tweak made in one file silently diverges from the other. This is Approach 3's biggest known liability.

**Mitigations**:

1. **Extract `IssueRowBody(title: String, assignee: String?, itemIndex: Int)` (F4)** into `log/compose/ai/IssueRowBody.kt` as the single source of truth for the title + assignee column. Both `IssueItem` (F8 refactor) and `AiIssueItem` (F3) call it. The title composition (`"$displayId $title"`) stays in the caller because the AI row may render a different prefix (TBD in §10).
2. **Shared constants**: move `STATUS_CIRCLE_SIZE` to `IssueRowBody.kt` and reference it from both rows. Both rows' leading circles are 48.dp and can never drift.
3. **Paparazzi-style preview (F6)**: a `@Preview` composable that stacks `[AiIssueItem(Ready), AiIssueItem(Processing), AiIssueItem(Error), IssueItem(regular)]` in one column. When Paparazzi (or the equivalent screenshot harness used in this repo) runs in CI, any spacing/typography change in one row but not the other flips the golden image and the PR fails review.
4. **Test tag parity**: `AiIssueItem` uses `AutomationTestTags.IssuesList.indexedItemTitle(itemIndex)` / `indexedItemAssignedTo(itemIndex)` tags wired by `IssueRowBody`, so automation suites already asserting on those tags don't need a fork.

---

## 9. Scroll + divider rules

| Boundary | Divider | Rationale |
|----------|---------|-----------|
| Between two AI rows | `AlloyDivider.AlloyHorizontalDivider(modifier = Modifier.padding(horizontal = 16.dp))` | Matches existing regular-list slim inset divider (see `IssuesListComposable.kt:157`). |
| After last AI row (AI → regular boundary) | A dedicated `AiRegularSectionDivider()` composable rendered as its own `item("section_divider")`. Implementation: a `Spacer(Modifier.height(12.dp))` above a 1.dp full-bleed `AlloyHorizontalDivider` (no horizontal padding). | Full-bleed + extra vertical breathing room makes the group change obvious without inventing a custom 2dp thick rule. Matches iOS sectioned-list visual weight. |
| Between two regular rows | Unchanged — slim `AlloyHorizontalDivider` with 16.dp horizontal padding. | Zero regression requirement. |
| After last regular row | **No divider.** | Unchanged from today (`if (index != issues.size - 1)` guard stays). |
| Above sticky header (scrolled state) | None — the header's own opaque background is the visual separator from the search bar / banner above it. | Sticky headers look cluttered with a top rule. |

---

## 10. Resolved decisions (2026-04-19)

All three open questions were resolved before Trinity started. Sherlock investigated pgf + iOS; user confirmed header decision.

1. **Status type: reuse `com.plangrid.pgfoundation.feature.issues.models.IssueSuggestion` + `SuggestionStatus` from pgf.**
   - Evidence: `pgf/feature/issues/src/commonMain/kotlin/com/plangrid/pgfoundation/feature/issues/models/IssuePendingSuggestions.kt:11-20` defines `enum class SuggestionStatus { ERROR, PROCESSING, PENDING_SUGGESTIONS }` and `data class IssueSuggestion(status, analytics)`. Exposed via `IssueSuggestionRepository.statusMap: Flow<Map<GGIssueUid, IssueSuggestion>>`. iOS already consumes the same symbol at `IssuesListFeature.swift:125`. Zero Android imports today.
   - Consequence: **F1 is DELETED** — no new Android-local type. `AiSuggestionStatus` references in this doc are rewritten to `SuggestionStatus` / `IssueSuggestion`. ViewModel exposes `aiStatus: Map<GGIssueUid, IssueSuggestion>`.
2. **Error-row tap = navigate to detail (match iOS inert behavior).**
   - Evidence: `ListSingleIssueView.swift:289-293` tap handler unconditional; reducer at `IssuesListFeature.swift:614-627` only branches on editMode, never on status; grep for "retry" returned 0 matches. Warning icon at `:111-115` is non-interactive.
   - Consequence: `AiIssueItem` uses the same `Modifier.clickable { onIssueClicked(issue.uid.uidString) }` as `IssueItem`. No `onRetry` callback. No new `IssuesListCallbacks` entry. Warning triangle overlay is purely visual.
3. **Sticky header = text only, no icon (match iOS).**
   - Consequence: `AiSectionHeader` renders the localized "Generate with AI" string in Alloy typography over an opaque Alloy surface background. No leading drawable.

---

## 11. Spec-to-ADR traceability stubs (EARS)

Each requirement below should graduate into a formal acceptance criterion in the ticket's ADR.

- **R1 — Section presence**: WHEN the `aiStatus` map emitted by `IssuesViewModel` is non-empty, THE issues list SHALL render a "Generate with AI" section immediately above the regular issues list.
- **R2 — Sticky behavior**: WHILE at least one AI issue row is within the visible viewport of the issues list, THE "Generate with AI" section header SHALL remain pinned at the top of the list. WHEN the last AI row leaves the viewport, THE header SHALL scroll out of view with the section.
- **R3 — Processing indicator**: WHILE an AI issue's status is `Processing`, THE row's leading status symbol SHALL display a sweep-gradient stroke ring (Purple700 → AutodeskBlue700 → Transparent) rotating clockwise at a 1400 ms period with linear easing, overlaid on a Charcoal900 filled circle containing a white 4-point star.
- **R4 — Error indicator**: WHILE an AI issue's status is `Error`, THE row's leading status symbol SHALL display the Charcoal900 circle with white 4-point star AND a small warning-triangle glyph overlaid at the bottom-end corner of the circle. THE ring animation from R3 SHALL NOT be rendered.
- **R5 — Ready indicator**: WHILE an AI issue's status is `Ready`, THE row's leading status symbol SHALL display the Charcoal900 filled circle with a centered white 4-point star and NO animated ring and NO warning overlay.

---

## Verification checkpoints (for the human)

- [ ] **CP1 — after F1/F9**: confirm `AiSuggestionStatus` location and ViewModel state shape before any UI is touched.
- [ ] **CP2 — after F4/F8**: confirm the `IssueRowBody` extraction is a pixel-identical no-op for the regular row (existing screenshots unchanged).
- [ ] **CP3 — after F3 + F6**: review the Paparazzi preview of AI + regular rows stacked for drift and indicator correctness.
- [ ] **CP4 — after F5/F7**: verify one-LazyColumn scrolling, sticky-header unsticking behavior (R2), and infinite-scroll pagination still triggers when regular list grows.

---

## Code paths referenced (absolute)

- `/Users/sagihatzabi/Development/Android/build-mobile/android/issues/src/main/java/com/plangrid/android/issues/log/compose/IssuesListComposable.kt` (`LazyIssueList` at lines 74–161; `Data` branch at line 226)
- `/Users/sagihatzabi/Development/Android/build-mobile/android/issues/src/main/java/com/plangrid/android/issues/log/compose/IssueItem.kt` (row to be refactored to use `IssueRowBody`; `STATUS_CIRCLE_SIZE = 48` at line 43)
- `/Users/sagihatzabi/Development/Android/build-mobile/android/issues/src/main/java/com/plangrid/android/issues/IssuesViewModel.kt` (`IssueViewState.Data` at lines 160–166; emission at line 378)
- `/Users/sagihatzabi/Development/Android/build-mobile/android/alloycompose/src/main/res/drawable/ic_star_4_point.xml` (4-point star icon, present untracked in the current branch)
