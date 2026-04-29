# AlloyUI Compliance Audit: quick-create-with-ai — android

> **Auditor**: Alloy (AlloyUI Auditor)
> **Scope**: Phase 6 sub-step 2 (post-Rev SPEC_FORGE_PASS)
> **Branch**: `shaggi/quick-create-with-ai-android`
> **Date**: 2026-04-19

## Files Audited

**UI composables (issues module):**
- `/Users/sagihatzabi/Development/Android/build-mobile/android/issues/src/main/java/com/plangrid/android/issues/quick_create/composables/QuickCreateAiToggleRow.kt` (new)
- `/Users/sagihatzabi/Development/Android/build-mobile/android/issues/src/main/java/com/plangrid/android/issues/quick_create/composables/QuickCreateToolbar.kt` (modified)
- `/Users/sagihatzabi/Development/Android/build-mobile/android/issues/src/main/java/com/plangrid/android/issues/quick_create/composables/QuickCreateActionCard.kt` (modified)
- `/Users/sagihatzabi/Development/Android/build-mobile/android/issues/src/main/java/com/plangrid/android/issues/quick_create/composables/QuickCreateImageArea.kt` (modified)
- `/Users/sagihatzabi/Development/Android/build-mobile/android/issues/src/main/java/com/plangrid/android/issues/quick_create/composables/QuickCreateScreen.kt` (modified)
- `/Users/sagihatzabi/Development/Android/build-mobile/android/issues/src/main/java/com/plangrid/android/issues/log/compose/IssuesListComposable.kt` (modified)
- `/Users/sagihatzabi/Development/Android/build-mobile/android/issues/src/main/java/com/plangrid/android/issues/log/compose/IssueItem.kt` (modified)
- `/Users/sagihatzabi/Development/Android/build-mobile/android/issues/src/main/java/com/plangrid/android/issues/log/compose/ai/AiIssueItem.kt` (new)
- `/Users/sagihatzabi/Development/Android/build-mobile/android/issues/src/main/java/com/plangrid/android/issues/log/compose/ai/IssueRowBody.kt` (new)
- `/Users/sagihatzabi/Development/Android/build-mobile/android/issues/src/main/java/com/plangrid/android/issues/log/compose/ai/IssuesListSections.kt` (new)
- `/Users/sagihatzabi/Development/Android/build-mobile/android/issues/src/main/java/com/plangrid/android/issues/log/compose/ai/IssuesSectionHeader.kt` (new)
- `/Users/sagihatzabi/Development/Android/build-mobile/android/issues/src/main/java/com/plangrid/android/issues/log/compose/ai/AiRowPreview.kt` (new)

**AlloyCompose widgets (design-system module):**
- `/Users/sagihatzabi/Development/Android/build-mobile/android/alloycompose/src/main/java/com/plangrid/alloy/compose/widget/field/LocalShowAiIndicator.kt` (new)
- `/Users/sagihatzabi/Development/Android/build-mobile/android/alloycompose/src/main/java/com/plangrid/alloy/compose/widget/field/AlloyFieldLabel.kt` (new)
- `/Users/sagihatzabi/Development/Android/build-mobile/android/alloycompose/src/main/java/com/plangrid/alloy/compose/widget/field/AlloyInputField.kt` (modified — added `showAiIndicator` param)
- `/Users/sagihatzabi/Development/Android/build-mobile/android/alloycompose/src/main/java/com/plangrid/alloy/compose/widget/field/AlloyInputFieldContent.kt` (modified — label replaced with `AlloyFieldLabel`)
- `/Users/sagihatzabi/Development/Android/build-mobile/android/alloycompose/src/main/java/com/plangrid/alloy/compose/widget/field/AlloyInputFieldFoundation.kt` (modified — CompositionLocalProvider)

Total: **16 files** audited (12 modified/new issues-module composables + 4 alloycompose widgets).

---

## Violations

