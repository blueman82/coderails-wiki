---
title: "finishing-a-development-branch"
type: skill
created: 2026-06-25
last_updated: 2026-06-25
sources: [sources/pr_19-30_self-containment-and-hardening.md, sources/pr_42_skills-hooks-seam.md]
tags: [skill, workflow, branch, completion]
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

## Failure modes encoded

- Merging with failing tests.
- Merging with uncommitted scratch files or debug code.
- PR description that no longer matches the actual changes after review revisions.
- Skipping final check because "we already tested this" earlier.

## Source

`coderails/skills/finishing-a-development-branch/SKILL.md`

## See also

[[receiving-code-review]] — what happens just before this  
[[merge]] — the command this hands off to  
[[verification-before-completion]] — the verification discipline  
[[enforce_pr_workflow]] — the hook that guards the merge step
