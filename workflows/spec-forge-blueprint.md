# Spec-Forge: Spec-Driven Feature Implementation Workflow

> **Status**: DRAFT v2 - Decisions locked, ready for YAML generation
> **Date**: 2026-04-16
> **Author**: Rick + Sagi (collaborative design)

---

## 1. The Problem

The Issues Universe has workflows for UI-reverse-engineering (`specter`), epic orchestration (`epic-forge`), bug fixes (`bug-fix`), and research (`ticket-research`). But there's a critical gap:

**No workflow takes an _existing product spec_ and turns it into verified, shipped code across platforms.**

Ezra already produces specs at `pgf/feature/issues/specs/`. Nobody consumes them systematically.

---

## 2. Research: The SDD Landscape (2025-2026)

### 2.1 Frameworks Analyzed

| Framework | Approach | Phases | Verification Model | Multi-Platform | Best For |
|-----------|----------|--------|-------------------|----------------|----------|
| **GitHub Spec Kit** | Static-spec, agent-agnostic | Specify -> Plan -> Tasks -> Implement | Clarify + Analyze (pre-impl) | No (single-repo) | Greenfield, single-platform |
| **AWS Kiro** | 3-phase IDE-native | Requirements -> Design -> Tasks | EARS notation, living docs | No | AWS-centric teams |
| **OpenSpec** | Version-control-for-intent | Project -> Specs -> Changes/Proposals | Proposal approval gate | No | Brownfield, change-heavy |
| **BMAD-METHOD** | 19-agent, scale-adaptive | 4 phases, PRD-driven | Story files with embedded context | No | Enterprise, compliance |
| **cc-sdd** | Kiro-style, multi-agent | Steering -> Requirements -> Design -> Tasks | Gap/design/impl validation | Yes (agent-level) | Team workflows |
| **Tessl** | Spec-as-source (beta) | Spec -> Generated code | 1:1 spec-to-file mapping | No | Experimental/greenfield |
| **ralph-specum** | Claude Code plugin | Research -> Requirements -> Design -> Tasks -> Implement | 3-layer verification, TDD/POC workflows | No | Claude Code projects |

### 2.2 What We Steal From Each

| Framework | What We Take | Applied Where |
|-----------|-------------|---------------|
| **Spec Kit** | `constitution` concept, `clarify`/`analyze` pre-impl gates | Phase 2 research validation |
| **Kiro** | EARS notation for acceptance criteria | ADR requirement mapping |
| **OpenSpec** | Delta markers (CREATE/MODIFY/REMOVE), proposal-as-gate | ADR component plan, platform gate |
| **BMAD** | Scale-adaptive tracks, story files with full embedded context | Task format, quick mode |
| **cc-sdd** | `research.md` phase, 3 validation commands (gap/design/impl) | Phase 2, drift checks |
| **Tessl** | Specs should outlive sessions (spec-anchored approach) | Specs in pgf/ persist |
| **ralph-specum** | 3-layer verification, POC/TDD branching, VF loop, quality checkpoints | Phase 5 verify loop, task format |

### 2.3 What Nobody Does Well

1. **Multi-platform from one spec**: Every tool assumes one codebase, one platform.
2. **Spec-vs-implementation diff**: Tools verify "does it compile" but not "does it match the spec."
3. **ADR generation per platform**: No tool generates per-platform architecture decisions from a shared spec.
4. **Correction loop with spec as oracle**: Nobody systematically detects drift and generates targeted fix tasks.

---

## 3. Decisions (Locked)

### Decision 1: Drift Check Frequency — Every 3 Tasks (Default)

#### Research-Backed Analysis

The frameworks we studied fall into 4 verification strategies. Here's how each performs and why "every 3" wins for our use case:

**Strategy A: Every Task (ralph-specum Layer 3 "periodic" mode)**
Ralph-specum runs artifact review on phase boundaries, every 5th task, and the final task. That's close to "every task" in small specs. Their own data shows this catches contradictions early (Layer 1 catches ~15% of false completions), but Layer 3 artifact review is expensive — it invokes a full spec-reviewer agent with design.md + requirements.md cross-referencing.

