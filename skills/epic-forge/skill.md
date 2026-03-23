# Epic Forge — End-to-End Feature Orchestrator

## What This Is

Epic Forge is a full-cycle feature implementation orchestrator. It takes a Jira Epic (or a spec), builds a dependency-ordered execution plan, creates Jira tickets, sets up git infrastructure, and drives parallel worktree-based implementation through to a final PR.

## When to Use

- User provides a Jira Epic key and wants it implemented end-to-end
- User wants to plan and break down a feature into tickets
- User says "forge this epic", "implement the feature", "create tickets for X"

## Invocation

This skill is available as a Claude Code global skill (`/epic-forge`). Invoke it directly:

```
/epic-forge [epic-key | resume | status]
```

| Argument | Behavior |
|----------|----------|
| *(empty)* | Interactive start — asks for Epic key or spec |
| `SCCOM-1234` | Resume or start session for that Epic |
| `resume` | Resume most recent active session |
| `status` | Show all active Epic Forge sessions |

## Phases

1. **Analyze** — Fetch Jira Epic, gather spec, ask clarifying questions, produce `spec.md`
2. **Plan** — Explore codebase, break work into dependency-layered tasks, get user approval
3. **Tickets** — Create Jira child tickets linked to the Epic, one per task
4. **Setup** — Create main feature branch (`{user-prefix}/issues/{EPIC-KEY}-{name}`)
5. **Execute** — Implement each ticket in its own worktree, in dependency order, with PRs against the feature branch
6. **Finalize** — Create final PR from feature branch to `dev`

## Key Constraints

- PRs for individual tickets target the **feature branch**, not `dev`
- Final PR to `dev` is created **only after user confirmation**
- Tasks with unmerged dependencies are **blocked** — dependency gate is strict
- State persists in `.agents/epic-forge/{epic-key}/` for resumability
- At most 2-3 implementation agents run in parallel
- **Every iOS worktree must be bootstrapped immediately after `git worktree add`** — use the `ios/worktree-bootstrap` skill. Do NOT run `make ios-build-ready`.

## Branch Prefix

Branch naming uses a per-user prefix: `{prefix}/issues/{KEY}-{short-description}`.

On first run, Epic Forge asks the user: _"What's your branch prefix? (e.g. your name or initials)"_
The answer is saved to `.agents/epic-forge/user-config.json`:
```json
{ "branchPrefix": "john" }
```
On all subsequent runs, the prefix is loaded from this file — never asked again.

## State Files

```
.agents/epic-forge/
  user-config.json   # Per-user config (branchPrefix)
  {epic-key}/
    state.json    # Progress tracker (phase, tasks, statuses)
    spec.md       # Feature understanding
    plan.md       # Dependency-ordered task list
    status.md     # Human-readable dashboard
```

## Required Tools

- `jira_getIssue`, `jira_searchIssues`, `jira_createIssue`, `jira_transitionIssue` — Jira lifecycle
- `git worktree`, `gh pr create` — Branch and PR management
- Agent tool (Task) — Dispatch implementation sub-agents
- File read/write — State persistence

## Related Skills

- **`ios/worktree-bootstrap`** — Must be applied to every iOS worktree after creation. Handles Pods, SPM packages, Gemfile, and local package symlinks without requiring Artifactory credentials or VPN.
