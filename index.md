---
title: Coderails Wiki Index
type: index
created: 2026-05-30
last_updated: 2026-05-31
sources: []
tags: [index, catalog]
---

# Coderails Wiki — Content Catalog

This is the first file Claude reads when answering queries about the coderails plugin.
Schema and maintenance protocols are in `/Users/harrison/Documents/Github/coderails/AGENTS.md`.

## What this plugin is

`coderails` is a Claude Code plugin (not an application) that ships three things:
1. A workflow command chain: `prep → push → merge → wiki`
2. Four skills: `agentic-loop`, `handoff`, `planning-sequence`, `premortem`
3. A discipline loop: hooks that enforce confidence labels, verify-loop resolution, and destructive-bash blocking

Source: `coderails/CLAUDE.md`. Plugin manifest: `coderails/.claude-plugin/plugin.json`.

---

## Commands

Slash commands in `coderails/commands/`. Advisory only — Claude chooses to invoke them.
See [[enforcement-model]] for why hooks enforce and commands do not.

| Command | File | Status |
|---|---|---|
| `/coderails:workflow` | `commands/workflow.md` | Not yet documented |
| `/coderails:prep` | `commands/prep.md` | Not yet documented |
| `/coderails:push` | `commands/push.md` | Not yet documented |
| `/coderails:merge` | `commands/merge.md` | Not yet documented |
| `/coderails:workflow-init` | `commands/workflow-init.md` | Not yet documented |
| `/coderails:test-gate-setup` | `commands/test-gate-setup.md` | Not yet documented |
| `/coderails:assumptions` | `commands/assumptions.md` | Not yet documented |
| `/coderails:disconfirm` | `commands/disconfirm.md` | Not yet documented |
| `/coderails:notchecked` | `commands/notchecked.md` | Not yet documented |
| `/coderails:verify` | `commands/verify.md` | Not yet documented |

Config resolution pattern shared by `workflow.md`, `prep.md`, `push.md`, `workflow-init.md`: [[config-resolution]]

---

## Hooks

Hook event wiring in `coderails/hooks/hooks.json`. Scripts in `coderails/hooks/scripts/`.
See [[discipline-loop]] for the design rationale across all hooks.

| Script | Event | Mode | Status |
|---|---|---|---|
| `inject_context.sh` | `UserPromptSubmit` | silent — injects `[ctx]` date/cwd/branch | Not yet documented |
| `discipline_catchup.sh` | `UserPromptSubmit` | warn — re-injects discipline nudge if prior response missed labels | Not yet documented |
| `check_confidence_labels.sh` | `Stop` | **block** (exit 2) — ≥200-char response with no label | Not yet documented |
| `check_verify_loop.sh` | `Stop` | **block** (exit 2) — DNV bullet names a source-resolvable file token | [[check_verify_loop]] |
| `destructive_bash_gate.sh` | `PreToolUse (Bash)` | **block** (permissionDecision: deny) | Not yet documented |
| `test_gate.sh` | `PreToolUse (Bash)` | **block** on `git commit` if tests fail — opt-in via `.claude/test_command` | Not yet documented |

---

## Skills

Skills in `coderails/skills/*/SKILL.md`.

| Skill | Trigger summary | Status |
|---|---|---|
| `agentic-loop` | Multi-PR autonomous sessions; sits above `/workflow` | Not yet documented |
| `handoff` | Generates memory file + continuation prompt for next session | Not yet documented |
| `planning-sequence` | Pre-Parade → Premortem → Red Team on a plan | Not yet documented |
| `premortem` | Backwards-failure-mode analysis on a plan or decision | Not yet documented |

---

## Design

Architectural decisions and invariants.

- [[enforcement-model]] — hooks vs. commands; the central design distinction
- [[discipline-loop]] — how the three discipline hooks compose into a coherent loop
- [[config-resolution]] — dual-path `workflow.config.yaml` lookup used by all workflow commands
- [[install-and-cache-trap]] — idempotency contracts for `install.sh`/`uninstall.sh` and the reload-plugins caveat
- [[hook-exit-codes]] — which hook events block on exit 2; why coderails uses two block mechanisms

---

## Investigations

Point-in-time filed investigations (`<topic>_<YYYY-MM-DD>.md` naming).

- [[install-cache-trap_2026-05-30]] — initial investigation filed at wiki bootstrap

---

## Sources

Ingested PR and change records (`sources/pr_<N>_*.md`; session sources for direct-edit work).

- [[session_2026-05-31_verify-loop-hardening]] — verify-loop rework (meta-exclusion, threshold, string-coercion), cache-trap, exit-2 findings

---

## Templates

Skeletons for each page type live in `templates/`. Use them when creating new pages:

- `templates/command.md`
- `templates/hook.md`
- `templates/skill.md`
- `templates/design.md`
- `templates/investigation.md`
- `templates/source.md`
