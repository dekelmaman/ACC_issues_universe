# Knowledge Base Convention

> This rule applies to ALL workflows and agents in the Issues Universe.
> Agents MUST read this rule before writing any output artifacts.

## Purpose

The knowledge base persists workflow artifacts (research, ADRs, drift reports, task plans)
in the universe repo so they are version-controlled, shared across the team, and available
to future workflow runs.

## Directory Structure

```
knowledge-base/
  {project}/                    # Project name derived from git remote or working dir
    {feature}/                  # Feature name from workflow params
      research.md               # Sherlock's codebase analysis
      adr-{platform}.md         # Neo's Architecture Decision Records
      tasks-{platform}.md       # Neo's task breakdowns
      drift-log.md              # Accumulated drift check reports
      review-{platform}.md      # Rev's final compliance audit
```

## Project Name Resolution

The `{project}` folder is derived from the working directory where the workflow runs:

1. Parse the git remote: `git remote get-url origin`
2. Extract the repo name: `plangrid/build-mobile` -> `build-mobile`
3. If no git remote, use the directory basename

**Current known projects:**
| Remote | Project Name |
|--------|-------------|
| `plangrid/build-mobile` | `build-mobile` |

## Path Variable

All workflows must resolve and pass this path:

```
KB_PATH = {UNIVERSE_PATH}/knowledge-base/{project}/{feature}/
```

Where:
- `{UNIVERSE_PATH}` = `~/.rick/universes/ACC_issues_universe` (or resolved dynamically)
- `{project}` = resolved from git remote (see above)
- `{feature}` = from workflow params

## What Gets Saved

| Artifact | Written By | When | Overwrites Previous? |
|----------|-----------|------|---------------------|
| `research.md` | Sherlock (spec-research) | Phase 1 | Yes (latest research replaces old) |
| `adr-{platform}.md` | Neo (adr-forge) | Phase 3 | Yes (latest ADR replaces old) |
| `tasks-{platform}.md` | Neo (spec-implement) | Phase 4 | Yes (latest plan replaces old) |
| `drift-log.md` | Rev (spec-implement) | Phase 5 (ongoing) | No (append-only log) |
| `review-{platform}.md` | Rev (spec-review) | Phase 6 | Yes (latest review replaces old) |

## What Does NOT Get Saved

- Source specs — those live in the project repo (`pgf/feature/issues/specs/`)
- Code changes — those are in the project repo branches
- Workflow state — that's in `~/.rick/state/` (session-scoped)

## Agent Instructions

When a workflow tells you to save an artifact:

1. Resolve the knowledge-base path using the project name from git remote
2. Create the feature directory if it doesn't exist
3. Write the file to the knowledge-base path
4. ALSO write a copy to `.claude/specs/{feature}/` in the project directory
   (for fast local access during the current session)

The knowledge-base copy is the **persistent** version.
The `.claude/specs/` copy is the **working** version (may be deleted between sessions).

## Reading Previous Knowledge

Before starting a new workflow run for a feature, agents SHOULD check if
previous artifacts exist in the knowledge-base:

```
Does knowledge-base/{project}/{feature}/research.md exist?
  YES -> Read it. Prior research may save time.
         But re-verify findings — code changes between runs.
  NO  -> Fresh research needed.
```

This is especially valuable for:
- **Re-running** a workflow after a failed attempt (don't re-research)
- **Multi-platform** runs (iOS run can read Android's research + ADR)
- **Team awareness** (another developer can see what decisions were made)

## Git Workflow

Knowledge-base files are committed to the universe repo via branch + PR
(per Ground Rules). They are NOT auto-committed — the user or Rick must
explicitly push changes.

Suggested commit convention:
```
kb({project}/{feature}): {what changed}

Examples:
kb(build-mobile/quick-create): add research + android ADR
kb(build-mobile/skeleton-loader): update drift log after fix cycle
```
