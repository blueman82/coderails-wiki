---
title: "Skill: wiki-ingest"
type: skill
created: 2026-06-25
last_updated: 2026-06-25
sources:
  - skills/wiki-ingest/SKILL.md
tags: [skill, wiki, ingest, documentation, knowledge-base]
---

# Skill: wiki-ingest

The primary write operation for the coderails wiki. Translates a shipped change — a PR, a decision, a shipped feature — into permanent wiki entries. The human stays involved throughout; this is not a silent auto-documentation tool.

Source: `coderails/skills/wiki-ingest/SKILL.md`
Invoked as: `coderails:wiki-ingest`

## Trigger

```
'Use this skill when the user wants wiki pages created or updated to document a change —
a merged PR, shipped feature, or engineering decision. Trigger on any request to push
content into the wiki: "ingest this", "create wiki pages for this PR", "add to wiki",
"document this in the wiki", "capture this change", "file this in the wiki".'
```

The user always has a concrete artifact to record (PR number, description, decision).

## Key phases

1. **Load schema** — reads `AGENTS.md` from the project directory for vault path and git workflow config. If missing, stops and tells the user to run [[wiki-init]] first.
2. **Set up workspace** — if `git.worktree: true` (team repos), branches from `origin/main` into a timestamped worktree to prevent parallel session conflicts. Personal wikis (`git.worktree: false`) write directly to the vault.
3. **Read the source** — from an inbox file, a PR number (`gh pr view`), or a description. Multiple source types supported.
4. **Discuss key takeaways** — pauses to ask the user what to emphasise, what decisions matter, and what connects to existing pages. Non-negotiable step; no silent auto-ingest.
5. **Check existing coverage** — reads `index.md` to avoid duplicating what's already known. Curator principle: add only what's new and non-obvious.
6. **Write/update pages** — creates a `sources/` record, updates affected concept/skill/command/hook pages, creates new pages when warranted, updates `index.md` and appends to `log.md`.
7. **Commit** — direct commit (personal) or PR+squash+merge+pull (team).

## Failure modes encoded

- **Silent ingest**: the mandatory Step 3 discussion prevents Claude from auto-documenting without the human reviewing emphasis and connections.
- **Overwriting known facts**: the `index.md` check (Step 4) guards against writing redundant pages that diverge from established ones.
- **Skipping the source record**: the flow requires a `sources/` page first — this maintains traceability between wiki knowledge and the PR that introduced it.
- **Worktree skipped on team repos**: the `git.worktree` flag is explicit so parallel wiki sessions on shared repos don't conflict.

## Key design decision: worktree flag

The `git.worktree` bool in `AGENTS.md` is the only configuration difference between personal and team wiki setups. (verified: SKILL.md Step 1). All other steps are identical. This keeps the skill portable across solo and collaborative contexts without branching the logic significantly.

## Relationship to the wiki family

- Depends on [[wiki-init]] to have created the vault and `AGENTS.md` first.
- Produces the source records and page updates that [[wiki-lint]] later audits for structural integrity.
- [[wiki-query]] reads what this skill writes.
- The four skills form a closed loop: init → ingest → lint → query → (ingest again as knowledge gaps are discovered).

## Relationship to /workflow

Called explicitly as the final phase of `/coderails:workflow` (after `/merge`) to capture what was just shipped. Also usable standalone after any ad-hoc change the user wants recorded.

## See also

- [[wiki-init]] — prerequisite; creates the vault and AGENTS.md this skill reads
- [[wiki-lint]] — runs after ingest to audit what was written
- [[wiki-query]] — reads what ingest writes
- [[workflow]] — calls wiki-ingest as its closing phase
