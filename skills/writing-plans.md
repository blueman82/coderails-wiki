---
title: "Skill: writing-plans"
type: skill
created: 2026-06-25
last_updated: 2026-07-11
sources:
  - sources/session_2026-06-25_agentic-loop-upgrade-arc.md
  - sources/pr_50_planning-sequence-gate.md
  - sources/pr_15-17_loop-hardening-registration-eval-freeze-ledger-dry.md
  - sources/pr_138_remove-specs-plans-tracking.md
tags: [skill, writing-plans, planning, decomposition, artifact-chain, eval-freeze, task-evals]
---

# Skill: writing-plans

The plan-writing skill for the agentic-loop's Phase 2.7b (formerly Phase 2.8, merged into Phase 2.7 by PR #86). Turns a resolved spec/design into an ordered set of self-contained tasks, each dispatchable to a worker without re-reading the conversation. Coderails-owned; no cross-plugin dependency.

Source: `coderails/skills/writing-plans/SKILL.md`
Companion: `coderails/skills/writing-plans/plan-anti-patterns.md`
Invoked as: `coderails:writing-plans`

## Trigger

```
'Use when you have a spec or resolved design for a multi-step task and need a
durable, task-by-task implementation plan before any code is written — each task
with exact files, interfaces, bite-sized steps, and verify-criteria. Not for
single trivial edits, which need no separate plan.'
```

**Complexity guard is load-bearing.** "Not for single trivial edits" mirrors the agentic-loop Phase 2.7b guard (≥3 work-units or a cross-unit dependency). A plan on trivial work is ceremony that trains the loop to skip it on real work.

## When this skill was added

