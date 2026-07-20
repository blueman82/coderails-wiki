---
title: "PR #250: Loop retro promotion — 4 repo-agnostic lessons promoted"
type: source
created: 2026-07-20
last_updated: 2026-07-20
sources: []
tags: [retro-promotion, judge-architecture, bypass-design, lessons, learned-failure-modes]
---

## Summary

PR #250 promotes four repo-agnostic failure-mode lessons from `standing-orders.md` to `learned-failure-modes.md`, sourced from 46 retros across 46 coderails sessions. The PR completed all automated gates (tier-review verdict: legitimate tier=1) but demonstrates a design constraint: routine PRs cannot auto-merge due to branch-protection ruleset design ("bypass actors: none"), forcing human approval between completion and merge.

## The four promoted lessons

1. **Green suite ≠ gate works** (recurred 7 times, SO-38)
   - Negative controls matter; passing tests prove the test suite runs, not that the gate logic is sound
   - A gate can be theoretically broken and still pass its own green-path tests
   - Evidence: 7 sessions where green suite misled on gate correctness

2. **Frozen evals can be defective** (recurred 4 times, SO-33)
   - Smoke-run both sides; a failing evaluation at freeze-time and a passing evaluation are different findings
   - A frozen, blind-authored check can itself be defective independent of the code under test
   - Evidence: 4 sessions demonstrating the gap between "check passed" and "property holds"

3. **Bash cwd drifts across worktrees** (recurred 6 times, SO-17)
   - Anchor absolute; the working directory resets silently between Bash tool invocations when using git worktrees
   - An assumption about cwd persisting across sequential calls is unsafe in parallel or distributed execution
   - Evidence: 6 sessions with worktree-driven cwd resets causing probe failures

4. **Multi-item "done" is per-item claim** (recurred 2 times, SO-26)
   - Verify each; a worker's "all done" on a multi-item fix list is N claims not one
   - A finding can be fixed in one PR and resurface in a sibling, undetected if only the summary is read
   - Evidence: 2 sessions where multi-item summaries masked item-level regressions

## Evaluation evidence

- **Recurrence threshold:** All four lessons crossed the 2× recurrence bar (across ≥2 distinct sessions)
- **Scope:** All four are repo-agnostic — applicable to any coderails-using repo, not coderails-specific
- **Session count:** Promoted from 46 retros total; no single lesson dominates (7, 4, 6, 2 recurrences)
- **Judge verdict:** Tier-1 legitimate (posted by coderails-blueman82-judge 2026-07-20T02:19:31Z)

## Design constraint: why routine PRs don't auto-merge

The branch-protection ruleset on `main` enforces `bypass actors: none` (verified — [[tier-review-spec]]), implementing the judge architecture principle: "a check performed by the party with motive to pass it is not a check." (verified — [[judge-architecture]])

- The daemon (coderails-blueman82-judge) posts unforgeable tier-review verdicts as an independent evaluator
- The ruleset requires that verdict to merge
- But grants NO bypass authority to any account, including the judge itself
- This forces explicit human oversight of the merge decision

**Consequence:** Routines that complete all gates cannot proceed to merge autonomously. They must await manual merge, at which point they cannot resume post-merge automation (the routine run is stuck at "merge attempted").

**Design decision:** This is intentional, not a gap. The spec trades efficiency for audit integrity. Future solution: a Rachel routine-PR-watcher that detects merge and writes a resumable signal (filed as assistant-agent task 2026-07-20).

## Impact

Four durable, compounding lessons are now in `learned-failure-modes.md`, raising the baseline for any repo adopting coderails' agentic-loop. These escape from session-local standing-orders (ephemeral, loop-scoped) and become project-wide knowledge.

## PR state

- **Created:** 2026-07-20T02:07:42Z
- **Status:** OPEN (awaiting human merge approval)
- **Commits:** 4 (all edits to learned-failure-modes.md)
- **Review:** 3 issues corrected by code-reviewer (evidence citations verified and amended)
- **Evals:** 4/4 PASS (manifest check, promotion criteria, field structure, repo-agnostic principle confirmed)
- **Blocker:** Routine cannot merge without human approval (ruleset design); awaiting manual merge to resume post-merge automation

---

**Related:** [[tier-review-spec]], [[judge-architecture]], [[learned-failure-modes]], [[loop-retro-promotion]]
