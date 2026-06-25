---
title: "enforce_pr_workflow.sh"
type: hook
created: 2026-06-25
last_updated: 2026-06-25
sources: [sources/pr_19-30_self-containment-and-hardening.md]
tags: [hook, PreToolUse, enforcement, pr-workflow, workflow-chain]
---

# enforce_pr_workflow.sh

PreToolUse(Bash) hook that mechanically guards the PR workflow chain: blocks `gh pr create` unless `/coderails:push` ran this session, and blocks `gh pr merge` unless `/pr-review-toolkit:review-pr` ran this session.

## Event and mode

| Field | Value |
|---|---|
| Event | `PreToolUse (Bash)` |
| Mode | **block** (permissionDecision: deny) |
| Timeout | (default) |

## Logic summary

Skip gates (cheap first):

1. If `workflow.config.yaml` is absent (NO_CONFIG sentinel) — pass immediately. The hook is opt-in via the full workflow stack. (inferred — PR #30 body)
2. If the Bash command does not contain `gh pr create` or `gh pr merge` — pass.
3. For `gh pr create`: scan the session transcript for evidence that `/coderails:push` ran. If not found — deny.
4. For `gh pr merge`: scan the session transcript for evidence that `/pr-review-toolkit:review-pr` ran. If not found — deny.

## Block condition

`gh pr create` called without a prior `/coderails:push` in the session transcript, OR `gh pr merge` called without a prior `/pr-review-toolkit:review-pr` in the session transcript. Both checks are NO_CONFIG-gated.

## Log output

Appends to `$CLAUDE_DISCIPLINE_LOG` on block. Format matches `key=value` convention.

## Why it exists

Before this hook, the workflow chain (`/push → /review-pr → /merge`) was advisory: Claude could invoke `gh pr create` or `gh pr merge` directly, bypassing the mandated push and review steps. This hook converts those two checkpoints from advisory to mechanical. Closes review finding #C. (verified — PR #30)

## Auto-chmod

This hook is auto-chmod'd by `install.sh`'s hooks.json-derivation (PR #28). No manual chmod step needed when adding new hooks that follow the `hooks.json` registration pattern.

## Environment variables

- `CLAUDE_DISCIPLINE_LOG` — path to the shared discipline log
- `workflow.config.yaml` — presence/absence is the NO_CONFIG opt-in gate

## See also

[[enforcement-model]] — the hook/command distinction; why this is a hook not a command  
[[no_edit_on_main]] — the companion PreToolUse enforcement hook (same PR wave)  
[[discipline-loop]] — broader discipline hook composition  
[[push]] — the command this hook requires ran before `gh pr create`  
[[workflow]] — the full chain this hook enforces  
`coderails/hooks/scripts/enforce_pr_workflow.sh`
