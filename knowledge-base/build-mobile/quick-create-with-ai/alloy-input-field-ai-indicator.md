# AlloyInputField AI Indicator — Approach 3 (CompositionLocal)

> **Source**: `.claude/specs/alloy-input-field-modified-indicator.md`
> **Decision**: Approach 3 selected for quick-create AI feature
> **Date**: 2026-04-16

## What It Does

Adds a purple 4-pointed star icon at the end of the floating label on `AlloyInputField` to indicate AI-enriched content. Visible only when the field has a value (label is floating).

## How It Works

A `CompositionLocal` (`LocalShowAiIndicator`) carries the indicator state down the composable tree. A shared `AlloyFieldLabel` composable reads the local and conditionally renders the star. All label rendering paths use this same composable.

```
AlloyInputField(showAiIndicator = true)
  └── AlloyInputFieldFoundation
        └── CompositionLocalProvider(LocalShowAiIndicator provides true)
              └── [any rendering path]
                    └── AlloyFieldLabel(text, ...)
                          └── Row { Text + if(LocalShowAiIndicator.current) StarIcon }
```

## Production Changes Required

| Action | File | Change |
|--------|------|--------|
| CREATE | `LocalShowAiIndicator.kt` | `compositionLocalOf { false }` |
| CREATE | `AlloyFieldLabel.kt` | Shared label composable: `Row { Text + conditional Star }` |
| MODIFY | `AlloyInputField.kt` | Add `showAiIndicator: Boolean = false` param |
| MODIFY | `AlloyInputFieldFoundation.kt` | Wrap output with `CompositionLocalProvider` |
| MODIFY | `AlloyEditTextFoundation.kt` | Replace label `AlloyTextView` with `AlloyFieldLabel` |
| MODIFY | `AlloyInputFieldContent.kt` | Replace label `AlloyTextView` with `AlloyFieldLabel` |
| MODIFY | `AlloyExposedField.kt` | Replace label rendering with `AlloyFieldLabel` |

## Key Design Properties

- **Star color**: `AlloyTheme.colors.tertiary_03_500` (Purple500)
- **Star icon**: `ic_star_4_point.xml` (4-pointed star vector, tinted at runtime)
- **Star scales with label**: Yes (~75% when floating — barely noticeable at 12dp)
- **Border gap**: Automatically includes star width (Material measures full label composable)
- **RTL**: Automatic (Row handles it)
- **Positioning reliability**: 100% (no coordinate tracking)

## Why Approach 3 Over Alternatives

| Criteria | Approach 1 (Row) | Approach 3 (CompositionLocal) |
|----------|------------------|-------------------------------|
| Parameter threading | Through ALL layers | Foundation + Local (zero threading through intermediates) |
| Future extensibility | Low (boolean per feature) | High (`AlloyFieldLabel` is single source of truth) |
| Shared label composable | No | Yes — one composable for all paths |
| Adding next label feature | Thread another boolean everywhere | Add to `AlloyFieldLabel` only |

Approach 3 was chosen because quick-create with AI will likely add more label features (badges, confidence indicators, etc.). The `AlloyFieldLabel` abstraction pays for itself with the second feature. Zero parameter threading through `AlloyEditText`, `AlloyExposedField`, and other intermediate layers.

## POC Status

- POC implemented at `alloycompose/src/.../field/ModifiedIndicatorApproach3.kt`
- Preview comparison at `ModifiedIndicatorPreview.kt`
- Star drawable at `res/drawable/ic_star_4_point.xml`

## Usage in ADR

When writing the quick-create Android ADR, reference this file for the AI indicator approach:
- Requirement mapping: "AI-enriched fields show visual indicator" -> CompositionLocal + AlloyFieldLabel
- Component plan: use the file change table above
- EARS: WHEN a field contains AI-suggested content, THE floating label SHALL display a purple star icon within the same animation frame as the label float
