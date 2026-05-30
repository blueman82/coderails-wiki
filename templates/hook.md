---
title: "<Hook script name>"
type: hook
created: YYYY-MM-DD
last_updated: YYYY-MM-DD
sources: []
tags: []
---

# <script-name>.sh

<!-- One-sentence summary: what lifecycle event, what it checks, block or warn. -->

## Event and mode

| Field | Value |
|---|---|
| Event | `UserPromptSubmit` / `Stop` / `PreToolUse (Bash)` |
| Mode | block (exit 2) / warn / silent |
| Timeout | N seconds (hooks.json) |

## Logic summary

<!-- What the script checks. Enumerate the skip gates in order (cheap first). -->
<!-- Cite hooks/scripts/<name>.sh:line where the key decisions live. -->

## Block condition

<!-- Exact condition that causes exit 2 or permissionDecision: deny. -->

## Log output

<!-- What it appends to $CLAUDE_DISCIPLINE_LOG. -->

## Environment variables

<!-- CLAUDE_DISCIPLINE_LOG, CLAUDE_HOOK_TAIL_LINES, etc. -->

## See also

<!-- [[discipline-loop]], [[enforcement-model]], or related hook pages. -->
