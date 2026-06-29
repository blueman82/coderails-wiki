---
title: "PR 72 — config.sh walk-up infinite-loop hang on symlinked paths"
type: source
created: 2026-06-29
last_updated: 2026-06-29
sources: []
tags: [source, hook, config-resolution, config.sh, infinite-loop, symlink, test-runner, regression]
---

# PR 72 — config.sh walk-up infinite-loop hang on symlinked paths

<!-- Ingested by /wiki-ingest after merge. Immutable record of what changed. -->

## PR metadata

| Field | Value |
|---|---|
| PR number | #72 |
| Branch | `bug/config-walkup-symlink-hang` |
| Merged | 2026-06-29 |
| Merge SHA | `3862614` (fix commit `04b26c9`) |
| JIRA ticket | — |

## Summary

`coderails::config_path()` in `scripts/lib/config.sh` had an **infinite loop**. The
walk-up from a start dir to the git root terminated only on `d == git_root` *string
equality*. But `git rev-parse --show-toplevel` returns a **symlink-resolved** path
(macOS `/tmp` → `/private/tmp`). When the start dir was in the unresolved namespace,
the equality never matched, `dirname` walked `d` up to `/`, and `dirname /` == `/`
spun forever. (verified — empirical reproduction)

**Fix:** canonicalise `start` with `start=$(cd "$start" && pwd -P)` so it shares
git_root's namespace, plus a hard `|| "$d" == "/"` floor so the loop can never spin
past the filesystem root. (verified — config.sh:39,45)

## Why it only triggered on the NO_CONFIG path

A *found* config short-circuits (`return 0`) **before** reaching the broken
terminator. So the hang fired only when no `.claude/workflow.config.yaml` existed
anywhere in the walk — exactly the NO_CONFIG case. That is why it surfaced as a
hang in `enforce_pr_workflow.test.sh`'s **"no workflow.config.yaml → allow"** case
(check #12), which wedged the whole `run_all.sh` suite at 11 ok with no failure
line — looking identical to "still running." (verified — test isolation)

## Origin — a good design, an incomplete implementation, a widening refactor

- **PR #67** (`2ef04ae`, walk-up) introduced the loop, replacing the old hardcoded
  dual-path `projects/<name>/` lookup. The design was correct (layout-agnostic), but
  the terminator assumed `dirname` lands on git_root exactly. Commit `0ba7cdb` even
  flagged "document the loop's repo-root **assumption**" — the symlink case violates it.
- **PR #71** (`8eabb10`) extracted the loop into `scripts/lib/config.sh` as the shared
  resolver and wired [[enforce_pr_workflow]] to source it — **with no test**. That both
  spread the bug into the hook's opt-in check and removed the chance to catch it.
- **PR #72** (this) fixes the loop and adds the missing test.

(verified — git log lineage)

## The verification trap it caused

Two independent sessions hit this hang and both initially misread it. The captured
suite output stopped at the same line (entering `enforce_pr_workflow.test.sh`) with
**0 FAIL** — and "no failure line" was wrongly read as "passing." It was actually
"killed mid-hang." Lesson recorded: **absence of a failure signal is not evidence of
success**, especially when the process was killed. The Stop-hook DNV/label gates
blocking the premature "all green" claim were correct — there really was a bug, just
not in the PR under review. (decision/lesson — this session)

## Tests

New `hooks/scripts/tests/config.test.sh` (5 cases), placed in `hooks/scripts/tests/`
because that is the **only** dir `run_all.sh` globs (it does `cd "$(dirname "$0")"`
then `for f in *.test.sh` — a `scripts/tests/` location would never be discovered).
Each case runs the resolver under a **watchdog** (background job + sleeper `kill -9`,
exit 137 → `TIMEOUT`) so a regression surfaces as a FAIL, never an actual hang. The
reproducer (`ln -s "$REAL" "$LINK"`, resolve `$LINK`) forces the namespace mismatch
on any OS, not just the macOS `/tmp` quirk. Verified to FAIL pre-fix (TIMEOUT) and
PASS post-fix — the gold standard for a regression test. Full suite now **15/15,
307 ok / 0 FAIL** (was hanging). (verified — both review agents + direct run)

## Impact

- [[config-resolution]] — the design page documented the OLD dual-path lookup; updated
  to the walk-up resolver (#67/#71) and this hang fix.
- [[enforce_pr_workflow]] — sources `config.sh` for its opt-in check; was the hang's
  point of manifestation.
- Test-runner gotcha recorded: `run_all.sh` discovers only its own directory.

## See also

[[config-resolution]] · [[enforce_pr_workflow]] · [[discipline-loop]]
`coderails/scripts/lib/config.sh` · `coderails/hooks/scripts/tests/config.test.sh`
