---
title: "PR 86 — agentic-loop: harden per 7 resolved design decisions"
type: source
created: 2026-07-01
last_updated: 2026-07-06
sources: []
tags: [skill, agentic-loop, enforcement-model, self-attestation, hardening]
---

# PR 86 — agentic-loop: harden per 7 resolved design decisions

<!-- Ingested by /wiki-ingest after merge. This is an immutable record of what changed. -->

## PR metadata

| Field | Value |
|---|---|
| PR number | #86 |
| Branch | `docs/agentic-loop-hardening-spec-plan` |
| Merged | 2026-07-01 |
| Merge SHA | `b8a1958` |
| JIRA ticket | — |

## Summary

Hardened `skills/agentic-loop/SKILL.md` against 7 findings from a review of the skill, each
already resolved to a single decision before implementation (design doc:
`docs/coderails/specs/2026-07-01-agentic-loop-hardening-design.md`; plan:
`docs/coderails/plans/2026-07-01-agentic-loop-hardening-plan.md`). Pure docs/markdown/bash-comment
edits — no code, no tests, per the spec's own carve-out (verified: PR body).

1. **Stage-map overview** — a 5-row stage table (Setup / Pre-flight / Build / Review & Ship /
   Wrap-up) added under `## The phases`, grouping the 19+ numbered phases so a cold reader has a
   shape to hold in mind before the phase-by-phase detail.
