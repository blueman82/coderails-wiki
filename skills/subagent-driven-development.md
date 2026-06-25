---
title: "subagent-driven-development"
type: skill
created: 2026-06-25
last_updated: 2026-06-25
sources: [sources/pr_19-30_self-containment-and-hardening.md]
tags: [skill, agents, delegation, implementation, orchestration]
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

## Failure modes encoded

- Implementer prompts so vague that subagents invent requirements.
- Main context drifting into implementation mode (breaks the delegation model).
- Accepting subagent output without verification.
- Not including file paths and line-number constraints in the prompt (agent invents structure).

## Source

`coderails/skills/subagent-driven-development/SKILL.md`  
`coderails/skills/subagent-driven-development/implementer-prompt.md`  
`coderails/skills/subagent-driven-development/task-reviewer-prompt.md`

## See also

[[dispatching-parallel-agents]] — when multiple subagents run simultaneously  
[[executing-plans]] — the plan-driven execution skill  
[[agentic-loop]] — Phase 3 uses this skill explicitly  
[[self-containment]] — why this was vendored: agentic-loop needed a coderails-native reference
