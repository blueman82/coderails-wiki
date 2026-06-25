---
title: Enforcement Model
type: design
created: 2026-05-30
last_updated: 2026-06-25
sources:
  - commands/workflow.md
  - CLAUDE.md
  - hooks/hooks.json
tags:
  - hooks
  - enforcement
  - design-law
  - slash-commands
---

# Enforcement Model

The most important design law in coderails. Get it wrong and you ship a system that *looks* like it enforces things but doesn't.

## The Law

**Hooks are mechanical enforcement. Slash commands are advisory.**

From `CLAUDE.md` lines 42–49 (verified):

> **Hooks = mechanical enforcement.** They run automatically on lifecycle events and can *block* (exit 2 / `permissionDecision: deny`). Use a hook when behaviour must be enforced regardless of whether Claude cooperates.
> **Slash commands = advisory.** Claude has to *choose* to invoke them. Use a command to encode a workflow, not to enforce one.
>
> If you're asked to "make X mandatory," that belongs in a `PreToolUse` hook, not a command.

`commands/workflow.md` lines 184–186 states this again as a design negative (verified):

> **Not enforcement.** Slash commands are advisory — Claude has to choose to invoke them. Mechanical enforcement (refusing `gh pr create` unless `/push` ran, refusing `gh pr merge` unless `/pr-review-toolkit:review-pr` ran) belongs in `PreToolUse` hooks, not here. See the companion enforce-pr-workflow.sh hook design.

## Why This Matters

A slash command that says "you must do X before Y" can be ignored. There is nothing stopping Claude — or a user — from running `gh pr merge` directly, bypassing `/coderails:push` entirely. The command encodes the *happy path* workflow. It is not a gate.

A `PreToolUse` hook on `Bash(gh pr merge*)` fires whether or not any slash command was involved. Claude cannot skip it. The user cannot skip it. That is the distinction.

## How Hooks Block

Stop hooks (fired after a response is generated) block by calling `exit 2` with a message on stderr. The harness then shows the message and forces a re-generate. Example: `check_confidence_labels.sh` lines 65–66 (verified):

```bash
echo "[discipline-block] response made substantive claims without (verified)/(inferred)/(guess) labels. Add them before stopping." >&2
exit 2
```

PreToolUse hooks block by emitting a JSON response with `permissionDecision: "deny"`. Example: `destructive_bash_gate.sh` uses this pattern (inferred from CLAUDE.md:68 and hook conventions).

## Current Hook Map

| Event | Script | Mode |
|---|---|---|
| `SessionStart` | `inject_bootstrap.sh` | silent — bootstraps session with `coderails:using-coderails` context |
| `UserPromptSubmit` | `inject_context.sh` | silent — prepends `[ctx]` (cwd, branch, date) |
| `UserPromptSubmit` | `discipline_catchup.sh` | warn |
| `Stop` | `check_confidence_labels.sh` | **block** — ≥200-char response with no confidence label |
| `Stop` | `check_verify_loop.sh` | **block** — any untagged DNV bullet |
| `Stop` | `loop_state_guard.sh` | **block** — agentic-loop active but progress.json absent/mismatched |
| `Stop` | `loop_stall_guard.sh` | **block** — agentic-loop active + incomplete + no LOOP-STOP declaration |
| `PreToolUse` (Bash) | `destructive_bash_gate.sh` | **block** |
| `PreToolUse` (Bash) | `test_gate.sh` | **block** on `git commit` if tests fail — opt-in only |
| `PreToolUse` (Bash) | `enforce_pr_workflow.sh` | **block** — `gh pr create` without prior `/push`, `gh pr merge` without prior `/review-pr` |
| `PreToolUse` (Write\|Edit\|MultiEdit) | `no_edit_on_main.sh` | **block** — code-file edits on main/master |

`discipline_catchup.sh` is the only surviving warn-mode hook. Everything else that should be enforced has been promoted to block-mode or moved to a PreToolUse gate. See [[discipline-loop]] for the history of why warn-mode was abandoned. (updated 2026-06-25 — added inject_bootstrap, enforce_pr_workflow, no_edit_on_main, loop-state hooks)

## When to Use Which

| Goal | Mechanism |
|---|---|
| Encode a multi-step workflow | Slash command (`commands/*.md`) |
| Prevent a specific tool call unless conditions met | `PreToolUse` hook |
| Enforce a constraint on every response | `Stop` hook |
| Share reusable workflow logic across commands | `scripts/lib/git-common.sh` |

If someone asks "can we make the strictcode check mandatory before push?", the answer is: add a `PreToolUse` hook that fires on `Bash(gh pr create*)` and checks for strictcode evidence, not a new instruction in `/push`. The command already runs it; the hook enforces it.

## Scope Assumptions

**GitHub-only.** The enforcement scripts (`enforce_pr_workflow.sh`, `merge.sh`, `push.sh`) use the `gh` CLI, and `scripts/lib/git-common.sh`'s `require::repo` helper validates the remote against `github.com`. GitLab, Bitbucket, and Gitea remotes are unsupported — this is a deliberate scope decision, not an oversight. Surfaced user-facing in README as of PR #43 (df4b372). (verified: git-common.sh `require::repo`, PR #43)

**`enforce_pr_workflow` is opt-in.** The hook no-ops when no `workflow.config.yaml` exists. Without config, `gh pr merge` goes through unguarded. `merge.sh` now surfaces this gap with an informational notice before merge (added PR #43). Run `/coderails:init` to generate the config and activate enforcement. (verified: enforce_pr_workflow.sh NO_CONFIG guard, scripts/merge.sh PR #43 addition)

## Cross-References

- [[discipline-loop]] — the specific Stop hooks that enforce self-checking discipline
- [[install-and-cache-trap]] — editing hooks in the repo does not update the running cache without reinstall
- [[no_edit_on_main]] — PreToolUse hook that blocks code edits on main/master (added 2026-06-25)
- [[enforce_pr_workflow]] — PreToolUse hook that gates `gh pr create`/`gh pr merge` (added 2026-06-25)
- [[inject_bootstrap]] — SessionStart hook that bootstraps coderails context (added 2026-06-25)
- [[merge]] — enforcement-gap notice added to merge.sh in PR #43
