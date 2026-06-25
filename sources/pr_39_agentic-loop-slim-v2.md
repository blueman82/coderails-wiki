---
title: "PR 39 — agentic-loop slim v2 (reference-not-embody): refs normalised, zero further candidates"
type: source
created: 2026-06-25
last_updated: 2026-06-25
sources:
  - skills/agentic-loop/SKILL.md
tags: [source, pr, agentic-loop, slim, reference-not-embody, autonomy-superset]
---

# PR 39 — agentic-loop slim v2 (reference-not-embody)

<!-- Ingested by /wiki-ingest. Immutable record of what changed + the design verdict that closed the task. -->

## PR metadata

| Field | Value |
|---|---|
| PR number | #39 |
| Branch | `slim-agentic-loop-v2` |
| Merged | 2026-06-25T14:12:57Z |
| Merge SHA | `51414a8` |
| JIRA ticket | — |

## Summary

The "slim agentic-loop v2: reference-not-embody" task aimed to shrink `skills/agentic-loop/SKILL.md` by replacing inline restatements of generic discipline with one-line references to the now-vendored `coderails:` skills — without deleting the autonomy-specific layer. The realistic delta turned out to be **zero** after one cosmetic PR:

- **PR #39** normalised 13 bare/unqualified skill references (`/agentic-loop`, `/planning-sequence`, `/premortem`, `/wiki-query`, `/workflow`, `/wiki-ingest`, `/wiki-lint`, `coderails:handoff`, `coderails:writing-plans`, `pr-review-toolkit`) to the fully-qualified `/coderails:` form so they resolve as invocable references. Prose-only; no C1/C2 frozen region touched; the three agentic-loop hook suites stayed green. This was the *substance* of "reference-not-embody."
- **The re-scan verdict:** two independent passes (main context + a fresh read-only Explore agent that was not told the expected answer) classified all 22 phases against a four-part test and **both converged on zero replaceable passages.**

## The four-part test (why nothing else could be slimmed)

A passage is replaceable by a one-line skill reference ONLY IF all four hold:
1. **Generic** — not specific to autonomous operation.
2. **Fully covered** — the named vendored skill already says it, completely.
3. **Worker-facing** — it goes into a *spawned agent's* prompt (workers can invoke skills); the orchestrator's own behaviour does not count.
4. **No autonomy delta** — it adds nothing beyond the generic skill.

Every phase failed part 3 (orchestrator-facing) or part 4 (it is a **superset** of the vendored skill), or was a frozen region. The embodied skills are autonomy *supersets, not duplicates* — e.g. `using-git-worktrees` says "make a worktree"; agentic-loop Phase 2 adds the dirty-base clean-base check + branch-off-freshly-fetched-`origin/main` + foreign-file exclusion. The last tempting near-miss, **Phase 5** ("reproduce before fixing" = `coderails:systematic-debugging`'s Iron Law), is KEEP: it is wrapped in an autonomy delta — the source-of-truth caching warning (Slack pin-bar / GitHub / Jira / browser all cache) plus "STOP and report — don't ship a fix to a non-bug," which ties to the loop's hard-stop condition.

## Caveats / gotchas

- **Do NOT re-attempt the slim.** Any further "slimming" would delete autonomy deltas the constraints exist to preserve. The phase count stays at 459 lines by design, not by neglect.
- The adjacent option of additive "see also" cross-refs was declined — it grows line count, the opposite of slimming.
- This is the v2 follow-up to Spec B's earlier 454→434 cut (see [[agentic-loop]] § Slimming). Spec B removed genuine bloat (war-stories, corporate residue); v2 confirmed the residue is now exhausted.

## Wiki pages updated

- [[agentic-loop]] — § Slimming extended with the v2 outcome.
