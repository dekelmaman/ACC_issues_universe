# Implementation Plan — Quick Create AI → Issue Details Integration (Android)

**Author:** Neo (Software Architect)
**Revised:** 2026-04-20 — updated after Sherlock iOS research + user Q&A
**Branch context:** `shaggi/quick-create-with-ai-android`
**Scope:** Wire the AI suggestion pipeline (`IssueSuggestionRepository`) into the Issue Details screen so that AI-generated issues show:
1. A dedicated AI header at the top of the screen.
2. AI-styled editable fields (Title, Description, Root Cause, Custom Fields) with a 4-point star indicator + purple value text.
3. Loading shimmer variants for those fields while suggestion is still `PROCESSING`.
4. Automatic reversion to normal styling once the user edits an AI-suggested field (value-comparison based — same as iOS).
5. Suggestions deleted from DB on screen close (`rejectAll` on dismiss — same as iOS).

> **No code is produced here.** This is a task DAG with file-level acceptance criteria. Implementation is handed to Trinity task-by-task following the ordering below.

---

## Revision Notes (what changed from v1)

| Area | Old assumption | Corrected behaviour (iOS-verified) |
|------|---------------|-------------------------------------|
| Repo API | Expected combined `(status + fields)` Flow | Two-step: collect `statusMap` Flow, then on PENDING call `pendingSuggestionsByIssueUid.list()` |
| Field values | Use `IssueSuggestionFields` struct | Use `List<IssueFieldPendingSuggestion>` from `pendingSuggestionsByIssueUid` |
| `isSuggested` logic | Track `editedFieldIds: Set` in State | Value-comparison: `isSuggested = pendingSuggestions.any { it.field == X && it.suggestedValue != null }` |
| On user edit | Delete per-field suggestion row from DB | No DB write; StateMapper re-computes `isSuggested` → drops naturally when value diverges |
| On screen close | No cleanup | Emit `Effect.CleanupSuggestions` → call `rejectAllPendingSuggestionsForUid` (deletes all rows for this issue) |
| `approveAll` | Expected to be called somewhere | Dead code on iOS; never call it |
| `LocalShowAiIndicator` | Thought it needed manual `CompositionLocalProvider` in field composables | Already wired via `AlloyInputField(showAiIndicator: Boolean)` param — just pass `showAiIndicator = isAiSuggested` |
| Purple text scope | Unclear | Value text only — label gets the star icon, value text gets Purple500 |
| AI Header visibility | Only PENDING_SUGGESTIONS | Show for any non-null status (PROCESSING, PENDING, ERROR) |
| Per-session styling | Unclear | DB-driven: `rejectAll` on dismiss clears rows → re-open finds empty list → no styling |

---

## 1. Prerequisites (must hold before any task begins)

