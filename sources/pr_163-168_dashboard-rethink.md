---
title: "PR #163-168 — dashboard-rethink cluster"
type: source
created: 2026-07-14
last_updated: 2026-07-14
sources: []
tags: [dashboard, deck-trim, memory-trail-removal, multi-loop, gates-freshness, headless-exemption, clean-break, agentic-loop]
---

# PR #163-168 — dashboard-rethink cluster

<!-- Ingested by /wiki-ingest after merge. This is an immutable record of what changed. -->

One agentic-loop session (`dashboard-rethink`, session `0767967d`, authorised 2026-07-13 by Gary after a live-dashboard walkthrough surfaced five concrete problems in one sitting: half the deck buttons are theatre, two panels are dead weight, the loop panel shows a week-dead loop with a misleading label, gate state lags 2 minutes, and headless runs return discipline-hook repair text instead of answers). Six PRs shipped as independent work units against that one spec, all merged to `main` 2026-07-14.

## PR metadata

| PR | Work unit | Branch | Merge SHA (main) | Model |
|---|---|---|---|---|
| #163 | t1 deck trim | `rethink/t1-deck-trim` | `9ab336c` | sonnet |
| #166 | t2 remove Memory.Trail | `rethink/t2-remove-trail` | `db16d77` | sonnet |
| #165 | t3 multi-loop collector | `rethink/t3-multiloop-collector` | `ce79b08` | sonnet |
| #168 | t4 multi-loop panel + hero | `rethink/t4-multiloop-panel` | `967ad77` | opus |
| #164 | t5 gates freshness | `rethink/t5-gates-freshness` | `a2bcb57` | sonnet (fallback from haiku after 2 failed gate attempts) |
| #167 | t6 headless hook exemption | `rethink/t6-headless-exemption` | `16074a9` | sonnet |

A seventh work unit, t7 (why intermediate turns are missing from run logs despite `--include-partial-messages --verbose`), produced no PR by design — see the "t7 finding" section below.

## Summary

Five independent fixes to `skills/dashboard/`, sharing one design spec and one loop, each with its own review + eval artifact:

