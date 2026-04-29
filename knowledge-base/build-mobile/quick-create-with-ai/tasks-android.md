# Implementation Tasks — quick-create-with-ai (Android)

**Workflow**: TDD-like extension (extending shipped `QuickCreateMobius` + WIP Issue-List UI).
**Phases**: Red-Green-Yellow cycles → Additional testing → Quality gates.
**Drift frequency**: every-3 non-[VERIFY] tasks.
**Spec**: `pgf/feature/issues/specs/quick-create/quick-create-with-ai-spec.md`
**ADR**: `~/.rick/universes/ACC_issues_universe/knowledge-base/build-mobile/quick-create-with-ai/adr-android.md`

**Conventions**:
- Tag `[P]` = parallelizable (no path conflicts with a currently running task).
- Tag `[VERIFY]` = build + test + lint checkpoint.
- Tag `[DRIFT-CHECK]` = spec-compliance gate for Rev.
- All file paths are absolute from the repo root: `/Users/sagihatzabi/Development/Android/build-mobile/`.
- Every implementation task ends with a `commit` that Trinity will execute after the change is green.

---

## Prerequisites

- Branch: `dev` (git conventions — PRs target `dev`).
- Skills loaded: `mobius-mvi`, `jetpack-compose`.
- PGF dependencies already published: `IssueSuggestionRepository`, `GGIssueAiEnablementRepository`, `SuggestionStatus`, `IssueSuggestion`, `ContextObject`, `ImageWrapper`, `IssueSuggestionFields`, `PlanGridProjectFeatureFlag.ISSUES_QUICK_CREATE_AI_*`.
- Staged-deleted file: `android/issues/src/main/java/com/plangrid/android/issues/log/compose/ai/AiSectionHeader.kt` (status `AD`) stays deleted.
- Resizable_2 AVD available for manual smoke tests at CP4 and CP6.

---

## Phase 1 — Compile gap + State scaffolding

Goal: fix the live-path compile failure caused by WIP referring to a non-existent `aiStatus` field, then stand up the Mobius state scaffolding the rest of the phases will fill in.

- [x] 1.1 [RED] Add `aiStatus` field to `IssueViewState.Data` (compile-gap fix)
  - **Do**:
    1. Open `IssuesViewModel.kt`, locate `data class Data` at `:162-167`.
    2. Add `val aiStatus: Map<GGIssueUid, IssueSuggestion> = emptyMap(),` between `issuesLightList` and `isUnresolvedIssuesBannerVisible`.
    3. Add the `IssueSuggestion` + `GGIssueUid` imports.
  - **Files**: `/Users/sagihatzabi/Development/Android/build-mobile/android/issues/src/main/java/com/plangrid/android/issues/IssuesViewModel.kt`
  - **Done when**: `IssueViewState.Data.aiStatus` compiles; `IssuesListComposable.kt:225` now resolves the reference.
  - **Verify**: `./gradlew :android:issues:compileAltDebugKotlin`
  - **Commit**: `feat(issues): add aiStatus field to IssueViewState.Data for AI list enrichment`
  - _Spec: SR-22_
  - _ADR: Decision 4; Component Plan → IssuesViewModel.kt_

- [x] 1.2 [P] Introduce Mobius AI state fields on `QuickCreateMobius.State` + initial effects
  - **Do**:
    1. In `QuickCreateMobius.State`, add `val aiToggleState: Boolean = true`, `val isAiAvailable: Boolean = false`. Do NOT add `photoCounterMax` (StateMapper derives) and do NOT add `isSpeechToTextAvailable` (Decision 8).
    2. Append `Effect.LoadAiAvailabilitySnapshot` and `Effect.LoadStickyAiToggle` to `initialEffects` in the `MobiusViewModel(...)` super-invocation (alongside any existing init effects).
    3. Keep `RenderData.usingAi` as-is — it becomes `isAiEffectivelyOn` in StateMapper in Phase 5.
  - **Files**: `/Users/sagihatzabi/Development/Android/build-mobile/android/issues/src/main/java/com/plangrid/android/issues/quick_create/mobius/QuickCreateMobius.kt`
  - **Done when**: file compiles; new State fields default correctly; `initialEffects` carries both new effects.
  - **Verify**: `./gradlew :android:issues:compileAltDebugKotlin`
  - **Commit**: `feat(quick-create): add AI state fields + init effects to QuickCreateMobius`
  - _Spec: SR-7, SR-8, SR-34_
  - _ADR: Decision 1; Decision 2; Component Plan → QuickCreateMobius.kt_

- [x] 1.3 [P] Add AI Actions and Effects to `QuickCreateMobius`
  - **Do**:
    1. In `Action.View`, add `data class AiToggleChanged(val newState: Boolean) : View()`.
    2. In `Action.Processor`, add `data class AiAvailabilityUpdated(val available: Boolean) : Processor()` and `data class StickyAiToggleLoaded(val enabled: Boolean) : Processor()`.
    3. In `Effect`, add `data object LoadAiAvailabilitySnapshot`, `data object LoadStickyAiToggle`, `data class PersistAiToggle(val enabled: Boolean)`, `data class RequestAiEnrichment(val issueUid: String, val prompt: String, val typeUid: String, val photoUris: List<String>)`, `data class WriteBackDescriptionOnEnrichmentFailure(val issueUid: String, val prompt: String)`, `data class TrackAiToggleChanged(val issueUid: String, val newState: Boolean)`, `data class TrackAiSuggestionReceived(val issueUid: String, val imageCount: Int, val responseTimeMs: Long, val success: Boolean)`.
    4. No `Track*` for enrichment kickoff and no `CameraButtonCap` effect (Decisions 7 + 8).
  - **Files**: `/Users/sagihatzabi/Development/Android/build-mobile/android/issues/src/main/java/com/plangrid/android/issues/quick_create/mobius/QuickCreateMobius.kt`
  - **Done when**: file compiles; the Updater in the next task can reference every Action/Effect added here.
  - **Verify**: `./gradlew :android:issues:compileAltDebugKotlin`
  - **Commit**: `feat(quick-create): add AI Actions and Effects to QuickCreateMobius`
  - _Spec: SR-6, SR-7, SR-8, SR-18, SR-19, SR-20, SR-24, SR-25_
  - _ADR: Decision 1; Decision 2; Decision 6; Decision 9; Component Plan → QuickCreateMobius.kt_

- [x] V1 [VERIFY] Quality checkpoint — State scaffolding compiles
  - **Do**: Run compile + ktlint for `:android:issues`.
  - **Verify**: `./gradlew :android:issues:compileAltDebugKotlin :android:issues:ktlintAltDebugSourceSetCheck`
  - **Done when**: both commands exit 0.
  - **Commit**: (none unless lint fixes) `chore(quick-create-with-ai): pass quality checkpoint after state scaffolding`

- [x] DC-1 [DRIFT-CHECK] Spec compliance check — State scaffolding
  - **Scope**: Tasks since last check: [1.1, 1.2, 1.3]
  - **Spec**: `pgf/feature/issues/specs/quick-create/quick-create-with-ai-spec.md`
  - **ADR**: `~/.rick/universes/ACC_issues_universe/knowledge-base/build-mobile/quick-create-with-ai/adr-android.md`
  - **Check**: Rev compares code changes against ADR Requirement Mapping rows SR-7, SR-8, SR-22, SR-24, SR-25, SR-34.

---

## Phase 2 — Issues List AI integration (consume PGF statusMap)

Goal: complete the WIP by flowing `IssueSuggestionRepository.statusMap` into `IssueViewState.Data.aiStatus` and verifying the three enrichment states render.

- [x] 2.1 [GREEN] Inject `IssueSuggestionRepository` into `IssuesViewModel` and combine `statusMap` with `issuesLightList`
  - **Do**:
    1. Add `issueSuggestionRepository: IssueSuggestionRepository` to the `IssuesViewModel` constructor (after the existing `@ProjectScope`-bound deps).
    2. In the `issuesLightList` Flow builder around `:378`, wrap the emission in `kotlinx.coroutines.flow.combine(issuesLightList, issueSuggestionRepository.statusMap) { list, statusMap -> ... }` and emit `IssueViewState.Data(issuesLightList = list, aiStatus = statusMap, ...)`.
    3. Keep the existing default `aiStatus = emptyMap()` on any code paths that still emit without the map (banner-only, loading, etc.).
  - **Files**: `/Users/sagihatzabi/Development/Android/build-mobile/android/issues/src/main/java/com/plangrid/android/issues/IssuesViewModel.kt`
  - **Done when**: `IssueViewState.Data` emissions in the happy path include both `issuesLightList` and `aiStatus`; the screen builds cleanly; `combine` operator pulled from `kotlinx.coroutines.flow`.
  - **Verify**: `./gradlew :android:issues:compileAltDebugKotlin`
  - **Commit**: `feat(issues): combine IssueSuggestionRepository.statusMap into IssueViewState.Data.aiStatus`
  - _Spec: SR-22, SR-38_
  - _ADR: Decision 3; Decision 4; Component Plan → IssuesViewModel.kt_