| # | Prerequisite | How to verify | Status |
|---|---|---|---|
| P1 | `IssueSuggestionRepository.statusMap: Flow<Map<GGIssueUid, IssueSuggestion>>` emits `PROCESSING → PENDING_SUGGESTIONS → ERROR`. `pendingSuggestionsByIssueUid(uid): ListQuery<IssueFieldPendingSuggestion>` holds per-field suggested values. These are the only two API surfaces needed. | Confirmed in `pgf/feature/issues/src/commonMain/.../IssueSuggestionRepository.kt` | ✅ Confirmed |
| P2 | Navigation from `AiIssueItem` passes the same `GGIssueUid` the repository is keyed on. | Inspect `IssuesListComposable` tap handler → details fragment args. | ⬜ Verify before T04 |
| P3 | `AlloyColorPallete.Purple500` is the approved AI accent color for value text. | Confirmed in alloy module. | ✅ Confirmed |
| P4 | `ic_star_4_point` drawable lives at `android/alloycompose/src/main/res/drawable/ic_star_4_point.xml`. | Confirmed per current branch `git status`. | ✅ Confirmed |
| P5 | `LocalShowAiIndicator` is already wired: `AlloyInputField(showAiIndicator: Boolean = false)` → `AlloyInputFieldFoundation` provides it → `AlloyFieldLabel` reads it and renders the star. T09 only needs to pass `showAiIndicator = isAiSuggested`. | Confirmed in `AlloyInputField.kt:88`, `AlloyInputFieldFoundation.kt:462`, `AlloyFieldLabel.kt:54`. | ✅ Confirmed |
| P6 | Feature flag `isAiSuggestionsFlagOn` gates the entire AI-details experience. | Read `IssuesFeatureFlag` and confirm semantics. | ⬜ Verify before T04 |
| P7 | `IssueFieldPendingSuggestion.field: IssueSuggestionFieldsEnum` values (`TITLE`, `DESCRIPTION`, `ROOT_CAUSE`, `CUSTOM_FIELD`) map to the correct Details fields. For `CUSTOM_FIELD`, `associatedUid` matches the `GGIssueCustomFieldDefinitionUid` used in `State.customFields`. | Read `IssuePendingSuggestions.kt` alongside `IssueDetailsMobius.State.CustomField`. | ⬜ Verify before T06 |

**If any unverified prerequisite fails, halt and raise before coding.**

---

## 2. Target Files

| File | Change kind |
|---|---|
| `android/issues/src/main/java/com/plangrid/android/issues/details/mobius/IssueDetailsMobius.kt` | MODIFY — extend `RenderDataField`, `State`, add `Event` and `Effect` |
| `android/issues/src/main/java/com/plangrid/android/issues/details/mobius/IssueDetailsUpdater.kt` | MODIFY — handle `SuggestionLoaded`, emit `CleanupSuggestions` |
| `android/issues/src/main/java/com/plangrid/android/issues/details/mobius/IssueDetailsProcessor.kt` | MODIFY — two-step repo subscription + cleanup handler |
| `android/issues/src/main/java/com/plangrid/android/issues/details/mobius/IssueDetailsStateMapper.kt` | MODIFY — value-comparison `isSuggested` projection |
| `android/issues/src/main/java/com/plangrid/android/issues/details/composables/DetailsScreen.kt` | MODIFY — render AI header, `LoadingAiField`, pass `isAiSuggested`, dispatch cleanup |
| `android/issues/src/main/java/com/plangrid/android/issues/details/composables/AiDetailsHeader.kt` | **NEW** — header composable |
| `android/issues/src/main/java/com/plangrid/android/issues/details/composables/fields/LoadingAiField.kt` | **NEW** — shimmer placeholder |
| `android/issues/src/main/java/com/plangrid/android/issues/details/composables/fields/TitleField.kt` | MODIFY — accept `isAiSuggested`, pass to `AlloyInputField(showAiIndicator)` |
| `android/issues/src/main/java/com/plangrid/android/issues/details/composables/fields/DescriptionField.kt` | MODIFY — same |
| `android/issues/src/main/java/com/plangrid/android/issues/details/composables/fields/RootCauseField.kt` | MODIFY — same (exposed/tap field) |
| `android/issues/src/main/java/com/plangrid/android/issues/details/composables/fields/CustomField.kt` | MODIFY — same |
| `android/app/src/main/java/com/plangrid/android/di/project/ProjectModule.kt` | MODIFY (if needed) — bind `IssueSuggestionRepository` |

---

## 3. Task DAG

```
Wave A — Foundation:
  T01 ── T02 ── T03
                 ├── T04 (Updater)
                 └── T05 (Processor)

Wave B — Mapping + UI leaves (all parallel):
  T02,T05 ── T06 (StateMapper)
  T07 (AiDetailsHeader)
  T08 (LoadingAiField)
  T01 ── T09 (Field composables)

Wave C — Integration:
  T06,T07,T08,T09 ── T10 (DetailsScreen wiring)
  T03,T05 ── T11 (Cleanup on dismiss)
  T04,T05 ── T13 (FF gating)

Wave D — Validation:
  T06 ── T14 (StateMapper tests)
  T04,T05 ── T15 (Processor/Updater tests)
  T07,T08,T09 ── T16 (Compose previews)
  T10,T11,T13 ── T17 (Manual QA)
  T12 (analytics — OPTIONAL)
```

