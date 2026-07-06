---
title: "Hook: test_gate"
type: hook
created: 2026-05-31
last_updated: 2026-07-06
sources:
  - hooks/scripts/test_gate.sh
  - sources/pr_76_harden-hook-stdin-read.md
  - sources/pr_79_sync-docs-drift.md
tags: [hook, pretooluse-hook, enforcement, test-gate, opt-in, block]
---

# Hook: test_gate

A `PreToolUse (Bash)` hook that blocks `git commit` when the project's test command fails — but only if the project has opted in by creating `.claude/test_command`.

Source: `hooks/scripts/test_gate.sh`

## Event and mode

| Field | Value |
|---|---|
| Event | `PreToolUse (Bash)` |
| Mode | **block** (`permissionDecision: "deny"`, exit 0) — opt-in only |
| Timeout | 120 seconds (hooks.json) — runs the full test suite |

## Opt-in mechanism

The hook is a **no-op unless the project contains `.claude/test_command`**. (verified: test_gate.sh:12–14)

To enable it in a project, create `.claude/test_command` with the test command on its first line — for example:

```
npm test
```

or

```
pytest tests/
```

The hook reads only the first line (`head -1`). If that line is empty, the hook exits 0 without running anything. (verified: test_gate.sh:16–18)

This opt-in design means the gate is globally registered (in `hooks.json`) but effectively scoped to projects that declare a test command. Projects without `.claude/test_command` are unaffected.

## Logic summary

1. Extract `tool_input.command` from the payload. If absent or empty, exit 0. (verified: test_gate.sh:8–9)
2. Check whether the command contains `git commit`. If not, exit 0 — only commits are gated. (verified: test_gate.sh:8)
3. Check for `.claude/test_command`. If not present, exit 0. (verified: test_gate.sh:12–13)
4. Read the first line of `.claude/test_command`. If empty, exit 0. (verified: test_gate.sh:16–18)
5. Run the test command via `eval`. Capture stdout+stderr to `/tmp/claude_test_gate.log`. (verified: test_gate.sh:21)
6. If the test command exits non-zero, deny the commit and return the last 20 lines of output (up to 1500 chars) in the reason. (verified: test_gate.sh:22–30)
7. If tests pass, exit 0 — the commit proceeds. (verified: test_gate.sh:32–33)

## Block condition

`git commit` is attempted while `.claude/test_command` exists and its command exits non-zero. The deny JSON (verified: test_gate.sh:23–29):

```json
{
  "hookSpecificOutput": {
    "hookEventName": "PreToolUse",
    "permissionDecision": "deny",
    "permissionDecisionReason": "Test gate failed. Project test_command: <cmd>\n\nLast 20 lines of output:\n<log tail>"
  }
}
```

The log tail is truncated at 1500 characters to keep the reason readable in the Claude UI.

**Why JSON deny and not exit 2:** Same rationale as [[destructive_bash_gate]] — the JSON form delivers the test output via `permissionDecisionReason`. See [[hook-exit-codes]].

## Setup command

[[test-gate-setup]] (`/coderails:test-gate-setup`) is the companion slash command that writes `.claude/test_command` interactively, so users do not have to create the file manually. (inferred: by convention with the command name `test-gate-setup` visible in the plugin)

## No logging

This hook does not append to `$CLAUDE_DISCIPLINE_LOG`. (verified: test_gate.sh — no reference to `$CLAUDE_DISCIPLINE_LOG` or `discipline.log`)

## Environment variables

None configurable. The hook uses `CLAUDE_DISCIPLINE_LOG` only implicitly (it doesn't log). The 120-second timeout set in `hooks.json` is the practical bound for test suites.

## Stdin read convention (PR #76)

This hook reads its payload via `IFS= read -r -d '' -t 5 input || true`. One of the two scripts used in the guard test in `stdin_bounded_read.test.sh` (the test asserts it exits within 8 seconds against a never-EOF pipe). See [[pr_76_harden-hook-stdin-read]].

## Gate-pattern convention (PR #79)

`test_gate.sh` uses inline `if`-blocks (not named gate functions). It is one of the **five core gate scripts** that use this pattern: `check_verify_loop.sh`, `check_confidence_labels.sh`, `no_edit_on_main.sh`, `destructive_bash_gate.sh`, and `test_gate.sh`. This was confirmed and documented in `AGENTS.md` by PR #79 (fixing a doc-drift finding from `/sync-docs`). Support/context scripts (`inject_context.sh`, `inject_bootstrap.sh`, `discipline_catchup.sh`) also use inline blocks but are not part of the gate-pattern convention — they have no deny-path. New hook scripts should prefer named gate functions (like `enforce_pr_workflow.sh`). (verified: PR #79 body + AGENTS.md hook conventions)

## Related

- [[hook-exit-codes]] — why PreToolUse hooks use `permissionDecision: "deny"` + exit 0
- [[enforcement-model]] — hooks vs. commands; test_gate is the opt-in variant
- [[discipline-loop]] — full hook set
- [[sync-docs]] — the `/sync-docs` skill (PR #79) that surfaced the gate-pattern doc-drift finding this page documents
