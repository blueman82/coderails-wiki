---
title: "PR #288 — VITEST guard on defaultNotify() stops tests firing real macOS notifications"
type: source
created: 2026-07-24
last_updated: 2026-07-24
sources: []
tags: [dashboard, runner, escalate, notify, osascript, tests, vitest, false-positive]
---

# PR #288 — VITEST guard on defaultNotify() stops tests firing real macOS notifications

<!-- Ingested by /wiki-ingest after merge. This is an immutable record of what changed. -->

## PR metadata

| Field | Value |
|---|---|
| PR | [#288](https://github.com/blueman82/coderails/pull/288) |
| Title | notify leak in tests |
| Branch | `fix/notify-leak-in-tests` |
| Merged | 2026-07-24T00:31:55Z |
| Merge commit | `a3ba36c` |
| Head at merge | `3406e64` |
| Files | `skills/dashboard/runner/src/escalate.ts` (+9), `test/escalate.test.ts` (+17/-7), `test/sweep.test.ts` (+2/-2) — 3 files, +28/-9 |

## The incident

On 2026-07-22 a run of `sweep.test.ts` fired genuine macOS notifications via `osascript` —
titled `Routine failed: run-a`/`Routine failed: run-b` — indistinguishable at a glance from a
real routine failure. See [[dashboard-runner]]'s escalation-taxonomy section for the two-channel
escalation design (`osascript` notification + run note) this leaked out of.

## Root cause

`defaultNotify()` in `skills/dashboard/runner/src/escalate.ts` had no test-environment guard.
Any test that exercises a failure path without passing its own `notifyImpl` falls through to the
real notifier. `sweep.test.ts`'s per-intent failure-boundary test (forcing `appendRun` to throw
by pointing `runsDir` at a file instead of a directory) drove exactly this path without a mock.

## The fix

`defaultNotify()` now checks `process.env.VITEST` and returns before reaching the `osascript`
call — the guard is the **first statement in the function body**:

```ts
export function defaultNotify(title: string, message: string): void {
  if (process.env.VITEST) return;
  // ...osascript invocation unchanged below
}
```

Two follow-on changes make the guard's own coverage and an already-flaky test consistent:

- **`escalate.test.ts`'s argv-injection test** asserts on `defaultNotify`'s own body (verifying
  the `osascript` argv-injection defence documented in [[dashboard-runner]]), so it must defeat
  the new guard to still exercise that code path. It deliberately `delete`s
  `process.env.VITEST` around the `escalate()` call and restores it in a `finally` block. Safe
  because `node:child_process` is mocked file-wide in that test file, so `execFileSync` never
  reaches a real `osascript`.
- **`sweep.test.ts`'s previously-leaking test** now also passes an explicit `notifyImpl: vi.fn()`
  mock to `sweepOnce()`, so it no longer depends solely on the `VITEST` guard to stay silent —
  belt-and-braces against the same leak recurring if the guard is ever weakened.

A one-word comment fix in the same `sweep.test.ts` test corrects a wrong errno name
(`ENOTDIR` → `EEXIST`) in the comment describing why `appendRun`'s `mkdirSync(dir, {recursive:
true})` throws when `runsDir` points at a file — comment-only, no behavioural change.

## Verification

`(inferred — this ingest did not independently re-run the suite; the fix is a 9-line
process.env.VITEST early-return plus two test-side adjustments, verified by reading the merged
diff directly)`.

## Scope note

A local worktree for this branch carried additional unpushed work — a `sweep.ts` timeout-marker
feature and `skills/docs-sync/SKILL.md` changes — that was **not** part of what merged. Merge
commit `a3ba36c` contains only the 3-file `VITEST`-guard fix described above.

## Wiki pages updated

- [[dashboard-runner]] — "Injection-hardening in the notification path" section gains a note on
  the `VITEST` test-environment guard now sitting in front of the `osascript` call.

## Related

- [[pr_262_runner-stdin-stall]] — the same `escalate.ts`/`defaultNotify()` injection-hardening
  this guard sits alongside, and the same runner sub-project's pattern of unit-test seams hiding
  real-world defects (there: a passing test hid a live stdin stall; here: a passing test caused a
  live notification leak — same lesson, opposite direction).
- [[pr_260_263_dashboard-security-review]] — prior security-review pass over this same
  escalation/notification path.
