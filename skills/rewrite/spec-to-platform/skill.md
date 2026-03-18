# Spec to Platform — Convert agnostic UI spec to platform implementation plan

## Required Capabilities
- Read spec files (ui-spec.md, screen-composition.md)
- Read platform addendum (ios-addendum.md, android-addendum.md)
- Read existing codebase patterns
- Write implementation plan files

## Inputs
- Platform-agnostic UI spec(s) from investigate-ui
- Platform addendum (conventions, patterns, design system)
- Existing codebase for pattern matching

## Outputs
- Platform-specific implementation plan with:
  - File list (which files to create/modify)
  - View code structure per component
  - Data model changes needed (from blockers)
  - Wiring instructions (how to plug into existing app)
  - Task breakdown (ordered, dependency-aware)

## Process

### 1. Read All Specs
Read screen-composition.md + all component ui-spec.md files.
Collect all blockers across components.

### 2. Read Platform Addendum
Load the target platform's addendum for:
- View framework patterns (how views are structured)
- State management conventions
- Design system component mapping
- Dependency injection patterns
- File organization conventions

### 3. Map Spec Elements to Platform Components
For each UI spec element, determine:
- Which platform component/widget renders it
- Which design system token maps to the spec's color/typography
- How data binding works for this element

### 4. Resolve Blockers First
For each blocker from the spec:
- Define the exact model/data layer change needed
- Identify which files to modify
- These become the FIRST tasks in the plan

### 5. Generate Task List
Order tasks by dependency:
1. Data model changes (blockers)
2. Data layer changes (repository, datasource)
3. View components (leaf components first, then containers)
4. Wiring (plug into existing app)
5. Verification (build, compare with screenshot)

Each task must have:
- What to do
- Which file(s) to create/modify
- Dependencies (which tasks must complete first)
- Acceptance criteria (how to know it's done)

### 6. Write Implementation Plan
Save to `./specs/<feature>/ui/implementation-plan-<platform>.md`

## Rules
- The plan is platform-SPECIFIC — this is where framework code belongs
- Reference the agnostic spec for WHAT to build, the addendum for HOW
- Never modify the agnostic spec — it's the source of truth
- Task ordering must respect dependencies — no task references uncommitted changes