2. **Merge Phase 2.7 + 2.8** — both phases restated the identical `≥3 work-units or cross-unit
   dependency` guard as their opening sentence; merged into one Phase 2.7 (`spec.md` write) with
   lettered sub-steps 2.7a (`spec.md`, was 2.7's content) / 2.7b (`plan.md` via
   `/coderails:writing-plans`, was 2.8's content), guard stated once. Phase 2.5/2.6 were
   deliberately NOT folded into this merge — they fire unconditionally (no guard) and each carries
   6 inbound cross-references elsewhere in the file that a merge would risk breaking for no
   corresponding duplication removed.
3. **Remove orchestrator self-demote power (Phase 4b)** — previously the orchestrator could write
   "reviewed, not compat — `<reason>`" to unilaterally downgrade the independent `code-simplifier`
   reviewer's clean-break `MERGE-BLOCKER` finding to a logged note. The party demoting was the same
   party whose worker shipped the compat path — a self-grading loophole. Now the orchestrator's
   only two moves are (a) actually fix it, or (b) hard-stop and hand it to a human (logged
   who/when/SHA/reason). A fully-unattended envelope that cannot tolerate ever hard-stopping here
   must have auto-demote authority explicitly granted **at Phase 0 envelope-authorisation time** —
   never something the orchestrator grants itself mid-run.
4. **Drop the Phase 13 scorecard** — "human turns approaching zero" couldn't distinguish a
   well-calibrated loop from one that silently suppressed asks (and a freshly-spawned auditor
   grading the same orchestrator-authored `progress.json` records was rejected too — it grades
   homework against homework, and risks a Goodhart's-law failure where the safe strategy becomes
   padding the record to look thorough). Replaced with two raw, unscored facts: `LOOP-STOP`
   category counts (from `progress.json`'s `loop_stop_counts`) and a flat, unscored "decisions
   absorbed" list. The human judges; the process no longer pre-grades itself.
5. **Document `model: sonnet` as advisory** — `AGENTS.md`'s "Enforcement ceilings" list gains a
   bullet stating no hook gates `Agent`/`Task` spawn calls on model choice, and that this is
   deliberate: the rule is cost control, not correctness, and a blunt model-gate hook couldn't
   distinguish Phase 2.5's legitimate opus-escalation exception from a disallowed spawn without a
   self-reported flag — reintroducing the same trust-the-agent problem one level down.
6. **Single-loop-per-directory invariant** — `progress.json` is keyed only on project working
   directory (Phase -2), not session; two concurrent `agentic-loop` sessions in the same checkout
   race for last-writer-wins ownership. `loop_state_guard.sh` already fails closed on session
   mismatch (the dangerous silent-data-loss case is handled), but does not prevent the race itself.
   Documented in both `SKILL.md`'s `## Context-window persistence` section and the header comment
   of `hooks/scripts/lib/agentic_loop_path.sh`, pointing at `coderails:using-git-worktrees` as the
   resolution — no locking machinery was built (rejected as disproportionate complexity for a rare,
   unsupported-configuration failure mode).

   **Superseded 2026-07-06 by [[pr_23-24_hook-lib-observability-and-repo-keyed-loop-state|PR #24]]**:
   the calculus changed once the shared-checkout workflow began *prescribing* mid-session
   `EnterWorktree` hops, and a live orphaning incident (a worktree hop silently splitting a loop's
   state onto a new cwd-keyed path) showed "just use worktrees" was itself a new failure mode this
   point didn't anticipate. PR #24 repo-keys the slug via `git --git-common-dir` instead — a ~15ms
   keying change, not the locking machinery rejected above. The two decisions aren't in tension:
   this point rejected a heavy mechanism for a narrower problem (concurrent-session collision); PR
   #24 ships a light mechanism for the problem the later workflow shift actually created (worktree-hop
   orphaning). See [[agentic-loop-path-keying]] for the full design. Historical text above preserved
   as the record of what was decided at the time.
7. **Replace 3 stale memory-file citations** — `SKILL.md` cited `feedback_wiki_ingest_and_lint_post_merge`,
   `feedback_parallel_wiki_agents`, and `feedback_three_parallel_adversarial_agents` by name; none
   existed in the design author's memory directory at design time. Each citation was replaced with
   the underlying principle restated as self-contained inline prose, so the skill file no longer
   depends on external per-user/per-machine state that ships-elsewhere users won't have.

**Post-merge review** (4 agents + a security pass) found and fixed, on the same PR branch before
merge: 2 Critical issues (stale cross-references at `SKILL.md:435` and `:480` that still described
the pre-rewrite Phase 13 shape after the scorecard was dropped in point 4) and 1 Important issue
(Phase 0's thinking-block template didn't force an explicit answer on the new auto-demote-authority
carve-out from point 3 — fixed by adding a required yes/no + verbatim-quote line to the Phase 0
template). All fixes landed inside the same merge commit `b8a1958` (verified: `git log b8a1958..HEAD`
is empty — HEAD is b8a1958, confirming no separate post-merge commits exist on `main`).

## Files changed

- `skills/agentic-loop/SKILL.md` (+34/-19) — all 7 decisions plus the 3 post-merge-review fixes.
- `AGENTS.md` (+9/-0) — new "Enforcement ceilings" bullet for `model: sonnet` (point 5).
- `hooks/scripts/lib/agentic_loop_path.sh` (+6/-0) — header-comment invariant sentence (point 6).
- `docs/coderails/specs/2026-07-01-agentic-loop-hardening-design.md` (new, +329) — full spec:
  problem/options/decision/reasoning for all 7 points.
- `docs/coderails/plans/2026-07-01-agentic-loop-hardening-plan.md` (new, +427) — 8-task
  implementation plan with a Pre-Parade/Premortem/Red-Team stress-test.

## Wiki pages updated

- [[agentic-loop]] — phase table, stage-map, Phase 2.7 merge, Phase 4b self-attestation fix, Phase
  13 rewrite, single-loop invariant, all reflected.
- [[enforcement-model]] — cross-reference added for the `model: sonnet` advisory-ceiling decision
  and the Phase 4b self-attestation fix as an enforcement-boundary case study.

## Caveats / gotchas

- **Out of scope, flagged not fixed**: `docs/coderails-review.md:160` cites `SKILL.md:209`,
  `SKILL.md:239,260`, and `SKILL.md:207-260` as evidence for a prior correction. Those line numbers
  are now stale (new locations: 220, 250/272) because of the stage-map addition (point 1) and the
  Phase 2.7/2.8 merge (point 2) shifting the file's line count. The spec explicitly left this
  untouched and flagged it as a follow-up task — Task 8 of the implementation plan (read-only)
  confirmed the drift but did not fix `coderails-review.md`. This is a known, deliberate gap, not
  an oversight to re-open as a fresh finding.
- The Phase 4b auto-demote-authority carve-out (point 3) must be **quoted verbatim from the
  envelope**, not inferred from a general "fully autonomous" classification — Phase 0's
  thinking-block template now has a dedicated yes/no + quote line enforcing this after the
  post-merge Important-severity fix.
- Phase 2.5 and Phase 2.6 remain **unmerged** despite point 2's phase-consolidation theme — a
  deliberate exclusion (unconditional firing + 6 inbound cross-references each), not an
  inconsistency.
