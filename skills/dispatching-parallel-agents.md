---
title: "dispatching-parallel-agents"
type: skill
created: 2026-06-25
last_updated: 2026-06-25
sources: [sources/pr_19-30_self-containment-and-hardening.md]
tags: [skill, agents, parallel, worktrees, orchestration]
---

# dispatching-parallel-agents

Governs when and how to fan work out to parallel subagents: task decomposition, worktree setup, output collection, and merging results.

## Trigger phrases

When a task has genuinely independent subtasks that can run concurrently; "dispatch agents", "parallel agents", "fan out". Also referenced by [[agentic-loop]] for multi-PR sessions.

## Relationship to /workflow

Above the workflow chain when multiple independent PRs are needed. Each subagent runs its own `prep → push → merge` loop on its own branch via a worktree.

## Key phases / steps

1. Identify truly independent subtasks (no shared state or file overlap).
2. Create one worktree per agent via [[using-git-worktrees]].
3. Send all agents in a single message (parallelism requires single-message dispatch).
4. Collect results; merge branches in dependency order.
5. Remove worktrees after merge.

## Failure modes encoded

- Dispatching agents on tasks with shared state, causing merge conflicts.
- Dispatching sequentially when tasks are independent (wasted time).
- Forgetting to pass absolute paths to agents (relative paths break in worktrees).
- Not collecting and reconciling agent outputs before reporting done.

## Source

`coderails/skills/dispatching-parallel-agents/SKILL.md`

## See also

[[using-git-worktrees]] — the mechanics of worktree isolation  
[[subagent-driven-development]] — the execution pattern inside each subagent  
[[agentic-loop]] — uses parallel dispatch for multi-PR sessions
