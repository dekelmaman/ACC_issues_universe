# Research: quick-create-with-ai (Android)

> Phase 1 research. Platform: Android. Base Quick Create (without-AI) already shipped.
> Goal: map shipped base + WIP Issue-List AI UI vs the still-missing AI components.

---

## Spec Summary

Feature: Quick Create Issue — **AI delta** over shipped base. Spec at `pgf/feature/issues/specs/quick-create/quick-create-with-ai-spec.md` (Phase 2, v1.0). Decisions at `pgf/feature/issues/specs/quick-create/quick-create-decisions-spec.md` (C10–C19 close the Phase 2 scope).

Key requirements extracted from the spec:

- **Entry-point icon** swaps to an AI sparkle when AI is available (spec lines 51–52).
- **Composite "AI available"** = `issuesQuickCreateAiSuggestions ON  ||  (issuesQuickCreateAiEnabled ON  &&  server AI enablement true)` (spec 56–68, 275–282).
- **Effective "AI on"** = toggle ON AND available — drives AI send, counter, prompt copy (spec 64–68).
- **Toolbar subtitle** "Powered by Autodesk AI" only when available; Beta badge retained (spec 128–136).
- **Generate with AI row** — visible only when available; toggle default ON; sticky across sessions; state combines with availability (spec 156–170).
- **Image Carousel** — 5-image cap + "n/5" overlay when toggle ON; camera button disabled at cap; unlimited when toggle OFF (spec 139–152).
- **Input Field** — label "Prompt" when toggle ON, "Description" when OFF; shared 500-char cap; AI indicator star on floating label (spec 173–189 + KB doc).
- **Speech-to-Text** — optional, gated on `issuesQuickCreateSpeechToText` flag + supported locale (spec 192–210).
- **Issue-List enrichment states** — In progress / Ready to review / Failed, rendered on the row (spec 116–123).
- **Enrichment** — trigger on Next/Review when AI effectively on; first 5 images + prompt + type; server-scoped endpoint; on failure save prompt to description and show "failed" state without blocking toast (spec 213–254, C16–C17).
- **Feature flags**: `issuesQuickCreateAiSuggestions`, `issuesQuickCreateAiEnabled`, `issuesQuickCreateSpeechToText` (spec 269–283).
- **Analytics**: 8 Phase-2 events (toggle_changed, ai_suggestion_received, issue_details_opened, create_another_clicked, review_button_clicked, delete_confirmed, delete_cancelled, speech_to_text_clicked) with `ai_flag_enabled` / `ai_toggle_state` as common params (spec 288–306).

---

## KB Docs Summary

1. **`alloy-input-field-ai-indicator.md`** — mandates **Approach 3 (CompositionLocal)** for the purple 4-point star on `AlloyInputField`'s floating label. Creates `LocalShowAiIndicator`, a shared `AlloyFieldLabel`, and threads a new `showAiIndicator: Boolean` param through `AlloyInputField` + foundation. POC was at `alloycompose/.../field/ModifiedIndicatorApproach3.kt` but **that POC code is NOT in the current branch** (grep `LocalShowAiIndicator` returns 0 matches across the repo).
2. **`issue-list-ai-combined-plan.md`** — authoritative plan Trinity is following. Combines **Approach 2 (stickyHeader) + Approach 3 (dedicated `AiIssueItem` + `LazyListScope` section extensions)**. Reuses pgf `SuggestionStatus` / `IssueSuggestion` — no new Android-local enum. §10 resolves three open decisions: status type reuse, error-row tap routes to details (no retry), sticky header is text-only. §3 lists the F1–F9 file plan; §5 locks the view-side partition.
3. **`issue-list-ai-states-approaches.md`** — supersedes the raw approach comparison; ends with recommendation of Approach 3 (dedicated `AiIssueItem`). Also confirms `IssueLightListModel` stays **unchanged** — AI status is a side map keyed by `GGIssueUid`.
4. **`research-and-poc-suggestions.md`** — 10 risk-scored areas with iOS references. High-risk: Issue List section architecture, AI enrichment request pipeline (image→base64 JPEG, 5 images, `executeSync`, post-navigation failure write-back). Medium: composite availability, toggle persistence, failure handling. Low: sparkle icon, toolbar subtitle, counter overlay (already commented-out in code), STT plumbing, prompt/description label swap.

