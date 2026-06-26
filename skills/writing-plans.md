---
title: "Skill: writing-plans"
type: skill
created: 2026-06-25
last_updated: 2026-06-26
sources:
  - sources/session_2026-06-25_agentic-loop-upgrade-arc.md
  - sources/pr_50_planning-sequence-gate.md
tags: [skill, writing-plans, planning, decomposition, artifact-chain]
---

# Skill: writing-plans

The plan-writing skill for the agentic-loop's Phase 2.8. Turns a resolved spec/design into an ordered set of self-contained tasks, each dispatchable to a worker without re-reading the conversation. Coderails-owned; no cross-plugin dependency.

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

**Complexity guard is load-bearing.** "Not for single trivial edits" mirrors the agentic-loop Phase 2.8 guard (≥3 work-units or a cross-unit dependency). A plan on trivial work is ceremony that trains the loop to skip it on real work.

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

## Relationship to agentic-loop

Phase 2.8 invokes `coderails:writing-plans` to produce `plan.md` in the loop-state dir. Two consumption directions are stated explicitly in Phase 2.8 (not in the `## Context-window persistence` section, which is a no-touch region):
- Phase 3 builds its task list directly from `plan.md` (not from conversation state).
- After any compaction, the orchestrator re-reads `plan.md` to recover *scope* the same way it re-reads `progress.json` to recover *position*. `plan.md` is the static SSOT; `progress.json` is the dynamic cursor against it.

## E→D tie

The per-task construction step in this skill references `coderails:test-driven-development`. This is the designed connection: the decomposition artifact (plan) carries the construction method (TDD). When a worker reads their task from `plan.md`, the construction method is already in the task.

## Failure modes encoded

- Tasks that say "implement X" without specifying files, interfaces, or steps (a worker cannot start without re-reading the conversation).
- `TODO`, `TBD`, "handle edge cases" without showing how.
- "Similar to Task N" — tasks must be self-contained; a worker may read tasks out of order.
- No self-review gate — tasks that don't trace to spec requirements leave requirements unimplemented.
- Writing a plan for a single-agent-simple task (ceremony that trains skipping on real work).

## See also

- [[test-driven-development]] — referenced in per-task construction step (E→D tie)
- [[agentic-loop]] — Phase 2.8 invokes this skill; Phase 3 consumes the output
- [[spec-plan-progress-artifact-chain]] — how `plan.md` fits in the full artifact chain
