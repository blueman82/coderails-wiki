---
title: "PR #21-22 — Loop 2 suggestion-tier follow-ups: merge.sh tempfile arm + test-coverage completions"
type: source
created: 2026-07-06
last_updated: 2026-07-06
sources: []
tags: [source, merge, tempfile, trust-floor, push, loop-state-guard, install-mode-sweep, test-coverage]
---

# PR #21-22 — Loop 2 suggestion-tier follow-ups

Ingested by `/coderails:wiki-ingest` after merge. Immutable record of what changed.

## PR metadata

| Field | Value |
|---|---|
| PR number | #21, #22 |
| Base | `9f8f1af` |
| Merge SHA | #21 → `d0e4c5a`; #22 → `1033512` (final `main` after this cluster) |
| Merged | 2026-07-06 |

## Summary

Loop 2 closed 4 of 6 suggestion-tier residuals recorded against
[[pr_11-14_gate-hardening-followups]] (PRs #11–14). Two work units:

**PR #21 (wu-b1) — merge.sh tempfile failure-reason arm.** `pr::_trusted_comment_bodies_or_fail`
in `scripts/lib/git-common.sh` sets `PR_TRUST_FETCH_FAIL_REASON="tempfile"` when its own
`mktemp` call fails, *before* any `gh` API call is attempted (verified:
`scripts/lib/git-common.sh:182-196`). Before this PR, `merge.sh`'s `case` on that variable had
no `tempfile)` arm, so a local mktemp failure fell into the generic `*)` branch and printed
"GitHub fetch failed — could not fetch PR comments" — actively misleading, since no fetch was
ever attempted. `merge.sh` gains a dedicated `tempfile)` arm on **both** gates (review-artifact
and eval-artifact), each with its own gate-specific wording, deliberately *not* saying "GitHub
fetch failed." TDD: Test 3d in `merge.test.sh` and Test 4d in `merge_evals_gate.test.sh`, both
including message-distinctness assertions against the identity/fallback messages. Reader
exit-code contract (0/1/2) unchanged; `git-common.sh` itself untouched (the reason was already
being set — only the `merge.sh` consumer was missing the case arm).

**PR #22 (wu-a1/a2/a3) — test-only coverage completions**, closing gaps flagged but not filled
by [[pr_11-14_gate-hardening-followups]]:
- `push_staging.test.sh` gains a **PRE-STAGED NEW FILE** case (a file already `git add`ed before
  `push.sh` runs — proves `git add -u`'s tracked-only migration doesn't regress already-staged
  new content) and a **DELETED TRACKED FILE** case (proves `git add -u` still stages deletions
  of tracked paths, not just modifications).
- `loop_state_guard_evals.test.sh` gains a `tier!=0`, justified, **result key absent entirely**
  fixture — the first fixture to genuinely reach `als_read_loop_evals_result`'s final `else`
  branch (NO-GO). A pre-existing fixture's comment claiming to exercise that branch was a
  misnomer; it actually matched an earlier `elif`. Covered at both the reader level and the
  end-to-end guard-invocation level.
- `install_mode_sweep.test.sh` gains a pre-seeded conflicting `commands/workflow.md` file (an
  empty commands dir never triggers install.sh's conflict-scan prompt at all, so the "decline"
  path was previously untested) with before/after cksum + file-count assertions, plus a
  NOGIT-sandbox pre-seed of the same stale-marketplace-key fixture the tracked-run sandbox
  already used, so the no-git path also exercises real `jq` mutation logic rather than an inert
  empty sandbox.

All new assertions were mutation-tested (5 mutations across the three files, each biting
exactly its intended check — verified by the loop's own grading pass, not restated here).

## Files changed

- `scripts/merge.sh` — two new `tempfile)` case arms (review gate, eval gate)
- `hooks/scripts/tests/merge.test.sh` — Test 3d + distinctness checks
- `hooks/scripts/tests/merge_evals_gate.test.sh` — Test 4d
- `hooks/scripts/tests/push_staging.test.sh` — 2 new fixtures
- `hooks/scripts/tests/loop_state_guard_evals.test.sh` — 1 new fixture, reader + e2e
- `hooks/scripts/tests/install_mode_sweep.test.sh` — conflict-decline fixture + NOGIT pre-seed

## Wiki pages updated

- [[merge]] — tempfile case arm documented alongside the existing identity/permission split
- [[trust-floor]] — `PR_TRUST_FETCH_FAIL_REASON` vocabulary extended to include `tempfile`
- [[task-evals-gate]] — cross-reference note only (the eval-gate tempfile arm shares the same
  fetch helper this page already documents)

`skills/sync-docs.md` was checked and requires no update — neither PR touches anything it
describes.

## Caveats / gotchas

**Declined-for-now, recorded not shipped:** a symmetry suggestion to give the eval-gate
tempfile arm a distinct wording pair beyond what Test 4d already asserts (i.e. an even finer
per-gate message split) was identified during Loop 2 planning and deliberately not shipped —
recorded here as the sixth suggestion-tier residual, still open. Not a bug; a style/verbosity
call left to a future pass.

**Test-comment correction:** the `loop_state_guard_evals.test.sh` fixture note about a prior
"(exercises final else)" comment being a misnomer is itself now a permanent code comment in
that file — not just a wiki note — so future readers of the test file see the correction
in-line.
