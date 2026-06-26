---
title: "enforce_pr_workflow.sh"
type: hook
created: 2026-06-25
last_updated: 2026-06-26
sources: [sources/pr_19-30_self-containment-and-hardening.md, sources/pr_40_hook-hardening.md, sources/pr_42_skills-hooks-seam.md, sources/pr_46_gate-git-push-on-main.md]
tags: [hook, PreToolUse, enforcement, pr-workflow, workflow-chain]
---

# enforce_pr_workflow.sh

PreToolUse(Bash) hook that mechanically guards the PR workflow chain: blocks `gh pr create` unless `/coderails:push` ran this session; blocks `gh pr merge` unless `/pr-review-toolkit:review-pr` ran this session; (since PR #40) blocks `git merge` on `main`/`master` unless `/pr-review-toolkit:review-pr` ran this session; and (since PR #46) blocks `git push` that lands on `main`/`master` unless `/pr-review-toolkit:review-pr` ran this session.

## Event and mode

| Field | Value |
|---|---|
| Event | `PreToolUse (Bash)` |
| Mode | **block** (permissionDecision: deny) |
| Timeout | (default) |

## Logic summary

Skip gates (cheap first):

1. Empty command string — pass.
2. `--help` / `--dry-run` flags — pass. `git merge --abort/--continue/--quit/--skip` (conflict-resolution ops) — pass.
3. Command does not match `gh pr create`, `gh pr merge`, or `git merge` — pass.
4. `workflow.config.yaml` absent (NO_CONFIG sentinel) — pass. The hook is opt-in via the full workflow stack. (verified — PR #30)
4b. For `git merge` only: if not on `main` or `master` — pass. Feature branches are unconditionally allowed. Detached HEAD / empty branch name falls through to allow (same safe-fail default as [[no_edit_on_main]]).
5. No transcript path in hook payload — pass (can't enforce).
6. Transcript scan for required preceding step. If evidence found — pass. If not found — deny.

Subcommand routing after Gate 3: `create` → requires `/coderails:push` evidence; `merge` (gh) or `git_merge` → requires `/pr-review-toolkit:review-pr` evidence.

## Block condition

- `gh pr create` called without a prior `/coderails:push` in the session transcript.
- `gh pr merge` called without a prior `/pr-review-toolkit:review-pr` in the session transcript.
- `git merge` on `main`/`master` called without a prior `/pr-review-toolkit:review-pr` in the session transcript. (added PR #40)

All checks are NO_CONFIG-gated (Gate 4).

## Why the `git merge` gate was added (PR #40)

The `finishing-a-development-branch` skill includes a "merge locally" option that bypasses the PR path entirely — no `gh pr merge` ever runs, so the pre-existing gate never fires. This left a bypass route. The `git merge` gate closes it: even a local fast-forward merge on main now requires review evidence. (verified — PR #40)

## merge-base exclusion: the word-boundary footgun (PR #42)

PR #40's original gate regex was `\bgit +merge\b`. This also matched `git merge-base` because in POSIX ERE (used by `grep -E`), `-` is a word boundary — so the boundary fires between `merge` and `-base`, and `\bmerge\b` matches. `git merge-base` is a read-only ancestor-lookup plumbing command; blocking it was wrong.

**Before (PR #40):** `\bgit +merge\b`  
**After (PR #42):** `\bgit +merge([[:space:]]|$)`

The fix requires the token after "merge" to be whitespace or end-of-line, excluding `merge-base`, `merge-file`, and `merge-tree`. Applied at both Gate 3 (command classification) and the subcommand-detection block. New test Case 14 asserts `git merge-base HEAD main` on main → allow. (verified — hook source lines 28 and 37)

See [[skills-hooks-seam]] for the general pattern and a note on hyphenated-command regex design.

## Reordered git-merge block-message hint (PR #42)

The block message for `git merge` on main now leads with the actual resolution ("Run /pr-review-toolkit:review-pr first") before listing `/coderails:merge` and the settings.json bypass. Matches the adjacent `gh pr merge` hint order.

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
[[finishing-a-development-branch]] — the skill whose local-merge option motivated the git-merge gate  
[[skills-hooks-seam]] — the cross-reference convention this hook participates in; the merge-base regex footgun  
`coderails/hooks/scripts/enforce_pr_workflow.sh`