---

## Existing Code Analysis

### Already Shipped (base Quick Create without-AI)

| Area | File / Component | Relevance |
|------|------------------|-----------|
| Entry fragment | `android/issues/src/main/java/com/plangrid/android/issues/quick_create/QuickCreateFragment.kt` | Shipped; AI sparkle icon swap lands here + in `IssuesListComposable` FAB. |
| Mobius loop | `android/issues/.../quick_create/mobius/QuickCreateMobius.kt` | Full State/Effect/RenderData. `RenderData.usingAi: Boolean = false` already declared at `:384` — unused placeholder ready for wiring. `DESCRIPTION_MAX_CHARS = 500` at `:394` already matches spec C4. |
| Updater | `.../quick_create/mobius/QuickCreateUpdater.kt` | Will need new Actions for toggle-changed and AI-availability updates. |
| Processor | `.../quick_create/mobius/QuickCreateProcessor.kt` | `finalizeIssue` at `:265-320` saves description unconditionally — needs branching on "AI effectively on" per spec (description becomes the prompt only, not persisted to issue description field until failure). |
| State mapper | `.../quick_create/mobius/QuickCreateStateMapper.kt` | Maps State → RenderData; will produce `usingAi`, prompt labels, counter, etc. |
| Sticky defaults | `.../quick_create/QuickCreateStickyDefaults.kt` | Persists `lastUsedTypeUid` (global) + placement (per project). AI toggle persistence should join this class — add `aiToggleState: Boolean` property with a SharedPreferences-backed accessor (parallel to iOS `quickCreateIssueAIToggleState`). |
| Toolbar | `.../quick_create/composables/QuickCreateToolbar.kt` | Single-row `TopAppBar` with Beta badge + title + close. No subtitle slot today. Missing: "Powered by Autodesk AI" subtitle gated by availability; AI sparkle leading icon. |
| Action card | `.../quick_create/composables/QuickCreateActionCard.kt` | `AlloyExpandableInputField` + 500-char cap + `showSpeechToTextButton = true` already wired at `:200`. AI-specific assistive caption gated by `usingAi` already exists at `:211-221`. **Missing:** label swap Prompt↔Description, placeholder swap, AI indicator star on label, and the "Generate with AI" toggle row between type/placement and the description field. |
| Image area | `.../quick_create/composables/QuickCreateImageArea.kt` | Holds carousel + draw overlay. **Photo counter overlay is COMMENTED OUT at `:102-126`** — UI scaffold already written, just disabled. Needs: uncomment, reshape to "n/5" format, gate on `usingAi && photos.isNotEmpty()`. |
| Carousel | `.../quick_create/composables/QuickCreateCarousel.kt` | "Autodesk AI analyzes the first five photos" text already gated by `usingAi` (per research KB doc, Area 7). **Missing:** camera-button-disabled behavior when 5 photos hit AND toggle ON. |
| Analytics | `.../quick_create/analytics/QuickCreateAnalytics.kt` | Emits 11 events via `AnalyticsEvents.IssueQuickCreation*`. `aiFlagEnabled` and `aiToggleState` are **hardcoded to `false`** at `:23-24` with an explicit TODO comment "When AI is implemented, replace with injected values from IssuesFeatureFlag." No AI-specific events wired yet (`toggle_changed`, `ai_suggestion_received`, `issue_details_opened`). |
| Feature flag gate | `android/components/.../IssuesFeatureFlag.kt` | Consumes `ISSUES_QUICK_CREATE` and `ISSUES_QUICK_CREATE_SPEECH_TO_TEXT`. **Does NOT consume** `ISSUES_QUICK_CREATE_AI_SUGGESTIONS` or `ISSUES_QUICK_CREATE_AI_ENABLED` (enum entries exist in pgf at `PlanGridProjectFeatureFlag.kt:259-260` but are unused on Android). |
| Issues list screen | `android/issues/.../log/compose/IssuesListComposable.kt` | Flat `LazyColumn` over a single `List<IssueLightListModel>` — base shape is "one list, no sections". Modified on this branch (see WIP below). |
| Issues ViewModel | `android/issues/.../IssuesViewModel.kt` | `IssueViewState.Data(issuesLightList, isUnresolvedIssuesBannerVisible, dataVersion)` at `:160-167`. No AI awareness. Flow built from `IssuesRepository`. No injected `IssueSuggestionRepository`. |

