---
title: Repo Hosting
type: design
created: 2026-06-01
last_updated: 2026-06-01
sources:
  - .claude-plugin/plugin.json
  - AGENTS.md
tags: [infrastructure, git, github, repo, operational]
---

# Repo Hosting

Where the two coderails repos live and how to clone them. Both are **private** repos
under the github.com **`blueman82`** account (a separate gh-authed account from
`harrison_adobe`; switch with `gh auth switch -h github.com -u blueman82`).

## The two repos

| Repo | Remote | What it is | Clone to |
|---|---|---|---|
| Plugin | `github.com/blueman82/coderails` (private) | The plugin source — commands, hooks, scripts, skills, install/uninstall. Ships via `install.sh` + `/plugin install`. | `~/Documents/Github/coderails` |
| Wiki | `github.com/blueman82/coderails-wiki` (private) | This knowledge base. Obsidian vault; maintained by `/wiki-ingest` + `/wiki-lint`. | `~/Documents/Github/coderails-wiki` |

Both were initialised and pushed on 2026-06-01. (verified: `gh repo create` this session)

## History note — the "not a git repository" claim is now stale

Earlier wiki pages (and `coderails/CLAUDE.md`) state the plugin "is not a git
repository." That was true until 2026-06-01, when the plugin folder was
`git init`-ed and pushed to `blueman82/coderails`. The verify-loop pages'
`(inferred … repo carries no git history)` notes remain accurate **as historical
record** — the 3-generation hook history genuinely could not be read from git at the
time, because there was no git history yet. Going forward, plugin changes *can* be
traced through git. (verified this session)

> ⚠️ NOTE: `coderails/CLAUDE.md` line "It is **not** a git repository" is now stale.
> Not corrected in-source this session (source edits weren't authorised); flagged here.

## Plugin .gitignore

The plugin repo excludes two things only (OS cruft + foreign-plugin scratch):

```
.DS_Store
.remember/
```

`.remember/` is created and owned by the **`remember` plugin** (its SessionStart hook
writes handoff/history buffers there) — machine-local session scratch, not coderails
source, so it never pushes. (verified: `.gitignore` + `.remember/` inspection this session)

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
