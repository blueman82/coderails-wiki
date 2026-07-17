---
title: "Hook: crack_on_prose_gate"
type: hook
created: 2026-07-17
last_updated: 2026-07-17
sources:
  - hooks/scripts/crack_on_prose_gate.sh
  - hooks/scripts/tests/crack_on_prose_gate.test.sh
  - sources/pr_238_crack-on-prose-gate.md
tags: [hook, stop-hook, crack-on, discipline, session-scoped, askuserquestion]
---

# Hook: crack_on_prose_gate

A `Stop` discipline hook. While a "crack on" envelope is active in a session, it
blocks a final assistant message that hands a **question back to the user in
prose** — closing the evasion where the model asks in plain text instead of
calling the (already-denied) `AskUserQuestion` tool.

It is the prose half of the crack-on human-ask waiver. [[crack_on_gate]] denies
the `AskUserQuestion` **tool**; this hook covers the case that gate cannot see —
a question written as ordinary text in the final message, which produces no tool
call and no `PreToolUse` event.

Source: `hooks/scripts/crack_on_prose_gate.sh` (shipped in [[pr_238_crack-on-prose-gate]])

## Event and mode

| Field | Value |
|---|---|
| Event | `Stop` (matcher: none — all Stop events) |
| Mode | **block** (exit 2 + stderr reason) when a prose question is detected and the crack-on flag is stamped |
| Timeout | 15 seconds (hooks.json), registered after [[check_verify_loop]] in the Stop array |

Sibling to [[crack_on_gate]], not an extension of it — that hook stays two-event
and dependency-free; this one is a discipline Stop hook in the
[[check_confidence_labels]] / [[check_verify_loop]] family (sources
`lib/discipline_common.sh`, blocks via exit 2 + stderr). (verified:
`crack_on_prose_gate.sh:1-9`, header)

## Activation scope

Active **only** when this session's `crack_on_active` flag is stamped — the same
session-only flag [[crack_on_gate]] writes, resolved with the identical
`<base>/<session_id>/crack_on_active` keying (never the `progress.json` resolver,
which drifts — see [[crack_on_gate]]'s "Why the flag is session_id-keyed" section
for the full rationale). No flag → the hook exits 0 and the turn ends normally.
(verified: `crack_on_prose_gate.sh:111-121`)

## Classifier — deterministic two-tier heuristic, NOT an LLM judge

A judge was considered and rejected for a Stop hook: seconds of latency on every
in-envelope turn end, a network/API dependency inside the hook sandbox,
nondeterminism that can't be locked in a fixture test, and a new failure mode
(judge outage) that would have to fail one way or the other anyway. The
heuristic's known miss-cases are bounded and documented instead. (verified:
`crack_on_prose_gate.sh:11-16`)

**Preprocess.** Drop fenced code blocks, inline backtick spans, and blockquote
lines before classifying — quoted material is not the model asking. (verified:
`crack_on_prose_gate.sh:150-156`)

1. **Tier 1 (positional).** The last content line of the prose body (the text
   before a trailing `## Did Not Verify` section) or of the whole message ends
   with `?`, once trailing quotes/brackets/markdown are stripped. A question
   nothing follows is a question awaiting an answer; the self-answered rhetorical
   form ("Should I use X? No — because …") carries its answer after the `?` and
   does not match. (verified: `crack_on_prose_gate.sh:176-181`)
