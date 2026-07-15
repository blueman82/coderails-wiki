---
title: "Hook: loop_stall_guard.sh"
type: hook
created: 2026-06-25
last_updated: 2026-07-15
sources:
  - sources/session_2026-06-25_agentic-loop-upgrade-arc.md
  - sources/pr_49_gate-function-rename.md
  - sources/pr_76_harden-hook-stdin-read.md
  - sources/pr_96-98_mode-aware-install-argument-injection-guard-hook-owned-counter.md
  - sources/pr_86-107_2026-07-08_loop-lib-residuals.md
  - sources/pr_118-123_self-improving-loops.md
  - sources/pr_184_185_186_loop-cost-tracking.md
tags: [hook, agentic-loop, anti-stall, stop-hook, loop-stop, loop-state, hook-owned-counter, malformed-transcript, retro-gate, schema-version-2, cost-tracking]
---

# loop_stall_guard.sh (C2)

Stop hook that blocks an active, incomplete agentic loop from stopping without a `LOOP-STOP: <category> — <reason>` declaration. The declaration-based anti-stall layer (C2). Sits after `loop_state_guard` (C1) in Stop hook order.

Source: `coderails/hooks/scripts/loop_stall_guard.sh`
Shared lib (Gates 1–4): `coderails/hooks/scripts/lib/loop_state_common.sh` (extracted PR #49; `als_gate_*` functions shared with [[loop_state_guard]])

## Event and mode

| Field | Value |
|---|---|
| Event | `Stop` |
| Mode | block (exit 2) |
| Timeout | 15s (hooks.json) |
| Stop order | 4th (last Stop hook; after confidence, verify, loop_state_guard) |

## What it enforces

**Declaration presence + valid category.** When an agentic loop is active and incomplete, the orchestrator's stopping turn must contain a line matching `LOOP-STOP: <category> — <reason>` where `<category>` is from the fixed vocabulary. It does NOT judge whether the reason is legitimate (honest boundary: forces categorised declaration, cannot force honesty).

## The LOOP-STOP declaration

```
LOOP-STOP: <hard-stop|approval-gate|awaiting-input|complete> — <reason>
```

**Vocabulary** (defined once in `lib/loop_state_common.sh` as `LOOP_STOP_VOCAB`, consumed by both the regex and the block message):

| Category | Maps to |
|---|---|
| `hard-stop` | Verification failure, premise disproven, ambiguous out-of-envelope decision, unauthorised destructive op |
| `approval-gate` | Named risk boundary the envelope flagged for sign-off (pause-then-proceed) |
| `awaiting-input` | Planned interaction point inside the loop (Phase -1 improve-prompt ask, Phase 1 plan confirmation) |
| `complete` | All authorised work done — **MUST also set `progress.json status: "complete"` and run Phase 13** |

The `complete` ⇒ teardown coupling is load-bearing: a `LOOP-STOP: complete` tag in text only satisfies the current turn's gate. If the model declares `complete` in text but leaves `status: in-progress`, every later stop still demands a tag and C1 still treats the loop as active. The two actions are atomic in SKILL.md.

## Logic: skip gates (cheap first)

Gates 1–4 are implemented as named `als_gate_*` functions in `loop_state_common.sh` (shared with [[loop_state_guard]] — extracted PR #49, formerly byte-identical between the two guards). Gates 5–6 are local to this script.

1. **`als_gate_no_transcript`** — allow.
2. **`als_gate_stop_hook_active`** — allow (avoid stop-loop).
3. **`als_gate_not_a_loop`** — allow (not a loop). Same structured `jq` detection as C1, via `loop_state_common.sh`. This gate's underlying transcript parse is the same one made malformed-line-tolerant by [[pr_86-107_2026-07-08_loop-lib-residuals|PRs #91/#107]] — see [[loop_state_guard]]'s "Malformed-transcript tolerance" section for the fix detail; this hook shares the fix via the common lib, nothing local changed.
4. **`als_load_progress` / done-and-not-rearmed check** — allow if `status == "complete"` AND not re-armed AND session-owned (shared off-switch with C1).
5. **Last assistant message contains a valid `LOOP-STOP` line** — allow (regex built from `LOOP_STOP_VOCAB`; category must be followed by a non-alphanumeric char or end-of-line so "completed" doesn't match "complete").
6. **BLOCK (exit 2)** — active + incomplete + no valid declaration.

## Block message design

The block message contains the **exact copy-paste tag template** built from `LOOP_STOP_VOCAB`, stating the `complete` ⇒ teardown coupling. The model copies the template; it never reconstructs the format. (Same lesson as C1's path-in-block-message: remove any derivation the model must do.)

## Multi-hook thrash risk

A stopping turn inside an active loop now carries three co-requirements: confidence labels, DNV resolution, and `LOOP-STOP`. The agentic-loop skill's Phase 0.5 bundles all three into one "stop-ceremony" note so the model emits them together rather than thrashing one hook while satisfying another.

## Phase 13 stall metric

`progress.json` gains a `loop_stop_counts` object (`{hard-stop, approval-gate, awaiting-input, complete}`). Phase 13 surfaces it. The `awaiting-input` count is the primary **avoidable-stall** signal — if a model rubber-stamps `awaiting-input` to escape the gate, it shows up in its own audit rather than hiding behind a technically-valid tag.

**Counter is now hook-owned, sole-writer (PR #98, merged 2026-07-05).** This
hook — not the orchestrator — increments `loop_stop_counts.<category>` after
validating the declared category, via a tmp-file `jq` read-modify-write (same
pattern as `scripts/post_review.sh`'s `write_cache`). Previously the counter had
**two writers** (this hook's presence check plus orchestrator prose asking the
agent to self-maintain the field) that raced under concurrent Stop-hook
invocations, undercounting 2 loops in a recorded session. `SKILL.md` now states
the field is HOOK-OWNED in 5 places — the orchestrator reads it as-is and
carries it forward verbatim on any progress.json rewrite, same treatment as
`completed_marker`. The write is best-effort and never fatal: a missing
`progress.json`, malformed JSON, absent `jq`, or a failed `mv` is logged and
swallowed, and the hook still exits 0 to let the declared stop through. See
[[spec-plan-progress-artifact-chain]] for the full race analysis and
[[pr_96-98_mode-aware-install-argument-injection-guard-hook-owned-counter]] for
the fix detail (including four standing-invariant tests added in a post-review
round: no-clobber, jq-absence fail-open, last-declaration-wins tie-break,
degraded-filesystem safety).

## Log output

Appends a `key=value` line to `$CLAUDE_DISCIPLINE_LOG`:
`hook=loop_stall_guard session=<id> invocations=<n> declared=<0|1> blocked=<0|1>`

## Known limitations

- Cannot force the declared reason or category to be truthful; a model can rubber-stamp `awaiting-input`. The Phase 13 KPI is the auditable counter-pressure.
- Four Stop hooks (confidence, verify, C1, C2) firing on the same stopping turn is the highest ceremony cost in the system; the main thing to watch in Phase 13 is avoidable-stall counts from stop-ceremony thrash. (A fifth Stop hook, [[unregistered_loop_guard]], was added PR #17 — it only fires on *unregistered* loops and is a nudge, not a co-requirement of this ceremony, so it does not add to this specific thrash risk.)

## Retro gate on `complete` (PR #119)

This hook additionally hosts the **retro gate** (`als_gate_retro_on_complete` in the shared lib). On a `LOOP-STOP: complete` declaration only, it blocks (exit 2) unless a parseable `retro.json` sits beside `progress.json` — the Phase 13 write contract. Category is lowercase-normalised before the check (a case-insensitive upstream match feeding a case-sensitive compare was the Critical caught in review — `Complete`/`COMPLETE` had bypassed the gate entirely). **Block-before-bump:** the gate call runs before `bump_loop_stop_count`, so a blocked `complete` never increments the counter. Fail-open when jq is absent (matches `bump_loop_stop_count`), with a `retro_gate=skipped_no_jq` breadcrumb; fail-closed when `ALS_PATH` is unset. Presence + parse only — provenance/content fidelity is not checkable here, same honest boundary as every other guard. See [[pr_118-123_self-improving-loops]].

**Gate widened to `schema_version >= 1`, forward-compatible (PR #186, 2026-07-15).** Originally required an **exact** `schema_version` 1; now accepts any value `>= 1` (non-numeric, absent, or `< 1` still block). This is what let Phase 13 bump the retro's own schema to 2 — see below — without needing a matching same-PR gate change. `loop_stall_guard.test.sh` carries both directions as explicit controls: schema_version 1/2/99 all allow (forward-compat proof), 0/absent/non-numeric all still block. See [[pr_184_185_186_loop-cost-tracking]].

**`retro.json` is now `schema_version` 2 as of the same cluster** — Phase 13's teardown write contract gained a cost-mining sub-step (`hooks/scripts/lib/loop_cost.sh`'s `dc_mine_token_usage`) that writes `retro.cost` (the miner's frozen per-model token/USD breakdown) and lifts its `models_used` array to top-level `retro.models_used`. This gate itself is unchanged by that addition beyond the version-check widening above — it still checks presence + `schema_version`, never the cost field's correctness, so a miner failure (which fails open to empty `cost`/`models_used`) cannot stall a loop. See [[agentic-loop]]'s Phase 13 section for the full field contract and [[pr_184_185_186_loop-cost-tracking]] for the source record.

## Stdin read convention (PR #76)

This hook reads its payload via `IFS= read -r -d '' -t 5 input || true`. See [[pr_76_harden-hook-stdin-read]] for the full convention and the fail-open rationale.

## See also

- [[loop_state_guard]] — C1: presence/ownership guard; shares loop-active detection and the active-window off-switch
- [[unregistered_loop_guard]] — new sibling Stop hook (PR #17): covers loops that never reached C1/C2 at all because they never registered
- [[hook-exit-codes]] — this hook blocks via plain `exit 2` on `Stop`, the mechanism this page's table documents
- [[agentic-loop]] — the skill that must emit `LOOP-STOP` declarations; Phase 0.5 is the stop-ceremony note
- [[spec-plan-progress-artifact-chain]] — the two-hook loop-state guard architecture; why C1 speaks before C2
- [[discipline-loop]] — full hook composition (5 Stop hooks + 2 SubagentStop hooks, 14 hooks total across all events, as of PR #17)
- [[enforcement-model]] — hooks vs. commands
- [[pr_96-98_mode-aware-install-argument-injection-guard-hook-owned-counter]] — PR #98: this hook becomes the sole writer of loop_stop_counts
- [[pr_86-107_2026-07-08_loop-lib-residuals]] — PRs #91/#107: malformed-transcript tolerance in the shared `loop_state_common.sh` parse this hook's `als_gate_not_a_loop` depends on
