---
title: "PR #287 — Context Trend panel, and the decoupled SSE frame it needed"
type: source
created: 2026-07-23
last_updated: 2026-07-23
sources:
  - skills/dashboard/app/src/lib/collect/contextTrend.ts
  - skills/dashboard/app/src/lib/collect/index.ts
  - skills/dashboard/app/src/app/api/events/route.ts
  - skills/dashboard/app/src/hooks/useDashboardState.ts
  - skills/dashboard/app/src/components/ContextTrendPanel.tsx
  - skills/dashboard/app/src/components/RailLeft.tsx
  - skills/dashboard/app/test/contextTrend.test.ts
  - skills/dashboard/app/test/useDashboardState.hook.test.tsx
  - skills/dashboard/app/test/events.test.ts
tags: [source, dashboard, sse, context-trend, token-reduction, first-paint, tri-state, stale-branch-revival, honest-statistics]
---

# PR #287 — Context Trend panel, and the decoupled SSE frame it needed

<!-- Ingested by /wiki-ingest after merge. This is an immutable record of what changed. -->

## PR metadata

| Field | Value |
|---|---|
| PR number | #287 |
| Branch | `feat/context-trend-panel` |
| Merged | 2026-07-23 (`2026-07-23T22:36:45Z`) |
| Merge SHA | `43af8046` |
| JIRA ticket | — |

## Summary

