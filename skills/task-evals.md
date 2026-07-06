---
title: "Skill: task-evals"
type: skill
created: 2026-07-06
last_updated: 2026-07-06
sources:
  - sources/pr_1-4_task-evals-feature.md
tags: [skill, task-evals, anti-gaming, evals-json, tiering, verify-criteria, oracle-independence]
---

# Skill: task-evals

Turns any non-trivial task into a frozen, tiered set of independent, game-resistant success evals — before implementation starts, not after. The one place coderails breaks its own pattern of self-verification.

Source: `coderails/skills/task-evals/SKILL.md`
Design spec: `coderails/docs/coderails/specs/2026-07-03-task-evals-design.md`
Invoked as: `coderails:task-evals`

## Why this skill exists

Every other verification loop in coderails is self-verification, stacked with conflicts of interest: `writing-plans` verify-criteria are written by the same process that then implements against them; agentic-loop workers verify their own artifact; Phase 4b reviews code quality, not goal attainment; Phase 13 self-audits are explicitly unscored; `/merge`'s review-artifact gate proves review *happened*, not that the goal state was reached. The one exception was the hand-written public-readiness suite (E0–E10): negative controls, end-state assertions against fresh surfaces, independent GO/NO-GO gating. This skill generalises that pattern into a reusable discipline. (verified: SKILL.md "Why this skill exists")

## Trigger

```
'Use at task intake, before implementation starts, to turn any non-trivial task
into a frozen, tiered set of independent, game-resistant success evals — inside
an agentic loop or not. Trigger at loop scope (per-loop and per-work-unit), when
a plan is written, or directly on user request.'
```

Three invocation points (verified: SKILL.md "Invocation contract"): **agentic-loop Phase 2.7c** (loop scope), **writing-plans' final task** (pr scope), or **directly by the user**.

## The five anti-gaming rules

Generation requirements, not descriptions of an ideal — an eval failing any one is not valid:

1. **Freeze-before-build** — frozen (`frozen_at` + `frozen_sha`) before implementation starts; post-freeze edits are recorded `amendments` with reasons.
2. **Negative controls** — every scripted eval carries a command proving it *can* fail. An eval that never fails proves nothing.
3. **End-state surfaces** — assertions run against merged state, fresh clone, or deployed artifact — never working-tree self-reports.
4. **Oracle independence** — must not share an oracle with the implementation (same regex, same fixture, same test the implementation writes).
5. **Grader independence** — judgement evals graded by a fresh subagent that receives only `evals.json` + artifact references — never the implementation conversation. The orchestrator never hand-writes `result`.

**Gameability self-check** (run once per eval, immediately before freezing): *"Can the implementer pass this by (a) editing the eval, (b) asserting on the working tree, (c) self-reporting, or (d) reusing its own oracle? Any yes → rewrite."* No partial pass — a failing eval is rewritten, not annotated.

## Tier rules (self-exemption defence)

Concrete predicates, same design rationale as agentic-loop Phase 2.6's "what named thing does this remove?" test:

- **Tier 0 (exempt, justified)** — single work-unit AND no outward/irreversible surface AND an existing test/verify-criterion already covers the goal state. Still a written artifact (`tier_justification` required) — the gates accept a justified exemption, never an absence.
- **Tier 1 (standard)** — 3–5 end-state evals, ≥1 negative control, P0/P1 split.
- **Tier 2 (full suite)** — ≥3 work-units (the line agentic-loop Phase 2.7/3 already draw) OR any irreversible/outward surface (publish, deploy, migration, data deletion, external send).

## Schema (schema_version 1)

Scope is `pr` or `loop`. Each eval carries an ID, `priority` (`P0` blocks the gate, `P1` doesn't), `mode` (`scripted` or `agent-run`), `surface`, an `assert` one-liner, a `cmd` or verifier instruction, a `negative_control` (required for scripted), `status`, and `evidence`. GO requires **all P0 evals pass**; P1 failures don't block but must be listed unresolved. See [[task-evals-gate]] for the full JSON shape and how the two enforcement seams consume it.

## Where evals.json lives

- **Loop scope** → the loop-state dir beside `progress.json` (path from `agentic_loop_path.sh`), outside the repo, never committed.
- **PR scope** → working material only. The durable artifact is the SHA-bound PR comment `scripts/post_evals.sh` posts (marker `coderails-eval-summary`).

## Enforcement wiring

Two components consume this skill's output, both live from day one (owner choice, no advisory phase-in):

- **`/merge` gate** (pr scope) — [[merge]] reads the artifact `/coderails:post-evals` posts, fail-closed, additive to the existing review-artifact gate.
- **`loop_state_guard` gate** (loop scope) — [[loop_state_guard]] blocks `LOOP-STOP: complete` at ≥3 work-units without a passing loop-scope `evals.json`, fail-open when `work_units` is absent (legacy loop).

Full architecture: [[task-evals-gate]].

## Verifier agent contract (agent-run evals)

A fresh sonnet subagent grades judgement evals. Its prompt contains ONLY the `evals.json` content, artifact references, and the confidence-label contract — explicitly nothing else: not the implementation conversation, not the implementer's summary, not the orchestrator's opinion. Same "the author is the least able to see its own shims" principle as agentic-loop Phase 4b's clean-break gate. The assembly script (`post_evals.sh` for pr scope, a direct `evals.json` update for loop scope) folds results in and computes `result` itself — the verifier never writes `result` directly.

## Failure modes encoded

- An eval whose oracle is the implementation's own test or regex (oracle non-independence).
- A scripted check with no negative control — passes trivially, proves nothing.
- Working-tree self-reports standing in for merged/deployed-state assertions.
- The implementer grading its own judgement evals instead of a fresh, context-blind verifier.
- A tier-0 exemption with no written justification — silence is never an acceptable substitute for the artifact.
- Post-freeze eval edits made silently instead of recorded as `amendments`.

## Relationship to /workflow and agentic-loop

Not itself part of the `/workflow` chain directly — invoked as writing-plans' final task (pr scope) or agentic-loop's Phase 2.7c (loop scope). Sits at the same "gate before merge/completion" altitude as the review-artifact seam, but is independently sourced: an oracle-independent eval suite, not a code review.

## Source

`coderails/skills/task-evals/SKILL.md`
`coderails/docs/coderails/specs/2026-07-03-task-evals-design.md`

## See also

- [[task-evals-gate]] — design page: the dual-scope enforcement architecture (pr-scope PR-comment gate + loop-scope Stop-hook gate)
- [[post-evals]] — the `/coderails:post-evals` command that posts the pr-scope artifact
- [[merge]] — the pr-scope gate consumer
- [[loop_state_guard]] — the loop-scope gate consumer
- [[writing-plans]] — invokes this skill as the plan's mandatory final task
- [[agentic-loop]] — invokes this skill at Phase 2.7c (loop scope)
- [[review-artifact-seam]] — the predecessor truth-seam this design largely mirrors (marker SSOT, SHA-bound artifact, fail-closed gate)
- [[pr_1-4_task-evals-feature]] — the cluster source record
