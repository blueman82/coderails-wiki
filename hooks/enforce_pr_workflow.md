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
3. Command does not match `gh pr create`, `gh pr merge`, `git merge`, or `git push` — pass. The `\bgit +push([[:space:]]|$)` anchor mirrors the `git merge` anchor for form consistency (no `git push-*` plumbing exists to exclude). (verified — hook source line 34)
4. `workflow.config.yaml` absent (NO_CONFIG sentinel) — pass. The hook is opt-in via the full workflow stack. (verified — PR #30)
4b. For `git merge` and `git push` only: pass unless the operation touches `main`/`master`.
   - `git merge` integrates into the **checked-out** branch → the current branch decides. If not on `main`/`master` — pass.
   - `git push` is decided by its **destination** → gate when on `main`/`master`, OR when the command names an explicit main/master destination refspec (`HEAD:main`, `feature:master`, `:refs/heads/main`) from any branch. (verified — PR #46, hook source lines 67–86)
   Feature branches are unconditionally allowed. Detached HEAD / empty branch name falls through to allow (same safe-fail default as [[no_edit_on_main]]).
5. No transcript path in hook payload — pass (can't enforce).
6. Transcript scan for required preceding step. If evidence found — pass. If not found — deny.

Subcommand routing after Gate 3: `create` → requires `/coderails:push` evidence; `merge` (gh), `git_merge`, or `git_push` → requires `/pr-review-toolkit:review-pr` evidence.

## Block condition

- `gh pr create` called without a prior `/coderails:push` in the session transcript.
- `gh pr merge` called without a prior `/pr-review-toolkit:review-pr` in the session transcript.
- `git merge` on `main`/`master` called without a prior `/pr-review-toolkit:review-pr` in the session transcript. (added PR #40)
- `git push` landing on `main`/`master` (current branch OR explicit destination refspec) called without a prior `/pr-review-toolkit:review-pr` in the session transcript. (added PR #46)

All checks are NO_CONFIG-gated (Gate 4).

## Why the `git merge` gate was added (PR #40)

The `finishing-a-development-branch` skill includes a "merge locally" option that bypasses the PR path entirely — no `gh pr merge` ever runs, so the pre-existing gate never fires. This left a bypass route. The `git merge` gate closes it: even a local fast-forward merge on main now requires review evidence. (verified — PR #40)

## merge-base exclusion: the word-boundary footgun (PR #42)

PR #40's original gate regex was `\bgit +merge\b`. This also matched `git merge-base` because in POSIX ERE (used by `grep -E`), `-` is a word boundary — so the boundary fires between `merge` and `-base`, and `\bmerge\b` matches. `git merge-base` is a read-only ancestor-lookup plumbing command; blocking it was wrong.

**Before (PR #40):** `\bgit +merge\b`  
**After (PR #42):** `\bgit +merge([[:space:]]|$)`

The fix requires the token after "merge" to be whitespace or end-of-line, excluding `merge-base`, `merge-file`, and `merge-tree`. Applied at both Gate 3 (command classification) and the subcommand-detection block. New test Case 14 asserts `git merge-base HEAD main` on main → allow. (verified — hook source lines 28 and 37)

See [[skills-hooks-seam]] for the general pattern and a note on hyphenated-command regex design.

## The `git push` gate: destination-refspec model (PR #46)

PR #46 added `git_push` as a fourth gated subcommand, mirroring the `git merge` gate's review-evidence requirement. The key difference is **what decides whether to gate**:

- `git merge` integrates into the checked-out branch, so the **current branch** is sufficient.
- `git push`'s effect is decided by its **destination**, so the destination must be parsed. Gate 4b gates when the current branch is `main`/`master` **OR** the command names an explicit main/master destination refspec (`HEAD:main`, `feature:master`, `:refs/heads/main`) from any branch.

The destination anchor is `:(refs/heads/)?(main|master)([[:space:];&|)]|$)`. The trailing class accepts whitespace, EOL, **or** a shell separator (`;& |)`) so that `git push origin HEAD:main;echo` can't abut a metachar onto the ref to evade the gate. (verified — hook source lines 81–84)

**Documented limitation:** bare positional `git push origin main` from an off-main branch is **not** parsed (low risk — it pushes local `main`, which tracks `origin/main`). The colon-refspec form is the realistic direct-to-main bypass, and that is closed. Feature-branch pushes are never gated — the PR flow requires them. (verified — hook source lines 9–10, 70–73)

**Review-caught Critical:** the first implementation checked the current branch only, which let a `HEAD:main` refspec push from a feature branch through — a silent false-allow flagged Critical by the [[pr-review-toolkit]] multi-agent review and fixed by destination-refspec gating + the metachar anchor, each TDD'd. Enforce test suite went 14→27 cases. This does **not** reverse [[pr_44_no-edit-plugin-source]]'s "don't gate git push for [[no_edit_on_main]]" decision — that was edit-seam protection; this is review-evidence enforcement in a different hook. See [[pr_46_gate-git-push-on-main]].

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
