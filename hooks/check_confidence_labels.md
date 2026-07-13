---
title: "Hook: check_confidence_labels"
type: hook
created: 2026-05-30
last_updated: 2026-07-13
sources:
  - hooks/scripts/check_confidence_labels.sh
  - sources/pr_57-62_subagent-enforcement-gate-hardening.md
  - sources/pr_76_harden-hook-stdin-read.md
  - sources/pr_159_retire-catchup-add-telemetry.md
  - sources/pr_155-158_ceremony_noise_envelope_anchoring.md
tags: [hook, stop-hook, subagentstop-hook, discipline, confidence-labels, enforcement, loop-demotion]
---

# Hook: check_confidence_labels

A `Stop` and `SubagentStop` lifecycle hook that blocks Claude Code (exit 2) when a substantive response carries no `(verified)`, `(inferred)`, or `(guess)` confidence label. Wired to both events as of PR #57. As of PR #155 (2026-07-13, a concurrent session's work — see [[discipline-loop]] for the fuller writeup), a `Stop`-event block demotes to a model-visible `additionalContext` warn when an [[agentic-loop]] session is active and incomplete; `SubagentStop` always blocks regardless of loop state.

Source: `hooks/scripts/check_confidence_labels.sh`

## What it enforces

Any response of 200 characters or more must contain at least one confidence label. A response that makes claims without labelling them trips this hook. (verified: check_confidence_labels.sh:9, 44, 46)

## Promotion history

The hook comment on line 3 reads: `BLOCK-MODE: exits 2 when confidence labels are missing (promoted from warn-mode 2026-05-05)`. (verified: check_confidence_labels.sh:3)

Before that date it emitted a warning without blocking. After 2026-05-05 it exits 2, causing the model to be shown the error and prompted to revise.

## Logic

The script detects `hook_event_name` and takes a different path depending on the event:

**SubagentStop path (PR #57):**
1. Read `.last_assistant_message` directly from the payload — the subagent's actual final output.
2. Proceed with MIN_LEN check and label grep (same as Stop path below).

Note: `transcript_path` on a SubagentStop payload is the **parent** session transcript, not the subagent's. Reading it would check the wrong content; this was the design motivation for the separate path. (verified: check_confidence_labels.sh)

**Stop path:**
1. Read transcript path from the hook payload. If absent or file missing, exit 0.
2. Extract the last assistant text block with retry-backoff (same pattern as [[check_verify_loop]]).

**Shared steps (both paths):**
3. If the text is shorter than `MIN_LEN` (default 200, overridable via `$CLAUDE_HOOK_MIN_LEN`), exit 0 — short replies are out of scope.
4. Check for any match of `\((verified|inferred|guess)` via `grep -qE`. If found, exit 0.
5. On `Stop` only, if an [[agentic-loop]] session is active and incomplete (`als_loop_active_incomplete`), emit an `additionalContext` warn instead of blocking, log `would_block=1 warned=1 blocked=0`, and exit 0 (PR #155). Evaluated lazily — only once a block is imminent — so non-loop sessions never pay the transcript-invocation scan.
6. Otherwise: log `blocked=1` and exit 2 with the message (rewritten PR #159, 2026-07-13, to match [[check_verify_loop]]'s more actionable style): `[discipline-block] response >=${MIN_LEN} chars with no confidence label. Rule (CLAUDE.md): tag each substantive claim (verified)/(inferred)/(guess) — e.g. "the cache matches the repo (verified — diffed both trees)". Add labels to the claims you made, then stop again.`

(verified: check_confidence_labels.sh)

## Configurable thresholds

| Env var | Default | Purpose |
|---|---|---|
| `CLAUDE_HOOK_MIN_LEN` | 200 | Minimum response length (chars) before labels are required |
| `CLAUDE_HOOK_TAIL_LINES` | 200 | Lines from transcript tail to parse |
| `CLAUDE_HOOK_MAX_ATTEMPTS` | 5 | Retry attempts for transcript-flush race |
| `CLAUDE_HOOK_SLEEP_S` | 0.3 | Sleep between attempts |

(verified: check_confidence_labels.sh:6–9)

## Logging

Logs a structured key=value line to `$CLAUDE_DISCIPLINE_LOG` (default `~/.claude/discipline.log`) on every run, including `text_len`, `attempts`, `matched`, and `would_block`. A second line is written on actual block with `blocked=1`, or on loop-demoted warn with `would_block=1 warned=1 blocked=0`. As of PR #159 (2026-07-13), every logged line also carries `event=<hook_event_name>` (`Stop` or `SubagentStop`), enabling main-agent-vs-subagent segmentation of the log that was previously impossible to reconstruct after the fact. (verified: check_confidence_labels.sh:51, 53, 74, 76, 83, 85)

## Stdin read convention (PR #76)

This hook reads its payload via `IFS= read -r -d '' -t 5 input || true` instead of the old `input=$(cat)`. The 5-second timeout is an in-process backstop: if the parent process dies without closing stdin (orphaning the hook), the read times out, `input` is empty, jq yields empty, and the hook exits 0 (allow). This fail-open is deliberate — a dead parent means no live tool call to gate. See [[pr_76_harden-hook-stdin-read]] for the full rationale and the 5 ≤ `hooks.json timeout` invariant.

## Related

- [[discipline-loop]]
- [[enforcement-model]]
- [[check_verify_loop]]
- [[pr_159_retire-catchup-add-telemetry]] — the `event=` telemetry + block-message rewrite PR
- [[agentic-loop]] — the loop-demotion predicate's consumer context
- [[pr_155-158_ceremony_noise_envelope_anchoring]] — PR #155's full mechanism writeup (predicate truth table, lazy evaluation, fail-toward-blocking `jq` emission) plus the retro-mining guidance: track the `blocked=1` line count over time, not the `would_block=1` flagged count, which stays high by design
