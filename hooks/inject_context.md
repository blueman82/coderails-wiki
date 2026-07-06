---
title: "Hook: inject_context"
type: hook
created: 2026-05-31
last_updated: 2026-06-29
sources:
  - hooks/scripts/inject_context.sh
  - sources/pr_76_harden-hook-stdin-read.md
  - sources/pr_79_sync-docs-drift.md
tags: [hook, userpromptsubmit-hook, context-injection, silent, discipline]
---

# Hook: inject_context

A `UserPromptSubmit` lifecycle hook that silently prepends a `[ctx]` freshness line (date, cwd, git branch) to every prompt — and on the first prompt of a session also injects the discipline reminder.

Source: `hooks/scripts/inject_context.sh`

## Event and mode

| Field | Value |
|---|---|
| Event | `UserPromptSubmit` |
| Mode | silent — `additionalContext` JSON, exit 0 |
| Timeout | 5 seconds (hooks.json) |

## Why it cannot use exit 2

On `UserPromptSubmit`, exit 2 **erases the user's prompt and delivers stderr only to the user, not to Claude**. There is no way to inject context into Claude's view using exit 2 on this event. The hook therefore emits `additionalContext` JSON at exit 0, which Claude sees prepended to the incoming prompt. See [[hook-exit-codes]] for the full exit-code table. (inferred from [[hook-exit-codes]], which cites Claude Code docs fetched 2026-05-31)

## Logic summary

1. Read `transcript_path` from the payload. (verified: inject_context.sh:7)
2. Build the `[ctx]` line: current date (`%Y-%m-%d`), `cwd=<pwd>`, `branch=<git branch --show-current>`. If `git branch` fails (not in a repo), the branch field is `none`. (verified: inject_context.sh:9)
3. If a transcript file exists, count prior assistant turns with `jq`. If that count is 0, append the discipline reminder to the `[ctx]` line. (verified: inject_context.sh:12–16)
4. Emit `additionalContext` JSON and exit 0. (verified: inject_context.sh:19)

The discipline reminder text injected on session start (verified: inject_context.sh:15):

> `[discipline] Label every non-trivial claim (verified)/(inferred)/(guess). After multi-file changes include ## Did Not Verify listing what was not checked.`

## The `[ctx]` line format

```
[ctx] 2026-05-31 | cwd=/Users/harrison/Documents/Github/myproject | branch=main
```

On the first prompt of a session, the discipline reminder follows, appended to the same string.

## Session-start detection

"First prompt" is defined as: a transcript file is present AND the count of `assistant`-type entries in it is exactly 0. (verified: inject_context.sh:13–14)

This is a deliberate conservative definition: if the transcript is absent or unreadable, the hook does not inject the discipline reminder (the `if` block is skipped). The `[ctx]` line is always injected regardless.

## No logging

This hook does not append to `$CLAUDE_DISCIPLINE_LOG`. (verified: inject_context.sh — no reference to `$CLAUDE_DISCIPLINE_LOG`)

## Environment variables

None. This hook reads no configurable env vars. (verified: inject_context.sh)

## Relationship to discipline_catchup

`inject_context.sh` re-injects the discipline rules once, at session start. [[discipline_catchup]] covers the ongoing case: if a later response misses the rules, it injects a catch-up nudge on the next prompt. The two hooks work in sequence — inject once up front, catch up when needed.

## Stdin read convention (PR #76)

This hook reads its payload via `IFS= read -r -d '' -t 5 input || true`. See [[pr_76_harden-hook-stdin-read]] for the full convention.

## Related

- [[hook-exit-codes]] — explains why UserPromptSubmit hooks must use additionalContext, not exit 2
- [[discipline-loop]] — how inject_context fits into the full discipline loop
- [[discipline_catchup]] — the companion hook that catches misses after session start
- [[sync-docs]] — the `/sync-docs` skill this page's PR #79 source record fixed doc-drift findings from
