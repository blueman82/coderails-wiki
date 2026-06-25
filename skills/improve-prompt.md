---
title: "Skill: improve-prompt"
type: skill
created: 2026-06-25
last_updated: 2026-06-25
sources: [skills/improve-prompt/SKILL.md]
tags: [skill, prompt-engineering, improve-prompt, diagnosis, rewrite]
---

# Skill: improve-prompt

Takes an underspecified prompt and rewrites it so execution produces the right result first time, with no rework. Structured around 7 diagnostic foundations and a context-first gap-resolution order.

Source: `coderails/skills/improve-prompt/SKILL.md`
Invoked as: `coderails:improve-prompt`

## Trigger phrases

```
/improve-prompt, "improve this prompt", "what's missing from this prompt",
"tighten this task description"
```

Also triggers when a prompt is vague, underspecified, or missing success criteria that would produce a poor result. "Just do it" skips improvement and executes immediately.

## Context-first, then gap — this ordering is load-bearing

Before diagnosing or asking, the skill consults sources in order:
1. Memory files for prior context on the topic
2. `CLAUDE.md` — stack, conventions, preferences
3. Open files / codebase
4. Current conversation

What context resolves, the skill does not ask about. This prevents unnecessary interruptions in sessions with rich context.

## The 7 foundations (diagnosis)

Each is marked ✓, ✗, or N/A. A `<thinking>` block reasons through them before producing output.

1. **Defines done** — output format, scope, completeness, success criteria
2. **Names assumptions** — stack, environment, audience, prior context
3. **Specifies constraints** — limits on what to touch or change
4. **Single scope** — one task, not several bundled
5. **States what's been tried** — prior attempts (N/A for greenfield)
6. **Execution override** — execute or discuss?
7. **Defines role** — persona if it sharpens output (N/A for mechanical tasks)

**Golden rule**: would a capable colleague, handed this prompt cold, produce the right result? Apply after diagnosis and after rewriting.

## Ask vs assume

- **Ask** when context doesn't resolve a gap and the risk of a wrong assumption is high (bounded choices → `AskUser` with multiple choice; open-ended → one terse question; one question at a time, most blocking first).
- **Assume** when context provides solid grounding and the wrong-assumption risk is low. State every assumption explicitly.

## Rewrite structure

Role first → positive framing → XML structure for mixed instruction/context/examples → explain the WHY for non-obvious constraints → include 3–5 `<example>` tags when format is specific → verify all 7 foundations pass before presenting.

## Relationship to agentic-loop

[[agentic-loop]] **Phase -1 explicitly delegates to this skill** (verified: SKILL.md phase table). Before planning begins, the orchestrator runs `coderails:improve-prompt` on the authorising prompt to close ambiguities before they propagate into the plan. This is why Phase -1 is "sharpen the authorising prompt" — a vague authorisation envelope causes scope drift across all subsequent phases.

## Failure modes encoded

- Asking before checking context — unnecessary interruptions when memory/CLAUDE.md/codebase already resolve the gap.
- Presenting a rewrite without verifying all 7 foundations pass — the failure was moved, not fixed.
- Bundling multiple questions — ask the most blocking gap first, resolve it, then re-evaluate.
- Output inside blockquotes in terminal renderers — the rewrite is always inside a triple-backtick fence.

## See also

- [[agentic-loop]] — Phase -1 delegates to this skill before planning begins
- [[planning-sequence]] — a planning technique often benefits from prompt improvement upstream
- [[writing-plans]] — plans built on well-specified prompts decompose more cleanly
