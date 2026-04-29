# Quick Create with AI — Research & POC Suggestions for Android

> **Purpose**: Identify areas where the spec is ambiguous or where Android lacks infrastructure, so the ADR can be written with confidence and the implementation avoids surprises.
> **Date**: 2026-04-16
> **Source**: Cross-referencing the quick-create-with-ai spec against iOS implementation (shipped) and Android implementation (non-AI only).

---

## How to Read This Document

Each area is rated by **risk level**:
- **HIGH** — No existing Android code. Requires POC to validate approach before ADR.
- **MEDIUM** — Partial infrastructure exists. Needs research to confirm approach, may need small POC.
- **LOW** — Infrastructure exists. Just needs mapping from spec to existing patterns.

---

## Area 1: Issue List — AI Suggestions Section & Row States

**Risk: HIGH — Needs POC**

### What the Spec Says
- Issue list shows enrichment states: **in progress**, **ready to review**, **failed**
- Each row shows a status indicator (spinner/badge/error icon)
- iOS creates a **separate "Generate with AI" section** above regular issues

### What iOS Does
- `IssuesListFeature.swift` splits issues into two arrays: AI issues (any issue with an `IssueSuggestion` entry in `suggestionsStatusMap`) and regular issues
- A separate section with header "Generate with AI" is rendered above the regular list
- Each row shows an `AISuggestionsPin`: dark circle with sparkle icon, animated spinning gradient ring for `.processing`, warning icon for `.error`
- Photo grid view has the same pin logic

### What Android Has Today
- `IssueItem.kt` renders a single flat list — **no section concept** at all
- Each row has a status circle (colored by issue status) with a type pin code
- `IssueLightListModel` has no field for AI suggestion state
- The `LazyIssueList` in `IssuesListComposable.kt` iterates a single flat data list — no grouped/sectioned layout

### What Needs Research & POC

1. **Sectioned LazyColumn architecture**: Android's issue list is a flat `LazyColumn`. iOS uses a sectioned approach with a `.ai` section type. We need a POC for:
   - How to split the data into AI section + regular section without breaking existing scroll state, pagination, and sticky headers
   - Whether to use `LazyColumn` with `stickyHeader {}` items or a custom section composable
   - How this interacts with the existing `FilterChipRow` and `ExpandableFabSwitcher` overlays

2. **AI status data flow**: The issue list currently reads from `IssueLightListModel` which comes from the PGF repository. We need to research:
   - Does the PGF `issueSuggestionRepository` already expose a status map observable (like iOS's `statusMap()`)?
   - What's the Kotlin equivalent of iOS's `suggestionsDataProvider.statusMap()` → `[GGIssueUid: IssueSuggestion]`?
   - How does the status get into the Mobius model for the issues list? (new Effect + Event pair, or a secondary stream merged into `renderDataStream`?)

3. **AI status pin composable**: Need a POC for the visual indicator:
   - Dark circle with AlloyCompose sparkle icon
   - Animated spinning gradient ring for "processing" state (iOS uses `AngularGradient` rotation animation)
   - How to overlay this on the existing status circle without breaking the type pin code display
   - Compose animation for the spinning ring: `InfiniteTransition` + `drawBehind` with sweep gradient?

**Suggested POC deliverable**: A standalone composable `AiSuggestionStatusIndicator(status: AiSuggestionStatus)` with three visual states, tested in a preview alongside the existing `IssueItem`.

---

## Area 2: AI Availability Composite Check

**Risk: MEDIUM — Needs Research**

### What the Spec Says
```
AI available = AI suggestions flag is ON
               OR (AI enabled flag is ON AND server AI enablement is true)
```

### What iOS Does
- `IssuesFlags.swift:57-58`: `isQuickCreateAiSuggestionsEnabled || (isQuickCreateAiEnabled && aiEnablementProvider.isAiEnabled())`
- `AiEnablementProvider.swift`: calls into PGF's `ggIssueAiEnablementRepository.isAiEnabled()`

### What Android Has Today
- PGF flag enum entries exist: `ISSUES_QUICK_CREATE_AI_SUGGESTIONS` and `ISSUES_QUICK_CREATE_AI_ENABLED`
- But `IssuesFeatureFlag.kt` on Android does **NOT consume them** — only `issuesQuickCreate` and `isSpeechToTextEnabled` are wired
- No `AiEnablementProvider` equivalent on Android
- No consumption of `ggIssueAiEnablementRepository` on Android side

### What Needs Research

1. **PGF AI enablement repository**: Confirm `ggIssueAiEnablementRepository.isAiEnabled()` exists in the PGF Kotlin code and is accessible from Android. iOS calls it via KMP — does Android already have the dependency wired?

2. **Flag consumption**: Adding the two flag checks to `IssuesFeatureFlag.kt` is straightforward, but the composite logic (`flagA || (flagB && serverCheck)`) needs to be a single computed property. Research where this should live: `IssuesFeatureFlag`? A new `AiAvailabilityProvider`? Inside the Mobius State?

3. **Reactivity**: iOS's `isAiSuggestionsAvailable` is re-evaluated on every read (it's a computed var on the State). On Android with Mobius, the availability check runs at init time. If AI availability changes mid-session (server disables it), does the Android Mobius loop pick it up? Or does it need a separate reactive stream?

