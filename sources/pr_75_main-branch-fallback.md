---
title: "PR 75 — main() falls back to 'main' on empty symbolic-ref"
type: source
created: 2026-06-29
last_updated: 2026-06-29
sources: []
tags: [git-common, sync, bugfix, pipeline-exit-status]
---

# PR 75 — main() falls back to 'main' on empty symbolic-ref

<!-- Ingested by /wiki-ingest after merge. This is an immutable record of what changed. -->

## PR metadata

| Field | Value |
|---|---|
| PR number | #75 |
| Branch | `bug/main-branch-fallback` |
| Merged | 2026-06-29 |
| Merge SHA | `feaefe9` |
| JIRA ticket | — |

## Summary

The `main()` helper in `scripts/lib/git-common.sh` answers one question: *what is this
repo's default-branch name?* It reads `git symbolic-ref refs/remotes/origin/HEAD`, strips
to the last path segment, and is supposed to fall back to the literal `main` when git can't
resolve the marker. The fallback never fired. `main()` returned a **blank string** whenever
`refs/remotes/origin/HEAD` was unset.

Root cause is a **pipeline-exit-status** bug — the same class as [[pr_74_config-test-watchdog-stall]]'s
mis-mechanism, here in its purest form. (`sync::main_branch`, the consumer this bug starved,
was added by PR #73 — which was never ingested, so it has no source page.)

```bash
# before — || binds to the PIPELINE's exit status, which is sed's
main() { git symbolic-ref refs/remotes/origin/HEAD 2>/dev/null | sed 's@.*/@@' || echo main; }
```

A pipeline's exit status is its **last** command — `sed`. `sed` exits `0` even on empty
input. So when `git symbolic-ref` fails (no marker), `sed` emits nothing and returns success,
the `||` sees success, and `echo main` never runs. Result: blank.

```bash
# after — capture the result, substitute-if-empty
main() { local m; m=$(git symbolic-ref refs/remotes/origin/HEAD 2>/dev/null | sed 's@.*/@@'); echo "${m:-main}"; }
```

`${m:-main}` keys the fallback off **empty output** — the actual symptom — not git's exit
code. This is the stronger contract: "blank for any reason → fall back", which is what the
helper always should have had. (Rejected alternatives: `set -o pipefail` has blast radius
across the whole sourced lib, whose other functions deliberately swallow pipeline failures;
`${PIPESTATUS[0]}` works but is brittle for a one-liner.)

## Files changed

- `scripts/lib/git-common.sh` — the one-line `main()` fix.
- `hooks/scripts/tests/git-common.test.sh` — new regression test: a bare `git init` repo
  (no `remote set-head`, so no `origin/HEAD`) must yield `main`, not blank. The deliberate
  inverse of the file's existing topology, which calls `remote set-head origin main` to make
  `main()` resolve (test lines 28-31).

## When the marker is missing (why it stayed latent)

`refs/remotes/origin/HEAD` is set by `git clone` / `gh repo clone`. Normal everyday clones
have it, so `main()` worked in real use — the [[pr_73]]-era #73 merge resolved the branch
correctly. The blank only appeared in **bare `git init` test scaffolding** (no remote → no
marker). A broken safety net that hadn't fallen yet.

## Wiki pages updated

- (none beyond index + log) — `main()` and `sync::main_branch` have no dedicated page; the
  helper is documented inline where it's consumed ([[merge]], [[push]]).

## Caveats / gotchas

- **`${m:-main}` substitutes on empty, not whitespace.** If git ever emitted whitespace-only
  output that survived `sed`, the fallback wouldn't fire. Not reachable through any real path
  (`symbolic-ref` emits one ref line; `sed s@.*/@@` collapses to the last segment), so
  hardening further is YAGNI — an accepted boundary, consistent with the "enumerated, not
  exhaustive" limits in [[enforcement-model]].
- **Downstream blast radius was a silent no-op, not a crash.** A blank `main()` fed
  `sync::main_branch` (added in #73), which then quietly synced nothing useful. Caught by the
  silent-failure review: external callers in `push.sh`/`merge.sh` consume `main()` as a git
  ref argument and are now safe given a guaranteed-non-empty return.
- Three independent reviews (code, tests, silent-failure) all clean, 0 findings.

## Lesson

`cmd | filter || fallback` keys the fallback off **`filter`'s** exit, not `cmd`'s. When the
failure you care about is upstream of a filter that exits 0 on empty input, the `||` is dead
code. Capture and test the **output** (`${var:-default}`) instead of chaining `||` onto a
pipeline. Same family as the #74 watchdog mis-mechanism: reason about *which* process owns the
exit status / the fd / the signal.
