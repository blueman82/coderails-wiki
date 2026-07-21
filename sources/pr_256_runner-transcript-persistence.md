---
title: "PR #256 — sweeper run transcripts persisted to outputPath"
type: source
created: 2026-07-21
last_updated: 2026-07-21
sources: []
tags: [dashboard, runner, routines, diagnosability, run-log, tier-gate, self-edit, mutation-testing]
---

# PR #256 — sweeper run transcripts persisted to outputPath

<!-- Ingested by /wiki-ingest after merge. This is an immutable record of what changed. -->

## PR metadata

| Field | Value |
|---|---|
| PR | [#256](https://github.com/blueman82/coderails/pull/256) |
| Title | Persist routine run stdout/stderr to outputPath in exec.ts |
| Branch | `fix/exec-output-persistence` |
| Merged | 2026-07-21T15:05:59Z |
| Merge commit | `529c77c` |
| Head at merge | `f4493c6` |
| Tier | 1 |
| Files | `skills/dashboard/runner/src/exec.ts`, `src/sweep.ts`, `test/exec.test.ts`, `test/sweep.test.ts` |

## The defect

`runClaude()` in the runner returned a run's stdout/stderr **in memory only**. It never wrote
the `outputPath` that the sweeper had *already recorded in the run ledger* — `sweep.ts` builds
`startRecord.outputPath = join(runsDir, "<runId>.log")` and stores it, but nothing ever created
that file.

Consequence: a scheduled routine that ran RED left no transcript on disk, so the failure was
undiagnosable. The motivating observation was `sync-docs-nightly` showing RED with nothing to
inspect. The ledger pointed at a file that never existed.

This is an asymmetry with the live-run path, not a design choice: `app/src/app/api/run/route.ts`
already persisted its output to `outputPath`. Only the sweeper path was missing it.

## The fix

- `exec.ts` gains an optional `outputPath` on `ExecOptions` and a `persistOutput()` helper.
  The call sits **first inside the `execFile` callback, before any branch returns**, so it covers
  every settle path: success, non-zero exit, SIGKILL timeout, and ENOENT-style spawn failure.
- `sweep.ts` passes `{ outputPath: startRecord.outputPath }` — the same path already in the ledger.

Two deliberate design decisions, both documented inline:

1. **Plain text, not stream-json.** The task suggested `--output-format stream-json` for parity
   with `route.ts`. Rejected: those flags exist for `route.ts`'s live SSE path, which streams
   partial messages chunk-by-chunk. The sweep path uses buffered `execFile` with a single
   post-hoc write, and its sole consumer is a human diagnosing a failed routine, for whom
   readable text beats a JSON-lines stream. Verified safe with both readers — `extractResultText`
   falls back to the raw log, and `projectAssistantText`'s machinery-stripping fallback leaves
   plain sweeper text intact.
2. **`persistOutput` swallows its own write failures** (try/catch, `console.error`, no rethrow).
   Losing a transcript must never mask the `ExecResult` the caller needs to gate the run. An
   EACCES/ENOSPC escaping here would reject the promise the sweeper awaits, converting a real run
   outcome into a generic `runner-error` plus a quarantine — precisely the failure the comment
   names. This is a deliberate fail-open on the diagnostic side-channel, not an oversight.

## What review caught — and why it mattered

The PR shipped with 4 persistence tests and a green suite. Review found the tests did **not**
prove what they appeared to.

A first mutation (deleting the `persistOutput` call) turned 3 of 4 red, which looked like proof.
But a second mutation — **moving** the call to after the SIGKILL early-return, breaking
persistence on the timeout path outright — left all 125 tests green. Cause: `outputPath` appeared
in `exec.test.ts` only inside the new persistence block; the existing B4 timeout and ENOENT tests
never supplied it, so neither asserted a transcript.

Two further gaps followed from the same reading:

- Deleting `persistOutput`'s entire try/catch, so a write failure rejects the promise, also left
  125/125 green. The documented contract was asserted **in prose only**.
- The four `sweep.test.ts` assertions used `stringContaining(runsDir)`, pinning only the
  *directory*. Passing a wrong filename inside the right directory kept the suite green — which
  decouples the transcript from the ledger path, the exact failure this PR exists to prevent.

Three tests were added and one assertion tightened (to read the ledger back via `readRuns` and
pin the exact `join(runsDir, runId + ".log")`). No production-code change was required — the
implementation was correct throughout; only the evidence was missing.

Each fix is mutation-proved: every mutation above left the suite fully green before and fails it
now. Suite went 125 → 128.

## Tier-gate: the first live `self_edit` denial

PR 256 touches `skills/dashboard/`, which is on `TIER_GATE_PATH_DENYLIST`
(`scripts/tier-gate/tier-gate-runner.sh:768`). The daemon posted
`tier-review → FAILURE, verdict=self_edit` at both head SHAs, refusing to judge the PR at all —
the denylist check runs on the file list *before* any judging logic, so it never reached the
eval artifact.

This is the **first production firing of the DENY path**, which prior records noted had never
fired live. It worked as designed: the automation will not machine-approve a change to its own
control surface, and the merge decision went to the human.

Note the two signals were not in conflict. The eval artifact computed `GO` (all four evals pass)
while the gate said `self_edit`. The evals judge whether the change is sound; the denylist judges
whether a machine may be the one to approve it.

> ⚠️ Legibility defect surfaced by this PR: `self_edit` posts as `FAILURE` with an **empty status
> description**, visually identical to a broken build. It prompted a direct "are tests still
> failing?" question when they were not. Worth posting as neutral/pending, or filling the
> description with `verdict=self_edit — human merge required, not a test failure`. Any such fix
> would itself touch `scripts/tier-gate/` and so trip the same denylist — correctly.

See [[dashboard-runner]], [[routines]], [[pr_232_tier-review-gate]], and
[[tier-gate-path-denylist-dashboard_2026-07-21]] for why the dashboard is denylisted.