| File:line | Issue | Severity | Suggested Fix |
|-----------|-------|----------|---------------|
| `quick_create/composables/QuickCreateToolbar.kt:48` | Uses legacy Material `TopAppBar` (wrapper around Material2) rather than an Alloy equivalent. Mixed Material (`androidx.compose.material.IconButton`, `androidx.compose.material.TopAppBar`) and Material3 (`androidx.compose.material3.Icon`, `IconButton`) imports. | SUGGESTION | No AlloyUI toolbar widget exists today — this matches the existing `QuickCreate` pattern and is ACCEPTABLE as an AlloyUI gap. But pick ONE Material version: the file imports both `material.IconButton` (line 14) and `material3.IconButton` (line 79) — unify. |
| `quick_create/composables/QuickCreateToolbar.kt:85-96` | Uses raw `androidx.compose.material3.Icon` for the sparkle and back chevron instead of `AlloyImage.AlloyIconView`. | SUGGESTION | Replace with `AlloyImage.AlloyIconView(painter = painterResource(...), iconColors = AlloyTint.Purple)` (or a tint mapped to `tertiary_03_500`) to match the `actions` branch (line 106) which uses `AlloyImage.AlloyIconView` correctly. |
| `quick_create/composables/QuickCreateToolbar.kt:87,95` | `contentDescription = null` / `""` on the sparkle and back arrow inside a tappable `IconButton`. The back IconButton has `testTag` but zero a11y label — TalkBack will read nothing meaningful. | BLOCKER | Back icon: `contentDescription = stringResource(R.string.accessibility_back)`. Sparkle: leave `null` (decorative, grouped with text subtitle) — OK. |
| `quick_create/composables/QuickCreateToolbar.kt:88,91` | Hardcoded `16.dp` (sparkle size) and `4.dp` (spacer) instead of `Dimens.SpacingM`/`Dimens.SpacingXXS`. | NIT | `Modifier.size(Dimens.SpacingM)` and `Spacer(Modifier.width(Dimens.SpacingXXS))`. |
| `quick_create/composables/QuickCreateActionCard.kt:49` | `private val FIELD_HEIGHT = 80.dp` — hardcoded field height. | NIT | Acceptable: no AlloyUI token matches "80dp field height". Leave as-is; document as component-local constant (it already is `private val`). |
| `quick_create/composables/QuickCreateActionCard.kt:108` | `Arrangement.spacedBy(-Dimens.SpacingM)` — negative spacing is a layout hack for overlapping borders of the two picker fields. | SUGGESTION | Pre-existing pattern (not introduced by this feature), but flagged for future refactor: AlloyCompose fields should expose a "stitched-pair" variant. No action required. |
| `quick_create/composables/QuickCreateImageArea.kt:78` | `.background(Color.Black)` — hardcoded raw `Color.Black` instead of an Alloy token. | SUGGESTION | Use `AlloyColorPallete.Charcoal900` (or a dedicated media-backdrop token if one exists). Acceptable as AlloyUI gap for "photo viewer backdrop". |
| `quick_create/composables/QuickCreateImageArea.kt:120,128` | Photo-counter overlay uses hardcoded `Color.Black.copy(alpha = 0.5f)` and `Color.White`, plus inline `12.sp` font size inside `TextStyle(...)`. | BLOCKER | (1) Colors: `AlloyColorPallete.Charcoal900.copy(alpha = 0.5f)` + `AlloyColorPallete.White`. (2) Font size: swap the raw `TextStyle` for `textStyle = AlloyTextStyle.Caption` (already 12.sp) and apply the white color via `textColor = AlloyStateColorsDefault.colors(Color.White, Color.White)` or an Alloy "OnDark" token. Avoid constructing raw `TextStyle` objects inside screens. |
| `quick_create/composables/QuickCreateImageArea.kt:89` | `AsyncImage(..., contentDescription = null)` — decorative is fine, but adding a count-aware a11y label on the parent Box would help. | NIT | Optional: add `.semantics { contentDescription = "Photo ${selectedPhotoIndex + 1} of ${photos.size}" }` on the outer Box when `photoCounterText != null`. |
| `quick_create/composables/QuickCreateScreen.kt:85` | `.background(Color.Black)` on the content `BoxWithConstraints`. | NIT | Same as QuickCreateImageArea:78 — pre-existing raw `Color.Black`. Consider `AlloyColorPallete.Charcoal900`. |
| `quick_create/composables/QuickCreateAiToggleRow.kt:39` | `stateDescription = if (checked) "enabled" else "disabled"` — **hardcoded English**, not localized. | BLOCKER | Extract to `R.string.issues_quick_create_ai_toggle_state_enabled` / `..._disabled` and use `stringResource(...)`. TalkBack announces in device locale. |
| `quick_create/composables/QuickCreateAiToggleRow.kt:44` | `contentDescription = "$label, $stateDescription"` uses a hardcoded `", "` separator and a manually-composed string. Compose has a native `toggleableState` semantics that TalkBack already verbalizes ("switch, on/off"). | SUGGESTION | Prefer `Modifier.semantics(mergeDescendants = true) { role = Role.Switch; contentDescription = label }` on the Row and let `AlloySwitch` carry its own state semantics. Or use `Modifier.toggleable(value = checked, ...)`. The manual string is redundant with the switch's built-in announcement. |
| `log/compose/ai/AiIssueItem.kt:18` | Imports `androidx.compose.material.Icon` (Material2) instead of `AlloyImage.AlloyIconView`. Used on line 92 for the error-warning glyph and line 128 for the white star. | BLOCKER | Replace both with `AlloyImage.AlloyIconView(painter = ..., iconColors = AlloyStateColorsDefault.colors(Color.White, Color.White))` (for the star on the gradient circle) and `AlloyImage.AlloyIconView(painter = ..., iconColors = AlloyTint.Gray)` (for the warning). The whole point of AlloyUI is that raw `Icon` never appears in feature modules. |
| `log/compose/ai/AiIssueItem.kt:95` | `tint = AlloyColorPallete.Charcoal900` on the warning icon — uses a palette constant directly. The warning glyph should use `AlloyTheme.colors.error` or a semantic warning token for dark-mode correctness. | SUGGESTION | `tint = AlloyTheme.colors.error` (or `AlloyTint.Alert` if available). `Charcoal900` will be wrong in dark mode. |
| `log/compose/ai/AiIssueItem.kt:110,116,132,157` | Hardcoded `.size(STATUS_CIRCLE_SIZE.dp)` is fine (matches `IssueItem.STATUS_CIRCLE_SIZE = 48` constant), but the **20.dp** star size (line 132) and **24.dp** (`ERROR_OVERLAY_SIZE_DP`, line 47) are raw `.dp` literals. | NIT | 20.dp is `Dimens.SpacingL` (if defined) — or leave. 24.dp is a standard icon size; consider a shared `AlloyIconSize.Medium` constant. |
| `log/compose/ai/AiIssueItem.kt:131` | `tint = Color.White` — hardcoded raw color instead of a token. On the blue→purple gradient the star must stay white, so this is intentional, but prefer an Alloy token. | NIT | `tint = AlloyColorPallete.White` if it exists, else `Color.White` is acceptable for contrast on fixed-brand-gradient surfaces. |
| `log/compose/ai/AiIssueItem.kt:92` | `contentDescription = null` on the error warning Icon. Users using TalkBack have no indication that a suggestion errored. | BLOCKER | `contentDescription = stringResource(R.string.issues_ai_suggestion_error_accessibility)` (e.g. "AI suggestion failed"). The row's click target stays on the whole Row, but the error icon itself needs a label because it conveys meaningful state. |
| `log/compose/ai/AiIssueItem.kt:138-167` | Animated `Canvas` + `sweep-gradient` — no AlloyUI equivalent for "processing ring". This is a genuine AlloyUI gap. Implementation is clean (uses `AlloyColorPallete.Purple700` / `AutodeskBlue700` tokens). | PASS (gap) | None. Document as "AlloyProgressRing variant needed for AI-gradient case" in the drift log. The existing `AlloyProgressRing` uses a different visual; a future `AlloyProgressRing.Gradient` could consolidate. |
| `log/compose/ai/AiIssueItem.kt:71-78` | `Row { ... }.clickable { onClick(issue.uid.uidString) }` — no `Role.Button` semantic, no ripple customization. Regular `IssueItem.kt` uses `combinedClickable` with long-press support; AI row has only `clickable`. | SUGGESTION | For click targets ≥48dp tall this is fine (row height = `STATUS_CIRCLE_SIZE + padding = 80dp`). But add `Role.Button` for a11y: `.clickable(onClickLabel = stringResource(R.string.issues_ai_open_suggestion), role = Role.Button) { ... }`. |
| `log/compose/ai/IssuesListSections.kt:28,57,108,119` | Hardcoded `16.dp` (divider horizontal padding) and `12.dp` (section divider top padding) instead of tokens. | NIT | `16.dp` → `Dimens.SpacingM`. `12.dp` → `Dimens.SpacingS`. Keeps spacing consistent if design ever tweaks the scale. |
| `log/compose/ai/IssuesSectionHeader.kt:34` | `background(AlloyColorPallete.Charcoal050)` — direct palette constant (not a semantic token). In dark mode this is hardcoded to the light-mode gray. | BLOCKER | Use `AlloyTheme.colors.surfaceVariant` or `AlloyTheme.colors.background2` (whichever exists as a semantic "pinned section header" token). If none exists, file a design-system ticket; a sticky header should not hardcode a palette shade. |
| `log/compose/ai/IssuesSectionHeader.kt:42` | `AlloyTheme.typography.body2.copy(fontWeight = FontWeight.Bold)` — ad-hoc typography mutation. AlloyUI has explicit `AlloyTextStyle` enum entries. | SUGGESTION | If no `AlloyTextStyle.BodyBold` exists, use `AlloyTextStyle.Caption` with bold override once. Acceptable for now but adds drift risk — same pattern appears in `IssueRowBody.kt:41`. |
| `log/compose/ai/IssueRowBody.kt:41` | Same ad-hoc `.copy(fontWeight = FontWeight.SemiBold)` on `body1`. Also matches the pre-existing pattern in `IssueItem.kt`, so it's consistent with the codebase. | NIT | If/when AlloyUI gains `AlloyTextStyle.BodySemiBold`, migrate. No action now — matches existing `IssueItem` behavior. |
| `log/compose/IssueItem.kt:59-60` | `Color(android.graphics.Color.parseColor(statusColor.fillColor(...)))` — string-parsed colors from a domain enum (`IssueStatus`). This is pre-existing (not from this feature), but violates the "no UIColor/palette leaks in domain layer" rule. | SUGGESTION | Out of scope for this feature. Flag for a future refactor: `IssueStatus` should expose `AlloyColorToken` instead of hex strings. |
| `log/compose/IssueItem.kt:87` | `AlloyCheckBox.AlloyCheckBox(...)` — correct AlloyUI usage. | PASS | None. |
| `log/compose/IssuesListComposable.kt:10-13` | Imports `androidx.compose.material.AlertDialog`, `Surface`, `Text`, `TextButton` (Material2). Used for the "no type access" dialog (lines 339-348). | SUGGESTION | Pre-existing legacy. Use `com.plangrid.android.components.compose.dialogs.AlertDialog` (same wrapper already used in `QuickCreateScreen.kt:33`). Fix in a follow-up cleanup. |
| `log/compose/IssuesListComposable.kt:292-294,475-477` | Inline `Brush.linearGradient(colors = listOf(AlloyColorPallete.AutodeskBlue700, AlloyColorPallete.Purple700))` — raw `Brush` construction for the QuickCreate FAB. Same gradient recipe is used in `AiIssueItem.kt:119-124` (where it's `AutodeskBlue500`+`Purple500`). | SUGGESTION | Extract to a shared `AlloyBrush.AiGradient700` / `AiGradient500` helper in `alloycompose`. Currently the 500-vs-700 split is intentional (lighter on the avatar, darker on the FAB per iOS parity) but the brush construction is duplicated 3× across the feature. |
| `log/compose/IssuesListComposable.kt:302,482` | Sparkle FAB uses `painterResource(com.plangrid.android.alloy.compose.R.drawable.ic_star_4_point)` directly. OK — this is the canonical AI icon drawable shipped in the alloycompose module. | PASS | None. |
| `alloycompose/widget/field/AlloyFieldLabel.kt:11,56` | Imports `androidx.compose.material.Icon` (Material2) and uses it for the AI-indicator star inside the SHARED design-system label. This is the worst place for a raw `Icon` — it leaks Material2 into every `AlloyInputField` call-site. | BLOCKER | Replace with `AlloyImage.AlloyIconView(painter = painterResource(R.drawable.ic_star_4_point), iconColors = AlloyStateColorsDefault.colors(AlloyTheme.colors.tertiary_03_500, AlloyTheme.colors.tertiary_03_500), enabled = enabled)`. The whole premise of AlloyUI is that `widget/field/` never imports `androidx.compose.material.Icon`. |
| `alloycompose/widget/field/AlloyFieldLabel.kt:55,60` | Hardcoded `4.dp` spacer width and `12.dp` star size. | NIT | `Dimens.SpacingXXS` (4dp) and — if no Dimens.IconSmall token — leave 12.dp as a local constant with a comment (it's scaled to the 75% floating-label size from the KB doc). |
| `alloycompose/widget/field/AlloyFieldLabel.kt:57` | `contentDescription = null` on the star Icon. OK — it's decorative and the field's label already carries the semantic meaning. | PASS | None. |
| `alloycompose/widget/field/LocalShowAiIndicator.kt:17` | Clean `compositionLocalOf { false }`. No styling in this file. | PASS | None. |
| `alloycompose/widget/field/AlloyInputField.kt:88,161` | New `showAiIndicator: Boolean = false` parameter threaded through both public entry points. Default false preserves existing call-sites. | PASS | None. |
| `alloycompose/widget/field/AlloyInputFieldFoundation.kt:460-463` | `CompositionLocalProvider(LocalAlloySttProvider provides sttProvider, LocalShowAiIndicator provides showAiIndicator)` — exactly as the KB doc prescribes. | PASS | None. |
| `alloycompose/widget/field/AlloyInputFieldContent.kt:301-312` | `AlloyFieldLabel(...)` replaces the inline `AlloyTextView` for the label. Correct per KB Approach 3. | PASS | None. |

### Severity Totals
- **BLOCKER**: 5 (hardcoded `Icon` in AlloyFieldLabel; `Icon` in AiIssueItem; hardcoded `Charcoal050` in sticky header; hardcoded English `"enabled"/"disabled"`; missing a11y label on error icon)
- **SUGGESTION**: 10
- **NIT**: 9

---

## Positive Notes

1. **CompositionLocal pattern is textbook-clean.** `LocalShowAiIndicator` + `AlloyFieldLabel` exactly matches the KB Approach 3 spec (no parameter threading through `AlloyEditText`, `AlloyExposedField`, etc.). Default `false` means zero risk to existing call-sites.
2. **Alloy token coverage is strong where it counts.** `AlloyTheme.colors.tertiary_03_500` (Purple500) is correctly wired for the floating-label AI star per ADR Decision 5. The AI gradient uses `AlloyColorPallete.AutodeskBlue500/700` + `Purple500/700` — no random hex codes.
3. **The `/ai/` package split is excellent.** `AiIssueItem`, `IssueRowBody`, `IssuesSectionHeader`, `IssuesListSections`, `AiRowPreview` cleanly isolate the feature. `IssueRowBody` is shared with the regular `IssueItem`, preventing typography drift — this is the mitigation Trinity promised in plan §8 mitigation 3.
4. **Previews are comprehensive.** `AiRowPreview.AiAndRegularRowStackPreview` stacks every AI state next to a regular `IssueItem` so any padding/ellipsis drift is caught by screenshot review. Three individual `@Preview`s per state plus a combined one — gold standard for AlloyUI component work.
5. **`QuickCreateActionCard` correctly threads `showAiIndicator = renderData.isAiEffectivelyOn`** from `QuickCreateScreen.kt:139` through to `AlloyInputField.AlloyExpandableInputField` — the SR-15 (now scoped to user-facing field) is implemented end-to-end.
6. **Correct AlloyUI usage in `QuickCreateAiToggleRow`**: `AlloyText.AlloyTextView` + `AlloySwitch.AlloyToggleSwitch` + `Dimens.SpacingM/S`. Clean, no Material2 imports.
7. **FAB gradient reuse is consistent.** The QuickCreate FAB and the AI row avatar use the same Blue→Purple gradient recipe, visually tying the entry point to the output.

---

## Verdict: ALLOY_FAIL

**Rationale.** The 5 BLOCKER items must land before the feature ships:

1. **`AlloyFieldLabel.kt:56`** — raw `androidx.compose.material.Icon` inside the SHARED design-system label. This leaks Material2 into every `AlloyInputField` call-site once any consumer opts into `showAiIndicator`. Non-negotiable: swap for `AlloyImage.AlloyIconView`.
2. **`AiIssueItem.kt:18,92,128`** — same `material.Icon` leak in the feature module. Two places (error glyph + star-on-gradient). Both must use `AlloyImage.AlloyIconView`.
3. **`AiIssueItem.kt:92`** — error glyph has `contentDescription = null`. A11y users get zero feedback that a suggestion errored. Add a localized label.
4. **`IssuesSectionHeader.kt:34`** — sticky header uses `AlloyColorPallete.Charcoal050` directly. This hardcodes the light-mode gray; dark mode will render as the light palette value. Must be a semantic `AlloyTheme.colors.*` token.
5. **`QuickCreateAiToggleRow.kt:39`** — TalkBack state description `"enabled"/"disabled"` is hardcoded English. Must be localized via `stringResource`.

The 10 SUGGESTION items (mixed Material2/3, raw `Color.Black`, ad-hoc `TextStyle` in the photo-counter overlay, `Arrangement.spacedBy(-16.dp)` hack, etc.) are consistency/readability improvements that should be addressed in a follow-up cleanup PR but do not block merge if the 5 BLOCKERS are fixed.

Once the BLOCKERS are resolved, the feature is clean and ready to ship under AlloyUI standards.
