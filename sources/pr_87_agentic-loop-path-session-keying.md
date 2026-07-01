---
title: "PR 87 — agentic-loop progress.json path keyed on session_id, not just cwd"
type: source
created: 2026-07-01
last_updated: 2026-07-01
sources: []
tags: [agentic-loop, progress-json, session-id, race-condition, loop-state]
---

# PR 87 — agentic-loop progress.json path keyed on session_id, not just cwd

<!-- Ingested by /wiki-ingest after merge. This is an immutable record of what changed. -->

## PR metadata

| Field | Value |
|---|---|
| PR number | #87 |
| Branch | `fix/agentic-loop-path-collision` |
| Merged | 2026-07-01 |
| Merge SHA | `344a849` |
| JIRA ticket | — |

## Summary

Fixes the "single-loop-per-directory" limitation that [[pr_86_agentic-loop-hardening|PR #86]] had just documented as an accepted, unfixed race: two concurrent `agentic-loop` sessions in the same working directory previously raced for ownership of one `progress.json` (last-writer-wins), with git worktrees recommended as the workaround (verified: PR #86 source, `hooks/scripts/lib/agentic_loop_path.sh`'s pre-#87 header comment).

`agentic_loop_path.sh` now keys the path on **cwd AND session_id** (`<base>/<slug>/<session_id>/progress.json`, was `<base>/<slug>/progress.json`) instead of cwd alone (verified: `git show 5223f49 -- hooks/scripts/lib/agentic_loop_path.sh`). `session_id` comes from the Stop-hook JSON payload (guard scripts) or `$CLAUDE_CODE_SESSION_ID` (set in every Bash tool call — the orchestrator's own path resolution). This is a **verified platform guarantee, not something derivable from repo source alone**: Claude Code's `session_id` stays stable for the life of one continuous conversation, including across that conversation's own compaction/restart, while differing between genuinely separate conversations (stated in the PR body and the helper's header comment; correctly flagged by reviewers as "worth being aware of rather than a defect" since it can't be checked from source).

Effect: two concurrent sessions in one directory now get independent `progress.json` files automatically — no worktrees required for *this specific* collision. `loop_state_guard.sh`'s session-mismatch check is retained but its meaning narrows: it previously caught the routine cross-session race as its main job; now it only fires on a rarer case — a file's content disagreeing with its own session-scoped path (e.g. copied, hand-edited, or corrupted content) — a corruption signal rather than the routine race it used to catch (verified: PR body commit `5223f49`).

## The residual bug found during the same PR's review (Critical, fixed same-PR)

A follow-up PR-review pass (5 reviewer dispatches + native `/security-review`) caught a **Critical** bug in the initial fix itself: both the guard scripts' `jq` extraction (`.session_id // "?"` in `loop_state_guard.sh` / `loop_stall_guard.sh`) and `agentic_loop_path.sh`'s own `session_id="${2:-${CLAUDE_CODE_SESSION_ID:-unknown-session}}"` fallback used **fixed sentinel strings** (`"?"` and `"unknown-session"` respectively) whenever a real `session_id` was missing or null. Two different sessions both hitting that missing/null edge case would resolve to the **identical** fallback path — silently reintroducing the exact cross-session collision this PR existed to close, just gated behind a rarer trigger condition (a malformed or absent `session_id` field) instead of the routine case (verified: PR review comment, `IC_kwDOSuEVG88AAAABIa78Lw`, `head_sha=8cc9d63`).

The orchestrator independently reproduced this pre-fix (two malformed payloads → same `.../?/progress.json` path) before the reviewer's finding landed; `silent-failure-hunter` also flagged it independently.

**Fix (commit `ebfee9c` + `8cc9d63`):** a new `als_sanitise_session_id()` helper in `hooks/scripts/lib/loop_state_common.sh` generates a unique fallback (`unknown-$$-$(date +%s%N)` — PID + high-resolution timestamp) instead of any shared constant, applied consistently across all three call sites:
- `agentic_loop_path.sh`'s own default (fixed first, commit `ebfee9c`)
- `loop_state_guard.sh`'s `session_id` extraction (fixed second, commit `8cc9d63` — initially missed)
- `loop_stall_guard.sh`'s `session_id` extraction (fixed second, commit `8cc9d63` — initially missed)

The first commit (`ebfee9c`) fixed only the path helper's own default; the guard scripts' `jq` extraction still fell back to the shared `"?"` sentinel until the reviewer's Critical finding forced a second commit (`8cc9d63`) to wire the same sanitiser into both guards. A new discriminating regression test was added, verified red on the pre-fix code and green after.

This is the instructive part of the story: **a fix for a race condition itself shipped with a residual race-like edge case**, caught by independent review rather than the original author. The fix for a collision needs the *same* uniqueness discipline (no shared fallback constants) applied at *every* call site that can independently generate a fallback — missing even one reopens the exact bug class the PR was written to close.

## Files changed

- `hooks/scripts/lib/agentic_loop_path.sh` — path keyed on cwd + session_id; unique per-invocation fallback for missing session_id
- `hooks/scripts/lib/loop_state_common.sh` — new `als_sanitise_session_id()` helper
- `hooks/scripts/loop_state_guard.sh`, `hooks/scripts/loop_stall_guard.sh` — wired to use the sanitiser instead of raw `jq ... // "?"`
- `hooks/scripts/tests/agentic_loop_path.test.sh`, `loop_state_guard.test.sh`, `loop_stall_guard.test.sh` — regression coverage, including the discriminating "distinct sessions → distinct paths" test and the guard-level "`session_id: null` no longer resolves to the shared `?` path" test
- `skills/agentic-loop/SKILL.md` — "Context-window persistence" section (`progress.json` durable-artifact description) updated to state cwd+session_id keying; the "Single-loop-per-directory invariant" subsection rewritten as "Concurrent loops in one directory" describing the new safe-by-default behaviour
- `docs/REFERENCE.md` — two stale rows describing the old cwd-only path format corrected (one previously self-contradicted its own "Session-keyed" annotation)
- `commands/post-review.md`, `commands/prep.md` — minor doc touch-ups in the same PR (doc-bot commits)

## Wiki pages updated

- [[agentic-loop]] — "Single-loop-per-directory invariant" section replaced (not appended) to reflect that concurrent sessions in one directory are now safe by default; worktrees reframed as optional (still useful for working-tree isolation, no longer required for this collision)

## Caveats / gotchas

- The whole fix rests on a **platform guarantee that cannot be verified from repo source**: Claude Code's `session_id` stability across one conversation's compaction/restart, while differing across genuinely separate conversations. `(inferred, per PR review comment — a correct-per-reviewers assumption, not independently re-derivable here)`
- `loop_state_guard.sh`'s session-mismatch check is not removed — its *purpose* narrows from "catch the routine cross-session race" to "catch file/path corruption" now that the routine race no longer occurs by construction.
- Historical design docs (`docs/coderails/plans/`, `docs/coderails/specs/`) still describe the pre-fix cwd-only state — correctly left untouched per the review, since they are point-in-time records of past decisions, not live documentation.
- Supersedes the "Single-loop-per-directory invariant" resolution recorded by [[pr_86_agentic-loop-hardening]] (worktrees-as-workaround) for this specific race — worktrees remain useful for a different reason (working-tree isolation between loops), not this collision.
