---
title: "executing-plans"
type: skill
created: 2026-06-25
last_updated: 2026-06-25
sources: [sources/pr_19-30_self-containment-and-hardening.md]
tags: [skill, plans, execution, tasks, orchestration]
---

# executing-plans

Governs how to execute a `plan.md` — reading the task list, sequencing work, tracking progress, and knowing when to deviate vs. follow the plan as written.

## Trigger phrases

When a plan file exists and work needs to begin; "execute the plan", "start implementing", "work through the plan". Invoked after [[writing-plans]] produces the plan artifact.

## Relationship to /workflow

Downstream of [[writing-plans]], upstream of `/coderails:push`. The plan provides the task sequence; this skill governs execution discipline within that sequence.

## Key phases / steps

1. Read `plan.md` in full before touching any code.
2. Execute tasks in order unless a dependency reason exists to reorder.
3. Tick tasks off in `plan.md` as they complete (or mark in `progress.json` if inside agentic-loop).
4. When reality diverges from the plan — pause, assess, update plan.md explicitly before continuing.
5. Don't improvise scope mid-execution; surface additions as plan amendments.

## Failure modes encoded

- Starting with the "interesting" task rather than the sequenced first task.
- Silently skipping a task because it seems hard, then reporting done.
- Adding scope mid-execution without updating the plan (plan becomes stale).
- Treating the plan as a checklist to tick mechanically rather than a guide to verify against.

## Source

`coderails/skills/executing-plans/SKILL.md`

## See also

[[writing-plans]] — produces the plan artifact this skill consumes  
[[subagent-driven-development]] — execution pattern for individual tasks  
[[agentic-loop]] — uses plan.md as a durable artifact (Phase 2.8)  
[[spec-plan-progress-artifact-chain]] — how plan.md fits in the full artifact chain
