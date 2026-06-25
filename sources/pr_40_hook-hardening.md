---
title: "PR #40 — chore(hooks): gate git merge on main, jq-ify inject_bootstrap, add test runner"
type: source
created: 2026-06-25
last_updated: 2026-06-25
sources: []
tags: [source, hooks, enforcement, inject_bootstrap, test-runner]
---

# PR #40 — hook hardening

<!-- Ingested by /wiki-ingest after merge. This is an immutable record of what changed. -->

## PR metadata

| Field | Value |
|---|---|
| PR number | #40 |
| Branch | `chore/hook-hardening` |
| Merged | 2026-06-25 |
| Merge SHA | `2e31188` |
| JIRA ticket | — |

## Summary

Three independent hardening changes to the hooks subsystem. First, `enforce_pr_workflow.sh` gains a `git merge` gate (in addition to the existing `gh pr create` / `gh pr merge` gates), but only when on `main` or `master`; conflict-resolution ops (`--abort`, `--continue`, `--quit`, `--skip`) and `--help`/`--dry-run` are exempted. Second, `inject_bootstrap.sh` drops its hand-rolled `escape_for_json()` bash function in favour of a single `jq -n --arg ctx ...` call — matching every other hook in the repo. Third, a new `hooks/scripts/tests/run_all.sh` aggregate test runner discovers and executes all `*.test.sh` suites in its directory, counting failures and exiting non-zero if any fail or if zero test files are found.

## Files changed

- `hooks/scripts/enforce_pr_workflow.sh` — added `git merge` gate (Gate 2 conflict-resolution exemption, Gate 3 pattern extended, Gate 4b branch check, subcommand=`git_merge` path through Gate 6 and deny output)
- `hooks/scripts/inject_bootstrap.sh` — removed `escape_for_json()` function + `printf` JSON assembly; replaced with `jq -n --arg ctx "$session_context"` one-liner
- `hooks/scripts/tests/run_all.sh` — new aggregate test runner (zero-test guard, per-suite pass/fail summary, non-zero exit on any failure)
- `hooks/scripts/tests/enforce_pr_workflow.test.sh` — added git-merge test cases (on main with evidence, on main without evidence, on feature branch, `--abort`, master-arm)
- `hooks/scripts/tests/inject_bootstrap.test.sh` — portable plugin-root path (was hardcoded); unconditional anti-double-escape assertion

## Wiki pages updated

- [[enforce_pr_workflow]] — git-merge gate added; logic summary, block condition, skip gates updated
- [[inject_bootstrap]] — jq refactor documented; escape_for_json removal noted

## Caveats / gotchas

- The `git merge` gate only fires on `main`/`master`. Feature branches are unconditionally allowed — the hook exits 0 at Gate 4b. Detached HEAD or empty branch name also falls through (safe-fail default, same as [[no_edit_on_main]]).
- The jq refactor in inject_bootstrap.sh is a clean-break removal: `escape_for_json()` does not exist anywhere in the script anymore. If you're debugging a session-start JSON malformation, the only escaping in play is jq's built-in.
- `run_all.sh` has a zero-test guard: if the glob `*.test.sh` matches nothing (e.g. a misconfigured CI environment), the runner exits 1 rather than silently reporting "0/0 suites passed". This prevents false-success from an empty test directory.
- The evidence check for `git merge` on main reuses the same `review-pr` transcript scan as `gh pr merge` (both require `/pr-review-toolkit:review-pr` to have run). There is no separate gate for git-merge — same evidence, same transcript path.
