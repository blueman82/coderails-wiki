---
title: "PR #204 — mechanical cost-reporter on LOOP-STOP: complete"
type: source
created: 2026-07-17
last_updated: 2026-07-22
sources: []
tags: [hook, agentic-loop, loop-stall-guard, complete-gate, cost-tracking, reporter-not-gate, system-message, anti-fabrication, schema-version-2]
---

# PR #204 — mechanical cost-reporter on `LOOP-STOP: complete`

Adds `als_report_cost_on_complete` to `hooks/scripts/lib/loop_state_common.sh`,
called from `loop_stall_guard.sh`'s `gate_loop_stop_declared` (line 68), AFTER
`als_gate_proofs_on_complete` and BEFORE `bump_loop_stop_count`. Merged
2026-07-17, merge SHA `ce6b9d6f2f50e3ecd2e86601a3474d16725a606e`.

## Why it exists

`skills/agentic-loop/teardown.md`'s "Loop cost" bullet already said, in bold,
that a complete loop "must print it, not merely write it to disk" — this was
prose, not enforcement (see [[pr_184_185_186_loop-cost-tracking]], which added
the field but only as a reporting instruction). Loop `0d3fb487` read that file,
ran Phase 13, and silently omitted the cost line anyway — then authored a
fabricated explanation for why there was nothing to print. The diagnosis is the
same one that motivated every other `complete`-branch gate in this hook: a rule
*described* in SKILL.md prose is not a rule *enforced* by a hook. This PR makes
model compliance irrelevant to whether the human sees the cost.

## What it is: a REPORTER, not a gate

Named `als_report_cost_on_complete` deliberately — every path returns 0. This
is the fourth thing on `loop_stall_guard`'s `complete` branch, in call order:

1. `als_gate_retro_on_complete` — **blocks**
2. `als_gate_work_units_on_complete` — **blocks** (the deferral gate, [[pr_194_198_loop-complete-deferral-and-proof-gates]])
3. `als_gate_proofs_on_complete` — **blocks** (the proof gate, same PR pair)
4. `als_report_cost_on_complete` — **never blocks**

## Deliberate posture inversion

Every other gate in `loop_state_common.sh` — and `check_verify_loop.sh`/
`check_confidence_labels.sh` elsewhere in the repo — follows the house
fail-toward-blocking idiom: run the check, let a failure exit non-zero. This
function does the opposite on purpose. Rationale: `dc_mine_token_usage`
(`hooks/scripts/lib/loop_cost.sh:7-12`, see [[pr_184_185_186_loop-cost-tracking]])
is contractually fail-open to `{}` on any mining error and documents that it
"must never block a caller" — `retro.json`'s `.cost` can legitimately be `{}`
(miner ran, failed open) on an otherwise perfectly valid, already-finished loop.
A fail-closed reporter would deadlock a loop that has ALREADY passed the
retro/work_units/proof gates above it — strictly worse than the unrecorded-cost
bug it exists to fix. Do not "fix" this function into the house fail-closed
style; that would be reintroducing the deadlock this design explicitly avoids.

## Behaviour matrix

Fires only on `category == "complete"` (case-insensitive, same `category_lc`
idiom as every sibling gate). By the time it runs, `retro.json` is already
proven present and parseable with `schema_version >= 1` by the retro gate
above it, so this function does not re-validate existence/parseability — it
only branches on the fields it needs.

| Condition | Output |
|---|---|
| `schema_version < 2` (legacy, pre-cost-miner retro) | silent (42 of 46 on-disk retros at merge time were legacy) |
| `schema_version >= 2`, `.cost` populated (has a usd total) | `Loop cost: $<usd> (<tokens> tokens), prices as of <date>, N days old` |
| `schema_version >= 2`, `.cost` non-empty but missing `total_usd_estimate` and/or `total_tokens` | `cost recorded but incomplete (missing <field(s)>)` |
| `schema_version >= 2`, `.cost == {}` (miner ran, failed open) | `cost unavailable (miner returned no data)` |
| `schema_version >= 2`, `.cost` absent (teardown skipped the mining sub-step) | `cost not recorded` |

