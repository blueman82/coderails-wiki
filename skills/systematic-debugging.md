---
title: "systematic-debugging"
type: skill
created: 2026-06-25
last_updated: 2026-06-25
sources: [sources/pr_19-30_self-containment-and-hardening.md]
tags: [skill, debugging, diagnosis, root-cause]
---

# systematic-debugging

Prevents thrashing on bugs by enforcing a structured hypothesis-then-verify loop before touching any code.

## Trigger phrases

When a bug needs diagnosis; "debug this", "systematic-debugging", "why is X failing". Use proactively before making any speculative fix.

## Relationship to /workflow

Sits beside the workflow chain, invoked during the code/iterate phase. Not wired into the chain itself but called explicitly when a bug is encountered.

## Key phases / steps

1. Reproduce the failure with a minimal case.
2. Form one hypothesis at a time — don't generate a list of guesses.
3. Design a check that proves or disproves the hypothesis without changing code.
4. Act only after the hypothesis is confirmed.
5. Apply the fix surgically; verify the reproduction case passes.

## Failure modes encoded

- Applying multiple speculative fixes at once, making it impossible to know which one worked.
- Skipping reproduction and fixing from the symptom description alone.
- Over-engineering the fix beyond what the confirmed hypothesis requires.
- Touching adjacent code while "in the area".

## Source

`coderails/skills/systematic-debugging/SKILL.md`

## See also

[[verification-before-completion]] — the verification discipline after the fix  
[[test-driven-development]] — write a failing test first, then fix
