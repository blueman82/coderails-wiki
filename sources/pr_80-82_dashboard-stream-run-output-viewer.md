---
title: "PR #80–82 — dashboard streaming output-viewer"
type: source
created: 2026-07-08
last_updated: 2026-07-08
sources: []
tags: [source, dashboard, sse, run-log, streaming, security, agentic-loop]
---

# PR #80–82 — dashboard streaming output-viewer

Ingested by `/wiki-ingest` after merge. Immutable record of what changed.

This cluster closes the gap [[dashboard-run-log-streaming-viewer-gap_2026-07-07]] documented
the same day: that investigation found the per-run `.log` file was write-only (a single
post-exit `appendFileSync`), `RunRecord.outputPath` was dead data nothing read, and
`fs.watch` on `runsDir` was a one-shot red herring that never carried log content over SSE.
All three PRs below are one theme — a real live output-viewer — landed as three sequential
PRs in the same loop.

## PR metadata

| Field | Value |
|---|---|
| PR #80 | `feat/dashboard-stream-run-output`, merged 2026-07-08T06:42:57Z, SHA `2a49d05b2b834abc9a56e0fd7d74012138f44f77` |
| PR #81 | `feat/dashboard-output-viewer-panel`, merged 2026-07-08T07:31:36Z, SHA `f2371ea5df89cdf63173190c4c3198afe19bd77c` |
| PR #82 | `docs/dashboard-output-viewer-wiring`, merged 2026-07-08T07:50:54Z, SHA `a383f36b065c8bde25f8f23fb5f1b82c7edf825f` |
| Repo | `blueman82/coderails` |

## Summary

**PR #80 (backend).** `POST /api/run` (`skills/dashboard/app/src/app/api/run/route.ts`) now
splices `STREAM_JSON_FLAGS` (`["--output-format", "stream-json", "--include-partial-messages",
"--verbose"]`) immediately after `"-p"` before spawning `claude`, so the child process emits
structured JSONL rather than plain text — `--verbose` is required alongside
`--output-format stream-json` under `--print` (confirmed empirically: omitting it fails fast).
Each `stdout`/`stderr` chunk is now appended to `runs/<runId>.log` **and** published on a new
in-process pub/sub, `skills/dashboard/app/src/lib/runOutputBus.ts`
(`createRunOutputBus()`/the `runOutputBus` singleton) — a `Set<listener>` with a `publish`/
`subscribe` API. This is a clean break from the old model the investigation page documented:
one post-exit `appendFileSync(outputPath, stdout + stderr)` is gone; the log file is now
written incrementally, during the run.

