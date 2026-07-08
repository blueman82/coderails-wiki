---
title: "Hook: unregistered_loop_guard.sh"
type: hook
created: 2026-07-06
last_updated: 2026-07-08
sources:
  - sources/pr_15-17_loop-hardening-registration-eval-freeze-ledger-dry.md
  - sources/pr_23-24_hook-lib-observability-and-repo-keyed-loop-state.md
  - sources/pr_86-107_2026-07-08_loop-lib-residuals.md
tags: [hook, agentic-loop, unregistered-loop, stop-hook, loop-state, nudge, heuristic, malformed-transcript]
---

# unregistered_loop_guard.sh

Stop hook that nudges — never blocks — when a session looks like an unregistered agentic loop: many sequential agent dispatches, no `progress.json`, and no `agentic-loop` Skill invocation anywhere in the transcript. New **sibling** hook, not an extension of [[loop_state_guard]] — a different question (heuristic detection of a loop that never registered) with a different evidence class (no ground truth to enforce against).

Source: `coderails/hooks/scripts/unregistered_loop_guard.sh`
Shared lib: none — this hook does not extend `loop_state_common.sh`; its detection logic is local (see "Design" below)
Path helper: `coderails/hooks/scripts/lib/agentic_loop_path.sh` (used to check whether a `progress.json` exists)

## Why this hook exists

[[loop_state_guard]] enforces presence + ownership of `progress.json` — but only for loops that already registered one. On 2026-07-06 a 17-work-unit `subagent-driven-development` execution ran 9 units with zero hook signal, because the orchestrator never invoked `coderails:agentic-loop` and never registered anything — an unregistered loop is structurally invisible to every existing loop-state hook, all of which key off a file that was never created. This hook closes that blind spot with a heuristic nudge, since — unlike registered-loop hooks — it has no ground truth to enforce a block against. Full incident record: [[pr_15-17_loop-hardening-registration-eval-freeze-ledger-dry]] (F1).

## Event and mode

| Field | Value |
|---|---|
| Event | `Stop` |
| Mode | **nudge only** (`hookSpecificOutput.additionalContext` on stdout, exit 0) — never blocks |
| Timeout | 5s floor (per the repo's stdin-read convention) |

## Trip condition

All three must hold:

1. **≥3 distinct dispatch turns.** Counts DISTINCT `message.id` values carrying an `Agent` tool_use, via structured `jq` — never a text grep. Parallel fan-outs (several `Agent` calls in one turn) share a single `message.id` and count as 1, so they never trip the nudge. Sequential dispatches — the incident's actual shape, an orchestrator dispatching one implementer after another across separate turns — each carry a distinct `message.id` and count as N. Threshold of 3 mirrors the existing tier-2 work-unit calibration used elsewhere (task-evals, agentic-loop's own complexity guard).
2. **No `progress.json`** at the path `agentic_loop_path.sh` resolves for this session.
3. **No `agentic-loop` Skill invocation** anywhere in the transcript — same structured `jq` match on `name == "Skill"` / `input.skill` matching the agentic-loop pattern that [[loop_state_guard]]'s `als_gate_not_a_loop` uses, applied here as a positive detection rather than a skip-gate.

## Design decisions

