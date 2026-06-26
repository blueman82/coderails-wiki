---
title: "no_edit_on_main.sh"
type: hook
created: 2026-06-25
last_updated: 2026-06-26
sources: [sources/pr_19-30_self-containment-and-hardening.md, sources/pr_44_no-edit-plugin-source.md, sources/pr_57-62_subagent-enforcement-gate-hardening.md]
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

1. If target file is **not** in the gated set — pass. The gated set is code extensions (`.py`, `.ts`, `.tsx`, `.js`, `.jsx`, `.go`) **plus plugin source carried in markdown**: `skills/*/SKILL.md` and `commands/*.md` (added PR #44). Plain docs/config — `README.md`, `docs/*.md`, non-`SKILL.md` skill markdown, `.json` — still pass.
2. If not on `main` or `master` branch — pass.
3. Otherwise — deny with a message indicating the current branch and suggesting the user checkout a feature branch.

(verified — PR #27, 11/11 TDD tests; PR #44 extended to 17/17, plugin-source arms covered relative + absolute)

## Block condition

Tool is Write, Edit, or MultiEdit AND current git branch is `main`/`master` AND the target is a gated source file — a code extension, **or** plugin source in markdown (`skills/*/SKILL.md`, `commands/*.md`). All three conditions must hold for the deny to fire.

The path arms are anchored on a `/` boundary (`*/skills/*/SKILL.md|skills/*/SKILL.md`) so a stray directory like `myskills/` can't match, with a bare relative arm for a path passed without a leading directory. (verified — PR #44, `case`-statement glob)

## Log output

Appends to `$CLAUDE_DISCIPLINE_LOG` on block. Format matches the `key=value` convention used by other hooks.

## Environment variables

- `CLAUDE_DISCIPLINE_LOG` — path to the shared discipline log (default `~/.claude/discipline.log`)

## Design note

Uses `permissionDecision: deny` (JSON output to stdout), mirroring `destructive_bash_gate.sh`. This is the PreToolUse block mechanism — distinct from the `exit 2` used by Stop hooks. See [[hook-exit-codes]].

**Why plugin-source markdown is gated (PR #44).** `skills/*/SKILL.md` and `commands/*.md` ARE the plugin's source — editing `agentic-loop/SKILL.md` directly on main is the same class of mistake as committing a `.py` straight to main. The original "docs-only carve-out" let all markdown through, which had let a SKILL.md edit land direct on main earlier the same session. The carve-out is now narrowed to *plain* docs (root `README.md`, `docs/`, non-`SKILL.md` skill references).

**Why `git push` is deliberately NOT gated.** Edit-time (this hook) is the correct seam for the direct-to-main concern. Gating `git push` would be (a) redundant — the edit is already blocked here, and GitHub branch protection covers the server side; (b) breaking — the PR workflow *requires* pushing feature branches (`push.sh`); (c) brittle — "a push targeting main" hides behind implicit upstream, `HEAD`, and refspecs, with no clean token to match (unlike `gh pr create` / `git merge` in [[enforce_pr_workflow]]). Strengthening this edit-time gate is what makes a push-time gate unnecessary. (decision — PR #44 discussion)

## See also

[[enforcement-model]] — the hook/command distinction  
[[destructive_bash_gate]] — mirrors the same deny mechanism  
[[discipline-loop]] — the broader discipline hook composition  
[[enforce_pr_workflow]] — the companion PR-workflow enforcement hook  
`coderails/hooks/scripts/no_edit_on_main.sh`  
`coderails/hooks/tests/no_edit_on_main.test.sh`
