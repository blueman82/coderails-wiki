---
title: "PR 56 — writing-skills using-coderails example + CLAUDE_MD_TESTING"
type: source
created: 2026-06-26
last_updated: 2026-06-26
sources: []
tags: [source, writing-skills, docs, skill-testing]
---

# PR 56 — writing-skills using-coderails example + CLAUDE_MD_TESTING

<!-- Ingested by /wiki-ingest after merge. Immutable record of what changed. -->

## PR metadata

| Field | Value |
|---|---|
| PR number | #56 |
| Branch | `chore/writing-skills-using-coderails-example` |
| Merged | 2026-06-26 |
| Merge SHA | `0f0969e` (squash) |
| JIRA ticket | — |

## Summary

Two doc changes to the `writing-skills` skill:
1. **Stale-example fix** in `writing-skills/SKILL.md` — a naming-convention example read `using-skills not skill-usage`; updated to `using-coderails not skill-usage` to reflect the bootstrap skill's actual name after the coderails rename.
2. **New worked example** `writing-skills/examples/CLAUDE_MD_TESTING.md` — a full test-campaign walkthrough (testing CLAUDE.md documentation variants), which `testing-skills-with-subagents.md` already *referenced* ("See examples/CLAUDE_MD_TESTING.md") but which did not exist until now. This closes a previously-dangling reference.

## Files changed

- `skills/writing-skills/SKILL.md` — one-line example correction.
- `skills/writing-skills/examples/CLAUDE_MD_TESTING.md` — **new** worked example.

## Wiki pages updated

- [[writing-skills]] — last_updated bumped; skill-testing note + this source added

## Caveats / gotchas

- This is purely documentation of the **manual** skill-testing ritual (`testing-skills-with-subagents.md`, RED/GREEN/REFACTOR via subagents). It adds no automation — coderails still has zero automated skill testing. See [[skill-testing-state_2026-06-26]].
- Landed alongside [[pr_55_remove-dead-evals]] in the same session; the two are thematically paired (one removes a dead *automated*-looking eval, one completes the *manual*-testing docs).
