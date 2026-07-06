---
title: "subagent-driven-development"
type: skill
created: 2026-06-25
last_updated: 2026-07-06
sources:
  - sources/pr_19-30_self-containment-and-hardening.md
  - sources/pr_15-17_loop-hardening-registration-eval-freeze-ledger-dry.md
tags: [skill, agents, delegation, implementation, orchestration, sdd-ledger, pre-flight, eval-freeze]
---

# subagent-driven-development

The execution pattern for implementing tasks via subagents: how to write implementer prompts, review subagent output, and loop until the task is verified — keeping the main context as orchestrator.

## Trigger phrases

When Claude needs to implement code using subagents; "subagent-driven-development", "implement via subagent". Also referenced by [[agentic-loop]] Phase 3 for worker-prompt construction.

## Relationship to /workflow

Core execution layer. The workflow chain uses this pattern inside the code/iterate phase. agentic-loop Phase 3 explicitly references `coderails:subagent-driven-development` for worker prompts.

## Key phases / steps

1. Write a tight implementer prompt: task, files, constraints, success criteria.
2. Dispatch the subagent (`Agent` tool, isolation: worktree if needed).
3. Review the subagent's output against success criteria.
4. Loop until verified — don't accept "I believe it's done".
5. Main context stays orchestrator; no direct code edits in the main thread.

## Pre-Flight Plan Review gained an evals-frozen assertion ([[pr_15-17_loop-hardening-registration-eval-freeze-ledger-dry|PR #15]], F2 belt-and-braces)

Before dispatching Task 1, the Pre-Flight Plan Review scan list now includes a bullet asserting `evals.json` exists with a non-empty `frozen_at`/`frozen_sha` — if absent, stop and invoke `/coderails:task-evals` first rather than proceeding into implementation. This is the second, independent guard against the same failure [[writing-plans]]'s freeze-timing fix addresses at the source: even if a plan somehow reaches this skill without evals frozen (a stale plan, a hand-edited one, or the wording bug PR #15 itself just fixed), this pre-flight check catches it before Task 1 ever dispatches. Belt-and-braces, not redundant — the two fixes protect different failure points (plan authoring vs. plan execution).

## Ledger relocated to the session-keyed loop-state dir ([[pr_15-17_loop-hardening-registration-eval-freeze-ledger-dry|PR #16]], WU4)

The progress ledger moved from a repo-root-keyed `$root/.coderails/sdd` directory — shared by **every** session working in that checkout, a latent collision bug for concurrent SDD runs, and wholesale-destroyed by `git clean -fdx` — to the session-keyed loop-state directory: the same directory `progress.json` lives in, resolved via `hooks/scripts/lib/agentic_loop_path.sh` + `dirname`. Clean-break migration: no shim, no fallback to the old path. The ledger itself is renamed `sdd-ledger.md`, distinct from `progress.json`/`progress.md`.

**Fact-ownership split** (now explicit): the ledger is **task-level truth with commit ranges, controller-owned**; `progress.json.work_units[id].status` is **unit-level truth, orchestrator-owned**. These are two different levels of granularity tracked by two different owners, not duplicate state.

`sdd-workspace` (the script resolving this path) fails loud on two conditions rather than silently degrading: an unset `CLAUDE_CODE_SESSION_ID` (the path helper's own per-call session-id fallback is non-idempotent — tolerable for a stateless reader, fatal for a ledger owner needing the same path across repeated calls) and non-absolute helper output (a previously-silent cwd-fallback bug, reproduced during PR #16's own review). First dedicated test infrastructure for any SDD script: `sdd-workspace.test.sh`, a 6-check smoke test written TDD-style (red against the old script, green against the new one).

**One-time migration risk, accepted not handled:** any session with an in-flight ledger at the old path when PR #16 merged loses continuity with it — no shim bridges the two. Recovery: `git log` remains authoritative for what actually happened, per this skill's pre-existing "trust the ledger and git log" contract.

## Failure modes encoded

- Implementer prompts so vague that subagents invent requirements.
- Main context drifting into implementation mode (breaks the delegation model).
- Accepting subagent output without verification.
- Not including file paths and line-number constraints in the prompt (agent invents structure).
- A plan reaching Task 1 dispatch with unfrozen evals — caught by the Pre-Flight Plan Review bullet (PR #15).
- Two concurrent SDD sessions in the same checkout colliding on ledger/task-brief/review-package state — closed by the session-keyed ledger relocation (PR #16).

## Source

`coderails/skills/subagent-driven-development/SKILL.md`  
`coderails/skills/subagent-driven-development/implementer-prompt.md`  
`coderails/skills/subagent-driven-development/task-reviewer-prompt.md`
`coderails/skills/subagent-driven-development/scripts/sdd-workspace`

## See also

[[dispatching-parallel-agents]] — when multiple subagents run simultaneously  
[[executing-plans]] — the plan-driven execution skill  
[[agentic-loop]] — Phase 3 uses this skill explicitly; `sdd-ledger.md` is a sibling file in the same session-keyed dir  
[[self-containment]] — why this was vendored: agentic-loop needed a coderails-native reference  
[[writing-plans]] — the freeze-timing fix (PR #15) this skill's pre-flight bullet backstops  
[[task-evals]] — the skill invoked if the pre-flight assertion finds no frozen evals  
[[loop-progress-fields]] — consolidating page for `work_units` (orchestrator-owned, unit-level truth) vs. this skill's own ledger (task-level, controller-owned) fact-ownership split  
[[pr_15-17_loop-hardening-registration-eval-freeze-ledger-dry]] — PR #15 (pre-flight bullet) and PR #16 (ledger relocation) source record