---

## 4. Tasks

### T01 — Extend `RenderDataField` with `LoadingAiField` and `isAiSuggested` flag
**Depends on:** — (leaf)
**File:** `IssueDetailsMobius.kt` (~line 412)
**Scope:**
- Add `data class LoadingAiField(val fieldKind: FieldKind) : RenderDataField()` where `FieldKind` is a local enum `{ TITLE, DESCRIPTION, ROOT_CAUSE, CUSTOM_FIELD }`. Carrying `fieldKind` lets the shimmer render a meaningful label.
- Add `val isAiSuggested: Boolean = false` to: `Title`, `Description`, `RootCause`, `CustomField`.
**Acceptance criteria:**
- All existing `when(field)` expressions compile — add exhaustive `is RenderDataField.LoadingAiField -> { /* no-op */ }` for now.
- Default `isAiSuggested = false` — no runtime behavior change.

---

### T02 — Extend `IssueDetailsMobius.State` with AI slice
**Depends on:** T01
**File:** `IssueDetailsMobius.kt`
**Scope:** Add to `State`:
- `aiSuggestionStatus: SuggestionStatus? = null`
- `pendingSuggestions: List<IssueFieldPendingSuggestion> = emptyList()`

> **No `editedFieldIds` — dropped.** `isSuggested` is derived by value-comparison in StateMapper. A pending suggestion row staying in the list until `rejectAll` on dismiss is the source of truth (mirrors iOS exactly).

**Acceptance criteria:**
- Default constructor unchanged in behavior.
- Additive only — no reshuffling of existing fields.

---

### T03 — Add `Event.SuggestionLoaded` and `Effect.SubscribeToSuggestions` / `Effect.CleanupSuggestions`
**Depends on:** T02
**File:** `IssueDetailsMobius.kt`
**Scope:**
- `data class SuggestionLoaded(val suggestions: List<IssueFieldPendingSuggestion>, val status: SuggestionStatus?) : Event()`
  - `suggestions` is empty list when status is PROCESSING or ERROR (no fields yet).
- `data class SubscribeToSuggestions(val issueUid: GGIssueUid) : Effect()`
- `data class CleanupSuggestions(val issueUid: GGIssueUid) : Effect()` — triggers `rejectAllPendingSuggestionsForUid`; dispatched on screen close.

**Acceptance criteria:** Compiles; existing Events/Effects unchanged.

---

### T04 — Updater handles `SuggestionLoaded`; emits subscription on init
**Depends on:** T03
**File:** `IssueDetailsUpdater.kt`
**Scope:**
- On the existing "issue loaded / init" event, also dispatch `Effect.SubscribeToSuggestions(issueUid)` — **gated by `isAiSuggestionsFlagOn`**.
- On `Event.SuggestionLoaded`, update `state.aiSuggestionStatus` and `state.pendingSuggestions`. No `editedFieldIds` to touch.
**Acceptance criteria:**
- Unit test: `SuggestionLoaded(suggestions=[...], PROCESSING)` → `Next.next` with status and empty list set.
- Unit test: `SuggestionLoaded(suggestions=[title, desc], PENDING_SUGGESTIONS)` → `Next.next` with both fields in list.
- When flag off → no `SubscribeToSuggestions` effect emitted.

---

### T05 — Processor: two-step repo subscription + cleanup handler
**Depends on:** T03
**File:** `IssueDetailsProcessor.kt` (+ DI wiring if `IssueSuggestionRepository` not yet injected)
**Scope:**

