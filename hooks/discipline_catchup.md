---
title: "Hook: discipline_catchup"
type: hook
created: 2026-05-31
last_updated: 2026-06-29
sources:
  - hooks/scripts/discipline_catchup.sh
  - sources/pr_76_harden-hook-stdin-read.md
tags: [hook, userpromptsubmit-hook, discipline, warn, confidence-labels, did-not-verify]
---

# Hook: discipline_catchup

A `UserPromptSubmit` hook that inspects the last assistant response and injects a `[discipline-catchup]` nudge into the next turn if discipline requirements were missed. The only surviving warn-mode hook in coderails.

Source: `hooks/scripts/discipline_catchup.sh`

## Event and mode

| Field | Value |
|---|---|
| Event | `UserPromptSubmit` |
| Mode | warn — `additionalContext` JSON, exit 0 |
| Timeout | 10 seconds (hooks.json) |

## Why it is warn-mode and not block-mode

This hook fires on `UserPromptSubmit`, not `Stop`. On `UserPromptSubmit`, exit 2 would **erase the user's prompt and show stderr only to the user** — Claude would never see the discipline nudge. The hook therefore uses `additionalContext` at exit 0, which prepends the nudge to Claude's view of the incoming prompt. See [[hook-exit-codes]]. (inferred from [[hook-exit-codes]], which cites Claude Code docs fetched 2026-05-31)

The `Stop` hooks ([[check_confidence_labels]], [[check_verify_loop]]) handle the blocking case. `discipline_catchup` is belt-and-suspenders: by the time the user submits the next prompt, the transcript is fully flushed, making transcript parsing race-free. (inferred: hook comment at discipline_catchup.sh:3–5)

## Logic summary

1. Read `$CLAUDE_DISCIPLINE_LOG`, `$CLAUDE_HOOK_TAIL_LINES` (default 200), `$CLAUDE_HOOK_MIN_LEN` (default 200). (verified: discipline_catchup.sh:7–9)
2. Read `transcript_path` from payload. If absent or file missing, exit 0. (verified: discipline_catchup.sh:14–16)
3. Extract the last assistant text from the transcript tail using `jq`. (verified: discipline_catchup.sh:18–24)
4. If the last text is empty or shorter than `MIN_LEN`, exit 0 — short replies are out of scope. (verified: discipline_catchup.sh:26–27)
5. Count unique files edited this session (`Write`, `Edit`, `MultiEdit` tool calls). (verified: discipline_catchup.sh:30–37)
6. Check for confidence labels: if the response contains no `(verified)`, `(inferred)`, or `(guess)`, add to `missing[]`. (verified: discipline_catchup.sh:41–43)
7. Check for `## Did Not Verify` section: if `file_count >= 3` and the section is absent, add to `missing[]`. (verified: discipline_catchup.sh:44–46)
8. If `missing` is empty, exit 0. (verified: discipline_catchup.sh:48–50)
9. Log to `$CLAUDE_DISCIPLINE_LOG` and emit `additionalContext` JSON with a `[discipline-catchup]` message listing what was missed. (verified: discipline_catchup.sh:53–68)

## Two checks: labels and Did-Not-Verify

| Check | Condition | Threshold |
|---|---|---|
| Confidence labels | No `(verified/inferred/guess)` in response | Any response ≥ MIN_LEN |
| Did Not Verify section | No `## Did Not Verify` heading | Only when `file_count >= 3` |

The Did-Not-Verify check in this hook fires at **≥ 3 files**, while [[check_verify_loop]] (the Stop-hook that enforces DNV resolution) lowered its file-edit threshold to ≥ 1. This means a single-file session can trip `check_verify_loop` on a bad DNV bullet, but `discipline_catchup` will not nag for a missing DNV section — that asymmetry is intentional: requiring a DNV section after touching one file would be too noisy. (verified: discipline_catchup.sh:44 for the `>= 3` threshold; [[check_verify_loop]] page for the `>= 1` threshold)

## Log output

On every trigger, appends a structured key=value line to `$CLAUDE_DISCIPLINE_LOG` (default `~/.claude/discipline.log`):

```
<ISO-timestamp> hook=catchup session=<session_id> missing="<what was missed>"
```

(verified: discipline_catchup.sh:54–58)

## Environment variables

| Env var | Default | Purpose |
|---|---|---|
| `CLAUDE_DISCIPLINE_LOG` | `~/.claude/discipline.log` | Log destination |
| `CLAUDE_HOOK_TAIL_LINES` | 200 | Lines from transcript tail to parse |
| `CLAUDE_HOOK_MIN_LEN` | 200 | Min response length before labels are required |

(verified: discipline_catchup.sh:7–9)

## Only surviving warn-mode hook

As of 2026-05-05, `check_confidence_labels.sh` was promoted from warn to block. `discipline_catchup` is now the only hook in coderails that explicitly warns (via `additionalContext`) without blocking. It exists because UserPromptSubmit cannot block usefully — the warn channel is the only effective channel on this event. (inferred: the hook comment at discipline_catchup.sh:3; promotion history documented in [[check_confidence_labels]])

## Relationship to other discipline hooks

```
inject_context.sh      — injects discipline reminder on session start
discipline_catchup.sh  — catches misses on the next user prompt (this hook)
check_confidence_labels.sh — blocks at Stop if labels missing
check_verify_loop.sh   — blocks at Stop if DNV bullet names unresolved file token
```

See [[discipline-loop]] for the full design rationale.

## Stdin read convention (PR #76)

This hook reads its payload via `IFS= read -r -d '' -t 5 input || true`. See [[pr_76_harden-hook-stdin-read]] for the full convention.

## Related

- [[hook-exit-codes]] — why UserPromptSubmit hooks must use additionalContext, not exit 2
- [[discipline-loop]] — the full discipline hook composition
- [[check_confidence_labels]] — the Stop-hook that blocks on missing confidence labels
- [[inject_context]] — the session-start context injection hook