- **Cost**: ~30s per invocation (sub-agent spin-up + file reads + analysis). A 15-task spec adds ~5 extra minutes.
- **Benefit**: Maximum accuracy. Drift never exceeds 1 task worth of work before correction.
- **When it makes sense**: Mission-critical specs, compliance-driven work (BMAD's "Enterprise Track" equivalent).
- **When it's overkill**: Most of our Android/iOS feature work. A skeleton loader doesn't need compliance-grade verification.

**Strategy B: Every 3 Tasks (cc-sdd `validate-impl` cadence)**
cc-sdd runs its `validate-impl` command at phase gates but recommends "validation waves" every 3-5 tasks for active development. This is also close to ralph-specum's "every 5th task" cadence but slightly more aggressive.

The key insight from cc-sdd: **3 tasks is roughly one logical unit of work** (create component, wire it up, test it). If drift happens, it happened within a coherent unit, so the fix task is also coherent — not a sprawling multi-concern patch.

- **Cost**: ~2 minutes overhead per 15-task spec (5 drift checks).
- **Benefit**: Catches drift before it compounds into multi-task fixes. Correction tasks are small and focused.
- **Sweet spot**: Our typical Android Mobius/Compose tasks are 2-3 files each. After 3 tasks, you've touched ~6-9 files — enough to drift, small enough to course-correct cheaply.

**Strategy C: Per Phase Only (Kiro's model)**
Kiro validates at phase boundaries (Requirements -> Design -> Tasks). No mid-implementation checks. Martin Fowler's analysis notes this works for greenfield where the spec is fresh and the agent starts from scratch, but fails in brownfield where existing code creates unexpected friction.

- **Cost**: Minimal (~1-2 drift checks per spec).
- **Benefit**: Fast execution, minimal interruptions.
- **Risk**: A Phase 1 with 20 tasks could drift significantly by task 15, requiring a large correction batch at phase end. The "big bang fix" problem — same issue as waterfall testing.

**Strategy D: Smart/Complexity-Based (Tessl's aspiration)**
Tessl's 1:1 spec-to-code mapping means every file change is validated against its spec automatically. This is ideal but requires the tight coupling that only works for generated code. For hand-guided implementation (Trinity writing real code), we'd need heuristics like "check after any task that creates a new file" or "check after tasks touching >3 files." Fragile, hard to debug, and the heuristics themselves would drift.

#### Our Configuration

```yaml
drift_frequency:
  default: "every-3"      # Standard: check after every 3rd task
  quick: "per-phase"      # Quick mode: check at phase boundaries only
  strict: "every-task"    # Strict mode: check after every task
```

**Adaptive override**: If a drift check returns `DRIFT_MAJOR`, temporarily escalate to "every-task" for the next 3 tasks, then relax back to "every-3". This is stolen from ralph-specum's recovery mode concept — when things go wrong, increase monitoring.

---

### Decision 2: One ADR Per Required Platform (Smart Detection)

ADRs are generated **only for platforms the feature actually needs**. Not every feature touches all 3 layers.

#### Platform Detection Logic

Neo reads the spec + research.md and determines which platforms are required:

```
Neo reads spec requirements:
  - "Add skeleton loader to Android issue type screen" 
  - Sherlock found: IssueTypeTemplateScreen.kt (Android), no iOS equivalent in scope

Neo's platform analysis:
  - Android: REQUIRED (spec explicitly targets Android screen)
  - PGF: CHECK (does this feature need shared models/use-cases?)
    - Sherlock: "Loading state already exists in IssueTypeTemplateModel.kt in pgf/"
    - Result: NOT REQUIRED (pgf layer already handles it)
  - iOS: NOT REQUIRED (not in spec scope)

Platforms for ADR generation: [android]
```

**But when the user says "build quick-create for Android":**
```
Neo reads spec:
  - Quick Create requires new use-cases, new models, server API integration
  - Sherlock: "Use-case layer lives in pgf/feature/issues/usecases/"
  - Sherlock: "API models live in pgf/feature/issues/models/"

Platforms for ADR generation: [android, pgf]
  - pgf ADR: shared use-cases, models, API contracts
  - android ADR: Compose UI, Mobius integration, AlloyUI components
```

#### ADR Structure (per platform)

Each ADR references the shared `research.md` but is self-contained for its platform:

```
.claude/specs/{feature}/
  research.md           # Shared (Sherlock output)
  adr-android.md        # Android-specific decisions
  adr-pgf.md            # PGFoundation shared layer decisions  
  tasks-android.md      # Android implementation tasks
  tasks-pgf.md          # PGF implementation tasks (executed FIRST — dependency)
```

**Execution order**: PGF tasks always execute before platform tasks (foundation before features).

---

### Decision 3: Independent from Specter — Ezra Finds the Spec

spec-forge is a standalone workflow. It does NOT compose with specter at the workflow level. Instead, **Phase 0 uses Ezra to locate the spec**.

#### How Ezra Finds Specs

Ezra already writes specs to `pgf/feature/issues/specs/{feature-folder}/`. Phase 0 delegates to Ezra to find the right spec:

```
User: /rick run spec-forge --feature="quick-create"

Phase 0 (Ezra):
  1. Search pgf/feature/issues/specs/ for folders matching "quick-create"
  2. Found: pgf/feature/issues/specs/quick-create/quick-create-spec.md
  3. Read and validate spec completeness (all template sections present)
  4. Return spec path + summary

Output:
  spec_path: pgf/feature/issues/specs/quick-create/quick-create-spec.md
  spec_summary: "Quick Create allows users to create issues with minimal input..."
  completeness: PASS (all sections present)
```

**If no spec found**: Ezra reports "No spec found for '{feature}'. Run specter or create one manually." Workflow stops.

**If spec is incomplete**: Ezra flags missing sections. Rick asks user: "Spec is missing [Analytics, Edge Cases]. Continue anyway or fix first?"

#### Why Independent

- Specter produces specs from screenshots (reverse-engineering existing UI)
- Ezra produces specs from PRDs/requirements (forward-engineering new features)
- spec-forge consumes specs regardless of origin
- A future `full-feature` workflow can compose: `specter -> spec-forge` or `ezra-create -> spec-forge`

---

### Decision 4: Platform-Specific Skill Routing

Trinity and Neo load platform-appropriate skills from both the Universe and the project.

#### Universe Skills (in `~/.rick/universes/ACC_issues_universe/skills/`)

| Skill | Platform | Used By |
|-------|----------|---------|
| `tca/composable-architecture` | iOS | Neo (planning), Trinity (impl), Rev (review) |
| `tca/dependencies` | iOS | Neo, Trinity |
| `tca/case-paths` | iOS | Neo, Trinity |
| `tca/identified-collections` | iOS | Neo, Trinity |
| `tca/navigation` | iOS | Neo, Trinity |
| `tca/tca-testing` | iOS | Trinity (tests) |
| `swiftui/modern-patterns` | iOS | Trinity, Rev |
| `rewrite/spec-to-platform` | Both | Neo (ADR generation) |
| `rewrite/investigate-ui` | Both | Phase 2 research |
| `testing/tdd` | Both | Task planning (TDD workflow) |

#### Project Skills (in `.claude/skills/`)

| Skill | Platform | Used By |
|-------|----------|---------|
| `mobius-mvi` | Android | Neo (planning), Trinity (impl) |
| `jetpack-compose` | Android | Neo, Trinity |
| `AlloyUI` | iOS (SwiftUI) | Trinity, Alloy Auditor |
| `tca-reducer` | iOS | Neo, Trinity |
| `localization` | Both | Trinity (string resources) |
| `maestro` | Both | Test planning |

#### Routing Table

When Trinity starts a task, the workflow injects a skill preamble:

```
Platform: Android
  -> Load: mobius-mvi, jetpack-compose, localization
  -> Trinity prompt includes: "Follow Mobius/Compose patterns from loaded skills"

Platform: iOS  
  -> Load: tca-reducer, AlloyUI, tca/composable-architecture, swiftui/modern-patterns, localization
  -> Trinity prompt includes: "Follow TCA/SwiftUI patterns from loaded skills"

Platform: PGF (shared)
  -> Load: mobius-mvi (for shared model patterns), localization
  -> Trinity prompt includes: "Write platform-agnostic Kotlin. No UI code."
```

---

### Decision 5: Specs Live in the Project (`pgf/feature/issues/specs/`)

This is already where Ezra writes them. Confirmed structure:

```
pgf/feature/issues/specs/
  quick-create/
    quick-create-spec.md          # Ezra's output (committed, shared)
  filter-by-company/
    filter-by-company-spec.md     
  details/
    permittedStatuses/
      spec.md
```

**Ephemeral work artifacts** (research, ADRs, tasks, drift reports) go to `.claude/specs/{feature}/` — NOT committed:

```
.claude/specs/quick-create/
  research.md                # Sherlock's codebase analysis
  adr-android.md             # Neo's Android architecture decisions
  adr-pgf.md                 # Neo's PGF architecture decisions
  tasks-android.md           # Neo's Android task breakdown
  tasks-pgf.md               # Neo's PGF task breakdown
  drift-report-001.md        # Rev's drift check after tasks 1-3
  drift-report-002.md        # Rev's drift check after tasks 4-6
```

**Why this split**:
- Specs are product documents → shared across team → committed in pgf/
- Work artifacts are session-specific → only useful during implementation → .claude/
- This mirrors Tessl's spec-anchored philosophy: the spec outlives the implementation session

---

### Decision 6: Issues Reviewer (Rev) Upgrade Plan

Rev currently reviews iOS code only (TCA patterns, AlloyUI, SwiftUI, memory leaks). For spec-forge, Rev needs two new capabilities:

#### 6a. New Capability: Spec Drift Detection

**Current rules.md** (what Rev does today):
```
- Check for: AlloyUI compliance, TCA patterns, memory leaks, async safety, naming
- Flag UIColor/UIKit usage in domain/model layers
- Flag reducers exceeding 300 lines
- Flag missing @Dependency usage
- Flag force unwraps in production code
```

**New rules to add** (spec-forge mode):
```markdown
## Spec Drift Detection Mode

When invoked with `mode: spec-drift`, Rev operates differently from standard code review.

### Inputs Required
- `spec_path`: Path to the product spec (from pgf/feature/issues/specs/)
- `adr_path`: Path to the platform ADR (from .claude/specs/{feature}/)
- `diff_range`: Git diff range showing code changes since last drift check
- `tasks_completed`: List of task IDs completed since last check

### Drift Detection Process

1. **Parse spec requirements**: Extract all behavioral requirements from the spec
   - User journeys (happy path, error recovery, interrupted flow)
   - Visual states table
   - Component specifications
   - Edge cases table
   - Data flow description
   - Analytics events

2. **Parse ADR requirement mapping**: For each spec requirement, find the ADR's
   platform-specific implementation decision
   - The ADR's "Requirement Mapping" table is the oracle
   - Each row maps: Spec Requirement -> Platform Implementation -> EARS Acceptance Criteria

3. **Compare against actual code changes**: For each completed task:
   - Read the task's expected files and changes
   - Read the actual git diff for those files
   - Score: does the code match the ADR's implementation decision?

4. **Generate Spec Compliance Matrix**:
   | Spec Req | ADR Decision | Code Status | Drift Level | Evidence |
   |----------|-------------|-------------|-------------|----------|
   | SR-1     | ShimmerEffect when loading | Implemented correctly | NONE | file.kt:42 |
   | SR-2     | Mirror dimensions | 2dp off specification | MINOR | file.kt:78 |
   | SR-3     | Crossfade 300ms | Not implemented (task scope) | PENDING | N/A |
   | SR-4     | Error fallback | Different pattern used | MAJOR | file.kt:95 |

5. **Drift Level Classification**:
   - `NONE`: Implementation matches ADR decision exactly
   - `PENDING`: Requirement not yet in scope of completed tasks (not drift, just not reached)
   - `MINOR`: Implementation is close but has small divergences (wrong spacing, slightly
     different naming, non-functional differences). Log warning, continue.
   - `MAJOR`: Implementation contradicts ADR decision OR misses a requirement that WAS in
     scope of completed tasks. STOP. Generate correction tasks.

### Output Format

```text
## Drift Check Report #{N}
Tasks checked: [list of task IDs]
Git range: {start_sha}..{end_sha}

### Spec Compliance Matrix
[table as above]

### Verdict: DRIFT_NONE | DRIFT_MINOR | DRIFT_MAJOR

### If DRIFT_MAJOR — Correction Tasks:
- [ ] FIX-{N}: {description}
  - **Do**: {specific fix instructions}
  - **Files**: {files to modify}
  - **Spec**: {which requirement was violated}
  - **ADR**: {which ADR decision was not followed}
```

### Critical Rules for Drift Detection
- PENDING is NOT drift. Don't flag requirements that haven't been reached yet.
- MINOR drift is acceptable. Log it but don't block.
- Only MAJOR triggers correction. Be conservative — false positives waste time.
- Always cite evidence: file path + line number + what was expected vs what exists.
- Correction tasks must be specific enough for Trinity to execute without ambiguity.
```

#### 6b. New Capability: Android Code Review

Rev currently only reviews iOS code (TCA, SwiftUI, AlloyUI). For spec-forge to work on Android, Rev needs Android review rules:

```markdown
## Android Review Mode

When reviewing Android/Kotlin code, check for:

### Mobius/MVI Patterns
- Updater functions are pure (no side effects)
- Effects are handled by Processors, not Updaters
- Model changes go through proper Next.Builder patterns
- ViewModel exposes renderDataStream correctly

### Compose Patterns
- State hoisting (no internal mutableStateOf in shared components)
- Proper Modifier chain ordering (clickable before padding, etc.)
- LazyColumn/LazyRow key usage for stable recomposition
- Preview annotations present for all public composables
- No side effects in composable functions (use LaunchedEffect/SideEffect)

### AlloyCompose Compliance
- Using AlloyCompose components where available (not raw Material3)
- Design tokens from AlloyTheme (not hardcoded colors/dimensions)
- Accessibility: contentDescription on interactive elements

### General Kotlin
- No force casts (as!) in production code
- Sealed class/interface for state modeling
- Coroutine scope management (no GlobalScope)
- Null safety: avoid !! operator
```

#### 6c. Updated tools.md

Rev's current `tools.md`:
```
allowed: Read, Glob, Grep
model: sonnet
max-turns: 30
skills: [tca/composable-architecture, tca/dependencies, tca/case-paths, tca/identified-collections, swiftui/modern-patterns]
```

Proposed update:
```
allowed: Read, Glob, Grep, Bash
model: sonnet
max-turns: 40
skills: [tca/composable-architecture, tca/dependencies, tca/case-paths, tca/identified-collections, swiftui/modern-patterns]
```

Changes:
- Add `Bash` (needed for `git diff` in drift detection mode)
- Increase `max-turns` from 30 to 40 (drift detection needs more turns to read spec + ADR + diff)
- Android/Compose skills are project-level (mobius-mvi, jetpack-compose) — loaded at runtime, not in tools.md

---

## 4. Final Workflow Design

### 4.1 Flow Diagram

```
[Phase 0: FETCH SPEC]   Ezra locates spec in pgf/feature/issues/specs/
     |
     v
[Phase 1: RESEARCH]     Sherlock explores codebase for patterns, risks, commands
     |
     v
[Phase 2: PLATFORM]     Rick asks user: which platform(s)? Neo detects if pgf needed.
     |
     v
[Phase 3: ADR]          Neo generates ADR per required platform (pgf first if needed)
     |
     v  (user approval gate)
[Phase 4: TASK PLAN]    Neo breaks each ADR into ordered tasks
     |
     v
[Phase 5: IMPLEMENT]    Trinity executes tasks with drift check every 3 tasks
     |                   
     |---[DRIFT_MAJOR]---> Rev generates FIX tasks -> Trinity fixes -> re-check
     |       |              (max 3 retries, then escalate to user)
     |       v
     |   [DRIFT_NONE/MINOR] -> continue
     |<------+
     v
[Phase 6: FINAL REVIEW] Rev full audit against spec + Alloy Auditor for AlloyUI
     |
     v
[Phase 7: PR + JIRA]    Clippy creates PR, Shimi-T updates ticket (if provided)
```

### 4.2 Phase Details

---

#### Phase 0: FETCH SPEC (Ezra)

**Agent**: Ezra (Product Spec Creator)
**Purpose**: Locate and validate the product spec from the project.

```yaml
- id: fetch-spec
  agent: ezra
  description: "Locate the feature spec in pgf/feature/issues/specs/"
  prompt: |
    Find the product spec for feature "{{feature}}" in the project.

    Search path: pgf/feature/issues/specs/
    Known spec structure: {feature-folder}/{feature-name}-spec.md

    Steps:
    1. Search for folders matching "{{feature}}" (fuzzy — "quick-create" matches "quick-create/")
    2. If found, read the spec file and validate completeness:
       - All template sections present (Overview, User Journeys, User Flow, Screen Layout,
         Visual States, Components, Edge Cases, Data Flow, Analytics, References)
       - No empty sections (except "Out of Scope" which may legitimately be empty)
    3. Extract a structured summary:
       - Feature name
       - Key requirements (bullet list)
       - Visual states defined
       - Analytics events listed
       - Any platform-specific notes

    Output format:
    ```
    SPEC_FOUND: [path]
    SPEC_COMPLETENESS: PASS | PARTIAL (missing: [sections])
    SPEC_SUMMARY:
      Feature: [name]
      Requirements: [count]
      Visual States: [list]
      Analytics Events: [count]
      Platform Notes: [any platform-specific mentions]
    ```

    If no spec found, output:
    ```
    SPEC_NOT_FOUND: No spec matching "{{feature}}" in pgf/feature/issues/specs/
    Available specs: [list folder names]
    ```
  auto_continue: true
```

**Example**:
```
User: /rick run spec-forge --feature="skeleton-loader"

Ezra (Product Manager): 
SPEC_NOT_FOUND: No spec matching "skeleton-loader" in pgf/feature/issues/specs/
Available specs: quick-create, filter-by-company, details/permittedStatuses

Rick: No spec found. Want me to ask Ezra to create one, or provide a path manually?
```

```
User: /rick run spec-forge --feature="quick-create"

Ezra (Product Manager):
SPEC_FOUND: pgf/feature/issues/specs/quick-create/quick-create-spec.md
SPEC_COMPLETENESS: PASS
SPEC_SUMMARY:
  Feature: Quick Create Issue
  Requirements: 14
  Visual States: 6 (default, loading, AI-enriching, success, error, disabled)
  Analytics Events: 8
  Platform Notes: None (platform-agnostic spec)
```

---

#### Phase 1: RESEARCH (Sherlock)

**Agent**: Sherlock (Codebase Detective)
**Purpose**: Understand what exists in the codebase before designing anything.

```yaml
- id: research
  agent: sherlock
  collaborators: [neo]
  description: "Deep codebase research for the feature — Sherlock investigates, Neo provides architectural context"
  prompt: |
    Read the feature spec at {{spec_path}}.
    
    Research the codebase to answer:
    1. What existing code overlaps with this feature?
    2. What components/modules can be reused?
    3. What are the build/test/lint commands for each relevant module?
    4. What risks exist? (legacy paths, recent migrations, known issues)
    5. Does this feature require changes in pgf/ (shared layer)?
    
    You have a collaborator: **Neo** (Software Architect).
    Delegate architectural questions to Neo: "Is this the right layer for X?"
    Neo helps determine if pgf changes are needed vs platform-only.

    Save output to .claude/specs/{{feature}}/research.md
  depends_on: [fetch-spec]
  auto_continue: true
```

---

#### Phase 2: PLATFORM SELECTION (Rick — Interactive)

**Agent**: Rick (orchestrator, not a sub-agent)
**Purpose**: User chooses platform(s). Neo determines if pgf is needed.

```yaml
- id: platform-select
  agent: rick
  description: "Ask user which platform(s), then Neo determines if pgf layer is also needed"
  prompt: |
    Present the spec summary from Phase 0 and research findings from Phase 1.
    
    Ask the user:
    "Which platform should I forge this for?"
    1. Android
    2. iOS
    3. Both (sequential — Android first, then iOS)
    
    After user picks, consult Neo:
    "Given the spec requirements and research, does this feature need pgf/ changes
    (shared models, use-cases, API contracts) or is it platform-only?"
    
    Output: platforms list (e.g., ["pgf", "android"] or ["ios"])
    
    Rule: If pgf is needed, it ALWAYS comes first in the execution order.
  depends_on: [research]
  auto_continue: false  # HUMAN GATE — waits for user input
```

---

#### Phase 3: ADR GENERATION (Neo, per platform)

**Agent**: Neo (Software Architect)
**Purpose**: For each required platform, generate an Architecture Decision Record.

```yaml
- id: generate-adr
  agent: neo
  collaborators: [sherlock]
  description: "Generate ADR for each required platform"
  prompt: |
    Read:
    - Feature spec at {{spec_path}}
    - Research at .claude/specs/{{feature}}/research.md
    - Selected platforms: {{platforms}}

    For EACH platform in {{platforms}}, generate an ADR at .claude/specs/{{feature}}/adr-{{platform}}.md

    Each ADR must contain:
    1. Context — what the spec requires
    2. Decisions — which platform patterns to use, with rationale
    3. Alternatives Considered — table with pros/cons/decision
    4. Requirement Mapping — table mapping EVERY spec requirement to:
       - Platform-specific implementation approach
       - EARS acceptance criteria (WHEN/WHERE/WHILE + SHALL/SHOULD)
    5. Component Plan — using delta markers:
       - CREATE: new files to create
       - MODIFY: existing files to change (cite file path from Sherlock)
       - No DELETE unless explicitly needed
    6. Dependencies — what must be built first (pgf before platform)
    
    Platform skill loading:
    - Android: load mobius-mvi, jetpack-compose skills
    - iOS: load tca/composable-architecture, swiftui/modern-patterns, AlloyUI skills
    - PGF: load mobius-mvi for shared model patterns
    
    Delegate to Sherlock for any "does X exist?" or "where is Y?" questions.
  depends_on: [platform-select]
  auto_continue: false  # USER APPROVAL GATE — user reviews ADR before implementation
```

---

#### Phase 4: TASK PLANNING (Neo)

**Agent**: Neo (Software Architect)
**Purpose**: Break each ADR into ordered implementation tasks.

```yaml
- id: plan-tasks
  agent: neo
  description: "Generate task breakdown from ADRs"
  prompt: |
    For EACH platform ADR in .claude/specs/{{feature}}/:
    
    Generate tasks-{{platform}}.md following ralph-specum's task format:
    - Each task: Do / Files / Done when / Verify / Commit / Spec trace
    - Quality checkpoints every 2-3 tasks ([VERIFY] tagged)
    - Choose workflow: POC-first (greenfield) or TDD (extending existing)
    - Tasks trace back to spec requirements: _Spec: SR-1, ADR: Component Plan item 3_
    
    Execution order rules:
    - pgf tasks execute BEFORE platform tasks (foundation first)
    - Within a platform: dependencies before dependents
    - Mark independent tasks with [P] for potential parallelism
    
    Insert a DRIFT CHECK marker after every 3rd non-[VERIFY] task:
    ```
    - [ ] DC-1 [DRIFT-CHECK] Spec compliance check after tasks 1.1-1.3
      - **Scope**: Tasks 1.1, 1.2, 1.3
      - **Spec**: {{spec_path}}
      - **ADR**: .claude/specs/{{feature}}/adr-{{platform}}.md
    ```
    
    Save to .claude/specs/{{feature}}/tasks-{{platform}}.md
  depends_on: [generate-adr]
  auto_continue: true
```

---

#### Phase 5: IMPLEMENT + VERIFY LOOP (Trinity + Rev)

**Agent**: Trinity (Implementor) + Rev (Issues Reviewer for drift checks)
**Purpose**: Execute tasks with continuous spec drift detection.

```yaml
- id: implement
  agent: trinity
  collaborators: [sherlock, issues-reviewer]
  description: "Execute tasks with drift checking every 3 tasks"
  prompt: |
    Execute tasks from .claude/specs/{{feature}}/tasks-{{platform}}.md
    
    For each platform (pgf first if present, then target platform):
    
    IMPLEMENTATION LOOP:
    1. Read next task from tasks file
    2. If task is [DRIFT-CHECK]:
       - Delegate to Issues Reviewer (Rev) in spec-drift mode:
         - spec_path: {{spec_path}}
         - adr_path: .claude/specs/{{feature}}/adr-{{platform}}.md
         - diff_range: git diff from last drift check (or branch start)
         - tasks_completed: list since last check
       - On DRIFT_NONE or DRIFT_MINOR: continue
       - On DRIFT_MAJOR:
         a. Read Rev's correction tasks
         b. Execute each FIX task
         c. Re-run drift check (max 3 retries)
         d. If still DRIFT_MAJOR after 3 retries: STOP, escalate to user
    3. If task is [VERIFY]: run verification command, retry on failure (max 3)
    4. If task is regular: execute it, commit, mark [x]
    5. Repeat until all tasks complete
    
    Skill loading per platform:
    - Android: mobius-mvi, jetpack-compose, localization
    - iOS: tca-reducer, AlloyUI, tca/composable-architecture, swiftui/modern-patterns, localization
    - PGF: mobius-mvi, localization
    
    Delegate to Sherlock when you need to understand existing code before modifying it.
  depends_on: [plan-tasks]
  auto_continue: false
```

**The Drift Check in Detail**:

```
Tasks 1.1, 1.2, 1.3 completed
     |
     v
DC-1 [DRIFT-CHECK] triggered
     |
     v
Rev reads: spec + ADR + git diff $LAST_CHECK..HEAD
     |
     v
Rev generates Spec Compliance Matrix:
  SR-1: NONE (matches perfectly)
  SR-2: MINOR (2dp off, logged)
  SR-3: PENDING (not in scope yet)
  SR-4: MAJOR (wrong pattern used!)
     |
     v
Verdict: DRIFT_MAJOR
     |
     v
Rev generates: FIX-1 task with specific instructions
     |
     v
Trinity executes FIX-1 -> commits
     |
     v
Rev re-checks: SR-4 now NONE
     |
     v
Verdict: DRIFT_NONE -> continue to tasks 1.4, 1.5, 1.6
```

**Adaptive frequency**: If a drift check returns DRIFT_MAJOR, the next interval shrinks to every 1 task for the next 3 tasks. Then relaxes back to every 3. This prevents cascading drift after a correction.

---

#### Phase 6: FINAL REVIEW (Rev + Alloy Auditor)

```yaml
- id: final-review
  agent: issues-reviewer
  collaborators: [alloy-auditor]
  description: "Full spec compliance audit + AlloyUI audit"
  prompt: |
    Final review of {{feature}} implementation.
    
    Read:
    - Original spec: {{spec_path}}
    - All ADRs: .claude/specs/{{feature}}/adr-*.md
    - Full git diff: git diff {{base_branch}}...HEAD
    
    Generate FINAL Spec Compliance Matrix covering ALL requirements.
    Every spec requirement must be NONE or MINOR. Any MAJOR = FAIL.
    
    Delegate to Alloy Auditor for AlloyUI/AlloyCompose compliance check.
    
    Output: SPEC_FORGE_PASS or SPEC_FORGE_FAIL with findings.
    If FAIL: generate remaining fix tasks, loop back to Phase 5.
  depends_on: [implement]
  auto_continue: true
```

---

#### Phase 7: PR + JIRA (Clippy + Shimi-T)

```yaml
- id: ship
  agent: clippy
  collaborators: [shimi-t]
  description: "Create PR with spec compliance matrix, update Jira"
  skip_if: "{{#unless ticket_key}}true{{/unless}}"
  prompt: |
    Create PR for the {{feature}} implementation.
    
    PR body must include:
    1. Summary: what was implemented
    2. Spec Compliance Matrix from Phase 6 (proving every requirement is met)
    3. ADR decisions summary (key architecture choices)
    4. Test plan
    
    If ticket_key provided:
    - Delegate to Shimi-T: transition {{ticket_key}} to "In Review"
    - Post comment with PR link and compliance summary
  depends_on: [final-review]
  auto_continue: false
```

---

## 5. Workflow Parameters (Final)

```yaml
name: spec-forge
description: "Implement a feature from a product spec with continuous verification"

params:
  feature:
    description: "Feature name to find spec (e.g., 'quick-create', 'filter-by-company')"
    required: true
  platform:
    description: "Target platform: android, ios, or 'ask' for interactive selection"
    default: "ask"
  ticket_key:
    description: "Jira ticket key (optional — enables Jira transitions)"
    required: false
  drift_frequency:
    description: "Drift check cadence: 'every-task', 'every-3', 'per-phase'"
    default: "every-3"
  quick:
    description: "Skip interactive confirmations (platform, ADR approval)"
    default: false
```

---

## 6. Implementation Roadmap

### V1 (MVP) — Single Platform
- Phase 0: Ezra fetches spec
- Phase 1: Sherlock researches
- Phase 2: User picks ONE platform (no multi-platform)
- Phase 3: Neo generates ONE ADR (+ pgf if needed)
- Phase 4: Neo generates tasks
- Phase 5: Trinity implements with drift checks every 3 tasks
- Phase 6: Rev reviews (with new spec-drift rules)
- No Jira, no PR automation

### V2 — Full Feature
- Multi-platform support (sequential: pgf -> android -> ios)
- Jira integration (Shimi-T)
- PR creation (Clippy)
- Adaptive drift frequency

### V3 — Advanced
- Parallel worktree execution for multi-platform
- Smart drift frequency (complexity-based)
- Composition with specter workflow

---

## 7. Files to Create/Modify for V1

| Action | File | Description |
|--------|------|-------------|
| CREATE | `workflows/spec-forge.yaml` | The workflow definition |
| MODIFY | `agents/issues-reviewer/rules.md` | Add spec-drift detection rules + Android review rules |
| MODIFY | `agents/issues-reviewer/tools.md` | Add Bash, increase max-turns |
| CREATE | `skills/spec-forge/skill.md` | Skill definition for spec-forge conventions |

---

## Sources

- [GitHub Spec Kit](https://github.com/github/spec-kit)
- [GitHub Blog: Spec-Driven Development with AI](https://github.blog/ai-and-ml/generative-ai/spec-driven-development-with-ai-get-started-with-a-new-open-source-toolkit/)
- [Augment Code: 6 Best SDD Tools for 2026](https://www.augmentcode.com/tools/best-spec-driven-development-tools)
- [Martin Fowler: Understanding SDD - Kiro, spec-kit, and Tessl](https://martinfowler.com/articles/exploring-gen-ai/sdd-3-tools.html)
- [AWS Kiro Docs: Specs](https://kiro.dev/docs/specs/)
- [Smart Ralph (ralph-specum)](https://github.com/tzachbon/smart-ralph)
- [cc-sdd by gotalab](https://github.com/gotalab/cc-sdd)
- [BMAD-METHOD](https://github.com/bmad-code-org/BMAD-METHOD)
- [OpenSpec Framework](https://gist.github.com/Darkflib/c7f25b41054a04a5835052e5a21cdf82)
- [Tessl: Spec-Driven Development](https://docs.tessl.io/use/spec-driven-development-with-tessl)
- [Spec-Driven Development is Eating Software Engineering (30+ frameworks map)](https://medium.com/@visrow/spec-driven-development-is-eating-software-engineering-a-map-of-30-agentic-coding-frameworks-6ac0b5e2b484)
- [Agentic Workflows in 2026: Ultimate Guide](https://www.vellum.ai/blog/agentic-workflows-emerging-architectures-and-design-patterns)