---

## Area 3: AI Enrichment Request Pipeline

**Risk: HIGH — Needs Research + POC**

### What the Spec Says
- On save with AI on: send enrichment request with issue ID, prompt text, type/subtype UIDs, and up to 5 images
- Server endpoint: project-scoped issue suggestions endpoint
- On failure: save prompt to description field

### What iOS Does
- `IssueSuggestionsDataDependency.swift`: Converts images to base64 JPEG (0.9 quality), builds `ContextObject(generalDescription, images)`, calls `issueSuggestionRepository.executeSync()`
- The repository is PGF Kotlin code called via KMP

### What Android Has Today
- **Nothing**. No enrichment request code exists. No API client, no image conversion, no suggestion repository usage.

### What Needs Research & POC

1. **PGF `issueSuggestionRepository`**: Confirm this KMP repository exists and has an `executeSync()` method callable from Android. What are the exact parameter types? Does the Kotlin API match what iOS calls via its Swift bridge?

2. **Image preparation pipeline**: iOS converts `UIImage` to base64 JPEG. Android needs:
   - Photo URIs → Bitmap → compressed JPEG → base64 string
   - This may be expensive for 5 images. Research: should this run on `Dispatchers.IO`? Should images be resized before base64 encoding? What's the max payload size the server accepts?
   - POC: Write a utility that takes 5 photo URIs and produces the `ImageWrapper` objects the PGF repository expects. Measure timing and memory usage.

3. **Fire-and-forget vs. tracked**: iOS fires the enrichment from the QuickCreate reducer and catches errors silently. On Android with Mobius, this would be a Processor effect. But the enrichment runs AFTER the user has left the QuickCreate screen (they may already be on the next issue). Research:
   - Does the Mobius processor survive screen navigation? (Probably not — the ViewModel gets cleared)
   - Should this be a WorkManager job? A coroutine in a broader scope (Application/Service)?
   - iOS uses a `SuggestionsRepositoryActor` — is there an Android equivalent pattern?

4. **Prompt-to-description fallback**: On failure, the prompt needs to be saved to the issue description. But the issue was already saved. Research: does the PGF repository have an `updateIssueDescription()` API? Or does Android need to call the sync API to patch the description field?

**Suggested POC deliverable**: Minimal end-to-end test — hard-code an issue ID + 3 images + a prompt, call the PGF suggestion repository, verify the server receives the request and returns a response.

---

## Area 4: Entry Point Sparkle Icon

**Risk: LOW — Mapping Exercise**

### What the Spec Says
- FAB shows sparkle icon when AI available, standard bolt icon otherwise

### What iOS Does
- `QuickCreateButton.swift`: `.aiSparkle` vs `.bolt` based on `isAiSuggestionsAvailable`

### What Android Has Today
- `IssuesListComposable.kt` renders `ExpandableFabSwitcher` with a camera icon for Quick Create
- FAB visibility is controlled by `QuickCreateFabVisibilityHelper`

### What Needs Research

1. **AlloyCompose sparkle icon**: Does `AlloyIcon` have an AI sparkle variant? Check AlloyCompose icon set. If not, a new icon drawable is needed (like the `ic_star_4_point.xml` from the modified indicator POC).

2. **Conditional icon swap**: Simple — just add the `isAiAvailable` boolean to the composable params. Low effort. But confirm the AlloyCompose FAB component supports icon swapping without visual jank.

---

## Area 5: Generate with AI Toggle + Sticky Preference

**Risk: MEDIUM — Needs Research**

### What the Spec Says
- Toggle visible only when AI available
- Default: on when unset
- Persisted across sessions (global, not per-project)
- If availability drops mid-session, AI processing turns off even if persisted toggle is on