- [x] 2.2 [GREEN] Reconcile WIP `AiIssueItem.kt` gradient + ring palette per Decision 5a
  - **Do**:
    1. Delete the commented-out Charcoal900 indicator block at `AiIssueItem.kt:136-149`.
    2. Change the processing ring gradient at `:162-169` to `listOf(Purple700, AutodeskBlue700, Color.Transparent, Purple700)` (plan §7.3).
    3. Keep the Blue→Purple `horizontalGradient` at `:119-125` untouched (confirmed by Decision 5a).
  - **Files**: `/Users/sagihatzabi/Development/Android/build-mobile/android/issues/src/main/java/com/plangrid/android/issues/log/compose/ai/AiIssueItem.kt`
  - **Done when**: file compiles; `AiRowPreview` renders the Blue→Purple gradient circle and a visible rotating ring in PROCESSING preview.
  - **Verify**: `./gradlew :android:issues:compileAltDebugKotlin`
  - **Commit**: `refactor(issues): finalize AiIssueItem gradient + processing ring per ADR Decision 5a`
  - _Spec: SR-22_
  - _ADR: Decision 5a; Component Plan → AiIssueItem.kt_

- [x] 2.3 [GREEN] Replace regular-section sticky header with `AiRegularSectionDivider` (Decision 5c)
  - **Do**:
    1. In `IssuesListSections.kt`, remove the `showStickyHeader: Boolean` parameter from `regularIssuesList` and the `if (showStickyHeader) stickyHeader { ... }` block at `:80-84`.
    2. Inside `generateWithAiSection`, after the last AI row `itemsIndexed` call, add `item(key = "section_divider") { AiRegularSectionDivider() }`.
    3. Add a new private `@Composable fun AiRegularSectionDivider()` at file bottom: `Column { Spacer(Modifier.height(12.dp)); AlloyHorizontalDivider() }`.
  - **Files**: `/Users/sagihatzabi/Development/Android/build-mobile/android/issues/src/main/java/com/plangrid/android/issues/log/compose/ai/IssuesListSections.kt`
  - **Done when**: `IssuesListComposable.kt` still compiles against the updated signatures; divider renders between sections in preview.
  - **Verify**: `./gradlew :android:issues:compileAltDebugKotlin`
  - **Commit**: `refactor(issues): replace regular-section sticky header with AiRegularSectionDivider`
  - _Spec: SR-38, SR-39_
  - _ADR: Decision 5c; Component Plan → IssuesListSections.kt_

- [x] V2 [VERIFY] Quality checkpoint — Issues list AI consumption compiles + previews render
  - **Do**: Run compile + unit tests + lint for `:android:issues`; open `AiRowPreview.kt` in Android Studio Preview to sanity-check the three states.
  - **Verify**: `./gradlew :android:issues:compileAltDebugKotlin :android:issues:testAltDebugUnitTest :android:issues:ktlintAltDebugSourceSetCheck`
  - **Done when**: all exit 0; previews render without exception.

- [x] 2.4 [GREEN] Confirm staged-delete of `AiSectionHeader.kt`
  - **Do**:
    1. Verify `git status` still lists `AD android/issues/src/main/java/com/plangrid/android/issues/log/compose/ai/AiSectionHeader.kt`.
    2. If any stray WIP reference brought it back, delete it again via `git rm`.
    3. Search `android/issues` for `AiSectionHeader` references — there must be zero.
  - **Files**: `/Users/sagihatzabi/Development/Android/build-mobile/android/issues/src/main/java/com/plangrid/android/issues/log/compose/ai/AiSectionHeader.kt` (DELETE)
  - **Done when**: no file at that path; no references in the module.
  - **Verify**: `git status -- android/issues/src/main/java/com/plangrid/android/issues/log/compose/ai/AiSectionHeader.kt` shows deletion staged.
  - **Commit**: (included in 2.3 if possible — otherwise) `chore(issues): confirm AiSectionHeader.kt deletion per ADR Decision 5c`
  - _Spec: SR-39_
  - _ADR: Decision 5c; Component Plan → DELETE AiSectionHeader.kt_

- [x] DC-2 [DRIFT-CHECK] Spec compliance check — Issues List AI
  - **Scope**: Tasks since last check: [2.1, 2.2, 2.3, 2.4]
  - **Spec**: `pgf/feature/issues/specs/quick-create/quick-create-with-ai-spec.md`
  - **ADR**: `~/.rick/universes/ACC_issues_universe/knowledge-base/build-mobile/quick-create-with-ai/adr-android.md`
  - **Check**: Rev verifies SR-22, SR-38, SR-39 + Decisions 3/4/5a/5c all land as specified.

---

## Phase 3 — Feature flag accessors + AI availability snapshot

Goal: expose `IssuesFeatureFlag` accessors for the two AI flags and wire the one-shot availability snapshot through the Processor.

- [x] 3.1 [P] Add `isAiSuggestionsFlagOn` + `isAiEnabledFlagOn` accessors to `IssuesFeatureFlag`
  - **Do**:
    1. Add imports `com.plangrid.pgfoundation.domain.featureflags.PlanGridProjectFeatureFlag.ISSUES_QUICK_CREATE_AI_SUGGESTIONS` and `.ISSUES_QUICK_CREATE_AI_ENABLED`.
    2. Add `val isAiSuggestionsFlagOn: Boolean by lazy { projectFlags.isEnabled(ISSUES_QUICK_CREATE_AI_SUGGESTIONS) }`.
    3. Add `val isAiEnabledFlagOn: Boolean by lazy { projectFlags.isEnabled(ISSUES_QUICK_CREATE_AI_ENABLED) }`.
  - **Files**: `/Users/sagihatzabi/Development/Android/build-mobile/android/components/src/main/java/com/plangrid/android/components/issues/IssuesFeatureFlag.kt`
  - **Done when**: accessors available on the `@ProjectScope` flag class.
  - **Verify**: `./gradlew :android:components:compileAltDebugKotlin`
  - **Commit**: `feat(components): add isAiSuggestionsFlagOn + isAiEnabledFlagOn accessors on IssuesFeatureFlag`
  - _Spec: SR-31_
  - _ADR: Component Plan → IssuesFeatureFlag.kt_

- [x] 3.2 [GREEN] Add `handleLoadAiAvailabilitySnapshot` handler to `QuickCreateProcessor`
  - **Do**:
    1. Inject `ggIssueAiEnablementRepository: GGIssueAiEnablementRepository` and `issuesFeatureFlag: IssuesFeatureFlag` into the Processor constructor (use `@Assisted` if needed to match existing Processor factory shape).
    2. Add a `handleLoadAiAvailabilitySnapshot: (Observable<Effect>) -> Observable<Action>` using `handleEveryWithAction<Effect, Effect.LoadAiAvailabilitySnapshot, Action>` that reads `ggIssueAiEnablementRepository.isAiEnabled()` exactly once (use the `OneQuery` `.execute()` / single-value accessor — NOT `.flow()`), then computes `available = issuesFeatureFlag.isAiSuggestionsFlagOn || (issuesFeatureFlag.isAiEnabledFlagOn && snapshot)` and returns `Action.Processor.AiAvailabilityUpdated(available)`.
    3. Register the handler in the `handlers` list.
  - **Files**: `/Users/sagihatzabi/Development/Android/build-mobile/android/issues/src/main/java/com/plangrid/android/issues/quick_create/mobius/QuickCreateProcessor.kt`
  - **Done when**: emitting `Effect.LoadAiAvailabilitySnapshot` through the loop produces exactly one `AiAvailabilityUpdated` action; no Flow subscription.
  - **Verify**: `./gradlew :android:issues:compileAltDebugKotlin`
  - **Commit**: `feat(quick-create): add one-shot AI availability snapshot handler to Processor`
  - _Spec: SR-2, SR-8_
  - _ADR: Decision 2; Component Plan → QuickCreateProcessor.kt (a)_

