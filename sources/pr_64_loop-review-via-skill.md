---
title: "PR #64 — loop Phase 4b: invoke review-pr Skill instead of hand-rolling agents"
type: source
created: 2026-06-26
last_updated: 2026-06-26
sources: []
tags: [agentic-loop, enforce_pr_workflow, pr-review, gate-evidence]
---

# PR #64 — loop Phase 4b: invoke review-pr Skill instead of hand-rolling agents

<!-- Ingested by /wiki-ingest after merge. This is an immutable record of what changed. -->

## PR metadata

| Field | Value |
|---|---|
| PR number | #64 |
| Branch | `feat/loop-review-via-skill` |
| Merged | 2026-06-26 |
| Merge SHA | `ee44aa0` |
| JIRA ticket | — |

## Summary

Phase 4b of the agentic-loop skill was changed so that the PR-review step **invokes `/pr-review-toolkit:review-pr <PR#>` as a Skill** (passing the PR number as the argument) instead of hand-rolling the six reviewer agents as separate `Agent` or `Task` spawns. A one-sentence note was also added to CLAUDE.md's skills↔hooks seam convention paragraph to record that the merge gate's accepted review evidence is the `review-pr` Skill invocation with the PR number in args.

The root cause: `enforce_pr_workflow.sh` (gate function `enforce_required_step`) scans the transcript for Skill invocations of `/pr-review-toolkit:review-pr`. A manual agent fanout produces no such Skill record, so the gate sees no evidence and blocks `gh pr merge` / `git merge` on main. The agentic loop was therefore blocked at merge despite having run all six reviewer agents.

## Files changed

- `skills/agentic-loop/SKILL.md` — Phase 4b heading and body rewritten: Skill invocation is the required form; the table of six reviewers is retained as reference (now the Skill's internal implementation); the enforcement reason is called out prominently.
- `CLAUDE.md` — skills↔hooks seam convention paragraph extended with one sentence: the merge gate accepts only the `review-pr` Skill (with PR number) as evidence, not a manual agent fanout.

## Wiki pages updated

- [[agentic-loop]] — Phase 4b table row updated; new "Phase 4b — invoke the review-pr Skill" section added; cross-references and sources updated.
- [[enforce_pr_workflow]] — new "Accepted PR-review evidence" section added explaining the Skill-only evidence model and the per-PR number requirement.

## Caveats / gotchas

The gate checks for **invocation evidence**, not completion (existing known limitation documented in the "Evidence model and known limitation" section of [[enforce_pr_workflow]]). A Skill call that errors immediately still satisfies the gate — the real enforcement is the Skill's own behaviour plus GitHub branch protection. This pre-existing limitation is unchanged by PR #64.