Adds a **Context Trend panel** to the dashboard's left rail: one dot per
agentic-loop session, x = session start date, y = orchestrator cache-read
tokens per assistant turn, with the 2026-07-17 token-reduction cutover drawn
as an annotation. 17 files, +1,518 lines. 599 tests green, lint clean,
`next build` compiles, and verified live on the running dashboard (verified —
PR body's verification section).

Two things make this more than "a new panel", and both are the compounding
knowledge:

1. **The panel's data path had to be re-architected before it could ship** —
   collecting it inline on the activity slice regressed
   [[pr_265_266_268_270_271_dashboard-vitals-lint-tile-and-usage-perf|#271]]'s
   first-paint by ~50×.
2. **The panel is deliberately built to refuse a conclusion** its own source
   audit never established.

## Why it was extracted rather than PR'd from its branch

The panel was unshipped work sitting on `worktree-token-work`, a branch **46
commits behind `main`** and jumbled with work that had already landed
separately (row-1 gate removal via #273, etc.). Only the panel was genuinely
unshipped, so it was extracted onto a fresh branch off `main` rather than
PR'd from the stale one (verified — PR body). This is the same disposition
recorded in the memory feedback *fresh branch + honest tier*: put a rebased
change on a fresh branch off `main` rather than force-pushing a stale one.

## The architectural decision: a slow collector gets its own SSE frame

The branch's original design called `await collectContextTrend(...)` **inside
`collectActivitySlice`**, so the whole activity frame — including the System
Vitals / KPI tiles — waited on it. `collectContextTrend` streams every
coderails-slug transcript under `projectsDir` (~3,900 files / ~1.9GB on this
machine), so on a cold cache the KPI tiles sat at "loading…" for ~10s,
regressing #271's ~0.2s first-paint by roughly 50× (verified — PR body).

The fix generalises into a rule the dashboard did not previously have to
state: **the single-SSE-provider rule (one connection, established by
[[pr_80-82_dashboard-stream-run-output-viewer|PRs #80–82]]) says one
*connection*, not one *frame*.** A collector whose latency class differs by
orders of magnitude from its neighbours gets its own event on that same
connection, so it never gates them.

Concretely (verified — `lib/collect/index.ts` at `43af8046`):

- New `refreshContextTrend()` and a `"context-trend"` aggregator event, added
  to `AggregatorEventName`, `AggregatorEventPayloadMap`, the
  `AggregatorEventListener` overloads and the `emit` overloads — so a call
  site emitting the wrong payload shape for the name is a compile error.
- `start()` fires `void refreshContextTrend()` **unconditionally and
  independent of `refreshActivity()`**, so a frame always arrives — including
  the `null` fallback when the collect errors. That is what stops the panel
  hanging in "loading" forever.
- `watchDir(deps.projectsDir, ...)` now schedules **both** an activity
  refresh and a separate `scheduleContextTrendRefresh()`, each on its own
  debounce timer, both cleared in `stop()`.

## The tri-state field, and why "loading" is the absence of a frame

`Snapshot.contextTrend` is `ContextTrendSummary | null | undefined`:

| Value | Meaning |
|---|---|
| `undefined` | the frame hasn't arrived yet — **loading** |
| `null` | source unreadable — **unavailable** |
| summary | data |

The design point: **the loading state is encoded as the absence of the frame,
not as a payload value** — the emitted payload is never `undefined`
(verified — the `AggregatorEventPayloadMap` comment). The panel therefore
keys off this field's own tri-state rather than off health's load signal,
which is what stops it flashing "unavailable" during the load window — the
same flash PR #283 removed for the KPI tiles. This is a second instance of
the three-state pattern
[[pr_265_266_268_270_271_dashboard-vitals-lint-tile-and-usage-perf|#265]]
introduced for the LINT FINDINGS tile (*loading* must be distinguishable from
*genuine collector failure*); #287 generalises it from a tile's array-emptiness
convention to a field-level tri-state on the snapshot itself.

## The wiring bug class unit tests structurally cannot catch

`SSE_EVENT_NAMES` in `useDashboardState.ts` is the array the hook loops over
to call `addEventListener`. An event name can be present in the
`DashboardEvent` union **and** correctly handled by `mergeDashboardEvent`, and
still be silently dropped by the browser because it was never registered.
Every unit test that calls `mergeDashboardEvent` directly passes anyway — the
defect lives only in the wiring.

This was found by **live-testing, not by the suite** (verified — PR body), and
is now guarded by a mutation-proven regression test in
`test/useDashboardState.hook.test.tsx`: a `FakeSource` records the listeners
the hook registers, asserts `listeners.has("context-trend")` **before** firing
anything, then fires a frame through the same `addEventListener` path
production uses. A companion test asserts a listener exists for all six names.

Generalises to: **a registration list is a seam, and a seam needs a test that
exercises the registration, not the handler.**

## The collector's measurement discipline

`contextTrend.ts` carries several deliberate choices worth keeping, each
guarding a specific misreading (verified — source comments at `43af8046`):

- **Session time is the first non-null message timestamp inside the jsonl —
  never file mtime.** The `remember` plugin rewrites transcripts on resume and
  bumps mtime; the audit documents a real session (`007e525b…`) that mtime
  would have misfiled across the cutover.
- **Orchestrator-only on both numerator and denominator.** Only top-level
  `<slug>/<sid>.jsonl` files are read; subagent transcripts under
  `<sid>/subagents/` are deliberately never opened. The audit's first pass
  divided a subagent-tree-pooled numerator by an orchestrator-only turn count
  and had to be corrected.
- **Cohort membership needs a marker AND structural corroboration** — a loop
  marker in the text plus an actually-existing `<sid>/subagents/` dir, because
  the marker alone matches any session that merely *mentioned* the skill.
  (Same marker-plus-corroboration shape as the `wiki_taxonomy_gate` vault
  identification in [[pr_285_wiki_taxonomy_gate_doc]] — inferred, the two were
  not written with reference to each other.)
- **No mtime prefilter**, unlike `usage.ts`'s rolling windows: old sessions
  stay in this population forever, so file recency says nothing about
  relevance. The cache, not a filter, is what keeps repeat collection cheap.
- **Cache eviction on disappearance**: entries for files gone from disk
  (deleted worktree dirs) are dropped each pass so the map tracks the live
  corpus rather than growing forever.

## Honest statistics: a panel built to refuse a headline

The audit's verdict is **INDETERMINATE**, and both the collector and the panel
are built to preserve that. The collector reports the raw per-session series
plus per-side summary stats (n, median, Q1, Q3) and **never computes a
headline saving**. The panel draws the cutover as an annotation the series
runs *straight through* — no gap, no colour change, no causal claim — and
shows per-side medians and n side by side rather than a single "saved X%".
Below `MIN_READABLE_N = 20` a side's median is drawn in the caveat colour and
said *in words* to be uncallable; the constant is explicitly labelled a
judgement constant, not statistics, and it only selects presentation — every
session stays plotted either way.

This is the [[trust-floor]] discipline applied to a chart: the presentation
layer is not allowed to assert more than the measurement supports. Compare
[[pr_265_266_268_270_271_dashboard-vitals-lint-tile-and-usage-perf|#265]]'s
refusal to regex-scan lint prose for a count — same principle, that a
plausible-looking number must not be manufactured from data that doesn't
carry it.

## The shared parse cache lives in the route, not module scope

`route.ts` declares `const sharedContextTrendCache: ContextTrendFileCache =
new Map()` and passes it into every aggregator via a new
`AggregatorDeps.contextTrendCache`. The collector has its own module-scope
`moduleCache` fallback, but the route passes an explicit reference because
**module-scope caches can be less reliable under production bundling**
(verified — the comment in `route.ts`). Aggregators are per-SSE-connection, so
without a shared cache each new browser tab would pay the full cold parse.
Note the divergence from #271, which relied on module scope alone for
`collectUsage` — #287 treats the explicit-reference form as the more robust
one.

## Files changed

Collector and wiring:
- `src/lib/collect/contextTrend.ts` (new, +294) — the collector
- `src/lib/collect/index.ts` (+58/−2) — `refreshContextTrend`, the
  `"context-trend"` event, the debounce timer, `contextTrendCache` dep
- `src/app/api/events/route.ts` (+11) — the shared cross-connection cache
- `src/hooks/useDashboardState.ts` (+15/−1) — tri-state snapshot field, the
  merge case, `SSE_EVENT_NAMES` registration

UI:
- `src/components/ContextTrendPanel.tsx` (new, +361) — hand-rolled SVG scatter
  (no chart library), cutover annotation, compaction track, hover readout
- `src/components/RailLeft.tsx` (+3/−1) — mounts the panel; adds a
  `data-testid="system-vitals"` handle so tests can assert the tiles paint
  independently of the panel

Tests (+~770 lines): `test/contextTrend.test.ts`,
`src/components/ContextTrendPanel.test.tsx`,
`test/useDashboardState.hook.test.tsx` (the wiring regression),
`test/events.test.ts`, `test/useDashboardState.test.ts`.

## Wiki pages updated

- [[dashboard]] — new "Context Trend panel" section; the decoupled-frame rule
  recorded alongside the existing single-SSE-provider note

## Caveats / gotchas

- **⚠️ The panel's spec document is not in the repo.** Both
  `contextTrend.ts` and `ContextTrendPanel.tsx` cite
  `docs/TOKEN-REDUCTION-AUDIT.md` as the thing they operationalise ("Renders
  docs/TOKEN-REDUCTION-AUDIT.md as a live panel"), but that path is **not
  tracked in git** (verified — `git ls-files | grep -i token-reduction`
  returns nothing at `43af8046`; no add-commit exists on any ref). The
  constants the collector hardcodes — `CUTOVER_MS`
  (`2026-07-17T20:22:00Z`, the #228/#229/#230 merge boundary),
  `WINDOW_START_MS` (`2026-07-07`), `MIN_READABLE_N = 20` — are therefore
  **freehand-looking to any future reader**, with their justification living
  only in an untracked file and in this page. Anyone re-tuning them has no
  in-repo source to check against. (See the memory feedback *freehand
  constants are claims*.)
- **The audit's method and verdict have no wiki page** — though the cutover
  itself does. PRs #228/#229/#230, the measures under audit, are covered by
  [[pr_228_229_230_token-burn-reduction-and-agents-split]] (corrected by lint
  2026-07-23: this page originally claimed the cutover had no coverage at
  all). `CUTOVER_MS` (`2026-07-17T20:22:00Z`) lines up with #228's merge at
  `20:22:29Z` (verified — `gh pr view 228`), so the constant is anchored to a
  documented event even though the audit that chose it is not. What remains
  genuinely uncovered is the audit's *method and INDETERMINATE verdict* —
  that lives only in the untracked `docs/TOKEN-REDUCTION-AUDIT.md` and in
  this page.
- **Debounce cadence is new load.** `scheduleContextTrendRefresh` fires on
  every `projectsDir` write at the 2s activity debounce, so during an active
  loop the collector re-stats every transcript every couple of seconds. The
  shared cache makes this incremental (stat sweep + reparse of only changed
  files), same cadence as `health` — acceptable, but new load on the LAN
  dashboard (see [[pr_179_dashboard-lan-access]]). Flagged non-blocking by the
  author, not by a reviewer.
- **Reconnect shows "loading…" again.** A reconnect builds a fresh aggregator
  with a fresh snapshot, resetting `contextTrend` to `undefined`. Consistent
  with health/activity on reconnect; the shared route-level cache means the
  data returns fast even though the state resets.
- **The cohort is coderails-only by construction.** `PROJECT_SLUG_TOKEN =
  "coderails"` filters project dirs by substring, so the panel is not a
  general token-trend view — it measures this project's loops only, and a
  project dir whose slug happened to contain "coderails" would be swept in
  (inferred — substring match, not exact).
- **Panel correctness is asserted, not gated.** The live verification (KPI
  tiles paint in ~2s while the panel still shows "loading…" — a state
  impossible on a shared frame, hence the decoupling proof) is recorded in the
  PR body as an author claim. No automated check enforces the first-paint
  budget, so a future inline-collect regression would not be caught by the
  suite.
