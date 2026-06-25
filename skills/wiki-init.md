---
title: "Skill: wiki-init"
type: skill
created: 2026-06-25
last_updated: 2026-06-25
sources:
  - skills/wiki-init/SKILL.md
tags: [skill, wiki, init, obsidian, bootstrap, knowledge-base]
---

# Skill: wiki-init

The one-time bootstrap skill for a project wiki. Creates the vault directory, wires up Obsidian and qmd, seeds initial pages, writes AGENTS.md and updates CLAUDE.md. Everything [[wiki-ingest]], [[wiki-lint]], and [[wiki-query]] depend on originates here.

Source: `coderails/skills/wiki-init/SKILL.md`
Invoked as: `coderails:wiki-init`

## Trigger

```
'Initialize an LLM Wiki for the current project — a persistent, compounding knowledge
base maintained by Claude and browsed in Obsidian. Use when the user wants to set up
a wiki, knowledge base, or second brain for a project. Also use when they mention
Karpathy's LLM Wiki pattern, AGENTS.md, or want to organize project knowledge beyond
CLAUDE.md. Triggers on "wiki init", "create wiki", "knowledge base", "set up obsidian
wiki", or explicit /wiki-init.'
```

## What this skill bootstraps

The coderails wiki implements the Karpathy LLM Wiki pattern: instead of re-deriving knowledge from raw code on every question, the LLM incrementally builds and maintains a persistent, interlinked markdown vault that compounds over time. The human browses it in Obsidian; the LLM does the bookkeeping. (verified: SKILL.md intro paragraph)

## Key phases

1. **Understand the project** — reads CLAUDE.md and explores the codebase to understand domain and what knowledge is worth pre-compiling.
2. **Propose and approve wiki structure** — proposes domain-adapted page types (NOT hardcoded to any template). Iterates with the user before creating anything. Backend vs frontend vs infrastructure projects get different directory structures.
3. **Create the vault** — `mkdir`, `git init`, then copies bundled Obsidian config files and the Marp slides plugin from the skill's own `assets/` directory. Critical: these are exact production copies — never hand-generated.
4. **Verify assets** — explicitly checks that `marp/main.js` exists and is >1MB and that `community-plugins.json` lists `"marp"` (not `"marp-slides"` — a different plugin). Any failure is fatal: stop and ask the user for help. Do not improvise.
5. **Create foundation files** — `index.md` (content catalog, read first on every query), `log.md` (append-only), and per-type template skeletons.
6. **Seed initial pages** — 5-10 pages covering architecturally important parts. Discusses each with the user as created; no silent auto-ingest.
7. **Create AGENTS.md** — writes the schema file to the *project directory* (not the vault). This is the single source of truth for wiki conventions; it's what the LLM reads at conversation start.
8. **Update CLAUDE.md** — adds a "Wiki Knowledge Base" section near the top telling Claude to read AGENTS.md at session start.
9. **Wire up tooling** — registers the vault with qmd (`qmd collection add`) and Obsidian (via JSON registry mutation + `open obsidian://...`). The user's only manual step: clicking "Trust author and enable plugins" on first Obsidian open.
10. **Commit and report** — commits everything; reports vault path, how to open in Obsidian, what to ingest first.

## Critical gotcha: AGENTS.md lives in the project, not the vault

`AGENTS.md` is written to the project directory (e.g. `/Users/harrison/Github/coderails/AGENTS.md`), not inside the wiki vault. The vault has no `schema.md`. This separation means the schema travels with the source repo, not the wiki. (verified: SKILL.md Step 6 and AGENTS.md preamble)

## Critical gotcha: never hand-generate the Marp plugin

`main.js` is ~3.6MB of compiled JavaScript. The SKILL.md is explicit: copy from bundled assets, verify size >1MB, stop on failure. Generating or improvising this file would silently break Obsidian's plugin loading. (verified: SKILL.md Step 3)

## Key design decision: domain-adaptive structure

The skill explicitly does NOT use hardcoded page types. A backend service gets `services/`, `entities/`, `concepts/`; a frontend app gets `components/`, `routes/`, `state/`. This means the wiki schema in AGENTS.md is a product of negotiation between the user and Claude during init, not a fixed template. (inferred: the examples in SKILL.md Step 2 are explicitly labelled as examples, not exhaustive)

## Relationship to the wiki family

This skill is the prerequisite for all three others. [[wiki-ingest]] fails (and tells the user to run `/wiki-init`) if AGENTS.md doesn't exist. [[wiki-lint]] and [[wiki-query]] similarly gate on AGENTS.md being present.

## See also

- [[wiki-ingest]] — the ongoing write operation; runs after every merged PR
- [[wiki-lint]] — audits structural integrity; should run after every ingest
- [[wiki-query]] — reads the vault this skill bootstrapped
- [[workflow]] — wiki-init is a one-time setup; workflow uses ingest/lint/query in its recurring cycle
