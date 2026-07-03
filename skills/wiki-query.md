---
title: "Skill: wiki-query"
type: skill
created: 2026-06-25
last_updated: 2026-07-03
sources:
  - skills/wiki-query/SKILL.md
  - sources/pr_89-91_skills-doc-frontmatter-injection.md
tags: [skill, wiki, query, search, knowledge-base, obsidian, marp, matplotlib, context-fork]
---

# Skill: wiki-query

The read operation for the wiki. Answers questions grounded in wiki knowledge, with citations. Also renders Marp slide decks and matplotlib charts from wiki content. Non-trivial answers get filed back into the wiki as investigation pages — so answering a hard question permanently enriches the vault.

Source: `coderails/skills/wiki-query/SKILL.md`
Invoked as: `coderails:wiki-query`

## Trigger

```
'Use when the user wants to search, query, or look up information in the project's LLM
Wiki. Triggers on any phrasing like "search wiki", "search the wiki", "query wiki",
"ask the wiki", "what does the wiki say", or requests to find project-specific answers
grounded in wiki content. Also triggers when the user wants to generate slides (Marp)
or charts (matplotlib) drawing on wiki knowledge. Do NOT trigger for general coding
questions, wiki maintenance tasks (adding, filing, ingesting, linting), or wiki
initialization.'
```

The trigger explicitly excludes maintenance operations — query is read-plus-file-back, not write-first.

## Key phases

1. **Load schema** — reads `AGENTS.md` for vault path and conventions. Missing AGENTS.md → tells user to run [[wiki-init]] first.
2. **Search the wiki** — reads `index.md` first (always), then drills into relevant pages. Uses `qmd search "<query>"` if qmd is configured for ranked results. Falls back to raw codebase only if the wiki doesn't cover the topic.
3. **Answer** — synthesizes from wiki pages with citations (`"According to [[page_name]]..."`). Output form is chosen to fit the question: prose, comparison table, Marp slide deck (`.md` with Marp frontmatter in `assets/`), or matplotlib chart (Python script → `.png` in `assets/`).
4. **File back** — when the answer reveals something non-obvious or reusable, creates an investigation page in `investigations/`, updates `index.md`, and appends to `log.md`.

## File-back: the compounding mechanism

The file-back step (Step 4) is the mechanism that makes the wiki compound. Hard questions that took effort to assemble become investigation pages the LLM (and human) can reference on the next query. This is the direct expression of the Karpathy LLM Wiki pattern's core claim: the LLM does the bookkeeping so knowledge doesn't evaporate between sessions. (inferred: this is the pattern's stated purpose, described in the wiki-init SKILL.md intro)

Not every answer gets filed back — only answers that "took real effort to assemble" or that are "non-obvious or reusable." The skill leaves this to judgment, not a mechanical threshold. (verified: SKILL.md Step 4)

## Answer format flexibility

Query is the only wiki skill that produces non-markdown outputs. Two noteworthy cases:

- **Marp slide deck**: writes a `.md` file to `$vault/assets/` with Marp frontmatter. Rendered live in Obsidian via the Marp plugin that [[wiki-init]] installs. Useful for presenting wiki knowledge as a deck.
- **Matplotlib chart**: writes and executes a Python script, saves `.png` to `assets/`. Useful for visualizing counts, timelines, or relationships.

These are output forms; they don't change the query logic. (verified: SKILL.md Step 3)

## Key design decision: index.md first, always

Step 2 reads `index.md` before any content page. This is a deliberate forcing function: `index.md` is the catalog, and reading it first prevents the LLM from reading every page to find a tangentially related one. The catalog is the navigation layer. (inferred: this is the stated rationale in AGENTS.md Query workflow)

## Failure modes encoded

- **Triggering on maintenance requests**: the trigger description explicitly excludes ingest, lint, and init intent. These operations have side effects (writes, commits); query should not invoke them inadvertently.
- **Skipping file-back**: the step makes no-file-back the exception, not the rule. If an answer required real effort, it should be written back so the next query is faster.
- **Falling back to code before wiki**: Step 2 mandates reading wiki pages before falling back to the raw codebase. The wiki should be the first source, not a supplement.

## Isolated subagent execution: `context: fork` (PR #90)

`SKILL.md` frontmatter gained `context: fork` (PR #90, merged 2026-07-03): wiki-query now runs
in an isolated forked subagent context, keeping bulky wiki reads (Step 2's index.md + drill-down
pass) out of the calling session's context window.

**`agent: Explore` was deliberately withheld**, same rationale as [[wiki-lint]]: this skill's
Step 4 file-back writes investigation pages and commits them — a read-only `Explore` agent
could not perform that write.

**Accepted tradeoff, not a bug:** forking this skill reduces the parent session's incidental
visibility into its autonomous file-back writes to the vault. Reviewed and accepted in PR #90
— record here so it isn't re-opened as a lint finding. (verified: [[pr_89-91_skills-doc-frontmatter-injection]], `git diff` on `skills/wiki-query/SKILL.md`)

## Relationship to the wiki family

- Reads what [[wiki-ingest]] wrote and what [[wiki-lint]] kept healthy.
- Depends on [[wiki-init]] having created `index.md` and the vault structure it navigates.
- File-back output (investigation pages) is itself ingested content — over time, query output and ingest output are indistinguishable in the vault.
- The four skills close the loop: init → ingest → lint → query → (discoveries prompt new ingest).

## See also

- [[wiki-ingest]] — writes the pages this skill reads
- [[wiki-lint]] — audits structural integrity of the vault; query relies on lint having kept links valid
- [[wiki-init]] — created the vault, installed Marp, registered with qmd
- `/coderails:workflow` — query is not a formal step in workflow, but investigation pages created by file-back often inform the next prep cycle
