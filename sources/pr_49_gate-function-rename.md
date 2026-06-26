---
title: "PR #49 — refactor/name gate functions"
type: source
origin: "PR #49 (merged 2026-06-26, squash 0426312)"
created: 2026-06-26
last_updated: 2026-06-26
sources: []
tags: [source, PR, hook, enforce_pr_workflow, loop_state_guard, loop_stall_guard, refactor, named-functions, loop-state-common]
---

# PR #49 — refactor/name gate functions

Replaces positional `# Gate N` early-exit comments in the three workflow/loop hook scripts with named bash functions. Extracts the byte-identical Gates 1–4 shared between the two loop guards into a new shared library `hooks/scripts/lib/loop_state_common.sh`. Also documents a known limitation in `enforce_pr_workflow`'s evidence model, deferred for a future fix.

## PR metadata

| Field | Value |
|---|---|
| PR number | #49 |
| Title | refactor/name gate functions |
| Merged | 2026-06-26 |
| Merge SHA | `0426312` |
| JIRA ticket | — |

## Summary

The hook scripts formerly used comments of the form `# Gate N` to label each early-exit check; the number conveyed position, not purpose, and would renumber on insertion. PR #49 replaces these with named bash functions, making each gate's purpose self-documenting, greppable, and stable against insertion. The refactor mirrors the naming idiom already used in `scripts/lib/git-common.sh` (`require::` / `pr::` prefixes). No behaviour changed — only readability and maintainability.

## Named functions introduced

### `enforce_pr_workflow.sh` (7 functions)

| Function | Former label | Purpose |
|---|---|---|
| `gate_has_command` | Gate 1 | Pass if command string is empty |
| `gate_safe_passthrough` | Gate 2 | Pass for `--help`, `--dry-run`, abort/continue/quit/skip ops |
| `gate_in_scope` | Gate 3 | Pass if command is not a gated subcommand; sets `$subcommand` |
| `gate_config_present` | Gate 4 | Pass if `workflow.config.yaml` absent (NO_CONFIG opt-in) |
| `gate_targets_main` | Gate 4b | Pass if `git merge`/`git push` does not target main/master — **the headline rename** |
| `gate_have_transcript` | Gate 5 | Pass if no transcript path in payload |
| `enforce_required_step` | Gate 6 | Scan transcript; pass if evidence found, deny if not |

`gate_targets_main` is the most meaningful rename: the former label "Gate 4b" communicated only position; the new name states the decision the gate makes.

### `loop_state_common.sh` (new shared library)

Gates 1–4 of `loop_state_guard.sh` (C1) and `loop_stall_guard.sh` (C2) were byte-identical. PR #49 extracts them once into `hooks/scripts/lib/loop_state_common.sh` as:

| Function | Purpose |
|---|---|
| `als_gate_no_transcript` | Gate 1: pass if no transcript |
| `als_gate_stop_hook_active` | Gate 2: pass if `stop_hook_active == true` (avoid stop-loop) |
| `als_gate_not_a_loop` | Gate 3: pass if no agentic-loop Skill `tool_use` in transcript |
| `als_load_progress` | Loads `progress.json`; used by Gate 4 (done-and-not-rearmed) in both guards |

Each guard (`loop_state_guard.sh`, `loop_stall_guard.sh`) sources `loop_state_common.sh` and then adds its own Gates 5–6 for the guard-specific enforcement logic.

## Files changed

- `hooks/scripts/enforce_pr_workflow.sh` — 7 local gate functions replacing positional comments
- `hooks/scripts/lib/loop_state_common.sh` — **new file**: `als_gate_*` / `als_load_progress` shared functions
- `hooks/scripts/loop_state_guard.sh` — sources shared lib; retains own Gates 5–6
- `hooks/scripts/loop_stall_guard.sh` — sources shared lib; retains own Gates 5–6
- `hooks/scripts/tests/enforce_pr_workflow.test.sh` — test comments updated to name functions
- `hooks/scripts/tests/loop_state_guard.test.sh` — test comments updated
- `hooks/scripts/tests/loop_stall_guard.test.sh` — test comments updated
- `CLAUDE.md` — "Hook script conventions" paragraph rewritten (see below)
- `.gitignore` — `.worktrees` added (machine-local git worktree dirs)

## CLAUDE.md hook conventions update

The "Hook script conventions" paragraph was previously written as a general rule covering all hook scripts, and contained a false claim that `check_verify_loop.sh` used numbered gates (it does not — it has a single linear flow). PR #49 rewrites the paragraph to:

- Scope the named-function pattern explicitly to the three scripts that use it (`enforce_pr_workflow.sh`, `loop_state_guard.sh`, `loop_stall_guard.sh`)
- Correct the false claim about `check_verify_loop.sh`
- Describe the pattern accurately: each exit point is a named function documenting the gate's purpose

## Known limitation — deferred

The `enforce_pr_workflow` evidence check looks for *invocation evidence* of `/coderails:push` or `/pr-review-toolkit:review-pr` in the transcript (a Skill tool call with the right name), not *completion evidence*. Two weaknesses were documented:

1. **Hollow invocation**: a Skill invocation that errors immediately or is short-circuited still satisfies the gate.
2. **Substring false-positive**: a `gh pr create` command that appears only in assistant prose (not a tool call) can trigger a false block (the transcript scan may match a discussion of the command).

The real "no unreviewed merge to main" guarantee is server-side GitHub branch protection. The local hook is a redirect + audit layer. The fix was deferred by the maintainer; no code changed for this in PR #49. (inferred — from PR #49 body and team-lead briefing)

## Wiki pages updated

- [[enforce_pr_workflow]] — named-function catalogue, `gate_targets_main` note, known-limitation section
- [[loop_state_guard]] — `als_gate_*` shared lib extraction, updated shared-lib reference
- [[loop_stall_guard]] — `als_gate_*` shared lib extraction, updated shared-lib reference

## See also

- [[discipline-loop]] — broader hook composition
- [[spec-plan-progress-artifact-chain]] — the two-hook loop-state guard architecture
