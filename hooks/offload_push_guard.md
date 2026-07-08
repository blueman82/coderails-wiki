---
title: "Hook: offload_push_guard.sh"
type: hook
created: 2026-07-08
last_updated: 2026-07-08
sources:
  - sources/pr_108_2026-07-08_offload-push-guard.md
tags: [hook, stop-hook, subagentstop, nudge, enforce-pr-workflow, discipline-log, offload]
---

# offload_push_guard.sh

`Stop`/`SubagentStop` hook that nudges — never blocks — when the final assistant
message tells the **user** to run a `git push` to `main`/`master` from their own
shell, instead of clearing [[enforce_pr_workflow]]'s push gate itself by running
`/pr-review-toolkit:review-pr` in this session first.

Source: `coderails/hooks/scripts/offload_push_guard.sh`
Shared libs: `hooks/scripts/lib/loop_state_common.sh` (`als_log`,
`als_sanitise_session_id`) and `hooks/scripts/lib/discipline_common.sh`
(`dc_stable_text`) — reuses existing helpers, adds no new lib functions.

## Why this hook exists

`enforce_pr_workflow.sh` blocks `git push` to `main`/`master` unless
`/pr-review-toolkit:review-pr` ran in **this session's** transcript. That block
is self-clearable — run review-pr in-session, then push — but an agent can
mis-handle it by telling the user "run this push yourself from your own
shell," offloading work the session could do itself (and a worker's review-pr
does not count; the gate scans the orchestrating session's own transcript). A
`PreToolUse` hook can't block a bad *sentence* — only a bad tool call — so this
is a Stop-time nudge instead: it fires only when the final text both names a
push to main/master **and** carries an offload-to-user cue near it.

## Event and mode

| Field | Value |
|---|---|
| Event | `Stop` + `SubagentStop` |
| Mode | **nudge only** (`hookSpecificOutput.additionalContext` on stdout, exit 0) — never blocks |
| Timeout | 5s floor (per the repo's stdin-read convention) |

## Trip condition

Both must match in the same final-message text, via `opg_is_offload_push()`:

1. **A push-to-main/master token** — `git( -C <path>)? push ...(origin )?(main|master)`,
   case-insensitive, allowing an optional leading `! ` run-it-yourself prefix
   and an optional `-C <path>` between `git` and `push`. Not anchored to line
   start, so it matches inside a longer sentence or a fenced command.
2. **An offload-to-user cue** — a leading `! ` prefix, or phrases asking the
   user to run the command themselves: "your own shell", "run (it|this)
   yourself", "from your shell", "you run", "needs your shell", "un-gated
   shell" (case-insensitive).

A plain "I pushed to main" (no offload cue) does not match. A
`/coderails:push` suggestion does not match (no push-to-main/master token — it
targets a feature branch). A feature-branch push does not match regardless of
offload phrasing.

## Text source by event

- **`SubagentStop`**: reads `last_assistant_message` directly from the hook
  payload.
- **`Stop`**: reads the transcript path from the payload and calls
  `dc_stable_text` (from `discipline_common.sh`) over the last
  `CLAUDE_HOOK_TAIL_LINES` (default 200) lines, the same retry-until-stable
  extraction `check_confidence_labels.sh` and `unregistered_loop_guard.sh` use.

## Nudge-once-per-session

Same ledger idiom as [[unregistered_loop_guard]]: before emitting, greps the
discipline log for a prior `hook=offload_push_guard .*session=<sid> .*nudged=1`
line for this session, BRE-escaping the session id first (it can contain
literal BRE metacharacters, e.g. `.`, that `als_sanitise_session_id` does not
strip). If found, exits 0 silently with `reason=already_nudged_this_session`.

## Skip gates (cheapest first)

| Order | Condition | Outcome |
|---|---|---|
| 1 | No transcript / no `last_assistant_message` | allow, silent (`reason=no_transcript`) |
| 2 | Final text empty | allow, silent (`reason=empty_text`) |
| 3 | No push-to-main/master token | allow, silent (`reason=no_match`) |
| 4 | No offload-to-user cue | allow, silent (`reason=no_match`) |
| 5 | Already nudged this session | allow, silent (`reason=already_nudged_this_session`) |
| — | Both tokens matched | **nudge** (`nudged=1`) |

## Nudge content

Delivered via `hookSpecificOutput.additionalContext` on stdout with `exit 0`
(model-visible per the hooks docs) — never stderr-on-exit-0, which this repo
reserves for block-precedent hooks' exit-2 messages. The nudge text names the
fix directly: run a genuine `/pr-review-toolkit:review-pr` in this session,
then `git push` here — don't offload to the user's shell or add a
`settings.json` bypass.

## Design decisions

- **Sibling to [[unregistered_loop_guard]], not an extension of it.** Same
  "nudge, never block" contract and the same nudge-once-per-session ledger
  idiom (grep the discipline log for a prior `nudged=1` line, BRE-escape the
  session id), but answers an unrelated question — this hook has nothing to do
  with agentic-loop registration.
- **YAGNI cuts, explicit in the source:** no blocking, no config flag, no
  classification of push types beyond the push+offload match, and no new lib
  functions — it reuses `als_log`/`als_sanitise_session_id` and `dc_stable_text`,
  the same helpers `check_confidence_labels.sh` and `unregistered_loop_guard.sh`
  already use.
- **Registered on both `Stop` and `SubagentStop`** in `hooks.json`, because the
  offload sentence can appear in either a top-level session's final message or
  a subagent's final message.

## Test coverage

`hooks/scripts/tests/offload_push_guard.test.sh` — 20/20 assertions: offload +
push-to-main nudges; `! git -C <path> push origin main` nudges; the agent
doing the push itself does not nudge; a `/coderails:push` suggestion does not
nudge (no push token); a feature-branch push does not nudge; an offload cue
with no push token does not nudge; a repeat `Stop` in the same session is
suppressed; `SubagentStop` correctly reads `last_assistant_message`; missing
transcript / empty text / distinct sessions all behave correctly. Full hook
suite (`run_all.sh`): 37/38 at merge time — the one failure
(`install_mode_sweep.test.sh`) is pre-existing/environment-dependent,
reproduced identically on baseline `origin/main` before this change.
`hooks_json_timeout_floor.test.sh`'s `EXPECTED_BACKSTOP_COUNT` bumped 13→14 for
the new hook.

## See also

- [[enforce_pr_workflow]] — the push gate this hook's nudge exists to help
  agents clear correctly, in-session, instead of offloading
- [[unregistered_loop_guard]] — the sibling hook this one borrows its
  nudge-never-block contract and ledger idiom from; unrelated subject matter
- [[discipline-loop]] — full Stop/SubagentStop hook composition
- [[pr_108_2026-07-08_offload-push-guard]] — PR #108 source record
