---
title: "PR #78 — guard hooks.json timeout floor >= read -t 5 backstop"
type: source
created: 2026-06-29
last_updated: 2026-06-30
sources:
  - sources/pr_80_guard-read-t-floor.md
tags: [test, hook, stdin, timeout, invariant, guard, pr76-followup]
---

# PR #78 — guard hooks.json timeout floor >= read -t 5 backstop

<!-- Ingested by /wiki-ingest after merge. This is an immutable record of what changed. -->

## PR metadata

| Field | Value |
|---|---|
| PR number | #78 |
| Branch | `test/hooks-json-timeout-floor` (inferred) |
| Merged | 2026-06-29 |
| Merge SHA | `ceabb75` |
| JIRA ticket | — |

## Summary

Closes the **known gap** deferred from PR #76's 6-agent review: no automated test guarded `min(hooks.json timeout) >= 5`. Added `hooks/scripts/tests/hooks_json_timeout_floor.test.sh`. Auto-discovered by `run_all.sh`'s `*.test.sh` glob — no wiring needed. Suite 17/18 → 18/18. (verified: PR #78 body)

## The invariant

`read -t 5` in every hook script is the in-process backstop for the orphaned-hook scenario. For the in-process backstop to be meaningful, the `hooks.json` harness-level timeout must be **>= 5** for every hooks/scripts/ entry. If a hooks.json timeout were declared below 5, the harness would kill the hook before the in-process backstop could fire — silently breaking the documented guarantee without any obvious failure signal. (verified: PR #78 body)

Today the minimum declared timeout is exactly 5 (`inject_context.sh`, `destructive_bash_gate.sh`, `no_edit_on_main.sh`) — so the invariant holds only by equality with zero margin. This PR turns that implicit equality into an explicit machine check.

## What the test does

- Parses `hooks/hooks.json` with `jq` and extracts every declared `timeout` for entries whose `command` references `hooks/scripts/`
- Hooks with no declared timeout use Claude Code's 60 s default (safe) and are **intentionally excluded** (`select(.timeout != null)`)
- Asserts `min(declared timeouts) >= 5`; on violation, prints a clear message naming the offending value
- Is parameterisable on the JSON path (default: real `hooks/hooks.json`) so the fixture-RED path can be demonstrated without repo changes
- Auto-discovered by `run_all.sh`'s `*.test.sh` glob — no wiring needed

## The fractional-timeout bug fixed during review

A naive implementation would compare timeouts using `bash -lt`, which silently errors on a float (e.g. `4.5`). This would cause a fractional timeout below the floor to silently be treated as a pass — a false-negative failure mode. The fix: move the comparison entirely into `jq`, which handles floats correctly. (`jq`: `min < $floor` → `"below_floor": true/false`; bash only inspects the boolean result.) (verified: PR #78 body + script)

The empty-result guard also fixed: if `jq` returns an empty set (e.g. broken filter or renamed field), the old approach would silently skip the assertion. Now the test FAILs explicitly when `count == 0`. (verified: script lines 46–51)

## Test results

```
ok   - min declared hooks/scripts/ timeout (5) >= read -t 5 floor
PASS
```

Against a fixture with timeout=3:
```
FAIL - hooks.json declares a timeout (3) below the read -t 5 in-process backstop floor
FAILED (1)
```

Full suite: 18/18 suites passed. (verified: PR #78 body)

## Files changed

| File | Change |
|---|---|
| `hooks/scripts/tests/hooks_json_timeout_floor.test.sh` | New — guard test for hooks.json timeout floor invariant (69 lines) |

## Context

Deferred follow-up from PR #76 review (comment-analyzer S1, silent-failure-hunter I2). That review noted:

> "Known gap (deferred): A guard test asserting `min(hooks.json timeout) >= 5` does not exist. The 5 ≤ 5 invariant currently holds only by equality and is unguarded."

PR #78 closes that gap. See [[pr_76_harden-hook-stdin-read]] for the full context of the `read -t 5` backstop design.

## Relationship to PR #76 and PR #80

PR #76 added the `read -t 5` in-process backstop across all 10 hook scripts.
PR #78 adds Half A of the invariant guard: asserting `min(hooks.json declared timeout) >= 5` so the harness-level timeouts cannot be lowered below the backstop without a test failure.
PR #80 (merged 2026-06-30) adds Half B: asserting every hook's in-process `read -t N` value equals the floor (5), and exactly 10 hook scripts carry the backstop idiom.

**Both halves are now fully guarded (PR #80)**: the full `read -t 5 <= min(hooks.json timeout)` invariant is tamper-evident end-to-end. A future PR that lowers the in-process `read -t` value alone (without touching hooks.json) OR a hook that silently gains/loses the backstop now fails the test suite. See [[pr_80_guard-read-t-floor]] for the Half B implementation details.

## Wiki pages updated

- [[pr_76_harden-hook-stdin-read]] — "Known gap" note updated: gap now CLOSED by PR #78
- [[discipline-loop]] — timeout-floor invariant guard noted in Stdin read convention section

## Caveats / gotchas

- The test only checks **declared** timeouts. Hooks with no `timeout` field in hooks.json use the Claude Code harness default (60 s) — these are safe and deliberately excluded.
- The jq filter walks `hooks | to_entries[] | .value[] | .hooks[]?` — if the hooks.json structure changes (e.g. renamed `hooks` key at top level), `count == 0` triggers the empty-result guard FAIL, surfacing the breakage explicitly.
