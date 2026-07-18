---
title: "Skill: memory-consolidation"
type: skill
created: 2026-07-07
last_updated: 2026-07-18
sources:
  - sources/pr_36-41-33-53-65_verified-routines.md
  - sources/pr_240_lrp-last-marker-gate.md
tags: [skill, memory, routines, agentic-os, sub-project-2-of-5, artifact-gate]
---

# Skill: memory-consolidation

Dedupes overlapping memories, flags stale or contradicted ones, and refreshes `MEMORY.md`'s index in a project's persistent memory directory (`~/.claude/projects/<slug>/memory/`) — the same directory the memory system documented in the user's global `CLAUDE.md` writes to during normal sessions. Shipped as PR #33 of [[pr_36-41-33-53-65_verified-routines]].

Source: `coderails/skills/memory-consolidation/SKILL.md`

## Trigger phrases

"consolidate memory", "clean up memory", "memory consolidation", or when running as a scheduled routine `(verified, skills/memory-consolidation/SKILL.md` frontmatter)`.

## Relationship to /workflow

Not part of the `/workflow` command chain. It runs standalone on demand, or as one of the three shipped [[routines|verified routines]] (`memory-consolidation-weekly`, weekly cadence, `"profile": "read-only"`) — the skill needs no routines setup to function; the scheduling is additive, not a prerequisite.

## Key phases / steps

1. **Locate the memory directory** — `~/.claude/projects/<project-slug>/memory/`, where `<project-slug>` matches the sanitised working-directory path the skill is invoked from. Read `MEMORY.md` first, the index of every memory file.
2. **Read every memory file** the index points to — each has YAML frontmatter (`name`, `description`, `metadata.type`) and a body.
3. **Find consolidation candidates**, three categories:
   - **Overlapping** — two or more files describing the same fact/decision/state. Merge into the most recent/complete one; delete the superseded file(s).
   - **Stale** — a `project` or `feedback` memory contradicted by a newer memory, or referencing work explicitly marked complete elsewhere. Flagged for the user rather than silently deleted — `feedback`-type memories in particular represent a standing instruction and must not be dropped without the user noticing.
   - **Contradicted** — two memories asserting incompatible facts. Both flagged; the skill does not silently pick a winner.
4. **Apply merges, update `MEMORY.md`** — for each decided merge/deletion, update or remove the affected file(s), then update the index. `MEMORY.md` must never be left pointing at a file that no longer exists.
5. **Write the durable report artifact** — see below.

`(verified, skills/memory-consolidation/SKILL.md)`.

## The artifact-gate connection

Step 5 writes `~/.claude/coderails-dashboard/routines/memory-consolidation/report-{date}.md` **unconditionally** — even when Step 3 found nothing to merge or flag. This unconditional write is the property that lets the `memory-consolidation-weekly` routine gate on the report's existence rather than on `claude`'s exit code: a run that legitimately found nothing to change still produces the artifact, so "nothing changed" and "the skill never ran" stay distinguishable `(verified, skills/memory-consolidation/SKILL.md)`. This makes it the reference example the routines doc itself points to for "a routine whose own skill writes its artifact-gate report natively" (`docs/routines.md`'s See Also section).

The report has a fixed shape (summary counts, merges list, flagged-not-auto-resolved list, whether `MEMORY.md`'s index was updated) — a predictable structure a future `contains`/`json-field` predicate could check against. The shipped `memory-consolidation-weekly` routine config uses a plain `exists` predicate today, and — per the analysis below — that is the *correct*, not merely adequate, choice for this routine's artifact shape.

## Why `exists` is correct here, not just adequate (confirmed 2026-07-18, PR #240)

[[loop-retro-promotion]]'s companion routine, `loop-retro-promotion-weekly`, needed its `exists` predicate replaced with a stricter `last-marker` predicate because its artifact (`promotion-runs.log`) is an **append-only log accumulating many runs' worth of terminal markers** — a stale success line from an earlier run can make a later, genuinely-failed run's file still read "exists." PR #240's own scope included checking whether `memory-consolidation-weekly` shares that same defect class, and it does not: this skill's Step 5 writes a **date-keyed report file, one per run** (`report-{date}.md`), never an append to a shared multi-run log. Because each run's artifact is freshly named, there is no stale-prior-run content for `exists` to be confused by — file-absence genuinely is this routine's failure signal, and `exists` is the right predicate, not a placeholder for a future stricter one. The decline was deliberate and recorded, not an oversight.

## Failure modes encoded

- **Silent memory drift** — without a periodic consolidation pass, `MEMORY.md`'s index and the underlying files can accumulate overlapping or contradictory entries indefinitely; this skill exists to catch that before it compounds.
- **Silently dropping a standing instruction** — the skill explicitly refuses to auto-resolve `feedback`-type staleness; it flags rather than deletes, because a `feedback` memory encodes something the user told Claude to do, not just a fact.
- **A routine reporting false success** — the unconditional-report-write design is itself a failure-mode fix for the general routines problem this whole sub-project addresses: an exit-0 skill run that did nothing observable would otherwise look identical, from the artifact gate's perspective, to a skill that never ran at all.

## See also

- [[routines]] — the scheduling convention this skill runs under as `memory-consolidation-weekly`
- [[dashboard-runner]] — the executor that evaluates this routine's artifact gate after each scheduled run
- [[loop-retro-promotion]] — the sibling routine whose append-log artifact shape DID need the stricter `last-marker` predicate; this page's own "Why `exists` is correct here" section documents the contrast
- [[pr_36-41-33-53-65_verified-routines]] — the source record for this page
- [[pr_240_lrp-last-marker-gate]] — the source record that considered and declined applying `last-marker` to this routine
