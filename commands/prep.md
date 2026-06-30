---
title: "/coderails:prep"
type: command
created: 2026-06-25
last_updated: 2026-06-30
sources: [commands/prep.md, sources/pr_81-83_review-artifact-seam.md]
tags: [command, prep, worktree, jira, branch, git, progress-json]
---

# /coderails:prep

The first command in the workflow chain. Creates a git worktree for isolated feature work and, if Jira is configured, opens and advances a ticket to the active state.

## Invocation

```
/coderails:prep <branch> [options]
/coderails:prep feature/foo-bar
/coderails:prep bug/fix-timeout --summary "Request timeout too short"
/coderails:prep feature/add-retry-logic --type Task --summary "Add retry logic" --description "..."
```

Branch name is required and must start with `feature/`, `bug/`, or `bugfix/`. The command parses its own arguments — it does not use positional `$1`/`$2` splitting, which breaks on tokens containing `/`, `:`, or spaces. (verified: prep.md:31)

## What it does

**Part 1 — Worktree creation:**

1. Strips the `feature/`/`bug/`/`bugfix/` prefix from the branch name and derives a path: `<config.worktree_base>-<description>`.
2. If `config.worktree_script` is set, runs `<script> <derived-path> <branch-name>` from the project root. Otherwise uses plain `git worktree add <derived-path> -b <branch-name>`.
3. Reports the worktree path so the developer can point their editor at it.

This enforces the "worktree before code" rule: feature work is always isolated from main. (inferred: consistent with /workflow Phase 1 and the no-edit-on-main hook design)

**Part 2 — Jira ticket creation (skipped if `config.jira` is null):**

1. Creates a ticket in `config.jira.project` with the parsed issue type, summary, and description. Always includes `Branch: <branch>` in the description body.
2. Sets the epic link (`config.jira.epic_field`) and component fields if configured.
3. Sets story points and fix version via the Jira `update` parameter's `set` operation.
4. Advances the ticket: transitions to `config.jira.transitions.start`, then attempts `config.jira.transitions.resolve` (non-fatal if that transition is unavailable from the start state).
5. Stores the ticket key in git branch config: `git config branch.<branch>.jira-ticket <KEY>`. This is what [[push]] reads at PR creation time for auto-resolve.

If Jira creation fails after the worktree is already created, the branches are safe — Jira failure is non-fatal. (verified: prep.md:118)

## Config fields read

See [[config-resolution]] for how `workflow.config.yaml` is located at runtime.

| Field | Used for |
|---|---|
| `config.worktree_base` | Base path for the derived worktree directory |
| `config.worktree_script` | Optional custom worktree creation script |
| `config.jira.project` | Jira project key for ticket creation |
| `config.jira.epic` / `config.jira.epic_field` | Epic link (custom field ID + epic key) |
| `config.jira.component_name` / `config.jira.component_id` | Component assignment |
| `config.jira.fix_version` | Fix version to set on the ticket |
| `config.jira.points_field` | Custom field ID for story points |
| `config.jira.transitions.start` | Transition name to advance ticket to active (e.g. "In Progress") |
| `config.jira.transitions.resolve` | Transition name for resolve — attempted but non-fatal if unavailable |
| `config.jira.mcp_namespace` | Jira MCP tool namespace (default: `jira`, giving `mcp__jira__*`) |

`NO_CONFIG` minimal mode: skip all Jira steps; use the git root as `worktree_base`.

## Scripts invoked

None directly. Worktree creation uses `git worktree add` via Bash, or the custom `config.worktree_script`. No `push.sh`/`merge.sh` involvement.

## Preconditions

- On `main` (or another stable base branch) with a clean working tree
- Git remote is reachable
- If Jira steps are needed: a Jira MCP server configured for the project's namespace, or `mcp__jira__*` permitted in settings

## Chain position

First in the chain. Invoked directly by [[workflow]] (Phase 1), or standalone when the developer wants to start work without the full orchestrator.

```
/workflow  →  /prep  →  (code)  →  /push  →  /merge
              ^^^^
```

The git branch config key set here (`branch.<name>.jira-ticket`) is the handoff mechanism to [[push]] for auto-resolve. Without `/prep`, the developer must set this config key manually for Jira auto-resolve to work.

## See also

- [[workflow]] — calls /prep as Phase 1
- [[push]] — reads the jira-ticket branch config set here
- [[config-resolution]] — walk-up config resolver
- [[repo-hosting]] — git remote requirements
