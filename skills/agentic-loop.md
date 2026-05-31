---
title: "Skill: agentic-loop"
type: skill
created: 2026-05-31
last_updated: 2026-05-31
sources:
  - skills/agentic-loop/SKILL.md
  - sources/session_2026-05-31_prompting-doc-alignment.md
tags: [skill, agentic-loop, multi-agent, orchestration, context-window, delegation]
---

# Skill: agentic-loop

The multi-agent orchestration discipline skill. Sits *above* `/workflow` — it uses `/workflow` as a subroutine for each PR in a loop. Covers autonomous multi-PR sessions where the user has waived per-step confirmation.

Source: `skills/agentic-loop/SKILL.md`

## When to load

Load immediately — before `/workflow`, `/prep`, `/push` — when the user authorises a sequence of agent-driven work. Trigger phrases (verified: SKILL.md:1 description block):

- "TeamCreate", "spawn a team", "no human gates", "self-merge", "crack on", "without the human", "no per-PR confirmation", "agentic loop", "multi-PR"
- Any authorisation of 3+ PRs in one instruction
- Autonomous merge + deploy + verify chains (even single-PR) when the user has waived per-step confirmation

Do NOT load for standard single-PR `/workflow` runs where the user is present at each gate.

## Structure: 13 phases + stop conditions

The skill defines 13 sequential phases plus stop conditions and a cadence note. Phases 4–7 repeat per PR/work-unit inside the loop. Key phases:

| Phase | Name | Core action |
|---|---|---|
| 0 | Read the authorisation envelope | `<thinking>` block: verbatim quote, envelope class, in/out-of-scope sub-actions |
| 1 | State the plan | Full plan in bullets; ask once to confirm |
| 2 | Pre-flight via spawned agent | Delegate `/planning-sequence`, `/premortem`, `/wiki-query` to sonnet agent, not main context |
| 3 | TeamCreate | Use `TeamCreate` tool for ≥3 sequential units; self-contained task descriptions |
| 4 | Spawn workers; don't block on idle pings | Check artifacts not pings |
| 5 | Disprove premise before each fix | Reproduce via SOT before spawning fix agent |
| 6 | Match confirmation to envelope | Don't ask for things inside the authorised scope |
| 7 | Skip-validation on cosmetic blockers | `./deploy --force --skip-drain --skip-validation` |
| 8 | Rebase before push on stale worktrees | `git fetch origin && git rebase origin/main` |
| 9 | Cluster wiki ingest | Run `/wiki-ingest` + `/wiki-lint` once at loop end, not per-PR |
| 10 | v2/v3 names when respawning | Dead agents keep pinging; versioned names identify the live one |
| 11 | Agent prompts include confidence labels | Pass the labelling standard into every spawned agent |
| 12 | Status reports are claims, not evidence | Re-check artifact at moment of action, not moment of report |

## When to delegate vs. work directly

Added 2026-05-31 (verified: SKILL.md:102):

Use subagents when:
- Tasks run in parallel
- Tasks need isolated context
- Tasks are genuinely independent workstreams

Work directly when:
- Single-file edits
- Sequential steps
- Work that requires shared context across steps

The handoff overhead of spawning an agent costs more than it saves on coupled, sequential work. TeamCreate is explicitly for ≥3 sequential units or cross-step dependency chains — not for "any multi-step task."

## Context-window persistence

Added 2026-05-31 (verified: SKILL.md:209):

Do not stop work early because the context window is filling or a token budget is approaching. Context will compact and the session will continue — treat that as a non-event, not a stop condition.

Before compaction happens, checkpoint state:
1. Commit all in-progress work to git
2. Write a brief progress note to a memory or a `progress.md` in the worktree
3. Record where the loop is in the phase sequence

**Git is the authoritative checkpoint.** Uncommitted work is unrecoverable state. Never declare "done" mid-loop because of token pressure. If a genuine stop condition is not met, keep going.

## Stop conditions

The loop runs autonomously until ANY of (verified: SKILL.md):
1. Verification failure that can't be auto-recovered (Phase 4 or Phase 12 artifact check fails)
2. Premise disproven (Phase 5 — symptom can't be reproduced via source of truth)
3. Genuinely ambiguous decision outside the authorisation envelope
4. Destructive/irreversible operation not previously authorised
5. All authorised work complete

On stop: report current state with confidence labels, propose the next move, then wait.

## Key architectural decisions encoded

- **Pre-flight agents use `model: sonnet`** — running skills, not architectural decisions; controls cost
- **Worker agents use `model: sonnet`** — orchestration pattern; main context handles decisions
- **Wiki ingest clusters, not per-PR** — see Phase 9; fragmented ingests produce fragmented wiki context
- **Scope-suppression in worker prompts goes first** — workers comply with prompt top; mid-section process notes get ignored (empirically observed failure)
- **Artifact verification not idle pings** — idle pings are noise; check git status, gh pr view, prod logs

## Cross-references

- [[enforcement-model]] — hooks vs. commands; agentic-loop is purely advisory (skill prose)
- [[discipline-loop]] — confidence labels and DNV rules propagate into spawned agent prompts
- [[session_2026-05-31_prompting-doc-alignment]] — source for changes 2 and 3 in this skill