Spec E (#17) — vendored from superpowers' writing-plans discipline. Same vendor-not-reference decision as Spec D: keeps coderails a self-contained zip.

## Key content

Each plan task carries four mandatory elements:
1. **Exact files** (`src/auth/validator.py:42-89`; no "somewhere in the auth module").
2. **Interfaces**: what this task consumes (exact signatures) and produces (exact function names, types) for neighboring tasks.
3. **Steps**: one action each, 2–5 minutes. "Write the failing test" is a step; "implement auth" is not.
4. **Verify-criteria**: runnable or inspectable — a test command with expected output, a grep, a visible UI state.

**Per-task construction method** references `coderails:test-driven-development` when the task is code (adds/alters a function, method, or branch that can carry a test). Docs/config/prose tasks verify by inspection.

**Self-review gate** (before the plan is final): every spec requirement maps to a task; placeholder scan; type consistency check.

## Mandatory planning-sequence gate

Since [[pr_50_planning-sequence-gate]] (2026-06-26), the skill's flow has a required adversarial step *after* the self-review gate and *before* implementation hand-off:

```
write plan → self-review gate → /coderails:planning-sequence → fold findings in → hand off to implementation
```

The `## Stress-test before implementation (required)` section instructs running [[planning-sequence]] (Pre-Parade → Premortem → Red Team) on the written plan, then folding its findings back into the plan inline — add tasks for gaps, tighten weak verify-criteria, record consciously-accepted failure modes — before handing off to `coderails:subagent-driven-development`/`coderails:executing-plans`. In an agentic-loop run, the sequence is delegated to a sub-agent (not main context), per [[agentic-loop]]; only the venue changes, not the gate. (verified — SKILL.md gate section)

This is **advisory, not mechanical** — no hook blocks an implementation that skips it. The complexity guard still applies: the gate rides on the skill, which itself does not fire for single trivial edits. (verified)

## Eval freeze happens once, at plan completion — not as the plan's final task ([[pr_15-17_loop-hardening-registration-eval-freeze-ledger-dry|PR #15]], F2)

The skill previously had one section — "Final eval-gate task" — that said both (a) every plan ends with a task invoking `/coderails:task-evals`, and (b) freeze-before-build means the evals are frozen before implementation starts. In practice (a) swallowed (b): a 2026-07-06 17-unit plan encoded freeze-and-grade as a single end-stage task, and the execution ran 8 tasks deep with no frozen `evals.json` at all — [[task-evals]]'s own freeze-before-build anti-gaming rule went structurally unenforced by the plan's own wording.

PR #15 splits this into two explicit sections:

1. **"Freeze evals after stress-test, before implementation"** — placed immediately after the `## Stress-test before implementation (required)` section above, as its explicit final step. This is where `/coderails:task-evals` is actually invoked and the `evals.json` frozen (`frozen_at`/`frozen_sha` populated) — before Task 1 is ever dispatched to a worker.
2. **"Final task: grade and post only"** — the plan's actual last task now only runs `/coderails:post-evals` against the already-frozen artifact. It never generates or (re-)freezes anything.

The corrected flow:

```
write plan → self-review gate → /coderails:planning-sequence → fold findings in
  → freeze evals (task-evals) → hand off to implementation → … → grade + post (final task)
```

Four cross-referencing docs were fixed in lockstep so none still describes freezing as happening "as the final task": [[task-evals]]'s invocation-point bullet, [[subagent-driven-development]]'s new Pre-Flight Plan Review bullet (a belt-and-braces pre-flight assertion that `evals.json` already has a non-empty `frozen_at`/`frozen_sha` before Task 1 is dispatched — stop and freeze first if not), `commands/workflow.md:157`, and `docs/REFERENCE.md:83`.

## Relationship to agentic-loop

Phase 2.7b (formerly 2.8, merged by PR #86) invokes `coderails:writing-plans` to produce `plan.md` in the loop-state dir — outside the repo, beside `progress.json`, already ephemeral and unaffected by the change below. Two consumption directions are stated explicitly in Phase 2.7b (not in the `## Context-window persistence` section, which is a no-touch region):
- Phase 3 builds its task list directly from `plan.md` (not from conversation state).
- After any compaction, the orchestrator re-reads `plan.md` to recover *scope* the same way it re-reads `progress.json` to recover *position*. `plan.md` is the static SSOT; `progress.json` is the dynamic cursor against it.

**Direct (non-loop) invocation, superseded 2026-07-11.** Outside an agentic-loop, this skill used to write its plan to `docs/coderails/plans/<name>.md` and commit it as a permanent repo artifact — documented in `docs/REFERENCE.md`'s Artifact-and-State-Locations table, not in this skill's own SKILL.md (which never hardcoded the path). Per [[pr_138_remove-specs-plans-tracking|PR #138]], `docs/coderails/plans/` is now gitignored; that convention no longer applies. A directly-invoked plan is now a session-local working document only, same treatment as the loop-scope `plan.md` above.

## E→D tie

The per-task construction step in this skill references `coderails:test-driven-development`. This is the designed connection: the decomposition artifact (plan) carries the construction method (TDD). When a worker reads their task from `plan.md`, the construction method is already in the task.

## Failure modes encoded

- Tasks that say "implement X" without specifying files, interfaces, or steps (a worker cannot start without re-reading the conversation).
- `TODO`, `TBD`, "handle edge cases" without showing how.
- "Similar to Task N" — tasks must be self-contained; a worker may read tasks out of order.
- No self-review gate — tasks that don't trace to spec requirements leave requirements unimplemented.
- Writing a plan for a single-agent-simple task (ceremony that trains skipping on real work).

## See also

- [[planning-sequence]] — required next step after the self-review gate (the mandatory planning-sequence gate)
- [[test-driven-development]] — referenced in per-task construction step (E→D tie)
- [[agentic-loop]] — Phase 2.7b (formerly 2.8) invokes this skill; Phase 3 consumes the output
- [[spec-plan-progress-artifact-chain]] — how `plan.md` fits in the full artifact chain
- [[task-evals]] — the skill invoked at the freeze-after-stress-test step (F2 fix, PR #15)
- [[subagent-driven-development]] — gained a Pre-Flight Plan Review bullet asserting evals are already frozen before Task 1 dispatch (F2 belt-and-braces, PR #15)
- [[loop-progress-fields]] — consolidating page for `work_units`, the field this skill's plan decomposition ultimately populates via `progress.json`
- [[pr_15-17_loop-hardening-registration-eval-freeze-ledger-dry]] — PR #15 source record: the freeze-timing restructure
- [[pr_138_remove-specs-plans-tracking]] — the 2026-07-11 change removing the direct-invocation plan output from repo tracking (loop-scope `plan.md` unaffected — already ephemeral)
