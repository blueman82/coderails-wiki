---
title: "Investigation: dashboard run/events architecture and the streaming output-viewer gap"
type: investigation
created: 2026-07-07
last_updated: 2026-07-16
status: closed
sources:
  - skills/dashboard/app/src/app/api/run/route.ts
  - skills/dashboard/app/src/app/api/events/route.ts
  - skills/dashboard/app/src/lib/runlog.ts
  - skills/dashboard/app/src/lib/collect/index.ts
  - skills/dashboard/app/src/hooks/useRunLifecycle.ts
  - docs/coderails/specs/2026-07-06-observability-dashboard-design.md
  - sources/pr_138_remove-specs-plans-tracking.md
tags: [investigation, dashboard, sse, run-log, streaming, gap-analysis]
---

# Investigation: dashboard run/events architecture and the streaming output-viewer gap

> ## CLOSED 2026-07-08
> This gap is closed. [[pr_80-82_dashboard-stream-run-output-viewer]] (PRs #80–82, merged
> 2026-07-08) shipped exactly the streaming output-viewer this page said was greenfield,
> unbuilt scope, and it respects every cross-PR constraint listed below — verified directly
> against the merged source, not just the PR description: the log is now written
> incrementally during the run (not once, post-exit); `outputPath` is now read by a new
> `GET /api/run/output` route instead of being dead data; run output rides the existing
> `/api/events` connection as a new `"run-output"` event rather than opening a second SSE
> stream; `runId` gained the strict-format check this page's constraint #2 called for at the
> new client-supplied-query-param trust boundary, and the new route goes further by never
> joining `runId` into a path at all; the new `streamJson.ts` parser is fully non-throwing
> per constraint #4; and the run token is still never present in a response body per
> constraint #5. See the Resolution section below for the point-by-point mapping.

Filed in response to a query asking what the wiki covers about `/api/run`,
`/api/events`, and the run-log artifact model (`runs.jsonl`, `runs/<runId>.log`),
and what a plan to add a streaming output-viewer would need to know that isn't
already documented.

## Finding: no wiki page documents the run-log artifact model as its own topic

[[dashboard]] and its source pages ([[pr_25_observability-dashboard]],
[[pr_70-71_2026-07-07_dashboard-input-fix-and-voice-announcements]]) describe
the run/button model and the SSE stream architecturally, but none describe
`runs.jsonl` / `<runId>.log` as a distinct artifact-model topic, and none
mention a log/output viewer as planned, in-progress, or rejected scope. This
page fills that gap.

## Finding: the per-run `.log` file is write-only — nothing reads it

Verified directly against source (2026-07-07):

- `route.ts` (`POST /api/run`) creates `runs/<runId>.log` and does
  `appendFileSync(outputPath, stdout + stderr)` exactly **once**, inside the
  `execFile` callback — i.e. only after the child process has fully exited.
  This is a batch write, not incremental/streaming.
- `RunRecord.outputPath` (defined in `lib/runlog.ts`) is stored as a string
  field in `runs.jsonl` but is otherwise inert. A repo-wide grep for
  `outputPath` returns exactly two non-test hits: the write site in
  `route.ts` and the type definition in `runlog.ts`. No route, collector, or
  component ever reads the path or opens the file.
- `readRuns()` (the only reader of `runs.jsonl`) folds to one `RunRecord` per
  `runId` (newest line wins) and returns only the metadata fields
  (`runId, button, argv, cwd, profile, startedAt, endedAt?, exitCode?,
  outputPath`) — never file contents.
- `useRunLifecycle.ts`, the sole frontend consumer of `RunRecord[]`, uses it
  only to derive active-run hue/progress for the R3F sphere
  (`deriveActiveRuns` filters on `endedAt === undefined`). No component
  anywhere renders run output text.

## Finding: the recursive `fs.watch` on `runsDir` is a red herring for "already streaming"

