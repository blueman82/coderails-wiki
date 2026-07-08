---
title: "PR #108 — offload_push_guard.sh Stop/SubagentStop hook"
type: source
created: 2026-07-08
last_updated: 2026-07-08
sources: []
tags: [hook, stop-hook, subagentstop, nudge, enforce-pr-workflow]
---

# PR #108 — Add offload_push_guard.sh Stop/SubagentStop hook

## PR metadata

| Field | Value |
|---|---|
| PR number | #108 |
| Branch | `hooks/offload-guard` |
| Merged | 2026-07-08 |
| Merge SHA | `fe8d794` |
| JIRA ticket | — |

## Summary

New `Stop`/`SubagentStop` hook, `offload_push_guard.sh`: nudges (never blocks)
when the final assistant message asks the user to run a `git push` to
`main`/`master` from their own shell, instead of clearing
[[enforce_pr_workflow]]'s push gate itself by running
`/pr-review-toolkit:review-pr` in-session first. Requires **both** a
push-to-main/master token and an offload-to-user cue in the same message (a
leading `! ` prefix, "your own shell", "run this yourself", "from your shell",
"you run", "needs your shell", "un-gated shell") — a plain "I pushed to main"
or a `/coderails:push` suggestion never matches. Nudge-once-per-session via the
discipline-log ledger, the same idiom [[unregistered_loop_guard]] uses.

Full hook detail: [[offload_push_guard]].

## Files changed

- `hooks/scripts/offload_push_guard.sh` — new hook script
- `hooks/scripts/tests/offload_push_guard.test.sh` — new test suite (20/20 assertions)
- `hooks/hooks.json` — registered in the `Stop` and `SubagentStop` arrays
- `docs/REFERENCE.md` — new Hook Activation Matrix row + lib-consumer updates
- `hooks/scripts/tests/hooks_json_timeout_floor.test.sh` — `EXPECTED_BACKSTOP_COUNT` bumped 13→14

## Wiki pages updated

- [[offload_push_guard]] — new hook page
- [[unregistered_loop_guard]] — cited as the sibling hook this one's contract and ledger idiom are drawn from

## Caveats / gotchas

- A worker's `/pr-review-toolkit:review-pr` run does **not** clear
  `enforce_pr_workflow`'s gate — the gate scans the *orchestrating* session's
  own transcript, not a subagent's. The nudge text says this explicitly so an
  agent doesn't "fix" the offload by delegating review-pr to a worker instead
  of running it itself.
- Test plan noted one pre-existing, environment-dependent failure
  (`install_mode_sweep.test.sh`, 37/38) reproduced identically on baseline
  `origin/main` before this change — not a regression introduced by this PR.

## Verification

- (verified) Hook source, gate order comment, regex definitions, and
  nudge-once ledger logic read directly from
  `git show origin/main:hooks/scripts/offload_push_guard.sh`.
- (verified) PR metadata (merge SHA, summary, test-plan claims) from
  `gh pr view 108 --json title,body,mergedAt,number`.

## See also

- [[offload_push_guard]] — the hook this PR ships
- [[enforce_pr_workflow]] — the push gate the hook nudges agents toward
  clearing correctly
- [[unregistered_loop_guard]] — sibling nudge-contract hook