**t1 — deck trim (#163).** `DashboardButton` gained an optional `hidden?: boolean` field (verified: `skills/dashboard/app/src/lib/config.ts:19`); config validation rejects a non-boolean value (`config.ts:80-82`); the button-list filter drops any `hidden: true` entry before rendering (`config.ts:94`, `config.buttons.filter((b) => !b.hidden)`). Applied live (not in-repo, since `~/.claude/coderails-dashboard.json` is per-user config) to trim the deck from 7 to 3 visible buttons — DEEP RESEARCH / WIKI QUERY / ASK — hiding 4 routine-trigger buttons (`wiki-lint`, `sync-docs-weekly`, `memory-consolidation-weekly`, `loop-retro-promotion`) that stay in config as live `buttonRef` targets for the routines runner; hiding is a display-layer decision, not a removal.

**t2 — Memory.Trail removal (#166), clean-break.** Removed the Memory.Trail panel, its collector, and its state slice end-to-end, plus the dead Command-Deck run-history list. Also removed the orphaned `memoryPaths` config field from **both** `DashboardConfig` declarations (the app's and the shared lib's) and all fixtures/docs (verified: `memoryPaths`/`MemoryTrail` return zero hits anywhere under `skills/dashboard/app/src`). "Clean-break" disposition — no back-compat shim, no deprecated-but-present field.

**t3 — multi-loop collector rewrite (#165).** `LoopInfo` gained `title` (resolution chain: `loop` field → `authorising_prompt_raw` → slug fallback), `lastUpdatedMs` (`last_updated` field wins, falls back to file mtime), and `units[]` with a `status` union of `"done" | "in-flight" | "pending"`, replacing the old `unitTitles`/`done` boolean pair. Both `progress.json` shapes parse: the current keyed-object schema and the legacy array schema. This is the data layer #168 (t4) builds on.

**t4 — multi-loop Directives panel + hero (#168), clean-break.** One card per **live** loop instead of a single "active loop" concept. `LOOP_LIVE_WINDOW_MS = 60 * 60_000` (verified: `skills/dashboard/app/src/hooks/useDashboardState.ts:142`) — a loop counts as live if `status !== "complete"` and `now - lastUpdatedMs <= LOOP_LIVE_WINDOW_MS`; `liveLoops()`/`stalledLoops()` (same file, lines 147/155) partition the non-complete set on that window, both newest-first. `useNow(30_000)` (`useDashboardState.ts:167`, consumed by `RailLeft.tsx:75` and `BottomHero.tsx:8`) is a ticking-clock hook so "live" status re-evaluates every 30s without a page reload. Stalled loops render as a dim sub-list (no cards); each live card gets a `Live.N` header suffix, an evals-frozen footer, and a decisions sub-list. `BottomHero` now follows `liveLoops(...)[0]` (the freshest live loop) instead of a single tracked "active loop." `selectActiveLoop` is retired — clean-break, not deprecated.

**t5 — gates freshness (#164).** `DEFAULT_GATES_POLL_MS` dropped from 120s to 30s (verified: `skills/dashboard/app/src/lib/collect/index.ts:74`, `const DEFAULT_GATES_POLL_MS = 30_000`). A new `GATES_RUNS_DEBOUNCE_MS = 3_000` (same file, line 76) debounces a gates refresh off runs-directory changes (line 213: `setTimeout(() => void refreshGates(), GATES_RUNS_DEBOUNCE_MS)`) — so a run finishing now updates gate state on the dashboard within ~3s of the runs-dir write, on top of the 30s poll floor, rather than waiting up to 2 minutes. Shipped after 2 failed gate attempts on haiku; routed to sonnet as fallback per Phase 2.8's model-routing step.

**t6 — headless hook exemption (#167).** `CODERAILS_HEADLESS_RUN=1` is set in exactly one place — `skills/dashboard/app/src/app/api/run/route.ts`'s `spawn(...)` call (verified: `route.ts:257`, `env: { ...process.env, CODERAILS_HEADLESS_RUN: "1" }`, spread-preserving so no other env var is clobbered). Both `check_confidence_labels.sh` and `check_verify_loop.sh` check for it (verified: `check_confidence_labels.sh:27`, `check_verify_loop.sh:81`, both `if [ "${CODERAILS_HEADLESS_RUN:-}" = "1" ] && [ "$hook_event" = "Stop" ]`) and skip enforcement **only** on the `Stop` event — `SubagentStop` still blocks unconditionally regardless of the flag, pinning test H5 in both hook suites. Rationale: a headless `claude -p` run (the dashboard's own run route) has no interactive turn left to satisfy a repair-turn block, so without the exemption the gate would displace the run's actual answer with gate-repair text. `AGENTS.md`'s hook-map rows and its enforcement-ceilings bullet were updated to document the exemption and flag any future second set-site as a security finding (see [[enforcement-model]]).

## t7 finding (no PR)

Filed as `wu7-finding.md` in the loop-state dir, not a wiki source page by design (no code change). Root cause for 0-model-turn run logs on slash-command-as-prompt runs: the slash command executes as a CLI local command, so the *outer* session takes 0 model turns even though the write path (stream-json log capture) is faithful — nothing is silently dropped, the log is just legitimately empty of model-turn events for that run shape. This validates t6's premise that a headless run genuinely has no interactive repair turn available.

## Files changed

- `skills/dashboard/app/src/lib/config.ts` — `hidden?: boolean` field, validation, filter (#163)
- `~/.claude/coderails-dashboard.json` (per-user, not in repo) — 4 buttons flagged hidden (#163)
- Memory.Trail panel/collector/slice files, dead run-history list, `DashboardConfig` (app + lib) `memoryPaths` field, fixtures/docs — deleted (#166)
- `skills/dashboard/app/src/lib/collect/sessions.ts` (or equivalent loop collector) — `LoopInfo.title`/`lastUpdatedMs`/`units[]` (#165)
- `skills/dashboard/app/src/hooks/useDashboardState.ts` — `LOOP_LIVE_WINDOW_MS`, `liveLoops`, `stalledLoops`, `useNow` (#168)
- `skills/dashboard/app/src/components/RailLeft.tsx`, `BottomHero.tsx` — multi-loop card rendering, hero follows `liveLoops[0]` (#168)
- `skills/dashboard/app/src/lib/collect/index.ts` — `DEFAULT_GATES_POLL_MS` 120s→30s, `GATES_RUNS_DEBOUNCE_MS` 3s (#164)
- `skills/dashboard/app/src/app/api/run/route.ts` — `CODERAILS_HEADLESS_RUN=1` spawn env (#167)
- `hooks/scripts/check_confidence_labels.sh`, `hooks/scripts/check_verify_loop.sh` — Stop-only headless exemption (#167)
- `AGENTS.md` — hook-map rows + enforcement-ceilings bullet (#167)

## Wiki pages updated

- [[dashboard]] — deck trim, Memory.Trail removal, multi-loop Directives panel, gates freshness cadence, headless exemption all folded into the skill page
- [[enforcement-model]] — headless-run exemption ceiling note
- [[agentic-loop]] — no direct edit from this cluster; linked as the loop this session ran under

## Caveats / gotchas

- **t1's live-config change is not in this repo.** The `hidden: true` flags live in `~/.claude/coderails-dashboard.json`, a per-user file explicitly excluded from PR tracking. The wiki records the mechanism (the `hidden` field + filter), not the specific live button state, which can change without a PR.
- **t2 and t4 are both clean-break, not deprecate-then-remove.** No back-compat shims exist for `memoryPaths`, the Memory.Trail slice, or `selectActiveLoop` — a wiki reader should not look for migration paths that don't exist.
- **t5's cadence numbers are two independent constants, not one.** `DEFAULT_GATES_POLL_MS` (unconditional 30s poll) and `GATES_RUNS_DEBOUNCE_MS` (3s debounce triggered by runs-dir changes, additive on top of the poll) — a run finishing can update gate state in ~3s even mid-poll-interval; conflating the two into "gates refresh every 30s" would understate the debounced-event path.
- **t6's exemption is env-triggered inside the same trust domain as the hooks it exempts** — see [[enforcement-model]]'s existing "honest boundary" framing. It is not a privilege-separated bypass; a second set-site anywhere else in the codebase would be a security regression, not a legitimate extension, per `AGENTS.md`'s explicit note.
- **PR #164's merge SHA (`a2bcb57`) lands after #168's (`967ad77`) in `git log main`** despite #164 being work-unit t5 and #168 being t4 — merge order in this loop did not follow work-unit numbering; don't infer sequencing from the PR/unit numbers.