### PGF already shipped (common KMP)

| Area | File | Relevance |
|------|------|-----------|
| Suggestions repo (status stream + executeSync) | `pgf/feature/issues/src/commonMain/.../repositories/IssueSuggestionRepository.kt` | `val statusMap: Flow<Map<GGIssueUid, IssueSuggestion>>` + `executeSync(issueUid, IssueSuggestionFields, ContextObject, SyncDoneListener)`. Already consumed by iOS. Android is a **zero-call-site** consumer today. |
| Impl | `pgf/feature/issues/issues-internal/src/commonMain/.../IssueSuggestionRepositoryImpl.kt` (510 lines) | Full impl shipped via KMP. |
| Models | `pgf/feature/issues/src/commonMain/.../models/IssuePendingSuggestions.kt` | `SuggestionStatus { ERROR, PROCESSING, PENDING_SUGGESTIONS }`, `IssueSuggestion(status, analytics)`, `ContextObject(generalDescription, images)`, `ImageWrapper(sourceType=BASE64, data)`, `IssueSuggestionRequest/Response/Fields`. |
| AI enablement repo | `pgf/feature/issues/src/commonMain/.../repositories/GGIssueAiEnablementRepository.kt` | `fun isAiEnabled(): OneQuery<Boolean>` — server-side composite input. |
| AI enablement sync | `pgf/legacy/src/commonMain/.../sync/types/issues/GGIssueAiEnablementSync.kt` | Periodic sync already shipped. |
| DI wiring | `pgf/legacy/src/commonMain/.../ProjectObjectRepository.kt:211, 2311-2318` | `ggIssueAiEnablementRepository` + suggestions repo exposed via the Project-level DI root. Android call site needs to inject this. |
| Flag enum entries | `pgf/domain/src/commonMain/.../PlanGridProjectFeatureFlag.kt:259-260` | `ISSUES_QUICK_CREATE_AI_SUGGESTIONS("issues_quick_create_ai_suggestions")`, `ISSUES_QUICK_CREATE_AI_ENABLED("issues_quick_create_ai_enabled")`. Entries exist; **Android consumer is missing** (see Platform Analysis). |

### WIP (local staged/modified changes — the delta that's already on-disk)

