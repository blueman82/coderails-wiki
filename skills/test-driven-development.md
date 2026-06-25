---
title: "Skill: test-driven-development"
type: skill
created: 2026-06-25
last_updated: 2026-06-25
sources:
  - sources/session_2026-06-25_agentic-loop-upgrade-arc.md
tags: [skill, tdd, test-driven, construction, red-green-refactor]
---

# Skill: test-driven-development

The construction discipline skill for coderails workers. Encodes red-green-refactor so workers build code test-first rather than code-then-test (or no test at all).

Source: `coderails/skills/test-driven-development/SKILL.md`
Companion: `coderails/skills/test-driven-development/testing-anti-patterns.md`
Invoked as: `coderails:test-driven-development`

## Trigger

```
'Use when about to implement or fix CODE that can carry a test — a feature, bugfix,
or refactor that adds or alters a function, method, or branch. Build test-first:
write the failing test, watch it fail for the right reason, then the minimal code
to pass, then refactor (red-green-refactor). Does NOT apply to docs, config, or
prose edits with no testable code — those verify by inspection, not a failing test.'
```

**Code-guard is load-bearing.** The description includes a "not for docs/config/prose" clause so the skill does not fire on non-code work (prose edits, config changes). The code-guard in the agentic-loop Phase 3/3a references mirrors this exactly: "if the change adds or alters a function, method, or branch that *can* carry a test."

## When this skill was added

Spec D (#16) — vendored from superpowers' red-green-refactor discipline into coderails' own namespace. The decision to vendor (rather than reference `superpowers:test-driven-development`) preserves coderails as a self-contained zip with zero cross-plugin dependencies.

## Key content

- **The Iron Law**: no production code without a failing test first. Code written before the test must be deleted (not kept "as reference" — that is testing after).
- **RED → verify-RED → GREEN → verify-GREEN → REFACTOR**. Both verify steps are mandatory and never skippable.
- **Rationalizations table**: "too simple to test," "test after," "already manually tested" — each maps to "STOP and start over."
- **Code-guard via description**: does not fire on docs/config/prose.
- **No model-selection guidance** — TDD is *how* to build, not *which model*. The `model: sonnet` rule in agentic-loop Phase 3/3a is not weakened.

## Relationship to agentic-loop

Phase 3 and Phase 3a (verified: SKILL.md) reference `coderails:test-driven-development` with the code-guard. The reference is near the top of the Phase 3a prompt-contract bullet list (Phase 9 lesson: scope-shaping instructions must register before the worker starts, not sit at the bottom where they get overlooked under load).

Placement matters: a construction-method instruction buried low in a prompt gets shortcut the same way Phase 9 war-story showed scope-suppression instructions do.

## Relationship to writing-plans

The `coderails:writing-plans` skill's per-task construction step references `coderails:test-driven-development` — a plan's tasks carry the construction method. This is the E→D tie: decomposition (E) points to construction (D).

## Failure modes encoded

- Writing code first, then writing a test that passes immediately (proves nothing — you never saw it fail).
- Keeping code written before the test "as reference" and adapting it (that is testing after).
- Calling a code change "mostly config" to self-exempt from TDD (the self-exemption the concrete "adds or alters a function, method, or branch" wording closes).
- Mocking without understanding dependencies (see companion testing-anti-patterns.md).

## See also

- [[writing-plans]] — references this skill in its per-task construction step (E→D tie)
- [[agentic-loop]] — Phase 3/3a wire the seam into worker task descriptions
- [[spec-plan-progress-artifact-chain]] — TDD is the construction layer of the full chain