**Handler for `Effect.SubscribeToSuggestions(uid)`:**
1. Collect `issueSuggestionRepository.statusMap` (a `Flow`) on the processor's coroutine scope, filtered for `uid`.
2. On each emission:
   - If status = `PROCESSING` or `ERROR`: emit `Event.SuggestionLoaded(emptyList(), status)`.
   - If status = `PENDING_SUGGESTIONS`: call `issueSuggestionRepository.pendingSuggestionsByIssueUid(uid).list()` (one-shot read), emit `Event.SuggestionLoaded(fields, PENDING_SUGGESTIONS)`.
3. If `uid` is absent from the map (no AI suggestion for this issue) → emit `SuggestionLoaded(emptyList(), null)`.
4. Subscription auto-cancels when processor coroutine scope is cancelled (Mobius loop disposal).

**Handler for `Effect.CleanupSuggestions(uid)`:**
- Call `issueSuggestionRepository.rejectAllPendingSuggestionsForUid(uid)`.
- This deletes all `IssueFieldPendingSuggestion` rows for this issue. On re-open, the subscription emits `[]` → no AI styling.

**DI:** Inject `IssueSuggestionRepository` via constructor. Check `ProjectModule.kt` — if not already bound, add binding there.

**Acceptance criteria:**
- Subscription started exactly once per loop lifetime.
- `rejectAll` called exactly once on `CleanupSuggestions`.
- No leak: tearing down the loop cancels the Flow collector.

---

### T06 — `IssueDetailsStateMapper`: value-comparison `isSuggested` projection
**Depends on:** T02
**File:** `IssueDetailsStateMapper.kt`
**Scope:**

For each AI-eligible field (Title, Description, RootCause, each CustomField):

```
pendingSuggestion = state.pendingSuggestions.firstOrNull { it.field == <fieldEnum> [&& it.associatedUid == fieldDefinitionUid for custom] }

isSuggested = pendingSuggestion != null && pendingSuggestion.suggestedValue != null

when (state.aiSuggestionStatus) {
    PROCESSING  → emit LoadingAiField(kind)        // regardless of current value
    PENDING     → emit <FieldType>(isAiSuggested = isSuggested)  // value already in issue state
    ERROR, null → emit <FieldType>(isAiSuggested = false)        // normal
}
```

Key rules:
- `isSuggested` is true as long as the DB row exists with a non-null `suggestedValue`. It drops automatically if the user edits (value comparison NOTE: Android does NOT need to compare values — the row stays until `rejectAll`; the star just shows until dismiss).
- When status is PROCESSING, the AI-eligible fields emit `LoadingAiField` *instead of* their normal variant and *instead of* the generic `LoadingField`.
- Non-AI fields (Location, Assignee, Date, Attachments, etc.) are never affected.

**Acceptance criteria:**
- Pure function; no side effects.
- Unit tests (T14) cover all branches.

---

### T07 — New `AiDetailsHeader` composable
**Depends on:** —
**File (new):** `android/issues/src/main/java/com/plangrid/android/issues/details/composables/AiDetailsHeader.kt`
**Scope:**
- `AiDetailsHeader(modifier: Modifier = Modifier)` — stateless, no ViewModel refs.
- Row layout: 4-point star icon (`com.plangrid.android.alloy.compose.R.drawable.ic_star_4_point`) + "Generated with AI" text + spacer + "AUTODESK AI" badge (filled Purple500) + "BETA" badge (outlined, orange/amber).
- Use existing Alloy badge component if one exists; otherwise a private `AiBadge(text, filled, color)` composable in this file.
- String keys: `issues_ai_details_header_title`, `issues_ai_details_badge_autodesk`, `issues_ai_details_badge_beta`.

**Acceptance criteria:**
- `@Preview` renders without crash.
- No raw color values — Alloy tokens only.

---

