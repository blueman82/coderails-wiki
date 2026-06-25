---
title: "no_edit_on_main.sh"
type: hook
created: 2026-06-25
last_updated: 2026-06-25
sources: [sources/pr_19-30_self-containment-and-hardening.md]
tags: [hook, PreToolUse, enforcement, main-branch-protection]
---

# no_edit_on_main.sh

PreToolUse hook that blocks code-file edits (Write/Edit/MultiEdit) on `main` or `master` branches, enforcing the no-direct-edits-on-main invariant.

## Event and mode

| Field | Value |
|---|---|
| Event | `PreToolUse (Write \| Edit \| MultiEdit)` |
| Mode | **block** (permissionDecision: deny) |
| Timeout | (default) |

## Logic summary

Skip gates (cheap first):

1. If not on `main` or `master` branch — pass immediately.
2. If target file extension is not in the code-file set (`.py`, `.ts`, `.tsx`, `.js`, `.jsx`, `.go`) — pass (docs, config, markdown allowed).
3. Otherwise — deny with a message indicating the current branch and suggesting the user checkout a feature branch.

(verified — PR #27, 11/11 TDD tests pass)

## Block condition

Tool is Write, Edit, or MultiEdit AND current git branch is `main`/`master` AND the target file has a code extension. All three conditions must hold for the deny to fire.

## Log output

Appends to `$CLAUDE_DISCIPLINE_LOG` on block. Format matches the `key=value` convention used by other hooks.

## Environment variables

- `CLAUDE_DISCIPLINE_LOG` — path to the shared discipline log (default `~/.claude/discipline.log`)

## Design note

Uses `permissionDecision: deny` (JSON output to stdout), mirroring `destructive_bash_gate.sh`. This is the PreToolUse block mechanism — distinct from the `exit 2` used by Stop hooks. See [[hook-exit-codes]].

## See also

[[enforcement-model]] — the hook/command distinction  
[[destructive_bash_gate]] — mirrors the same deny mechanism  
[[discipline-loop]] — the broader discipline hook composition  
[[enforce_pr_workflow]] — the companion PR-workflow enforcement hook  
`coderails/hooks/scripts/no_edit_on_main.sh`  
`coderails/hooks/tests/no_edit_on_main.test.sh`
