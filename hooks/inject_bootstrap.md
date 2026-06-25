---
title: "inject_bootstrap.sh"
type: hook
created: 2026-06-25
last_updated: 2026-06-25
sources: [sources/pr_19-30_self-containment-and-hardening.md, sources/pr_40_hook-hardening.md]
tags: [hook, SessionStart, bootstrap, self-containment]
---

# inject_bootstrap.sh

SessionStart hook that fires once per session to inject the `coderails:using-coderails` skill context, ensuring coderails self-bootstraps without superpowers installed.

## Event and mode

| Field | Value |
|---|---|
| Event | `SessionStart` |
| Mode | silent — context injection, no block |
| Timeout | (default) |

## Logic summary

On session start, the script reads `skills/using-coderails/SKILL.md` from the plugin root, wraps it in an `<EXTREMELY_IMPORTANT>` block, and emits it as `additionalContext` via the Claude Code SessionStart JSON format. Plugin root is resolved from `CLAUDE_PLUGIN_ROOT` env var, falling back to two levels above the script's own directory (`hooks/scripts/ → plugin root`). (verified — PR #23)

All JSON escaping is handled by a single `jq -n --arg ctx "$session_context"` call. (verified — PR #40 clean-break refactor)

## Block condition

None. This hook does not block. It is purely additive context injection.

## JSON construction (PR #40 refactor)

Prior to PR #40, the script used a hand-rolled `escape_for_json()` bash function + `printf` to build the JSON output. This carried a double-escape risk and was silently wrong for control characters and non-ASCII. PR #40 replaced it entirely with `jq -n --arg ctx ...`, matching every other hook in the codebase. The `escape_for_json()` function no longer exists in this script. (verified — PR #40)

## Log output

None (silent mode; no `$CLAUDE_DISCIPLINE_LOG` entry).

## Why it exists

Before self-containment, coderails depended on superpowers installing a SessionStart hook that injected workflow orientation. Once superpowers is uninstalled, sessions would start cold. This hook closes that gap. (verified — PR #23)

## See also

[[enforcement-model]] — hooks vs. commands distinction  
[[self-containment]] — the broader vendor initiative this hook is part of  
[[inject_context]] — the other context-injection hook (UserPromptSubmit, not SessionStart)  
`coderails/hooks/scripts/inject_bootstrap.sh`  
`coderails/skills/using-coderails/SKILL.md`
