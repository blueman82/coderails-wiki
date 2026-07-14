---
title: "Hook: inject_context"
type: hook
created: 2026-05-31
last_updated: 2026-07-14
sources:
  - hooks/scripts/inject_context.sh
  - sources/pr_76_harden-hook-stdin-read.md
  - sources/pr_79_sync-docs-drift.md
  - sources/pr_159_retire-catchup-add-telemetry.md
  - sources/pr_175-176_crack-on-gate-and-inbox-brief-button.md
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

## Relationship to discipline_catchup (retired PR #159, 2026-07-13)

`inject_context.sh` re-injects the discipline rules once, at session start. Until 2026-07-13, [[discipline_catchup]] covered the ongoing case: if a later response missed the rules, it injected a catch-up nudge on the next prompt, and the two hooks worked in sequence — inject once up front, catch up when needed. `discipline_catchup.sh` is now **retired** (file+test deleted, null-result justification: a flat 23-26% first-attempt miss rate was unchanged by the nudge since block-mode shipped 2026-05-05). `hooks.json`'s `UserPromptSubmit` array was briefly `inject_context.sh` only; as of PR #175 (2026-07-14) it gained a second entry, [[crack_on_gate]], which stamps a session-scoped "crack on" flag rather than injecting any context — the two hooks are independent (different job: freshness/discipline injection vs. envelope-state stamping) and neither reads the other's output. See [[pr_159_retire-catchup-add-telemetry]] and [[pr_175-176_crack-on-gate-and-inbox-brief-button|the crack-on gate's source record]].

## Stdin read convention (PR #76)

This hook reads its payload via `IFS= read -r -d '' -t 5 input || true`. See [[pr_76_harden-hook-stdin-read]] for the full convention.

## Related

- [[hook-exit-codes]] — explains why UserPromptSubmit hooks must use additionalContext, not exit 2
- [[discipline-loop]] — how inject_context fits into the full discipline loop
- [[discipline_catchup]] — the retired companion hook that used to catch misses after session start
- [[pr_159_retire-catchup-add-telemetry]] — the retirement PR
- [[sync-docs]] — the `/sync-docs` skill this page's PR #79 source record fixed doc-drift findings from
- [[crack_on_gate]] — the new (PR #175, 2026-07-14) `UserPromptSubmit` sibling hook, unrelated in purpose (envelope stamping, not context injection)
