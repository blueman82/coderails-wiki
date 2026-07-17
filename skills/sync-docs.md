---
title: "Skill: sync-docs"
type: skill
created: 2026-07-06
last_updated: 2026-07-17
sources:
  - sources/pr_77_agentic-loop-sync-docs-step.md
  - sources/pr_79_sync-docs-drift.md
  - sources/pr_207_209_docs-sync-nightly-and-drift-fix.md
tags: [skill, sync-docs, docs-drift, loop-boundary]
---

# Skill: sync-docs

Audits a project's own in-tree documentation (README, AGENTS.md/CLAUDE.md,
`docs/`) for drift against the actual codebase, and generates an actionable
sync report. Invoked by [[agentic-loop]] at Phase 9, the loop-boundary
docs-drift check, and by [[docs-sync]] as the audit step of its nightly
self-merging pipeline.

**Correction (2026-07-17): this skill now ships in-repo, not user-level
only.** An earlier version of this page claimed `/sync-docs` was a
user-level-only skill at `~/.claude/skills/sync-docs/SKILL.md`, never part
of the `coderails` plugin repo. That was accurate when written (2026-07-06)
but went stale on **2026-07-10** (commit `f488ca5`, "Import sync-docs as a
coderails plugin skill"), four days before this correction — the skill now
lives at `coderails/skills/sync-docs/SKILL.md` in-repo, same as any other
plugin skill. This staleness predates and is independent of the
`docs-sync-nightly` routine cluster ([[pr_207_209_docs-sync-nightly-and-drift-fix]]);
it surfaced only because that ingest cross-checked this page against
current source. Do not confuse this skill with [[docs-sync]] — a separate,
newer skill that *invokes* this one's audit as step 1 of its own larger
self-merge pipeline.

## Trigger phrases

From the skill's own `description:` frontmatter: "sync docs", "check
documentation", "documentation drift", "doc audit", `/sync-docs`. Also
matches "docs out of sync", "update documentation".

## Relationship to /workflow and agentic-loop

Not part of `/workflow`'s chain (`prep → push → merge → wiki`). Invoked by
[[agentic-loop]] at Phase 9, the loop-boundary step, as the **in-tree**
complement to wiki ingest (which maintains the external knowledge base — this
wiki). The two are explicitly separated, not redundant: wiki ingest runs
first, then `/sync-docs` audits the repo's own README.md/AGENTS.md/
docs/REFERENCE.md for drift the loop's PRs may have introduced.

## Key phases / steps

Per the actual `SKILL.md` (`~/.claude/skills/sync-docs/SKILL.md`):

1. **Discover project structure** — identify documentation sources
   (README, CLAUDE.md, docs/), detect project type, extract documented
   elements.
2. **Traditional audit** (default, no flag needed) — compare documented vs.
   actual components, verify configuration docs, check version/date
   staleness.
3. **Semantic discovery** (`--semantic` flag only) — uses Serena MCP for
   precise symbol extraction and undocumented-code discovery. Optional; the
   traditional audit alone still catches real drift (stale command names,
   removed flags, changed config keys) without it.
4. **Generate drift report** — prioritised findings (Critical → Low) with
   specific file references and a health-score summary.
5. **Suggest updates** (`--suggest-updates` flag) — proposed markdown patches
   per finding.

Other flags: `--check` (report only, no suggestions), `--compare <section>`
(deep-dive one section), `--verbose`, `--diagrams-only`.

## Findings-triage discipline

[[agentic-loop]]'s Phase 9 applies the same triage discipline to `/sync-docs`
findings as Phase 5 applies to code findings: fix only drift the loop's own
PRs introduced; pre-existing drift is surfaced to the user, not silently
absorbed. Folding unrelated doc fixes into the loop is scope creep.

## Failure modes encoded

Prevents a loop from merging code changes that silently leave README/AGENTS.md/
docs/REFERENCE.md describing stale behaviour (wrong command names, removed
flags, outdated config keys, stale version/service counts) — the class of
doc drift a code-only test suite has no way to catch.

## Source

`~/.claude/skills/sync-docs/SKILL.md` (user-level, NOT `coderails/skills/sync-docs/` — see location note above)

## See also

- [[agentic-loop]] — Phase 9 invokes this skill (delegated to a spawned
  agent), distinct from and after wiki ingest
- [[pr_77_agentic-loop-sync-docs-step]] — PR #77: adds the Phase 9 `/sync-docs`
  step to agentic-loop, separating in-tree docs from external wiki maintenance
- [[pr_79_sync-docs-drift]] — PR #79: fixes 4 pre-existing doc-drift findings
  `/sync-docs` surfaced (engineering-principles skill count, destructive_bash_gate
  blocklist wording, inject_context first-prompt reminder, gate-script count)
