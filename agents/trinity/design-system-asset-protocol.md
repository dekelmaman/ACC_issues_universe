# Design System Asset Protocol

When the implementation requires a design-system token (color, icon, font) that you suspect
does NOT exist in the design-system module, follow this protocol exactly.

## Cardinal rule: NEVER add an asset silently

The cost of asking the user is one prompt. The cost of a wrong asset (wrong hex, wrong icon
weight, wrong font fallback) baked into the design system is a future cleanup PR. Always ask.

## Procedure

### Step 1: Search exhaustively

Before claiming a token is missing, prove it. Run searches in this order:

1. **Exact name** — Grep the design-system module for the token name (e.g. `purple-800`).
2. **Alternate spellings** — Grep for variations: `purple_800`, `Purple-800`, `purple800`,
   `PURPLE_800`. Designs systems are not always consistent.
3. **Adjacent values** — If asking for the 800 shade, look at 700 and 900 to confirm the
   ramp pattern (does the system go 100/300/500/700? Then 800 may be intentional skip).
4. **Asset bundles** — For colors/icons: list the asset catalog folder
   (e.g. `Colors.xcassets/Purple/`, `Icons.xcassets/`) to see actually-shipped variants.
5. **Token registry** — Read the central token-mapping file (e.g. `ColorTokens.swift`,
   `IconAsset.swift`) end-to-end. Don't trust grep alone.

Report back what you searched and what you found. Even when missing, surface the closest
alternatives.

### Step 2: Ask the user

Use AskUserQuestion. Present the findings + four explicit options:

```
Token `<name>` is not in the design system.

Searched:
- Exact: not found in <files searched>
- Adjacent: <list of related shades/variants found>
- Closest match: <name and where it lives>

How should I proceed?

A) Add `<name>` to the design system
   - Provide the value (hex / asset / etc.) or accept my proposal: <proposal>
   - I will follow the multi-file add procedure for <type>
B) Use the closest existing alternative: <name>
   - I will swap this in throughout the impl plan
C) Skip this requirement
   - Mark the affected gap rows DEFERRED
D) Stop the workflow — I want to investigate
```

Wait for an explicit user choice. NEVER pick a default.

### Step 3: If user picks A — add it

The exact files depend on platform and design-system. Generic principle: the design system
typically has a multi-layer structure that ALL needs updating coherently:

- **Asset layer** — the binary (color JSON, icon SVG/PDF, font file)
- **Token layer** — a constant in code that points to the asset
- **API layer** — the public enum / accessor users call (with switch case mapping)

For a NEW COLOR (typical 3-file change on iOS):
1. Asset: `<DesignSystem>/Resources/Colors.xcassets/<Family>/<Name>.colorset/Contents.json`
2. Token constant: `<DesignSystem>/Components/Color/ColorTokens.swift`
3. Public enum + switch: `<DesignSystem>/Components/Color/AlloyColors.swift`

For a NEW ICON (typical 2-file change on iOS):
1. Asset: `<DesignSystem>/Resources/Icons.xcassets/<Name>.imageset/...`
2. Public enum: `<DesignSystem>/.../IconAsset.swift`

For a NEW FONT (typical 3-file change on iOS):
1. Asset: `<DesignSystem>/Resources/Fonts/<File>.ttf` AND `Info.plist` registration
2. Token: `<DesignSystem>/Components/Font/FontTokens.swift`
3. Public enum: `<DesignSystem>/Components/Font/AlloyFonts.swift`

If you do not know the project's exact layout, **delegate to Sherlock**: "How does
`<DesignSystem>` add a new <color | icon | font>? What files must change?"

After the addition:
- Verify the project still builds
- Confirm the new token is reachable from a test usage in the impl plan's primary file

### Step 4: If user picks B — substitute

- Update the impl plan: replace `<missing-token>` with `<existing-alternative>` in every
  gap row that referenced it.
- Document the substitution in the gap-analysis revision history so reviewers see why
  the design's exact token wasn't used.

### Step 5: If user picks C — defer

- Mark every gap row using the missing token as `DEFERRED — pending design-system add`.
- The workflow proceeds with the un-deferred rows.
- Add a TODO note to the gap analysis so the user can revisit.

### Step 6: If user picks D — stop

- Halt the workflow at this step.
- Output the searches you did + the choices the user could pick.
- The user resumes when they're ready.

## What NOT to do

- DO NOT pick the "closest alternative" silently because it looks similar. The design
  system uses specific shades/weights for specific purposes; a 700 vs 800 swap can be
  meaningful.
- DO NOT add an asset because the user implicitly approved by saying "go" earlier in the
  workflow. "Go" approves the workflow, not unilateral design-system mutations.
- DO NOT make the asset add look small to encourage easy approval. Show every file that
  will change BEFORE asking for approval.
- DO NOT bundle multiple asset additions into one approval. Ask per asset.

## Sibling-feature consistency

When you add a new token, immediately also check: does this new token replace usages of
an OLDER token in OTHER features? (E.g. if a new "AI purple" replaces "interactive blue"
for an AI surface, audit all AI surfaces in the codebase for the same swap.)

Surface this finding to the user as part of the same approval — but separately:
"Now that purple-800 exists, should I update <list of other AI-themed views> to use it?"
Default = no changes; user opts in per file.
