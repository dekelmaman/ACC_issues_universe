# ADR: quick-create-with-ai — android

Status: proposed
Platform: android (iOS already shipped; PGF shared layer already shipped)
Date: 2026-04-19
Author: Neo (Architect)

---

## Context

Phase 2 Quick Create with AI is a **delta over the already-shipped base Quick Create**. The spec at `pgf/feature/issues/specs/quick-create/quick-create-with-ai-spec.md` (v1.0) defines ~28 behavioral requirements that only add to (never replace) the base flow. Scope for this ADR is strictly the AI layer on Android; the base camera + Mobius loop + sticky defaults are all shipped and out of scope.

Current state of the branch (research.md is the oracle):

- **Shipped (don't touch)**: `QuickCreateFragment`, `QuickCreateMobius` (with an unused `RenderData.usingAi: Boolean = false` placeholder at `:384`), `QuickCreateUpdater`, `QuickCreateProcessor.finalizeIssue`, `QuickCreateStateMapper`, `QuickCreateToolbar`, `QuickCreateActionCard` (500-char cap + STT button already wired), `QuickCreateImageArea` (photo counter UI scaffold already written but commented out at `:102–126`), `QuickCreateCarousel` ("Autodesk AI analyzes first 5" copy gated by `usingAi`), `QuickCreateAnalytics` (hardcoded `aiFlagEnabled=false, aiToggleState=false` at `:23–24` with a TODO), `QuickCreateStickyDefaults` (type global + placement per project; no AI toggle yet).
- **Shipped in PGF (read-only Android consumer)**: `IssueSuggestionRepository.statusMap: Flow<Map<GGIssueUid, IssueSuggestion>>` + `executeSync(...)`; `GGIssueAiEnablementRepository.isAiEnabled(): OneQuery<Boolean>` (read once at session start — see Decision 2); `GGIssueAiEnablementSync`; `SuggestionStatus { ERROR, PROCESSING, PENDING_SUGGESTIONS }`; flag enum entries `ISSUES_QUICK_CREATE_AI_SUGGESTIONS` / `ISSUES_QUICK_CREATE_AI_ENABLED` at `PlanGridProjectFeatureFlag.kt:259-260`.
- **WIP on this branch (covers ~80% of the Issue-List AI UI)**: five new files under `android/issues/src/main/java/com/plangrid/android/issues/log/compose/ai/` — `AiIssueItem.kt`, `AiRowPreview.kt`, `IssueRowBody.kt`, `IssuesListSections.kt`, `IssuesSectionHeader.kt` — plus modifications to `IssueItem.kt` and `IssuesListComposable.kt`, a new drawable `ic_star_4_point.xml` in the issues module, two new strings, and a staged deletion of the now-superseded `AiSectionHeader.kt` (status `AD`).
- **Known compile gap**: `IssuesListComposable.kt:225` reads `issueState.aiStatus` but `IssueViewState.Data` at `IssuesViewModel.kt:162-167` has no such field. The live screen path does not compile today. Preview paths do.
- **Not yet started**: Toolbar AI variant (subtitle + sparkle), FAB sparkle, image counter "n/5" uncomment + format, Generate-with-AI toggle row, Input-field label/placeholder swap + AI indicator star (AlloyCompose work), composite AI-availability provider, AI toggle sticky persistence, enrichment request pipeline (image→base64 + executeSync + Application-scoped), failure fallback write-back to description, issue-list AI status VM wiring (plan F9), AI analytics events + `aiFlagEnabled`/`aiToggleState` injection.

Target platform context for this ADR: **Android only**. iOS is shipped. PGF is shipped. No KMP work required.

---

## Decisions

### Decision 1: MVI shape — extend the existing `QuickCreateMobius` loop; do not create a new feature

**Choice**: Extend the existing `QuickCreateMobius` loop with new Actions, Effects, and State fields. Do not fork a new `QuickCreateAiMobius`.

**Rationale**: Research confirms the base loop (`QuickCreateMobius`, `QuickCreateUpdater`, `QuickCreateProcessor`, `QuickCreateStateMapper`) is already a complete feature with exactly the joinery needed — a placeholder `RenderData.usingAi` (currently hardcoded false) is already wired through the StateMapper to downstream composables (`QuickCreateCarousel` reads it). The spec describes Phase 2 as a **delta** — toggle, prompt-vs-description label, n/5 counter, toolbar subtitle, enrichment trigger — all of which are state mutations on the same screen. A fork would duplicate the entire photo pipeline, sticky defaults, and type/placement logic. Extension keeps the blast radius small, honors the mobius-mvi skill's "split large updaters into private extension functions" rule, and aligns with how the iOS `QuickCreateIssue` TCA reducer handles both modes.

**Concretely** (for Trinity):
- Add to `QuickCreateMobius.State`: `aiToggleState: Boolean = true`, `isAiAvailable: Boolean = false`, `photoCounterMax: Int = 5` (only meaningful when AI effectively on). **No `isSpeechToTextAvailable` field** (Decision 8 — STT stays as-is in base flow).
- Add to `QuickCreateMobius.Action.View`: `AiToggleChanged(newState: Boolean)`.
- Add to `QuickCreateMobius.Action.Processor`: `AiAvailabilityUpdated(available: Boolean)`, `StickyAiToggleLoaded(enabled: Boolean)`.
- Add to `QuickCreateMobius.Effect`: `LoadAiAvailabilitySnapshot` (one-shot — Decision 2), `PersistAiToggle(enabled: Boolean)`, `TrackAiToggleChanged(issueUid: String, newState: Boolean)`, `RequestAiEnrichment(issueUid: String, prompt: String, typeUid: String, photoUris: List<String>)`, `WriteBackDescriptionOnEnrichmentFailure(issueUid: String, prompt: String)`.
- Add to `QuickCreateMobius.RenderData`: keep `usingAi` (effective AI-on), add `isAiAvailable: Boolean`, `isAiEffectivelyOn: Boolean`, `promptLabel: PromptLabelMode` (enum with `Prompt`, `Description`), `placeholderResId: Int`, `isAiToggleVisible: Boolean`, `isStickyHeaderShown: Boolean`, `photoCounterText: String?` (null when toggle OFF), `toolbarSubtitleResId: Int?`, `leadingToolbarIconVariant: ToolbarIconVariant` (sealed — `Close`, `SparkleClose`). **No `isCameraButtonEnabled`** (removed per Decision 7 spec-deviation override).

### Decision 2: AI enablement reactivity — one-shot snapshot at session start (no reactive stream)

**Choice**: Call `ggIssueAiEnablementRepository.isAiEnabled()` **ONCE** when the Quick Create flow opens — specifically from a Mobius `First` / init effect (`Effect.LoadAiAvailabilitySnapshot`). The Processor handler executes the `OneQuery<Boolean>` a single time (no `.flow()` bridge, no subscription, no `handleCancelingPreviousStream`), reads the feature-flag composite once at the same moment, and pushes a single `Action.Processor.AiAvailabilityUpdated(available)` back into the loop. Thereafter the value is frozen for the entire session. No reactive observation. No Flow. No mid-session re-evaluation.

**Rationale**: The spec treats AI availability as session-scoped (the "server flag may change mid-session" case is not a requirement — the edge case at spec:245 describes behavior if availability changes, not a mandate to react reactively). Observing once at init is simpler: no leak risk, no `handleCancelingPreviousStream` plumbing, no `OneQuery<T>.flow()` coupling, no flag-read-per-emission churn. Feature flags on Android are `@ProjectScope` (`IssuesFeatureFlag`), so the composite read at init is:

```
available = issuesFeatureFlag.isAiSuggestionsFlagOn
         || (issuesFeatureFlag.isAiEnabledFlagOn && snapshotServerEnablement)
```

Where `snapshotServerEnablement` is the single `Boolean` returned by `isAiEnabled().execute()` (or however `OneQuery` is read synchronously/suspending — one call, one value). The composite is evaluated once and stored in `State.isAiAvailable` for the rest of the flow. Because SuggestionStatus sync already happens in background PGF sync, the value read at open-time is the freshest the Android layer has.

This **diverges from iOS** in surface behavior (iOS re-reads availability on every TCA state compute; Android reads once). That divergence is deliberate and acceptable: spec does not require mid-session reactivity.

**Resolves** research NEO_QUESTION #7 (Reactivity of `isAiEnabled`). Android adopts a **one-shot** pattern. Matches the simplicity principle.

### Decision 3: Enrichment status observation — `IssueSuggestionRepository.statusMap` consumed in `IssuesViewModel.kt`

**Choice**: Inject `IssueSuggestionRepository` into `IssuesViewModel`. Combine `statusMap: Flow<Map<GGIssueUid, IssueSuggestion>>` with the existing `issuesLightList` flow using `kotlinx.coroutines.flow.combine`. Emit a single `IssueViewState.Data` with both fields populated. Partition (AI vs regular) is **view-side** inside `IssuesListComposable`.

**Rationale**: Research.md + combined-plan §4 agree. The ViewModel's existing `getIssuesList()` flow is the correct composition point; `combine` (not `merge`) is the right operator because we need both streams joined before emission. Keeping partition at the view lets the ViewModel stay dumb about the AI section ("VM doesn't need to know the AI section exists; it just publishes `aiStatus`"). Sherlock already drafted this flow in the WIP Composable (`LazyIssueList(aiIssues, regularIssues, aiStatus, ...)`); only the VM side is missing.

**Resolves** research NEO_QUESTION #4. Single-flow `combine` (option A in the question), not a second `StateFlow`. The test story stays intact because `combine` is trivially testable with Turbine.

### Decision 4: `aiStatus` field placement — add to `IssueViewState.Data` (fixes the compile gap)

**Choice**: Add `aiStatus: Map<GGIssueUid, IssueSuggestion> = emptyMap()` to `IssueViewState.Data` in `IssuesViewModel.kt:162-167`. Default empty. Populated by the combine described in Decision 3. Shape uses pgf's `IssueSuggestion` model directly (no Android-local enum) per combined-plan §10 #1.

**Rationale**: The `IssuesListComposable.kt:225` WIP already reads `issueState.aiStatus` and the preview path stubs it using real `IssueSuggestion`/`IssueSuggestionAnalytics` constructors. Trinity just needs to add the field and wire it from the VM. This is a two-line data-class change plus VM wiring — lowest-friction fix for the known compile gap. Using the pgf type directly (rather than an Android adapter) keeps symbol identity with the emission source and matches how iOS consumes `IssueSuggestion` from the same KMP repo.

**Concretely**:
```kotlin
data class Data(
    val issuesLightList: List<IssueLightListModel>,
    val aiStatus: Map<GGIssueUid, IssueSuggestion> = emptyMap(),   // ← NEW
    val isUnresolvedIssuesBannerVisible: Boolean = false,
    val dataVersion: Long = 0
) : IssueViewState()
```

### Decision 5: Indicator gradient / Error icon / Sticky header (three UX divergences in WIP)

**5a. Indicator gradient: keep WIP Blue→Purple linear gradient. Update plan §7.1 to match.**

**Choice**: Retain the `Brush.horizontalGradient(colors = [AutodeskBlue500, Purple500])` at `AiIssueItem.kt:119-125`. Remove the commented-out `Charcoal900` block at `:136-149` once accepted. Update the processing ring to match plan §7.3 colors (`Purple700 → AutodeskBlue700 → Transparent`) — WIP currently uses all-Purple gradient which should change.

**Rationale**: The WIP indicator is the visual anchor users encounter first; it directly mirrors the `quickCreateGradient` already in use on the FAB (`IssuesListComposable.kt:293-294`). Using the same gradient palette on the FAB and the list-row indicator creates visual continuity — "the thing you tap to create becomes the thing that shows processing". Charcoal900 in the KB plan was conservative (iOS parity at all costs); the AutodeskBlue→Purple choice is a deliberate Android polish that actually reads **better** against Charcoal050 sticky headers. iOS parity on color is not a hard requirement — the spec describes behavior, not exact hex values, and iOS itself uses a purple-forward scheme. The WIP processing ring's all-Purple sweep however is a bug (insufficient contrast against the filled circle); switch it to the plan's `Purple700 → AutodeskBlue700 → Transparent` for visible rotation.

**5b. Error icon placement: keep WIP trailing-row icon. Update plan §7.4 to match.**

**Choice**: Retain the `AiIssueItem.kt:91-100` trailing warning icon (`ic_warning`, `Charcoal900` tint, `24.dp`, `padding(end = Dimens.SpacingM)`). Do not move it onto the status circle.

**Rationale**: The trailing-row icon is the existing Android idiom for row-level error states (look-and-feel parity with other lists in Build). An overlay on a 48dp circle at 16dp gets visually noisy against both the gradient fill and the 4-point star glyph; it reduces legibility. iOS's bottom-end circle overlay is a visual carryover from the photo-grid pin (`AISuggestionsPin`) where the circle is the dominant element — on a list row, the title is the dominant element and the error signal belongs on the row, not the avatar. Combined-plan §10 explicitly notes "warning icon is non-interactive" which is preserved either way; tap routes the whole row to details.

**5c. "Issues list" sticky header: REMOVE. Replace with the plan's `AiRegularSectionDivider()`.**

**Choice**: Drop the `showStickyHeader` branch in `IssuesListSections.kt:80-84`. Replace it with an explicit `item(key = "section_divider") { AiRegularSectionDivider() }` between the two sections (plan §9). The regular section retains its existing unadorned `itemsIndexed` body.

**Rationale**: iOS has no "Issues list" section title. Adding one on Android creates an unnecessary UX divergence and visual weight on a list users already know. The combined plan §9 specifies a specific divider treatment (`Spacer(height = 12.dp)` + full-bleed 1.dp `AlloyHorizontalDivider`) that's precisely calibrated to signal the section break without a title. Keeping the second sticky header also introduces a second sticky-layout cost — both headers competing for the top of the viewport as the user scrolls — which the plan §6 deliberately avoided.

**Resolves** research NEO_QUESTIONs #1, #2, #3.

### Decision 6: Sticky preference storage — extend `QuickCreateStickyDefaults` with a global boolean

**Choice**: Add `aiToggleState: Boolean` as a read/write property on `QuickCreateStickyDefaults`, keyed by a new constant `KEY_AI_TOGGLE_STATE = "quickCreateIssueAIToggleState"` (intentional parity with iOS key name from `quick-create-decisions-spec.md:70`). Default `true` when unset (matches spec: "Default: ON" + C13). Mobius `State.aiToggleState` is the single source of truth for the in-session value; `QuickCreateStickyDefaults` is the persisted backing store. Loop reads from defaults once on init (seeded into `initialState`), writes back via a Processor effect on every `Action.View.AiToggleChanged`.

**Rationale**: `QuickCreateStickyDefaults` is already the SharedPreferences wrapper for exactly this pattern (type global, placement per-project). Adding a third key is trivial and keeps all quick-create persistence in one file. **Global**, not per-project, because the spec (§Generate with AI Row: "persisted across sessions", no scope qualifier) and iOS (`quickCreateIssueAIToggleState` is a flat UserDefaults key) both treat it as user-wide. This also makes it simpler for a user who moves between projects to keep their toggle preference.

The "single source of truth" debate (research NEO_QUESTION #6): Mobius `State` wins during the activity lifecycle; `QuickCreateStickyDefaults` wins on re-entry. They are synchronized unidirectionally: user action → Mobius State → Processor effect → SharedPreferences. Reads are init-time only. This matches iOS's TCA State + UserDefaults pair without the double-write ambiguity.

### Decision 7: 5-image cap enforcement — payload-only cap (no camera button disable)

> **Note**: The product spec was updated to align with this decision. The previous camera-button-disable language at spec lines 145, 151, 251 has been removed. Android behavior: users may capture any number of photos while AI toggle is ON; only the first 5 are sent to the AI enrichment payload. The `"n/5"` counter stays pinned at `"5/5"` when `photos.size > 5` to communicate the AI-send count.

**Choice**: No camera-button disable. No Updater guard on `CameraButtonClicked`. The only image-cap behavior lives in the enrichment effect payload: when `RequestAiEnrichment` fires in `finalizeIssue`, its photo payload is `state.photos.take(5)`. Users may continue capturing photos beyond 5 with AI toggle ON; the extras exist in the carousel but are not sent to AI.

The counter overlay (SR-9) behaves as follows:
- When AI toggle OFF: overlay hidden.
- When AI toggle ON AND `photos.size <= 5`: overlay shows `"${photos.size}/5"`.
- When AI toggle ON AND `photos.size > 5`: overlay stays pinned at `"5/5"` to communicate the AI-send count. (The carousel still holds all photos; only the counter is clamped.)

StateMapper drops `isCameraButtonEnabled` from RenderData entirely (unless used elsewhere for non-AI reasons; a Sherlock-verified check confirms it is not). The Updater does not add a `CameraButtonClicked` guard for the cap.

**Rationale**: User override. Captured photos are cheap; restricting capture UX is worse than silently clamping the AI payload. Failure mode is benign: user takes 7 photos with AI ON, enrichment fires with first 5, remaining 2 stay attached to the issue as normal photos. The counter "5/5" overlay communicates to the user how many will be used by AI without hiding the capture affordance.

### Decision 8: STT — re-use existing base Quick Create mechanism; no new work for this phase

> ⚠️ **SPEC DEVIATION (partial)** — This decision deliberately overrides spec `pgf/feature/issues/specs/quick-create/quick-create-with-ai-spec.md` lines **192–196**, which require STT visibility to be gated by the `issuesQuickCreateSpeechToText` flag AND device-locale support. Per user instruction, Android does NOT add flag/locale gating in this phase. STT already works in the shipped base Quick Create and will continue to work unchanged in the AI path. The `issuesQuickCreateSpeechToText` feature-flag wiring is **PARKED** — if platform/PM later wants flag gating, it is a separate spec-deviation reconciliation effort.

**Choice**: Re-use the STT mechanism already present in the base Quick Create flow, unchanged. No new files, no new providers, no new flag accessor, no locale gating added for the AI path. `QuickCreateActionCard` continues to render the STT mic button exactly as it does in base Quick Create — whatever conditions make it visible in base also make it visible in the AI path. No `RenderData.showSpeechToTextButton` field is added; `QuickCreateActionCard` keeps its existing STT behavior (the hardcoded `showSpeechToTextButton = true` at `:200` is NOT touched in this phase).

**Rationale**: User override. STT already ships and works on the base flow. Adding a provider, flag accessor, and locale list introduces new code, tests, and DI wiring for no observable user-facing improvement in Phase 2. The spec's flag gating is deferrable — treating it as tech debt is acceptable because STT is not AI-feature-specific. The analytics event for STT (`speech_to_text_clicked`, spec:296) is still required and remains wired.

### Decision 9: Analytics wiring — inject availability + toggle, resolve the TODO, add 3 new events

**Choice**: Mutate `QuickCreateAnalytics.kt` to:
1. Inject `IssuesFeatureFlag` into the constructor.
2. Replace the hardcoded `aiFlagEnabled = false` at `:23` with a computed property reading the composite: `issuesFeatureFlag.isAiSuggestionsFlagOn || issuesFeatureFlag.isAiEnabledFlagOn` (the flag-side of composite availability only — server enablement is not a flag; spec §Common Parameters: "Whether AI suggestions flag is ON at event time"). **Not** the full composite availability.
3. Remove the hardcoded `aiToggleState = false` at `:24`. Add it as a **parameter** on every track method (pulled from Mobius `State.aiToggleState` at call time inside the Processor's track effects).
4. Add three new methods: `trackToggleChanged(issueUid, newToggleState)`, `trackAiSuggestionReceived(issueUid, imageCount, responseTimeMs, success)`, `trackIssueDetailsOpened(issueUid, durationToReviewMs, aiFlagEnabled)`. Each wraps the appropriate `AnalyticsEvents.IssueQuickCreation*` constructor.
5. Delete the TODO comment.

**Rationale**: The spec (§Analytics, lines 297–306) lists the 8 Phase-2 events and two common parameters. The spec also carves out specific exclusions: `toggle_changed` does **NOT** include `ai_flag_enabled`; `issue_details_opened` does **NOT** include `ai_toggle_state`. These are enforced by the event constructors in AnalyticsLib (the codegen omits those parameters on those events). Injecting `IssuesFeatureFlag` is straightforward — it's already `@ProjectScope` and used elsewhere in the same module. Pulling `aiToggleState` from Mobius state rather than caching in `QuickCreateAnalytics` avoids staleness bugs.

The call-site map (which effect fires which event) is in the Requirement Mapping table below under SR-23 through SR-30.

### Alternatives Considered

| Option | Pros | Cons | Decision |
|--------|------|------|----------|
| **New `QuickCreateAiMobius`** (fork the loop) | Zero risk to base loop; pure "AI mode" encapsulation | Duplicates entire photo + sticky + type/placement logic; two codepaths to maintain; doesn't match iOS single-reducer design | Rejected — picked Decision 1 (extend existing loop) |
| Reactive `OneQuery<Boolean>.flow()` bridge into a Processor stream | Mid-session availability changes flow through within one tick | Extra plumbing (`handleCancelingPreviousStream`, OneQuery→Flow bridge); spec does not require mid-session reactivity; more code to maintain | Rejected — picked Decision 2 (one-shot at init) |
| Expose `aiStatus` as a separate `StateFlow` on `IssuesViewModel` | Unit-testable in isolation; no combine operator | Composable now has two sources of truth; race between emissions; plan §4 explicitly recommends the single-flow approach | Rejected — picked Decision 3 (combine) |
| Add a new Android-local `AiSuggestionStatus` enum | Independent domain typing; Compose previews don't drag in pgf | Drift risk vs iOS; combined-plan §10 #1 explicitly rejected; status field already uses pgf `SuggestionStatus` in WIP | Rejected — use pgf `IssueSuggestion`/`SuggestionStatus` directly |
| `WorkManager` job for enrichment | Survives process death; standard Android pattern | Overkill for a "fire-and-forget with fallback" operation; complex payload (5 base64 images) crosses the Worker input size limit (10KB data bundle) | Rejected — use Application-scoped coroutine (see Dependencies) |
| Overlay the warning icon on the circle | iOS parity | Visual noise against gradient + star; row-level error is Android idiom | Rejected — picked 5b trailing icon |
| Charcoal900 circle fill | Plan §7.1; iOS parity | Loses the visual continuity with the FAB gradient; less distinctive | Rejected — picked 5a Blue→Purple |
| Keep "Issues list" sticky header | WIP currently has it | UX divergence vs iOS; double sticky cost; zero user value | Rejected — picked 5c remove |

---

## Requirement Mapping

| ID | Spec Requirement | Android Implementation | EARS Acceptance Criteria |
|----|------------------|------------------------|--------------------------|
| SR-1 | Entry-point icon swaps to AI sparkle when AI available (spec:51–52) | MODIFY `IssuesListComposable.kt` FAB at `:296-317` to conditionally render sparkle (reuse `R.drawable.ic_star_4_point`) when `isAiAvailable`; FAB read the flag via new collected State from `IssuesViewModel.isAiAvailable` | WHEN AI is available AND the Issue-List FAB is rendered, THE FAB leading icon SHALL be `ic_star_4_point`; OTHERWISE it SHALL be `ic_camera`. |
| SR-2 | Composite "AI available" = flagA OR (flagB AND server) (spec:56–68) | Compute inline in the Processor init handler for `Effect.LoadAiAvailabilitySnapshot`: read `IssuesFeatureFlag.isAiSuggestionsFlagOn \|\| (IssuesFeatureFlag.isAiEnabledFlagOn && snapshotServerEnablement)` where `snapshotServerEnablement` is the one-shot value of `ggIssueAiEnablementRepository.isAiEnabled()`. Result dispatched as `Action.Processor.AiAvailabilityUpdated` exactly once at session start | WHERE the flag `issuesQuickCreateAiSuggestions` is ON at session start, THE AI availability SHALL be true for the session. WHERE that flag is OFF at session start, THE AI availability SHALL equal `issuesQuickCreateAiEnabled && snapshotServerEnablement` frozen for the session. |
| SR-3 | Effective "AI on" = toggle ON AND available (spec:64–68) | StateMapper computes `RenderData.isAiEffectivelyOn = state.aiToggleState && state.isAiAvailable` | WHILE both the user's AI toggle is ON and AI availability is true, THE effective-AI flag SHALL be true; otherwise false. |
| SR-4 | Toolbar shows "Powered by Autodesk AI" subtitle when available (spec:15, 77–79, 128–136) | MODIFY `QuickCreateToolbar.kt` to accept `isAiAvailable: Boolean`; render a two-line title slot (`Column { AlloyText(title) + if(isAiAvailable) AlloyText(subtitle) }`). Add string `issues_quick_create_toolbar_ai_subtitle` | WHEN `isAiAvailable` is true, THE toolbar SHALL render the subtitle "Powered by Autodesk AI" under the title. WHEN false, THE subtitle SHALL NOT be rendered. |
| SR-5 | Beta badge retained (spec:134) | Unchanged — `QuickCreateToolbar` already renders the Beta badge | (No delta) |
| SR-6 | Generate-with-AI row visible only when AI available (spec:156–170) | CREATE `QuickCreateAiToggleRow.kt` (composable) with `AlloyText` label + `AlloySwitch`. Hosted inside `QuickCreateActionCard.kt` between type/placement and description. Visibility: `RenderData.isAiAvailable` | WHEN `isAiAvailable` is true, THE Generate-with-AI row SHALL be rendered above the description field. WHEN false, THE row SHALL NOT be rendered. |
| SR-7 | Toggle default ON, sticky across sessions (spec:161–164, C13, C14) | MODIFY `QuickCreateStickyDefaults` to add `aiToggleState: Boolean` with key `quickCreateIssueAIToggleState`, default `true`. Mobius seeds `State.aiToggleState` from it on init | WHEN the user has never set the AI toggle, THE persisted toggle value SHALL be true. WHEN the user changes the toggle, THE new value SHALL be persisted within the same Processor tick. |
| SR-8 | AI availability captured at session start (spec:168, :245) | Decision 2: one-shot read of `isAiEnabled()` + feature-flag composite dispatched as `AiAvailabilityUpdated` at session start; Updater sets `State.isAiAvailable` once; StateMapper derives `isAiEffectivelyOn` from the frozen value for the session | WHEN Quick Create opens, THE effective-AI flag SHALL be derived from the availability snapshot captured at session start AND SHALL remain constant for the duration of the session. |
| SR-9 | Image Carousel shows "n/5" when toggle ON (spec:76, :106, :141–144) | MODIFY `QuickCreateImageArea.kt` — uncomment the photo-counter overlay at `:102-126`; format text as `"${min(photos.size, 5)}/5"`; gate on `RenderData.photoCounterText != null` (StateMapper emits null when `!isAiEffectivelyOn`) | WHILE the effective-AI flag is true AND at least one photo has been captured, THE image carousel SHALL display the overlay "n/5" where n is `min(photos.size, 5)`. When `photos.size > 5`, THE overlay SHALL display "5/5" to communicate the AI-send count. |
| SR-10 | Max 5 images sent to AI (spec:141, :249) | Enrichment Effect payload is `state.photos.take(5)`. Carousel holds all captured photos regardless of count | WHEN the effective-AI flag is true, THE enrichment payload SHALL contain exactly the first 5 photos (or fewer if less than 5 were captured); additional photos remain attached to the issue but are NOT sent to AI. |
| SR-CAP | First-5-only AI payload when user captures more than 5 (spec-aligned; see Decision 7) | Decision 7 — no UI guard; Updater does not no-op `CameraButtonClicked`; enrichment payload uses `photos.take(5)` | WHEN the user captures more than 5 images WHILE AI toggle is ON, THE system SHALL send only the first 5 images to the AI enrichment payload AND SHALL NOT block further camera captures. |
| SR-12 | Cannot delete last photo (spec:146, C5) | Base shipped; unchanged | (No delta) |
| SR-13 | Input field label swap "Prompt" ↔ "Description" (spec:175, :179) | MODIFY `QuickCreateActionCard.kt` to read `RenderData.promptLabelMode` and pass the corresponding `@StringRes` to `AlloyExpandableInputField`. Add string resources: `issues_quick_create_label_prompt`, `issues_quick_create_label_description` | WHILE effective-AI is true, THE input field label SHALL be "Prompt". WHILE effective-AI is false, THE label SHALL be "Description". |
| SR-14 | Placeholder swap (spec:180) | Same mechanism as SR-13. StateMapper emits `RenderData.placeholderResId` | WHILE effective-AI is true, THE placeholder SHALL be the AI-specific copy. WHILE false, THE placeholder SHALL be the standard copy. |
| SR-15 | AI indicator star on floating label (KB: alloy-input-field-ai-indicator) | MODIFY `:android:alloycompose` per KB table: CREATE `LocalShowAiIndicator.kt`, `AlloyFieldLabel.kt`; MODIFY `AlloyInputField.kt` (add `showAiIndicator: Boolean = false` param), `AlloyInputFieldFoundation.kt` (wrap with `CompositionLocalProvider`), `AlloyEditTextFoundation.kt`, `AlloyInputFieldContent.kt`, `AlloyExposedField.kt` (all replace `AlloyTextView` label with `AlloyFieldLabel`). MOVE `ic_star_4_point.xml` from `:android:issues/res/drawable` to `:android:alloycompose/res/drawable` to resolve the cross-module reference | WHEN an `AlloyInputField` is configured with `showAiIndicator = true` AND the field has content (floating label state), THE floating label SHALL display a purple 4-point star at its end within the same animation frame as the label float. |
| SR-16 | 500-char cap shared with STT (spec:181, C4) | Base shipped (`QuickCreateActionCard.kt` already uses `DESCRIPTION_MAX_CHARS = 500`); unchanged | (No delta) |
| SR-17 | ~~STT available when flag ON AND supported locale (spec:194, :202)~~ **PARKED PER USER OVERRIDE** — see Decision 8 spec-deviation block | No change — STT behavior matches the shipped base Quick Create flow (text input always available as fallback). No new provider, no flag accessor, no locale list added this phase | WHERE STT is visible in the base Quick Create flow, THE STT mic button SHALL also be visible in the AI path (no additional gating); text input remains always available as fallback. |
| SR-18 | On save with AI on: send enrichment (spec:217, :220) | Decision — new `Effect.RequestAiEnrichment(issueUid, prompt, typeUid, photoUris)` dispatched from `finalizeIssue` when `state.isAiEffectivelyOn`. Processor handler converts photos to base64 JPEGs on `Dispatchers.IO`, builds `ContextObject`, calls `issueSuggestionRepository.executeSync(issueUid, IssueSuggestionFields.ALL, context, syncDoneListener)` | WHEN the user taps Next/Review AND effective-AI is true, THE enrichment request SHALL be dispatched with `issueUid`, prompt text, type UID, and the first 5 photos' base64-encoded JPEGs. |
| SR-19 | Enrichment survives screen dismissal (spec:217; iOS SuggestionsRepositoryActor) | Use an Application-scoped coroutine (injected via `@ApplicationScope @IoScheduler`) as the host for the `executeSync` call in the Processor handler. Activity-scope cancellation of the Mobius loop does **not** cancel the app-scoped job | WHEN the user dismisses the Quick Create screen after tapping Next/Review, THE enrichment request SHALL continue until completion or error. |
| SR-20 | On enrichment failure: save prompt to issue description (spec:26, :231, :248, C16) | Effect handler's `SyncDoneListener.onError` callback dispatches a no-argument write-back: call `issuesRepository.updateDescriptionWithSideEffect(issueUid, promptText)` (same method used by base `finalizeIssue` at `QuickCreateProcessor.kt:282`) | WHEN the enrichment request fails, THE issue's description field SHALL be updated with the original prompt text; NO blocking alert SHALL be shown. |
| SR-21 | No blocking toast on failure (spec:27, :232, :248, C17) | No `addEvent(ShowFailureDialog)` emitted from the failure handler. Failure is logged via existing `Logging.log(ERROR, ...)` only | WHEN enrichment fails, THE UI SHALL NOT render any blocking alert or toast. |
| SR-22 | Issues list shows enrichment states: In progress / Ready to review / Failed (spec:116–123) | Decisions 3 + 4 + 5 — `IssueViewState.Data.aiStatus` map populated from `statusMap`; `AiIssueItem` renders per-state indicator (PENDING_SUGGESTIONS → static star; PROCESSING → rotating ring; ERROR → trailing warning icon) | WHEN `aiStatus[issueUid]?.status == PROCESSING`, THE row SHALL show an animated sweep-gradient ring on the leading circle. WHEN `ERROR`, THE row SHALL show a trailing `ic_warning` icon. WHEN `PENDING_SUGGESTIONS`, THE row SHALL show the static Blue→Purple gradient circle with white 4-point star. |
| SR-23 | Analytics: `issue_details_opened` event (spec:299) | Add `trackIssueDetailsOpened(issueUid, durationToReviewMs, aiFlagEnabled)` method to `QuickCreateAnalytics`. Call site: Issue Details screen open-path (wired in `IssuesNavigation` or equivalent; not covered by WIP — Trinity identifies the call site). Note: spec explicitly excludes `ai_toggle_state` on this event | WHEN the user opens an issue's details after Quick Create creation, THE `issue_details_opened` event SHALL be logged with `duration_to_review_ms`, `issue_id`, and `ai_flag_enabled`. |
| SR-24 | Analytics: `toggle_changed` event (spec:297) | Add `trackToggleChanged(issueUid, newToggleState)`. Call site: Updater's `AiToggleChanged` case → Effect → Processor. Note: spec explicitly excludes `ai_flag_enabled` on this event | WHEN the user changes the AI toggle, THE `toggle_changed` event SHALL be logged with `new_toggle_state` and `issue_id`. |
| SR-25 | Analytics: `ai_suggestion_received` event (spec:298) | Add `trackAiSuggestionReceived(issueUid, imageCount, responseTimeMs, success)`. Call site: enrichment effect handler's `onComplete`/`onError` | WHEN the enrichment request completes (success or error), THE `ai_suggestion_received` event SHALL be logged with `image_count`, `response_time_ms`, and `success`. |
| SR-26 | Analytics: `create_another_clicked` event (spec:292) | Already wired in `QuickCreateAnalytics.trackContinueClicked`; only needs unhardcoded `aiFlagEnabled`/`aiToggleState` (Decision 9) | WHEN the user taps "Next issue" / "Continue to next issue", THE `create_another_clicked` event SHALL include the real `ai_flag_enabled` and `ai_toggle_state`. |
| SR-27 | Analytics: `review_button_clicked` event (spec:293) | Already wired in `QuickCreateAnalytics.trackReviewClicked`; Decision 9 applies | WHEN the user taps "Review", THE `review_button_clicked` event SHALL include the real `ai_flag_enabled` and `ai_toggle_state`. |
| SR-28 | Analytics: `delete_confirmed` / `delete_cancelled` events (spec:294–295) | Base-shipped methods on `QuickCreateAnalytics` get Decision 9 unhardcoding | WHEN the user confirms or cancels delete, THE respective event SHALL include the real `ai_flag_enabled` and `ai_toggle_state`. |
| SR-29 | Analytics: `speech_to_text_clicked` event (spec:296) | Already wired as `trackSpeechToTextClicked`; Decision 9 applies | WHEN the user taps the STT mic button, THE event SHALL include `is_recording`, `ai_flag_enabled`, and `ai_toggle_state`. |
| SR-30 | Common params `ai_flag_enabled` + `ai_toggle_state` on all events (spec:303–306) | Decision 9 — inject `IssuesFeatureFlag`, compute `aiFlagEnabled` as `isAiSuggestionsFlagOn \|\| isAiEnabledFlagOn`; pass `aiToggleState` from Mobius state on every track call | WHERE an analytics event is defined with both common parameters, THE event SHALL be emitted with the live feature-flag read and the live toggle value. |
| SR-31 | Feature flags: `issuesQuickCreateAiSuggestions`, `issuesQuickCreateAiEnabled` (spec:271–272) | MODIFY `IssuesFeatureFlag.kt` — add two `by lazy` accessors: `isAiSuggestionsFlagOn` reading `ISSUES_QUICK_CREATE_AI_SUGGESTIONS`; `isAiEnabledFlagOn` reading `ISSUES_QUICK_CREATE_AI_ENABLED`. Add imports from `PlanGridProjectFeatureFlag` | WHEN the project flags API is queried for `ISSUES_QUICK_CREATE_AI_SUGGESTIONS`, THE accessor `isAiSuggestionsFlagOn` SHALL return the live value; same for `ISSUES_QUICK_CREATE_AI_ENABLED`. |
| SR-32 | Feature flag: `issuesQuickCreateSpeechToText` (spec:273) | Already wired as `isSpeechToTextEnabled` at `IssuesFeatureFlag.kt:219-221`. No change | (No delta) |
| SR-33 | AI unavailable → behave as Phase 1 (spec:68–69, :245, :247) | All AI-gated UI gated on `RenderData.isAiAvailable`; StateMapper forces `isAiAvailable = false` triggers `photoCounterText = null`, `isAiToggleVisible = false`, `promptLabel = Description`, etc. | WHEN AI availability is false, THE toolbar subtitle, toggle row, n/5 overlay, Prompt label, and sparkle icon SHALL all be hidden; THE flow SHALL operate identically to Phase 1. |
| SR-34 | Returning user: sticky toggle restored (spec:40) | Decision 6 — Mobius seeds `State.aiToggleState` from `QuickCreateStickyDefaults.aiToggleState` at init | WHEN Quick Create opens, THE `State.aiToggleState` SHALL equal the value persisted in `QuickCreateStickyDefaults`. |
| SR-35 | Invalid image with AI ON → same error as non-AI (spec:252) | Base-shipped photo-validation path in `QuickCreateProcessor`; no AI branch | (No delta) |
| SR-36 | Offline → AI entry disabled where connectivity required (spec:253) | Server AI enablement endpoint is unreachable offline, so the one-shot read of `isAiEnabled()` at session start returns the cached value; `AiAvailabilityUpdated` reflects the cache. Enrichment request that fires offline will fail via the normal error path (SR-20). No new UI | (No delta; leverages existing sync-cache + SR-20) |
| SR-37 | Creator-device note: enrichment visibility on other devices (spec:236) | PGF responsibility; shipped | (No delta) |
| SR-38 | Issue-list section presence (combined-plan R1) | `IssuesListComposable.kt` renders AI section only when `aiIssues.isNotEmpty()` | WHEN `aiStatus` is empty, THE AI section SHALL NOT be rendered. |
| SR-39 | Sticky header behavior (combined-plan R2) | `IssuesListSections.generateWithAiSection` uses `stickyHeader(key = "ai_header")` | WHILE at least one AI row is in the viewport, THE "Generate with AI" header SHALL be pinned. WHEN the last AI row leaves the viewport, THE header SHALL scroll out of view with the section. |

---

## Component Plan

**Legend**: MODIFY = change existing WIP or shipped file; CREATE = new file; DELETE = remove existing file. Prefer MODIFY over CREATE where WIP covers the work.

### Issues module (`android/issues/`)

| Action | File | Description |
|--------|------|-------------|
| MODIFY | `android/issues/src/main/java/com/plangrid/android/issues/log/compose/IssuesListComposable.kt` | **WIP already partitions** `aiIssues`/`regularIssues` at `:226-228` and calls `LazyIssueList` with all three lists. This file already has the correct call shape — no changes needed once `IssueViewState.Data.aiStatus` exists (Decision 4). One remaining modification: FAB leading icon swap per SR-1 — add `isAiAvailable` to `IssuesListCallbacks` or to a new state collected from `IssuesViewModel`, render `painterResource(R.drawable.ic_star_4_point)` on the Quick Create `FabItem` when true. |
| MODIFY | `android/issues/src/main/java/com/plangrid/android/issues/log/compose/IssueItem.kt` | WIP already delegates title+assignee to `IssueRowBody(...)`. No further change. |
| MODIFY | `android/issues/src/main/java/com/plangrid/android/issues/log/compose/ai/AiIssueItem.kt` (WIP) | (a) Per Decision 5a: keep current Blue→Purple linear gradient at `:119-125`; delete the commented-out Charcoal900 block at `:136-149`; change ring gradient at `:162-169` from all-Purple to plan §7.3 palette: `listOf(Purple700, AutodeskBlue700, Color.Transparent, Purple700)`. (b) Per Decision 5b: keep the trailing `ic_warning` at `:91-100`. No other changes. |
| MODIFY | `android/issues/src/main/java/com/plangrid/android/issues/log/compose/ai/IssuesListSections.kt` (WIP) | Per Decision 5c: remove the `showStickyHeader: Boolean` param at `:78` and the `if (showStickyHeader)` block at `:80-84`. Replace with a new private `@Composable fun AiRegularSectionDivider()` rendered as `item(key = "section_divider") { ... }` **inside `generateWithAiSection`** after the last AI row (easier than cross-calling between extensions; aligns with plan §9). Signature for `regularIssuesList` becomes the pre-WIP shape (no `showStickyHeader`). |
| MODIFY | `android/issues/src/main/java/com/plangrid/android/issues/log/compose/ai/IssuesSectionHeader.kt` (WIP) | No change needed — still used for the "Generate with AI" sticky header only (after Decision 5c the regular section has no sticky header). Preview renaming from "Issues list" to just removing that preview is optional polish. |
| MODIFY | `android/issues/src/main/java/com/plangrid/android/issues/log/compose/ai/IssueRowBody.kt` (WIP) | No change needed. |
| MODIFY | `android/issues/src/main/java/com/plangrid/android/issues/log/compose/ai/AiRowPreview.kt` (WIP) | No change needed. Preview survives as drift guard. |
| DELETE | `android/issues/src/main/java/com/plangrid/android/issues/log/compose/ai/AiSectionHeader.kt` | Already staged-deleted (git status `AD`). Confirm the deletion stays in the final diff — superseded by `IssuesSectionHeader.kt`. |
| MODIFY | `android/issues/src/main/java/com/plangrid/android/issues/IssuesViewModel.kt` | (a) Inject `IssueSuggestionRepository` via constructor. (b) Add `aiStatus: Map<GGIssueUid, IssueSuggestion> = emptyMap()` to `IssueViewState.Data` at `:162-167` (Decision 4). (c) Modify the existing `issuesLightList` flow (around `:378`) to `combine` with `issueSuggestionRepository.statusMap` so each `IssueViewState.Data` emission includes both lists and the map. |
| MODIFY | `android/issues/src/main/java/com/plangrid/android/issues/quick_create/mobius/QuickCreateMobius.kt` | Add state fields per Decision 1: `aiToggleState`, `isAiAvailable`, `isSpeechToTextAvailable`. Add RenderData fields per Decision 1. Add `Action.View.AiToggleChanged`, `Action.Processor.AiAvailabilityUpdated`, `Action.Processor.StickyAiToggleLoaded`. Add Effects: `LoadAiAvailabilitySnapshot` (one-shot; Decision 2), `PersistAiToggle`, `RequestAiEnrichment`, `WriteBackDescriptionOnEnrichmentFailure`, `TrackAiToggleChanged`, `TrackAiSuggestionReceived`. Initial effects (Mobius `First`) include `LoadAiAvailabilitySnapshot` and a one-shot `LoadStickyAiToggle`. No STT-observation effect (see Decision 8). |
| MODIFY | `android/issues/src/main/java/com/plangrid/android/issues/quick_create/mobius/QuickCreateUpdater.kt` | Handle new View action: `AiToggleChanged` → `copy(aiToggleState = newState)` + `addEffect(PersistAiToggle(newState))` + `addEffect(TrackAiToggleChanged(state.currentIssueUid, newState))`. Handle new Processor actions: `AiAvailabilityUpdated` → `copy(isAiAvailable = it.available)`; `StickyAiToggleLoaded` → `copy(aiToggleState = it.enabled)`. **No** `CameraButtonClicked` cap guard (Decision 7 spec-deviation override — camera remains fully enabled). Split the expanded Updater into `handleViewAction` / `handleProcessorAction` / `handleAiAction` private extension functions if it grows past ~150 lines. |
| MODIFY | `android/issues/src/main/java/com/plangrid/android/issues/quick_create/mobius/QuickCreateProcessor.kt` | (a) Add `handleLoadAiAvailabilitySnapshot` (one-shot; Decision 2): inject `GGIssueAiEnablementRepository` + `IssuesFeatureFlag`; read `isAiEnabled()` exactly once, compute composite, emit `AiAvailabilityUpdated`. No Flow, no subscription, no `handleCancelingPreviousStream`. (b) Add `handlePersistAiToggle`: inject `QuickCreateStickyDefaults`, set `aiToggleState`. (c) Add `handleRequestAiEnrichment`: inject `IssueSuggestionRepository`, `ImageToBase64Converter`, `@ApplicationScope CoroutineScope`; on `Dispatchers.IO`, encode first 5 photo URIs to base64 JPEGs, build `ContextObject(generalDescription = prompt, images = wrappers)`, call `executeSync(...)` with a `SyncDoneListener` that on success fires `TrackAiSuggestionReceived(success = true)` and on error fires `WriteBackDescriptionOnEnrichmentFailure + TrackAiSuggestionReceived(success = false)`. Launch inside `@ApplicationScope` so it survives Activity death. (d) Add `handleWriteBackDescriptionOnEnrichmentFailure`: reuse `IssuesRepository.updateDescriptionWithSideEffect(issueUid, prompt)`. (e) Modify `finalizeIssue` at `:265-320`: when `state.isAiEffectivelyOn` is true, **do not** save description to the issue (description is the prompt; server will populate description via enrichment). When false, preserve shipped behavior. Dispatch `RequestAiEnrichment` on Next/Review when effective-AI is true. No STT-availability handler (see Decision 8). |
| MODIFY | `android/issues/src/main/java/com/plangrid/android/issues/quick_create/mobius/QuickCreateStateMapper.kt` | Compute RenderData new fields: `isAiAvailable = state.isAiAvailable`; `isAiEffectivelyOn = state.aiToggleState && state.isAiAvailable`; `photoCounterText = if (isAiEffectivelyOn && state.photos.isNotEmpty()) "${min(state.photos.size, 5)}/5" else null` (clamped — Decision 7); `promptLabelMode = if (isAiEffectivelyOn) PromptLabelMode.Prompt else PromptLabelMode.Description`; `placeholderResId = if (isAiEffectivelyOn) R.string.issues_quick_create_placeholder_ai else R.string.issues_quick_create_placeholder_description`; `isAiToggleVisible = isAiAvailable`; `toolbarSubtitleResId = if (isAiAvailable) R.string.issues_quick_create_toolbar_ai_subtitle else null`; `leadingToolbarIconVariant = if (isAiAvailable) SparkleClose else Close`. **No `isCameraButtonEnabled`** (Decision 7). |
| MODIFY | `android/issues/src/main/java/com/plangrid/android/issues/quick_create/QuickCreateStickyDefaults.kt` | Add `var aiToggleState: Boolean` property (getter returns `preferences.getBoolean(KEY_AI_TOGGLE_STATE, true)`; setter `preferences.edit().putBoolean(KEY_AI_TOGGLE_STATE, value).apply()`). Add `private const val KEY_AI_TOGGLE_STATE = "quickCreateIssueAIToggleState"` to the companion. |
| MODIFY | `android/issues/src/main/java/com/plangrid/android/issues/quick_create/composables/QuickCreateToolbar.kt` | Accept `isAiAvailable: Boolean` and `subtitleText: String?` params. Replace the current single-line title with `Column { AlloyText(title) + if (subtitleText != null) AlloyText(subtitleText, typography = caption) }`. Wire leading icon variant: if `isAiAvailable`, render sparkle + close; otherwise render only close (existing behavior). |
| MODIFY | `android/issues/src/main/java/com/plangrid/android/issues/quick_create/composables/QuickCreateActionCard.kt` | (a) Accept new params: `isAiToggleVisible: Boolean`, `aiToggleState: Boolean`, `onAiToggleChanged: (Boolean) -> Unit`, `promptLabelText: String`, `placeholderText: String`, `showAiIndicator: Boolean`. **Do NOT touch** the hardcoded `showSpeechToTextButton = true` at `:200` (Decision 8 — STT gating is parked). (b) Conditionally render `QuickCreateAiToggleRow` above the description field when `isAiToggleVisible`. (c) Pass `promptLabelText` / `placeholderText` into `AlloyExpandableInputField`. (d) Pass `showAiIndicator` to `AlloyInputField` (requires SR-15 AlloyCompose work). (e) Keep the AI-specific assistive caption at `:211-221` gated on `usingAi` (already shipped). |
| MODIFY | `android/issues/src/main/java/com/plangrid/android/issues/quick_create/composables/QuickCreateImageArea.kt` | Uncomment the photo-counter overlay at `:102-126`. Format the text as `photoCounterText` from RenderData (StateMapper already produces `"n/5"` or null). Wrap the overlay in `if (photoCounterText != null) { ... }`. |
| MODIFY | `android/issues/src/main/java/com/plangrid/android/issues/quick_create/composables/QuickCreateCarousel.kt` | **No change needed** (Decision 7 — camera button stays always-enabled). The "AI analyzes first 5" copy is already gated on `usingAi` and is preserved. |
| CREATE | `android/issues/src/main/java/com/plangrid/android/issues/quick_create/composables/QuickCreateAiToggleRow.kt` | New composable: `Row { AlloyText("Generate with AI") + Spacer + AlloySwitch(checked = aiToggleState, onCheckedChange = onAiToggleChanged) }`. Padding matches the type/placement row for visual consistency. Accessibility: toggle is described as "Generate with AI, enabled/disabled". |
| CREATE | `android/issues/src/main/java/com/plangrid/android/issues/quick_create/ai/ImageToBase64Converter.kt` | Injectable utility. Input: `List<Uri>` (up to 5). Output: `List<ImageWrapper>` (pgf type, `sourceType = BASE64`, data = base64 string of JPEG 0.9 compressed bitmap). Runs on `Dispatchers.IO`. Target 1600px max edge to keep payload under ~5MB/image. |
| MODIFY | `android/issues/src/main/java/com/plangrid/android/issues/quick_create/analytics/QuickCreateAnalytics.kt` | Per Decision 9: (a) Inject `IssuesFeatureFlag`. (b) Replace `private val aiFlagEnabled = false` at `:23` with `private val aiFlagEnabled get() = issuesFeatureFlag.isAiSuggestionsFlagOn \|\| issuesFeatureFlag.isAiEnabledFlagOn`. (c) Remove `private val aiToggleState = false` at `:24`; add `aiToggleState: Boolean` as a parameter on every track method (callers pass from Mobius state). (d) Add `trackToggleChanged(issueUid, newToggleState)`, `trackAiSuggestionReceived(issueUid, imageCount, responseTimeMs, success)`, `trackIssueDetailsOpened(issueUid, durationToReviewMs)` — note exclusions per spec. (e) Delete the TODO comment. |
| MODIFY | `android/issues/src/main/res/values/strings.xml` | Keep WIP additions (`issues_ai_suggestions_header`). **Remove** `issues_list_header` (Decision 5c removes the regular-section sticky header). Add new strings: `issues_quick_create_toolbar_ai_subtitle = "Powered by Autodesk AI"`, `issues_quick_create_label_prompt = "Prompt"`, `issues_quick_create_label_description = "Description"`, `issues_quick_create_placeholder_ai = "Describe the issue for AI to enrich"`, `issues_quick_create_placeholder_description = "Describe the issue"`, `issues_quick_create_ai_toggle_label = "Generate with AI"`. |

### AlloyCompose module (`android/alloycompose/`) — SR-15 only

| Action | File | Description |
|--------|------|-------------|
| CREATE | `android/alloycompose/src/main/java/com/plangrid/alloy/compose/widget/field/LocalShowAiIndicator.kt` | `val LocalShowAiIndicator = compositionLocalOf { false }`. |
| CREATE | `android/alloycompose/src/main/java/com/plangrid/alloy/compose/widget/field/AlloyFieldLabel.kt` | `@Composable fun AlloyFieldLabel(text: String, ...) { Row { Text(text); if (LocalShowAiIndicator.current) Icon(painterResource(R.drawable.ic_star_4_point), tint = AlloyTheme.colors.tertiary_03_500) } }`. |
| MODIFY | `android/alloycompose/src/main/java/com/plangrid/alloy/compose/widget/field/AlloyInputField.kt` | Add `showAiIndicator: Boolean = false` param. Wrap foundation with `CompositionLocalProvider(LocalShowAiIndicator provides showAiIndicator)`. |
| MODIFY | `android/alloycompose/src/main/java/com/plangrid/alloy/compose/widget/field/AlloyInputFieldFoundation.kt` | Wrap output with `CompositionLocalProvider`. |
| MODIFY | `android/alloycompose/src/main/java/com/plangrid/alloy/compose/widget/field/AlloyEditTextFoundation.kt` | Replace label `AlloyTextView` call with `AlloyFieldLabel`. |
| MODIFY | `android/alloycompose/src/main/java/com/plangrid/alloy/compose/widget/field/AlloyInputFieldContent.kt` | Same replacement. |
| MODIFY | `android/alloycompose/src/main/java/com/plangrid/alloy/compose/widget/field/AlloyExposedField.kt` | Same replacement. |
| MOVE | `ic_star_4_point.xml` | Move `android/issues/src/main/res/drawable/ic_star_4_point.xml` to `android/alloycompose/src/main/res/drawable/ic_star_4_point.xml`. Update all `R.drawable.ic_star_4_point` references in the issues module to `com.plangrid.alloy.compose.widget.R.drawable.ic_star_4_point` (or similar cross-module import). This resolves the cross-module asset reference cited in research risks. |

Note: alloycompose path above is best-guess based on project conventions; Trinity should confirm exact `com.plangrid.alloy.compose.widget.field` package vs whatever the current file structure shows. Delegate to Sherlock via `Glob "android/alloycompose/src/**/AlloyInputField.kt"` before starting if uncertain.

### Components module (`android/components/`)

| Action | File | Description |
|--------|------|-------------|
| MODIFY | `android/components/src/main/java/com/plangrid/android/components/issues/IssuesFeatureFlag.kt` | Add two `import com.plangrid.pgfoundation.domain.featureflags.PlanGridProjectFeatureFlag.ISSUES_QUICK_CREATE_AI_SUGGESTIONS` and `.ISSUES_QUICK_CREATE_AI_ENABLED` imports. Add two accessors: `val isAiSuggestionsFlagOn: Boolean by lazy { projectFlags.isEnabled(ISSUES_QUICK_CREATE_AI_SUGGESTIONS) }` and `val isAiEnabledFlagOn: Boolean by lazy { projectFlags.isEnabled(ISSUES_QUICK_CREATE_AI_ENABLED) }`. |

---

## Dependencies

### Already shipped (consume read-only)

- **PGF contracts**:
  - `com.plangrid.pgfoundation.feature.issues.repositories.IssueSuggestionRepository` — `statusMap: Flow<Map<GGIssueUid, IssueSuggestion>>` and `executeSync(issueUid, IssueSuggestionFields, ContextObject, SyncDoneListener)`.
  - `com.plangrid.pgfoundation.feature.issues.repositories.GGIssueAiEnablementRepository.isAiEnabled(): OneQuery<Boolean>`.
  - `com.plangrid.pgfoundation.feature.issues.models.{IssueSuggestion, SuggestionStatus, ContextObject, ImageWrapper, IssueSuggestionFields, IssueSuggestionAnalytics}`.
  - `com.plangrid.pgfoundation.domain.featureflags.PlanGridProjectFeatureFlag.{ISSUES_QUICK_CREATE_AI_SUGGESTIONS, ISSUES_QUICK_CREATE_AI_ENABLED, ISSUES_QUICK_CREATE_SPEECH_TO_TEXT}`.
- **Shipped base Quick Create** (extend, do not replace):
  - `QuickCreateMobius`, `QuickCreateUpdater`, `QuickCreateProcessor`, `QuickCreateStateMapper`, `QuickCreateFragment`.
  - `QuickCreateToolbar`, `QuickCreateActionCard`, `QuickCreateImageArea`, `QuickCreateCarousel`.
  - `QuickCreateStickyDefaults`, `QuickCreatePhotoImporter`, `QuickCreatePhotoStorage`, `QuickCreateGallerySaver`.
  - `QuickCreateAnalytics` + `AnalyticsEvents.IssueQuickCreation*` typed constructors (codegen).
  - `IssuesRepository.updateDescriptionWithSideEffect(issueUid, description)` — already used by `finalizeIssue` at `QuickCreateProcessor.kt:282`; we call the same method on enrichment failure.
- **WIP Issue-List UI** (build on, do not duplicate):
  - `log/compose/ai/AiIssueItem.kt`, `IssueRowBody.kt`, `IssuesSectionHeader.kt`, `IssuesListSections.kt`, `AiRowPreview.kt`.
  - `IssueItem.kt` (already delegates to `IssueRowBody`).
  - `IssuesListComposable.kt` (already partitions and calls `LazyIssueList(aiIssues, regularIssues, aiStatus, ...)`).
  - WIP string resources and `ic_star_4_point.xml` (to be moved per SR-15).
- **AlloyCompose components** already in use: `AlloyInputField`, `AlloyExpandableInputField`, `AlloyText`, `AlloyBadge`, `AlloyFab`, `AlloyCheckBox`, `AlloyDivider`, `AlloyColorPallete`, `AlloySwitch`.
- **`:android:speechtotext`** already depended on via `android/issues/build.gradle.kts:43`.

### New constructs (all in `:android:issues`)

- `QuickCreateAiToggleRow.kt` (composable).
- `ImageToBase64Converter.kt` (injectable).
- New Mobius Actions/Effects/State fields and RenderData fields (listed in Decision 1).

### New accessors (in `:android:components`)

- `IssuesFeatureFlag.isAiSuggestionsFlagOn`, `.isAiEnabledFlagOn`.

### DI wiring notes

- `IssueSuggestionRepository` and `GGIssueAiEnablementRepository` are already exposed via `ProjectObjectRepository.kt:211, 2311-2318`. Trinity needs to add constructor injection parameters to `IssuesViewModel`, `QuickCreateProcessor`, and the new `AiAvailabilityProvider` usage (inline inside the Processor is fine — no new top-level provider class needed).
- An `@ApplicationScope` `CoroutineScope` (with `@IoScheduler` dispatcher) is required for the enrichment effect to survive Activity death (SR-19). Verify it exists in the app graph; if absent, Trinity creates one in the app-level Dagger module.

---

## Out of Scope

- **pgf/ changes** — PGF layer is shipped. Do not modify `IssueSuggestionRepository`, `GGIssueAiEnablementRepository`, `GGIssueAiEnablementSync`, or any KMP model.
- **iOS** — already shipped (v1.0).
- **Base Quick Create (Phase 1)** — do not rewrite `finalizeIssue` beyond the single AI-effective branch; do not touch camera/photo/type/placement logic.
- **LLM prompts and server field-mapping** — spec :312. Server owns this.
- **Full Issue Details AI review UI** — spec :313. Tap routes to details; the details screen's AI review UI is a separate spec.
- **Speech-to-Text full architecture** — `:android:speechtotext` module is consumed read-only. Per Decision 8 spec deviation, there is **no STT delta** this phase — base Quick Create's existing STT behavior is preserved unchanged. The `issuesQuickCreateSpeechToText` flag/locale gating is PARKED for a future phase.
- **Paparazzi screenshot harness** — plan §8 assumes one; Android currently has none in `:android:issues`. Tracked as accepted tech debt (see research NEO_QUESTION #8 resolution in Verification Checkpoints below). Trinity may propose adding it as a follow-up ticket.
- **AI indicator design-system migration** — the AI indicator drawable currently lives in `:android:issues/res/drawable/ic_star_4_point.xml`. SR-15 moves it to alloycompose for final placement; if Trinity encounters resistance (e.g., blocking review by AlloyCompose owners), ship with the drawable in-place and open a follow-up to migrate.

---

## Verification Checkpoints (for human review)

- [ ] **CP1 — after Decision 4 wiring (VM `aiStatus` field + combine)**: confirm the compile gap at `IssuesListComposable.kt:225` is resolved and the screen builds cleanly against live `IssuesViewModel`. Run `./gradlew :android:issues:assembleDebug`.
- [ ] **CP2 — after Decision 5a/5b/5c UX reconciliation**: visual review of `AiRowPreview.kt` — PROCESSING ring reads as Purple700 → AutodeskBlue700 gradient; ERROR state shows trailing `ic_warning`; no "Issues list" header; `AiRegularSectionDivider` appears between sections.
- [ ] **CP3 — after Decisions 1+2+3 Mobius wiring**: open Quick Create with `issuesQuickCreateAiSuggestions` ON — confirm `RenderData.isAiAvailable = true`. Kill the app, flip the flag OFF, reopen — confirm `isAiAvailable = false`. Verify the snapshot is read only once per session (add a log or Mobius integration test that counts invocations of `isAiEnabled()`).
- [ ] **CP4 — after Decision 7 payload-only cap**: manual test — capture 7 photos with AI ON, verify camera button stays enabled throughout, counter overlay displays "5/5" from the 5th photo onward, save issue, confirm enrichment payload contains exactly the first 5 photos (check analytics `ai_suggestion_received.image_count == 5` and/or server log). Toggle AI OFF: counter overlay disappears, camera still enabled.
- [ ] **CP5 — after SR-15 AlloyCompose changes**: PR to `:android:alloycompose` reviewed separately by alloycompose owners before merge into issues branch. The drawable move is the hardest merge — do it early.
- [ ] **CP6 — after SR-18/19/20 enrichment pipeline**: end-to-end test — create issue with AI ON and 3 photos, close Quick Create immediately, confirm enrichment completes; induce server failure, confirm prompt saved to description and no blocking UI appears.
- [ ] **CP7 — after Decision 9 analytics**: verify with analytics debug viewer that all 8 Phase-2 events fire with the correct `ai_flag_enabled` and `ai_toggle_state` values in all four availability × toggle permutations.

---

## Risk Register

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|-----------|
| Base64 payload size for 5 full-res JPEGs exceeds server limit | Medium | High | `ImageToBase64Converter` resizes to 1600px max edge + JPEG quality 0.9 (matches iOS) — keeps each image ≤ ~2MB base64. Verify max request size with backend owners before implementation. |
| Application-scoped coroutine for enrichment leaks | Low | Medium | Use a `SupervisorJob` + `CoroutineExceptionHandler` tied to app lifecycle; not unbounded `GlobalScope`. Log errors through existing `Logging.log`. |
| `stickyHeader` experimental API changes | Very low | Low | Already `@OptIn(ExperimentalFoundationApi::class)` in WIP. Monitor Compose BOM upgrades. |
| Drift between `IssueItem` and `AiIssueItem` rows | Medium | Medium | Plan §8 mitigations in place (shared `IssueRowBody`, shared `STATUS_CIRCLE_SIZE`, `AiRowPreview` stacked preview). Accept Paparazzi absence as tech debt; Trinity may open follow-up. |
| Optional migration trap (from user memory) hitting this feature | Low | Medium | Mobius/Compose layer touched here does not use `Optional.None` reference equality. Skip this risk. |
| DI graph missing `@ApplicationScope CoroutineScope` | Low | Medium | If absent, Trinity adds it to the app-level Dagger module. One-time cost. |
| AlloyCompose module owner pushback on AI indicator PR | Medium | Low | Fallback: keep drawable in issues module, pass `painter: Painter` into `AlloyInputField` as a workaround. KB `alloy-input-field-ai-indicator.md` already documents the Approach 3 design; resistance should be minimal. |
| Paparazzi absence hides row drift | Medium | Low | `AiRowPreview.kt` exists as a manual preview; Trinity runs it in Android Studio preview for every PR. Accepted as tech debt. |

---

## Open Questions Resolved

Each research NEO_QUESTION is answered here (by section reference):

- **NEO_QUESTION #1** (Reconcile UI drift in AI row indicator): answered in Decision 5a — keep WIP Blue→Purple gradient; fix processing ring to `Purple700 → AutodeskBlue700 → Transparent`.
- **NEO_QUESTION #2** (Error state overlay vs trailing icon): answered in Decision 5b — keep WIP trailing icon.
- **NEO_QUESTION #3** (Second sticky header): answered in Decision 5c — remove; use `AiRegularSectionDivider` per plan §9.
- **NEO_QUESTION #4** (`IssuesViewModel.Data.aiStatus` shape + combine strategy): answered in Decisions 3 + 4 — add field + `combine` the flows.
- **NEO_QUESTION #5** (Enrichment-survives-navigation scope): answered in SR-19 — Application-scoped `CoroutineScope`, not WorkManager.
- **NEO_QUESTION #6** (AI toggle source of truth): answered in Decision 6 — Mobius State is the in-session source of truth; `QuickCreateStickyDefaults` is the persisted backing store; single-direction sync.
- **NEO_QUESTION #7** (Reactivity of `isAiEnabled`): answered in Decision 2 — one-shot snapshot at session start (no reactive stream, no `OneQuery<T>.flow()` bridge).
- **NEO_QUESTION #8** (Paparazzi prerequisite): answered in "Out of Scope" — accepted as tech debt.

Non-architectural questions for the user (forwarded, not blocking):

- Drawable location (`:android:issues` vs `:android:alloycompose`) — ADR specifies move to alloycompose per SR-15 but allows fallback (see risk register).
- Confirm staged-deleted `AiSectionHeader.kt` stays deleted — ADR confirms intentional per Decision 5 UX resolution.
