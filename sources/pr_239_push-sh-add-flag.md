---
title: "PR #239 — push.sh --add flag closes the under-staging gap"
type: source
created: 2026-07-18
last_updated: 2026-07-18
sources: []
tags: [command, push, staging, git-add, untracked-files]
---

# PR #239 — push.sh --add flag closes the under-staging gap

<!-- Ingested by /wiki-ingest after merge. This is an immutable record of what changed. -->

## PR metadata

| Field | Value |
|---|---|
| PR number | #239 |
| Branch | `feat/push-sh-add-flag` |
| Merged | 2026-07-18 |
| Merge SHA | `777c8f4e40026dbb2b24a45b764b4dad02d63856` |
| JIRA ticket | — |

## Summary

Adds a repeatable `--add <path>` flag to `scripts/push.sh` so a caller can
explicitly name its own new untracked files to be staged (`git add -- <paths>`)
in addition to the existing `git add -u` (tracked-modified only). Default
no-flag behaviour is byte-identical to before — foreign untracked files stay
excluded by construction, since `--add` only stages what it's told to name.
`commands/push.md` is updated to instruct the actor invoking `/push` to pass
`--add` for any new files it created; the actor (not push.sh) is responsible
for knowing what it created. New test coverage in
`hooks/scripts/tests/push_staging.test.sh` covers: named-file staging with a
foreign file present and excluded, `--add` combined with a commit message in
both argument orders, multiple repeated `--add` flags, and a default-path
regression lock (no `--add` ⇒ new untracked file still not staged).

## Files changed

- `scripts/push.sh` — `push::main` gains a `want_add` latch that consumes the
  token immediately following each `--add` occurrence into an `add_paths`
  array; after `git add -u`, `git add -- "${add_paths[@]}"` runs if any paths
  were named.
- `commands/push.md` — documents the flag, explains why it's opt-in
  (`push.sh` never sweeps untracked files by design — see [[pr_11-14_gate-hardening-followups|PR #13]]), and warns against passing a
  space-separated variable (`--add $FILES`) since `--add` consumes exactly one
  token per occurrence.
- `hooks/scripts/tests/push_staging.test.sh` — five new check blocks (see
  Caveats).

## Wiki pages updated

- [[push]] — staging section extended to document `--add`
- [[pr_224_231_233_235_loop-tooling-hardening]] — cross-referenced as the
  incident that surfaced this gap (see Caveats)

## Caveats / gotchas

**The bug this fixes.** `push.sh`'s `git add -u` (since [[pr_11-14_gate-hardening-followups|PR #13]]) stages tracked-modified files only,
then *warns* about untracked files rather than staging them — deliberate, to
stop foreign working-tree files from being swept into a PR. But that means a
worker's own genuinely-new file (a new test file, a new source file) was
*also* silently left out unless someone ran a separate `git add` before
invoking `/push`. This hit [[pr_224_231_233_235_loop-tooling-hardening|PR #231]]: `run_all_skip.test.sh` was a new file
the worker created as part of that PR's `run_all.sh` SKIP-class change, and it
was at risk of dropping out of the PR under the old staging path — caught by
the worker, not by an automated gate. `--add` closes the gap without
reintroducing the `git add -A` over-staging problem PR #13 fixed: a file only
gets staged if it's named, so a *foreign* untracked file (something another
concurrent session left in the working tree) is still never swept up.

**Flag parsing is single-token-per-occurrence, not multi-value.** `--add`
must be repeated per file (`--add a --add b`), never
`--add $SPACE_SEPARATED_LIST` — the `want_add` latch consumes exactly the one
token following each `--add`, so a space-separated variable would silently
stage only the first path and misparse the rest as extra arguments (one of
which could get swallowed into the commit message). `commands/push.md` calls
this out explicitly as a footgun.

**Argument-order independence tested both ways.** `push.md`'s actual
invocation shape passes the commit message before `--add`
(`"$ARGUMENTS" --add path`); the test suite locks both that order and the
reverse (`--add path "message"`) to prove the `want_add` latch doesn't care
which side of the message the flag lands on.

**Directories/globs are out of scope.** `--add` takes individual file paths
only — passing a directory would stage everything under it, defeating the
whole point. Not enforced in code; documented as a caller responsibility in
`push.md`.
