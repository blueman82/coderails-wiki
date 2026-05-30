---
title: "Hook: check_verify_loop"
type: component
created: 2026-05-30
last_updated: 2026-05-30
sources:
  - hooks/scripts/check_verify_loop.sh
tags: [hook, stop-hook, discipline, enforcement, did-not-verify]
---

# Hook: check_verify_loop

A `Stop` lifecycle hook that blocks Claude Code (exit 2) when the last response leaves a source-resolvable claim unresolved in a `## Did Not Verify` section.

Source: `hooks/scripts/check_verify_loop.sh`

## What it enforces

A `## Did Not Verify` bullet that names a file or file:line token — for example `prep.md:96` or `hooks/scripts/x.sh` — identifies something that could have been confirmed by reading the working tree. Leaving it unresolved is the failure this hook catches. (verified: check_verify_loop.sh:2–8)

Bullets about runtime behaviour, external systems, or user intent name no such token. Those pass as genuinely unverifiable.

## Ordered checks

Checks run top-to-bottom. The first that matches decides. All but the last allow the stop; only an unresolved source token blocks. (verified: check_verify_loop.sh:10–16)

| Check | Condition | Outcome |
|---|---|---|
| no transcript | No transcript path in payload, or file does not exist | allow stop |
| conversation only | No files edited this turn (Write/Edit/MultiEdit targets) | allow stop |
| loop-guard | `stop_hook_active == true` (already blocked once this turn) | allow stop |
| no text | Last assistant response has no text | allow stop |
| no DNV | No `## Did Not Verify` (or `## Not Verified`) bullets in the response | allow stop |
| **block** | Any DNV bullet matches the source-token regex | **exit 2** |

File-edit threshold: the hook skips when `file_count -lt 1`, i.e. zero files edited (verified: check_verify_loop.sh:47). A single edited file brings the response in scope. This was lowered from `< 3` so single-file sessions are no longer unguarded (see [[install-cache-trap_2026-05-30]]).

## Source-token detection

The block trigger regex (verified: check_verify_loop.sh:117):

```
[A-Za-z0-9_./-]+\.(md|sh|json|py|ts|tsx|js|jsx|go|yaml|yml|txt)([:space:]|$)
| [A-Za-z0-9_./-]+:[0-9]+
```

A bullet hits this if it contains a path with a known extension, or a `file:line` reference.

## Transcript-flush race mitigation

The hook retries `extract_last_text` with configurable backoff (`MAX_ATTEMPTS`, `SLEEP_S`) until the extracted length stabilises at a positive value. Each assistant entry is reduced to a joined string of its text blocks; a non-text entry contributes `""` and can never shadow a real text block. (verified: check_verify_loop.sh:63–88)

## Loop-guard

`stop_hook_active` from the hook payload is checked before any expensive transcript parsing. When the model re-runs after a block and trips this hook again, the loop-guard exits 0, allowing the second stop. (verified: check_verify_loop.sh:53–56)

## Logging

Appends a structured key=value line to `$CLAUDE_DISCIPLINE_LOG` (default `~/.claude/discipline.log`) at two points: once after the DNV-item count is known, and again on block, both including `session_id`, `text_len`, `attempts`, `files`, `dnv_items`, and `resolvable_dnv_items`. (verified: check_verify_loop.sh:104, 120, 123)

## History

The hook previously contained a check that asked whether any Read/Grep/Bash tool call appeared after the DNV section in the transcript. That check had a jq operator-precedence bug causing the branch to evaluate incorrectly. Both the weak "ran a tool" check and the buggy jq expression were deleted. The source-token regex at line 117 is now the sole trigger. (inferred: task brief and prior session knowledge — this code is absent from the current file and the repo carries no git history)

## Related

- [[enforcement-model]]
- [[discipline-loop]]
- [[install-and-cache-trap]]
