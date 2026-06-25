---
title: "/coderails:disconfirm"
type: command
created: 2026-06-25
last_updated: 2026-06-25
sources: [commands/disconfirm.md]
tags: [command, discipline, introspection, adversarial, advisory]
---

# /coderails:disconfirm

Take the most recent recommendation, suggestion, or conclusion and argue against it. Find the strongest case it is wrong, incomplete, or harmful — then state honestly whether it still holds.

## Invocation

```
/coderails:disconfirm
```

No arguments. Operates on whatever Claude most recently recommended or concluded.

## What it does

1. Identifies the most recent recommendation, suggestion, or conclusion Claude made in this session.
2. Argues *against* it — not a balanced take, a steelman of the opposing case. Specifically:
   - What evidence would falsify the recommendation?
   - What edge cases break it?
   - What did the analysis miss or assume away?
   - Who would push back on it, and on what grounds?
3. Does not hedge. The opposition is given its strongest possible form.
4. After the disconfirmation, states honestly whether the original recommendation still holds, should be modified, or should be abandoned.

## When to invoke it

Use `/disconfirm` when a recommendation has consequential weight and you want adversarial pressure applied before committing:

- Before implementing an architectural decision that is hard to reverse.
- When a diagnosis feels confident but the evidence base is thin.
- After Claude has recommended an approach and you want to know what the case against it is.
- When you suspect Claude has anchored on its first interpretation and has not seriously considered alternatives.
- Before merging a plan that will run autonomously (e.g., as the input to [[agentic-loop]]).

The command is most valuable when the recommendation being challenged *feels* solid. If it already feels uncertain, `/assumptions` is the better first move.

## How it relates to the discipline loop

`/disconfirm` sits outside the mechanical enforcement layer entirely — there is no hook that runs adversarial analysis automatically. (verified: no corresponding hook exists in `hooks/hooks.json`)

It is a **deliberate user gesture**: an explicit request for Claude to challenge its own output. This is the sharpest of the four introspection commands because it does not just audit what was said — it argues the other side.

The discipline hooks ([[check_confidence_labels]], [[check_verify_loop]]) enforce that claims are labelled and that deferrals are explicit. They do not evaluate whether the claims are correct. `/disconfirm` is the manual lever for that evaluation.

The closest mechanical analogue is not a hook but the [[premortem]] skill, which applies backwards-failure-mode analysis to a plan. `/disconfirm` is the targeted, single-turn version: one recommendation, one adversarial turn. See [[discipline-loop]] and [[enforcement-model]].

## Config fields read

None. This command reads no `workflow.config.yaml` fields.

## Scripts invoked

None. This is a pure prose instruction to Claude — no shell scripts are called.

## Preconditions

There must be a recent recommendation in the session to argue against. If the session has produced no recommendation yet, there is nothing to disconfirm.

## See also

- [[discipline-loop]] — the discipline framework this command operates within
- [[enforcement-model]] — why this command is advisory, not enforced
- [[assumptions]] — surface hidden assumptions before making a recommendation
- [[notchecked]] — audit claims made without verification in recent responses
- [[verify]] — re-derive a specific claim from sources only
- [[premortem]] — backwards-failure-mode analysis on a plan; the skill-level counterpart
