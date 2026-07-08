---
title: "Hook: unregistered_loop_guard.sh"
type: hook
created: 2026-07-06
last_updated: 2026-07-08
sources:
  - sources/pr_15-17_loop-hardening-registration-eval-freeze-ledger-dry.md
  - sources/pr_23-24_hook-lib-observability-and-repo-keyed-loop-state.md
  - sources/pr_99_unregistered-loop-guard-nudge-once.md
tags: [hook, agentic-loop, unregistered-loop, stop-hook, loop-state, nudge, heuristic, nudge-once]
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

## Nudge-once-per-session (PR #99, 2026-07-08)

Before emitting, the hook now greps the discipline log (`als_log`'s existing ledger — the same file the emit path already writes to) for a prior `nudged=1` line for **this** `session_id`. If found, it exits 0 silently, logging `nudged=0 reason=already_nudged_this_session`, and does not re-emit. The check is:

```bash
esc_sid=$(printf '%s' "$session_id" | sed 's/[.[\*^$\\]/\\&/g')
if grep -q "hook=unregistered_loop_guard .*session=$esc_sid .*nudged=1" "$LOG_FILE" 2>/dev/null; then
  als_log "hook=unregistered_loop_guard session=$session_id dispatch_turns=$dispatch_turns nudged=0 reason=already_nudged_this_session"
  exit 0
fi
```

**Why:** without this, the nudge re-fired on every Stop for a session that kept meeting the trip conditions. For a genuinely one-off dispatch sequence, the honest response to the nudge is "no action needed" — but that response itself ends in a Stop, the conditions still hold, and the nudge fires again. Self-perpetuating, observed live 2026-07-08.

**Design notes:**
- Reuses the existing `als_log` ledger as the suppression source of truth rather than adding new per-session state — the emit path already recorded `nudged=1` lines keyed by `session_id`.
- Match is `session=$esc_sid` bounded by literal spaces on both sides, so a prior nudge for session `S-A` cannot substring-match and suppress session `S-AB`'s first nudge (tested both directions).
- The first nudge for any session is unaffected — this only suppresses repeats, never the initial signal.
- Fails open: a missing or unreadable log file means the grep finds nothing, so the hook falls through and emits — matching the rest of this hook's (and the repo's) fail-open convention.
- **`session_id` is escaped before interpolation into the grep BRE pattern** (`sed 's/[.[\*^$\\]/\\&/g'`), fixing a regression where a session id containing a literal BRE metachar (e.g. a `.`) could wildcard-match an unrelated session's log line and falsely suppress a legitimate first nudge. `als_sanitise_session_id` only strips `/` and collapses `..` (path-traversal defense) — a single `.` survives untouched, so this was a real reachable input. Reviewer-suggested; confirmed unreachable for real UUID-shaped session ids in current practice, but applied as defense-in-depth. See [[pr_99_unregistered-loop-guard-nudge-once]] for the full fix and test detail.

## Design decisions

