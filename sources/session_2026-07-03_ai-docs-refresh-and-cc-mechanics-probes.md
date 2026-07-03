---
title: "Session 2026-07-03 — ai_docs snapshot refresh + Claude Code mechanics probes"
type: source
created: 2026-07-03
last_updated: 2026-07-03
sources:
  - sources/pr_89-91_skills-doc-frontmatter-injection.md
  - sources/pr_92_exec-bit-sweep.md
tags: [source, ai_docs, claude-code-mechanics, ordering-probe, subagent-skills, hook-lifecycle, agentic-loop, post-review]
---

# Session 2026-07-03 — ai_docs snapshot refresh + Claude Code mechanics probes

Not a coderails-repo PR — this is a local-only `~/.claude` change (commit
`dbb6822`) plus a set of empirical findings from a temporary probe skill (now
deleted). Recorded here because it resolves two open questions [[pr_89-91_skills-doc-frontmatter-injection]]
left outstanding: the stale ai_docs snapshot flag, and the inconclusive
`post-review.md` injection ordering probe.

## ai_docs snapshot refresh

`~/.claude/commands/ai_docs/cc_skills_docs.md` was rewritten from the live
[code.claude.com/docs/en/skills](https://code.claude.com/docs/en/skills) page
(commit `dbb6822`, local `~/.claude` repo — not coderails). The prior snapshot
was dated 2025-11-07 and predated the `paths` / `user-invocable` frontmatter
fields that [[pr_89-91_skills-doc-frontmatter-injection]] (#89) actually used —
[[pr_89-91_skills-doc-frontmatter-injection]] records that this staleness made
a reviewer on that same PR cluster confidently wrong about which fields existed.
This refresh closes that gap; no further action needed in coderails itself
(the snapshot lives outside this repo).

## Empirical findings on Claude Code mechanics

From a temporary probe skill (created and deleted within this session):

1. **`$ARGUMENTS` substitution happens *before* `!`cmd`` injection executes.**
   Verified via a rendered marker (`ORDERING-PROBE:XYZZY-42`) confirming
   argument substitution resolves first, then bash-injection commands run.
   This settles the question [[pr_89-91_skills-doc-frontmatter-injection]]
   (#91) left open when it deferred a "Current Git Status"-style injection for
   `commands/post-review.md` — the ordering concern is resolved, so **that
   change is now unblocked**, pending owner approval (not yet actioned in this
   session; no code change made to `commands/post-review.md` or [[merge]]'s
   Current Git Status pattern).

2. **Skill-frontmatter hooks (`PostToolUse`) fire in the invoking turn AND
   persist across later turn boundaries** — observed 7 firings over ~4 minutes
   in one session — **but are session-scoped**: a separate session's Bash
   calls triggered zero additional firings. This is a Claude Code platform
   behaviour, not something coderails configures; recorded for anyone
   designing skill-scoped hook behaviour in the future.

3. **Subagent skill enumeration is fixed at spawn time.** A skill created
   mid-flight is invisible to an already-running subagent's `Skill` tool, but
   is visible to (a) subagents spawned after the skill's creation, and (b) the
   main session itself via live change detection. This explains the
   inconclusive result [[pr_89-91_skills-doc-frontmatter-injection]] reported
   during the #91 probe ("a freshly created skill was not visible to a
   subagent's Skill tool during the probe") — it wasn't an ordering artifact,
   it was this fixed-at-spawn enumeration behaviour.

4. **DECISION: migrating `loop_state_guard`/`loop_stall_guard` into
   `skills/agentic-loop/SKILL.md` frontmatter hooks is feasible but REJECTED.**
   No documented or observed guarantee that skill-frontmatter hooks survive
   post-compaction session restarts, and these two guards are safety-critical
   ([[loop_state_guard]], [[loop_stall_guard]] — see
   [[spec-plan-progress-artifact-chain]]) and need deterministic scope. They
   remain `hooks/hooks.json`-wired Stop hooks, not skill-frontmatter hooks.
   This is the same "skill-scoped hooks migration DEFERRED" item
   [[pr_89-91_skills-doc-frontmatter-injection]] flagged, now resolved as a
   firm rejection for this specific pair rather than an open question.

## Wiki pages updated

- [[merge]] — added a note that the ordering-probe blocker on the
  `post-review.md` injection is resolved (unblocked, not yet implemented)
- [[post-review]] — cross-referenced the same unblocking note
- [[spec-plan-progress-artifact-chain]] — recorded the reject-decision on
  migrating the two loop guards into skill frontmatter

## Caveats / gotchas

- No code changed in the coderails repo as a result of these probes — this is
  documentation of empirical findings and one design decision, filed for
  future sessions to build on rather than re-derive.
- The `post-review.md` injection itself is **unblocked, not shipped** — a
  future PR is needed to actually add the `!`cmd`` block, mirroring
  [[merge]]'s "Current Git Status" pattern. Do not assume it already exists.