### What iOS Does
- `PlanGridUserDefaults.quickCreateIssueAIToggleState` — simple boolean in UserDefaults
- `isAIEnabled` computed: `aiToggelState && issuesFlags.isQuickCreateAiSuggestionsAvailable`

### What Android Has Today
- `QuickCreateStickyDefaults.kt` stores type and placement in SharedPreferences
- `usingAi` field exists in `RenderData` but is hardcoded `false`

### What Needs Research

1. **Toggle in Mobius state**: The `usingAi` field in `RenderData` is already there but unused. Research: should the toggle state be in `QuickCreateMobius.State` (Mobius owns it) or in `QuickCreateStickyDefaults` (persisted independently)? iOS keeps it in TCA State AND UserDefaults. Mobius convention would be State + StickyDefaults together.

2. **Composable for the toggle row**: The spec shows a "Generate with AI" row between type/placement and the prompt field. Research: use `AlloySwitch` from AlloyCompose? Row layout: icon + label + toggle. Visibility animated or just conditional?

3. **Mid-session availability change**: In Mobius, the composite check happens once at init. If the server disables AI mid-session, the model won't know. Research: is there a reactive observable for AI enablement status from PGF that can push updates into the Mobius loop?

---

## Area 6: Prompt vs Description Field Switching

**Risk: LOW — UI-only change**

### What the Spec Says
- When AI effectively on: label = "Describe the issue", placeholder = AI prompt copy
- When AI off: label = "Description", placeholder = standard copy
- 500 character limit shared with STT

### What iOS Does
- Dynamic string swap based on `store.isAIEnabled` — different localization keys for label and placeholder (phone vs tablet variants for AI placeholder)

### What Android Has Today
- `QuickCreateActionCard.kt` already uses `AlloyExpandableInputField` with 500-char limit
- String resource `quick_create_assistive_text` already exists for the AI placeholder copy
- STT is already integrated into the description field

### What Needs Research

1. **Label switching**: Just swap the label string based on `usingAi`. Need to confirm: does `AlloyExpandableInputField` support dynamic label changes without losing focus or text? Quick manual test in preview.

2. **AlloyInputField AI indicator**: This is where the `alloy-input-field-ai-indicator.md` KB entry comes in. When AI is on and field has AI-suggested content, show the purple star. See Approach 3 (CompositionLocal) doc.

---

## Area 7: Image Counter Overlay "n/5"

**Risk: LOW — UI exists but is commented out**

### What the Spec Says
- When AI effectively on: show "n/5" overlay on image area

### What iOS Does
- Text overlay: `"%{count} of %{maxCount} max"` — shown only when `isAIEnabled`

### What Android Has Today
- `QuickCreateImageArea.kt` has a **commented-out photo counter** (lines 102-126) — the UI is already built but disabled
- `QuickCreateCarousel.kt` has the "Autodesk AI analyzes the first five photos" text gated by `usingAi` — already wired

### What Needs Research

1. **Uncomment and adapt**: The photo counter overlay exists. Research: does the commented-out version match the spec's "n/5" format? Or does it need the "of max" format like iOS?

2. **Image limit enforcement**: The spec says "first 5 only." Research: does the carousel need to visually indicate which photos will be sent to AI vs. ignored? iOS shows a purple dot on the first 5 thumbnails. Should Android do the same?

---

## Area 8: Toolbar "Powered by Autodesk AI" Subtitle

**Risk: LOW — UI change only**

### What the Spec Says
- Subtitle "Powered by Autodesk AI" shown when AI is available (not just when toggle is on)

### What iOS Does
- `QuickCreateIssue+MainView.swift`: conditional `Text` in the toolbar VStack, gated by `isAiSuggestionsAvailable`

### What Android Has Today
- `QuickCreateToolbar.kt` renders title "Quick Create Issue" with a close button
- No subtitle support

### What Needs Research

1. **AlloyCompose TopAppBar subtitle**: Does AlloyCompose's toolbar component support a subtitle? Or do we need a custom `Column { Title; Subtitle }` in the `title` slot? Check the AlloyCompose API.

2. **Beta badge**: The spec mentions an optional Beta badge. Research: does AlloyCompose have a badge component? Or is this a custom `AlloyChip` / custom composable?

---

## Area 9: Speech-to-Text with AI Prompt

**Risk: LOW — Already shipped on Android**

