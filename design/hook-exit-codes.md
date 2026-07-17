---
title: "Hook exit codes: which events block on exit 2"
type: design
created: 2026-05-31
last_updated: 2026-07-17
sources:
  - sources/session_2026-05-31_verify-loop-hardening.md
  - sources/pr_159_retire-catchup-add-telemetry.md
  - sources/pr_204_cost-reporter.md
tags: [design, hooks, exit-codes, blocking, enforcement, system-message, human-visible-channel]
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
`inject_context.sh` (the sole `UserPromptSubmit` hook as of PR #159, 2026-07-13)
uses the `additionalContext` JSON form at exit 0 to inject context, not exit 2.
(Historically `discipline_catchup.sh`, retired PR #159, used the same pattern for
the same reason — see [[discipline_catchup]] and [[pr_159_retire-catchup-add-telemetry]].)

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

## `additionalContext` vs. `systemMessage` — model-visible vs. human-visible (PR #204)

`additionalContext` (used by every hook above that emits JSON at exit 0) is
**model-visible only** — Claude Code's own docs state it "doesn't appear as a
chat message in the interface." A `Stop` hook's raw stdout otherwise goes only
to the debug log, invisible to the human. Until PR #204, no hook in coderails
ever emitted a top-level `systemMessage` field — the channel Claude Code's own
schema documents as "Warning message shown to the user." `loop_stall_guard.sh`'s
new [[loop_stall_guard|cost reporter]] (`als_report_cost_on_complete`) is the
first hook to use it, verified empirically with a live smoke test: it renders
in the terminal as `Stop says: <msg>`, reaching the human directly rather than
only steering the model's next turn. See [[pr_204_cost-reporter]] for why a
mechanically-printed cost line needed a human-visible channel specifically —
the whole point was to make the report reach the human regardless of whether
the model chose to relay it.

| Channel | Reaches | Used by |
|---|---|---|
| `additionalContext` (exit 0 JSON) | model only | every nudge/warn hook in the table above |
| `systemMessage` (exit 0 JSON, top-level) | **human, directly** | `loop_stall_guard.sh`'s cost reporter (PR #204) — the sole user so far |
| stderr on `exit 2` | model (fed back into the blocked turn) | every block-mode hook in this file |

## Related

- [[enforcement-model]] — the hooks-vs-commands distinction this refines
- [[discipline-loop]] — the Stop hooks that use exit 2
- [[check_verify_loop]] — a Stop hook using plain exit 2
- [[loop_state_guard]] — a Stop hook using plain exit 2 (C1)
- [[loop_stall_guard]] — a Stop hook using plain exit 2 (C2)
- [[unregistered_loop_guard]] — a Stop hook that deliberately does NOT use exit 2, nudging via exit-0 `additionalContext` instead
- [[pr_204_cost-reporter]] — the source record for the new `systemMessage` channel and the reporter that uses it
