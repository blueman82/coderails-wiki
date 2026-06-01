---
title: "Session: agentic-loop delegate-all-impl-to-sonnet"
type: source
origin: plugin commit 3c33f99 (feat(agentic-loop): delegate all impl+verify to sonnet agents)
created: 2026-06-01
last_updated: 2026-06-01
tags: [source, session, skill, agentic-loop, delegation, sonnet, orchestration]
---

# Session 2026-06-01 — agentic-loop delegate-all-impl-to-sonnet

Plugin commit `3c33f99` in `github.com/blueman82/coderails` updated
`skills/agentic-loop/SKILL.md` to make **main context a pure orchestrator
that never implements**. Every code change — even a single-file edit — is
delegated to a spawned sonnet agent that does the implementation AND verifies
its own artifact before reporting back. (verified: git show 3c33f99 --stat)

## Why this change was made

Two reasons, stated explicitly in Phase 3 of the skill: (verified: SKILL.md Phase 3)

1. **Keep main context clean.** Opus context is scarce and fills fast in long
   sessions. Even small file edits consume tokens that should be reserved for
   orchestration decisions.
2. **Keep cost down.** Sonnet does the typing; opus orchestrates. Implementation
   and mechanical verification are execution tasks — no architectural reasoning
   needed.

## What changed in SKILL.md (commit 3c33f99)

- **Frontmatter description updated** to advertise the single-agent delegation
  trigger — not just TeamCreate / multi-PR. Now explicitly covers "even a
  single-file edit" going to a sonnet agent. (verified: SKILL.md description line)

- **Phase 3 retitled** from "TeamCreate for ≥3 units" to "Delegate all
  implementation to sonnet agents; TeamCreate when work has ≥3 sequential units
  or dependency chains". Reframed as a **two-rung delegation ladder**: (1)
  single sonnet `Agent` for impl+verify = the default for 1–2 self-contained
  units; (2) `TeamCreate` for ≥3 PRs or dependency chains. (verified: SKILL.md Phase 3)

- **New Phase 3a added:** "Single sonnet agent for impl + verify (the
  TeamCreate-is-overkill case)". Defines the prompt contract the agent must
  receive: `model: sonnet` (non-negotiable), exact change + success criteria,
  a verify step the agent runs itself before reporting, confidence-labelled
  report-back, commit-for-durability instruction, and the escalate-to-TeamCreate
  trigger. (verified: SKILL.md Phase 3a)

- **Old guidance removed:** the prior Phase 3 included "work directly rather
  than delegating for single-file edits / sequential steps." This was removed
  because it contradicted the main-context-is-pure-orchestrator rule. The
  correct question is now "which rung?" not "delegate vs. do it yourself?"
  (verified: SKILL.md Phase 3)

- **Phase numbering unchanged.** Still 13 phases (0–12); Phase 3a is a
  sub-phase insert, not a renumber. (verified: SKILL.md)

## Delegation ladder (as defined post-commit)

| Rung | Trigger | Tool |
|---|---|---|
| 1 (default) | 1–2 self-contained units (bug fix, one PR, single-file change) | Single `Agent` with `model: sonnet` — does impl AND verify |
| 2 | ≥3 PRs or cross-step dependency chains | `TeamCreate` with `blockedBy` task list |

Main context: reads for orchestration decisions, plans, delegates, spot-checks
at dependency boundaries. Never calls `Edit`/`Write`/`MultiEdit` itself inside
an authorised loop.

## Related

- [[agentic-loop]]
