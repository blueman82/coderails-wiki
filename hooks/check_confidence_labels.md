---
title: "Hook: check_confidence_labels"
type: hook
created: 2026-05-30
last_updated: 2026-06-26
sources:
  - hooks/scripts/check_confidence_labels.sh
  - sources/pr_57-62_subagent-enforcement-gate-hardening.md
tags: [hook, stop-hook, subagentstop-hook, discipline, confidence-labels, enforcement]
---

# Hook: check_confidence_labels

A `Stop` lifecycle hook that blocks Claude Code (exit 2) when a substantive response carries no `(verified)`, `(inferred)`, or `(guess)` confidence label.

Source: `hooks/scripts/check_confidence_labels.sh`

## What it enforces

Any response of 200 characters or more must contain at least one confidence label. A response that makes claims without labelling them trips this hook. (verified: check_confidence_labels.sh:9, 44, 46)

## Promotion history

The hook comment on line 3 reads: `BLOCK-MODE: exits 2 when confidence labels are missing (promoted from warn-mode 2026-05-05)`. (verified: check_confidence_labels.sh:3)

Before that date it emitted a warning without blocking. After 2026-05-05 it exits 2, causing the model to be shown the error and prompted to revise.

## Logic

1. Read transcript path from the hook payload. If absent or file missing, exit 0.
2. Extract the last assistant text block with retry-backoff (same pattern as [[check_verify_loop]]).
3. If the text is shorter than `MIN_LEN` (default 200, overridable via `$CLAUDE_HOOK_MIN_LEN`), exit 0 — short replies are out of scope.
4. Check for any match of `\((verified|inferred|guess)` via `grep -qE`. If found, exit 0.
5. Otherwise: log `blocked=1` and exit 2 with the message: `[discipline-block] response made substantive claims without (verified)/(inferred)/(guess) labels. Add them before stopping.`

(verified: check_confidence_labels.sh:44–66)

## Configurable thresholds

| Env var | Default | Purpose |
|---|---|---|
| `CLAUDE_HOOK_MIN_LEN` | 200 | Minimum response length (chars) before labels are required |
| `CLAUDE_HOOK_TAIL_LINES` | 200 | Lines from transcript tail to parse |
| `CLAUDE_HOOK_MAX_ATTEMPTS` | 5 | Retry attempts for transcript-flush race |
| `CLAUDE_HOOK_SLEEP_S` | 0.3 | Sleep between attempts |

(verified: check_confidence_labels.sh:6–9)

## Logging

Logs a structured key=value line to `$CLAUDE_DISCIPLINE_LOG` (default `~/.claude/discipline.log`) on every run, including `text_len`, `attempts`, `matched`, and `would_block`. A second line is written on actual block with `blocked=1`. (verified: check_confidence_labels.sh:47–51, 60–64)

## Related

- [[discipline-loop]]
- [[enforcement-model]]
- [[check_verify_loop]]
