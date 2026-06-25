---
title: "verification-before-completion"
type: skill
created: 2026-06-25
last_updated: 2026-06-25
sources: [sources/pr_19-30_self-containment-and-hardening.md]
tags: [skill, verification, quality, completion]
---

# verification-before-completion

Prevents declaring a task done without verifying the implementation against the original requirements — runs a structured checklist before Claude says "done".

## Trigger phrases

Before any "done" declaration; "verify-before-completion", "check this is complete". Invoked at the end of any implementation unit.

## Relationship to /workflow

Embedded at the end of the code/iterate phase before `/push`. Also invoked at the end of each agentic-loop worker-unit before commit.

## Key phases / steps

1. Re-read the original requirements / acceptance criteria.
2. Verify each criterion is met — not "I believe it's met", but "I've confirmed it".
3. Run tests or checks as evidence.
4. Produce a Did Not Verify section for anything genuinely unverifiable.
5. Only then declare done.

## Failure modes encoded

- "I believe this is complete" without checking.
- Checking only the happy path.
- Forgetting requirements that were stated early in a long session.
- Omitting the Did Not Verify section and presenting partial work as complete.

## Source

`coderails/skills/verification-before-completion/SKILL.md`

## See also

[[finishing-a-development-branch]] — uses this discipline at branch wrap-up  
[[systematic-debugging]] — the companion discipline for diagnosis  
[[check_verify_loop]] — the hook that enforces Did Not Verify discipline