- **Nudge, not block — a deliberate deviation from every sibling loop-state hook.** [[loop_state_guard]] and [[loop_stall_guard]] both block on ground truth: a `progress.json` either exists with the right session stamp, or it doesn't. This hook has only a heuristic — dispatch-count-shaped evidence that a loop *might* be running unregistered — and heuristics that block risk nagging legitimate non-loop sessions (research fan-outs, single reviews) that also happen to dispatch several subagents. The flip condition is recorded explicitly in the source: a nudge that gets delivered but ignored in practice is the trigger to escalate this hook to a block in a future PR, not a silent design drift.
- **Sibling hook, not an extension of `loop_state_common.sh`.** Reuses the pattern (structured jq on tool_use, never a text grep) but does not touch the shared lib — this hook's detection question (heuristic unregistered-loop shape) is different in kind from the shared lib's job (ground-truth state of an already-registered loop).
- **`message.id`-based counting, not tool_use-call counting.** This is the specific design choice that keeps parallel fan-outs (a legitimate, common pattern — see [[dispatching-parallel-agents]]) from ever tripping the nudge: they share one message.id regardless of how many `Agent` calls they contain.
- **YAGNI cuts, explicit in the source:** no `subagent_type` classification (doesn't distinguish implementer dispatches from research/review dispatches), no dispatch-review-cycle state machine, no block-once marker, no changes to `lib/loop_state_common.sh`.
- **`jq` failures are logged with distinct reasons**, not silently folded into the below-threshold outcome: `jq_missing`, `jq_parse_error`, `payload_parse_error` are each a separate logged reason, keeping "jq isn't installed" auditably distinct from "the transcript didn't parse" or "the hook payload itself was malformed."
- **Silent on `als_count_invocations` parse failures, by explicit choice (PR #23).** This hook's `ulg_has_skill_invocation` calls the shared `als_count_invocations` directly (not through the retry wrapper `als_stable_invocations`), and [[pr_23-24_hook-lib-observability-and-repo-keyed-loop-state|PR #23]]'s hook-lib rework moved that function's jq-failure signalling from a direct `als_log` call to a stderr tag intended for the retry wrapper to pick up. This hook now explicitly discards that stderr (`2>/dev/null`) rather than let it leak to the hook's own stderr — preserving this hook's prior (pre-#23) behaviour of staying silent on a parse failure here, now an explicit decision instead of an accident of the old design. This still holds after [[pr_86-107_2026-07-08_loop-lib-residuals|PRs #91/#107]] widened `als_count_invocations`'s stderr vocabulary (`skipped_malformed=N`, `read_error`, `all_lines_malformed`) — this hook discards all of it identically via the same `2>/dev/null`, so it benefits from the more tolerant parse (fewer false "no invocation found" results from one bad transcript line) without needing any change of its own.

## Known residual: this hook's own `jq -s` slurp is untouched

`ulg_count_dispatch_turns` — the dispatch-turn counter **inside this hook's own
script**, not the shared lib — has the identical bare-`jq -s`-slurp fragility that
[[pr_86-107_2026-07-08_loop-lib-residuals|PRs #91/#107]] fixed in
`loop_state_common.sh`. A single malformed JSONL line here still collapses this
hook's OWN dispatch-turn count, independent of the shared-lib fix. This was
explicitly flagged as out-of-scope in PR #91's own description and remains a
standing residual — not fixed by this cluster.

## Stdin read convention

Reads its payload via `IFS= read -r -d '' -t 5 input || true`, the repo's bounded-read floor. See [[pr_76_harden-hook-stdin-read]] for the general convention this hook follows.

## Test coverage

`hooks/scripts/tests/unregistered_loop_guard.test.sh` — 27 checks, mirroring [[loop_state_guard]]'s test-file conventions. Landed alongside updates to `stdin_bounded_read.test.sh`, the timeout-floor test (backstop count bumped for the new hook entry), and the exec-bit invariant test (new file added at mode `100755`).

## Known limitations

- **Heuristic, not ground truth** — can miss unregistered-loop shapes outside its 3-distinct-dispatch-turn / no-progress.json / no-skill-invocation triple. A loop that dispatches exactly 2 sequential implementers, or one that briefly touches the `agentic-loop` Skill without properly registering `progress.json`, will not trip this hook.
- **Nudge can be ignored** — same honest boundary every hook in this repo has: it can force a signal to appear (`additionalContext`), not force the model to act on it. This is the explicit reason the flip condition to "block" is pre-recorded rather than left as an open question.
- **No `subagent_type` awareness** — treats every `Agent` tool_use identically; a session dispatching 3 sequential research or review agents (not implementers) can trip the same nudge as a genuine unregistered implementation loop. Accepted as a YAGNI cut, not a bug.

## See also

- [[loop_state_guard]] — the ground-truth sibling: blocks a *registered* loop missing/mismatching `progress.json`; this hook covers the case where registration never happened at all
- [[loop_stall_guard]] — C2, requires a `LOOP-STOP` declaration; also Stop-only, also ground-truth-based
- [[agentic-loop]] — the skill this hook detects the *absence* of being invoked
- [[hook-exit-codes]] — this hook nudges via exit-0 `additionalContext` rather than `exit 2`, the deliberate non-blocking choice this page's mechanism table explains
- [[discipline-loop]] — full Stop hook composition (now 5 Stop hooks with this addition)
- [[dispatching-parallel-agents]] — the legitimate parallel-fan-out pattern this hook's `message.id` counting is designed not to trip on
- [[pr_15-17_loop-hardening-registration-eval-freeze-ledger-dry]] — PR #17 source record (F1)
- [[pr_23-24_hook-lib-observability-and-repo-keyed-loop-state]] — PR #23 source record: the hook-lib jq-failure-signalling rework this hook's stderr-discard carve-out responds to
- [[dashboard]] / [[pr_25_observability-dashboard]] — PR #25 (2026-07-06): a 16-task, 5-fix-loop session run under a properly registered loop; cross-linked here as the kind of large multi-task session this guard is designed to flag if orchestration ever drifts unregistered
- [[pr_86-107_2026-07-08_loop-lib-residuals]] — PRs #91/#107: malformed-transcript tolerance in the shared lib's `als_count_invocations` (this hook's delegate call benefits passively); this hook's OWN separate `jq -s` slurp in `ulg_count_dispatch_turns` remains an untouched, explicitly out-of-scope residual
