---
title: "PRs #15–17 — Loop hardening: registration, eval-freeze timing, ledger DRY"
type: source
created: 2026-07-06
last_updated: 2026-07-06
sources: []
tags: [source, agentic-loop, loop-state, unregistered-loop, eval-freeze, writing-plans, subagent-driven-development, sdd-ledger, hook]
---

# PRs #15–17 — Loop hardening: registration, eval-freeze timing, ledger DRY

Three merged PRs closing gaps surfaced by a single 2026-07-06 incident: an unregistered 17-work-unit `subagent-driven-development` execution ran 9 units with zero hook-visible loop state and no frozen evals. Full incident background: memory file `project_loop_hardening_handoff.md` (owner's origin session, `f5fa5ea2-35ca-45e5-a558-1ccfcdc696fb`). This cluster's own session shipped the fix as three independent PRs.

## PR metadata

| Field | PR #15 | PR #16 | PR #17 |
|---|---|---|---|
| Title | F2: eval-freeze wording fix — freeze at plan completion, not final task | WU4: move SDD ledger workspace to session-keyed loop-state dir | F1: unregistered-loop nudge hook (`unregistered_loop_guard.sh`) |
| Merged | 2026-07-06 | 2026-07-06 | 2026-07-06 |
| Merge SHA | `dc55377` | `e4aafa6` | `d1b1360` |
| Branch | doc-only fix branch | ledger relocation branch | new hook branch |
| JIRA ticket | — | — | — |

## The incident (why this cluster exists)

On 2026-07-06 a 17-work-unit `subagent-driven-development` execution ran 9 units with no hook signal at all, because the orchestrator never invoked `coderails:agentic-loop` and never registered a `progress.json`. Two independent gaps let this happen unnoticed:

1. **F1 — detection gap.** [[loop_state_guard]] only gates loops that already registered a `progress.json`. An orchestrator that never registers is structurally invisible to it — registration depended entirely on the orchestrator remembering to load the skill, exactly the class of prompt-level discipline failure coderails exists to eliminate.
2. **F2 — sequencing gap.** [[writing-plans]]'s "Final eval-gate task" wording let freeze-before-build and grade-and-post flatten into one end-stage task. The same incident's 17-unit plan encoded both in one final task, so the execution ran 8 tasks deep with no frozen `evals.json` at all — the freeze-before-build anti-gaming rule ([[task-evals]]) was structurally unenforced by the plan's own wording.

A third, related but independently-scoped fix (WU4) rides in the same session: the SDD ledger's workspace directory was keyed on repo root, not session — a latent collision bug for any two concurrent SDD sessions in the same checkout, destroyed wholesale by `git clean -fdx`.

## Summary

**PR #15 (F2)** — restructures `writing-plans/SKILL.md`'s "Final eval-gate task" section into two explicit sections: freezing happens once, immediately after the mandatory planning-sequence stress-test and *before* Task 1 is dispatched; the plan's actual final task now only grades and posts the already-frozen evals via `/coderails:post-evals`. Four cross-referencing files corrected in lockstep so no doc still describes freezing as happening "as the final task": `task-evals/SKILL.md`'s invocation-point bullet, `subagent-driven-development/SKILL.md`'s new Pre-Flight Plan Review bullet (assert `evals.json` has non-empty `frozen_at`/`frozen_sha` before dispatching Task 1 — the belt-and-braces pre-flight check), `commands/workflow.md:157`, and `docs/REFERENCE.md:83`. Doc-only; verified by a lockstep grep across all five manifest files, not by a test file.

**PR #16 (WU4)** — moves the SDD progress ledger from a repo-root-keyed `$root/.coderails/sdd` directory (shared by every session working in that checkout — a latent collision bug, and destroyed entirely by `git clean -fdx`) to the session-keyed loop-state directory: the same directory `progress.json` lives in, resolved via `hooks/scripts/lib/agentic_loop_path.sh` + `dirname`. Clean-break: no read-both-paths shim, no fallback to the old location. The ledger is renamed `sdd-ledger.md` (distinct from `progress.json`/`progress.md`) and its fact-ownership split is now documented explicitly: the ledger is task-level truth with commit ranges, controller-owned; `progress.json.work_units[id].status` is unit-level truth, orchestrator-owned. `sdd-workspace` (the script resolving the workspace path) is hardened to fail loud on two conditions: an unset `CLAUDE_CODE_SESSION_ID` (the path helper's own per-call session-id fallback is non-idempotent — safe for a stateless reader, fatal for a ledger owner that must resolve the same path across multiple calls) and non-absolute helper output (a prior silent cwd-fallback bug, reproduced during review). First test infrastructure for any SDD script: a 6-check smoke test (`sdd-workspace.test.sh`), written first against the old script (confirmed red), then made green.

**PR #17 (F1)** — adds a new sibling Stop hook, `hooks/scripts/unregistered_loop_guard.sh`, that detects the unregistered-loop shape and nudges — it never blocks. Trip condition: ≥3 **distinct** `message.id` values carrying an `Agent` tool_use in the transcript (parallel fan-outs share one `message.id` and score 1, never tripping; sequential dispatches — the incident's actual shape — get distinct ids and score N), AND no `progress.json` at the resolved loop-state path, AND no `agentic-loop` Skill invocation anywhere in the transcript. All three structured `jq` matches on `tool_use` entries, never text greps, per the existing `loop_state_common.sh` convention. Delivery is `hookSpecificOutput.additionalContext` on stdout with exit 0 — model-visible, never a block. This is a deliberate deviation from every sibling loop-state hook, which all block on ground truth; this hook has only a heuristic, not ground truth, so nudge is the honest posture. The flip condition is recorded explicitly: a nudge delivered but ignored is the trigger to upgrade this hook to a block in a future PR. `jq` failures are logged with distinct reasons (`jq_missing` / `jq_parse_error` / `payload_parse_error`) rather than silently folded into the below-threshold case. New 27-check test file mirrors `loop_state_guard.test.sh`'s conventions; `stdin_bounded_read`, the timeout-floor test, and the exec-bit invariant test were all updated for the new file.

## Files changed

**PR #15** — `skills/writing-plans/SKILL.md`, `skills/task-evals/SKILL.md`, `skills/subagent-driven-development/SKILL.md`, `commands/workflow.md`, `docs/REFERENCE.md`.

**PR #16** — `skills/agentic-loop/SKILL.md` (one sentence, inside the `## Context-window persistence` no-touch region's neighbourhood — see [[agentic-loop]]), `skills/subagent-driven-development/SKILL.md`, `skills/subagent-driven-development/scripts/sdd-workspace`, `skills/subagent-driven-development/scripts/task-brief`, `skills/subagent-driven-development/scripts/review-package`, `skills/subagent-driven-development/scripts/tests/sdd-workspace.test.sh` (new).

**PR #17** — `hooks/scripts/unregistered_loop_guard.sh` (new), `hooks/scripts/tests/unregistered_loop_guard.test.sh` (new, 27 checks), `hooks/hooks.json`, `hooks/scripts/tests/hooks_json_timeout_floor.test.sh` (backstop count bumped), `hooks/scripts/tests/exec_bit_invariant.test.sh`, `hooks/scripts/tests/stdin_bounded_read.test.sh`.

## Wiki pages updated

[[loop_state_guard]] (F1 sibling-hook cross-reference), [[unregistered_loop_guard]] (new page), [[writing-plans]] (freeze-at-plan-completion restructure), [[task-evals]] (invocation-point correction), [[subagent-driven-development]] (Pre-Flight Plan Review bullet, ledger relocation), [[agentic-loop]] (sdd-ledger.md sibling-file sentence), [[discipline-loop]] (Stop hook composition count 4→5).

## Caveats / gotchas

- **PR #16 is an accepted one-time migration risk, not a handled one.** Any session with an in-flight SDD ledger at the old `$root/.coderails/sdd/progress.md` path when this PR merged loses continuity with that ledger — there is no shim. Recovery: `git log` remains authoritative for what commits actually happened, per the skill's pre-existing "trust the ledger and git log" contract.
- **F1 is a nudge, not a gate, by design** — it has no ground truth to enforce (unlike [[loop_state_guard]], which enforces presence of a real `progress.json`). Do not treat `unregistered_loop_guard.sh`'s silence as proof a loop is properly registered; it can still miss shapes outside its 3-dispatch heuristic.
- **PR #15's own frozen PR-scope evals carry two known, pre-existing defects** unrelated to the diff's correctness (flagged for the eval-grading agent, not fixed in the frozen `evals.json`): an `awk` range check whose start pattern is itself matched by its own end pattern (E4), and a file-existence assertion against a path shape (`skills/post-evals/SKILL.md`) that never matches how `/coderails:post-evals` is actually implemented (`commands/post-evals.md` + `scripts/post_evals.sh`) (E5).
