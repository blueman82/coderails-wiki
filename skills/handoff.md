---
title: "Skill: handoff"
type: skill
created: 2026-06-25
last_updated: 2026-06-25
sources: [skills/handoff/SKILL.md]
tags: [skill, handoff, session-continuity, memory, context-preservation]
---

# Skill: handoff

Generates two artefacts — a project memory file and a continuation prompt — so a fresh Claude Code session can continue the current work without re-deriving context.

Source: `coderails/skills/handoff/SKILL.md`
Invoked as: `coderails:handoff`

## Trigger phrases

```
"handoff", "hand off", "continue in new session", "pick this up later",
"save this for next session", "create a handoff"
```

Also fires proactively when a session is running long and the user signals they want to wrap up but continue later.

## What it produces

1. A **project memory file** — written to `~/.claude/projects/<project>/memory/` using a specific 8-category schema (goal, decisions, constraints, taxonomy/schema, key files, done so far, next steps, open questions).
2. A **continuation prompt** — a fenced code block ready to paste into a new session. Minimal — references the memory file by name and names the immediate next action.

## Key design decisions encoded

**Memory path derivation is non-trivial.** The skill derives the memory store path from cwd, but monorepo sub-projects are a trap: `projects/<sub>` has its own cwd-derived path that no future session will load, because sessions are keyed to the repo root. The skill explicitly walks up the directory tree, compares candidates, and picks the store that already holds a populated `MEMORY.md` — not necessarily the cwd-derived one. (verified: SKILL.md Step 2 monorepo edge case block)

**Continuation prompt is intentionally thin.** The memory file carries all detail. The prompt just references the file by name and states the immediate next action. This keeps the prompt pasteable without being a copy of the memory file.

**Session confirms before close.** Step 4 always shows the user the memory path and continuation prompt, then asks if anything is missing. The skill never silently writes and declares done.

## Failure modes encoded

- Writing to a cwd-derived path inside a monorepo sub-project, creating an orphaned memory file no session loads from.
- Writing verbose raw session content into the memory file (tool outputs, debugging dead-ends) that just bloats context without aiding the next session.
- Including anything derivable from git log or reading the code — the new session can re-derive those.

## Relationship to agentic-loop

[[agentic-loop]] Phase 9 clusters wiki ingest at loop end; `handoff` serves a parallel role at session end for human-driven (non-loop) work. The [[agentic-loop]] skill itself writes `progress.json` as its shareable state record — `handoff` is the human-session equivalent for context that doesn't live in an artifact file.

## See also

- [[agentic-loop]] — autonomous session state via `progress.json`; handoff is the human-session analogue
- [[writing-plans]] — sessions that produce a plan often need a handoff to carry the plan into the implementation session