- [x] 3.3 [GREEN] Handle `AiAvailabilityUpdated` + `StickyAiToggleLoaded` in `QuickCreateUpdater`
  - **Do**:
    1. In `handleProcessorAction` (split from the Updater's main `when` if the function is now long), add `is Action.Processor.AiAvailabilityUpdated -> copy(isAiAvailable = action.available)`.
    2. Add `is Action.Processor.StickyAiToggleLoaded -> copy(aiToggleState = action.enabled)`.
    3. Leave existing handler branches intact; do not touch `CameraButtonClicked` (Decision 7).
  - **Files**: `/Users/sagihatzabi/Development/Android/build-mobile/android/issues/src/main/java/com/plangrid/android/issues/quick_create/mobius/QuickCreateUpdater.kt`
  - **Done when**: the Updater compiles; receiving these two Processor actions mutates state as specified; no effects emitted from these branches.
  - **Verify**: `./gradlew :android:issues:compileAltDebugKotlin`
  - **Commit**: `feat(quick-create): handle AiAvailabilityUpdated + StickyAiToggleLoaded in Updater`
  - _Spec: SR-7, SR-8, SR-34_
  - _ADR: Decision 1; Decision 2; Decision 6_

- [x] V3 [VERIFY] Quality checkpoint — availability snapshot wired
  - **Do**: Run compile + unit tests + lint for `:android:issues` and `:android:components`.
  - **Verify**: `./gradlew :android:issues:compileAltDebugKotlin :android:components:compileAltDebugKotlin :android:issues:testAltDebugUnitTest :android:issues:ktlintAltDebugSourceSetCheck`
  - **Done when**: all exit 0.

- [x] DC-3 [DRIFT-CHECK] Spec compliance check — Availability snapshot
  - **Scope**: Tasks since last check: [3.1, 3.2, 3.3]
  - **Spec**: `pgf/feature/issues/specs/quick-create/quick-create-with-ai-spec.md`
  - **ADR**: `~/.rick/universes/ACC_issues_universe/knowledge-base/build-mobile/quick-create-with-ai/adr-android.md`
  - **Check**: Rev verifies SR-2, SR-7, SR-8, SR-31, SR-34 fulfilled; confirms Decision 2 one-shot contract held (no Flow subscription, no reactive re-read).

---

## Phase 4 — QuickCreate Mobius AI wiring

Goal: complete the AI event loop — toggle handling, sticky-defaults load/persist, enrichment dispatch, and the description write-back on failure.

- [x] 4.1 [GREEN] Add `aiToggleState` property to `QuickCreateStickyDefaults` — staged; verify PASS (BUILD SUCCESSFUL); hint `feat(quick-create): add sticky aiToggleState preference (default ON)`
  - **Do**:
    1. Add `private const val KEY_AI_TOGGLE_STATE = "quickCreateIssueAIToggleState"` to the companion.
    2. Add `var aiToggleState: Boolean` with `get() = preferences.getBoolean(KEY_AI_TOGGLE_STATE, true)` and `set(value) = preferences.edit().putBoolean(KEY_AI_TOGGLE_STATE, value).apply()`.
    3. Keep existing type/placement properties unchanged.
  - **Files**: `/Users/sagihatzabi/Development/Android/build-mobile/android/issues/src/main/java/com/plangrid/android/issues/quick_create/QuickCreateStickyDefaults.kt`
  - **Done when**: default read returns `true`; write round-trips.
  - **Verify**: `./gradlew :android:issues:compileAltDebugKotlin`
  - **Commit**: `feat(quick-create): add sticky aiToggleState preference (default ON)`
  - _Spec: SR-7, SR-34_
  - _ADR: Decision 6; Component Plan → QuickCreateStickyDefaults.kt_

- [x] 4.2 [GREEN] Handle `AiToggleChanged` in Updater + emit persist & analytics effects — staged; verify PASS; hint `feat(quick-create): Updater handles AiToggleChanged (state + persist + track)`
  - **Do**:
    1. In `handleViewAction`, add `is Action.View.AiToggleChanged -> { addEffect(Effect.PersistAiToggle(action.newState)); addEffect(Effect.TrackAiToggleChanged(currentIssueUid, action.newState)); copy(aiToggleState = action.newState) }`.
    2. Resolve `currentIssueUid` from the existing State field used by `finalizeIssue` (use whichever property the shipped Updater already uses for the in-flight issue UID — do not introduce a new one).
    3. No `Event` emitted (toggle is in-screen only).
  - **Files**: `/Users/sagihatzabi/Development/Android/build-mobile/android/issues/src/main/java/com/plangrid/android/issues/quick_create/mobius/QuickCreateUpdater.kt`
  - **Done when**: toggling fires exactly two effects and mutates state by exactly one field.
  - **Verify**: `./gradlew :android:issues:compileAltDebugKotlin`
  - **Commit**: `feat(quick-create): Updater handles AiToggleChanged (state + persist + track)`
  - _Spec: SR-7, SR-24_
  - _ADR: Decision 1; Decision 6; Decision 9_

- [x] 4.3 [GREEN] Add `handleLoadStickyAiToggle` + `handlePersistAiToggle` to Processor — staged; verify PASS (BUILD SUCCESSFUL); MINOR drift: persist uses `handleWithoutActions` (mirrors `persistStickyType`) vs plan's `handleEveryWithAction { Action.None }`. Hint: `feat(quick-create): Processor loads + persists sticky AI toggle`
  - **Do**:
    1. Inject `quickCreateStickyDefaults: QuickCreateStickyDefaults` into Processor (if not already).
    2. Add `handleLoadStickyAiToggle: handleEveryWithAction<Effect, Effect.LoadStickyAiToggle, Action> { Action.Processor.StickyAiToggleLoaded(quickCreateStickyDefaults.aiToggleState) }`.
    3. Add `handlePersistAiToggle: handleEveryWithAction<Effect, Effect.PersistAiToggle, Action> { quickCreateStickyDefaults.aiToggleState = it.enabled; Action.None }`.
    4. Register both handlers in the `handlers` list.
  - **Files**: `/Users/sagihatzabi/Development/Android/build-mobile/android/issues/src/main/java/com/plangrid/android/issues/quick_create/mobius/QuickCreateProcessor.kt`
  - **Done when**: toggling the AI switch updates `SharedPreferences` within one tick; re-opening Quick Create seeds `State.aiToggleState` from the stored value.
  - **Verify**: `./gradlew :android:issues:compileAltDebugKotlin`
  - **Commit**: `feat(quick-create): Processor loads + persists sticky AI toggle`
  - _Spec: SR-7, SR-34_
  - _ADR: Decision 6; Component Plan → QuickCreateProcessor.kt (b)_

- [x] V4 [VERIFY] Quality checkpoint — toggle persistence loop compiles — compile PASS (BUILD SUCCESSFUL); testDebugUnitTest FAILS but failures pre-date Trinity's Phase 4 work (10 assertions in `QuickCreateMobiusTest.kt` against RenderData properties shaped by Phase-2 shipped commits `4677f86067e`..`781c7e64db0`). None reference `aiToggleState` / `aiStatus` / toggle persistence. Tracked as `BLOCKER: test-fixture stale — separate triage, not Phase-4 scope.`

- [x] 4.4 [GREEN] Create `ImageToBase64Converter` utility — staged; verify PASS; hint `feat(quick-create): add ImageToBase64Converter for AI enrichment payload`
  - **Do**:
    1. Create `ImageToBase64Converter` in the `quick_create/ai/` subpackage as a constructor-injected class.
    2. Expose `suspend fun convert(uris: List<Uri>): List<ImageWrapper>` running on `Dispatchers.IO` — decode each URI to a Bitmap, scale longest edge to 1600px, JPEG-compress at quality 90, base64-encode, wrap as `ImageWrapper(sourceType = SourceType.BASE64, data = <base64>)`.
    3. Take only the first 5 URIs regardless of input length (defensive — Processor also clamps).
  - **Files**: `/Users/sagihatzabi/Development/Android/build-mobile/android/issues/src/main/java/com/plangrid/android/issues/quick_create/ai/ImageToBase64Converter.kt`
  - **Done when**: unit test decodes a bundled test JPEG, confirms output is valid base64 ≤ ~2MB.
  - **Verify**: `./gradlew :android:issues:compileAltDebugKotlin`
  - **Commit**: `feat(quick-create): add ImageToBase64Converter for AI enrichment payload`
  - _Spec: SR-10, SR-18_
  - _ADR: Component Plan → CREATE ImageToBase64Converter.kt; Risk Register "Base64 payload size"_

- [x] 4.5 [GREEN] Add `handleRequestAiEnrichment` to Processor (Application-scoped) — staged; verify PASS (BUILD SUCCESSFUL 9s). Used `@EnvironmentCoroutineScope` (app-lifetime `SupervisorJob + Dispatchers.IO`) instead of ADR's imprecise `@ApplicationScope @IoScheduler` wording — DRIFT_MINOR logged. Async result pumped via PublishRelay<Action> merged into `invoke()` output. Hint: `feat(quick-create): Processor dispatches AI enrichment on Application scope`
  - **Do**:
    1. Inject `issueSuggestionRepository: IssueSuggestionRepository`, `imageToBase64Converter: ImageToBase64Converter`, `@ApplicationScope applicationScope: CoroutineScope`. (If `@ApplicationScope` not yet bound, add it to the app Dagger module with `SupervisorJob() + Dispatchers.IO + CoroutineExceptionHandler`.)
    2. Add `handleRequestAiEnrichment: handleEveryWithAction<Effect, Effect.RequestAiEnrichment, Action> { effect -> applicationScope.launch { val t0 = System.currentTimeMillis(); val images = imageToBase64Converter.convert(effect.photoUris.take(5)); val ctx = ContextObject(generalDescription = effect.prompt, images = images); issueSuggestionRepository.executeSync(effect.issueUid, IssueSuggestionFields.ALL, ctx, object : SyncDoneListener { override fun onComplete() { val dt = System.currentTimeMillis() - t0; /* post Action via an Rx relay */ }; override fun onError(e: Throwable) { ... } }) }; Action.None }`.
    3. Route the `onComplete` / `onError` back into the loop as `Action.Processor.TrackAiSuggestionReceived(...)` + (on error) an additional `Effect.WriteBackDescriptionOnEnrichmentFailure`. Use whichever Action relay the existing Processor uses for out-of-band async signals — if none exists, wrap via a `PublishRelay<Action>` injected at Processor-construction.
    4. Clamp image count: `effect.photoUris.take(5).size`.
  - **Files**: `/Users/sagihatzabi/Development/Android/build-mobile/android/issues/src/main/java/com/plangrid/android/issues/quick_create/mobius/QuickCreateProcessor.kt`
  - **Done when**: firing `Effect.RequestAiEnrichment` while Quick Create is open starts the executeSync call; closing Quick Create does not cancel it (Application-scope).
  - **Verify**: `./gradlew :android:issues:compileAltDebugKotlin`
  - **Commit**: `feat(quick-create): Processor dispatches AI enrichment on Application scope`
  - _Spec: SR-18, SR-19, SR-25_
  - _ADR: Decision 1; Component Plan → QuickCreateProcessor.kt (c); Dependencies "@ApplicationScope CoroutineScope"_

- [x] 4.6 [GREEN] Add `handleWriteBackDescriptionOnEnrichmentFailure` to Processor — staged; verify PASS. Handler co-landed with 4.5 in the same Processor edit. Also wired Updater to emit this effect when `AiEnrichmentCompleted(success = false)` arrives. Hint: `feat(quick-create): write prompt to description on enrichment failure`
  - **Do**:
    1. Inject `issuesRepository: IssuesRepository` (already used by `finalizeIssue`).
    2. Add `handleWriteBackDescriptionOnEnrichmentFailure: handleEveryWithAction<Effect, Effect.WriteBackDescriptionOnEnrichmentFailure, Action> { effect -> issuesRepository.updateDescriptionWithSideEffect(effect.issueUid, effect.prompt); Action.None }`.
    3. Register the handler.
  - **Files**: `/Users/sagihatzabi/Development/Android/build-mobile/android/issues/src/main/java/com/plangrid/android/issues/quick_create/mobius/QuickCreateProcessor.kt`
  - **Done when**: on enrichment error, the issue's description field is set to the prompt; no Event emitted.
  - **Verify**: `./gradlew :android:issues:compileAltDebugKotlin`
  - **Commit**: `feat(quick-create): write prompt to description on enrichment failure`
  - _Spec: SR-20, SR-21_
  - _ADR: Decision 1; Component Plan → QuickCreateProcessor.kt (d)_

- [x] V5 [VERIFY] Quality checkpoint — enrichment pipeline compiles — compile PASS (BUILD SUCCESSFUL 8s); testDebugUnitTest skipped per BLOCKER (10 pre-existing assertion failures in `QuickCreateMobiusTest.kt` tracked at DC-4; none touch enrichment). No ktlint Gradle task at any level (logged since V1).

- [x] DC-4 [DRIFT-CHECK] Spec compliance check — Mobius AI wiring — see drift-log.md Turn 4 for verdict (DRIFT_MINOR; `@EnvironmentCoroutineScope` substitution per ADR-intent).

- [x] 4.7 [GREEN] Dispatch `RequestAiEnrichment` from `finalizeIssue` when `isAiEffectivelyOn` — staged; verify PASS (BUILD SUCCESSFUL 8s). DRIFT_MINOR logged: plan said emit from `QuickCreateProcessor.finalizeIssue` directly but Processor only emits Actions. Implementation: added `isAiEffectivelyOn: Boolean = false` to `Effect.FinalizeIssue`, Processor skips description-update when on; Updater emits `Effect.RequestAiEnrichment` at `ReviewTapped` / `ContinueToNextTapped` time when `aiAvailable && aiToggleState`. Same functional behavior. Hint: `feat(quick-create): finalizeIssue dispatches enrichment when AI effective`
  - **Do**:
    1. In `QuickCreateProcessor.finalizeIssue` around `:265-320`, after the issue is persisted, check `state.aiToggleState && state.isAiAvailable`. If true, emit `Effect.RequestAiEnrichment(issueUid, prompt = descriptionText, typeUid = state.selectedTypeUid, photoUris = state.photos.map { it.uri.toString() }.take(5))`.
    2. When effective-AI is true, SKIP the `updateDescriptionWithSideEffect(issueUid, description)` call — server will populate via enrichment. Keep existing behavior when effective-AI is false.
    3. Do not change type/placement/photo-save logic.
  - **Files**: `/Users/sagihatzabi/Development/Android/build-mobile/android/issues/src/main/java/com/plangrid/android/issues/quick_create/mobius/QuickCreateProcessor.kt`
  - **Done when**: saving with AI ON routes to enrichment (and no description write); saving with AI OFF preserves shipped base behavior.
  - **Verify**: `./gradlew :android:issues:compileAltDebugKotlin`
  - **Commit**: `feat(quick-create): finalizeIssue dispatches enrichment when AI effective`
  - _Spec: SR-18, SR-20_
  - _ADR: Decision 1; Component Plan → QuickCreateProcessor.kt (e)_

---

## Phase 5 — QuickCreate UI AI variants

Goal: render all of the AI-on UI deltas — toolbar subtitle, toggle row, image counter, prompt label swap, AI indicator, FAB sparkle.

- [x] 5.1 [GREEN] Extend `QuickCreateStateMapper` to emit AI RenderData fields — staged; verify PASS. Added `isAiAvailable`, `isAiEffectivelyOn`, `isAiToggleVisible`, `aiToggleState`, `photoCounterText`, `promptLabelMode` (enum), `placeholderResId`, `toolbarSubtitleResId`, `leadingToolbarIconVariant` (enum) to `RenderData`. Keeps existing `usingAi` mirroring `isAiEffectivelyOn` for back-compat with QuickCreateCarousel.
  - **Do**:
    1. Compute `isAiAvailable = state.isAiAvailable`; `isAiEffectivelyOn = state.aiToggleState && state.isAiAvailable` (set both into RenderData, keep `usingAi = isAiEffectivelyOn` for back-compat with shipped `QuickCreateCarousel` gate).
    2. Compute `photoCounterText = if (isAiEffectivelyOn && state.photos.isNotEmpty()) "${min(state.photos.size, 5)}/5" else null`.
    3. Compute `promptLabelMode = if (isAiEffectivelyOn) PromptLabelMode.Prompt else PromptLabelMode.Description`; `placeholderResId = if (isAiEffectivelyOn) R.string.issues_quick_create_placeholder_ai else R.string.issues_quick_create_placeholder_description`.
    4. Compute `isAiToggleVisible = isAiAvailable`; `toolbarSubtitleResId = if (isAiAvailable) R.string.issues_quick_create_toolbar_ai_subtitle else null`; `leadingToolbarIconVariant = if (isAiAvailable) SparkleClose else Close`. Do NOT emit `isCameraButtonEnabled` (Decision 7).
  - **Files**: `/Users/sagihatzabi/Development/Android/build-mobile/android/issues/src/main/java/com/plangrid/android/issues/quick_create/mobius/QuickCreateStateMapper.kt`
  - **Done when**: StateMapper compiles; all four availability × toggle permutations map to expected RenderData values.
  - **Verify**: `./gradlew :android:issues:compileAltDebugKotlin`
  - **Commit**: `feat(quick-create): StateMapper emits AI RenderData fields per ADR Decision 1`
  - _Spec: SR-3, SR-4, SR-6, SR-9, SR-13, SR-14, SR-33_
  - _ADR: Decision 1; Decision 7; Component Plan → QuickCreateStateMapper.kt_

- [x] 5.2 [GREEN] Add AI strings to `values/strings.xml` — staged; verify PASS. Added 6 new strings + kept `issues_ai_suggestions_header`; removed `issues_list_header` (ADR Decision 5c). `AiRowPreview.kt` updated to drop the stale reference to `R.string.issues_list_header` — preview now matches production layout (no sticky header on regular section).
  - **Do**:
    1. Keep the WIP `issues_ai_suggestions_header` addition.
    2. REMOVE `issues_list_header` (Decision 5c removes the regular-section sticky header).
    3. Add: `issues_quick_create_toolbar_ai_subtitle` ("Powered by Autodesk AI"), `issues_quick_create_label_prompt` ("Prompt"), `issues_quick_create_label_description` ("Description"), `issues_quick_create_placeholder_ai` ("Describe the issue for AI to enrich"), `issues_quick_create_placeholder_description` ("Describe the issue"), `issues_quick_create_ai_toggle_label` ("Generate with AI").
  - **Files**: `/Users/sagihatzabi/Development/Android/build-mobile/android/issues/src/main/res/values/strings.xml`
  - **Done when**: all new string resources resolve from Kotlin via generated `R.string.*`; Localization skill's conventions honored (strings are user-facing and tagged per module).
  - **Verify**: `./gradlew :android:issues:compileAltDebugKotlin`
  - **Commit**: `feat(quick-create): add AI-variant string resources and remove issues_list_header`
  - _Spec: SR-4, SR-6, SR-13, SR-14, SR-38_
  - _ADR: Component Plan → strings.xml_

- [x] 5.3 [GREEN] [P] Update `QuickCreateToolbar` for subtitle + leading icon variant — staged; verify PASS. Added `subtitleResId: Int = 0` + `leadingIconVariant: LeadingToolbarIconVariant = Close` params with defaults (source-compat). Title wrapped in Column, subtitle renders as Caption style below title. Leading-icon Row now composes sparkle + back-arrow when SparkleClose. Sparkle uses `R.drawable.ic_star_4_point` tinted `AlloyTheme.colors.tertiary_03_500`. Added AI-variant preview.
  - **Do**:
    1. Accept `isAiAvailable: Boolean` + `subtitleText: String?` params (or derive from an existing RenderData param passed from the Fragment).
    2. Replace the single-line title with `Column { AlloyText(titleText); if (subtitleText != null) AlloyText(subtitleText, typography = caption) }`.
    3. Render leading icon as sparkle + close when `isAiAvailable` is true; else close only. Keep existing Beta badge untouched.
  - **Files**: `/Users/sagihatzabi/Development/Android/build-mobile/android/issues/src/main/java/com/plangrid/android/issues/quick_create/composables/QuickCreateToolbar.kt`
  - **Done when**: preview renders subtitle only when `isAiAvailable = true`; Beta badge intact.
  - **Verify**: `./gradlew :android:issues:compileAltDebugKotlin`
  - **Commit**: `feat(quick-create): toolbar shows AI subtitle + sparkle leading icon when available`
  - _Spec: SR-4, SR-5_
  - _ADR: Decision 1; Component Plan → QuickCreateToolbar.kt_

- [x] V6 [VERIFY] Quality checkpoint — StateMapper + toolbar UI compile — compile PASS (BUILD SUCCESSFUL 8s). No ktlint task; test run deferred to V10 (still blocked on pre-existing MobiusTest failures).
  - **Do**: Run compile + unit tests + lint for `:android:issues`.
  - **Verify**: `./gradlew :android:issues:compileAltDebugKotlin :android:issues:testAltDebugUnitTest :android:issues:ktlintAltDebugSourceSetCheck`
  - **Done when**: all exit 0.

- [x] 5.4 [GREEN] Create `QuickCreateAiToggleRow` composable — staged; verify PASS. Row with Body-style text, Spacer weight(1f), and `AlloySwitch.AlloyToggleSwitch`. Padding mirrors `QuickCreateActionCard` type/placement row (horizontal SpacingM, vertical SpacingS). Semantics contentDescription = "Generate with AI, enabled/disabled". Two previews (ON/OFF).
  - **Do**:
    1. Create `QuickCreateAiToggleRow(checked: Boolean, onCheckedChange: (Boolean) -> Unit)`.
    2. Render `Row { AlloyText(stringResource(R.string.issues_quick_create_ai_toggle_label)); Spacer(Modifier.weight(1f)); AlloySwitch(checked, onCheckedChange) }`; padding matches the type/placement row in `QuickCreateActionCard`.
    3. Set `semantics { contentDescription = "Generate with AI, ${if (checked) \"enabled\" else \"disabled\"}" }` on the row.
  - **Files**: `/Users/sagihatzabi/Development/Android/build-mobile/android/issues/src/main/java/com/plangrid/android/issues/quick_create/composables/QuickCreateAiToggleRow.kt`
  - **Done when**: @Preview renders both checked/unchecked; accessibility label matches.
  - **Verify**: `./gradlew :android:issues:compileAltDebugKotlin`
  - **Commit**: `feat(quick-create): add QuickCreateAiToggleRow composable`
  - _Spec: SR-6, SR-7_
  - _ADR: Decision 1; Component Plan → CREATE QuickCreateAiToggleRow.kt_

- [x] 5.5 [GREEN] Integrate toggle row + swap label/placeholder in `QuickCreateActionCard` — staged; verify PASS. Added `isAiToggleVisible`, `aiToggleState`, `onAiToggleChanged`, `promptLabelText`, `placeholderText`, `showAiIndicator` params — all defaulted for source-compat with existing previews. Toggle row renders above description field when visible. Label/placeholder swap driven by caller (resolves from RenderData.promptLabelMode + placeholderResId in `QuickCreateScreen.kt`). STT button untouched (`showSpeechToTextButton = true` per Decision 8). `showAiIndicator` currently accepted but unused — wired to AlloyInputField in Task 6.4.
  - **Do**:
    1. Add params: `isAiToggleVisible: Boolean`, `aiToggleState: Boolean`, `onAiToggleChanged: (Boolean) -> Unit`, `promptLabelText: String`, `placeholderText: String`, `showAiIndicator: Boolean`.
    2. Render `if (isAiToggleVisible) QuickCreateAiToggleRow(aiToggleState, onAiToggleChanged)` above the description field.
    3. Pass `promptLabelText` + `placeholderText` into the existing `AlloyExpandableInputField`; pass `showAiIndicator` to the inner `AlloyInputField` (this will be wired after 6.x AlloyCompose work lands).
    4. Do NOT touch the hardcoded `showSpeechToTextButton = true` at `:200` (Decision 8).
  - **Files**: `/Users/sagihatzabi/Development/Android/build-mobile/android/issues/src/main/java/com/plangrid/android/issues/quick_create/composables/QuickCreateActionCard.kt`
  - **Done when**: toggle row renders only when `isAiToggleVisible`; label/placeholder swap live; STT button unchanged.
  - **Verify**: `./gradlew :android:issues:compileAltDebugKotlin`
  - **Commit**: `feat(quick-create): ActionCard hosts AI toggle row + swaps prompt label`
  - _Spec: SR-6, SR-13, SR-14, SR-15_
  - _ADR: Decision 1; Decision 8; Component Plan → QuickCreateActionCard.kt_

- [x] 5.6 [GREEN] Uncomment + wire the photo-counter overlay in `QuickCreateImageArea` — staged; verify PASS. Restored the commented overlay at the TopStart alignment with semi-transparent black pill + white 12sp text. Accepts `photoCounterText: String? = null` — renders only when non-null. StateMapper drives the clamp logic ("${min(photos.size, 5)}/5"). Caller in `QuickCreateScreen.kt` updated.
  - **Do**:
    1. Uncomment the counter overlay at `:102-126`.
    2. Accept a new `photoCounterText: String?` param from RenderData.
    3. Wrap the overlay in `if (photoCounterText != null)`; bind the displayed string to `photoCounterText`.
  - **Files**: `/Users/sagihatzabi/Development/Android/build-mobile/android/issues/src/main/java/com/plangrid/android/issues/quick_create/composables/QuickCreateImageArea.kt`
  - **Done when**: overlay renders only when StateMapper emits non-null `photoCounterText`; clamped "5/5" when photos > 5.
  - **Verify**: `./gradlew :android:issues:compileAltDebugKotlin`
  - **Commit**: `feat(quick-create): enable n/5 photo counter overlay for AI path`
  - _Spec: SR-9, SR-10_
  - _ADR: Decision 7; Component Plan → QuickCreateImageArea.kt_

- [x] DC-5 [DRIFT-CHECK] Spec compliance check — UI variants — see drift-log.md Turn 4 cont. for verdict (DRIFT_MINOR; `showAiIndicator` deferred to Phase 6).
  - **Scope**: Tasks since last check: [5.1, 5.2, 5.3, 5.4, 5.5, 5.6]
  - **Spec**: `pgf/feature/issues/specs/quick-create/quick-create-with-ai-spec.md`
  - **ADR**: `~/.rick/universes/ACC_issues_universe/knowledge-base/build-mobile/quick-create-with-ai/adr-android.md`
  - **Check**: Rev verifies SR-3, SR-4, SR-5, SR-6, SR-7, SR-9, SR-10, SR-13, SR-14, SR-33; confirms camera button is NOT gated (Decision 7), STT flag/locale is NOT added (Decision 8).

- [x] 5.7 [GREEN] Wire FAB leading-icon swap on `IssuesListComposable` — staged; verify PASS. Added `val isAiAvailable: Boolean by lazy { isAiSuggestionsFlagOn || isAiEnabledFlagOn }` to `IssuesViewModel`. DRIFT_MINOR: used flag-only snapshot (no `GGIssueAiEnablementRepository` server-side roundtrip) because the FAB renders synchronously. Plan allowed either approach. `IssuesListComposable.kt` `ExpandableFabSwitcher` now swaps `ic_camera → ic_star_4_point` when available.
  - **Do**:
    1. Expose `isAiAvailable` on `IssuesViewModel` (collect the same snapshot used by Quick Create, OR compute once from `IssuesFeatureFlag` + `GGIssueAiEnablementRepository` inside `IssuesViewModel` — whichever Sherlock confirms is already shared).
    2. Thread it into the `FabItem` construction at `IssuesListComposable.kt:296-317`: `leadingIcon = if (isAiAvailable) painterResource(R.drawable.ic_star_4_point) else painterResource(R.drawable.ic_camera)`.
  - **Files**: `/Users/sagihatzabi/Development/Android/build-mobile/android/issues/src/main/java/com/plangrid/android/issues/log/compose/IssuesListComposable.kt`, `/Users/sagihatzabi/Development/Android/build-mobile/android/issues/src/main/java/com/plangrid/android/issues/IssuesViewModel.kt`
  - **Done when**: FAB shows sparkle when AI available; falls back to camera when not.
  - **Verify**: `./gradlew :android:issues:compileAltDebugKotlin`
  - **Commit**: `feat(issues): swap FAB leading icon to sparkle when AI available`
  - _Spec: SR-1_
  - _ADR: Component Plan → IssuesListComposable.kt_

- [x] V7 [VERIFY] Quality checkpoint — UI variants compile + render — compile PASS (BUILD SUCCESSFUL 8s). Test suite deferred (QuickCreateMobiusTest pre-existing failures remain, none touch Phase 5 additions). No ktlint task.

---

## Phase 6 — AlloyCompose AI indicator (SR-15)

Goal: stand up the composition-local AI indicator inside `:android:alloycompose` and move the drawable to that module.

- [ ] 6.1 [GREEN] Create `LocalShowAiIndicator` composition-local
  - **Do**:
    1. Add `val LocalShowAiIndicator = compositionLocalOf { false }` in the field package.
  - **Files**: `/Users/sagihatzabi/Development/Android/build-mobile/android/alloycompose/src/main/java/com/plangrid/alloy/compose/widget/field/LocalShowAiIndicator.kt`
  - **Done when**: local resolves to `false` by default without a provider.
  - **Verify**: `./gradlew :android:alloycompose:compileAltDebugKotlin`
  - **Commit**: `feat(alloycompose): add LocalShowAiIndicator composition-local`
  - _Spec: SR-15_
  - _ADR: Component Plan → CREATE LocalShowAiIndicator.kt_

- [ ] 6.2 [GREEN] Move `ic_star_4_point.xml` from `:android:issues` to `:android:alloycompose`
  - **Do**:
    1. `git mv` the drawable from `android/issues/src/main/res/drawable/ic_star_4_point.xml` to `android/alloycompose/src/main/res/drawable/ic_star_4_point.xml`.
    2. Update every `R.drawable.ic_star_4_point` reference in `:android:issues` to the alloycompose-module R class (`com.plangrid.alloy.compose.widget.R.drawable.ic_star_4_point` or the project's conventional import).
    3. If resistance from AlloyCompose owners surfaces (Risk Register), abort the move, reset, and stop at task 6.1 — continue 6.3 via a `Painter` parameter fallback noted in the risk register.
  - **Files**: `/Users/sagihatzabi/Development/Android/build-mobile/android/alloycompose/src/main/res/drawable/ic_star_4_point.xml` (new), `/Users/sagihatzabi/Development/Android/build-mobile/android/issues/src/main/res/drawable/ic_star_4_point.xml` (deleted)
  - **Done when**: both modules compile; no dangling refs.
  - **Verify**: `./gradlew :android:alloycompose:assembleAltDebug :android:issues:assembleAltDebug`
  - **Commit**: `refactor(alloycompose): move ic_star_4_point drawable into alloycompose module`
  - _Spec: SR-15_
  - _ADR: Component Plan → MOVE ic_star_4_point.xml; Out of Scope fallback_

- [ ] 6.3 [GREEN] Create `AlloyFieldLabel` with indicator star
  - **Do**:
    1. Create `@Composable fun AlloyFieldLabel(text: String, style: TextStyle = ..., modifier: Modifier = Modifier)`.
    2. Body: `Row(verticalAlignment = CenterVertically) { AlloyText(text, style = style); if (LocalShowAiIndicator.current) { Spacer(Modifier.width(4.dp)); Icon(painterResource(R.drawable.ic_star_4_point), contentDescription = null, tint = AlloyTheme.colors.tertiary_03_500, modifier = Modifier.size(12.dp)) } }`.
  - **Files**: `/Users/sagihatzabi/Development/Android/build-mobile/android/alloycompose/src/main/java/com/plangrid/alloy/compose/widget/field/AlloyFieldLabel.kt`
  - **Done when**: preview renders with + without indicator (driven by `CompositionLocalProvider(LocalShowAiIndicator provides true/false)`).
  - **Verify**: `./gradlew :android:alloycompose:compileAltDebugKotlin`
  - **Commit**: `feat(alloycompose): add AlloyFieldLabel with AI indicator star support`
  - _Spec: SR-15_
  - _ADR: Component Plan → CREATE AlloyFieldLabel.kt_

- [ ] 6.4 [GREEN] Add `showAiIndicator` param to `AlloyInputField` and provide the local
  - **Do**:
    1. Add `showAiIndicator: Boolean = false` to the `AlloyInputField` public composable signature (default `false` keeps existing call-sites source-compat).
    2. Wrap the current body in `CompositionLocalProvider(LocalShowAiIndicator provides showAiIndicator) { ...existing body... }`.
    3. Pass through to `AlloyInputFieldFoundation` / `AlloyInputFieldContent` as needed so every label render site sits inside the provider.
  - **Files**: `/Users/sagihatzabi/Development/Android/build-mobile/android/alloycompose/src/main/java/com/plangrid/alloy/compose/widget/field/AlloyInputField.kt`, `/Users/sagihatzabi/Development/Android/build-mobile/android/alloycompose/src/main/java/com/plangrid/alloy/compose/widget/field/AlloyInputFieldFoundation.kt`
  - **Done when**: existing call-sites compile untouched; new AI-aware call-sites accept `showAiIndicator = true`.
  - **Verify**: `./gradlew :android:alloycompose:compileAltDebugKotlin`
  - **Commit**: `feat(alloycompose): AlloyInputField exposes showAiIndicator param`
  - _Spec: SR-15_
  - _ADR: Component Plan → MODIFY AlloyInputField.kt + AlloyInputFieldFoundation.kt_

- [ ] 6.5 [GREEN] Replace floating-label `AlloyTextView` with `AlloyFieldLabel` at every label site
  - **Do**:
    1. In `AlloyInputFieldFoundation.kt` and `AlloyInputFieldContent.kt`, locate every composable that renders the floating label as `AlloyTextView(text = labelText, ...)` and swap it to `AlloyFieldLabel(text = labelText, style = ...)`.
    2. Do NOT alter placement, animation, or `Modifier` chains — only swap the leaf composable.
    3. If the ADR-referenced `AlloyEditTextFoundation.kt` / `AlloyExposedField.kt` don't exist in this codebase, record that in the commit message and skip those files — label replacement is confined to the two foundation files that actually render labels.
  - **Files**: `/Users/sagihatzabi/Development/Android/build-mobile/android/alloycompose/src/main/java/com/plangrid/alloy/compose/widget/field/AlloyInputFieldFoundation.kt`, `/Users/sagihatzabi/Development/Android/build-mobile/android/alloycompose/src/main/java/com/plangrid/alloy/compose/widget/field/AlloyInputFieldContent.kt`
  - **Done when**: label renders identically when `showAiIndicator = false`; indicator appears in the floating label's end slot when true.
  - **Verify**: `./gradlew :android:alloycompose:compileAltDebugKotlin :android:alloycompose:testAltDebugUnitTest`
  - **Commit**: `refactor(alloycompose): route floating labels through AlloyFieldLabel`
  - _Spec: SR-15_
  - _ADR: Component Plan → MODIFY AlloyInputFieldFoundation.kt / AlloyInputFieldContent.kt_

- [ ] V8 [VERIFY] Quality checkpoint — AlloyCompose AI indicator
  - **Do**: Run compile + unit tests + lint for `:android:alloycompose` and `:android:issues`.
  - **Verify**: `./gradlew :android:alloycompose:compileAltDebugKotlin :android:alloycompose:testAltDebugUnitTest :android:alloycompose:ktlintAltDebugSourceSetCheck :android:issues:compileAltDebugKotlin`
  - **Done when**: all exit 0.

- [ ] DC-6 [DRIFT-CHECK] Spec compliance check — AI indicator
  - **Scope**: Tasks since last check: [5.7, 6.1, 6.2, 6.3, 6.4, 6.5]
  - **Spec**: `pgf/feature/issues/specs/quick-create/quick-create-with-ai-spec.md`
  - **ADR**: `~/.rick/universes/ACC_issues_universe/knowledge-base/build-mobile/quick-create-with-ai/adr-android.md`
  - **Check**: Rev verifies SR-1 + SR-15 end-to-end (FAB sparkle on list; star in prompt floating label when effective-AI on).

---

## Phase 7 — Analytics wiring

Goal: resolve the `QuickCreateAnalytics.kt:23-24` TODO and fire the 8 Phase-2 events with live `ai_flag_enabled` + `ai_toggle_state`.

- [ ] 7.1 [GREEN] Unhardcode `aiFlagEnabled` + `aiToggleState` in `QuickCreateAnalytics`
  - **Do**:
    1. Inject `issuesFeatureFlag: IssuesFeatureFlag` into the `QuickCreateAnalytics` constructor.
    2. Replace `private val aiFlagEnabled = false` at `:23` with `private val aiFlagEnabled: Boolean get() = issuesFeatureFlag.isAiSuggestionsFlagOn || issuesFeatureFlag.isAiEnabledFlagOn`.
    3. Remove `private val aiToggleState = false` at `:24`; add `aiToggleState: Boolean` as a parameter on every existing track method (callers pass it from Mobius state).
    4. Delete the TODO comment.
  - **Files**: `/Users/sagihatzabi/Development/Android/build-mobile/android/issues/src/main/java/com/plangrid/android/issues/quick_create/analytics/QuickCreateAnalytics.kt`
  - **Done when**: analytics class compiles; every pre-existing track method's call-sites now pass `aiToggleState`; `aiFlagEnabled` is live at event time.
  - **Verify**: `./gradlew :android:issues:compileAltDebugKotlin`
  - **Commit**: `feat(quick-create): unhardcode ai_flag_enabled + ai_toggle_state in analytics`
  - _Spec: SR-26, SR-27, SR-28, SR-29, SR-30_
  - _ADR: Decision 9; Component Plan → QuickCreateAnalytics.kt (a-c)_

- [ ] 7.2 [GREEN] Add 3 new tracking methods to `QuickCreateAnalytics`
  - **Do**:
    1. Add `fun trackToggleChanged(issueUid: String, newToggleState: Boolean)` — wraps the codegen `AnalyticsEvents.IssueQuickCreationToggleChanged(...)` constructor; **no** `ai_flag_enabled` param (spec exclusion).
    2. Add `fun trackAiSuggestionReceived(issueUid: String, imageCount: Int, responseTimeMs: Long, success: Boolean, aiToggleState: Boolean)` — wraps `AnalyticsEvents.IssueQuickCreationAiSuggestionReceived(...)`.
    3. Add `fun trackIssueDetailsOpened(issueUid: String, durationToReviewMs: Long)` — wraps `AnalyticsEvents.IssueQuickCreationIssueDetailsOpened(...)`; **no** `ai_toggle_state` param (spec exclusion).
  - **Files**: `/Users/sagihatzabi/Development/Android/build-mobile/android/issues/src/main/java/com/plangrid/android/issues/quick_create/analytics/QuickCreateAnalytics.kt`
  - **Done when**: all three methods callable from Processor; codegen constructor signatures matched.
  - **Verify**: `./gradlew :android:issues:compileAltDebugKotlin`
  - **Commit**: `feat(quick-create): add trackToggleChanged, trackAiSuggestionReceived, trackIssueDetailsOpened`
  - _Spec: SR-23, SR-24, SR-25_
  - _ADR: Decision 9; Component Plan → QuickCreateAnalytics.kt (d)_

- [ ] 7.3 [GREEN] Wire `TrackAiToggleChanged` + `TrackAiSuggestionReceived` handlers in Processor
  - **Do**:
    1. Add `handleTrackAiToggleChanged: handleEveryWithAction<Effect, Effect.TrackAiToggleChanged, Action> { quickCreateAnalytics.trackToggleChanged(it.issueUid, it.newState); Action.None }`.
    2. Add `handleTrackAiSuggestionReceived: handleEveryWithAction<Effect, Effect.TrackAiSuggestionReceived, Action> { quickCreateAnalytics.trackAiSuggestionReceived(it.issueUid, it.imageCount, it.responseTimeMs, it.success, /* aiToggleState from relay or state snapshot */); Action.None }`.
    3. Register both handlers.
    4. Update every existing track-effect handler to pass `aiToggleState` from the current State snapshot (inject a State-peek mechanism or carry `aiToggleState` on each `Track*` Effect payload — use the payload approach for idempotency).
  - **Files**: `/Users/sagihatzabi/Development/Android/build-mobile/android/issues/src/main/java/com/plangrid/android/issues/quick_create/mobius/QuickCreateProcessor.kt`
  - **Done when**: toggling the AI switch emits `toggle_changed` exactly once; enrichment success/failure emits `ai_suggestion_received` exactly once.
  - **Verify**: `./gradlew :android:issues:compileAltDebugKotlin`
  - **Commit**: `feat(quick-create): Processor fires toggle_changed + ai_suggestion_received events`
  - _Spec: SR-24, SR-25_
  - _ADR: Decision 9_

- [ ] V9 [VERIFY] Quality checkpoint — Analytics
  - **Do**: Run compile + unit tests + lint for `:android:issues`.
  - **Verify**: `./gradlew :android:issues:compileAltDebugKotlin :android:issues:testAltDebugUnitTest :android:issues:ktlintAltDebugSourceSetCheck`
  - **Done when**: all exit 0.

- [ ] 7.4 [GREEN] Wire `trackIssueDetailsOpened` at Issue Details open-path
  - **Do**:
    1. Delegate to Sherlock to confirm the issue-details entry-point (likely `IssuesNavigation` / a navigation callback invoked when the user taps a newly-created issue row).
    2. Measure `durationToReviewMs` as `System.currentTimeMillis() - creationTimestamp`, where `creationTimestamp` is carried from Quick Create (e.g., via a SavedStateHandle/extras).
    3. Call `quickCreateAnalytics.trackIssueDetailsOpened(issueUid, durationToReviewMs)`.
  - **Files**: navigation-layer site confirmed by Sherlock (likely `/Users/sagihatzabi/Development/Android/build-mobile/android/issues/src/main/java/com/plangrid/android/issues/log/IssuesNavigation.kt` — verify before editing)
  - **Done when**: opening the details screen after a Quick Create save emits exactly one `issue_details_opened` event; no event emitted when opening an older issue.
  - **Verify**: `./gradlew :android:issues:compileAltDebugKotlin`
  - **Commit**: `feat(issues): fire issue_details_opened after Quick Create completion`
  - _Spec: SR-23_
  - _ADR: Decision 9_

- [ ] DC-7 [DRIFT-CHECK] Spec compliance check — Analytics
  - **Scope**: Tasks since last check: [7.1, 7.2, 7.3, 7.4]
  - **Spec**: `pgf/feature/issues/specs/quick-create/quick-create-with-ai-spec.md`
  - **ADR**: `~/.rick/universes/ACC_issues_universe/knowledge-base/build-mobile/quick-create-with-ai/adr-android.md`
  - **Check**: Rev verifies all 8 Phase-2 events fire with the correct common-param set; `toggle_changed` excludes `ai_flag_enabled`; `issue_details_opened` excludes `ai_toggle_state`.

---

## Phase 8 — Testing (unit + preview + integration)

Goal: add test coverage for every Updater branch, every Processor handler, the StateMapper truth table, and the Issues-list combine flow.

- [ ] 8.1 [TEST][P] Unit tests for `QuickCreateUpdater` AI branches
  - **Do**:
    1. In `QuickCreateUpdaterTest`, add Kotest FreeSpec blocks for `Action.View.AiToggleChanged` (emits two effects; mutates `aiToggleState`), `Action.Processor.AiAvailabilityUpdated` (mutates `isAiAvailable`), `Action.Processor.StickyAiToggleLoaded` (mutates `aiToggleState`).
    2. Use `updater.invoke(state, action)` + `shouldContainExactly` / `shouldBe` matchers.
  - **Files**: `/Users/sagihatzabi/Development/Android/build-mobile/android/issues/src/test/java/com/plangrid/android/issues/quick_create/mobius/QuickCreateUpdaterTest.kt`
  - **Done when**: `./gradlew :android:issues:testAltDebugUnitTest --tests "*QuickCreateUpdaterTest*"` passes with 3+ new cases.
  - **Verify**: `./gradlew :android:issues:testAltDebugUnitTest --tests "*QuickCreateUpdaterTest*"`
  - **Commit**: `test(quick-create): cover AI Updater branches`
  - _Spec: SR-7, SR-8, SR-24_
  - _ADR: Decision 1; Testing section of mobius-mvi skill_

- [ ] 8.2 [TEST][P] Unit tests for `QuickCreateProcessor` AI handlers
  - **Do**:
    1. Add tests for `Effect.LoadAiAvailabilitySnapshot` (asserts exactly one `isAiEnabled()` call, emits one `AiAvailabilityUpdated`).
    2. Add tests for `Effect.PersistAiToggle` (stickyDefaults.aiToggleState written; no action emitted beyond `Action.None`).
    3. Add test for `Effect.RequestAiEnrichment` happy-path (encoded images passed; `executeSync` called; `TrackAiSuggestionReceived(success = true)` emitted).
    4. Add test for `Effect.RequestAiEnrichment` error-path (`WriteBackDescriptionOnEnrichmentFailure` + `TrackAiSuggestionReceived(success = false)` emitted).
    5. Use `getTestObserver(effect)`; mock `ggIssueAiEnablementRepository`, `issueSuggestionRepository`, `imageToBase64Converter`, `quickCreateStickyDefaults` via MockK.
  - **Files**: `/Users/sagihatzabi/Development/Android/build-mobile/android/issues/src/test/java/com/plangrid/android/issues/quick_create/mobius/QuickCreateProcessorTest.kt`
  - **Done when**: the 4 test blocks pass; mock verifications confirm Application scope used (not Mobius scope).
  - **Verify**: `./gradlew :android:issues:testAltDebugUnitTest --tests "*QuickCreateProcessorTest*"`
  - **Commit**: `test(quick-create): cover AI Processor handlers`
  - _Spec: SR-2, SR-8, SR-18, SR-19, SR-20, SR-25_
  - _ADR: Decisions 2, 6; Component Plan Processor rows_

- [ ] 8.3 [TEST][P] Unit tests for `QuickCreateStateMapper` AI truth table
  - **Do**:
    1. Add a parameterized Kotest block covering the 4 permutations of (`isAiAvailable` × `aiToggleState`) → expected `(isAiEffectivelyOn, promptLabelMode, placeholderResId, isAiToggleVisible, photoCounterText, toolbarSubtitleResId, leadingToolbarIconVariant)`.
    2. Cover `photos.size` ∈ {0, 3, 5, 7} to validate clamp behavior.
  - **Files**: `/Users/sagihatzabi/Development/Android/build-mobile/android/issues/src/test/java/com/plangrid/android/issues/quick_create/mobius/QuickCreateStateMapperTest.kt`
  - **Done when**: test suite passes; 4+4 = 8+ discrete cases covered.
  - **Verify**: `./gradlew :android:issues:testAltDebugUnitTest --tests "*QuickCreateStateMapperTest*"`
  - **Commit**: `test(quick-create): cover StateMapper AI truth table + photo clamp`
  - _Spec: SR-3, SR-4, SR-9, SR-13, SR-14, SR-33_
  - _ADR: Decision 1; Decision 7_

- [ ] V10 [VERIFY] Quality checkpoint — Mobius unit tests
  - **Do**: Run the full `:android:issues` unit-test suite.
  - **Verify**: `./gradlew :android:issues:testAltDebugUnitTest`
  - **Done when**: zero failures.

- [ ] 8.4 [TEST] Turbine test for `IssuesViewModel.combine(issuesLightList, statusMap)`
  - **Do**:
    1. In `IssuesViewModelTest`, emit a `List<IssueLightListModel>` and a `Map<GGIssueUid, IssueSuggestion>` independently; assert the combined emission carries both.
    2. Emit a status-map update alone; assert new `IssueViewState.Data` is emitted with updated `aiStatus`.
  - **Files**: `/Users/sagihatzabi/Development/Android/build-mobile/android/issues/src/test/java/com/plangrid/android/issues/IssuesViewModelTest.kt`
  - **Done when**: Turbine's `awaitItem()` shows both updates.
  - **Verify**: `./gradlew :android:issues:testAltDebugUnitTest --tests "*IssuesViewModelTest*"`
  - **Commit**: `test(issues): verify combine of issuesLightList + statusMap into IssueViewState.Data`
  - _Spec: SR-22, SR-38_
  - _ADR: Decision 3_

- [ ] 8.5 [TEST][P] Compose preview drift guard — `AiRowPreview`
  - **Do**:
    1. Verify `AiRowPreview.kt` still compiles with updated `AiIssueItem` palette.
    2. If Paparazzi is later added (risk register), record the preview golden; for now, preview must render in Android Studio without exception.
  - **Files**: `/Users/sagihatzabi/Development/Android/build-mobile/android/issues/src/main/java/com/plangrid/android/issues/log/compose/ai/AiRowPreview.kt`
  - **Done when**: preview compiles; Android Studio preview panel shows all three rows.
  - **Verify**: `./gradlew :android:issues:compileAltDebugKotlin`
  - **Commit**: (none — preview-only)
  - _Spec: SR-22_
  - _ADR: Decision 5a, 5b; Risk Register "Drift between IssueItem and AiIssueItem"_

- [ ] DC-8 [DRIFT-CHECK] Spec compliance check — Testing
  - **Scope**: Tasks since last check: [8.1, 8.2, 8.3, 8.4, 8.5]
  - **Spec**: `pgf/feature/issues/specs/quick-create/quick-create-with-ai-spec.md`
  - **ADR**: `~/.rick/universes/ACC_issues_universe/knowledge-base/build-mobile/quick-create-with-ai/adr-android.md`
  - **Check**: Rev samples three Updater branches, two Processor handlers, and the StateMapper truth table to confirm tests actually assert spec behavior (not just "doesn't crash").

---

## Phase 9 — Final quality gate

Goal: full module build + lint; run a manual smoke matrix across the 4 availability × toggle permutations on the Resizable_2 AVD.

- [ ] 9.1 [VERIFY] Full module build + lint
  - **Do**: Run assembleAltDebug + unit test + ktlint for all touched modules.
  - **Verify**: `./gradlew :android:issues:assembleAltDebug :android:alloycompose:assembleAltDebug :android:components:assembleAltDebug :android:issues:testAltDebugUnitTest :android:alloycompose:testAltDebugUnitTest :android:issues:ktlintAltDebugSourceSetCheck :android:alloycompose:ktlintAltDebugSourceSetCheck :android:components:ktlintAltDebugSourceSetCheck`
  - **Done when**: all exit 0.

- [ ] 9.2 [VERIFY] Manual smoke — 4 permutations
  - **Do**:
    1. On Resizable_2 AVD, install `:android:app:installAltDebug`.
    2. Test each of the 4 availability × toggle permutations per ADR Verification Checkpoints CP3 + CP4 + CP6 + CP7.
    3. Record results: (AI available × toggle ON) → sparkle FAB, subtitle, toggle ON, prompt label, n/5 overlay, enrichment fires; (available × OFF) → subtitle still, toggle OFF, description label, no overlay, no enrichment; (unavailable × *) → Phase 1 behavior only.
  - **Verify**: manual — matrix captured in PR description.
  - **Done when**: matrix green; no blocking alerts observed on enrichment failure (induced by turning airplane mode on after tapping Next).
  - **Commit**: (none — manual)

- [ ] DC-9 [DRIFT-CHECK] Final spec compliance sweep
  - **Scope**: Full Requirement Mapping table (SR-1 through SR-39).
  - **Spec**: `pgf/feature/issues/specs/quick-create/quick-create-with-ai-spec.md`
  - **ADR**: `~/.rick/universes/ACC_issues_universe/knowledge-base/build-mobile/quick-create-with-ai/adr-android.md`
  - **Check**: Rev completes the final sweep — confirms every SR row marked "MODIFY/CREATE" in the Component Plan has a matching commit, STT deviation (Decision 8) is correctly parked, camera-button override (Decision 7) correctly not implemented, and all 7 Verification Checkpoints pass.

---

## Summary

| Metric | Value |
|---|---|
| Total tasks | 48 |
| Phase 1 (compile gap + state scaffolding) | 3 implementation + 1 VERIFY + 1 DRIFT-CHECK |
| Phase 2 (issues list AI integration) | 4 implementation + 1 VERIFY + 1 DRIFT-CHECK |
| Phase 3 (flag accessors + availability snapshot) | 3 implementation + 1 VERIFY + 1 DRIFT-CHECK |
| Phase 4 (Mobius AI wiring) | 7 implementation + 2 VERIFY + 1 DRIFT-CHECK |
| Phase 5 (UI variants) | 7 implementation + 2 VERIFY + 1 DRIFT-CHECK |
| Phase 6 (AlloyCompose AI indicator) | 5 implementation + 1 VERIFY + 1 DRIFT-CHECK |
| Phase 7 (analytics) | 4 implementation + 1 VERIFY + 1 DRIFT-CHECK |
| Phase 8 (testing) | 5 implementation + 1 VERIFY + 1 DRIFT-CHECK |
| Phase 9 (final quality gate) | 2 VERIFY + 1 DRIFT-CHECK |
| [VERIFY] checkpoints | 12 |
| [DRIFT-CHECK] gates | 9 (every-3 cadence) |
| Estimated total time at 10 min/task | ~480 min (~8 h of focused work) |

Spec requirement coverage: SR-1, SR-2, SR-3, SR-4, SR-5, SR-6, SR-7, SR-8, SR-9, SR-10, SR-CAP, SR-12 (no-delta), SR-13, SR-14, SR-15, SR-16 (no-delta), SR-17 (parked — Decision 8), SR-18, SR-19, SR-20, SR-21, SR-22, SR-23, SR-24, SR-25, SR-26, SR-27, SR-28, SR-29, SR-30, SR-31, SR-32 (no-delta), SR-33, SR-34, SR-35 (no-delta), SR-36 (no-delta), SR-37 (no-delta), SR-38, SR-39.

Spec deviations tracked:
- **Decision 7** — camera button NOT gated; payload-only 5-image cap. Spec was updated to match (per approval notes).
- **Decision 8** — STT flag/locale gating PARKED; SR-17 intentionally not implemented this phase.
