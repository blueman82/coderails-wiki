---
title: "work_units, loop_stop_counts, and decisions_absorbed: progress.json's three tracked fields"
type: design
created: 2026-07-06
last_updated: 2026-07-12
sources:
  - sources/pr_1-4_task-evals-feature.md
  - sources/pr_96-98_mode-aware-install-argument-injection-guard-hook-owned-counter.md
  - sources/pr_15-17_loop-hardening-registration-eval-freeze-ledger-dry.md
  - sources/pr_95_slash-command-loop-detection.md
  - sources/pr_144-149_agentic-loop-hardening-from-loop-engineering.md
  - sources/pr_194_198_loop-complete-deferral-and-proof-gates.md
tags: [design, work-units, loop-stop-counts, decisions-absorbed, progress-json, agentic-loop, deferral-gate]
---

# work_units, loop_stop_counts, and decisions_absorbed: progress.json's three tracked fields

All three fields live inside the same `progress.json` object (see
[[spec-plan-progress-artifact-chain]]) but have different ownership models and
serve different consumers. This page is the single concept-level home for
all three — the detail already lives correctly on the pages that cite
them; this page maps the terrain rather than re-deriving it. `decisions_absorbed`
was added later ([[pr_144-149_agentic-loop-hardening-from-loop-engineering|PR #147]],
2026-07-12) and shares `work_units`' ownership model (orchestrator-owned, appended
at phase boundaries) rather than `loop_stop_counts`' (hook-owned).

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
  carry it forward on any wholesale rewrite of `progress.json` (the
  same treatment `completed_marker` gets) — **conditionally, not always
  verbatim**, as of [[pr_144-149_agentic-loop-hardening-from-loop-engineering|PR #147]]
  (2026-07-12): verbatim on a genuine mid-loop recovery rewrite, but reset to
  `{}` when the prior file's `status` was `"complete"` (a fresh re-arm after a
  prior loop already finished) — see "Known caveats" below.
- **`decisions_absorbed`** is **orchestrator-owned**, added by
  [[pr_144-149_agentic-loop-hardening-from-loop-engineering|PR #147]]
  (2026-07-12). A chronological (oldest-first) array of `{phase, decision}`
  objects, appended at the phase boundary where each in-scope autonomous
  decision is made — Phase 2.5 (design-fork auto-adopted), Phase 2.6
  (disposition defaulted), Phase 5 (a consciously-absorbed
  `/coderails:disconfirm` skip), Phase 6 (an in-scope action taken without a
  check-in). Phase 13's terminal "Decisions absorbed" self-audit bullet is
  specified as **copied verbatim** from this array — before this field
  existed, that bullet was reconstructed from conversation memory at
  teardown time, exactly the kind of after-the-fact self-report Phase 13
  otherwise exists to avoid. The `retro.json` teardown artifact carries the
  same array verbatim into its own field, by the same rule.

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
  [[loop_state_guard]]. Gained a **second consumer** with
  [[pr_194_198_loop-complete-deferral-and-proof-gates|PR #194]]
  (2026-07-17): `als_gate_work_units_on_complete` in [[loop_stall_guard]]
  reads the same field on a `LOOP-STOP: complete` declaration and blocks
  unless every entry is `"done"` or `"dropped"` with a non-empty
  `dropped_reason` — file-level absence still fails open (same posture as
  the threshold gate), but an individual unit that cannot be proven
  terminal now fails closed, which the threshold gate never did (it only
  counts entries, never judges their status).
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
- `loop_stop_counts` must be carried forward on any wholesale
  `progress.json` rewrite — but not unconditionally verbatim, as of
  [[pr_144-149_agentic-loop-hardening-from-loop-engineering|PR #147]]
  (2026-07-12). An orchestrator that reconstructs the file from scratch
  without re-reading the existing value will silently reset the counter to
  empty — the hook can't detect this, since it only ever adds to whatever
  key already exists. **The correct rule is conditional**: carry forward
  verbatim on a genuine mid-loop recovery rewrite, but reset to `{}` when the
  prior file's `status` was `"complete"` — a fresh re-arm after a prior loop
  already finished. Before this fix, a brand-new loop starting in the same
  repo/session-key slot could wrongly inherit an already-finished prior
  loop's stop-counts, since the pre-existing rule said "always carry forward
  verbatim" with no exception for the re-arm case.
- **`loop_stop_counts` could also silently stay null for a completely different reason: the guard never detected the loop at all** ([[pr_95_slash-command-loop-detection|PR #95]], 2026-07-08). If a loop was started via the slash-command form (`/coderails:agentic-loop`) rather than a programmatic `Skill` tool_use, `als_count_invocations` returned 0 invocations — the transcript records a slash-started loop as a `user`-role message with a string `.message.content` carrying `<command-name>...</command-name>`, a shape the pre-fix jq filter never matched. With 0 invocations, `als_gate_require_active_loop` exited before `als_load_progress`/`gate_loop_stop_declared` ever ran, so `bump_loop_stop_count` was never reached — the field stayed null for the loop's entire duration, with zero hook signal. This was NOT an `ALS_PATH` keying bug and NOT a tail-window blind spot (two earlier, disproven hypotheses) — it was a wholesale detection miss across the whole guard chain. Fixed by extending the count to also match the user `<command-name>` slash form.

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
- [[pr_95_slash-command-loop-detection]] — PR #95: fixes the slash-command
  detection gap that was one root cause of `loop_stop_counts` silently
  staying null (a wholesale guard-detection miss, not a keying bug)
- [[pr_144-149_agentic-loop-hardening-from-loop-engineering]] — PR #147
  (2026-07-12): adds `decisions_absorbed` as this page's third tracked field,
  and makes `loop_stop_counts` carry-forward conditional (reset to `{}` on
  re-arm after a completed loop) instead of unconditionally verbatim
- [[dashboard]] — the RailLeft Directives card that surfaces the last 5
  `decisions_absorbed` entries (PR #148/#149)
- [[pr_194_198_loop-complete-deferral-and-proof-gates]] — PR #194
  (2026-07-17): `work_units` gains its second consumer, the deferral gate
  in [[loop_stall_guard]], which blocks `LOOP-STOP: complete` on any unit
  that isn't provably `done`/`dropped`
