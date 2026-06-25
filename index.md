---
title: Coderails Wiki Index
type: index
created: 2026-05-30
last_updated: 2026-06-25
sources: []
tags: [index, catalog]
---

# Coderails Wiki — Content Catalog

This is the first file Claude reads when answering queries about the coderails plugin.
Schema and maintenance protocols are in `/Users/harrison/Github/coderails/AGENTS.md`.

## What this plugin is

`coderails` is a Claude Code plugin (not an application) that ships three things:
1. A workflow command chain: `prep → push → merge → wiki`
2. Skills: `agentic-loop`, `test-driven-development`, `writing-plans`, `handoff`, `improve-prompt`, `planning-sequence`, `premortem`, `wiki-ingest`, `wiki-init`, `wiki-lint`, `wiki-query`, `using-coderails`, `using-git-worktrees`, `requesting-code-review`, `receiving-code-review`, `finishing-a-development-branch`, `dispatching-parallel-agents`, `systematic-debugging`, `verification-before-completion`, `writing-skills`, `subagent-driven-development`, `executing-plans`, `brainstorming`
3. A discipline loop: hooks that enforce confidence labels, verify-loop resolution, destructive-bash blocking, main-branch protection, and PR-workflow sequencing. **superpowers is no longer a dependency** — all core dev-workflow skills are vendored into the `coderails:` namespace. See [[self-containment]].

Source: `coderails/CLAUDE.md`. Plugin manifest: `coderails/.claude-plugin/plugin.json`.

---

## Commands

Slash commands in `coderails/commands/`. Advisory only — Claude chooses to invoke them.
See [[enforcement-model]] for why hooks enforce and commands do not.

| Command | File | Status |
|---|---|---|
| `/coderails:workflow` | `commands/workflow.md` | [[workflow]] |
| `/coderails:prep` | `commands/prep.md` | [[prep]] |
| `/coderails:push` | `commands/push.md` | [[push]] |
| `/coderails:merge` | `commands/merge.md` | [[merge]] |
| `/coderails:init` | `commands/init.md` | [[init]] |
| `/coderails:test-gate-setup` | `commands/test-gate-setup.md` | [[test-gate-setup]] |
| `/coderails:assumptions` | `commands/assumptions.md` | [[assumptions]] |
| `/coderails:disconfirm` | `commands/disconfirm.md` | [[disconfirm]] |
| `/coderails:notchecked` | `commands/notchecked.md` | [[notchecked]] |
| `/coderails:verify` | `commands/verify.md` | [[verify]] |

Config resolution pattern shared by `workflow.md`, `prep.md`, `push.md`, `init.md`: [[config-resolution]]

---

## Hooks

Hook event wiring in `coderails/hooks/hooks.json`. Scripts in `coderails/hooks/scripts/`.
See [[discipline-loop]] for the design rationale across all hooks.

| Script | Event | Mode | Status |
|---|---|---|---|
| `inject_bootstrap.sh` | `SessionStart` | silent — bootstraps session with `coderails:using-coderails` | [[inject_bootstrap]] |
| `inject_context.sh` | `UserPromptSubmit` | silent — injects `[ctx]` date/cwd/branch | [[inject_context]] |
| `discipline_catchup.sh` | `UserPromptSubmit` | warn — re-injects discipline nudge if prior response missed labels | [[discipline_catchup]] |
| `check_confidence_labels.sh` | `Stop` | **block** (exit 2) — ≥200-char response with no label | [[check_confidence_labels]] |
| `check_verify_loop.sh` | `Stop` | **block** (exit 2) — any untagged `## Did Not Verify` bullet (total enforcement; only `(unverifiable: …)` tag passes) | [[check_verify_loop]] |
| `loop_state_guard.sh` | `Stop` | **block** (exit 2) — agentic loop active but `progress.json` absent / session-mismatch (presence + ownership) | [[loop_state_guard]] |
| `loop_stall_guard.sh` | `Stop` | **block** (exit 2) — agentic loop active + incomplete + no `LOOP-STOP: <category>` declaration | [[loop_stall_guard]] |
| `destructive_bash_gate.sh` | `PreToolUse (Bash)` | **block** (permissionDecision: deny) | [[destructive_bash_gate]] |
| `test_gate.sh` | `PreToolUse (Bash)` | **block** on `git commit` if tests fail — opt-in via `.claude/test_command` | [[test_gate]] |
| `enforce_pr_workflow.sh` | `PreToolUse (Bash)` | **block** — `gh pr create` without prior `/push`; `gh pr merge` without prior `/review-pr` (NO_CONFIG opt-in) | [[enforce_pr_workflow]] |
| `no_edit_on_main.sh` | `PreToolUse (Write\|Edit\|MultiEdit)` | **block** — code-file edits on `main`/`master` | [[no_edit_on_main]] |

