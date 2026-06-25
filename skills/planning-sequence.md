---
title: "Skill: planning-sequence"
type: skill
created: 2026-06-25
last_updated: 2026-06-25
sources: [skills/planning-sequence/SKILL.md]
tags: [skill, planning, premortem, red-team, pre-parade, anticipatory-analysis]
---

# Skill: planning-sequence

Runs three anticipatory planning techniques in fixed order — Pre-Parade, Premortem, Red Team — to surface structural gaps, execution failures, and adversarial weaknesses before commitment.

Source: `coderails/skills/planning-sequence/SKILL.md`
Invoked as: `coderails:planning-sequence`

## Trigger phrases

```
"run the planning sequence", "put this through the planning techniques",
"stress-test my plan", "Pre-Parade this", or proactively when a user is
about to commit to something high-stakes without adversarial planning
```

Also accepts an explicit subject: `/planning-sequence "migrate to Kubernetes"`.

## The three stages — always in this order

**Stage 1: Pre-Parade** (success-first). Imagine it worked brilliantly a year from now. Reverse-engineer 3–5 conditions that made it work. Flag each: in place / missing / uncertain. Surfaces what the team will assume is fine without checking.

**Stage 2: Premortem** (failure assumption). Assume it already failed. Work backwards across: assumption failures, execution failures, people failures, dependency failures, optimism bias, unknown unknowns, timing failures. Output 2–4 failure modes with likelihood/impact/mitigation. Ends with a verdict.

**Stage 3: Red Team** (adversarial attack). Identify the most credible adversary for this specific plan. Attack from their position with 2–3 specific vectors — what they do and what damage it causes. End with a challenge to the plan owner: what must change or be proved to survive?

**Closing synthesis**: 3–5 sentences. Where do all three stages agree? (Highest-confidence findings.) What must be addressed before committing?

## Critical implementation detail — Stage 2 runs inline

The skill explicitly states: "Run the premortem skill logic here. Do not call the skill separately — execute its process inline." (verified: SKILL.md Stage 2 header) This means [[premortem]]'s full process is embedded in Stage 2, not delegated. The two skills share logic but `planning-sequence` owns execution when called directly.

## What distinguishes the three stages

- Pre-Parade surfaces *conditions for success* — forensic reverse-engineering, not optimism.
- Premortem imagines *accidental failure* — organic breakdown from assumptions, execution, or dependencies.
- Red Team imagines *intentional opposition* — a motivated adversary exploiting the weakest points. It should "sting a little."

Running all three as variations of the same critique is the primary failure mode.

## Relationship to premortem

[[premortem]] is the standalone skill for backwards-reasoning from an assumed bad outcome. `planning-sequence` embeds its logic inline as Stage 2, extending it with Pre-Parade (Stage 1, which premortem omits) and Red Team (Stage 3, which premortem omits). The distinguishing signal for [[premortem]] alone: backwards reasoning from an assumed failure is explicitly requested. The distinguishing signal for `planning-sequence`: the user wants all three passes.

## Relationship to agentic-loop

[[agentic-loop]] Phase 2 delegates planning and premortem work to spawned sonnet agents during pre-flight. `planning-sequence` is one of the natural skills those agents would invoke for high-stakes decisions within a loop run.

## Failure modes encoded

- Running all three stages as the same critique with different labels — they must surface *different* things.
- Softening the Red Team to avoid discomfort — its value is exactly the discomfort.
- Generic risk lists ("scope creep", "team alignment") not grounded in the specific plan.
- Synthesis that just restates the three stages — it must add something by combining them.
- Pre-Parade conditions that are vague ("good execution") rather than checkable ("Alice owns the API contract, confirmed").

## See also

- [[premortem]] — Stage 2 runs this logic inline; standalone when only backwards-failure analysis is needed
- [[agentic-loop]] — Phase 2 pre-flight spawns agents that may invoke this
- [[improve-prompt]] — a prompt often benefits from improvement before running the planning sequence
- [[writing-plans]] — planning sequence is a natural upstream step before formal plan decomposition
