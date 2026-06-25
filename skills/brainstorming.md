---
title: "brainstorming"
type: skill
created: 2026-06-25
last_updated: 2026-06-25
sources: [sources/pr_19-30_self-containment-and-hardening.md]
tags: [skill, brainstorming, design, ideation, visual]
---

# brainstorming

Structured ideation skill — explores a problem space before committing to implementation, surfaces constraints and alternatives, and produces a Decision Ledger. Includes an optional visual companion (Node.js local server, blueprint/rail theme).

## Trigger phrases

Before any creative or architectural work: "brainstorm", "explore options", "what are the approaches". Use before `coderails:writing-plans` when the path isn't yet clear.

## Relationship to /workflow

Upstream of the workflow chain — runs before `/coderails:prep` when the solution space isn't yet defined. The skill ends with a decision that feeds into spec/plan authoring.

## Key phases / steps

1. Frame the problem: what are we actually solving?
2. Generate candidate approaches (at least 3) — don't evaluate yet.
3. Evaluate against constraints: complexity, reversibility, fit with existing patterns.
4. Record decisions and their rationale in the Decision Ledger.
5. Optionally launch the visual companion server for a structured session view.

## Decision Ledger

The coderails version adds a Decision Ledger panel (not present in the superpowers original): a running log of options considered and the reasoning behind what was selected. Surfaces the decision lineage for future review. (inferred — PR #25 body)

## Visual companion

An optional Node.js local server that renders a session view with a blueprint/rail visual theme. Requires Node.js; the skill operates fully without it. Start with `npm start` in the skill's scripts directory. (inferred — PR #25)

## Failure modes encoded

- Starting implementation before the problem is framed.
- Evaluating the first idea immediately and never generating alternatives.
- Making a design decision verbally without recording the reasoning (Decision Ledger addresses this).

## Source

`coderails/skills/brainstorming/SKILL.md`

## See also

[[writing-plans]] — produces the plan after brainstorming identifies the approach  
[[planning-sequence]] — the broader planning discipline (Pre-Parade → Premortem → Red Team)  
[[self-containment]] — added the blueprint theme + Decision Ledger as a coderails-specific twist
