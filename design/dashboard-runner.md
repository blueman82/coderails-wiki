---
title: "Dashboard runner"
type: design
created: 2026-07-07
last_updated: 2026-07-07
sources:
  - sources/pr_36-41-33-53-65_verified-routines.md
  - sources/pr_201_202_203_routine-followups.md
tags: [design, dashboard, runner, routines, artifact-gate, escalation, agentic-os, sub-project-2-of-5]
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

## The artifact-gate predicate evaluator

`checkArtifact()` (`skills/dashboard/runner/src/artifactGate.ts`) is what makes a routine's success mean more than `claude` exiting 0. It supports three predicate kinds against a routine's `expectedArtifact`:

- `exists` — file present and fresh (within `maxAgeSeconds`), nothing more.
- `contains: marker` — file present, fresh, and contains a literal marker string.
- `json-field: path/value` — file parses as JSON and a dotted path resolves to exactly the given value.

`artifactPath` supports `{date}` (`YYYY-MM-DD`), `{runId}`, and `{vault}` (the first entry of `wikiPaths`) template tokens, substituted at check time. See [[routines]] for the full field contract and worked example.

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

## See also

- [[intent-queue-runner-contract]] — the schema and lifecycle this runner consumes
- [[routines]] — the scheduling convention (cadence, config shape, launchd wiring) that seeds intents this runner processes
- [[memory-consolidation]] — one of the three shipped routines, a worked example of a skill that writes its own gate-checkable artifact
- [[dashboard]] — sub-project 1; owns `buildArgv` and the button model this runner reuses rather than re-implementing
- [[pr_36-41-33-53-65_verified-routines]] — the source record for this page
