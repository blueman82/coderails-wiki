---
title: "PR 50 — planning-sequence gate in writing-plans flow"
type: source
created: 2026-06-26
last_updated: 2026-06-26
sources: []
tags: [source, writing-plans, planning-sequence, workflow, gate, docs-reorg]
---

# PR 50 — planning-sequence gate in writing-plans flow

<!-- Ingested by /wiki-ingest after merge. Immutable record of what changed. -->

## PR metadata

| Field | Value |
|---|---|
| PR number | #50 |
| Branch | `feature/writing-plans-planning-sequence-gate` |
| Merged | 2026-06-26 |
| Merge SHA | `03b5b11` (merge commit) |
| JIRA ticket | — |

## Summary

Inserts a **mandatory `/coderails:planning-sequence` step into the [[writing-plans]] flow**, between the self-review gate and the hand-off to implementation. A new `## Stress-test before implementation (required)` section in `skills/writing-plans/SKILL.md` makes the order: write the plan → self-review gate → `/coderails:planning-sequence` (Pre-Parade → Premortem → Red Team) → fold findings back into the plan inline → only then hand off to `coderails:subagent-driven-development`/`coderails:executing-plans`. `docs/REFERENCE.md`'s `writing-plans` entry gains a matching **"Next step (required)"** line so the doc and the skill don't drift. (verified — both files on `main` post-merge)

The PR also **relocated** the engineering-principles spec/plan into `docs/coderails/{specs,plans}/`, conforming file locations to the structure REFERENCE.md already documented (5 mentions; a recorded 2026-06-25 reorg decision). REFERENCE.md needed no edit — it was already correct; only the files had drifted to `docs/` root. (verified)

## Files changed

- `skills/writing-plans/SKILL.md` (+8) — the gate section (the behavioural change)
- `docs/REFERENCE.md` (+2) — matching "Next step (required)" line in the writing-plans entry
- `docs/coderails/specs/2026-06-26-engineering-principles-design.md` (new) — design record
- `docs/coderails/plans/engineering-principles.md` (new) — implementation plan (13 tasks)
- `docs/coderails/plans/skill-vendoring.md` (relocated rename)

## Wiki pages updated

- [[writing-plans]] — added the mandatory planning-sequence gate to the documented flow
- [[planning-sequence]] — reconciled its relationship to writing-plans (now a **required downstream** gate, not only an optional upstream step)

## Caveats / gotchas

- **Relationship flip worth noting:** the wiki previously framed `planning-sequence` as a *natural upstream* step *before* plan decomposition. The gate makes it a *required downstream* step *after* the plan is written. Both readings are valid (you can stress-test an idea before planning AND stress-test the written plan before building) — but the **mandatory** instance is the post-plan one. (verified — SKILL.md gate text)
- **Mixed-concern PR:** the planning-sequence gate (the PR's namesake) was bundled with the engineering-principles **design docs**, which describe a *not-yet-implemented* feature — the `strictcode → engineering-principles` skill vendoring. No `skills/engineering-principles*` dirs exist yet; config still uses `strictcode_*`. The spec/plan are design records ahead of implementation. Per the very gate this PR lands, that implementation should itself pass `/coderails:planning-sequence` before being built. (verified — no engineering-principles skill dirs on `main`)
- **Pre-existing dead link** rode along in the relocated `docs/coderails/plans/skill-vendoring.md`: it cites `docs/superpowers/specs/2026-06-24-d-construction-seam-design.md`, a path removed in PR #44. Carried unchanged by the 100%-similarity rename, flagged in review, not fixed here. (verified — review)
- This is a **skill-flow change, not a hook.** It is advisory: `writing-plans` instructs the planning-sequence step, but nothing mechanically blocks an implementation that skips it. Consistent with the [[enforcement-model]] (commands/skills advise; hooks enforce).
