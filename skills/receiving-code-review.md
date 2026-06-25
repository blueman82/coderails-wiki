---
title: "receiving-code-review"
type: skill
created: 2026-06-25
last_updated: 2026-06-25
sources: [sources/pr_19-30_self-containment-and-hardening.md]
tags: [skill, code-review, workflow]
---

# receiving-code-review

Encodes how to receive and act on code review findings: triage by severity, apply fixes without scope creep, and close the loop with the reviewer.

## Trigger phrases

When the user says "I've got review feedback", "respond to review", or when review findings arrive during the workflow chain.

## Relationship to /workflow

Part of the review loop: `/pr-review-toolkit:review-pr` produces findings → this skill governs how Claude handles them → [[finishing-a-development-branch]] or re-push.

## Key phases / steps

1. Read all findings before acting — don't apply the first one in isolation.
2. Triage: must-fix bugs vs. suggestions vs. nitpicks. Apply must-fix first.
3. For each finding: fix surgically (match existing style; don't refactor adjacent code).
4. Re-push and confirm reviewer is satisfied before merging.

## Failure modes encoded

- Applying every suggestion including speculative ones, causing scope creep.
- Ignoring blocking findings because they're inconvenient.
- Not confirming review is satisfied before merging.

## Source

`coderails/skills/receiving-code-review/SKILL.md`

## See also

[[requesting-code-review]] — the companion skill for preparing the review  
[[finishing-a-development-branch]] — what comes after review is clean  
[[enforce_pr_workflow]] — hook that blocks merge until review ran
