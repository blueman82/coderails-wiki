---
title: "Hook: crack_on_gate"
type: hook
created: 2026-07-14
last_updated: 2026-07-14
sources:
  - hooks/scripts/crack_on_gate.sh
  - hooks/scripts/tests/crack_on_gate.test.sh
  - sources/pr_175-176_crack-on-gate-and-inbox-brief-button.md
tags: [hook, userpromptsubmit-hook, pretooluse-hook, askuserquestion, crack-on, discipline, session-scoped]
---

# Hook: crack_on_gate

A two-event discipline gate: while a "crack on" envelope is active in a session,
asking the human anything via `AskUserQuestion` is mechanically denied.

Source: `hooks/scripts/crack_on_gate.sh`

## Event and mode

| Field | Value |
|---|---|
| Events | `UserPromptSubmit` (stamp) + `PreToolUse` matching `AskUserQuestion` (deny) |
| Mode | silent stamp on `UserPromptSubmit`; **block** (`permissionDecision: deny`) on `PreToolUse` |
| Timeout | 5 seconds (hooks.json), both entries |

This is a single script registered at two separate hook events — the same
two-hooks-one-script pattern the loop-state guards use, but a new instance of it
outside that family. (verified: `hooks/hooks.json`)

## Logic summary

**`UserPromptSubmit` — stamp.** (verified: `crack_on_gate.sh:38-58`)

1. Read the payload's raw `.prompt` field — never the transcript, never injected
   context.
2. Match `(^|[^[:alnum:]])crack[[:space:]]+on([^[:alnum:]]|$)` case-insensitively:
   "crack" and "on" as whole words with any whitespace between them.
3. On a match, stamp `<base>/<session_id>/crack_on_active` (base =
   `$CLAUDE_AGENTIC_LOOP_DIR` or `~/.claude/agentic-loop`) with the current
   timestamp. Logs `stamped=1` only on a confirmed write; `stamped=0 err=write_failed`
   on a write failure — a failed stamp must never log as success.
4. No `session_id` in the payload → the gate stands aside entirely (exit 0,
   unstamped) — an unkeyable stamp could never be found again on the read side.
5. Always exits 0 on this event (matches the general rule that `UserPromptSubmit`
   hooks cannot use exit 2 without erasing the user's prompt — see
   [[hook-exit-codes]]).

**`PreToolUse` (matcher: `AskUserQuestion`) — deny.** (verified: `crack_on_gate.sh:61-79`)

1. If `tool_name != "AskUserQuestion"`, exit 0 immediately — this gate touches no
   other tool.
2. If this session's flag file exists, log `denied=1` and emit
   `permissionDecision: "deny"` with a reason telling the model to proceed
   autonomously or end the turn with a report.
3. No flag, or a different session's flag → exit 0 (allow).

## Why detection is raw-prompt-only, never transcript/context

This is the load-bearing design decision on this hook, and the reason it exists as
a standalone script rather than an extension of `inject_context.sh`. "Crack on" is
not a rare phrase in this repo's own sessions — it appears in the
[[agentic-loop]] skill body and in injected memory feedback files in essentially
every session. A hook that scanned the transcript or injected context for the
phrase would false-positive fleet-wide the first time any session so much as
*mentioned* crack-on mode, permanently suppressing `AskUserQuestion` for the rest
of that session regardless of what the human actually said. Reading only the raw
`.prompt` field of the current `UserPromptSubmit` payload guarantees the gate only
activates when a human literally typed the phrase, this turn. The test suite's
negative control (`crack_on_gate.test.sh`) encodes this directly: a transcript
laced with "crack on" mentions (skill-load message, memory feedback text) alongside
a benign raw prompt must NOT stamp the flag and must leave `AskUserQuestion`
allowed. (verified: `crack_on_gate.test.sh`, "NEGATIVE CONTROL" section)

## Why the flag is session_id-keyed, not routed through agentic_loop_path.sh

`lib/agentic_loop_path.sh` resolves its directory by **existence-probing**
`progress.json` (canonical path first, then a fallback probe) — the resolver used
by [[loop_state_guard]] and [[loop_stall_guard]]. That resolution can return a
*different* directory between the stamp write and the later read whenever the
underlying git-common-dir slug shifts mid-session (the documented slug-drift class
— see [[Case-insensitive retirement sweep|feedback]] and related loop-path
memories for other instances of this drift family). For those two guards, a missed
read fails toward the safe side (treating an active loop as unregistered still
surfaces a nudge or block). For this gate, a missed read would fail the **unsafe**
way: `AskUserQuestion` allowed despite an actually-active crack-on envelope,
silently defeating the gate's purpose. `crack_on_gate.sh` therefore keys its flag
path on `session_id` alone — `<base>/<session_id>/crack_on_active` — sharing only
the same base-directory env var (`$CLAUDE_AGENTIC_LOOP_DIR`) with the loop-path
family, not its resolution logic. The session id sanitisation (stripping `/`,
collapsing `..`) is a deliberately duplicated two-line transform, kept in lockstep
with the equivalent transform in `lib/agentic_loop_path.sh` and
`loop_state_common.sh` by comment convention rather than a shared function — this
script stays dependency-free like those do. (verified: `crack_on_gate.sh:16-24`,
header comment)

## Hard-stops are untouched by design

The deny is scoped to the `AskUserQuestion` tool matcher only. The four
agentic-loop hard-stops (see [[Never remove agentic-loop hard-stops|discipline-loop]]
invariant) are turn-ending `LOOP-STOP` "report and wait" *declarations* in the
final assistant message, not `AskUserQuestion` tool calls — this hook has no
mechanism (no Stop-event deny, no global flag) that could reach them, and none was
added. The test suite asserts this directly: with the crack-on flag live for a
session, `Bash`/`Write`/`Task` tool calls all still allow. (verified:
`crack_on_gate.test.sh`, "HARD-STOP PRESERVED" section)

## Environment variables

- `CLAUDE_AGENTIC_LOOP_DIR` — base directory for the flag file (default
  `~/.claude/agentic-loop`). Shared env var name with the loop-path family, but
  this script does not source or call into that family's resolver.
- `CLAUDE_DISCIPLINE_LOG` — same shared discipline log as every other hook (default
  `~/.claude/discipline.log`).

## Stdin read convention

Reads its payload via `IFS= read -r -d '' -t 5 input || true`, the same convention
documented in [[discipline-loop]]'s "Stdin read convention" section.

## Related

- [[discipline-loop]] — full hook composition; this is the newest `UserPromptSubmit`
  + `PreToolUse` addition
- [[agentic-loop]] — the skill whose body is the reason detection cannot scan the
  transcript
- [[loop_state_guard]] / [[loop_stall_guard]] — the sibling family that DOES use
  `agentic_loop_path.sh`'s resolver, and why this hook deliberately does not
- [[hook-exit-codes]] — why `UserPromptSubmit` hooks never use exit 2
- [[sources/pr_175-176_crack-on-gate-and-inbox-brief-button]] — the merge record