| File | Status | What it covers today | What's still missing / inconsistent |
|------|--------|----------------------|-------------------------------------|
| `android/issues/.../log/compose/ai/AiIssueItem.kt` | NEW (staged) | Full dedicated AI row per combined-plan §7: Charcoal/sparkle indicator (currently drawn with a **blue→purple linear gradient fill** at `:119-125`; the Charcoal900 version is commented out at `:136-149`), animated sweep-gradient ring on PROCESSING, error warning triangle trailing the row at `:91-100`. Previews for all three `SuggestionStatus` values. | (1) Indicator background diverges from plan §7.1 (`Charcoal900` specified; WIP uses AutodeskBlue500→Purple500 gradient). (2) Warning triangle is placed at `end` of the row (`:91-100`) — plan §7.4 puts it as an overlay at the bottom-end corner of the circle. Either plan or code needs to be reconciled. (3) Ring gradient uses `Purple500 → Purple300 → Transparent → Purple500`; plan §7.3 specified `Purple700 → AutodeskBlue700 → Transparent`. |
| `android/issues/.../log/compose/ai/AiRowPreview.kt` | NEW (staged) | Stacked paparazzi-style `@Preview` that renders AI section header + 3 AI rows (PENDING, PROCESSING, ERROR) + regular section header + one `IssueItem`. Matches plan §8 mitigation 3. | Good. No screenshot-test harness file yet (not in WIP); plan §4 CP3 assumes Paparazzi. |
| `android/issues/.../log/compose/ai/IssueRowBody.kt` | NEW (staged) | Extracts shared `Column { title + assignee }` with `AlloyText` + test tags `indexedItemTitle(itemIndex)` / `indexedItemAssignedTo(itemIndex)`. Called by both `IssueItem` (see below) and `AiIssueItem`. Matches plan §8 mitigation 1. | Good. |
| `android/issues/.../log/compose/ai/IssuesListSections.kt` | NEW (staged) | `LazyListScope.generateWithAiSection(...)` (stickyHeader + itemsIndexed + between-row divider) and `LazyListScope.regularIssuesList(...)` (preserves pre-existing `itemsIndexed` behavior; now **also** accepts `showStickyHeader: Boolean` to render an "Issues list" sticky header when the AI section is above it). | **Divergence from plan §3 F5 and §9:** plan specified a single `AiRegularSectionDivider()` between sections; WIP replaces that with a *second* sticky header ("Issues list"). This is a UX call that deviates from iOS (which has no "Issues list" title). Needs Neo's blessing or plan update. |
| `android/issues/.../log/compose/ai/IssuesSectionHeader.kt` | NEW (staged) | Generic sticky header over `AlloyColorPallete.Charcoal050` background, body2 Bold secondary-colored. Reused for both sections. | Good — supersedes the single-purpose `AiSectionHeader.kt`. |
| `android/issues/.../log/compose/ai/AiSectionHeader.kt` | **STAGED-DELETED (AD)** | n/a | **Deletion is intentional** — replaced by `IssuesSectionHeader.kt` above, which is generic enough to serve both AI and regular sections. Safe to drop from the final diff. |
| `android/issues/.../log/compose/IssueItem.kt` | MODIFIED | Now calls `IssueRowBody(title, assignee, itemIndex)` at `:118-122` instead of the inlined title/assignee `Column`. Still owns the colored status circle + multi-select checkbox + combinedClickable long-press. No visual change. | Good — matches plan F8. |
| `android/issues/.../log/compose/IssuesListComposable.kt` | MODIFIED | `LazyIssueList` now takes `aiIssues`, `regularIssues`, `aiStatus: Map<GGIssueUid, IssueSuggestion>` and delegates to the two `LazyListScope` extensions at `:135-157`. Partition happens view-side at `:226-228` using `issueState.aiStatus`. Preview at `:371-498` stubs 3 AI-status entries using `IssueSuggestion`/`IssueSuggestionAnalytics`/`SuggestionStatus`. | **Compile-blocking inconsistency**: line `:225` reads `issueState.aiStatus` but `IssueViewState.Data` in `IssuesViewModel.kt:162-167` has **no `aiStatus` field** yet. The VM side of plan F9 is **not implemented**. Running the preview is fine, but the live screen path will not compile. |
| `android/issues/src/main/res/drawable/ic_star_4_point.xml` | NEW (staged) | 4-point star vector, 24dp. White fill. | Located in the **issues** module (not `android/alloycompose` as the AlloyInputField KB doc suggested). Re-usable by `AlloyInputField` indicator if/when it lands, but cross-module access means the indicator POC will either duplicate the drawable in alloycompose or take a `painter: Painter` input. |
| `android/issues/src/main/res/values/strings.xml` | MODIFIED | Adds `issues_ai_suggestions_header = "Generate with AI"` and `issues_list_header = "Issues list"`. | Good. No new AI-specific Quick Create strings (prompt label, toolbar subtitle, toggle label) yet — still missing. |

### Not Yet Started (AI components)

