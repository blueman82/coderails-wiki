---
title: "PR 99 — unregistered_loop_guard nudge-once-per-session"
type: source
created: 2026-07-08
last_updated: 2026-07-08
sources: []
tags: [source, unregistered_loop_guard, discipline-loop, stop-hook, nudge, self-perpetuating-loop, grep-metachar]
---

# PR #99 — unregistered_loop_guard nudge-once-per-session

<!-- Ingested by /wiki-ingest after merge. This is an immutable record of what changed. -->

## PR metadata

| Field | Value |
|---|---|
| PR number | #99 |
| Branch | `fix/unregistered-loop-guard-nudge-once` |
| Merged | 2026-07-08 |
| Merge SHA | `95479de58397f64ed6a2d29bd15579a918e303e` |
| Base | main |
| Commits | `1ccefa7` (nudge-once fix) + `88af636` (grep-metachar escape) |
| JIRA ticket | — |

## Summary

[[unregistered_loop_guard]] previously re-emitted its "3+ Agent dispatches, no agentic-loop registration" nudge on **every** Stop for a non-loop session, with no per-session termination. This made the nudge self-perpetuating for a session that genuinely was a one-off sequence of independent dispatches: nudge fires → the honest response is "no action needed" → that response ends in a Stop → the same conditions still hold → nudge fires again, forever. Observed live 2026-07-08.

The fix: before emitting, grep the discipline log (`als_log`'s existing ledger — the same file the emit path already writes `nudged=1` lines to) for a prior `nudged=1` line for **this** `session_id`. If found, exit 0 silently with `reason=already_nudged_this_session` and do not re-emit. The first nudge for a session is unaffected — this only suppresses repeats. No new state file was added; the existing ledger was reused as the source of truth.

**Commit 2 (`88af636`) hardens the suppression check itself:** `session_id` is interpolated into the grep pattern as a Basic Regular Expression, so a session id containing a literal BRE metachar (`.`, `*`, `^`, `$`, `[`, `\`) could wildcard-match an unrelated session's log line — e.g. a session id `s.1` would match a logged line for `sX1` (the `.` matching any character), falsely suppressing `s.1`'s legitimate first nudge. `als_sanitise_session_id` only strips `/` and collapses `..` (path-traversal defense) — a single `.` survives untouched, so this was a real reachable input, not a theoretical one. Fixed by escaping the session id with `sed 's/[.[\*^$\\]/\\&/g'` before interpolating it into the grep pattern.

## Files changed

- `hooks/scripts/unregistered_loop_guard.sh` — added the pre-emit suppression check (grep the discipline log for `hook=unregistered_loop_guard .*session=$esc_sid .*nudged=1`), the BRE-escape of `session_id`, and a new `als_log` line (`nudged=0 reason=already_nudged_this_session`) recording each suppressed repeat.
- `hooks/scripts/tests/unregistered_loop_guard.test.sh` — new Task 4 test block (nudge-once-per-session suppression): +75 lines.

## The fix (verified against the merge diff)

```bash
esc_sid=$(printf '%s' "$session_id" | sed 's/[.[\*^$\\]/\\&/g')
if grep -q "hook=unregistered_loop_guard .*session=$esc_sid .*nudged=1" "$LOG_FILE" 2>/dev/null; then
  als_log "hook=unregistered_loop_guard session=$session_id dispatch_turns=$dispatch_turns nudged=0 reason=already_nudged_this_session"
  exit 0
fi

als_log "hook=unregistered_loop_guard session=$session_id dispatch_turns=$dispatch_turns nudged=1"
jq -n --arg ctx "..." '{"hookSpecificOutput":{"hookEventName":"Stop","additionalContext":$ctx}}'
```

Match is `session=$esc_sid` **space-bounded** on both the field-name prefix and the trailing space before `nudged=1`, so a prior nudge for session `S-A` cannot substring-match and suppress session `S-AB`'s first nudge (tested in both directions — see Tests below). Missing/unreadable log file (the first-ever nudge, or any environment where the log can't be read) makes the grep find nothing, so the hook falls through and emits — the fail-open behaviour is unchanged from before this PR.

## Tests added (Task 4 block, `unregistered_loop_guard.test.sh`)

- **Repeat session, same conditions:** first Stop nudges (exit 0, non-empty stdout); second Stop for the same session is silently suppressed (empty stdout) and logs `reason=already_nudged_this_session`.
- **Fresh session still nudges exactly once:** proves the suppression branch doesn't regress the legitimate first-nudge path — exactly one `nudged=1` log line for a fresh session id.
- **Cross-session independence, both directions:** session `S-A` nudging first does not suppress `S-AB`'s first nudge, and (reverse direction) `S-AB` nudging first does not suppress `S-A`'s first nudge — proves the session-id match is exact, not a substring match, regardless of which session fires first.
- **BRE-metachar regression (the reason for commit 2):** session `sX1` nudges first; a later session with id `s.1` (containing a literal dot) must NOT be falsely suppressed by `sX1`'s log line. This test is written to fail against an unescaped grep (a BRE `.` wildcards to match `sX1`) and pass only once the session id is escaped before interpolation — a real regression test, not just an assertion of the fixed behaviour.

Reviewed via `pr-review-toolkit:review-pr` — `code-reviewer` returned 0 critical / 0 important / 1 suggestion (the grep-metachar hardening, applied in `88af636` even though refuted as unreachable for real UUID session ids); `pr-test-analyzer` confirmed the regression test fails pre-fix and passes post-fix. Eval artifact: GO, tier 1. Full hook suite 37/37 passing.

## Wiki pages updated

- [[discipline-loop]] — `unregistered_loop_guard` description in the Stop hook composition section gains the nudge-once-per-session behaviour.
- [[unregistered_loop_guard]] — dedicated hook page updated: Design decisions section gains the nudge-once mechanism; Known limitations updated; new Test coverage count.

## Caveats / gotchas

- **Fails open, not closed.** If the discipline log is missing, unwritable, or unreadable for any reason, the suppression check finds nothing and the hook emits — a broken log never wrongly suppresses a legitimate nudge, it can only (at worst) fail to suppress a repeat. This mirrors the rest of the discipline-loop design's fail-open convention (see [[discipline-loop]]).
- **The ledger is reused, not duplicated.** No new per-session state file was introduced — the existing `als_log` ledger (already written to on every nudge) is read back as the suppression source of truth. This keeps the fix to the same evidence class the hook already trusted.
- **Session-scoped, not global.** The suppression is keyed on `session_id` — a session that legitimately restarts (new session id) gets its own first nudge; this PR does not add any global "nudge the user at most once ever" behaviour, only "at most once per session."
- (verified) The grep-metachar fix in `88af636` was reviewer-suggested and confirmed unreachable for real (UUID-shaped) session ids in current practice, but applied anyway as defense-in-depth since `als_sanitise_session_id` does not strip single dots.
