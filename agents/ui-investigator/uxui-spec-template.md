# Symmetric UI Spec Template

This template is the SAME for both `current-spec.md` and `target-spec.md`. Symmetry is
mandatory — that is what makes the gap a clean text-vs-text diff.

## When to use

- `spec-current` step → describe the running app + current code in this format
- `spec-target` step → describe the target Figma / target screenshots in this format
- The two specs MUST share section names, table headers, and ordering — otherwise the
  gap-analysis step cannot diff them mechanically.

## Mandatory sections (in this order)

```markdown
# UI Spec: <feature name> @ <state name>

## Source
- Type: [code+screenshot | figma | screenshot-only]
- Code path: <path or N/A>
- Screenshot paths: <comma-separated paths or N/A>
- Figma URL + node-id: <url or N/A>
- Captured: <ISO date>

## Layout (top-to-bottom)

> One bullet per discrete container or section. Children indented underneath.
> Mention edge insets, alignment, and whether a section is conditional.

- Section: <name> (purpose, e.g. "navigation bar")
  - position: top | center | bottom | overlay
  - background: <token | "transparent" | "color-with-opacity">
  - padding: top=N, leading=N, trailing=N, bottom=N
  - children:
    - Element: <type — see Elements table below>
    - ...
  - conditions: <e.g. "hidden when isLoading" — or "always shown">
- Section: ...

## Elements

> One row per leaf-level element (Text, Button, Icon, Toggle, Image, etc.).
> If an element appears in multiple sections, list once with `appears in: [sections]`.

| ID | Element type | Text/Asset | Tokens | Size | Position | Conditions | Appears in |
|----|--------------|------------|--------|------|----------|------------|------------|
| E1 | Text | "Title" or `key.path` | font=`bodyBold`, color=`text.primary` | h2 | center, top | always | nav bar |
| E2 | Button (solid) | "Save" or `key.path` | bg=`primary.primary`, content=`primary.onPrimary` | normal | trailing, bottom | enabled when valid | bottom-buttons |
| E3 | Icon | `aiSparkleFilled` | color=`tertiary.purple500` | 24×24 | leading, top | when AI enabled | accessories-bar |

## Tokens referenced

> Every UNIQUE design-system token used by the spec. The `Status` column is filled in
> only on the TARGET spec (left blank on current).

### Colors

| Token | Used by | Status (TARGET only) |
|-------|---------|----------------------|
| `tertiary.purple500` | E3, E5 | EXISTS |
| `tertiary.purple800` | E7 | MISSING |

### Icons

| Token | Used by | Status |
|-------|---------|--------|
| `aiSparkleFilled` | E3 | EXISTS |

### Fonts

| Token | Used by | Status |
|-------|---------|--------|
| `bodyBold` | E1 | EXISTS |

## Strings referenced

| Key | Value (current language) | Used by |
|-----|--------------------------|---------|
| `issues.quick_create.title` | "Quick create issue" | E1 |
| `issues.quick_create.generate_button_text` | "Generate" | E2 |

## States

> Variants of the same screen — describe deltas relative to the default layout.

### Default
(described above)

### Compact (iPhone) vs Regular (iPad)
| Aspect | Compact | Regular |
|--------|---------|---------|
| Toggle label | hidden | "Generate with AI" |
| Bottom button width | equal-width | intrinsic, right-aligned |

### Loading
- Replace E2 with progress indicator
- Disable E5

### Empty / Error
- ...

## Notes

- Anything that cannot fit cleanly into the structured sections above
- Designer-comment annotations
- Edge cases discovered during investigation
- Outstanding questions for the user
```

## Symmetry rules (MANDATORY)

1. **Same section headers** in `current-spec.md` and `target-spec.md`. Do not rename.
2. **Same Element IDs** when an element exists in both — gap-analysis matches by ID.
   - If a target element is brand-new, give it a fresh ID (E20+) and note "NEW".
   - If a current element is removed, leave the row in but note "REMOVED in target".
3. **Same column order** in every table.
4. **Token names are exact** — no paraphrasing. Use the design-system's canonical name
   (e.g. `tertiary.purple500`, not "purple 500" or "the purple").
5. **Mark TARGET token status** explicitly (EXISTS / MISSING). Never leave blank.
6. **Don't include implementation guidance in the spec** — the spec is product-level.
   Code comments, file paths, and "how to implement" go in the GAP, not the spec.

## Filling in `current-spec.md`

- Trace from screenshots and code (delegate to Sherlock for cross-file work).
- For tokens, read the actual code — never guess from screenshot color.
- For strings, use the localization key NAME and current value side-by-side.

## Filling in `target-spec.md`

- Source = Figma if available; screenshot otherwise.
- For tokens, infer the canonical token name from the design-system mapping (e.g.
  `purple500`). If the Figma swatch maps to no known token → mark MISSING.
- Token status is the single most important output of `target-spec.md` — it drives the
  `prepare-tokens` step.

## Why symmetric?

A code-vs-design comparison is open-ended and ambiguous. Two structured specs in the
SAME format admit a mechanical diff: same Element IDs → compare attributes, missing
IDs → "added" or "removed", different token values → "changed". `gap-analysis` builds
the implementation plan directly from this diff — no human judgment required at the
diff level.
