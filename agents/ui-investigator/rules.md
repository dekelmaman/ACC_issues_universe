## Hard Rules

- ALWAYS request a screenshot before reading code. No screenshot = 80% confidence cap.
- ALWAYS decompose into components. Never produce a single monolithic spec for a full screen.
- ALWAYS identify blockers (model gaps, data layer changes) and put them at the TOP of each spec.
- ALWAYS trace every visual element to its data source. No element without a source.
- NEVER use platform-specific language in specs. No SwiftUI, Compose, UIKit, XML, CSS.
  - Say "40px circle" not "Circle().frame(width: 40)"
  - Say "bold headline" not ".headlineBold" or "fontWeight(.bold)"
  - Say "yellowOrange500" not "#F5A623" or "Color.orange"
- NEVER assume — verify from code. If two interpretations exist, ask the human via AskUserQuestion.
- NEVER ask questions you can answer from code (colors, sizes, fonts that are readable in source).
- ALWAYS include a "Scope" section per component — what it owns AND what it does NOT own.
- ALWAYS include a "Divergences" table when a rewrite version exists.
- Save specs to `./specs/<feature>/ui/<component>/ui-spec.md`.
- Save the screen composition to `./specs/<feature>/ui/screen-composition.md`.

## Confidence Gate

- 100% confidence (code + screenshot agree) → auto-document
- <100% confidence (ambiguous, multiple interpretations, design intent) → ask human via AskUserQuestion
- Questions must be: specific, screenshot-grounded, verified against both codebases first

## Spec Format

Every component spec MUST have these sections in this order:
1. Blockers (at the TOP)
2. Scope (what this component owns / does NOT own)
3. Layout (ASCII tree)
4. Elements (with data source, formatting, states, conditions)
5. Styling table
6. Interactions table
7. Data Sources table
8. Conditional Rendering table
9. Divergences from Rewrite (if applicable)
10. Investigation Log
