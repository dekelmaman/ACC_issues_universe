# Lens Memory

## Learnings

### Pin Badge Text (2026-03-18)
The issue list pin badge shows the ISSUE TYPE abbreviation ("PL", "CL", "CM"), NOT the status abbreviation ("O", "R", "P"). The color comes from status, but the text comes from the type. This was a major divergence in the first rewrite — confirmed by user. The type abbreviation is user-configured per issue type and lives on the IssueType model, not the Issue model.

### Component Boundaries Matter (2026-03-18)
First investigation produced a single screen-level spec. The implementing agent only replaced one view component (the list), missing toolbar, search bar, and FAB which were owned by the parent (SplitView). Lesson: ALWAYS decompose into components and map ownership. A "screen" is multiple view files.

### Blockers Must Be Top-Level (2026-03-18)
The `typePinCode` model gap was documented in a "Divergences" table. The implementing agent treated it as optional and used a fallback. Blockers must be at the TOP of the spec, not buried in tables.

### Localization Matters (2026-03-18)
Title format "#903 - Punch List" uses a localization key with dash separator. First rewrite used inline string interpolation without dash. Assignee fallback "Unassigned" must also be localized. Always note when text comes from localization.