`lib/collect/index.ts`'s aggregator does
`watch(runsDir, { recursive: true }, onChange)`, which does fire when
`<runId>.log` is written (since it's inside `runsDir`) — but the `onChange`
handler only calls `refreshRuns()`, i.e. re-reads `runs.jsonl` and emits a
`"runs"` SSE event with the metadata array. **The log file's bytes are never
read, never attached to the `Snapshot`, and never emitted over SSE.** A plan
should not assume any part of this path already streams output — the watch
firing on log writes is a coincidental one-shot "something in runsDir
changed" signal, not a content-streaming mechanism. Because the log write
happens post-exit, this watch fires at most once per run, at the same moment
the finish-line `runs.jsonl` write already fires it.

## Cross-PR constraints a streaming-viewer plan must respect

1. **Origin/Host guard on every route.** Both `/api/run` (POST) and
   `/api/events` (GET) call `isLocalOrigin(request)` and reject non-local
   Origin/Host with 403. A new log-read endpoint must do the same — this is
   the dashboard's baseline network-exposure control, not optional per-route
   hardening.
2. **Strict ID validation precedent.** [[pr_31_assistant-link-approve-button]]
   shipped a Critical path-traversal bug via an unvalidated `hash` param in
   `resolveQueueEntry`, fixed with a `/^[0-9a-f]{64}$/` check at two layers
   (API route + mutator). `runId` today is server-generated
   (`randomBytes(8).toString("hex")`, 16 hex chars) and never attacker-input,
   but the moment a plan adds a `GET /api/run-log?runId=...`-shaped endpoint,
   `runId` becomes client-supplied and needs the equivalent strict-format
   check before it's joined into a filesystem path.
3. **Single-SSE-provider requirement.** PR #25's own fix-loop record
   explicitly lists a "single-SSE-provider requirement" as one of five
   Critical/High findings closed before merge (see [[pr_25_observability-dashboard]]
   and [[dashboard]]'s Process record section). A streaming-viewer plan must
   not introduce a second competing EventSource/stream; new event types
   should extend the existing `/api/events` aggregator (`AggregatorEventName`
   currently `"runs" | "gates" | "activity"`) rather than opening a parallel
   channel per run.
4. **Collector never-throw discipline.** Every existing collector call inside
   the aggregator is wrapped (`safeCall`) so a throw degrades only that slice
   of the snapshot. A new log-tailing collector/source must follow the same
   pattern — a malformed or mid-write log file must not kill the whole SSE
   connection.
5. **Token/CSRF non-leakage.** The run token is deliberately never included in
   any API response body — only server-rendered into the page
   (`runlog.ts`'s `mintToken`/`getRunToken` comments are explicit about this,
   motivated by a real cross-module-graph bug found and fixed 2026-07-06). A
   log-viewer path must not require the token in a URL query string in a way
   that could land in access logs, and must not echo it back in any JSON body.
6. **Long-lived-POST lock coupling.** `route.ts`'s own comment states the POST
   response is "intentionally long-lived... don't convert this to
   fire-and-forget without redesigning the lock's release semantics" — the
   per-button lock is held for the child process's entire lifetime and
   released only in the `execFile` callback. A streaming-viewer feature that
   wants to show output *during* a run (not just after) needs a design for
   how a second reader (SSE tail) observes progress independently of that
   long-lived POST, since the POST itself blocks until exit and currently
   owns the only write to the log file.
7. **Design spec silence.** (⚠️ The spec cited here was deleted from the repo by
   [[pr_138_remove-specs-plans-tracking]] on 2026-07-11, after this investigation
   was filed; the observation is preserved as historical record.)
   `docs/coderails/specs/2026-07-06-observability-dashboard-design.md`
   lists "tail hook logs" only for the SYSTEM VITALS/health source
   (`~/.claude/discipline.log`) — an unrelated file — and never mentions
   run-output tailing as in-scope, deferred, or rejected. This is unbuilt
   scope, not a reversed decision; there is no supersession to reconcile.

## What a plan should NOT assume

- That `/api/events` already carries any run-output content (it doesn't — see
  above).
- That `fs.watch` on `runsDir` gives incremental tailing (it fires once,
  post-exit, and only triggers a metadata re-read).
- That `outputPath` is consumed anywhere today (it is dead data outside the
  write site).
- That adding a read endpoint for `<runId>.log` is safe without new `runId`
  format validation, simply because `runId` "is server-generated" — the
  validation belongs at the new trust boundary (client-supplied query param),
  not inherited from the write path's generation method.
- That true incremental (during-run) streaming is achievable without also
  touching `route.ts`'s write timing (currently a single post-exit
  `appendFileSync`) — reading a file that is only ever written once at the
  end cannot produce a live-tailing experience; the write side needs to
  change too (e.g. stream `stdout`/`stderr` chunks to the log file as they
  arrive) for a "live output viewer" to be more than a "view the finished
  transcript after the fact" feature.

## Resolution

**Closed 2026-07-08 by [[pr_80-82_dashboard-stream-run-output-viewer]] (PRs #80–82).**
Point-by-point against this page's own findings and constraints, verified against the merged
source (not just the PR description):

- The write-only `.log` finding is resolved: `POST /api/run` now appends each stdout/stderr
  chunk to `runs/<runId>.log` as it arrives (incremental), not once post-exit.
- The dead-`outputPath` finding is resolved: a new `GET /api/run/output` route reads a
  finished run's `RunRecord.outputPath` to serve settled output.
- The `fs.watch`-is-not-streaming finding is superseded, not falsified: run output no longer
  relies on `fs.watch` at all — a new in-process pub/sub (`runOutputBus.ts`) carries output
  chunks directly from the spawn callback to the `/api/events` aggregator.
- Constraint #1 (Origin/Host guard) — the new `GET /api/run/output` route calls
  `isLocalOrigin(request)`, same as the existing routes.
- Constraint #2 (strict ID validation) — `GET /api/run/output` validates `runId` against
  `/^[0-9a-f]{16}$/` at the point it becomes a client-supplied query param, and goes further
  than the precedent required: it never joins `runId` into a path at all, instead looking up
  the matching `RunRecord`'s own server-written `outputPath`.
- Constraint #3 (single-SSE-provider) — respected explicitly and by design: `runOutputBus.ts`'s
  own header comment states run output rides the existing `/api/events` connection as a new
  `"run-output"` event rather than opening a second stream.
- Constraint #4 (collector never-throw discipline) — the new `streamJson.ts` parser is fully
  non-throwing (`parseStreamJsonLine` returns `{ok: false, raw}` rather than throwing on a
  malformed or unrecognised line), and a parse failure never affects what's appended/published.
- Constraint #5 (token/CSRF non-leakage) — `GET /api/run/output` takes `token` as a query
  param compared server-side only; it is never echoed back in a response body.
- Constraint #6 (long-lived-POST/lock coupling) — resolved by design: output now streams out
  of the same long-lived POST via the pub/sub, so a second reader (the SSE subscriber) observes
  progress during the run without requiring the POST itself to become fire-and-forget.
- Constraint #7 (design spec silence) — no longer applicable; this is now built, documented
  scope (`docs/REFERENCE.md`, `skills/dashboard/SKILL.md`, both updated by PR #82).

This page's investigative content is left otherwise unchanged as a historical record of the
gap analysis that motivated the fix.

## See also

- [[pr_80-82_dashboard-stream-run-output-viewer]] — the source page for the fix that closes this gap
- [[dashboard]] — the skill's consolidating page
- [[pr_25_observability-dashboard]] — original run/events/runlog implementation, the single-SSE-provider fix-loop finding
- [[pr_70-71_2026-07-07_dashboard-input-fix-and-voice-announcements]] — the argv merged-prompt fix and its "argv-shape tests cannot prove delivery" lesson, directly relevant to test design for a streaming feature (behavioral verification, not shape/structure checks)
- [[pr_31_assistant-link-approve-button]] — the path-traversal precedent motivating strict ID validation on any new file-read endpoint
