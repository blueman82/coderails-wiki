---
title: "PR #175-176 — crack-on gate + inbox-brief dashboard button"
type: source
created: 2026-07-14
last_updated: 2026-07-14
sources:
  - hooks/scripts/crack_on_gate.sh
  - hooks/scripts/tests/crack_on_gate.test.sh
  - hooks/hooks.json
  - AGENTS.md
  - README.md
  - examples/dashboard-config.json
tags: [source, hooks, crack-on, askuserquestion, dashboard, ask-button, rachel, inbox-brief]
---

# PR #175-176 — crack-on gate + inbox-brief dashboard button

Ingested as one cluster: two independently merged PRs shipped the same day, on the
"Inbox Brief" feature — a dashboard button that runs the secretary's inbox-brief
routine — plus the discipline gate that governs autonomous "crack on" sessions
generally. The two are related (the button's spawn path is itself an instance of
the profile/bypass constraints [[dashboard]] already documents) but ship
independent artifacts, hence one combined source page rather than two thin ones.

## PR metadata

| Field | Value |
|---|---|
| PR #175 | "rethink/wu3 crack on gate" |
| Merge SHA (#175) | `09cb0d0b16144be0123d4a11f93d33134bc3da0b` |
| Merged | 2026-07-14T22:32:53Z |
| PR #176 | "wu2 inbox brief button" |
| Merge SHA (#176) | `fc122d3a5bece6ddf67443a0449744d1865ef82d` |
| Merged | 2026-07-14T22:36:26Z |

## Summary

**PR #175 — `crack_on_gate.sh`.** A new two-hook discipline gate, documented in full
on [[crack_on_gate]]. `UserPromptSubmit` stamps a session-only `crack_on_active` flag
file when the **raw submitted prompt** (not the transcript, not injected context)
contains "crack on" (case-insensitive, word-boundary). `PreToolUse` matching
`AskUserQuestion` denies the tool call while that session's flag is stamped. The
detection is deliberately raw-prompt-only: "crack on" appears in the
[[agentic-loop]] skill body and in injected memory in essentially every session, so
a transcript/context scan would false-positive fleet-wide and permanently suppress
human interaction — only a human actually typing the phrase activates the gate. The
four agentic-loop hard-stops are untouched by design: they are turn-ending
`LOOP-STOP` "report and wait" declarations, not `AskUserQuestion` tool calls, so this
deny cannot reach them (see the [[agentic-loop]] hard-stops
invariant; memory `feedback_never_remove_hard_stops` — no gate in this repo is
permitted to remove or weaken the four hard-stops, and this one doesn't).

**PR #176 — inbox-brief dashboard button.** Adds one entry to
`examples/dashboard-config.json`'s `buttons` array: `name: "inbox-brief"`,
`profile: "bypass"`, `bypassPermissions: true`, `cwd: "/Users/harrison/Github/assistant-agent"`.
The button's `command` spawns `claude` (via the dashboard's existing
`bypass`-profile spawn path, [[dashboard]]) with a prompt instructing it to
run exactly one Bash command and report its output:
`bin/rachel "Read tasks/inbox-brief.md and follow it." < /dev/null`. This is the
established **profile-bypass-can-only-spawn-`claude`** boundary already
documented on [[dashboard]] — the button cannot invoke `bin/rachel` (assistant-agent's
secretary entrypoint) directly; the dashboard's `run` route hardcodes
`spawn('claude', ...)`, so reaching `rachel` requires a `claude` instance whose
prompt tells it to shell out via Bash. `< /dev/null` starves the spawned process of
stdin, matching the non-interactive/headless invocation pattern the dashboard's
other spawned-`claude` buttons already rely on ([[dashboard]]'s
`CODERAILS_HEADLESS_RUN` exemption is a sibling instance of the same
headless-non-interactive constraint class, though this button does not itself set
that env var — it is a plain `bypass`-profile button, not the `route.ts` run-API
call site that does).

## Files changed

- `hooks/scripts/crack_on_gate.sh` (new, 95 lines) — the two-event gate script
- `hooks/scripts/tests/crack_on_gate.test.sh` (new, 180 lines) — behavioural test,
  including the mandatory negative control proving detection never reads the
  transcript
- `hooks/hooks.json` — two new entries: `UserPromptSubmit` array gains
  `crack_on_gate.sh` as a second hook (alongside `inject_context.sh`); a new
  `PreToolUse` matcher block for `AskUserQuestion` runs `crack_on_gate.sh`
- `AGENTS.md`, `README.md` — hook event-map table rows for both new entries
- `hooks/scripts/tests/hooks_json_timeout_floor.test.sh` — extended for the new
  hook entries (18 lines changed, 10 deletions) — same timeout-floor invariant
  documented on [[discipline-loop]]
- `hooks/scripts/tests/exec_bit_invariant.test.sh` — 2-line extension to cover the
  new script's executable bit
- `examples/dashboard-config.json` — one new button entry (8 lines added)

## Wiki pages updated

- [[crack_on_gate]] — new hook page
- [[discipline-loop]] — hook-count and event-map updates
- [[dashboard]] — inbox-brief button entry added to the button/config catalog
- [[index]] — hooks table + skills table cross-references

## Caveats / gotchas

- **Session-only flag path is a deliberate divergence from the canonical
  `agentic_loop_path.sh` resolver.** The gate keys its flag at
  `<base>/<session_id>/crack_on_active` directly, NOT through
  `lib/agentic_loop_path.sh`'s existence-probe resolution. That resolver can return
  different directories between the stamp and the read under slug drift (e.g. a
  mid-session `git init` shifting the git-common-dir slug) — a miss there fails in
  the **unsafe** direction for this gate (`AskUserQuestion` allowed despite an
  active crack-on envelope). Keying on `session_id` alone removes that drift class
  by construction. This is a considered exception to the "shared lib" convention,
  not an oversight — [[loop_state_guard]] and [[loop_stall_guard]] correctly use the
  shared resolver because their failure direction (missing a registered loop) is
  the opposite of safe-by-default here.
- **Hyphenated "crack-on" deliberately does not match.** The word-boundary regex
  requires "crack" and "on" as separate whitespace-joined words. A hyphenated form
  reads as prose *about* the gate, not the human authorising it.
- **The inbox-brief button is the first `bypass`-profile button whose declared
  command shells out to a second, non-`claude` entrypoint** (`bin/rachel`) rather
  than doing the task inline as the spawned `claude` session's own work. Every
  other bypass/auto button in `examples/dashboard-config.json` has the spawned
  `claude` do its own reasoning; this one is a thin dispatcher. Same
  profile/spawn-boundary rules apply regardless (see [[dashboard]]).
