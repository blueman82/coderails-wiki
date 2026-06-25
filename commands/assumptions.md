---
title: "/coderails:assumptions"
type: command
created: 2026-06-25
last_updated: 2026-06-25
sources: [commands/assumptions.md]
tags: [command, discipline, introspection, confidence-labels, advisory]
---

# /coderails:assumptions

Pause all work and produce a full inventory of every assumption currently in play, each marked `(verified)` or `(inferred)`. Forces an explicit audit of what is known vs. what is guessed before proceeding.

## Invocation

```
/coderails:assumptions
```

No arguments. Can be called at any point in a session.

## What it does

1. Halts any in-progress work for the current turn. No code is written; no files are edited.
2. Lists every assumption Claude is currently making about the user's task, the codebase, the environment, or any session state.
3. Marks each assumption with a confidence label:
   - `(verified)` — directly observed this session via a tool result, file read, or explicit user statement
   - `(inferred)` — pattern-matched, recalled, or assumed from context without a direct confirmation
4. Formats the output as a table: Assumption | Source | Confidence.
5. Does not proceed with any other work in the same turn. The output is a pure inventory.

## When to invoke it

Use `/assumptions` at inflection points where hidden assumptions could send work in the wrong direction:

- Before starting a non-trivial implementation, to surface what is being assumed about the codebase layout, dependencies, or user intent.
- When something unexpected happens mid-task and it is not clear why — the wrong assumption is often the root cause.
- Before making an architectural decision, to make explicit what constraints are being treated as given.
- Whenever a plan seems to be going off the rails and the cause is not obvious.

## How it relates to the discipline loop

`/assumptions` is the **manual, advisory counterpart** to what [[check_confidence_labels]] enforces automatically. (inferred)

[[check_confidence_labels]] is a Stop hook: it fires on every response ≥200 chars and blocks (exit 2) if no `(verified)`/`(inferred)`/`(guess)` label appears at all. It enforces a **floor** — at least one label in a substantive response.

`/assumptions` does something different: it forces a **full audit** of everything Claude is currently assuming, not just what appears in the current response. It does not run automatically; Claude must choose to invoke it. This makes it advisory — it is ignored if Claude does not call it.

The two mechanisms compose: the hook catches unlabelled responses mechanically; the command gives the user a lever to demand a deeper accounting when the situation calls for it. See [[discipline-loop]] and [[enforcement-model]].

## Config fields read

None. This command reads no `workflow.config.yaml` fields.

## Scripts invoked

None. This is a pure prose instruction to Claude — no shell scripts are called.

## Preconditions

None. Can be invoked at any point in a session.

## See also

- [[discipline-loop]] — the composing design: confidence labels, DNV sections, advisory commands, mechanical hooks
- [[enforcement-model]] — why this command is advisory and [[check_confidence_labels]] is mechanical
- [[check_confidence_labels]] — the Stop hook that enforces the confidence-label floor automatically
- [[disconfirm]] — the adversarial complement: argues against Claude's most recent recommendation
- [[notchecked]] — audits claims made without verification in recent responses
- [[verify]] — re-derives a specific claim from sources only
