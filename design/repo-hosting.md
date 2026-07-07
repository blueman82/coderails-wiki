---
title: Repo Hosting
type: design
created: 2026-06-01
last_updated: 2026-07-07
sources:
  - .claude-plugin/plugin.json
  - AGENTS.md
  - sources/repo-consolidation_2026-07-05.md
tags: [infrastructure, git, github, repo, operational]
---

# Repo Hosting

Where the coderails repos live and how to clone them. Both are **private** repos
under the github.com **`blueman82`** account (switch with `gh auth switch -h
github.com -u blueman82`). The plugin repo's history was scrubbed clean of a
legacy commit-author identity on 2026-07-05 — it is now flippable to public at
any time. See [[repo-consolidation_2026-07-05]] and [[history-identity-scrub]].

## The repos

| Repo | Remote | What it is | Clone to |
|---|---|---|---|
| Plugin | `github.com/blueman82/coderails` (private) | The plugin source — commands, hooks, scripts, skills, install/uninstall. Ships via `install.sh` + `/plugin install`. Full dev history (349 commits), clean identity. | `~/Documents/Github/coderails` |
| Wiki | `github.com/blueman82/coderails-wiki` (private) | This knowledge base. Obsidian vault; maintained by `/wiki-ingest` + `/wiki-lint`. | `~/Documents/Github/coderails-wiki` |

The plugin repo was initialised 2026-06-01. (verified: `gh repo create`)

## Consolidation — there was briefly a third repo (now gone)

Between 2026-07-03 and 2026-07-05 the plugin ran a **two-repo split**: the private
development repo plus a separate scrubbed single-commit *public export* repo, to
avoid exposing a legacy author identity in old history. On 2026-07-05 the history
was rewritten clean, the export repo was deleted, and the dev repo was renamed to
the canonical `coderails` — collapsing back to **one repo**. The name
`coderails-dev` (the dev repo's temporary name during the split) is now only a
stale GitHub rename redirect that resolves to `coderails`; it is **not** a
separate repo — never push to it. Full method: [[history-identity-scrub]].
(verified 2026-07-05 via `gh repo view` + `--mirror` clone)

## PR-number collision after the 2026-07-05 recreation

The delete-and-recreate step in [[history-identity-scrub]] reset GitHub's PR-number
sequence. This means wiki source pages that cite a PR number from **before** the
2026-07-05 recreation can collide with an unrelated PR bearing the same number
merged **after** it. Confirmed 2026-07-07: `sources/pr_69_no-edit-message-worktree.md`
and `sources/pr_70_gate-settings-json-edits.md` cite PR #69/#70 merged 2026-06-29 (in
the pre-recreation repo), while `gh pr view 69`/`gh pr view 70` today resolve to two
different, unrelated PRs merged 2026-07-07 (`security/wu1 subst audit` and `dashboard
input delivery`). Both pairs of pages are individually correct — they document real
work — but a bare PR-number citation is no longer a reliable unique key across the
recreation boundary. New source pages that would otherwise collide should use a
date-qualified filename (e.g. `pr_70-71_2026-07-07_<slug>.md`, see
[[pr_70-71_2026-07-07_dashboard-input-fix-and-voice-announcements]]) rather than the
bare `pr_<N>_<slug>.md` form. **Not yet done:** retitling the two older colliding
pages so a plain `pr_69`/`pr_70` search doesn't surface both eras ambiguously.

## History note — the "not a git repository" claim is now stale

Earlier wiki pages (and `coderails/CLAUDE.md`) state the plugin "is not a git
repository." That was true until 2026-06-01, when the plugin folder was
`git init`-ed and pushed to `blueman82/coderails`. The verify-loop pages'
`(inferred … repo carries no git history)` notes remain accurate **as historical
record** — the 3-generation hook history genuinely could not be read from git at the
time, because there was no git history yet. Going forward, plugin changes *can* be
traced through git. (verified this session)

`coderails/CLAUDE.md` was corrected on 2026-06-01 (`9f15823`): the "not a git
repository" line now reads "version-controlled in the private repo
`github.com/blueman82/coderails`." (verified: edit committed + pushed this session)

## Plugin .gitignore

The plugin repo excludes only machine-local, non-source files (OS cruft,
foreign-plugin scratch, runtime logs):

```
.DS_Store
.remember/
logs/
```

`.remember/` is created and owned by the **`remember` plugin** (its SessionStart hook
writes handoff/history buffers there) — machine-local session scratch, not coderails
source, so it never pushes. `logs/` holds runtime hook logs, added by a second machine
(`b5143fb`). (verified: `.gitignore` this session)

## Wiki .gitignore (Obsidian)

The wiki vault tracks Obsidian config (`app.json`, `appearance.json`, `graph.json`,
enabled-plugin manifests, and the Marp plugin's committed code) but ignores the
volatile workspace layout:

```
.obsidian/workspace.json
.obsidian/workspace-mobile.json
```

A fresh `git clone` on a machine with Obsidian opens cleanly as a vault — theme,
graph settings, and enabled plugins intact; only the last pane layout regenerates.
(verified: `git ls-files .obsidian/` + `.gitignore` this session)

## Related

- [[install-and-cache-trap]]
- [[enforcement-model]]
- [[pr_70-71_2026-07-07_dashboard-input-fix-and-voice-announcements]] — the source page that surfaced the PR-number collision above
