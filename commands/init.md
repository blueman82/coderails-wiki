---
title: "/coderails:init"
type: command
created: 2026-06-25
last_updated: 2026-06-26
sources: [commands/init.md, sources/pr_47_strictcode-skill-config.md]
tags: [command, scaffolding, setup, config, workflow]
---

# /coderails:init

Scaffolds `workflow.config.yaml` for the current project — the config file that every workflow command reads. This is a one-time setup step per project.

## Invocation

```
/coderails:init [project-name]
```

`project-name` is optional. If omitted, Claude derives it from `basename $(pwd)`.

## What it does

1. **Determines the config path** as `$(pwd)/.claude/workflow.config.yaml` (creates `.claude/` if needed). Run `/init` from the directory whose config you want to set — a project subdir in a monorepo, or the git root for a standalone repo. (verified: init.md:15, post-#67)
   - The *readers* then locate it by **walking up** from their cwd to the git root, first `.claude/workflow.config.yaml` found wins — so writing it at any level works for any layout. This replaced the old hardcoded `projects/<name>/` dual-path rule (PR #67). See [[config-resolution]] and [[pr_72_config-walkup-symlink-hang]].
3. **Guards against overwrite** — if the file already exists, asks the user to confirm before proceeding.
4. **Collects config values** interactively (one prompt listing all fields):
   - Jira project key, epic key, component name + ID, epic field ID, story-points field ID, fix version, start/resolve transition names, MCP namespace
   - Wiki path (relative to project dir)
   - Worktree base path (default: parent directory of git root)
   - Worktree script path
   - Strictcode paths (comma-separated globs)
   - Strictcode skill (slash-command to run, e.g. `/engineering-principles-python`, `/engineering-principles-go`, `/engineering-principles-ts`) — auto-detected from project files: `go.mod` → `/engineering-principles-go`; `package.json` + `.ts` files → `/engineering-principles-ts`; otherwise `/engineering-principles-python`. Answer "none" to disable engineering-principles entirely. (added PR #47)
5. **Writes `workflow.config.yaml`** using `null` for any field answered "none".
6. **Reports** the path written and reminds the user to commit the file.
7. **Warns about non-default Jira MCP namespaces**: if `mcp_namespace` is not `jira`, tells the user to add `"mcp__<mcp_namespace>__*"` to `.claude/settings.json` under `permissions.allow`.

## Config fields written

The full schema produced (verified from source):

```yaml
project: <name>
wiki_path: <path or null>
worktree_base: <path>
worktree_script: <path or null>
jira:
  project: <key or null>
  epic: <key or null>
  component_name: <name or null>
  component_id: "<numeric id or null>"
  epic_field: ""        # customfield_XXXXX or blank
  points_field: ""      # customfield_XXXXX or blank
  fix_version: ""       # e.g. v1.0 or blank
  mcp_namespace: "jira" # the <ns> in mcp__<ns>__* tool names
  transitions:
    start: ""           # e.g. "In Progress" or blank
    resolve: ""         # e.g. "Resolved" or blank
engineering_principles_paths:
  - "<glob>"            # or null
engineering_principles_skill: "/engineering-principles-python" # or /engineering-principles-go, /engineering-principles-ts, null (disables engineering-principles)
```

`engineering_principles_skill` was added by PR #47. Absent or null falls back to `/engineering-principles-python` at runtime (backward-compatible). See [[pr_47_strictcode-skill-config]].

## File written

`.claude/workflow.config.yaml` (path depends on repo layout — see step 2 above).

This is the single file the walk-up config resolver reads. All workflow commands (`workflow`, `prep`, `push`) and the [[enforce_pr_workflow]] hook locate it via the shared `scripts/lib/config.sh` resolver. See [[config-resolution]].

## Scripts invoked

None. This command is pure prose/interaction — no shell scripts. It writes the config file directly via the `Write` tool.

## Preconditions

- Must be run inside a git repository (`git rev-parse` must succeed).
- The user must have write access to create `.claude/workflow.config.yaml`.
- For Jira fields: the user needs to know their Jira project key, epic key, and custom field IDs in advance (available from the Jira project settings or Jira URL).

## One-time setup nature

Run once per project before using any workflow command. Re-running is safe — the command confirms before overwriting. After scaffolding, commit the config file so the workflow chain picks it up on every branch.

## Downstream dependents

Everything in the workflow chain reads this file. Without it, `NO_CONFIG` is returned and workflow commands stall. (verified from `CLAUDE.md`: "every workflow command reads `workflow.config.yaml` inline via a `!` bash substitution").

## See also

- [[config-resolution]] — the dual-path lookup that reads the file this command creates
- [[workflow]] — umbrella command that consumes the config
- [[prep]] — first workflow phase; reads `jira.*` fields
- [[push]] — reads `jira.*`, `worktree_*`, and `wiki_path`
- [[merge]] — reads `jira.transitions.resolve` and `wiki_path`
