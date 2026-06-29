---
title: "PR 70 — gate .claude/settings.json edits"
type: source
created: 2026-06-29
last_updated: 2026-06-29
sources: []
tags: [source, hook, no_edit_on_main, settings-json, self-escape, enforcement-ceiling]
---

# PR 70 — gate .claude/settings.json edits

<!-- Ingested by /wiki-ingest after merge. Immutable record of what changed. -->

## PR metadata

| Field | Value |
|---|---|
| PR number | #70 (supersedes closed #68) |
| Branch | `feature/gate-settings-json-v2` |
| Merged | 2026-06-29 |
| Merge SHA | `e52e541` |
| JIRA ticket | — |

## Summary

Adds a third arm to [[no_edit_on_main]] — a **permission-file arm** — that blocks
`Write`/`Edit`/`MultiEdit` on `.claude/settings.json` and `.claude/settings.local.json`
on **any** branch, in any repo. Matched on the `.claude/` parent (a `case` glob:
`*/.claude/settings.json|.claude/settings.json|*/.claude/settings.local.json|.claude/settings.local.json`)
so an unrelated `settings.json` elsewhere — `app/settings.json`, `.claude/sub/settings.json` —
passes. (verified — PR #70, no_edit_on_main.sh:32-54)

## Why — closing the self-escape door

The `settings.json` `permissions.allow` list can pre-approve a Bash command (e.g.
`Bash(git push:*)`), and that pre-approval resolves **upstream** of the PreToolUse hooks —
so an allowlisted command can skip [[enforce_pr_workflow]] entirely. The documented "escape"
for every gate is *adding such a rule*. That makes the permission file the one edit that can
dismantle the whole discipline layer. Before this PR, an agent could write that escape itself
via the edit tools. Now it can't. (verified — PR #70 rationale + [[enforcement-model]])

This is friction, not a lock — consistent with the [[enforcement-model]] ceiling:

- **Does close:** the edit-tool path to the permission file.
- **Does NOT close:** Bash-path edits (`sed -i`, `>` redirect, `tee` → `.claude/settings*.json`).
  Named as a follow-up via [[destructive_bash_gate]]'s in-Bash source-edit arm; not built here.
- **Cannot close locally:** an agent disabling the plugin or editing `hooks.json` — same trust
  domain. The true server-side lock remains **GitHub branch protection**.

## Any-branch, not main-only

Deliberate: a settings escape is dangerous on a feature branch too, so the arm fires before
the allowlist (these files are `.json`, otherwise allowlisted) and before the branch check.
This is the first [[no_edit_on_main]] arm that is **not** gated on main/master. (decision — PR #70)

## Collateral fix — ceiling_note test retarget

`ceiling_note.test.sh` grepped `CLAUDE.md` for the enforcement-ceiling marker phrases
(`redirect-and-audit layer`, `branch protection`, `invocation`). PR #67 consolidated the
working guide out of CLAUDE.md (now a thin pointer) into `AGENTS.md`, leaving the test stale —
it would have failed on main independently of this PR. Retargeted `GUIDE_MD` to `AGENTS.md`.
(verified — ceiling_note.test.sh)

## Process gotcha — force-push blocked, fresh branch used

PR #68 was the original. After rebasing #68 onto current main (to absorb #67's CLAUDE.md
consolidation + #71's config-lib), pushing the rewritten history needed `git push
--force-with-lease` — which [[destructive_bash_gate]] permanently blocks (force-push family,
no `--with-lease` exception). Rather than self-bypass via settings.json (now itself gated by
this very change), the rebased commits were pushed to a **fresh** branch
`feature/gate-settings-json-v2`, #68 closed as superseded, and #70 opened + merged. The gate
working as designed forced the safe route. (decision — this session)

## Tests

11 cases in `no_edit_on_main.test.sh`: both filenames; relative, `./`-relative, and absolute
forms; feature **and** main branches (proving any-branch); cross-repo mismatched-cwd; negatives
(`app/settings.json` not under `.claude/`, `.claude/sub/settings.json` and
`.claude/sub/settings.local.json` deeper-nested). Full hook suite green. (verified)

## Impact

- [[no_edit_on_main]] — third arm added; first any-branch gate in the hook.
- [[enforcement-model]] — concrete instance of "the documented escape is the attack surface."
- [[destructive_bash_gate]] — named owner of the un-built Bash-path follow-up.

## See also

[[no_edit_on_main]] · [[enforcement-model]] · [[destructive_bash_gate]] · [[discipline-loop]]
`coderails/hooks/scripts/no_edit_on_main.sh`
