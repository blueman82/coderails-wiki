---
title: "PR 118-123 — Self-Improving Loops (retro gate + dormant v2 promotion pipeline)"
type: source
created: 2026-07-09
last_updated: 2026-07-09
sources: []
tags: [agentic-loop, retro, standing-orders, loop-state, discipline-log, self-improvement, v2-dormant, graduation-predicate, phase-13]
---

# PR 118-123 — Self-Improving Loops

<!-- Ingested by /wiki-ingest after the cluster merged. Immutable record of what changed. -->

## What this cluster is

Closes the **record→read-back gap** in the agentic loop: coderails recorded loop outcomes exhaustively (`progress.json`, `evals.json`, Phase 13 audit, `discipline.log`) but nothing ever read them back. This cluster makes past-loop outcomes mechanically shape future loops — with no human in the feedback path — and lays a dormant v2 pipeline to eventually promote proven lessons into plugin source.

Six PRs, all merged to `main` (final `643e0ad`) 2026-07-09. Built as a single agentic-loop session (`15f29276`), the corpus's own first retro.

## The cluster

| PR | Change | Surface |
|---|---|---|
| #118 | `dc_mine_hook_blocks` — a discipline.log miner: per-session, per-hook `{events, flagged}` counts, fail-open `{}`. The one retro field the orchestrator can't have written itself (hook-authored). | `hooks/scripts/lib/discipline_common.sh` |
| #119 | `als_gate_retro_on_complete` — the **retro gate**: blocks a `LOOP-STOP: complete` declaration unless a parseable `retro.json` (`schema_version` 1) sits beside `progress.json`. Block-before-bump; case-normalised; fail-open on absent jq. | `hooks/scripts/lib/loop_state_common.sh`, `hooks/scripts/loop_stall_guard.sh` |
| #120 | Phase 13 write-contract + Phase 2 retro-intake + Phase 3/3a lesson-travel in the skill text. | `skills/agentic-loop/SKILL.md` |
| #121 | v2 **dormant** promotion pipeline + machine-maintained scaffold. | `skills/loop-retro-promotion/SKILL.md`, `skills/agentic-loop/learned-failure-modes.md` |
| #122 | `loop-retro-promotion-weekly` routine (wired via `buttonRef`, **not** the dead `skillCommand` field; `bypass` profile). | `examples/dashboard-config.json`, `docs/routines.md` |
| #123 | Docs sync (retro-gate clause in AGENTS/README/REFERENCE; skill count →34). | docs |

## Key architecture

- **retro.json** — per-loop, session-keyed, beside `progress.json`. Raw and **unscored by schema** (no verdict field). Carries `loop_stop_counts` verbatim (hook-owned), `hook_blocks` (mined), disposition record, evals result, artifacts, review themes.
- **standing-orders.md** — repo-keyed, machine-owned overlay (SKILL.md never changes; the overlay evolves). Maintenance is **additive-or-recurrence-only**: match→reset+append evidence; new→append; `loops_since_recurrence >= K` (K=5, constant in skill text) → **decay by MOVING to `standing-orders-decayed.md`** (tombstone, never delete — keeps the "clean decay" predicate observable). Read at Phase 2 intake, carried verbatim into worker prompts.
- **v2 graduation predicate (dormant)** — the promotion pipeline stays inert until, per repo key: ≥10 retros AND one full lesson lifecycle (added→recurred) AND one clean decay. All file-observable. Only then does it mine repo-agnostic lessons and ship them to `learned-failure-modes.md` through the full gate chain, manifest-locked to exactly that one file.

## Decisions worth keeping

- **rollback-on-regression was cut** — comparing a loop's metrics to a ledger median is a self-issued subtractive verdict, and at small n task-difficulty dominates the signal. Decay (a recurrence count) is observation, not adjudication. The feedback path is additive-or-recurrence-only, everywhere.
- **v2 writes exactly one repo file** (`learned-failure-modes.md`), a *separate* file from SKILL.md so the machine physically cannot edit gate logic — a mechanical rail, not a prompt rail.
- **Headless-hook boundary:** PreToolUse hooks don't fire under the routine's `claude -p`, so v2 merges ONLY via `/coderails:merge` (merge.sh's artifact gates are script-internal and survive `-p`); raw `gh pr merge` is prohibited in the skill text. The routine is the repo's first non-read-only (`bypass`) profile.

## Impact

The **retro gate is LIVE** — every agentic loop's `complete` now requires a retro.json (the feature gates its own completion; the shipping loop had to write its own retro to declare done). The **v2 pipeline ships dormant** — it will not fire for many loops (corpus starts at 1). The weekly routine ships in the example config; arming it is a per-machine step (copy the block into `~/.claude/coderails-dashboard.json`, fix absolute paths).

## Process notes (this loop's own retro)

The miner, run on the shipping session itself, showed the **orchestrator** was the top stop-hook tripper (`confidence_labels` flagged 36×, `verify_loop` 7×) — the system's first learned lesson is about its own author, and Phase 2 intake will now carry "pre-tag confidence + DNV in the same turn" forward. Other captured failure modes: silent-bypass via case/encoding mismatch (#119 Critical, the case-sensitive compare), decorative negative controls, invariants held by control-flow accident, committed≠pushed, worker Edit-tool writes leaking to the parent checkout, stale inherited config values.

## See also

- [[agentic-loop]] — the skill whose Phase 13 now writes retro.json + overlay
- [[loop_stall_guard]] — the hook that hosts the retro gate
- [[discipline-loop]] — hook composition
- [[spec-plan-progress-artifact-chain]] — loop-state artifact family (progress.json now has retro.json/standing-orders.md siblings)
