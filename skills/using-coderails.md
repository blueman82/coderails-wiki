---
title: "using-coderails"
type: skill
created: 2026-06-25
last_updated: 2026-06-25
sources: [sources/pr_19-30_self-containment-and-hardening.md]
tags: [skill, bootstrap, orientation, self-containment]
---

# using-coderails

Session orientation skill — establishes the coderails workflow context at the start of a session so Claude knows how the plugin's commands, skills, and hooks compose. Replaces the superpowers `using-superpowers` equivalent.

## Trigger phrases

Triggered by the [[inject_bootstrap]] SessionStart hook on every session. Also user-invokable directly.

## Relationship to /workflow

Above the workflow chain. This skill runs before any other command to orient the session. Other skills and commands assume this context is present.

## Key phases / steps

1. Reads the coderails plugin structure (commands, skills, hooks).
2. Establishes awareness of the full workflow chain (`prep → push → review → merge → wiki`).
3. Surfaces the key enforcement model distinction (hooks block, commands advise).
4. Notes the discipline loop and its hooks.

## Failure modes encoded

- Starting a session unaware of coderails conventions and reinventing workflow steps.
- Invoking superpowers skills that are no longer the coderails entry point.

## Source

`coderails/skills/using-coderails/SKILL.md`

## See also

[[inject_bootstrap]] — the SessionStart hook that auto-invokes this skill  
[[self-containment]] — why this replaced using-superpowers  
[[enforcement-model]]  
[[workflow]]
