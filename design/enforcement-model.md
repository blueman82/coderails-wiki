---
title: Enforcement Model
type: design
created: 2026-05-30
last_updated: 2026-07-06
sources:
  - commands/workflow.md
  - CLAUDE.md
  - hooks/hooks.json
  - sources/pr_43_rough-edges.md
  - sources/pr_57-62_subagent-enforcement-gate-hardening.md
  - sources/pr_86_agentic-loop-hardening.md
  - sources/pr_15-17_loop-hardening-registration-eval-freeze-ledger-dry.md
tags:
  - hooks
  - enforcement
  - design-law
  - slash-commands
  - self-attestation
---

# Enforcement Model

The most important design law in coderails. Get it wrong and you ship a system that *looks* like it enforces things but doesn't.

## The Law

**Hooks are mechanical enforcement. Slash commands are advisory.**

From `CLAUDE.md` lines 42–49 (verified):

> **Hooks = mechanical enforcement.** They run automatically on lifecycle events and can *block* (exit 2 / `permissionDecision: deny`). Use a hook when behaviour must be enforced regardless of whether Claude cooperates.
> **Slash commands = advisory.** Claude has to *choose* to invoke them. Use a command to encode a workflow, not to enforce one.
>
> If you're asked to "make X mandatory," that belongs in a `PreToolUse` hook, not a command.

`commands/workflow.md` lines 184–186 states this again as a design negative (verified):

> **Not enforcement.** Slash commands are advisory — Claude has to choose to invoke them. Mechanical enforcement (refusing `gh pr create` unless `/push` ran, refusing `gh pr merge` unless `/pr-review-toolkit:review-pr` ran) belongs in `PreToolUse` hooks, not here. See the companion enforce-pr-workflow.sh hook design.

## Why This Matters

A slash command that says "you must do X before Y" can be ignored. There is nothing stopping Claude — or a user — from running `gh pr merge` directly, bypassing `/coderails:push` entirely. The command encodes the *happy path* workflow. It is not a gate.

A `PreToolUse` hook on `Bash(gh pr merge*)` fires whether or not any slash command was involved. Claude cannot skip it. The user cannot skip it. That is the distinction.

## How Hooks Block

Stop hooks (fired after a response is generated) block by calling `exit 2` with a message on stderr. The harness then shows the message and forces a re-generate. Example: `check_confidence_labels.sh` lines 65–66 (verified):

```bash
echo "[discipline-block] response made substantive claims without (verified)/(inferred)/(guess) labels. Add them before stopping." >&2
exit 2
```

PreToolUse hooks block by emitting a JSON response with `permissionDecision: "deny"`. Example: `destructive_bash_gate.sh` uses this pattern (inferred from CLAUDE.md:68 and hook conventions).

## Current Hook Map