| Spec Component | Current State on branch | Needed Work |
|----------------|-------------------------|-------------|
| Toolbar AI variant (subtitle + sparkle leading icon) | MISSING | Extend `QuickCreateToolbar.kt` to render a two-line title slot when `isAiAvailable == true` (title + "Powered by Autodesk AI"). Add an AI sparkle leading icon variant. Wire new `RenderData` fields: `isAiAvailable`, `isAiEffectivelyOn`. |
| FAB AI sparkle (issue list + project home) | MISSING | `IssuesListComposable.kt` FAB at `:296-317` currently paints `ic_camera` with a linear Blue→Purple gradient background. Add conditional sparkle icon when AI available. No `ic_sparkle` drawable exists yet — either reuse the new `ic_star_4_point.xml` or author a dedicated one. |
| Image Carousel AI variant (n/5 cap + overlay + disabled cam button) | PARTIAL | The counter overlay scaffold at `QuickCreateImageArea.kt:102-126` is commented out. Carousel has the "AI analyzes first 5" text gated by `usingAi`. Needed: uncomment overlay + format as "n/5" + wire disabled-camera-when-cap behaviour in `QuickCreateCarousel.kt`. |
| Generate with AI Row (toggle) | MISSING | No composable exists. Needed: new `QuickCreateAiToggleRow.kt` with `AlloyText` label + `AlloySwitch`. Renders between type/placement row and description field. Visibility = `isAiAvailable`. Wires `onToggleChanged(Boolean)` into new `Action.View.AiToggleChanged`. |
| Input Field AI variant (label swap + indicator star) | PARTIAL | Label / placeholder swap is a 2-string-resource flip — KB doc `research-and-poc-suggestions.md` Area 6 labels this LOW risk. AI indicator star requires AlloyCompose changes per `alloy-input-field-ai-indicator.md`: `LocalShowAiIndicator` + `AlloyFieldLabel`. The POC code is **not in this branch**. Need to author the 3 new alloycompose files + 4 modifications per the KB table. |
| Speech-to-Text | PARTIAL (already flag-gated base) | `showSpeechToTextButton = true` already ON in `QuickCreateActionCard.kt:200`. Locale gating not yet wired. Add `issuesQuickCreateSpeechToText` consumer to `IssuesFeatureFlag.kt` (currently only `ISSUES_QUICK_CREATE_SPEECH_TO_TEXT` is wired as `isSpeechToTextEnabled` — confirm the spec-level flag matches). Add supported-locale gate. |
| Composite AI availability provider | MISSING | No `AiEnablementProvider` equivalent. Needed: new class combining `IssuesFeatureFlag.isAiSuggestionsFlagOn || (IssuesFeatureFlag.isAiEnabledFlagOn && ggIssueAiEnablementRepository.isAiEnabled())`. Reactive: wrap `isAiEnabled().asFlow()` in a stream so availability drop mid-session flows back into Mobius. |
| AI toggle persistence | MISSING | Extend `QuickCreateStickyDefaults` with `aiToggleState: Boolean` property (default true). Wire into `QuickCreateMobius.State.aiToggleState` + `RenderData.usingAi` (today hardcoded false at `RenderData:384`). |
| Enrichment request pipeline | MISSING | No Android call site for `IssueSuggestionRepository.executeSync`. Needed: (a) image→base64 JPEG utility (5 photos, Dispatchers.IO); (b) a coroutine scope outside `QuickCreateMobius` lifetime (Application-scoped or WorkManager) so the enrichment survives screen close; (c) failure handler that writes prompt back to description via `IssuesRepository.updateDescriptionWithSideEffect` (already used at `QuickCreateProcessor.kt:282`). |
| Issue-List AI status wiring | MISSING (UI built, data absent) | See WIP table — `IssueViewState.Data.aiStatus` field + VM collection of `issueSuggestionRepository.statusMap` are not yet added. This is plan F9 work. |
| Analytics AI events | MISSING | `QuickCreateAnalytics` currently hardcodes `aiFlagEnabled=false, aiToggleState=false`. Needs: inject availability + toggle; add `trackToggleChanged`, `trackAiSuggestionReceived`, `trackIssueDetailsOpened` using the `AnalyticsEvents.IssueQuickCreation*` constructors. Confirm generator emits those three events (spec lists them in the Phase 2 table). |

---

## Reusable Components

