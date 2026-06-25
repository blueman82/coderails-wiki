---
title: "requesting-code-review"
type: skill
created: 2026-06-25
last_updated: 2026-06-25
sources: [sources/pr_19-30_self-containment-and-hardening.md]
tags: [skill, code-review, workflow]
---

# requesting-code-review

Encodes the discipline for submitting code for review — what to include, how to frame the diff, and how to prepare a reviewer for a productive pass.

## Trigger phrases

When code is ready to submit for review; when the user says "request review" or "prepare for review". Also invoked by `/coderails:push` at the review stage of the workflow chain.

## Relationship to /workflow

Embedded in the workflow chain at the push→review boundary. `/coderails:push` → this skill → `/pr-review-toolkit:review-pr` → `/coderails:merge`.

## Key phases / steps

1. Verify the branch is clean and pushed.
2. Write a PR description that gives reviewers the "why" not just the "what".
3. Call `/pr-review-toolkit:review-pr` explicitly.
4. Await findings before merging.

## Failure modes encoded

- Opening a PR without a meaningful description, leaving reviewers to reconstruct intent from diffs.
- Merging before review completes.
- Skipping the review step when under time pressure (now also mechanically blocked by [[enforce_pr_workflow]]).

## Source

`coderails/skills/requesting-code-review/SKILL.md`  
`coderails/skills/requesting-code-review/code-reviewer.md` (companion)

## See also

[[receiving-code-review]] — the companion skill for the reviewer's side  
[[finishing-a-development-branch]] — the full branch-wrap-up sequence  
[[enforce_pr_workflow]] — the hook that mechanically requires review before merge  
[[push]]