2. **Tier 1b (positional).** A whole-line first-person-modal question ("Should I
   …?", "Shall we …?") in the body's last 3 content lines — catches the ask when
   a structural trailer (DNV section, `LOOP-STOP` line) follows it. (verified:
   `crack_on_prose_gate.sh:183-192`)
3. **Tier 2 (phrase).** ~15 high-precision second-person request phrases anywhere
   in the prose ("do you want", "let me know which", "would you prefer", "awaiting
   your", "please confirm", "your call", …) — question mark or not, since these
   are addressed to the user by construction. (verified:
   `crack_on_prose_gate.sh:194-221`)

## Failure direction and loop termination

**Fail-closed on discipline.** Biased toward blocking, like the sibling Stop
hooks — a false positive costs one forced rewrite of the turn's ending into a
declarative report; a false negative silently parks a crack-on envelope on a
question nobody will answer, the exact failure the envelope exists to prevent.
Infrastructure failures (no transcript, no session_id, unwritable counter) stand
aside / fail open with a log line. (verified: `crack_on_prose_gate.sh:34-40`)

**No infinite block loop.** A per-session counter file
(`<flag dir>/prose_question_blocks`) is reset on the first Stop attempt of each
turn (`stop_hook_active=false`) and caps this hook's blocks at
`CLAUDE_CRACK_ON_PROSE_MAX_BLOCKS` (default 3) per turn; at the cap the stop is
allowed and logged `capped=1`. A model that rephrases the same question N times
has defeated the heuristic, and an unbounded loop would be strictly worse. Unlike
the sibling hooks, `stop_hook_active` is NOT an unconditional allow — a rephrased
question on the continuation turn must still be caught, so the counter, not the
flag, is the terminator. (verified: `crack_on_prose_gate.sh:42-53, 130-142, 229-233`)

## Scope: Stop-only, never SubagentStop

Registered on `Stop`, never `SubagentStop`. A subagent's final message addresses
the **orchestrator**, not the human — a worker asking its parent a question is not
a breach of the envelope (the parent answers it autonomously) — and the
`SubagentStop` payload carries the parent's session_id, so registering there would
spuriously police worker reports against the parent's flag. (verified:
`crack_on_prose_gate.sh:61-65`)

## Hard-stops are untouched by design

A well-formed stopping turn ends with its declarative `LOOP-STOP: <category> —
<reason>` line (the [[agentic-loop]] ending-line contract), which no tier matches.
This gate never prevents stopping-with-a-report — it prevents
stopping-with-a-question — and the per-turn cap guarantees any stop eventually
lands even if mis-worded. (verified: `crack_on_prose_gate.sh:55-59`)

## Honest ceiling — what it cannot catch

This is mechanical pattern-matching over prose, and intent has no regex. What
passes, audited but not blocked:

- a declarative handoff with no interrogative marker at all ("Two options exist: A
  and B." / "Blocked pending your decision on X." phrased novelly),
- novel second-person phrasings outside the tier-2 list,
- a question inside plain double quotes (only backtick/fence/blockquote quoting is
  stripped — a terminal quoted question is a known false-positive, a mid-message
  one a known miss),
- anything after the per-turn cap.

The guarantee is "the cheap, common ask-in-prose evasions are mechanically caught,
every catch and every cap is logged" — **not** "asking is impossible". This is the
same class of ceiling as [[destructive_bash_gate]]'s pre-expansion regex. (verified:
`crack_on_prose_gate.sh:67-79`, header)

## Environment variables

- `CLAUDE_CRACK_ON_PROSE_MAX_BLOCKS` — per-turn block cap (default 3).
- `CLAUDE_AGENTIC_LOOP_DIR` — base directory for the flag file, shared with
  [[crack_on_gate]] (default `~/.claude/agentic-loop`).
- `CLAUDE_DISCIPLINE_LOG` — the shared discipline log (default
  `~/.claude/discipline.log`).
- `CODERAILS_HEADLESS_RUN` — set to `1` in dashboard-spawned `claude -p` runs; the
  hook exits 0 (no interactive human to answer a repair turn). Same exemption shape
  as the sibling Stop hooks. (verified: `crack_on_prose_gate.sh:99-105`)

## Related

- [[crack_on_gate]] — the tool-deny half of the same crack-on human-ask waiver;
  this hook is the prose half
- [[discipline-loop]] — full hook composition; this is the newest `Stop` addition
- [[check_confidence_labels]] / [[check_verify_loop]] — the Stop-hook family this
  one belongs to (exit 2 + stderr, sources `discipline_common.sh`)
- [[agentic-loop]] — the skill whose `LOOP-STOP` ending contract this gate leaves
  untouched, and the source of the crack-on envelope concept
- [[destructive_bash_gate]] — the same "mechanical pattern-matching has a ceiling"
  honest-boundary class
- [[pr_238_crack-on-prose-gate]] — the merge record