### T08 — New `LoadingAiField` composable
**Depends on:** —
**File (new):** `android/issues/src/main/java/com/plangrid/android/issues/details/composables/fields/LoadingAiField.kt`
**Scope:**
- `LoadingAiField(label: String, modifier: Modifier = Modifier)`.
- Renders an outlined field frame (matching `AlloyInputField` outline shape/border) with:
  - Label text + AI star indicator (wrap content in `CompositionLocalProvider(LocalShowAiIndicator provides true)`).
  - Body: `AlloySkeletonLine.AlloySkeletonSingleLine` shimmer (consistent with existing `LoadingField` skeleton in `DetailsScreen.kt:940`).
- Non-interactive.

**Acceptance criteria:**
- `@Preview` shows label + shimmer.
- Visual frame matches the outlined field style (same border radius, padding as a regular field).

---

### T09 — Field composables accept `isAiSuggested`
**Depends on:** T01
**Files:**
- `TitleField.kt`
- `DescriptionField.kt`
- `RootCauseField.kt`
- `CustomField.kt` (or wherever `RenderDataField.CustomField` is rendered)

**Scope:**
- Add `isAiSuggested: Boolean = false` parameter.
- When `true`:
  - Pass `showAiIndicator = true` to `AlloyInputField` (or `AlloyExpandableInputField`, `AlloyExposedInputField`) → this wires the star on the label automatically via `LocalShowAiIndicator`.
  - Override the field's value `textStyle` color to `AlloyColorPallete.Purple500`. **Label text stays default color** — only the value text turns purple.
- When `false`: no change to existing appearance.

**Acceptance criteria:**
- `isAiSuggested = false` (default) → existing previews unchanged.
- `isAiSuggested = true` `@Preview` per file shows star on label + purple value text.

---

### T10 — `DetailsScreen` wires everything together
**Depends on:** T06, T07, T08, T09
**File:** `DetailsScreen.kt`
**Scope:**
- Before the first field in `state.fields.forEachIndexed`: if `state.aiSuggestionStatus != null`, render `AiDetailsHeader()`.
- In the `when(field)` switch, add `is RenderDataField.LoadingAiField` branch → render `LoadingAiField(label = stringResource(labelResFor(field.fieldKind)))`.
- For `Title`, `Description`, `RootCause`, `CustomField` branches, pass `isAiSuggested = field.isAiSuggested`.

**Acceptance criteria:**
- Non-AI issues render identically to today.
- PROCESSING: header + shimmer fields for the 4 AI-eligible types.
- PENDING_SUGGESTIONS: header + populated fields with star + purple value text.
- ERROR: header shown, fields render normally.

---

### T11 — Cleanup suggestions on screen close
**Depends on:** T03, T05
**Files:** `DetailsScreen.kt`, `IssueDetailsUpdater.kt`
**Scope:**

**`DetailsScreen.kt`:** On `DisposableEffect` (screen leave/dispose) OR on the existing back-pressed / `closeDetailsScreen` callback, dispatch `mobius.action(Issue.CleanupSuggestions)` (a new Mobius action wrapping the effect).

**`IssueDetailsUpdater.kt`:** On `Issue.CleanupSuggestions` → emit `Effect.CleanupSuggestions(issueUid)`.

The Processor's handler (T05) calls `rejectAllPendingSuggestionsForUid`. On the next subscription emission, the Flow emits an empty map for this uid → `SuggestionLoaded([], null)` → `pendingSuggestions = []` → all `isSuggested = false`.

> This is the **sole mechanism** that makes AI styling per-session-only. It mirrors iOS `onDisappear → .cleanup → rejectAll`.

**Acceptance criteria:**
- After navigating away and back, `pendingSuggestions` is empty (verified by StateMapper emitting `isAiSuggested = false` for all fields).
- `rejectAll` is called exactly once per screen lifetime (not on every recompose).

---

