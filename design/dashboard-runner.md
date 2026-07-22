---
title: "Dashboard runner"
type: design
created: 2026-07-07
last_updated: 2026-07-22
sources:
  - sources/pr_36-41-33-53-65_verified-routines.md
  - sources/pr_201_202_203_routine-followups.md
  - sources/pr_240_lrp-last-marker-gate.md
  - sources/pr_256_runner-transcript-persistence.md
  - sources/pr_262_runner-stdin-stall.md
tags: [design, dashboard, runner, routines, artifact-gate, escalation, agentic-os, sub-project-2-of-5, last-marker, run-transcript, diagnosability, open-defects]
---

# Dashboard runner

`skills/dashboard/runner` — the one-shot sweeper that claims intents from the [[intent-queue-runner-contract|intent queue]], executes them, and — for routine-triggered intents — enforces the artifact gate. Shipped in [[pr_36-41-33-53-65_verified-routines|PR #41]].

## What it does, in order

Each sweep (`sweepOnce()`, `skills/dashboard/runner/src/sweep.ts`):

1. Ensures `processing/`, `archive/`, `quarantine/` exist (mode `0o700`).
2. Runs orphan recovery (see [[intent-queue-runner-contract]]'s Lifecycle section).
3. Lists `queue/*.json` and, for each file, atomically claims it via `renameSync` into `processing/`.
4. Parses the claimed file as an `Intent`; on parse failure or an unresolvable `button` name, quarantines it and moves on.
5. Looks up the intent's `button` in `DashboardConfig.buttons`, builds the argv via the shared `buildArgv`, and spawns `claude` (`runClaude`, `skills/dashboard/runner/src/exec.ts`).
6. If the claimed intent's button also resolves to a `RoutineDef` (by name or `buttonRef` — `findRoutine()`), evaluates that routine's artifact gate after a clean exit; otherwise a non-routine run gates on exit code alone.
7. Archives the processed file on success, or leaves the failure escalated (see below) and still archives it — quarantine is reserved for intents that couldn't even be parsed/resolved, not for intents that ran and failed.

## Per-intent failure boundary

**One bad intent quarantines, doesn't crash the sweep.** The entire claim-through-execute sequence for a single intent runs inside one `try`/`catch`; any uncaught exception — a malformed `buildArgv` input, an `appendRun` write failure, anything — results in a best-effort quarantine, a best-effort failed-run record, and a best-effort `runner-error` escalation, then the loop continues to the next file `(verified, skills/dashboard/runner/src/sweep.ts)`. This is deliberately separate from the claim step itself (losing a claim race to another sweeper instance isn't a failure of this intent — there's simply nothing left to process).

## Run transcript persistence

**Every settle path writes the run's stdout+stderr to the `outputPath` already in the ledger.**
`sweep.ts` builds `startRecord.outputPath = join(runsDir, "<runId>.log")` and records it; since
PR #256 (2026-07-21) `runClaude()` actually creates that file. The `persistOutput()` call sits
first inside the `execFile` callback, **before any branch returns**, so it covers success,
non-zero exit, SIGKILL timeout, and ENOENT-style spawn failure alike
`(verified, skills/dashboard/runner/src/exec.ts)`.

Before that fix the output was returned in memory only, so a routine that ran RED left no
transcript on disk and the ledger pointed at a file that never existed — the failure was
undiagnosable. The motivating case was `sync-docs-nightly` showing RED with nothing to inspect.
Only the sweeper path had this gap; the live-run path in `app/src/app/api/run/route.ts` already
persisted its output.

Two deliberate properties:

- **Plain text, not stream-json.** The sweep path uses buffered `execFile` with a single post-hoc
  write, and its only consumer is a human diagnosing a failed routine. `route.ts`'s stream-json
  flags exist for its live SSE path and buy nothing here. Safe with both readers — `extractResultText`
  falls back to the raw log and `projectAssistantText`'s machinery-strip leaves plain text intact.
- **The write is fail-open by design.** `persistOutput` catches and logs its own failures rather
  than rethrowing. An escaping EACCES/ENOSPC would reject the promise the sweeper awaits at
  `sweep.ts`, converting a run's real outcome into a generic `runner-error` plus quarantine via
  the per-intent boundary above. Losing the transcript must never mask the `ExecResult` that
  gates the run. The cost: a persistently failing write degrades silently back to the original
  symptom, with a `console.error` line as the only signal.

The tests pin the exact ledger path, not merely a path under `runsDir` — asserting only the
directory would let the transcript land somewhere the ledger doesn't point, which is the original
failure in a new costume. See [[pr_256_runner-transcript-persistence]].

**Caveat: the transcript is 0 bytes exactly when you need it most.** `persistOutput` writes a
file on *every* settle path including SIGKILL timeout, but `claude -p` buffers all output until
completion, so a run killed at the 30-minute ceiling has an empty buffer and the persisted
"transcript" is 0 bytes `(inferred from the buffered-execFile behaviour, PR #262 investigation —
open runner defect #2 below)`. PR #256 made the timeout path write *a* file; it did not make that
file useful for the timeout case.

## `runClaude()` closes the child's stdin — and why `stdio` can't

`runClaude()` captures the child returned by `execFile` and calls `child.stdin.end()` to send
EOF immediately (PR #262, 2026-07-22). Without it, `execFile`'s default stdio hands the child a
stdin PIPE the parent never writes to and never closes; the `claude` CLI blocks on it and after
3 seconds emits `Warning: no stdin data received in 3s, proceeding without it` — a mandatory ~3s
stall on **every** scheduled routine run (measured 6.9s → 3.58s per invocation once closed).

**Passing `stdio: ["ignore", "pipe", "pipe"]` does NOT work** — `execFile` silently drops the
`stdio` option and always pipes all three fds `(verified 2026-07-22 — child.stdin is still an
open pipe)`. Ending the returned child's stdin is the only remedy. The stdout/stderr pipes stay
untouched: they are the [[dashboard-runner#Run transcript persistence|ExecResult and persisted
transcript]]. The first fix attempt used the `stdio` approach, passed the unit test, and left the
warning live — only an eval driving the real binary caught it (a recurring dashboard lesson: shape
checks can't prove delivery). See [[pr_262_runner-stdin-stall]].

## The artifact-gate predicate evaluator

`checkArtifact()` (`skills/dashboard/runner/src/artifactGate.ts`) is what makes a routine's success mean more than `claude` exiting 0. It supports four predicate kinds against a routine's `expectedArtifact` `(verified, artifactGate.ts, corrected 2026-07-18 — this section previously listed only three)`:

- `exists` — file present and fresh (within `maxAgeSeconds`), nothing more.
- `contains: marker` — file present, fresh, and contains a literal marker string, anywhere in the file.
- `last-marker: success/failures` — added by PR #227 (2026-07-17), a stricter successor to `contains` for **append-only logs that accumulate many runs' worth of terminal markers**. Scans the file's lines for anything matching the marker set (`success` ∪ `failures`, substring match) and grades the run on the **last** matching line — not the literal last line of the file, since a routine may append a non-terminal trailing note after its real terminal marker within the same run. Passes iff that last terminal marker is the success marker; a file with no terminal marker at all reads NOT passed with an explicit "no terminal marker" reason, distinguishable from "file does not exist." This exists because a plain `contains: success-marker` check false-passes an aborted run whenever an *earlier* run that same file already wrote the success marker — the stale line is still `contains`-true even though the current run failed. First applied to `docs-sync-nightly`'s run log (PR #227); a second routine, `loop-retro-promotion-weekly`, adopted it a day later (PR #240, `success: "run=ok"`, `failures: ["abort=", "delivery=started"]`) for the identical append-log false-green reason — see [[loop-retro-promotion]]. A sibling routine, `memory-consolidation-weekly`, was evaluated against the same defect class by PR #240 and correctly left on `exists`, because its artifact is one unconditional file per run rather than a shared append-only log — see [[memory-consolidation]].
- `json-field: path/value` — file parses as JSON and a dotted path resolves to exactly the given value.

`artifactPath` supports `{date}` (`YYYY-MM-DD`), `{runId}`, and `{vault}` (the first entry of `wikiPaths`) template tokens, substituted at check time. See [[routines]] for the full field contract and worked example.

**`{date}` is resolved in LOCAL time, not UTC (fixed 2026-07-17, PR #202).** `sweepOnce()` previously derived `{date}` via `new Date().toISOString().slice(0, 10)` — always UTC regardless of `process.env.TZ` — while the producer (the `claude -p` run) writes its artifact keyed to its own LOCAL calendar date. Between local midnight and the local UTC offset the two disagree, and a genuinely correct run was graded `artifact-gate-failed`. Fixed with a `localDateIso()` helper (TZ-aware `getFullYear`/`getMonth`/`getDate` accessors) sourced from a new optional `clock?: () => Date` on `SweepOptions`, applied at the one `checkArtifact()` call site. Affects all four date-bearing gates, including `wiki-lint`'s own — its `{date}` lives in the `contains` predicate's **marker** text rather than its path, but `artifactGate.ts:92` resolves the token identically regardless of which predicate field it appears in. See [[pr_201_202_203_routine-followups]].

**Vault-root path-traversal defense**: the evaluator's path substitution is bounded so a crafted `{vault}`-relative path can't escape the vault root `(inferred from the source page's characterisation; not independently re-derived from `artifactGate.ts` for this page)`.

## Escalation taxonomy

Every failure is classified into exactly one `FailureClass` (`skills/dashboard/runner/src/escalate.ts`), always paired with a free-text `reason`:

| Failure class | Fires when |
|---|---|
| `skill-missing` | The routine's `foreignSkillPath` doesn't exist on disk — checked before spawning `claude` at all. |
| `claude-spawn-failed` | No `claude` binary found, or `execFile` itself failed to spawn (e.g. bad `cwd`). The process never started. |
| `exec-error` | `claude` spawned and exited non-zero, or was killed for exceeding the 30-minute timeout (reason text says "timeout" explicitly in that case). |
| `artifact-gate-failed` | `claude` exited 0 but `checkArtifact()` failed — missing, stale, or predicate mismatch. The routine "succeeded" by exit code and still failed. |
| `runner-error` | A failure in the runner's own bookkeeping (or a seed-time misconfiguration — unrecognised cadence, unresolvable `buttonRef`), not the routine's own execution. Also covers orphan recovery. |

`(verified, skills/dashboard/runner/src/escalate.ts` and `docs/routines.md)`.

Two channels fire per escalation, each independently try/caught so one channel's failure can't take out the other: a macOS notification (`osascript`, title `Routine failed: <routine name>`, body `<failure class>: <reason>`) and — if a vault is configured — a run note (`writeRunNote`, `<vault>/dashboard-runs/<routine>.md`, `type: routine-run` frontmatter, one `## [date] run <id> — green|red` section appended per run, for both successes and failures). Escalation itself must never throw: it runs inside the per-intent failure boundary, and a broken notification channel must not take down the run record `(verified, skills/dashboard/runner/src/escalate.ts)`.

**No deduplication, by design.** A permanently misconfigured routine re-escalates in full on every calendar fire, forever — accepted deliberately because a silently-swallowed misconfiguration was judged worse than repeated noise, and dedup would add state (what counts as "the same" failure, when to expire it) not worth building until the noise itself becomes the problem `(verified, docs/routines.md)`.

## Injection-hardening in the notification path

`defaultNotify()`'s `osascript` invocation passes `title`/`message` as trailing argv elements via AppleScript's `on run argv`, never interpolated into the AppleScript source string. This matters because an escalation `reason` can originate from artifact-derived text (e.g. an artifact-gate failure reason) — a real injection surface, not just defense-in-depth. Verified live in-repo (2026-07-07): a reason string containing an attempted `"; do shell script "..."` breakout produced a plain notification and did not execute the injected command `(verified, skills/dashboard/runner/src/escalate.ts` code comment)`.

## Exit contract

`main.ts`'s `run()` returns one of three codes: **0** clean sweep, **1** one or more routine failures (`result.failed > 0`), **2** the sweeper itself crashed before or during producing a `SweepResult` at all — judged strictly worse than "some routines failed," since even the failure bookkeeping in that case isn't trustworthy. A last-resort crash notification fires on exit 2, itself guarded so a broken notification channel can't mask the real crash `(verified, skills/dashboard/runner/src/main.ts)`.

`bin/sweeper.sh` runs `src/main.ts` directly via Node 24's built-in TypeScript type-stripping (no build step) — a fix within this same PR cluster, since the script's previous `../dist/main.js` target never existed (`dist/` is gitignored, no build step produces it), so every watch-plist fire had been dying `MODULE_NOT_FOUND` `(verified, skills/dashboard/runner/bin/sweeper.sh` code comment)`.

## Known open runner defects

The stdin stall (above) was ONE defect surfaced by an investigation into why a nightly
`docs-sync` run produced no output. **The runner is not healthy after PR #262** — five related
defects were found and left OPEN `(inferred — recorded from the PR #262 investigation notes, not
independently re-derived here; see [[pr_262_runner-stdin-stall]])`:

1. **30-minute SIGKILL timeout writes no terminal marker.** `DEFAULT_TIMEOUT_MS` kills a hung
   run, but nothing records a terminal marker on the kill — the outcome is legible only as a
   generic timeout.
2. **Killed runs leave a 0-byte output log** — see the transcript-persistence caveat above.
3. **The 1 MB `maxBuffer` default TERMINATES runs mid-execution**, not just truncates — an
   `execFile` whose combined output exceeds the default is killed with an error, so a chatty
   routine dies partway rather than completing.
4. **The same open-stdin bug lives in the dashboard web path**
   (`skills/dashboard/app/src/app/api/run/route.ts`) — PR #262 fixed only this sweeper path.
5. **Routines exit 0 without writing their run-log artifact** — the dominant observed failure
   (**6 red / 3 green on `docs-sync`** `(inferred — run ledger not read for this ingest)`). The
   artifact gate above exists to *catch* this class; the underlying exit-0-with-no-artifact
   behaviour is unfixed.

## See also

- [[intent-queue-runner-contract]] — the schema and lifecycle this runner consumes
- [[routines]] — the scheduling convention (cadence, config shape, launchd wiring) that seeds intents this runner processes
- [[memory-consolidation]] — one of the three shipped routines, a worked example of a skill that writes its own gate-checkable artifact
- [[dashboard]] — sub-project 1; owns `buildArgv` and the button model this runner reuses rather than re-implementing
- [[loop-retro-promotion]] — a worked example of `last-marker` adopted for its own routine's false-green fix (PR #240)
- [[pr_36-41-33-53-65_verified-routines]] — the source record for this page
- [[pr_201_202_203_routine-followups]] — PR #202, the UTC/local `{date}` skew fix
- [[pr_240_lrp-last-marker-gate]] — the predicate-kind staleness correction on this page (three kinds → four, `last-marker` described) and the second `last-marker` application
- [[pr_256_runner-transcript-persistence]] — the run-transcript persistence source record
- [[pr_262_runner-stdin-stall]] — the stdin-stall fix and the five open runner defects catalogued above
