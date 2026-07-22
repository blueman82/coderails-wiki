---
title: "PR #265/#266/#268/#270/#271 — SYSTEM VITALS LINT FINDINGS tile activated, then made fast"
type: source
created: 2026-07-22
last_updated: 2026-07-22
sources: []
tags: [source, dashboard, system-vitals, lint-findings, health, usage, performance, memo-cache, single-flight, loading-state, test-flake, wiki-lint]
---

# PR #265/#266/#268/#270/#271 — SYSTEM VITALS LINT FINDINGS tile activated, then made fast

<!-- Ingested by /wiki-ingest after merge. This is an immutable record of what changed. -->

## PR metadata

| Field | Value |
|---|---|
| PR #265 | `fix/dashboard-vitals-lint-tile` — merge `de483b2a`, 2026-07-22 |
| PR #266 | `fix/health-test-flake` — merge `b951d983`, 2026-07-22 |
| PR #268 | `fix/health-test-cleanup-safety` — merge `c088f3c8`, 2026-07-22 |
| PR #270 | `docs/dashboard-loading-state` — merge `6b917645`, 2026-07-22 |
| PR #271 | `perf/usage memo cache` — direct commit `cdfbf759` on `main`, 2026-07-22 (no merge commit; commit message cites `(#271)`) |

## Summary

