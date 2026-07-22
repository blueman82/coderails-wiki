---
title: "PR #262 — close child stdin so the claude CLI does not stall 3s per run"
type: source
created: 2026-07-22
last_updated: 2026-07-22
sources: []
tags: [dashboard, runner, routines, exec, execFile, stdin, stall, performance, open-defects]
---

# PR #262 — close child stdin so the claude CLI does not stall 3s per run

<!-- Ingested by /wiki-ingest after merge. This is an immutable record of what changed. -->

## PR metadata

| Field | Value |
|---|---|
| PR | [#262](https://github.com/blueman82/coderails/pull/262) |
| Title | fix(runner): close child stdin so the claude CLI does not stall 3s per run |
| Branch | `fix/runner-stdin-stall` |
| Merged | 2026-07-22T18:37:44Z |
| Merge commit | `416d90f` |
| Head at merge | `4209ff4` |
| Files | `skills/dashboard/runner/src/exec.ts` (+14/-1), `test/exec.test.ts` (+57) |

## The defect

`runClaude()` in `skills/dashboard/runner/src/exec.ts` spawns the `claude` CLI via
`execFile`, whose default stdio gives the child a **stdin PIPE the parent never writes to
and never closes**. The CLI waits on it and after 3 seconds emits to stderr:

```
Warning: no stdin data received in 3s, proceeding without it.
```

Every scheduled routine run paid a mandatory ~3-second stall.

The repo already knew about the *warning* — `app/src/lib/streamJson.ts` (commit fe421ae,
2026-07-12) filters this exact string out of dashboard output. That strips the symptom
downstream; this PR removes the cause.

## The fix

Capture the child returned by `execFile` and call `child.stdin.end()`, sending EOF
immediately so the CLI proceeds at once:

```ts
(child as { stdin?: { end?: () => void } | null } | undefined)?.stdin?.end?.();
```

stdout/stderr capture is deliberately untouched — they are the `ExecResult` and the
persisted transcript (see [[pr_256_runner-transcript-persistence|PR #256]]). The `.end()`
call is optional-chained because the injectable `execFileImpl` test seam returns no child
object.

## Caveats / gotchas

**`stdio: ["ignore", "pipe", "pipe"]` does NOT work.** `execFile` silently drops the `stdio`
option and always pipes all three fds — `child.stdin` is still an open pipe (verified
2026-07-22; a child running `cat` still blocks to timeout). Ending the returned child's
stdin is the only thing that works. This is the single non-obvious lesson of the PR: the
first fix attempt used the `stdio` approach, **made the unit test pass, and left the defect
live**. Only an end-to-end check driving the real `claude` binary caught it — the same
"argv-shape / config-shape checks can't prove delivery" lesson that recurs across the
dashboard sub-project (cf. [[pr_70-71_2026-07-07_dashboard-input-fix-and-voice-announcements|PR #70]],
[[pr_260_263_dashboard-security-review|PR #260/#263]]).

## Verification

- **End-to-end against the real `claude` binary**: stderr empty, `PONG` intact,
  **6.9s → 3.58s** per invocation.
- **Suite**: 130 passed (128 baseline + 2 new), `tsc --noEmit` clean.
- **Mutation-proven**: neutralising the fix line flips exactly 1 test red (129/130);
  restoring returns 130/130. The new integration test spawns `/bin/cat` and **hangs to
  timeout if stdin is left open** — an oracle independent of the unit-test seam that let the
  first (broken) `stdio` attempt pass.

## Five related runner defects remain OPEN

This fix landed inside a larger, unfinished investigation into why a nightly `docs-sync` run
produced no output. The stdin stall was ONE defect surfaced by that investigation; the runner
is **not** healthy after this PR. Five defects were found and left unfixed
`(inferred — recorded from the investigation notes, not independently re-derived here)`:

1. **30-minute SIGKILL timeout** (`DEFAULT_TIMEOUT_MS`) writes **no terminal marker** when it
   kills a run. The killed run's outcome is legible only as a generic timeout.
2. **Killed runs leave a 0-byte output log.** `claude -p` buffers all output until
   completion, so on SIGKILL there is nothing in the buffer to persist — [[pr_256_runner-transcript-persistence|PR #256]]'s
   `persistOutput` still writes a file on the timeout path, but that file is empty exactly
   when a transcript is most needed.
3. **1 MB `maxBuffer` default TERMINATES runs mid-execution** (not just truncates) — an
   `execFile` whose combined stdout/stderr exceeds the default `maxBuffer` is killed with an
   error, so a chatty routine dies partway through rather than running to completion.
4. **The same open-stdin bug lives in the dashboard web path**
   (`skills/dashboard/app/src/app/api/run/route.ts`) — this PR fixed only the sweeper path.
5. **Routines exit 0 without writing their run-log artifact** — the dominant observed failure
   (**6 red / 3 green on `docs-sync`** `(inferred — the run ledger was not read for this
   ingest)`). The artifact gate (see [[dashboard-runner]]) exists precisely to catch this
   class, but the underlying "exit 0 with no artifact" behaviour is unfixed.

## Wiki pages updated

- [[dashboard-runner]] — the `execFile`-drops-stdio gotcha added to a `runClaude()` note; a
  new **Known open runner defects** section listing all five above; the transcript-persistence
  section caveated for defect #2.
- [[docs-sync]] — the "Not yet exercised end-to-end" framing updated: the routine HAS since
  fired with a real failure mode (defect #5).

## Related

- [[enforcement-model]] — the sweeper runs from the working checkout, not an installed copy
  (established by [[pr_260_263_dashboard-security-review|PR #260/#263]] and
  [[pr_274_tier_gate_observability_fixes|PR #274]]), so this fix being *merged* does not prove
  it is *deployed* `(inferred — deployment state not verified in this ingest)`.