| Event | Script | Mode |
|---|---|---|
| `SessionStart` | [[inject_bootstrap]] (`inject_bootstrap.sh`) | silent — bootstraps session with `coderails:using-coderails` context |
| `UserPromptSubmit` | [[inject_context]] (`inject_context.sh`) | silent — prepends `[ctx]` (cwd, branch, date) |
| `UserPromptSubmit` | [[discipline_catchup]] (`discipline_catchup.sh`) | warn |
| `Stop` | [[check_confidence_labels]] (`check_confidence_labels.sh`) | **block** — ≥200-char response with no confidence label |
| `Stop` | [[check_verify_loop]] (`check_verify_loop.sh`) | **block** — any untagged DNV bullet (file_count gate removed PR #61) |
| `Stop` | [[loop_state_guard]] (`loop_state_guard.sh`) | **block** — agentic-loop active but progress.json absent/mismatched |
| `Stop` | [[loop_stall_guard]] (`loop_stall_guard.sh`) | **block** — agentic-loop active + incomplete + no LOOP-STOP declaration |
| `Stop` | [[unregistered_loop_guard]] (`unregistered_loop_guard.sh`) | **nudge, not block** — ≥3 distinct sequential `Agent` tool_use `message.id`s + no `progress.json` + no `agentic-loop` skill invocation; delivers via `additionalContext` with exit 0 (added PR #17) |
| `SubagentStop` | [[check_confidence_labels]] (`check_confidence_labels.sh`) | **block** — same as Stop; reads `last_assistant_message` (added PR #57) |
| `SubagentStop` | [[check_verify_loop]] (`check_verify_loop.sh`) | **block** — same as Stop; reads `last_assistant_message`, no file_count gate (added PR #57) |
| `PreToolUse` (Bash) | [[destructive_bash_gate]] (`destructive_bash_gate.sh`) | **block** — permanent blocklist + in-Bash source edits on main (extended PR #59) |
| `PreToolUse` (Bash) | [[test_gate]] (`test_gate.sh`) | **block** on `git commit` if tests fail — opt-in only |
| `PreToolUse` (Bash) | [[enforce_pr_workflow]] (`enforce_pr_workflow.sh`) | **block** — `gh pr create` without prior `/push`; `gh pr merge`/`git merge`/`git push` without prior `/review-pr` (per-PR + consume-on-use + positional push, PR #58) |
| `PreToolUse` (Write\|Edit\|MultiEdit) | `no_edit_on_main.sh` | **block** — source files (allowlist model PR #60): everything except doc/config/special dotfiles on main/master |

`discipline_catchup.sh` is the only surviving warn-mode hook. `unregistered_loop_guard.sh` is the only nudge-mode Stop hook — a deliberate deviation from every sibling loop-state hook (which all block on ground truth): it has only a heuristic, not ground truth, so nudge is the honest posture; a nudge delivered but ignored is the recorded trigger to upgrade it to a block in a future PR. Everything else that should be enforced has been promoted to block-mode or moved to a PreToolUse gate. See [[discipline-loop]] for the history of why warn-mode was abandoned and for the enforcement ceilings. (updated 2026-07-06 — added `unregistered_loop_guard.sh`, PR #17; previously updated 2026-06-26 — added SubagentStop hooks; updated destructive_bash_gate, enforce_pr_workflow, no_edit_on_main descriptions)

## When to Use Which

| Goal | Mechanism |
|---|---|
| Encode a multi-step workflow | Slash command (`commands/*.md`) |
| Prevent a specific tool call unless conditions met | `PreToolUse` hook |
| Enforce a constraint on every response | `Stop` hook |
| Share reusable workflow logic across commands | `scripts/lib/git-common.sh` |

If someone asks "can we make the engineering-principles check mandatory before push?", the answer is: add a `PreToolUse` hook that fires on `Bash(gh pr create*)` and checks for engineering-principles evidence, not a new instruction in `/push`. The command already runs it; the hook enforces it.

## Scope Assumptions

**GitHub-only.** The enforcement scripts (`enforce_pr_workflow.sh`, `merge.sh`, `push.sh`) use the `gh` CLI, and `scripts/lib/git-common.sh`'s `require::repo` helper validates the remote against `github.com`. GitLab, Bitbucket, and Gitea remotes are unsupported — this is a deliberate scope decision, not an oversight. Surfaced user-facing in README as of PR #43 (df4b372). (verified: git-common.sh `require::repo`, PR #43)

**`enforce_pr_workflow` is opt-in.** The hook no-ops when no `workflow.config.yaml` exists. Without config, `gh pr merge` goes through unguarded. `merge.sh` now surfaces this gap with an informational notice before merge (added PR #43). Run `/coderails:init` to generate the config and activate enforcement. (verified: enforce_pr_workflow.sh NO_CONFIG guard, scripts/merge.sh PR #43 addition)

## Prose-level enforcement inside a skill: the Phase 4b self-attestation case (PR #86)

Not every enforcement gap is hook-shaped. `skills/agentic-loop/SKILL.md`'s Phase 4b let the
orchestrator unilaterally demote an independent reviewer's clean-break compat `MERGE-BLOCKER` to a
logged note, by writing free-text "reviewed, not compat — `<reason>`". This is the same self-grading
problem the Law addresses (a check performed by the party with motive to pass it is not a check),
but the gate lived in skill prose, not a hook — so no `PreToolUse`/`Stop` mechanism could catch it;
the only lever available was rewriting the prose contract itself.

PR #86 removed the orchestrator's self-demote power entirely: the only two moves left are (a) fix
the finding, or (b) hard-stop to a human, logged who/when/SHA/reason. The one exception — a
fully-unattended envelope that cannot tolerate ever hard-stopping here — must be granted **at Phase 0
envelope-authorisation time**, quoted verbatim, never something the orchestrator grants itself
mid-run. See [[agentic-loop]] "Phase 4b self-attestation loophole closed" and
[[pr_86_agentic-loop-hardening]].

**Why this belongs in the enforcement-model lens even without a hook change:** the same design law
(hooks vs. advisory, self-grading vs. independent-grading) applies whether the mechanism is a
`PreToolUse` script or a skill's own control-flow contract. A skill can encode a "may not self-grade"
rule in prose; it just can't *mechanically enforce* it the way a hook can — Claude still has to
choose to follow the Phase 4b contract. That's an advisory ceiling, same category as the
`enforce_pr_workflow` "evidence not completion" ceiling documented above.

## The `model: sonnet` advisory ceiling (PR #86)

`agentic-loop` asserts `model: sonnet` for spawned workers roughly 6 times, but no hook gates
`Agent`/`Task` spawn calls on requested model — `hooks/hooks.json` and `hooks/scripts/*.sh` only
match `Bash` and `Write`/`Edit`/`MultiEdit` events, nothing for agent spawns. PR #86 documented this
explicitly in `AGENTS.md`'s "Enforcement ceilings" list as a **deliberate** choice, not a gap:
the rule is cost control, not correctness (an opus worker still produces a valid, fully-gated PR),
and a blunt model-gate hook couldn't distinguish Phase 2.5's sanctioned opus-escalation exception
from a disallowed spawn without trusting a self-reported flag — reintroducing the same
trust-the-agent problem the hook would exist to remove, one level down. See [[agentic-loop]] and
[[pr_86_agentic-loop-hardening]].

## Cross-References

- [[discipline-loop]] — the specific Stop hooks that enforce self-checking discipline
- [[install-and-cache-trap]] — editing hooks in the repo does not update the running cache without reinstall
- [[no_edit_on_main]] — PreToolUse hook that blocks code edits on main/master (added 2026-06-25)
- [[enforce_pr_workflow]] — PreToolUse hook that gates `gh pr create`/`gh pr merge` (added 2026-06-25)
- [[unregistered_loop_guard]] — Stop hook that nudges on the unregistered-loop shape (added PR #17)
- [[inject_bootstrap]] — SessionStart hook that bootstraps coderails context (added 2026-06-25)
- [[merge]] — enforcement-gap notice added to merge.sh in PR #43
- [[agentic-loop]] — Phase 4b self-attestation removal and the `model: sonnet` advisory ceiling (PR #86)
- [[pr_86_agentic-loop-hardening]] — source record for both PR #86 decisions above
