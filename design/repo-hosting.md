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
(`cc55b14`). (verified: `.gitignore` this session)

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