| Purpose | Reusable thing | Path |
|---------|---------------|------|
| Sticky list header base | `IssuesSectionHeader` | `android/issues/.../log/compose/ai/IssuesSectionHeader.kt` (WIP) |
| Shared row body (title + assignee) | `IssueRowBody` | `android/issues/.../log/compose/ai/IssueRowBody.kt` (WIP) |
| AI 4-point star vector | `R.drawable.ic_star_4_point` (issues module) | `android/issues/src/main/res/drawable/ic_star_4_point.xml` |
| Sweep/linear gradient primitives | `AlloyColorPallete.AutodeskBlue*`, `Purple*`, `Charcoal*` | `android/alloycompose/.../ui/AlloyColorPallete.kt` |
| Material AlloyCompose widgets | `AlloyInputField`, `AlloyExpandableInputField`, `AlloyText`, `AlloyBadge`, `AlloyFab`, `AlloyCheckBox`, `AlloyDivider` | `android/alloycompose/...` |
| FAB switcher | `ExpandableFabSwitcher` + `FabItem` | `android/issues/.../log/compose/ExpandableFabSwitcher.kt` (via `IssuesListComposable` usage) |
| Speech-to-text surface | `:android:speechtotext` module (already a dep at `android/issues/build.gradle.kts:43`) | `android/speechtotext/...` |
| Photo storage/import | `QuickCreatePhotoImporter`, `QuickCreatePhotoStorage`, `QuickCreateGallerySaver` | `android/issues/.../quick_create/photos/` |
| pgf suggestions repo (Flow<Map>) | `IssueSuggestionRepository.statusMap` + `executeSync` | pgf path above |
| pgf AI enablement repo | `GGIssueAiEnablementRepository.isAiEnabled(): OneQuery<Boolean>` | pgf path above |
| AI toggle sticky pref pattern | `QuickCreateStickyDefaults` SharedPreferences shape | `android/issues/.../quick_create/QuickCreateStickyDefaults.kt` |
| Analytics emitter | `ProjectScopedAnalyticsDatasource` + `AnalyticsEvents.IssueQuickCreation*` | via `QuickCreateAnalytics` |
| Existing `usingAi` state pin | `RenderData.usingAi` + `QuickCreateCarousel` "analyzes first five" copy | Already wired; hardcoded source — needs toggle plumbing |

---

## Platform Analysis

### PGF changes needed: **NO — PGF layer is already shipped**.

Evidence:

- `IssueSuggestionRepository.kt:13` exposes `val statusMap: Flow<Map<GGIssueUid, IssueSuggestion>>`.
- `IssueSuggestionRepository.kt:15` exposes `fun executeSync(issueUid, IssueSuggestionFields, ContextObject, SyncDoneListener)`.
- `GGIssueAiEnablementRepository.kt:6` exposes `fun isAiEnabled(): OneQuery<Boolean>`.
- Both are DI-wired at the root in `ProjectObjectRepository.kt:211, 2311-2318`.
- `GGIssueAiEnablementSync.kt` exists at `pgf/legacy/src/commonMain/.../sync/types/issues/` — periodic sync already shipped.
- Flag enum entries `ISSUES_QUICK_CREATE_AI_SUGGESTIONS` and `ISSUES_QUICK_CREATE_AI_ENABLED` exist at `PlanGridProjectFeatureFlag.kt:259-260`.

**Conclusion**: Phase 5a (PGF layer) can be skipped. All missing work is Android-module.

NEO_QUESTION: Should we at least verify by running the common tests for `IssueSuggestionRepositoryImpl` before declaring PGF out of scope? Running `./gradlew :pgf:feature:issues:issues-internal:test` would give us a sanity check that the shipped PGF code actually works against the current API contract Android expects.

### Modules involved (Android-side)

- `:android:issues` — primary (WIP already here)
- `:android:components` — `IssuesFeatureFlag` (add two flag accessors + locale-aware STT gate)
- `:android:alloycompose` — `AlloyInputField` modifications per `alloy-input-field-ai-indicator.md`
- `:android:speechtotext` — already a dep; no changes expected
- `:pgf:feature:issues` — **read-only** consumption (repos + models + flag enum already imported via existing `implementation(project(":pgf:feature:issues"))` at `android/issues/build.gradle.kts:49`)
- `:pgf:legacy` — **read-only** consumption (sync + enablement already imported)

### Dependency chain