The four non-legacy rows are deliberately **distinct messages, never
collapsed into one**. `{}` = miner ran and came back empty; absent = the
teardown step itself was skipped; incomplete = the miner partially wrote
before failing. Collapsing any two into a single message, or into a silent
return, would recreate — inside the very mechanism built to fix it — the
original failure: a cost that exists (or doesn't) on disk but never reaches
the human in an honest, distinguishable way.

**`schema_version` is the row-2-vs-row-4 discriminator, not cost-presence** —
rows 2 and 4 (legacy silent vs. sv2-but-absent) both have `.cost` absent, so
cost-presence alone cannot tell "legacy loop, nothing to report" from "sv2
loop, teardown skipped a step it should have run."

## Anti-fabrication is the design principle

"Visibly-wrong beats plausibly-fabricated" governs every branch:

- A non-scalar `.cost.total_usd_estimate`/`.total_tokens` (array/object) is
  selected to empty and falls into the incomplete path, rather than being
  passed through `jq -r`, which would emit a pretty-printed multi-line blob
  (real newlines) into a one-line terminal message — verified during the
  build.
- `printf '%.2f'` silently prints `0.00` on a non-numeric input (verified) —
  the USD-rounding step therefore gates on a strict numeric-string pattern
  first and only rounds a confirmed number; a garbage value prints raw
  rather than fabricating `$0.00`.
- `date -j -f "%Y-%m-%d"` on macOS does NOT reject trailing garbage — it
  silently accepts `"2026-06-24FORGED"` and parses only the leading date
  (verified) — so the staleness-age computation gates on the exact
  `YYYY-MM-DD` shape before calling `date` at all, and prints the raw
  `prices_as_of` string verbatim on anything else.
- The final message is stripped to printable + space/tab characters before
  reaching the terminal (`tr -c '[:print:][:space:]' ' '` then collapsing
  newlines/tabs to spaces) — every interpolated value is `retro.json`-derived
  and JSON-well-formed-but-not-necessarily-control-character-free, so a raw
  ESC or embedded newline is neutralised rather than trusted to render
  safely.

The discipline-log entry records only the outcome CLASS (`reported` /
`miner_failed_open` / `cost_absent` / `cost_incomplete`), never the message
body — the body interpolates untrusted `retro.json` values, and log
sanitisation is a backstop, not a reason to widen what's logged.

## `additionalContext` vs. `systemMessage` — the channel-discovery finding

Before this PR, no hook in coderails emitted a top-level `systemMessage` field
— every existing hook used `additionalContext` (model-visible only; Claude
Code's own docs state it "doesn't appear as a chat message in the interface").
A Stop hook's stdout otherwise goes to the debug log, invisible to the human.
`systemMessage` ("Warning message shown to the user") is the channel that
actually reaches the human's terminal — verified empirically with a live smoke
test during this PR's build, rendering as `Stop says: <msg>`. This is a new
precedent for the hook system: the first coderails hook to deliberately target
the human over the model. See [[hook-exit-codes]] for the fuller
channel-mechanics writeup this PR's discovery was folded into.

## Files changed

- `hooks/scripts/lib/loop_state_common.sh` — new `als_report_cost_on_complete` function (+212 lines)
- `hooks/scripts/loop_stall_guard.sh` — one-line call site, `complete` branch, after the proof gate and before `bump_loop_stop_count`
- `hooks/scripts/tests/loop_stall_guard.test.sh` — new coverage (+374 lines)
- `skills/agentic-loop/SKILL.md`, `skills/agentic-loop/teardown.md` — Phase 13's "Loop cost" bullet reworded: the hook is now the floor, not a reason to omit assembling `retro.cost` correctly — the hook reports what was written, it does not compute it
- `docs/REFERENCE.md` — `loop_stall_guard.sh` table row updated in lockstep

## Wiki pages updated

- [[loop_stall_guard]] — new "Cost reporter on `complete` (PR #204)" section, joining the retro/deferral/proof gate sections; now three blocking gates plus one non-blocking reporter in call order
- [[hook-exit-codes]] — new `systemMessage` channel note alongside the existing `additionalContext`-vs-`exit 2` treatment
- [[agentic-loop]] — Phase 13's cost-reporting cross-reference updated to note the mechanical print

## See also

- [[pr_184_185_186_loop-cost-tracking]] — the cost-mining sub-step and `retro.json` `schema_version` 2 this reporter reads; the prose "must print it" instruction this PR mechanises
- [[pr_194_198_loop-complete-deferral-and-proof-gates]] — the two blocking gates this reporter runs immediately after
- [[loop_stall_guard]] — hosts the call site
- [[enforcement-model]] — hooks vs. commands; this is a hook that deliberately never enforces, a third category alongside block/nudge
