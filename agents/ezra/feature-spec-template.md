# Feature Spec Template

This is the mandatory template that Ezra must use for all feature specifications. Every spec must follow this exact structure with all sections included.

## Usage

**For Base Specs:** Use this template for the shared foundation spec.
- Header: `# [Feature Name] - Feature Spec (v1.0)`
- Include shared user journeys, layouts, components, and analytics (naming conventions + common parameters)

**For Phase Specs:** Use this template for each phase, but:
- Header: `# [Feature Name] - [Phase Name] - Feature Spec (v1.0)` (e.g., "v2.0" if updating)
- Start with: "This spec extends `<feature>-spec.md` (base v[version])"
- **DELTA ONLY:** Include only sections that are new or changed in this phase
- Reference base spec instead of repeating shared sections
- Include phase-specific analytics (new/modified events for this phase only)

## Version Bumping Protocol

When updating any spec, **ALWAYS ASK THE USER:**
- "Should we bump [spec name] from v1.0 to v1.1?"
- "Should we also bump the base/related specs due to cross-dependencies?"
- Document the rationale: "Bumping because [shared sections changed / phase-specific sections changed]"

This ensures version numbers stay synchronized and readers understand what changed.

---

```markdown
# [Feature Name] - Feature Spec (v1.0)

**For phase specs:** This spec extends `<feature>-spec.md` (base v1.0)

## Overview
Brief description of what the feature does and its purpose.
Include any important scope limitations (e.g., "covers non-AI mode only").

---

## Overview
Brief description of what the feature does and its purpose.
Include any important scope limitations (e.g., "covers non-AI mode only").

---

## User Journeys
Complete end-to-end scenarios showing how real users experience the feature.
Each journey is a named narrative from trigger to resolution.

### Happy Path
Numbered steps from entry to successful completion.
Include what the user sees, taps, and what feedback the system gives at each step.

### Error Recovery
What happens when something goes wrong mid-flow and how the user recovers.

### Interrupted Flow
User leaves mid-task (backgrounding, navigation away, force-close) — what state
is preserved, what is lost, and what happens when they return.

### Returning User
What the experience looks like on second use — sticky defaults, remembered
preferences, any onboarding that doesn't repeat.

---

## User Flow
Step-by-step workflow from entry point to completion.
Include decision points and alternative paths.
Use numbered lists for main flow, bullet points for variations.

### Entry Points
Table of where users can access this feature.

### [Specific Behaviors]
Any unique workflow patterns or modes specific to this feature.

---

## Screen Layout
ASCII diagram showing the visual hierarchy and component relationships.
Include overlays, cards, toolbars, and their relative positions.

---

## Visual States
Describe every visual state the feature can be in and what the user sees.
This section ensures designers and developers build all states, not just the happy path.

### [State Name]
| Element | Appearance | User action available |
|---------|------------|----------------------|
| [UI element] | [What it looks like in this state] | [What the user can do] |

Common states to cover:
- **Default / idle** — first load, nothing in progress
- **Loading / in progress** — waiting for a response (spinner? shimmer? skeleton?)
- **Success** — operation completed, what changes visually
- **Error / failure** — what the user sees, how they retry or dismiss
- **Empty** — no data to show (first use, cleared state)
- **Disabled / unavailable** — feature gated off, what's hidden vs grayed out

---

## Components

### [Component Name]
Detailed specification for each UI component.

| Element | Description |
|---------|-------------|
| [Element] | [Behavior/Purpose] |

**Constants:**
| Property | Value/Constraint |
|----------|------------------|
| [Property] | [Value] |

**Behavior:**
- Bullet points describing interactions, state changes, and responses
- Include platform-specific variations when needed

---

## [Feature Capabilities]
Break down the feature into logical capability areas.
Explain how each capability works and interacts with others.

---

## Edge Cases & Error Handling
| Scenario | Behavior |
|----------|----------|
| [Error condition] | [How system responds] |

---

## Data Flow
Describe how data moves through the system in **product terms**:
- What the user provides (inputs)
- What the system does with it (processing, API calls)
- What comes back (results, state changes)
- Where data is stored and for how long

Use plain language — no class names, method signatures, or file paths.

---

## [Persistence/State]
Describe what data persists across sessions, how defaults work,
and any sticky behaviors.

---

## Feature Flags
| Flag | Purpose |
|------|---------|
| [flag_name] | [What it controls] |

---

## Analytics
### Events
Event tables describe **what** to track and **when** — not which code files fire them.

| Event Name | Trigger | Additional Parameters |
|------------|---------|---------------------|
| [event.name] | [When fired] | [Key parameters with types] |

---

## Out of Scope
List what this spec does NOT cover.
Reference related features or future phases.

---

## References
Lookup table mapping spec concepts to source files (for developers only).
This is the ONLY section where file paths and code identifiers appear.

| Concept | Source |
|---------|--------|
| [Feature area] | [File path or module] |
```
