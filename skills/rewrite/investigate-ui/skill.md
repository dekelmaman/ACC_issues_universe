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

### 4. Find Source Code
Search the codebase for ALL view files that contribute to this screen.
Find the root/container, child views, reusable sub-components.
Also find the rewrite version if it exists.

### 5. Component Decomposition
Map each visual checklist item to the source file that renders it.
Output a component tree showing parent-child relationships.
Every visual element must be assigned to exactly one component.

### 6. Trace Data Sources
For each visual element:
- Find the code that renders it
- Trace the data source (model property → origin)
- Note transforms (localization, formatting, color mapping)
- Mark confidence: confirmed / observed / uncertain

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
Save `screen-composition.md` + one `ui-spec.md` per component.
Each spec must have: Blockers, Scope, Layout, Elements, Styling, Interactions, Data Sources, Conditional Rendering, Divergences, Investigation Log.

## Platform-Agnostic Rules
- No framework names (SwiftUI, Compose, UIKit, XML, CSS)
- Colors by token name ("yellowOrange500") not hex or framework color
- Typography by role ("bold headline") not framework style
- Sizes in px/pt, not framework units
- Track localization keys — note when text is localized