`lib/collect/index.ts`'s aggregator subscribes to `runOutputBus` and re-emits each chunk as a
new `"run-output"` SSE event (`{runId, chunk}`) on the **existing** `/api/events` connection —
`AggregatorEventName` widened from `"runs" | "gates" | "activity"` to include `"run-output"`.
The bus's own header comment is explicit that this is deliberately **not** a second SSE
endpoint: "the repo rule is one SSE provider, so run output rides the existing `/api/events`
connection... instead of opening a new stream" — directly satisfying the investigation's
single-SSE-provider constraint (traced back to PR #25's own fix-loop finding).

A new forward-compatible, non-throwing line parser, `skills/dashboard/app/src/lib/streamJson.ts`
(`StreamJsonSplitter` + `parseStreamJsonLine`), splits arbitrary chunk boundaries into complete
JSONL lines and parses each one. Its header comment records that a real probe of this exact
invocation (2026-07-07) observed 19+ distinct `type` values across ~90 lines in one short run,
and the CLI gives no schema guarantee it has seen every type it will emit — so the parser
maintains no allowlist, and a malformed or unrecognised line comes back as `{ok: false, raw}`
rather than throwing. Parse results are **not currently consumed further** — `route.ts` parses
each line purely to prove it's at least well-formed-or-gracefully-skipped; what gets
appended/published is always the raw chunk text, so a parse failure can never crash the run or
drop output. This satisfies the investigation's "collector never-throw discipline" constraint.

**Critical fix from review:** the original implementation had no `child.on('error', ...)`
handler. Node fires `"error"` instead of `"exit"` when the process never launches at all
(e.g. `ENOENT` if `claude` isn't on `PATH`, `EACCES`, a bad `cwd`) — without that handler the
promise inside the POST handler never resolves (the request hangs forever) and the per-button
lock is never released (the button 409s until the 24h stale-lock TTL). Fixed by adding
`child.on("error", ...)` that logs the failure and funnels through the same `settle()` helper
the `"exit"` path uses, releasing the lock with `exitCode: -1`.

**PR #81 (frontend).** New `OutputViewerPanel.tsx`
(`skills/dashboard/app/src/components/OutputViewerPanel.tsx`) in the COMMAND DECK. A still-live
run streams from `DashboardState.runOutput` (a `Record<runId, string>` accumulated from the
`"run-output"` SSE event in `useDashboardState.ts`); a finished run's full output is instead
fetched once from a new route, `GET /api/run/output`
(`skills/dashboard/app/src/app/api/run/output/route.ts`).

That route's `RUN_ID_PATTERN` is `/^[0-9a-f]{16}$/` (`runId` is always exactly 16 lowercase hex
chars — `randomBytes(8).toString("hex")`, minted only in `api/run/route.ts`). Crucially it goes
one step further than format validation: **`runId` is never joined into a filesystem path at
all.** The route looks up the matching `RunRecord` in `runs.jsonl` by `runId` and reads *that
record's own* `outputPath` field — the only path ever opened is server-written, not client-
supplied — making it path-traversal-safe by construction rather than by regex alone. This
directly satisfies the investigation's "strict ID validation precedent" constraint (traced to
the PR #31 path-traversal fix) and goes further than that precedent required. The route returns
a discriminated body: `{status: "ok", output}` (200), `{status: "in-progress"}` (409, when
`record.endedAt` is still `undefined` — avoids handing back a partial read indistinguishable
from a genuinely short-but-complete run), or `{status: "error", error}`.

Run-history rows are clickable to switch which run's output is shown; `selectDefaultRunId()`
prefers a still-active run over the most recently started one.

**Critical fix from review:** the original `fetchSettledOutput` swallowed all fetch/parse
errors silently — a network failure or malformed response would render identically to "no
output yet." Fixed with a discriminated `SettledOutputResult` type (`{ok: true, output}` |
`{ok: false, kind: "in-progress"}` | `{ok: false, kind: "error", error}`), so network errors,
non-2xx responses, and malformed JSON each surface a distinct, visible error state in the panel
with a retry button, instead of collapsing into an empty viewer. Also added during review: the
`409`/`"in-progress"` signal itself as its own case (distinct from error — the client already
has the correct data via the live SSE buffer for this case and should keep using it rather than
trust a partial read); and pruning of the `runOutput` map in `useDashboardState.ts` so
accumulated live-chunk entries for runs that have aged out of the server's run-history window
(`runsLimit` in `lib/collect/index.ts`) don't leak forever — filtered against the current
`"runs"` snapshot's `runId` set on every `"runs"` event.

**PR #82 (docs).** `docs/REFERENCE.md` gained a "Run output viewer" paragraph under the
`dashboard` skill entry, and `skills/dashboard/SKILL.md`'s COMMAND DECK bullet (panel 4 of 7)
gained a clause: "plus a Run Output viewer: click any run-history row to view its output —
live-streaming while the run is still going, settled (fetched once) once it ends." Docs-only;
no code changed.

## Files changed

- `skills/dashboard/app/src/app/api/run/route.ts` — stream-json flag splicing, incremental
  chunk append + publish, `child.on("error", ...)` fix
- `skills/dashboard/app/src/lib/runOutputBus.ts` — new, in-process pub/sub
- `skills/dashboard/app/src/lib/streamJson.ts` — new, non-throwing forward-compatible parser
- `skills/dashboard/app/src/lib/collect/index.ts` — `AggregatorEventName` gains `"run-output"`,
  subscribes to `runOutputBus`
- `skills/dashboard/app/src/app/api/run/output/route.ts` — new `GET` route, runId-keyed lookup
  via `runs.jsonl`, never path-joins `runId`
- `skills/dashboard/app/src/components/OutputViewerPanel.tsx` — new COMMAND DECK panel
- `skills/dashboard/app/src/hooks/useDashboardState.ts` — `runOutput` map + accumulation +
  pruning
- `docs/REFERENCE.md`, `skills/dashboard/SKILL.md` — docs (PR #82)

## Wiki pages updated

- [[dashboard]] — COMMAND DECK / architecture section extended with the output-viewer
- [[dashboard-run-log-streaming-viewer-gap_2026-07-07]] — marked CLOSED, linked to this page

## Caveats / gotchas

- The stream-json parser's output is not consumed for anything beyond
  well-formed-or-gracefully-skipped verification today — no UI currently renders per-event-type
  structure (e.g. distinguishing an `assistant` chunk from a `tool_use` chunk); the panel
  renders accumulated raw chunk text as one opaque stream. A future pass could use the parsed
  `type` field for richer rendering, but nothing in this cluster requires it (verified: not
  read anywhere).
- `RUNS_LOOKUP_LIMIT = 10_000` in the output route is a generous cap for a lookup-by-id, not a
  pagination control — deliberately not importing the aggregator's smaller `runsLimit` constant
  since this route's job (find one record) is a different shape of read.
- Token continues to never appear in a response body — `GET /api/run/output` takes `token` as a
  query param compared server-side, same non-leakage discipline as the original investigation's
  constraint #5.