- **Nudge, not block — a deliberate deviation from every sibling loop-state hook.** [[loop_state_guard]] and [[loop_stall_guard]] both block on ground truth: a `progress.json` either exists with the right session stamp, or it doesn't. This hook has only a heuristic — dispatch-count-shaped evidence that a loop *might* be running unregistered — and heuristics that block risk nagging legitimate non-loop sessions (research fan-outs, single reviews) that also happen to dispatch several subagents. The flip condition is recorded explicitly in the source: a nudge that gets delivered but ignored in practice is the trigger to escalate this hook to a block in a future PR, not a silent design drift.
- **Sibling hook, not an extension of `loop_state_common.sh`.** Reuses the pattern (structured jq on tool_use, never a text grep) but does not touch the shared lib — this hook's detection question (heuristic unregistered-loop shape) is different in kind from the shared lib's job (ground-truth state of an already-registered loop).
- **`message.id`-based counting, not tool_use-call counting.** This is the specific design choice that keeps parallel fan-outs (a legitimate, common pattern — see [[dispatching-parallel-agents]]) from ever tripping the nudge: they share one message.id regardless of how many `Agent` calls they contain.
- **YAGNI cuts, explicit in the source:** no `subagent_type` classification (doesn't distinguish implementer dispatches from research/review dispatches), no dispatch-review-cycle state machine, no block-once marker, no changes to `lib/loop_state_common.sh`.
- **`jq` failures are logged with distinct reasons**, not silently folded into the below-threshold outcome: `jq_missing`, `jq_parse_error`, `payload_parse_error` are each a separate logged reason, keeping "jq isn't installed" auditably distinct from "the transcript didn't parse" or "the hook payload itself was malformed."
- **Silent on `als_count_invocations` parse failures, by explicit choice (PR #23).** This hook's `ulg_has_skill_invocation` calls the shared `als_count_invocations` directly (not through the retry wrapper `als_stable_invocations`), and [[pr_23-24_hook-lib-observability-and-repo-keyed-loop-state|PR #23]]'s hook-lib rework moved that function's jq-failure signalling from a direct `als_log` call to a stderr tag intended for the retry wrapper to pick up. This hook now explicitly discards that stderr (`2>/dev/null`) rather than let it leak to the hook's own stderr — preserving this hook's prior (pre-#23) behaviour of staying silent on a parse failure here, now an explicit decision instead of an accident of the old design.

## Stdin read convention

Reads its payload via `IFS= read -r -d '' -t 5 input || true`, the repo's bounded-read floor. See [[pr_76_harden-hook-stdin-read]] for the general convention this hook follows.

## Test coverage

`hooks/scripts/tests/unregistered_loop_guard.test.sh` — grew from 27 checks to include a new Task 4 block (+75 lines, PR #99): repeat-session suppression (nudge fires on the first Stop, is suppressed on the second), fresh-session still-nudges-once regression guard, cross-session independence in both directions (exact session-id match, not substring), and a BRE-metachar regression test (`s.1` vs `sX1`) that fails against an unescaped grep and passes only once the session id is escaped. Full suite 37/37 passing post-PR-99. Landed alongside updates to `stdin_bounded_read.test.sh`, the timeout-floor test (backstop count bumped for the new hook entry), and the exec-bit invariant test (new file added at mode `100755`).

## Known limitations

- **Heuristic, not ground truth** — can miss unregistered-loop shapes outside its 3-distinct-dispatch-turn / no-progress.json / no-skill-invocation triple. A loop that dispatches exactly 2 sequential implementers, or one that briefly touches the `agentic-loop` Skill without properly registering `progress.json`, will not trip this hook.
- **Nudge can be ignored** — same honest boundary every hook in this repo has: it can force a signal to appear (`additionalContext`), not force the model to act on it. This is the explicit reason the flip condition to "block" is pre-recorded rather than left as an open question.
- **No `subagent_type` awareness** — treats every `Agent` tool_use identically; a session dispatching 3 sequential research or review agents (not implementers) can trip the same nudge as a genuine unregistered implementation loop. Accepted as a YAGNI cut, not a bug.
- **Session-scoped suppression only (PR #99)** — the nudge-once behaviour is keyed on `session_id`; a new session that meets the same conditions gets its own first nudge. This is not a global "nudge at most once ever" — it specifically targets the self-perpetuating repeat-nudge failure mode within one session.

## See also

- [[loop_state_guard]] — the ground-truth sibling: blocks a *registered* loop missing/mismatching `progress.json`; this hook covers the case where registration never happened at all
- [[loop_stall_guard]] — C2, requires a `LOOP-STOP` declaration; also Stop-only, also ground-truth-based
- [[agentic-loop]] — the skill this hook detects the *absence* of being invoked
- [[hook-exit-codes]] — this hook nudges via exit-0 `additionalContext` rather than `exit 2`, the deliberate non-blocking choice this page's mechanism table explains
- [[discipline-loop]] — full Stop hook composition (now 5 Stop hooks with this addition)
- [[dispatching-parallel-agents]] — the legitimate parallel-fan-out pattern this hook's `message.id` counting is designed not to trip on
- [[pr_15-17_loop-hardening-registration-eval-freeze-ledger-dry]] — PR #17 source record (F1)
- [[pr_23-24_hook-lib-observability-and-repo-keyed-loop-state]] — PR #23 source record: the hook-lib jq-failure-signalling rework this hook's stderr-discard carve-out responds to
- [[pr_99_unregistered-loop-guard-nudge-once]] — PR #99 source record: the nudge-once-per-session fix and grep-metachar hardening
- [[dashboard]] / [[pr_25_observability-dashboard]] — PR #25 (2026-07-06): a 16-task, 5-fix-loop session run under a properly registered loop; cross-linked here as the kind of large multi-task session this guard is designed to flag if orchestration ever drifts unregistered