```
:android:issues
   ├── :android:alloycompose   ← needs AlloyFieldLabel + LocalShowAiIndicator
   ├── :android:components     ← needs AI flag accessors
   ├── :pgf:feature:issues     ← read-only (IssueSuggestionRepository, GGIssueAiEnablementRepository)
   └── :pgf:legacy             ← read-only (ProjectObjectRepository DI root, syncs)
```

---

## Build / Test / Lint Commands

| Module | Build | Unit test | Lint |
|--------|-------|-----------|------|
| `:android:issues` | `./gradlew :android:issues:assembleDebug` | `./gradlew :android:issues:testDebugUnitTest` | `./gradlew :android:issues:lintDebug` |
| `:android:alloycompose` | `./gradlew :android:alloycompose:assembleDebug` | `./gradlew :android:alloycompose:testDebugUnitTest` | `./gradlew :android:alloycompose:lintDebug` |
| `:android:components` | `./gradlew :android:components:assembleDebug` | `./gradlew :android:components:testDebugUnitTest` | `./gradlew :android:components:lintDebug` |
| Full alt-debug APK (per CLAUDE.md rule) | `./gradlew :android:app:assembleDebug` then `:android:app:installAltDebug` | n/a | n/a |
| `:pgf:feature:issues` common tests | n/a (library) | `./gradlew :pgf:feature:issues:issues-internal:test` | n/a |

Compose previews run in Android Studio's preview pane. No screenshot-test harness (Paparazzi) is wired in the `:android:issues` `build.gradle.kts` — the plan's §8 mitigation 3 + §10 CP3 reference a harness that doesn't exist here yet. **This is a risk**: the stacked `AiRowPreview.kt` can't regress-guard without a screenshot test runner.

NEO_QUESTION: Should adding a Paparazzi harness to `:android:issues` be a prerequisite of this feature, or accepted as tech debt? The plan relies on it as the primary drift-prevention mitigation.

---

## Risks & Constraints

- **GG Issues vs PG Tasks** — base Quick Create is GG Issues (Compose + Mobius). All AI work stays on this path. PG Tasks (legacy `IssueDetailFragment`) is **out of scope**. Memory-note confirmed.
- **Mobius loop lifetime vs enrichment**: enrichment outlives the screen (user navigates to next issue or list). `QuickCreateMobius` is ActivityScope; firing enrichment from its Processor risks cancellation when the activity is destroyed. Need Application-scope coroutine or WorkManager.
- **Kotlin Optional migration risk** — flagged in user memory; this feature does NOT touch `Optional` APIs in the Mobius/Compose layer I inspected. Low probability of hitting the same trap as `DatePickerView`.
- **IssueLightListModel immutability** — the combined plan keeps it untouched; partition uses a side map. Respects Approach 3's "regular list untouched" pledge.
- **Base64 image payload size** — 5 full-res camera JPEGs could be ~20 MB base64-encoded. Research KB flagged this. iOS uses JPEG 0.9 quality; Android needs equivalent resizing/compression policy decided in ADR.
- **sticky-header experimental API** — `LazyListScope.stickyHeader` needs `@OptIn(ExperimentalFoundationApi::class)`; already OK-ed in `IssuesListSections.kt` WIP.
- **`aiStatus` compile gap** — as noted, `IssuesListComposable.kt:225` reads a field that doesn't exist on `IssueViewState.Data` yet. The branch currently cannot compile the live screen path (preview path only).
- **Drawable module placement** — `ic_star_4_point.xml` is in the `:android:issues` res folder, not `:android:alloycompose`. The `AlloyInputField` AI indicator POC will need the asset in alloycompose or accept a `Painter` param; WIP chose issues-module scope which keeps the asset self-contained but duplicates if alloycompose adopts Approach 3.
- **AI indicator at two fidelity levels**: the 48dp circle star on Issue-List rows (implemented in WIP) uses a different color treatment than the spec/plan stated (Charcoal900 plan vs Blue→Purple WIP). UX decision pending.
- **Warning-triangle placement**: plan says overlay on circle; WIP uses trailing row icon. UX decision pending.
- **Feature flag wiring**: on Android, `IssuesFeatureFlag` is `@ProjectScope` — matches the PRD ("Project / local" scope column in spec §Feature Flags table). OK.