### T12 — Analytics (OPTIONAL — confirm with PM)
**Depends on:** —
**File:** `IssueDetailsProcessor.kt` or `IssueDetailsAnalytics.kt`
**Scope:** Emit `issue_details_opened_from_ai_suggestion` (issueUid, suggestionStatus) on first `SuggestionLoaded` with non-null status.
**NOTE:** Skip if product has not requested analytics for this ticket.

---

### T13 — Feature flag gating
**Depends on:** T04, T05
**Files:** `IssueDetailsUpdater.kt`, `IssueDetailsProcessor.kt`, `DetailsScreen.kt`
**Scope:**
- Updater: gate `SubscribeToSuggestions` effect emission behind `isAiSuggestionsFlagOn`.
- Processor: if flag is off and effect somehow arrives, no-op.
- DetailsScreen: guard `AiDetailsHeader` render and `LoadingAiField` render behind flag as defense-in-depth.
**Acceptance criteria:** Flag OFF → zero AI UI, even if issue has suggestion rows in DB.

---

### T14 — Unit tests: StateMapper
**Depends on:** T06
**File:** `IssueDetailsStateMapperAiTest.kt` (new)
**Cases:**
1. `status == null`, `pendingSuggestions = []` → all fields normal.
2. `status == PROCESSING` → AI-eligible fields emit `LoadingAiField`; others normal.
3. `status == PENDING_SUGGESTIONS`, suggestions present for title+desc → those fields have `isAiSuggested = true`.
4. `status == PENDING_SUGGESTIONS`, `pendingSuggestions = []` (after cleanup) → all fields normal.
5. `status == ERROR` → header shown but all fields normal.

---

### T15 — Unit tests: Processor + Updater
**Depends on:** T04, T05
**Files:** `IssueDetailsUpdaterAiTest.kt`, `IssueDetailsProcessorAiTest.kt` (new)
**Cases:**
- Updater: flag ON + init → `SubscribeToSuggestions` emitted.
- Updater: flag OFF + init → no subscription effect.
- Updater: `SuggestionLoaded(suggestions, PENDING)` → State updated correctly.
- Updater: `CleanupSuggestions` action → `Effect.CleanupSuggestions` emitted.
- Processor: PROCESSING status emission → `SuggestionLoaded([], PROCESSING)` forwarded.
- Processor: PENDING status emission → `pendingSuggestionsByIssueUid.list()` called once, result forwarded.
- Processor: `CleanupSuggestions` effect → `rejectAll` called exactly once.
- Processor: loop dispose → Flow collector cancelled (fake repo tracks subscriber count → 0).

---

### T16 — Compose previews
**Depends on:** T07, T08, T09
**Scope:** Every new composable has ≥1 `@Preview`. Existing field composables gain a second preview with `isAiSuggested = true`.
**Acceptance criteria:** All render in Android Studio without crash; no missing tokens.

---

### T17 — Manual end-to-end QA
**Depends on:** T10, T11, T13
**Cases:**
1. Flag OFF → create AI issue, open details → no header, no purple, no star.
2. Flag ON + issue PROCESSING → open details → header + shimmer fields for title/desc/rootcause/custom.
3. Flag ON + issue PENDING → open details → header + purple value text + star on label.
4. Edit title field → title value diverges from suggestion → star + purple drops for title (StateMapper re-evaluates); other fields keep AI styling.
5. Navigate away, reopen → suggestions deleted by `rejectAll` → no header, no AI styling.
6. Non-AI issue → no header, no AI styling anywhere.
**Acceptance criteria:** All six pass; no regression on non-AI issues.

---

## 5. Risk Register

| # | Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|---|
| R1 | `SuggestionLoaded` fires after user edited a field → `isSuggested` re-evaluates to `true` unexpectedly. | Low | Medium | `pendingSuggestions` row still exists until `rejectAll`. The star will re-appear on re-evaluation. Acceptable — iOS has the same behaviour. If product disagrees, track edits in memory to suppress. |
| R2 | `IssueFieldPendingSuggestion.associatedUid` for `CUSTOM_FIELD` does not match `GGIssueCustomFieldDefinitionUid`. | Medium | High | Verify P7 before T06. Add `mapSuggestionCustomAttrs` helper in StateMapper if uid format differs. |
| R3 | Flow collector not cancelled on loop dispose → subscription leak. | Low | High | T05: run collector in Processor's own coroutine scope. T15 leak test. |
| R4 | ~~LocalShowAiIndicator not wired~~ | — | — | **Resolved** — already wired via `AlloyInputField(showAiIndicator)`. |
| R5 | Existing `LoadingField` fires in parallel with `LoadingAiField` during PROCESSING → duplicate shimmer. | Medium | Low | T06: StateMapper emits `LoadingAiField` *instead of* `LoadingField` for AI-eligible slots when PROCESSING. |
| R6 | Navigation passes wrong uid type (Android domain `IssueUid` vs pgf `GGIssueUid`) to repository. | Medium | High | Verify P2. Processor already uses `issueUid.fromDomain()` pattern for other repo calls — apply same conversion. |
| R7 | Feature flag leak: flag off but AI UI still renders. | Medium | Medium | Triple-gated: Updater (no subscription) + Processor (no-op) + DetailsScreen (no-render). T17 case 1 verifies. |
| R8 | `CleanupSuggestions` not fired on all exit paths (back button, system back gesture, process death). | Medium | High | Wire `DisposableEffect` in `DetailsScreen.kt` (covers all exit paths including system back). Test with back gesture in T17 case 5. |
| R9 | `rejectAll` called multiple times (e.g. both `DisposableEffect` and back-press handler). | Low | Low | Idempotent — deleting already-deleted rows is a no-op. |
| R10 | AlloyUI compliance drift in `AiDetailsHeader`/`LoadingAiField`. | Low | Medium | Run `alloy-auditor` agent on new files before merge. |
| R11 | `pendingSuggestionsByIssueUid.list()` is called on main thread → ANR risk on large datasets. | Low | Medium | Processor runs on its own coroutine scope (IO dispatcher assumed). Confirm dispatcher in T05. |

---

## 6. Wave Ordering

| Wave | Tasks | Gate |
|------|-------|------|
| A — Foundation | T01, T02, T03, T04, T05 | Build green + T04/T05 unit tests pass |
| B — Mapping + UI leaves (parallel) | T06, T07, T08, T09 | Build green + T14 pass + previews render |
| C — Integration | T10, T11, T13 | Build green + manual smoke test |
| D — Validation | T14, T15, T16, T17, (T12) | All tests pass; QA sign-off |

---

## 7. Open Questions (resolved)

| # | Question | Answer |
|---|---|---|
| Q1 | Combined observable or two-step? | Two-step: `statusMap` Flow + `pendingSuggestionsByIssueUid.list()`. Confirmed in iOS + repo interface. |
| Q2 | `approveAll` — when called? | Never. iOS only calls `rejectAll` on dismiss. `approveAll` is dead code. |
| Q3 | Per-session styling mechanism? | `rejectAll` on screen close deletes DB rows. Re-open finds empty list → no styling. No in-memory flag. |
| Q4 | Per-field suggestion deletion on edit? | No DB write on edit. Styling drops via value-comparison in StateMapper. |
| Q5 | `LocalShowAiIndicator` — manual or via param? | Via `AlloyInputField(showAiIndicator = isAiSuggested)` — already wired. |
| Q6 | Purple text scope? | Value text only. Label gets the star, not purple color. |
| Q7 | AI Header during ERROR? | Show the header for any non-null status. |
| Q8 | IssueType field AI styling? | Out of scope. |
| Q9 | Analytics? | Confirm with PM before T12. |

---

**End of plan.**
