---
title: "Hook exit codes: which events block on exit 2"
type: design
created: 2026-05-31
last_updated: 2026-07-06
sources:
  - sources/session_2026-05-31_verify-loop-hardening.md
tags: [design, hooks, exit-codes, blocking, enforcement]
---

# Hook exit codes — what blocks, and where stderr goes

Establishes which Claude Code hook events treat `exit 2` as a block, and why
coderails uses two different block mechanisms. Source: official Claude Code hooks
docs (`code.claude.com/docs/en/hooks.md`), fetched 2026-05-31. (verified by fetch)

## The universal rule

- **Exit 0** — success. stdout JSON is parsed (JSON is only processed on exit 0).
- **Exit 2** — blocking error. stdout/JSON ignored; stderr is acted on per event.
- **Other non-zero** — non-blocking error; execution continues.

## Exit-2 behaviour for the events coderails touches

| Event | Blocks on exit 2? | Blocks what | stderr goes to |
|---|---|---|---|
| `PreToolUse` | **Yes** | the tool call | fed back to Claude |
| `UserPromptSubmit` | **Yes** | the prompt (and erases it) | **user only — not Claude** |
| `Stop` | **Yes** | Claude stopping; continues turn | fed back to Claude |
| `PostToolUse` | No | tool already ran | informational to Claude |
| `Notification` / `SessionStart` / `SessionEnd` | No | nothing | user only |
| `PreCompact` | **Yes** | context compaction | fed back to Claude |

(verified: docs fetched 2026-05-31)

The sharp consequence for coderails: on `UserPromptSubmit`, exit 2 would **erase
the user's prompt and show stderr only to the user, not to Claude**. That is why
`discipline_catchup.sh` (a UserPromptSubmit hook) uses the `additionalContext`
JSON form at exit 0 to nudge Claude, not exit 2.

## Why coderails uses two block mechanisms

| Mechanism | Used by | Form |
|---|---|---|
| `permissionDecision: "deny"` + exit 0 | `destructive_bash_gate.sh`, `test_gate.sh` (PreToolUse) | JSON on stdout |
| plain `exit 2` + stderr | `check_confidence_labels.sh`, `check_verify_loop.sh` (Stop) | exit code |

For **PreToolUse**, both exit 2 and the JSON `deny` block the tool call, but the
JSON form carries a structured `permissionDecisionReason` (and supports
`allow`/`ask`/`defer`). coderails uses the JSON form to deliver a useful reason
(e.g. "Destructive pattern detected: …") rather than a bare stderr string.

For **Stop**, the only JSON alternative is `decision: "block"`, which adds nothing
over exit 2, so coderails uses the simpler exit 2 + stderr.

Rule of thumb: pick one mechanism per hook — exit 2 *or* exit-0-with-JSON, never
both (JSON is ignored on exit 2). Use the JSON form only where its richer reason
field earns its keep.

## Related

- [[enforcement-model]] — the hooks-vs-commands distinction this refines
- [[discipline-loop]] — the Stop hooks that use exit 2
- [[check_verify_loop]] — a Stop hook using plain exit 2
