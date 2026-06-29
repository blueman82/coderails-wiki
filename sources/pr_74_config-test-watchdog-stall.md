---
title: "PR 74 — config.test.sh watchdog stalled 5s per call (orphaned-sleep pipe-fd)"
type: source
created: 2026-06-29
last_updated: 2026-06-29
sources: []
tags: [source, test-runner, config.test.sh, watchdog, performance, comment-accuracy, regression]
---

# PR 74 — config.test.sh watchdog stalled 5s per call

<!-- Ingested by /wiki-ingest after merge. Immutable record of what changed. -->

## PR metadata

| Field | Value |
|---|---|
| PR number | #74 |
| Branch | `bug/config-test-watchdog-slow` |
| Merged | 2026-06-29 (`55e0d63`; fix `eaba541`, comment `f9a2fbc`) |
| JIRA ticket | — |

## Summary

The watchdog [[pr_72_config-walkup-symlink-hang|PR #72]] added to `config.test.sh`
worked correctly but was **slow**: every `resolve()` call paid the full **5 s**
watchdog `sleep`, even on the happy path. With 5 calls that was ~25 s — over half
the whole `run_all.sh` runtime. The suite was ~45 s wall at ~32 % CPU (the low CPU
% was the tell: it was *sleeping*, not computing). Fix drops it to ~21 s. (verified —
measured before/after on origin/main)

## The real mechanism (and a corrected comment)

The naive teardown was `kill $w` (signal the watchdog **subshell**). The first-written
comment explained the stall as "the subshell is blocked inside `sleep` and ignores
SIGTERM." **That explanation is empirically false** and was corrected before merge:

- `kill $w` kills the subshell in ~3 ms — it does **not** ignore SIGTERM. (verified —
  reproduced, 0.024 s)
- But killing the subshell **orphans its `sleep` child** (reparented to init, survives).
  (verified — the orphaned `sleep` outlived `kill $w`)
- That orphaned `sleep` still holds the **write end of the `$()` command-substitution
  pipe**. `$()` blocks until *all* writers close that fd — so it waited the full 5 s for
  the orphan to exit. The stall was a pipe-fd lifetime issue, not a signal issue.

**Fix:** `pkill -P $w` kills the `sleep` child **directly**, closing the pipe fd at once;
the now-childless subshell exits and `wait $w` reaps it (zombie hygiene only). The whole
`$()` body is wrapped in `{ ... } 2>/dev/null` to swallow the job-control
"Terminated: sleep" notice the **parent shell** prints when it reaps the killed job —
that notice surfaces in the owning shell's scope, so only a whole-block redirect catches
it (a narrower `{ wait; } 2>/dev/null` does not). (verified — review + direct test)

## Review caught the false comment

The [[pr_64_loop-review-via-skill]] comment-analyzer flagged the original comment's causal claim as false
(the load-bearing finding); code-reviewer independently proposed a narrower redirect and
then **self-retracted** it after testing (the whole-block wrap is the correct mechanism,
not over-broad); code-simplifier confirmed ship-as-is. The comment was rewritten to the
pipe-fd truth in a follow-up commit. Lesson reinforced: **a comment that misexplains the
mechanism is worse than none** — the fix can be right while its stated reason misleads the
next maintainer. (decision/lesson — this session)

## Caveats / gotchas

- The whole-block `{ } 2>/dev/null` also swallows the resolver's *own* stderr (the inner
  `bash -c` has no redirect of its own) — a minor, bounded debuggability cost. Assertions
  read only stdout, so a real failure still surfaces as FAIL or TIMEOUT; a debugger can
  drop the outer redirect ad-hoc. No clean narrower form keeps both, because the notice and
  the resolver stderr share one scope. (verified — review)
- The watchdog itself is **load-bearing** — it converts a resolver infinite-loop regression
  (the [[pr_72_config-walkup-symlink-hang|#72]] class) into a FAIL/TIMEOUT instead of a
  suite hang. The fix preserves the TIMEOUT path (forced hang still trips at 5 s). Do not
  remove the watchdog. (verified)
- The three inner `2>/dev/null` redirects are now redundant under the outer wrap, but they
  pre-date this PR — left untouched as out-of-scope for a surgical fix. (verified — simplifier)

## Wiki pages updated

- [[config-resolution]] — test-runner section: the #72 watchdog is now fast + its comment
  corrected.
- [[pr_72_config-walkup-symlink-hang]] — back-reference: the watchdog it introduced was
  optimised here.

## See also

[[pr_72_config-walkup-symlink-hang]] · [[config-resolution]] · [[pr_64_loop-review-via-skill]]
`coderails/hooks/scripts/tests/config.test.sh`
