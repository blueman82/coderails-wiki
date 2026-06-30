---
title: "PR #80 — guard both halves of the timeout invariant (assert read -t value matches floor)"
type: source
created: 2026-06-30
last_updated: 2026-06-30
sources:
  - sources/pr_78_hooks-json-timeout-floor.md
  - sources/pr_76_harden-hook-stdin-read.md
tags: [test, hook, stdin, timeout, invariant, guard, pr78-followup, tamper-evident]
---

# PR #80 — guard both halves of the timeout invariant

<!-- Ingested by /wiki-ingest after merge. This is an immutable record of what changed. -->

## PR metadata

| Field | Value |
|---|---|
| PR number | #80 |
| Branch | `test/guard-read-t-floor` (inferred) |
| Merged | 2026-06-30 |
| Merge SHA | `f5aacde` |
| JIRA ticket | — |

## Summary

Extends `hooks/scripts/tests/hooks_json_timeout_floor.test.sh` with "Half B" of the timeout invariant. PR #78 added Half A — asserting `min(declared hooks.json timeout) >= 5`. This PR adds Half B — asserting (1) exactly 10 hook scripts carry the `IFS= read -r -d '' -t N` backstop, and (2) every N equals the floor (5). Together both halves are now **tamper-evident**: a future PR that lowers the in-process `read -t` value without touching hooks.json timeouts, or a hook silently gaining/losing the backstop, now fails the test suite. Also includes two hardening fixes from review: dropped a redundant `|| echo 0` and made the `read -t` extraction end-of-line-tolerant via `grep -oE` instead of `sed` requiring a trailing token. (verified: PR #80 body)

## The two-half invariant

The full invariant is: **`read -t READ_T_FLOOR <= min(hooks.json declared timeout)`**.

Half A (PR #78) guards: `min(hooks.json declared timeout) >= READ_T_FLOOR (5)`.
Half B (this PR) guards: every hook's in-process `read -t N` == `READ_T_FLOOR (5)`.

Without Half B, someone could lower `READ_T_FLOOR` from 5 to, say, 2 inside the scripts while leaving hooks.json at 5 — Half A would still pass, the constraint would be silently broken. With both halves, any change to either side that creates a disagreement is caught automatically. (verified: PR #80 body)

## What Half B tests

Half B adds two new assertions to the existing test file:

**Backstop count assertion**: Discovers all `.sh` files in `hooks/scripts/` (excluding `tests/` and `lib/` subdirectories) that contain the bounded-read backstop idiom (`IFS= read -r -d '' -t`). Asserts the count equals `EXPECTED_BACKSTOP_COUNT` (10). A hook silently gaining or losing the backstop now fails.

**Per-script value assertion**: For each of the 10 discovered hook files, extracts the integer N from `IFS= read -r -d '' -t N` and asserts `N == READ_T_FLOOR`. Uses `grep -oE "read -r -d '' -t [0-9]+"` → `grep -oE '[0-9]+$'` — end-of-line-tolerant (no trailing token dependency). A hook with a divergent timeout value now fails individually with a named error. (verified: PR #80 diff)

## Review hardening fixes

Two fixes landed from the code review before merge:

1. **Dropped redundant `|| echo 0`**: The backstop_count line had `grep -c . 2>/dev/null || echo 0` — the `|| echo 0` was superfluous because `grep -c .` on an empty string input returns `0` with exit 1, and the next `[ "$backstop_count" -ne "$EXPECTED_BACKSTOP_COUNT" ]` test handles that correctly. Removed. (verified: PR #80 body)

2. **End-of-line-tolerant extraction**: The original `read -t` value extraction relied on a trailing token after the integer (e.g. `read -t 5 input`) for the sed pattern to match. Lines formatted as `read -t 5` with nothing after `5` would silently skip. Replaced with `grep -oE` which extracts the last integer from the matched portion regardless of trailing content. (verified: PR #80 body)

## Files changed

| File | Change |
|---|---|
| `hooks/scripts/tests/hooks_json_timeout_floor.test.sh` | Extended — added Half B: backstop count + per-script `read -t N == floor` assertions (+43 lines) |

## Relationship to PR #78

PR #78 added Half A of the invariant guard: asserting hooks.json timeouts cannot be lowered below the `read -t 5` floor.
PR #80 adds Half B: asserting the actual in-process `read -t` value across all hook scripts equals the floor constant.

Together they make the full `read -t 5 <= min(hooks.json timeout)` invariant **fully tamper-evident**. See [[pr_78_hooks-json-timeout-floor]] for Half A context and [[pr_76_harden-hook-stdin-read]] for the original `read -t 5` backstop design.

## Context: closes test-analyzer S1 from PR #78's review

The PR #78 review (test-analyzer agent) raised finding S1:

> "Half A guards the timeout floor but NOT the actual read -t value, so the floor constant could go stale if someone lowered the in-process bound."

PR #80 closes that finding. The two-half structure means the test suite now couples both sides of the invariant: changing either the hooks.json timeouts or the in-process `read -t` value alone (without keeping them in sync) produces a test failure. (verified: PR #80 body)

## Wiki pages updated

- [[pr_78_hooks-json-timeout-floor]] — updated to note that both halves of the invariant are now guarded; "Only Half A" note replaced with "Both halves fully guarded (PR #80)"
- [[index]] — source row for PR #80 added; PR #78 entry updated to note the invariant is now fully covered

## Caveats / gotchas

- `EXPECTED_BACKSTOP_COUNT` is a hardcoded constant (10). Adding a new hook script with the backstop will fail the count assertion — this is intentional (the test forces the author to update the constant deliberately).
- The extraction uses `head -1` — if a hook file had multiple `read -r -d '' -t N` lines with different values, only the first would be checked. In practice each hook has exactly one such call; the count assertion also ensures no new hooks slip by undetected.
- Half B only detects the integer portion of the timeout: `grep -oE '[0-9]+$'`. A fractional value like `read -t 4.5` would match `4` — but bash rejects fractional `-t` values on macOS 3.2 anyway (the constraint documented in PR #76), so `4.5` would already be a portability bug caught at a different layer.
