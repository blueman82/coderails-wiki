---
title: "Dashboard lockfile emnapi drift — 2026-07-08"
type: investigation
created: 2026-07-08
last_updated: 2026-07-08
sources: []
tags: [dashboard, npm, lockfile, npm-ci, native-deps, closed-not-merged]
---

# Dashboard lockfile emnapi drift

> Filed: 2026-07-08. Point-in-time snapshot — may be superseded.

> Note: this is a **closed-not-merged PR finding**, not the usual merged-PR
> record — filed here rather than in `sources/` because there's no merge to
> make immutable. Also: this topic has no relation to
> [[pr_79_sync-docs-drift]] (PR #79, docs drift, merged 2026-06-29) — flagging
> only because a same-numbered-PR mixup is an easy mistake in this vault, and
> PR numbers are not part of this page's name for that reason.

## Question

Why did `npm ci` fail on the dashboard's first run, and was the standalone
fix PR (`fix/dashboard-lockfile-emnapi-drift`, opened 2026-07-07) the thing
that resolved it?

## Evidence

- `skills/dashboard/scripts/start-dashboard.sh` runs `npm ci → build → start`
  as its first step (see [[dashboard]]).
- `npm ci` failed with `EUSAGE` because `skills/dashboard/app/package-lock.json`
  was missing two transitive/optional native-dep entries:
  `@emnapi/core@1.11.2` and `@emnapi/runtime@1.11.2`.
- The `fix/dashboard-lockfile-emnapi-drift` PR regenerated the lock via
  `npm install --package-lock-only`, adding the two missing entries.
- Checked against main's current committed
  `skills/dashboard/app/package-lock.json`: `npm ci --dry-run` resolves
  cleanly (exit 0, 408 packages, no `EUSAGE`), and the lockfile already
  contains the `@emnapi` entries (19 refs).
- `git diff origin/main..<pr-head> -- package-lock.json` is **empty** —
  byte-identical to main's already-fixed lockfile.

## Findings

(verified) The drift was real and did block `npm ci` on first run. (verified)
The standalone PR's fix was correct in isolation (regenerate lock via
`npm install --package-lock-only`). (verified) By the time this PR was
reviewed, a separate, later dashboard PR had already regenerated and
committed the same lockfile fix to main, making this PR's diff a no-op —
confirmed by the empty `git diff` above.

## Resolution

PR closed as superseded, not merged. No lockfile change needed — main
already carries the fix.

**General lesson:** optional/transitive native deps (`@emnapi/*` and similar
napi packages) can drift out of a committed `package-lock.json`, and
`npm ci` (exact-lock, unlike `npm install`) fails hard on that drift where
`npm install` would silently repair it. The fix is
`npm install --package-lock-only`. When multiple sessions touch dashboard
deps in parallel, one session's lock regeneration can silently resolve
another session's pending fix — verify a lockfile PR's diff against current
main before merging it, not just against the branch's own base commit.

## See also

[[dashboard]] — first-run build step this drift blocked.
