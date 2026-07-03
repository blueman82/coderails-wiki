---
title: "Skill: wiki-lint"
type: skill
created: 2026-06-25
last_updated: 2026-07-03
sources:
  - skills/wiki-lint/SKILL.md
  - sources/pr_89-91_skills-doc-frontmatter-injection.md
tags: [skill, wiki, lint, health-check, quality, audit, context-fork]
---

# Skill: wiki-lint

The wiki's structural integrity auditor. Diagnoses contradictions, stale pages, orphaned pages, missing cross-references, and coverage gaps — then suggests new questions and connections. Always runs after [[wiki-ingest]]; can also be run standalone as a periodic health check.

Source: `coderails/skills/wiki-lint/SKILL.md`
Invoked as: `coderails:wiki-lint`

## Trigger

```
'Use this skill to audit the quality and structural integrity of the project's LLM Wiki
— not to read or query it for information. Trigger when the user says "wiki-lint", wants
to lint the wiki, run a wiki health check, find contradictions or stale pages, detect
orphaned pages or dead links, discover missing cross-references, or identify coverage
gaps. The user's intent is diagnosing wiki health or improving wiki quality. Do not
trigger when the user wants to look up what the wiki says about a topic, query wiki
content, or read a wiki page.'
```

The trigger explicitly excludes query intent — lint is about wiki health, not wiki knowledge.

## Key phases

1. **Load schema** — reads `AGENTS.md` for vault path, git config, and `git.stale_days` (default 30). Branches into a worktree if `git.worktree: true`.
2. **Analyse** — reads all markdown files in the vault (excluding `.obsidian/`, `templates/`, `inbox/`). Parses YAML frontmatter and all `[[wiki-links]]`.
3. **Check** — six check categories (see below).
4. **Report** — counts per category, then per-finding details.
5. **Suggest** — new questions worth investigating, sources to find, connections worth creating as new pages.
6. **Update log** — appends a lint entry to `log.md`.
7. **Commit** — same worktree or direct pattern as [[wiki-ingest]].

## Six check categories

| Category | What it catches |
|---|---|
| Contradictions | Pages with `⚠️ CONTRADICTION` flags; claims superseded by newer sources |
| Stale pages | `last_updated` older than `git.stale_days` |
| Orphan pages | Zero inbound links (excludes `index.md`, `log.md`, `AGENTS.md`) |
| Missing concepts | Terms mentioned across multiple pages with no own page |
| Missing cross-references | Pages that name a concept but don't `[[link]]` it, when the page exists |
| Data gaps | Topics the wiki covers partially given project structure; inbox backlog |

## Failure modes encoded

- **Lint masquerading as query**: the trigger description explicitly excludes information-lookup intent. Lint is destructive in the sense of being a write operation (log update, potential fixes) — it should not fire when the user just wants to read.
- **Stale staleness threshold**: `git.stale_days` is configurable in AGENTS.md. The default of 30 days prevents lint from flagging every page as stale on a fast-moving project. If not set, defaults to 30. (verified: SKILL.md Step 0)
- **Orphan detection scope**: `index.md` and `log.md` are explicitly excluded from orphan checks — they're structural files, not content pages, and won't have inbound links by design.

## Key design decision: suggest after report

Step 4 (Suggest) comes after the report, not before. The pattern is: find what's broken, fix it, then surface what should exist next. This prevents lint from becoming a passive report card — it should always produce actionable next steps for ingest or investigation.

## Relationship to the wiki family

- Audits what [[wiki-ingest]] writes. AGENTS.md (ingest workflow, step 5) explicitly says "then run `/wiki-lint`" — lint is the mandatory post-ingest step.
- Is distinct from [[wiki-query]]: lint cares about wiki structure; query cares about wiki content.
- [[wiki-init]] creates the vault structure that lint uses as a baseline for orphan and gap detection.

## See also

- [[wiki-ingest]] — always run lint after ingest; the two are paired
- [[wiki-query]] — reads the same vault pages lint audits
- [[wiki-init]] — established the page-type structure lint checks against
- `/coderails:workflow` — lint runs as part of the workflow's post-merge wiki phase
