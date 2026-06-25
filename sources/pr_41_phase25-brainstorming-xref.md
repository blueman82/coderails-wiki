---
title: "PR #41 — docs(agentic-loop): Phase 2.5 reuses brainstorming's design-quality lens by reference"
type: source
created: 2026-06-25
last_updated: 2026-06-25
sources: []
tags: [source, agentic-loop, brainstorming, phase-2.5, design-quality, reference-not-embody]
---

# PR #41 — Phase 2.5 brainstorming design-quality xref

<!-- Ingested by /wiki-ingest after merge. This is an immutable record of what changed. -->

## PR metadata

| Field | Value |
|---|---|
| PR number | #41 |
| Branch | `chore/phase25-brainstorming-xref` |
| Merged | 2026-06-25 |
| Merge SHA | `fd3e5e1` |
| JIRA ticket | — |

## Summary

A one-bullet additive change to `skills/agentic-loop/SKILL.md` Phase 2.5 (the design-fork resolver). The new bullet instructs the spawned design agent to apply `/coderails:brainstorming`'s design-quality discipline — weigh viable approaches against each other, cut speculative work (YAGNI), prefer shapes whose units stay small and independently testable (design-for-isolation) — **without** invoking brainstorming's human-approval gates.

The rationale encoded in the bullet: the loop can't run brainstorming itself because brainstorming's steps block on a human. Phase 2.5 reuses the design *thinking* by reference rather than invoking the gated skill. The SKILL.md text points readers to Phase 2.7 for the explicit statement: "a loop cannot brainstorm with itself."

This is the same reference-not-embody pattern as the Slimming v2 work (PR #39): the loop borrows principles from a vendored skill without being able to delegate to a skill that requires human interaction mid-loop.

## Files changed

- `skills/agentic-loop/SKILL.md` — Phase 2.5: one bullet added (line 163 post-merge)

## Wiki pages updated

- [[agentic-loop]] — Phase 2.5 table row updated; new "Phase 2.5 — brainstorming design-quality by reference" section added; source in frontmatter

## Caveats / gotchas

The durable point to preserve: brainstorming is human-gated *by construction* — its approval steps require a human. An autonomous loop that tries to invoke it would stall at the first gate. The solution is for Phase 2.5 to apply the *criteria* brainstorming uses (YAGNI, design-for-isolation, weigh approaches) without calling the skill. This is not a limitation to be fixed later; it is the intended design boundary between interactive skills and autonomous loop behaviour.

No frozen region was touched; the three agentic-loop hook suites stayed green. (verified: PR #41 description)
