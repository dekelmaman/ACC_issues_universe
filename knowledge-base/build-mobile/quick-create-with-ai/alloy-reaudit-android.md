# Alloy Re-Audit: quick-create-with-ai — android

Re-audit of the 5 BLOCKERs flagged in `alloy-audit-android.md` after Trinity applied fixes.
All evidence verified by reading current file state (staged changes present).

## Per-Blocker Status

| Blocker | Status | Evidence |
|---------|--------|----------|
| **B1** — raw `Icon` in `AlloyFieldLabel` (shared design-system label) | **PASS** | `android/alloycompose/src/main/java/com/plangrid/alloy/compose/widget/field/AlloyFieldLabel.kt:7-23` — no `androidx.compose.material.Icon` import. `AlloyFieldLabel.kt:56-64` uses `AlloyImage.AlloyIconView` with `AlloyStateColorsDefault.colors(colorEnabled = AlloyTheme.colors.tertiary_03_500, colorDisabled = AlloyTheme.colors.tertiary_03_500)` and `contentDescription = null` (decorative — field label carries semantics). |
| **B2** — raw `Icon` in `AiIssueItem` + null `contentDescription` on error icon | **PASS** | `android/issues/src/main/java/com/plangrid/android/issues/log/compose/ai/AiIssueItem.kt:3-43` — no `androidx.compose.material.Icon` import. Error icon at `AiIssueItem.kt:92-102` is `AlloyImage.AlloyIconView` with `contentDescription = stringResource(R.string.issues_ai_error_icon_description)`. String confirmed at `android/issues/src/main/res/values/strings.xml:17` → `"AI suggestion failed"`. Decorative star at `AiIssueItem.kt:131-139` also migrated to `AlloyIconView` (bonus — keeps `contentDescription = null`, ornament). |
| **B3** — `AlloyColorPallete.Charcoal050` on sticky header | **PASS** | `android/issues/src/main/java/com/plangrid/android/issues/log/compose/ai/IssuesSectionHeader.kt:34` — `background(AlloyTheme.colors.tertiary_07_050)`. No `AlloyColorPallete` import remains in this file (`IssuesSectionHeader.kt:12-15`). Semantic theme token → dark-mode safe. |
| **B4** — hardcoded English `"enabled"/"disabled"` in TalkBack state | **PASS** | `android/issues/src/main/java/com/plangrid/android/issues/quick_create/composables/QuickCreateAiToggleRow.kt:39-45` — uses `stringResource(id = if (checked) R.string.issues_quick_create_ai_toggle_enabled else R.string.issues_quick_create_ai_toggle_disabled)`. Both strings defined in `android/issues/src/main/res/values/strings.xml:24-25`. TalkBack will now announce in device locale. |
| **B5** — `Color.Black`/`Color.White` + inline `TextStyle` in photo-counter overlay | **PASS** | `android/issues/src/main/java/com/plangrid/android/issues/quick_create/composables/QuickCreateImageArea.kt:113-130` — overlay Box background is `AlloyTheme.colors.tertiary_11.copy(alpha = 0.5f)`; text uses `AlloyText.AlloyTextView` with `textStyle = AlloyTheme.typography.caption` and `textColor = AlloyTextColor.Contrast`. No `Color.Black` / `Color.White` / inline `TextStyle(` inside the overlay. (Outer Box `background(Color.Black)` at `QuickCreateImageArea.kt:77` was a SUGGESTION, not the B5 scope — unchanged and still acceptable per original audit.) |

## DRIFT_ALLOY_2A verdict

**ACCEPT** — rationale:

The decorative white 4-point star sits inside the gradient `AiStatusIndicator` circle at
`AiIssueItem.kt:131-139`. The parent `AiIssueItem` is a `Row.clickable { onClick(...) }` whose
content (displayId + title + assignee) is read aggregated by TalkBack on the click target. The
star is **pure ornament** — it conveys no information beyond "this row is an AI suggestion", which
is already implied by the section header ("Generate with AI") and by the error/processing visual
state that has its own labeled icon.

Setting `contentDescription = null` on decorative imagery adjacent to semantic text is the
standard Compose accessibility idiom (per AOSP `Image`/`Icon` documentation and Material a11y
guidelines). Announcing "AI" on the star would **double-announce** alongside the section header,
which harms TalkBack UX rather than helping it.

The B2 fix correctly distinguishes between:
- **Decorative star** (ornament, `null` OK) — stays `null`
- **Error warning icon** (state indicator, conveys meaning) — gets a localized label

This split matches the intent of the original BLOCKER callout and is aligned with WCAG 1.1.1
decorative-image guidance.

## Verdict: **ALLOY_PASS**

All 5 blockers resolved with verified file-state evidence. One documented drift
(`DRIFT_ALLOY_2A`) accepted as standard a11y practice.

---

### Summary of remaining non-blocker items (unchanged — SUGGESTION/NIT)

These were NOT blockers in the original audit; no re-verification required, but flagged for
future cleanup passes:

- `QuickCreateImageArea.kt:77` — outer `.background(Color.Black)` (media backdrop, pre-existing)
- `QuickCreateScreen.kt:85` — same `Color.Black` on `BoxWithConstraints` (pre-existing)
- `QuickCreateToolbar.kt:48,85-96` — mixed Material2/Material3 imports; raw `Icon` for sparkle
- `AiIssueItem.kt:95` — `Charcoal900` on warning tint (dark-mode drift, SUGGESTION)
- `IssuesListComposable.kt:292-294,475-477` — duplicated `Brush.linearGradient` recipe