---

## Open Questions for Neo

- NEO_QUESTION: **Reconcile UI drift in AI row indicator.** WIP `AiIssueItem.kt:119-125` uses a Blue→Purple linear gradient filled circle; plan §7.1 says Charcoal900 filled circle (the Charcoal version is checked in commented-out at `:136-149`). Which is the intended final look? Same decision for the PROCESSING ring palette (WIP all-purple vs plan Purple700 → AutodeskBlue700 → Transparent).
- NEO_QUESTION: **Error state overlay vs trailing icon.** WIP renders the warning triangle as a trailing `Icon` at the end of the row (`AiIssueItem.kt:91-100`); plan §7.4 puts it as an overlay at the bottom-end corner of the status circle. Which is correct for parity with iOS `AISuggestionsPin`?
- NEO_QUESTION: **Second sticky header for the regular section.** WIP introduces an "Issues list" sticky header above the regular section when the AI section is present (`IssuesListSections.kt:80-84`). iOS only has the "Generate with AI" section header and a divider below — no regular-section title. Keep the Android-specific second header (user-facing UX), or drop it and add the `AiRegularSectionDivider` from plan §9?
- NEO_QUESTION: **`IssuesViewModel.Data.aiStatus` shape.** Plan F9 says add `aiStatus: Map<GGIssueUid, IssueSuggestion>` fed by `issueSuggestionRepository.statusMap`. Combine inside the existing `getIssuesList()` flow (collect `statusMap` in parallel and merge into emission), or expose as a second `StateFlow` and have the composable combine? The former is cleaner; the latter is easier to unit-test in isolation.
- NEO_QUESTION: **Enrichment-survives-navigation scope.** The spec requires enrichment to complete after the Quick Create screen is dismissed. The Mobius Processor is @ActivityScope. Should we (a) fire from a new Application-scoped service that wraps `IssueSuggestionRepository.executeSync`, (b) use WorkManager, or (c) move the suggestion trigger to the `IssuesViewModel` on the list screen? Option (a) parallels iOS's `SuggestionsRepositoryActor`.
- NEO_QUESTION: **AI toggle source of truth.** iOS uses *both* TCA State and UserDefaults. Android convention: Mobius State + `QuickCreateStickyDefaults`. Should we add the toggle to both, or treat `QuickCreateStickyDefaults.aiToggleState` as the single source of truth and project it into `State` at init?
- NEO_QUESTION: **Paparazzi prerequisite.** Plan §8 relies on a stacked-preview screenshot test as the primary drift guard between `IssueItem` and `AiIssueItem`. `:android:issues` does not have Paparazzi wired today. Add as prerequisite or accept as tech debt?
- NEO_QUESTION: **Reactivity of `isAiEnabled`.** `GGIssueAiEnablementRepository.isAiEnabled()` returns `OneQuery<Boolean>`. To react to mid-session changes, do we subscribe as a `Flow` and re-push into Mobius state, or accept init-time-only evaluation (matches iOS computed-var-every-read behavior)?

## Open Questions for User (non-architectural)

- The `quick-create-with-ai-spec.md` and `alloy-input-field-ai-indicator.md` both assume an AI-indicator asset lives in alloycompose; WIP placed the drawable in `:android:issues`. Is that deliberate (keeps it self-contained during prototyping) or should Trinity move it to `:android:alloycompose` during final implementation?
- The staged-deleted `AiSectionHeader.kt` is replaced by the generic `IssuesSectionHeader.kt`. Confirm the rename is intentional so we can drop the old file from the final diff without losing git blame intent.

---

## Confidence

- **Overall: HIGH** for the mapping (shipped / WIP / missing) — I read every WIP file, inspected base QC files, confirmed PGF contracts, and grep-verified the absence of Android consumers.
- **Medium** for UX-decision questions (indicator gradient, error placement, regular-section header) — these are legitimately ambiguous and need Neo/designer input.
- **High** that PGF is out of scope for this feature.
- **High** that the `IssueViewState.Data.aiStatus` compile gap is a real, blocking WIP inconsistency.
