---
title: "Skill: premortem"
type: skill
created: 2026-06-25
last_updated: 2026-06-25
sources: [skills/premortem/SKILL.md]
tags: [skill, premortem, adversarial-analysis, failure-modes, planning]
---

# Skill: premortem

Assumes a plan, decision, or approach has already failed, then reasons backwards to identify the most credible failure modes and causes — before commitment, while action is still possible.

Source: `coderails/skills/premortem/SKILL.md`
Invoked as: `coderails:premortem`

## Trigger phrases

```
"premortem this", "premortem <X>", "steelman the failure",
"what could go wrong with this plan"
```

The distinguishing signal is **backwards reasoning from an assumed bad outcome** — not forward-looking checklists, code review, or general architecture critique. If the user asks "what should I check before X", that's a checklist, not a premortem.

## The process

1. **Establish what's being premortemed** — extract from context if clear; ask one question if genuinely ambiguous.
2. **Reason adversarially** (in `<thinking>` block) across 7 lenses: assumption failures, execution failures, people failures, dependency failures, optimism bias, unknown unknowns, timing failures. Goal: the *most credible* failure modes, not a comprehensive list.
3. **Assess each failure** — likelihood (low/medium/high), impact, and a concrete mitigation. "Monitor closely" is explicitly called out as not a mitigation.
4. **Verdict** — one paragraph: sound plan with risks to address, structural problem, or fatal flaw?

Output is 2–5 failure modes. More is noise.

## Key design decisions encoded

**Specificity is the whole value.** Generic risk lists ("scope creep", "team alignment") without grounding in the specific plan are called out explicitly as a failure mode. Every failure mode must be specific to *this* plan.

**The verdict must be direct.** A premortem that finds catastrophe in everything is as useless as one that finds nothing. If the plan is mostly fine, say so.

**Can be applied to your own previous response.** The skill explicitly supports premortemed self-critique: "If you've premortemed your own answer, say so explicitly and update your recommendation if warranted."

## Relationship to planning-sequence

[[planning-sequence]] embeds the full premortem process inline as Stage 2, between Pre-Parade (Stage 1) and Red Team (Stage 3). `premortem` is the standalone skill when only backwards-failure analysis is requested. The difference: `premortem` alone; `planning-sequence` when all three passes are wanted. (verified: planning-sequence SKILL.md Stage 2 header: "Do not call the skill separately — execute its process inline.")

The two skills share the same 7 adversarial lenses and the same output structure (failure modes with likelihood/impact/mitigation + verdict).

## Failure modes encoded

- Generic risk lists not grounded in the specific plan.
- Polite softening — honest adversarial thinking is the point.
- Exhaustive lists over sharp ones — five specific failure modes beat fifteen shallow ones.
- Mitigations that are reassurances rather than actions.
- A verdict that hedges by listing everything as "worth considering."

## See also

- [[planning-sequence]] — embeds this skill's logic inline as Stage 2; use `planning-sequence` when all three passes (Pre-Parade + Premortem + Red Team) are wanted
- [[agentic-loop]] — Phase 2 pre-flight delegates premortem work to spawned agents
- [[improve-prompt]] — useful upstream when the plan being premortemed is ambiguously specified
