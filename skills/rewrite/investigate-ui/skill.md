# Investigate UI — Reverse-engineer a screen into component specs

## Required Capabilities
- Read source files (view code, models, state management)
- Search codebase by pattern (find view files for a feature)
- Read image files (screenshots)
- Ask user questions (for ambiguous elements)
- Write files (output spec documents)

## Inputs
- Screen/feature name or path
- Screenshot of the legacy/production version (mandatory, ask if not provided)
- Optional: screenshot of the current rewrite for comparison

## Outputs
- `screen-composition.md` — component tree, ownership map, blockers, data flow
- One `ui-spec.md` per component — scoped spec with elements, data sources, styling, interactions

## Process

### 1. Clarify Scope
If the target is ambiguous, ask:
- Full screen or single component?
- Which codebase version? (legacy/production, current rewrite, or both)

### 2. Get Screenshot
Before reading code, ensure you have a screenshot. Ask if not provided.
No screenshot = 80% confidence cap on visual elements.

### 3. Visual Checklist
Number every distinct visual element in the screenshot. This list drives everything.

For EVERY text element, ask Sherlock to verify these properties from code:
- **Line limit** (single line? multi-line? max lines?)
- **Truncation** (truncated with "..."? clipped? wrapped?)
- **Text overflow** (what happens when text is too long?)

For EVERY visual element, ask Sherlock to verify:
- **Exact size** (fixed? flexible? min/max constraints?)
- **Padding/margins** (exact values from code)
- **Corner radius** (if rounded)
- **Opacity** (any conditional opacity changes?)
- **Visibility conditions** (when shown/hidden?)
- **Accessibility** (label, trait, identifier)

Do NOT rely on screenshot alone for these — screenshots can't tell you line limits or truncation behavior. The CODE is the source of truth for visual properties.

### 4. Find Source Code
Search the codebase for ALL view files that contribute to this screen.
Find the root/container, child views, reusable sub-components.
Also find the rewrite version if it exists.

### 5. Component Decomposition
Map each visual checklist item to the source file that renders it.
Output a component tree showing parent-child relationships.
Every visual element must be assigned to exactly one component.

### 6. Trace Data Sources AND Visual Properties
For each visual element, delegate to Sherlock to verify from code:

**Data source:**
- Find the code that renders it
- Trace the data source (model property → origin)
- Note transforms (localization, formatting, color mapping)
- Mark confidence: confirmed / observed / uncertain

**Visual properties (MANDATORY — do NOT skip):**
- Text: lineLimit, truncationMode, font/style, color, alignment
- Container: padding, spacing, frame constraints (width/height/min/max)
- Shape: cornerRadius, fill color, stroke, shadow
- Image/Icon: size (explicit frame), renderingMode, tint color
- Layout: alignment within parent, spacing between siblings
- Conditional: opacity changes, hidden states, disabled appearance

Every property must be either confirmed from code or flagged as uncertain. Never assume "it looks like one line" from a screenshot — verify with Sherlock that the code sets `lineLimit(1)` or equivalent.

### 7. Identify Blockers
Check if any element requires changes outside the view layer:
- Model doesn't carry needed data
- Repository/datasource needs new query
- Missing service or dependency
- Cross-component wiring needed

These go at the TOP of each spec.

### 8. Hidden Elements
Read code for states not visible in screenshot:
- Edit mode, loading, error, empty, disabled
- Feature flag gated elements
- Different data states

### 9. Compare Versions
If rewrite exists, identify divergences per component.
For each: can code alone determine which is correct? If not, ask.

### 10. Ask Questions
Use AskUserQuestion for uncertain items. Rules:
- Specific: reference exact element
- Screenshot-grounded: "I see X in screenshot, found Y in code"
- Verified first: check both codebases before asking

### 11. Write Specs

Save to `./specs/<feature>/ui/`:
- `screen-composition.md` at the root
- `<component>/ui-spec.md` per component

Each component spec MUST have these sections in this order:
1. **Blockers** (at the TOP — model gaps, data layer changes needed)
2. **Scope** (what this component owns AND what it does NOT own)
3. **Layout** (ASCII tree)
4. **Elements** (each with: visual, content, data source, formatting, text constraints [lineLimit, truncation], size constraints [frame, min/max], states, conditions)
5. **Styling table** (element, size, color/token, typography, lineLimit, truncation, padding, cornerRadius, notes)
6. **Interactions table** (trigger, element, action)
7. **Data Sources table** (element, source property, transform, example value)
8. **Conditional Rendering table** (condition, shows, hides)
9. **Divergences from Rewrite** (if applicable: element, legacy, rewrite, resolution)
10. **Investigation Log** (screenshot status, checklist count, auto-documented, human-answered, unresolved)

## Confidence Gate

- **100% confidence** (code + screenshot agree, single interpretation) → auto-document, no question needed
- **<100% confidence** (ambiguous, multiple interpretations, design intent unclear) → ask human via AskUserQuestion
- Questions must be: specific, screenshot-grounded, verified against both codebases first
- Never ask the HUMAN about things provable from code — but DO ask SHERLOCK. Every visual property must be verified from code via Sherlock, not assumed from the screenshot.

## Mandatory Sherlock Verification

For every element in the spec, Lens MUST delegate to Sherlock to confirm these from code before writing the spec. Do NOT write a spec element without code evidence for:
- Text line limits and truncation behavior
- Exact padding/margin values
- Frame constraints (fixed size, min/max, flexible)
- Colors (token names, not visual guesses)
- Font/typography (style name, not visual guess)
- Corner radii
- Shadow parameters
- Conditional visibility logic
- Accessibility labels and identifiers

If Sherlock can't find a property in code, mark it as "uncertain (not found in code)" in the spec — never assume defaults.

## Platform-Agnostic Rules
- No framework names (SwiftUI, Compose, UIKit, XML, CSS)
- Colors by token name ("yellowOrange500") not hex or framework color
- Typography by role ("bold headline") not framework style
- Sizes in px/pt, not framework units
- Track localization keys — note when text is localized
