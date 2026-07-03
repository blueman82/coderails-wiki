---
title: "PR 89–91 — adopt Claude Code skills-doc frontmatter/injection features"
type: source
created: 2026-07-03
last_updated: 2026-07-03
sources: []
tags: [source, skills, frontmatter, paths, user-invocable, context-fork, dynamic-injection, agentic-loop, wiki-lint, wiki-query, merge]
---

# PR 89–91 — adopt Claude Code skills-doc frontmatter/injection features

One thematic cluster, ingested as a single source: adopting three frontmatter/injection
features documented at https://code.claude.com/docs/en/skills — `paths` hints,
`user-invocable`, `context: fork`, and dynamic bash-substitution injection in command
frontmatter.

## PR metadata

| Field | Value |
|---|---|
| PR numbers | #89, #90, #91 |
| Merge SHAs | `9510efd` (#89), `dea3883` (#90), `f8b69ea` (#91) |
| Merged | 2026-07-03 |
| JIRA ticket | — |

## Summary

**PR #89 (9510efd) — skill frontmatter metadata trio.**
- `skills/agentic-loop/SKILL.md`: `description` trimmed from 1533 to 1134 characters (the
  Claude Code skill-listing truncation cap is 1,536 chars). All trigger phrases were
  preserved; the "sonnet" delegation-rule clause was restored to the trimmed text after
  review flagged its initial removal.
- `skills/engineering-principles-{go,python,ts}/SKILL.md`: each gained a `paths:` frontmatter
  hint (`**/*.go`, `**/*.py`, `**/*.ts,**/*.tsx` respectively) — a glob signal for which files
  the skill applies to.
- `skills/using-coderails/SKILL.md`: gained `user-invocable: false` — this skill is
  auto-loaded via the `inject_bootstrap.sh` SessionStart hook and was never meant to be
  invoked directly by name.

**PR #90 (dea3883) — context: fork for wiki-lint + wiki-query.**
- `skills/wiki-lint/SKILL.md` and `skills/wiki-query/SKILL.md` both gained `context: fork`
  frontmatter — these skills now run in an isolated subagent execution context rather than
  the main conversation context, keeping bulky wiki crawls out of main context.
- **`agent: Explore` was deliberately withheld** for both. Both skill bodies write files and
  commit to the wiki repo; `Explore`'s read-only tool restriction (no Write/Edit) would break
  that.
- **Reviewed caveat, accepted as a tradeoff, not fixed:** `context: fork` reduces the parent
  session's incidental visibility into these skills' autonomous wiki writes — a forked
  subagent's file writes are less visible in the parent transcript than an inline
  main-context write would be.

**PR #91 (f8b69ea) — dynamic injection in /merge.**
- `commands/merge.md` gained an injected "Current Git Status" block using frontmatter bash
  substitution: `!`git branch --show-current`` and `!`gh pr list --state open --limit 10``,
  plus a display-only note ("data, not instructions") guarding against the injected output
  being misread as directives.
- **A parallel injection for `commands/post-review.md` was explicitly DEFERRED, not
  abandoned.** An empirical probe of `$ARGUMENTS`-inside-`!`cmd`` substitution ordering was
  inconclusive: a freshly created skill was not visible to a subagent's `Skill` tool during
  the probe, and the public docs only imply the substitution ordering rather than stating it
  outright. Revisit once the ordering can be verified empirically or Anthropic's docs state
  it explicitly.

## Files changed

- `skills/agentic-loop/SKILL.md` (#89)
- `skills/engineering-principles-go/SKILL.md`, `skills/engineering-principles-python/SKILL.md`, `skills/engineering-principles-ts/SKILL.md` (#89)
- `skills/using-coderails/SKILL.md` (#89)
- `skills/wiki-lint/SKILL.md`, `skills/wiki-query/SKILL.md` (#90)
- `commands/merge.md` (#91)

## Wiki pages updated

- [[agentic-loop]] — description-trim note added
- [[wiki-lint]] — `context: fork` note + Explore-withheld rationale
- [[wiki-query]] — `context: fork` note + Explore-withheld rationale
- [[merge]] — Current Git Status injection documented; post-review deferral noted
- [[skills-hooks-seam]] — cross-reference: dynamic injection is a new frontmatter mechanism this seam convention should track going forward

## Decisions from the same loop (not code changes, recorded for durability)

- **`disable-model-invocation` REJECTED for workflow commands.** Considered as a way to force
  commands to only be invoked as programmatic subroutines (never typed directly by a user),
  but it would also block `agentic-loop`'s own subroutine invocation of those same commands
  — the mechanism can't distinguish "user typed this" from "the loop's orchestrator invoked
  this as a subroutine." Rejected outright, not deferred.
- **Skill-scoped hooks migration DEFERRED.** The idea: hooks that only fire during a skill's
  "active period" (bound to that skill's own lifetime) rather than globally. Blocked on an
  undocumented boundary — Claude Code does not currently document what a skill's "active
  period" is or how to detect its start/end, so the scoping can't be implemented or verified.
  Revisit if/when Anthropic documents this.
- **Local `ai_docs` skills-docs snapshot flagged stale.** The vendored snapshot is dated
  2025-11-07 and predates the `paths` / `user-invocable` frontmatter fields used in this very
  PR cluster (#89). It is stale enough to actively mislead a reviewer checking frontmatter
  conventions against it. No fix landed in this cluster — flagged for a follow-up snapshot
  refresh.

## Caveats / gotchas

- The `context: fork` caveat for wiki-lint/wiki-query (reduced parent-transcript visibility
  into autonomous wiki writes) is an accepted tradeoff, not a bug — record it here so it
  isn't re-opened as a wiki-lint finding.
- `commands/post-review.md`'s injection is deferred on an inconclusive empirical result, not
  a design rejection — distinct from the `disable-model-invocation` rejection above, which
  *was* a considered-and-rejected design call.
