---
title: "Hook: loop_stall_guard.sh"
type: hook
created: 2026-06-25
last_updated: 2026-06-29
sources:
  - sources/session_2026-06-25_agentic-loop-upgrade-arc.md
  - sources/pr_49_gate-function-rename.md
  - sources/pr_76_harden-hook-stdin-read.md
tags: [hook, agentic-loop, anti-stall, stop-hook, loop-stop, loop-state]
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
3. **`als_gate_not_a_loop`** — allow (not a loop). Same structured `jq` detection as C1, via `loop_state_common.sh`.
4. **`als_load_progress` / done-and-not-rearmed check** — allow if `status == "complete"` AND not re-armed AND session-owned (shared off-switch with C1).
5. **Last assistant message contains a valid `LOOP-STOP` line** — allow (regex built from `LOOP_STOP_VOCAB`; category must be followed by a non-alphanumeric char or end-of-line so "completed" doesn't match "complete").
6. **BLOCK (exit 2)** — active + incomplete + no valid declaration.

## Block message design

The block message contains the **exact copy-paste tag template** built from `LOOP_STOP_VOCAB`, stating the `complete` ⇒ teardown coupling. The model copies the template; it never reconstructs the format. (Same lesson as C1's path-in-block-message: remove any derivation the model must do.)

## Multi-hook thrash risk

A stopping turn inside an active loop now carries three co-requirements: confidence labels, DNV resolution, and `LOOP-STOP`. The agentic-loop skill's Phase 0.5 bundles all three into one "stop-ceremony" note so the model emits them together rather than thrashing one hook while satisfying another.

## Phase 13 stall metric

`progress.json` gains a `loop_stop_counts` object (`{hard-stop, approval-gate, awaiting-input, complete}`) the orchestrator increments per declaration. Phase 13 surfaces it. The `awaiting-input` count is the primary **avoidable-stall** signal — if a model rubber-stamps `awaiting-input` to escape the gate, it shows up in its own audit rather than hiding behind a technically-valid tag.

## Log output

Appends a `key=value` line to `$CLAUDE_DISCIPLINE_LOG`:
`hook=loop_stall_guard session=<id> invocations=<n> declared=<0|1> blocked=<0|1>`

## Known limitations

- Cannot force the declared reason or category to be truthful; a model can rubber-stamp `awaiting-input`. The Phase 13 KPI is the auditable counter-pressure.
- Four Stop hooks (confidence, verify, C1, C2) is the highest ceremony cost in the system; the main thing to watch in Phase 13 is avoidable-stall counts from stop-ceremony thrash.

## Stdin read convention (PR #76)

This hook reads its payload via `IFS= read -r -d '' -t 5 input || true`. See [[pr_76_harden-hook-stdin-read]] for the full convention and the fail-open rationale.

## See also

- [[loop_state_guard]] — C1: presence/ownership guard; shares loop-active detection and the active-window off-switch
- [[agentic-loop]] — the skill that must emit `LOOP-STOP` declarations; Phase 0.5 is the stop-ceremony note
- [[spec-plan-progress-artifact-chain]] — the two-hook loop-state guard architecture; why C1 speaks before C2
- [[discipline-loop]] — full hook composition (all six Stop hooks)
- [[enforcement-model]] — hooks vs. commands
