---
title: "PR #43 — chore: strip stale failure_log rows, document GitHub-only, surface merge enforcement gap"
type: source
created: 2026-06-25
last_updated: 2026-06-25
sources: []
tags: [source, rough-edges, github-only, enforcement-gap, failure-log, merge]
---

# PR #43 — rough edges cleanup

<!-- Ingested by /wiki-ingest after merge. This is an immutable record of what changed. -->

## PR metadata

| Field | Value |
|---|---|
| PR number | #43 |
| Branch | `chore/rough-edges` |
| Merged | 2026-06-25 |
| Merge SHA | `df4b372` |
| JIRA ticket | — |

## Summary

Three small rough-edges fixes: cleaned the seeded failure log template, made the GitHub-only constraint explicit in the README, and surfaced the `enforce_pr_workflow` opt-in gap in `merge.sh` with an informational notice.

## Files changed

- `templates/failure_log.md` — F1: removed 3 pre-filled dev-history rows from the Entries table; users now start with a clean empty table.
- `README.md` — F2: tightened the Requirements bullet for `/push`/`/merge` to say "GitHub-hosted repo" explicitly.
- `scripts/merge.sh` — F3: added an `info` line before `gh pr merge` when no `workflow.config.yaml` is present, notifying that `enforce_pr_workflow` is inactive and pointing to `/coderails:init`.

## Durable points

**GitHub-only is a deliberate scope decision.** The workflow relies on `gh` (GitHub CLI) and `scripts/lib/git-common.sh`'s `require::repo` helper, which only matches `github.com` remotes. GitLab, Bitbucket, and Gitea are out of scope — not an oversight. (verified: git-common.sh `require::repo` + merge.sh:12 precondition)

**`enforce_pr_workflow` creates a silent no-enforcement gap.** The hook is opt-in: it no-ops when no `workflow.config.yaml` exists. Without the config, `gh pr merge` goes through unguarded. `merge.sh` now surfaces this gap with an explicit info notice so users know enforcement is inactive. This is the correct layering: the hook is the enforcer, the script provides visibility. (verified: scripts/merge.sh F3 addition)

## Wiki pages updated

- [[merge]] — F3: enforcement-gap notice added to Design notes and Preconditions sections.
- [[enforcement-model]] — F2: GitHub-only constraint documented as a deliberate design assumption.

## Caveats / gotchas

F1 (failure_log template cleanup) has no wiki counterpart — no failure_log page exists in the vault, and none is needed. The template is a seeded file for users; its content is user-data, not plugin-architecture knowledge.