Stop hook order: `check_confidence_labels` → `check_verify_loop` → `loop_state_guard` (C1) → `loop_stall_guard` (C2). The two loop-state hooks share `hooks/scripts/lib/loop_state_common.sh` (vocab + active-loop detection) and `hooks/scripts/lib/agentic_loop_path.sh` (sole path authority). The three discipline hooks share `hooks/scripts/lib/discipline_common.sh` (transcript extraction + retry loop, added PR #29). See [[spec-plan-progress-artifact-chain]].

---

## Skills

Skills in `coderails/skills/*/SKILL.md`.

| Skill | Trigger summary | Status |
|---|---|---|
| `agentic-loop` | Multi-PR autonomous sessions; sits above `/workflow` | [[agentic-loop]] |
| `test-driven-development` | Worker construction discipline (red-green-refactor); code-guarded | [[test-driven-development]] |
| `writing-plans` | Turn a resolved spec into a durable task-by-task `plan.md` (Phase 2.8) | [[writing-plans]] |
| `handoff` | Generates memory file + continuation prompt for next session | [[handoff]] |
| `improve-prompt` | Rewrites underspecified prompts using 7 diagnostic foundations; Phase -1 of agentic-loop | [[improve-prompt]] |
| `planning-sequence` | Pre-Parade → Premortem → Red Team on a plan | [[planning-sequence]] |
| `premortem` | Backwards-failure-mode analysis on a plan or decision (standalone; embedded in planning-sequence Stage 2) | [[premortem]] |
| `wiki-ingest` | Primary wiki write operation; translates a PR or decision into permanent vault entries | [[wiki-ingest]] |
| `wiki-init` | One-time vault bootstrap: creates vault, seeds AGENTS.md, wires Obsidian + qmd | [[wiki-init]] |
| `wiki-lint` | Wiki structural integrity auditor: contradictions, stale pages, orphans, missing cross-refs | [[wiki-lint]] |
| `wiki-query` | Wiki read operation; answers questions with citations, files hard answers back as investigations | [[wiki-query]] |
| `using-coderails` | Session orientation; auto-loaded at SessionStart via inject_bootstrap hook | [[using-coderails]] |
| `using-git-worktrees` | Git worktree mechanics for parallel agent isolation | [[using-git-worktrees]] |
| `requesting-code-review` | PR review submission discipline; sits at push→review boundary | [[requesting-code-review]] |
| `receiving-code-review` | Acting on review findings; triage by severity | [[receiving-code-review]] |
| `finishing-a-development-branch` | Final verification before merge; branch wrap-up | [[finishing-a-development-branch]] |
| `dispatching-parallel-agents` | Fan-out to parallel subagents with worktree isolation | [[dispatching-parallel-agents]] |
| `systematic-debugging` | Hypothesis-then-verify loop before touching code | [[systematic-debugging]] |
| `verification-before-completion` | Structured checklist against requirements before declaring done | [[verification-before-completion]] |
| `writing-skills` | Meta-skill for authoring new coderails skills | [[writing-skills]] |
| `subagent-driven-development` | Delegation pattern: implementer prompts → subagent → verify loop | [[subagent-driven-development]] |
| `executing-plans` | Executing a `plan.md` task list with scope discipline | [[executing-plans]] |
| `brainstorming` | Structured ideation + Decision Ledger before committing to a design | [[brainstorming]] |

---

## Design

Architectural decisions and invariants.

- [[enforcement-model]] — hooks vs. commands; the central design distinction
- [[discipline-loop]] — how the six discipline hooks compose into a coherent loop
- [[config-resolution]] — dual-path `workflow.config.yaml` lookup used by all workflow commands
- [[install-and-cache-trap]] — idempotency contracts for `install.sh`/`uninstall.sh` and the reload-plugins caveat
- [[hook-exit-codes]] — which hook events block on exit 2; why coderails uses two block mechanisms
- [[repo-hosting]] — where the plugin + wiki repos live (private, `blueman82`), clone locations, `.gitignore` rules
- [[spec-plan-progress-artifact-chain]] — the agentic-loop's full spec→plan→progress durable-artifact chain + the two-hook (C1 presence/ownership, C2 anti-stall) loop-state guard architecture
- [[self-containment]] — the 2026-06-25 decision to vendor 12 superpowers skills into coderails; superpowers now uninstallable

---

## Investigations

Point-in-time filed investigations (`<topic>_<YYYY-MM-DD>.md` naming).

- [[install-cache-trap_2026-05-30]] — initial investigation filed at wiki bootstrap
- [[install-bash32-bad-substitution_2026-06-01]] — `${answer,,}` (bash 4) broke install.sh on macOS bash 3.2; fixed with `tr` lowercase

---

## Sources

Ingested PR and change records (`sources/pr_<N>_*.md`; session sources for direct-edit work).

- [[session_2026-05-31_verify-loop-hardening]] — verify-loop rework (meta-exclusion, threshold, string-coercion), cache-trap, exit-2 findings
- [[session_2026-05-31_prompting-doc-alignment]] — prompting-doc alignment: --no-verify gate, agentic-loop context/delegate guidance, discipline scope+floor clauses, workflow parallel-calls
- [[session_2026-06-01_verify-loop-total-enforcement]] — verify-loop escalated to total enforcement: source-token regex + meta_pattern removed, any untagged DNV bullet blocks, `(unverifiable: …)` is the sole escape
- [[session_2026-06-01_agentic-loop-delegate-all-impl]] — agentic-loop: main context becomes pure orchestrator; every code change (even single-file) goes to sonnet agent for impl+verify; two-rung delegation ladder; new Phase 3a (plugin commit 3c33f99)
- [[session_2026-06-25_agentic-loop-upgrade-arc]] — the agentic-loop upgrade arc (PRs #12–#18): Specs A (clean-migration), C1 (progress.json lifecycle), C2 (anti-stall), B (slim), D (vendored TDD), E (spec→plan artifacts), + #18 tool-agnostic cleanup
- [[pr_19-30_self-containment-and-hardening]] — self-containment + plugin hardening cluster (PRs #19–#30): 12 skills vendored from superpowers, SessionStart bootstrap hook, no_edit_on_main + enforce_pr_workflow enforcement hooks, install.sh chmod derivation, discipline_common.sh extraction

---

## Templates

Skeletons for each page type live in `templates/`. Use them when creating new pages:

- `templates/command.md`
- `templates/hook.md`
- `templates/skill.md`
- `templates/design.md`
- `templates/investigation.md`
- `templates/source.md`