### What the Spec Says
- STT available when flag is on AND device locale supported
- Streaming transcription, shared 500-char cap

### What Android Has Today
- STT is already integrated into the description field (`showSpeechToTextButton = true` in `QuickCreateActionCard.kt`)
- `IssuesFeatureFlag.isSpeechToTextEnabled` is already wired

### What Needs Research

1. **STT during AI mode**: When AI is on, the text goes to the "prompt" field (which maps to the AI enrichment payload). When AI is off, it goes to "description" (saved to issue). Research: is the existing STT integration agnostic to where the text ends up? (It should be — it just writes to the same text field.)

---

## Area 10: Failure Handling — Prompt Saved to Description

**Risk: MEDIUM — Needs Research**

### What the Spec Says
- On enrichment failure: save prompt to description field
- No blocking toast
- Issue list shows "failed" state

### What iOS Does
- `QuickCreateIssue+MainStoreHelpers.swift`: When AI is on, description is NOT saved during normal flow (it's sent to AI instead). On failure/discard, `saveUserDescription(ignoreAIState: true)` forces saving.

### What Android Has Today
- `QuickCreateProcessor.kt` `finalizeIssue()` always saves description to the issue. No conditional logic for AI.

### What Needs Research

1. **Conditional save logic**: When AI is on, the "description" text is actually a prompt for AI — it should NOT be saved to the issue description field during normal save (AI will generate the description). Only on AI failure should it fall back. Research: how to modify the Mobius `finalizeIssue` effect to branch based on AI state.

2. **Post-navigation description update**: If enrichment fails AFTER the user has left QuickCreate (they're on the next issue or the list), the prompt needs to be saved to the issue. Research: what mechanism updates an issue's description from outside QuickCreate? PGF repository `updateIssue()`? Direct database update?

---

## Priority Matrix

| Area | Risk | POC Needed? | Blocks ADR? |
|------|------|-------------|-------------|
| 1. Issue List AI Section & Row States | HIGH | Yes — sectioned list + animated status pin | Yes |
| 2. AI Availability Composite Check | MEDIUM | No — research only | Yes |
| 3. AI Enrichment Request Pipeline | HIGH | Yes — end-to-end API call test | Yes |
| 4. Entry Point Sparkle Icon | LOW | No | No |
| 5. AI Toggle + Sticky Preference | MEDIUM | No — research only | Partially |
| 6. Prompt vs Description Switching | LOW | No | No |
| 7. Image Counter Overlay | LOW | No | No |
| 8. Toolbar Subtitle | LOW | No | No |
| 9. Speech-to-Text | LOW | No | No |
| 10. Failure Handling | MEDIUM | No — research only | Partially |

## Recommended Order

1. **Area 3** (Enrichment Pipeline) — If PGF repository doesn't work from Android, everything else is moot
2. **Area 2** (AI Availability) — Need to confirm the composite check infrastructure exists
3. **Area 1** (Issue List) — Most complex UI work, biggest unknown
4. **Area 10** (Failure Handling) — Affects save flow architecture
5. **Area 5** (Toggle) — Affects Mobius state design
6. Everything else is low-risk UI mapping

---

## iOS Reference Files for Android Developers

When implementing each area, reference these iOS files for the "known good" implementation:

| Area | iOS Reference | What to Look At |
|------|--------------|-----------------|
| AI Availability | `IssuesFlags.swift:57-58` | Composite boolean formula |
| AI Availability | `AiEnablementProvider.swift:30-35` | Server check via PGF |
| Toggle | `QuickCreateIssue.swift:46-49, 105, 324-335` | Computed property, init, persistence |
| Enrichment | `IssueSuggestionsDataDependency.swift:64-131` | Payload construction, image handling |
| Issue List | `IssuesListFeature.swift:170-205` | Section splitting logic |
| Issue List | `ListSingleIssueView.swift:132-139, 187-242` | Pin view and animation |
| Sparkle Icon | `QuickCreateButton.swift:153-155` | Conditional icon |
| Prompt/Desc | `QuickCreateIssue+MainView.swift:408-413` | Dynamic string swap |
| Toolbar | `QuickCreateIssue+MainView.swift:100-110` | Conditional subtitle |
| Counter | `QuickCreateIssue+MainView.swift:243-248, 266-271` | n/5 overlay |
| Failure | `QuickCreateIssue+MainStoreHelpers.swift:12-29` | Conditional description save |
| STT | `SpeechToTextFeature.swift`, `SpeechClient.swift` | Full STT architecture |
