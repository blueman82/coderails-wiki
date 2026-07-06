---
title: "work_units and loop_stop_counts: progress.json's two tracked fields"
type: design
created: 2026-07-06
last_updated: 2026-07-06
sources:
  - sources/pr_1-4_task-evals-feature.md
  - sources/pr_96-98_mode-aware-install-argument-injection-guard-hook-owned-counter.md
  - sources/pr_15-17_loop-hardening-registration-eval-freeze-ledger-dry.md
tags: [design, work-units, loop-stop-counts, progress-json, agentic-loop]
---

# work_units and loop_stop_counts: progress.json's two tracked fields

Both fields live inside the same `progress.json` object (see
[[spec-plan-progress-artifact-chain]]) but have opposite ownership models and
serve different consumers. This page is the single concept-level home for
both — the detail already lives correctly on the seven-plus pages that cite
them; this page maps the terrain rather than re-deriving it.

## Context

`work_units` and `loop_stop_counts` are easy to conflate because they sit
side by side in the same JSON object and are both read at loop boundaries.
They are not the same kind of fact:

- **`work_units`** is **orchestrator-owned**. It is a JSON object keyed by
  unit id, each entry carrying at least a `status`
  (`pending`/`in-progress`/`done`/`blocked` with `blockedBy`). The
  orchestrator writes it at every phase boundary as part of maintaining
  `progress.json` (per [[agentic-loop]]'s Context-window persistence
  section).
- **`loop_stop_counts`** is **HOOK-OWNED**. It is written solely by the
  `loop_stall_guard` hook on each valid `LOOP-STOP` declaration
  ([[pr_96-98_mode-aware-install-argument-injection-guard-hook-owned-counter|PR #98]]).
  The orchestrator never writes or increments it — only reads it, and must
  carry it forward verbatim on any wholesale rewrite of `progress.json` (the
  same treatment `completed_marker` gets).

Per the wiki's own 2026-07-06 full-vault-resweep entry (`log.md`), citations
of these two fields were spread across pages with no single concept-level
home: `work_units` across 7 pages (`hooks/loop_state_guard.md`,
`skills/task-evals.md`, `skills/writing-plans.md`,
`design/task-evals-gate.md`, `skills/subagent-driven-development.md`,
`skills/agentic-loop.md`, `design/spec-plan-progress-artifact-chain.md`) and
`loop_stop_counts` across 3 pages (mutually consistent per PR #98). This page
consolidates the concept; it does not replace the fuller per-page detail,
which stays exactly where it is.

## The rule

**`work_units` schema and consumer** (from
[[spec-plan-progress-artifact-chain]] / `skills/agentic-loop/SKILL.md`'s
Context-window persistence section): a JSON object keyed by unit id, each
entry at minimum `{"status": "pending"|"in-progress"|"done"|"blocked",
"blockedBy": [...]}`. The loop-scope eval gate
([[loop_state_guard]]) reads `.work_units | length` off this field to decide
whether the `>=3` work-unit eval threshold applies — **failing open (no
block) when the field is absent**, so the orchestrator must keep it
populated whenever the loop tracks one or more work-units.

**`loop_stop_counts` schema and writer**: a per-category tally object,
`{"hard-stop": N, "approval-gate": N, "awaiting-input": N, "complete": N}`,
built from `LOOP_STOP_VOCAB` (defined once in
`hooks/scripts/lib/loop_state_common.sh`, shared by both loop-state hooks so
the block message can never advertise a category the match regex rejects).
`loop_stall_guard.sh`'s `bump_loop_stop_count()` increments the relevant key
via a tmp-file `jq` read-modify-write on every valid declaration — best-effort
and never fatal: a missing file, malformed JSON, absent `jq`, or a failed
`mv` is logged and swallowed, and the hook still exits 0. Phase 13 of
[[agentic-loop]] reports the counts raw, unscored — no self-issued verdict,
per the honest-audit framing that section describes.

## Where it is enforced

- **`work_units`**: read by `als_read_work_units()`
  (`hooks/scripts/lib/loop_state_common.sh:138-145`), fail-open on
  absent/malformed input; consumed by the `>=3`-work-unit threshold gate in
  [[loop_state_guard]].
- **`loop_stop_counts`**: written exclusively by `bump_loop_stop_count()`
  (`hooks/scripts/loop_stall_guard.sh:52-58`, called at `:85` on every valid
  `LOOP-STOP` declaration) — see [[loop_stall_guard]] for the full mechanism
  and its four standing invariant tests (no-clobber, jq-absence fail-open,
  last-declaration-wins tie-break, degraded-filesystem safety).

## Known caveats / edge cases

- The eval-gate threshold silently does not apply if `work_units` is absent
  from `progress.json` — this is a deliberate fail-open, not a bug, but it
  means an orchestrator that forgets to populate the field also forgets to
  trigger the loop-scope eval requirement.
- `loop_stop_counts` must be carried forward verbatim on any wholesale
  `progress.json` rewrite. An orchestrator that reconstructs the file from
  scratch without re-reading the existing value will silently reset the
  counter to empty — the hook can't detect this, since it only ever adds to
  whatever key already exists.

## See also

- [[spec-plan-progress-artifact-chain]] — the full artifact chain
  (`spec.md` → `plan.md` → `progress.json`) this page's two fields sit inside;
  already has its own detailed "loop_stop_counts: hook-owned, sole writer
  (PR #98)" section
- [[agentic-loop]] — the skill that reads/writes `work_units` and reads
  `loop_stop_counts` at Phase 13
- [[loop_state_guard]] — the hook that reads `work_units` for the eval-gate
  threshold
- [[loop_stall_guard]] — the hook that is the sole writer of
  `loop_stop_counts`
- [[task-evals-gate]] — the dual-scope eval-gate design that
  `work_units.length` feeds
- [[writing-plans]] · [[subagent-driven-development]] · [[task-evals]] — the
  other pages citing `work_units` in task/ledger context
- [[pr_1-4_task-evals-feature]] — introduced the loop-scope eval gate reading
  `work_units`
- [[pr_96-98_mode-aware-install-argument-injection-guard-hook-owned-counter]]
  — PR #98: made `loop_stop_counts` hook-owned with a sole writer
- [[pr_15-17_loop-hardening-registration-eval-freeze-ledger-dry]] — PR #16:
  documents the `work_units`-vs-ledger fact-ownership split (ledger =
  task-level, controller-owned; `work_units` = unit-level, orchestrator-owned)
