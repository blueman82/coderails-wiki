---
title: "inject_bootstrap.sh"
type: hook
created: 2026-06-25
last_updated: 2026-06-25
sources: [sources/pr_19-30_self-containment-and-hardening.md]
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

On session start, the script runs the `coderails:using-coderails` skill to orient the session with coderails-specific workflow guidance. This replaces the equivalent superpowers bootstrap that the superpowers plugin previously provided. (verified — PR #23)

## Block condition

None. This hook does not block. It is purely additive context injection.

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
