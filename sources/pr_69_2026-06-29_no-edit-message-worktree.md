---
title: "PR 69 — no_edit_on_main: steer block message toward worktree + branch"
type: source
created: 2026-06-29
last_updated: 2026-06-29
sources: []
tags: [hook, no_edit_on_main, worktree, message, ux]
---

# PR 69 — no_edit_on_main: steer block message toward worktree + branch

<!-- Ingested by /wiki-ingest after merge. This is an immutable record of what changed. -->

## PR metadata

| Field | Value |
|---|---|
| PR number | #69 |
| Branch | `worktree-no-edit-on-main-worktree-msg` |
| Merged | 2026-06-29 |
| Merge SHA | `86e701a` |
| JIRA ticket | — |

## Summary

Prose-only change to [[no_edit_on_main]]. The hook's PreToolUse deny message (and its
matching header doc-comment) previously framed a plain feature branch as the primary way
to get unblocked, with a worktree as a parenthetical afterthought. The message now leads
with **"create an isolated worktree + branch first"** and gives the concrete invocation
`git worktree add <path> -b <name>` (or `/coderails:prep`). The plain-feature-branch
framing was dropped entirely.

No logic, branch-detection, or allowlist behaviour changed — only the human-readable
`reason` string and the header comment. The 20/20 test suite (`no_edit_on_main.test.sh`)
still passes unchanged because the tests assert on the ALLOW/DENY decision, not the
message text. Two PR-review agents (code-reviewer, comment-analyzer) returned zero
blocking findings.

## Files changed

- `hooks/scripts/no_edit_on_main.sh` — deny `reason` string + header comment (+3/-3)

## Wiki pages updated

- [[no_edit_on_main]] — added a design note on why the message recommends worktree + branch

## Caveats / gotchas

- The reword is also *more accurate* than the old text: "create a feature branch first"
  implied a bare `git checkout -b` in place would suffice, but that leaves you in the same
  working tree on a non-default branch. Worktree + branch is the isolation discipline
  [[prep]] actually enforces ("Feature and bug work must be isolated in a git worktree").
- **`scripts/merge.sh` exits 128 when run from inside a linked worktree.** Observed during
  this PR's merge: the merge itself succeeds, but the script's post-merge `git checkout main`
  fails because `main` is already checked out in the primary working tree (git forbids
  checking out a branch already live in another worktree). The PR still merges cleanly; the
  post-merge branch-switch + pull must then be done manually from the primary tree. (inferred
  — observed sequence: "✓ Merged" then exit 128 only on the checkout step; merge.sh not read
  line-by-line to confirm the exact failing command.) Candidate follow-up if the worktree+merge
  flow is used often.