Five PRs, same day, one coherent arc: the SYSTEM VITALS LINT FINDINGS tile went
from a permanently-dead stub to a real collector (#265), a flake that #265's own
test suite introduced was fixed (#266), a crash-safety bug in that fix was fixed
(#268), the resulting three-tile-state UI was documented (#270), and — separately
but in the same cluster — the pre-existing usage-window collector's per-refresh
full re-parse was fixed for performance (#271).

### #265 — LINT FINDINGS: dead tile to real collector

`skills/dashboard/app/src/lib/collect/health.ts` previously hardcoded
`{key:"lintFindings", value:null, note:"unavailable: wiki-lint persists no report
file"}` unconditionally. That premise was false: `skills/wiki-lint/SKILL.md`
Step 5 has always appended `## [YYYY-MM-DD] lint | <summary>` to `$vault/log.md`
— the file existed, nothing read it `(verified, git diff de483b2a^1 de483b2a --
skills/dashboard/app/src/lib/collect/health.ts)`.

Fix: new `skills/dashboard/app/src/lib/collect/lint.ts` exporting
`collectLintFindings(vaultPaths, now)`. It parses `log.md` by splitting on the
`## [YYYY-MM-DD] lint | ...` heading regex, attributes each heading's own body up
to the next heading, and reads a structured `<!-- lint-findings: N -->` record if
present in that entry, selecting the entry by **max date**, not file position
(`mostRecentLintEntry`) `(verified, lint.ts)`. Falls back to `"Nd since last
lint"` (recency) when no structured record exists for the latest entry, with the
day count clamped to 0 for a future-dated heading (clock skew / hand-edited log)
rather than showing a negative number `(verified, lint.ts:daysSince)`.

**Deliberate design rule, stated in the source comment**: the prose summary text
after the heading is NEVER regex-scanned for a count. A paragraph saying "999
orphan links" would otherwise surface 999 as though it were a real, current
count `(verified, lint.ts` top-of-file comment)`.

Companion change: `skills/wiki-lint/SKILL.md` Step 5 now instructs appending the
`<!-- lint-findings: N -->` line immediately after the summary heading, N being
the sum of every finding category from Step 2 (contradictions + stale pages +
orphan pages + missing concepts + missing cross-references + data gaps + inbox
backlog items), 0 if the pass was clean. Explicitly does not change what's
reported to the user in Step 3, and is never derived by re-parsing the prose
`(verified, git diff de483b2a^1 de483b2a -- skills/wiki-lint/SKILL.md)`.

**Vault path source**: `collectLintFindings` takes `vaultPaths: string[]`, which
`health.ts` resolves from `~/.claude/coderails-dashboard.json`'s `wikiPaths[]`
array via `src/lib/config.ts`'s `loadConfig()` — **not** the plugin-repo-scoped
`workflow.config.yaml`'s `wiki_path` field, which is a distinct, easily-confused
name `(verified, ~/.claude/coderails-dashboard.json` on this machine holds
`wikiPaths: ["/Users/harrison/Github/coderails-wiki"]`; `config.ts` declares
`wikiPaths: string[]` on `DashboardConfig)`.

**Third tile state (`RailLeft.tsx`)**: previously a tile with no data and a tile
whose collector genuinely couldn't produce a value both rendered "unavailable,"
making a dashboard that just hadn't finished its first collect look identical to
a broken one. `collectHealth()` always returns all six `HealthTile` keys as a
fixed set (never a subset) once it resolves, so an empty `health` array
specifically means "the activity collect hasn't resolved yet" — the initial SSE
snapshot ships `health: []` before the first async collect finishes. `RailLeft`
now hoists `healthNotYetLoaded = health.length === 0` once, above the per-tile
map, and renders a `"loading…"` state for every tile when true, before falling
through to the existing `tile.value !== null` / "unavailable" branches
`(verified, git diff de483b2a^1 de483b2a -- ...RailLeft.tsx)`.

### #266 — fixed a flake #265 shipped

`health.test.ts`'s "never throws with no options (real default paths)" test
intermittently timed out at vitest's default 5000ms. Root cause: `collectHealth()`
called with no options walks the real `~/.claude/projects` tree (this machine:
~1.7GB, ~3750 `.jsonl` files) — ~1.6s idle, 3.6–5.5s observed under CPU
contention, so a fixed 5s budget was flaky by construction, not a logic bug.

Fix splits the test in two: a new deterministic-fixture test
(`makeTmpProjectsDir` + an unreadable `chmod 0o000` loops dir + malformed
JSONL/log lines) keeps the never-throws contract under controlled inputs; the
original no-options test is kept as a **separate**, minimal residual — its only
job is exercising `DEFAULT_PROJECTS_DIR`/`DEFAULT_LOOPS_DIR`/
`DEFAULT_DISCIPLINE_LOG_PATH` resolution — now at a 30s timeout, documented
inline as a **hang detector** ("is it ever going to resolve at all"), not a
speed gate `(verified, git diff b951d983^1 b951d983 -- .../health.test.ts)`.

### #268 — cleanup crash-safety fix on #266's own new test

The new fixture test `chmodSync`'d a temp dir to `0o000` and restored it
(`chmodSync(..., 0o755)`) as the **last statement**, after the `expect(...)`
assertion. If the assertion failed, the restore never ran, and `afterEach`'s
`rmSync(unreadableLoopsDir, { force: true })` **also** threw `EACCES` — `force`
suppresses missing-path errors only, not permission errors — turning one test
failure into two, plus an undeletable directory left on disk. Fixed by wrapping
the assertion in `try { ... } finally { chmodSync(..., 0o755); }`
`(verified, git diff c088f3c^1 c088f3c -- .../health.test.ts)`.

### #270 — documented the third tile state

`skills/dashboard/SKILL.md`'s SYSTEM VITALS bullet updated from "A source that
can't be read locally shows 'unavailable', never a guess" to state all three
states: "loading…" until the first activity collect resolves, then
"unavailable" for an honest collector failure, or the value once collected
`(verified, git diff 6b917645^1 6b917645 -- skills/dashboard/SKILL.md)`.

### #271 — usage collector performance fix (module-scope memo + single-flight)

Separate problem, same cluster: `collectUsage` (pre-existing, not part of the
#265 lint-tile work) re-parsed all in-window transcript files on every refresh —
measured on this machine at ~407MB across ~998 candidate files. A breakdown
isolated the cost: directory walk 94ms, stat prefilter 9ms,
streaming-with-zero-parsing 3,960ms — I/O and `readline` line iteration
dominate, not `JSON.parse`, so a parse-only optimization could not have closed
the gap. Churn measurement: only 6 of 998 files (1.16%) changed across a 20s
window; a separate 40s watch of all 1002 candidates found 7 files grew and 0
shrank, confirming transcripts are append-only in practice.

Fix (`skills/dashboard/app/src/lib/collect/usage.ts`):

- A module-scope `Map<path, {mtimeMs, size, events}>` (`fileMemo`). A cache hit
  requires **both** `mtimeMs` and `size` to match what was last read; on a miss
  the file is re-read and the entry replaced. Module scope, not per-aggregator,
  is deliberate: a `DashboardState` aggregator is constructed fresh per SSE
  connection, so a narrower-scoped memo would be cold on every browser tab and
  every reconnect `(verified, usage.ts` `fileMemo` comment)`.
- Memo entries whose path leaves the candidate set (aged out of the 7-day
  window, or deleted) are evicted every call, bounding memo growth
  `(verified, usage.ts, collectUsageUncached)`.
- Single-flight: `collectUsage` holds one `inFlight` promise at module scope;
  a second call arriving while one is in progress (e.g. two open browser tabs)
  awaits the same promise instead of re-reading every candidate file
  `(verified, usage.ts)`.
- A cheap prefilter, `line.includes('"type":"assistant"')`, skips
  `JSON.parse` entirely for the ~60% of lines that can never carry a usage
  object `(verified, usage.ts` `collectFileEvents)`.
- Result, measured: cold 5,824ms → warm 197ms (29.5×); totals verified
  byte-identical against the unmodified pre-#271 code on the same input.

**The cross-file dedup trap this PR had to get right.** Cross-file dedup (the
same `message.id` appearing in two different transcript files) is rebuilt from
scratch on **every** `collectUsage` call, over the current candidate set —
never persisted. If it were persisted: a message id first counted from file A
would keep suppressing that id from file B forever, even after A ages out of
the 7-day window and B is still in-window — a silent, permanent undercount that
grows over days. An implementation that gets this wrong passes every other
test in the suite. A dedicated test
(`usage.test.ts`, "keeps totals correct across a shared message id when the
first file ages out of the window (never silently persists cross-file dedup)")
covers exactly this case and was mutation-proved: swapping the merge-time fresh
`Set` for a persisted module-level `Set` makes the test fail, returning
`{inputTokens: 0, outputTokens: 0, totalTokens: 0, cacheReadTokens: 0}` instead
of `{10, 20, 30, 0}` `(verified, usage.ts` `mergeCandidateEvents` comment +
`usage.test.ts:389-429)`.

**Design alternative considered and rejected: SQLite.** A database was proposed
and rejected after a `/coderails:disconfirm` pass and an architecture review.
Reasons: the working set is ~36k events (7–14MB), needing no query engine or
durability guarantee; Node's built-in `node:sqlite` prints an
`ExperimentalWarning` on every start of what is a public plugin; its only real
benefit (warm cold-start) costs one non-blocking background rebuild instead of
adding a database; and the machine already has a recorded concurrent-write
JSON corruption incident (recorded as `feedback_loop_state_json_corruption` in
project memory, not a wiki page — plain text here, not a wiki-link, since it
resolves to nothing in this vault), so adding another corruptible file was
judged a net negative. The measured 1.16% per-20s churn is what made a plain in-memory
Map sufficient instead `(inferred — this rationale was reported by the
delegating session, not found written into the #271 commit message or PR body;
`gh pr view 271` shows only an auto-generated Edit-log body with no discussion
text. Recorded here as the stated design reasoning, not independently
re-derived from a review artifact)`.

## Files changed

- `skills/dashboard/app/src/lib/collect/lint.ts` (new, #265)
- `skills/dashboard/app/src/lib/collect/health.ts` (#265)
- `skills/dashboard/app/src/components/RailLeft.tsx` (#265)
- `skills/dashboard/app/src/styles/hud.css` (#265, `.hud-kpi-loading`)
- `skills/wiki-lint/SKILL.md` (#265, Step 5 structured record)
- `skills/dashboard/app/test/lint.test.ts`, `.../test/events.test.ts` (new, #265)
- `skills/dashboard/app/test/health.test.ts` (#265, #266, #268)
- `skills/dashboard/SKILL.md` (#270)
- `skills/dashboard/app/src/lib/collect/usage.ts` (#271)
- `skills/dashboard/app/test/usage.test.ts` (#271)

## Wiki pages updated

- [[dashboard]] — new "SYSTEM VITALS: LINT FINDINGS tile activated, then made
  fast" section

## Caveats / gotchas

- `wikiPaths` (per-user `~/.claude/coderails-dashboard.json`) is the config field
  `collectLintFindings` actually reads — not `workflow.config.yaml`'s
  `wiki_path`. Easy to confuse; the two are unrelated config surfaces for
  unrelated tools (dashboard vs. `/wiki-ingest` et al).
- PR #271 landed as a direct commit to `main` (`cdfbf759`), not a squash-merged
  PR — `gh pr view 271` returns a title and an auto-generated Edit-log body,
  no merge commit exists for it. Verified via `git log --oneline -1 cdfbf759`
  showing it as a direct, non-merge commit.
- The SQLite-rejected design rationale above is reported, not independently
  verified from a durable artifact — no review doc or PR description carries it
  on this repo as of this ingest.
