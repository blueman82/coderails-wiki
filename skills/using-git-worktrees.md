---
title: "using-git-worktrees"
type: skill
created: 2026-06-25
last_updated: 2026-06-25
sources: [sources/pr_19-30_self-containment-and-hardening.md]
tags: [skill, git, worktrees, parallel-agents]
---

# using-git-worktrees

Guides Claude through git worktree mechanics for parallel agent work — creating, populating, and cleaning up worktrees to allow multiple agents to work on isolated branches simultaneously.

## Trigger phrases

When a task requires parallel agent development on separate branches, or when "worktree" appears in the task context.

## Relationship to /workflow

Sits beside the workflow chain. Not invoked by `/workflow` itself, but invoked by skills like [[dispatching-parallel-agents]] when the parallel work needs isolated filesystem state.

## Key phases / steps

1. Create a new worktree from main (`git worktree add -b <branch> <path>`).
2. Orient the agent to work within the worktree path (absolute paths only).
3. Complete work and commit within the worktree.
4. Remove the worktree cleanly after merge (`git worktree remove`).

## Failure modes encoded

- Parallel agents stepping on each other's file changes by sharing a working directory.
- Forgetting to remove stale worktrees after merge.
- Using relative paths that resolve incorrectly in a worktree context.

## Source

`coderails/skills/using-git-worktrees/SKILL.md`

## See also

[[dispatching-parallel-agents]] — the skill that decides when to create worktrees  
[[subagent-driven-development]] — the execution skill used within each worktree  
[[self-containment]] — why this was vendored from superpowers
