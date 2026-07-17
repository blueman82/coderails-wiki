---
title: "finishing-a-development-branch"
type: skill
created: 2026-06-25
last_updated: 2026-07-17
sources: [sources/pr_19-30_self-containment-and-hardening.md, sources/pr_42_skills-hooks-seam.md, sources/pr_224_231_233_235_loop-tooling-hardening.md]
tags: [skill, workflow, branch, completion, worktree-lock]
---

# finishing-a-development-branch

Wraps up a development branch for merge: final verification, cleanup, PR readiness check, and confirming nothing is left open.

## Trigger phrases

When a branch is ready to merge; "finish the branch", "wrap up", "prepare to merge". Sits near the end of the workflow chain.

## Relationship to /workflow

Downstream of the review loop. Typically: review done → this skill → `/coderails:merge`. Also invoked by [[agentic-loop]] before the merge phase.

## Key phases / steps

1. Confirm all tests pass on the branch.
2. Confirm no stray uncommitted changes.
3. Confirm the PR description accurately reflects what landed (review may have changed scope).
4. Confirm review approval is recorded.
5. Hand off to `/coderails:merge`.

## Hook-gated actions

**Option 1 — merge locally:** If a skill phase instructs `git merge` directly on main/master (rather than via `gh pr merge`), [[enforce_pr_workflow]] will block it unless `/pr-review-toolkit:review-pr` ran this session. Resolution: run `/pr-review-toolkit:review-pr` first, then retry the merge. In a workflow.config.yaml-configured repo this gate is always active; it is not opt-in at the command level. (verified — PR #42, CLAUDE.md seam convention)

## Step 6 worktree cleanup — lock-aware (PR #235)

Step 6 (merged-worktree removal, run per-work-unit by [[agentic-loop]] Phase
4b) now checks lock state before removing, not just the provenance check
(only remove worktrees the loop itself created, under `.worktrees/` or
`worktrees/`). Lock reason is parsed from `git worktree list --porcelain`'s
`locked <reason>` line:

- **Unlocked** — remove normally, unchanged.
- **Locked, live pid** — a pid is parsed out of the lock reason (harness
  format: `claude session <name> (pid NNNNN start <date>)`), then `kill -0`
  checked. Live → **report and defer, never force**.
- **Locked, dead pid (or no pid parseable)** — a confirmed-dead pid clears
  the lock (`git worktree unlock`) and removes; an unparseable reason is
  treated the same as "live" for safety (report, leave alone).

Only a confirmed-dead pid ever triggers a force-clear — a merged PR does not
by itself mean the worktree is safe to remove, since another session may
still be actively working in it at the exact moment teardown runs. See
[[pr_224_231_233_235_loop-tooling-hardening]] for the full source record.

## Failure modes encoded

- Merging with failing tests.
- Merging with uncommitted scratch files or debug code.
- PR description that no longer matches the actual changes after review revisions.
- Skipping final check because "we already tested this" earlier.
- Force-removing a locked worktree without checking whether its pid is dead (PR #235).

## Source

`coderails/skills/finishing-a-development-branch/SKILL.md`

## See also

[[receiving-code-review]] — what happens just before this  
[[merge]] — the command this hands off to  
[[verification-before-completion]] — the verification discipline  
[[enforce_pr_workflow]] — the hook that guards the merge step  
[[pr_224_231_233_235_loop-tooling-hardening]] — PR #235, the lock-aware Step 6 documented above
