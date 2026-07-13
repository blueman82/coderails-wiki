---
title: "Skill: task-evals"
type: skill
created: 2026-07-06
last_updated: 2026-07-13
sources:
  - sources/pr_1-4_task-evals-feature.md
  - sources/pr_7-10_task-evals-followups.md
  - sources/pr_15-17_loop-hardening-registration-eval-freeze-ledger-dry.md
  - sources/pr_138_remove-specs-plans-tracking.md
  - sources/pr_144-149_agentic-loop-hardening-from-loop-engineering.md
tags: [skill, task-evals, anti-gaming, evals-json, tiering, verify-criteria, oracle-independence, tier-justification, eval-freeze, strongest-surface, grade-loop]
---

# Skill: task-evals

Turns any non-trivial task into a frozen, tiered set of independent, game-resistant success evals — before implementation starts, not after. The one place coderails breaks its own pattern of self-verification.

Source: `coderails/skills/task-evals/SKILL.md`
Invoked as: `coderails:task-evals`

> The design spec formerly cited here (`docs/coderails/specs/2026-07-03-task-evals-design.md`) no longer exists — removed from repo tracking by [[pr_138_remove-specs-plans-tracking]] (2026-07-11). This page's own content is now the durable record.

## Why this skill exists

Every other verification loop in coderails is self-verification, stacked with conflicts of interest: `writing-plans` verify-criteria are written by the same process that then implements against them; agentic-loop workers verify their own artifact; Phase 4b reviews code quality, not goal attainment; Phase 13 self-audits are explicitly unscored; `/merge`'s review-artifact gate proves review *happened*, not that the goal state was reached. The one exception was the hand-written public-readiness suite (E0–E10): negative controls, end-state assertions against fresh surfaces, independent GO/NO-GO gating. This skill generalises that pattern into a reusable discipline. (verified: SKILL.md "Why this skill exists")

## Prerequisite: gather context before generating evals

Before drafting a single eval: wiki first, codebase only where the wiki doesn't cover it — a wiki read is cheaper and already states the invariants and constraints the goal state must respect, prior decisions, and known gotchas that a codebase read would otherwise have to re-derive. If the project has no wiki (`config.wiki_path` is null), the context read is codebase-only. Dispatched to a sonnet agent rather than done inline, the same delegation pattern `agentic-loop` Phase 2 uses for its own pre-flight checks — keeps the orchestrator's context clean and makes the read a discrete, reportable step. The agent returns distilled findings, not raw file dumps. Inside an agentic loop, the orchestrator's Phase 2 pre-flight wiki read already satisfies this prerequisite — reuse its findings rather than re-reading per invocation. This is a context-gathering step, not a verification step — never conflate it with the gameability self-check or the six anti-gaming rules below. Tightened to this wording by [[pr_7-10_task-evals-followups|PR #7]] (originally added by [[pr_1-4_task-evals-feature|PR #1]]).

## Trigger

```
'Use at task intake, before implementation starts, to turn any non-trivial task
into a frozen, tiered set of independent, game-resistant success evals — inside
an agentic loop or not. Trigger at loop scope (per-loop and per-work-unit), when
a plan is written, or directly on user request.'
```

Three invocation points (verified: SKILL.md "Invocation contract"): **agentic-loop Phase 2.7c** (loop scope), **writing-plans' freeze-after-stress-test step** (pr scope — corrected wording, [[pr_15-17_loop-hardening-registration-eval-freeze-ledger-dry|PR #15]]; previously miswritten as "writing-plans' final task," which let freeze-before-build and grade-and-post flatten into one end-stage task and produced an unfrozen 17-unit execution — see [[writing-plans]] for the fix), or **directly by the user**.

## The six anti-gaming rules

Generation requirements, not descriptions of an ideal — an eval failing any one is not valid:

1. **Freeze-before-build** — frozen (`frozen_at` + `frozen_sha`) before implementation starts; post-freeze edits are recorded `amendments` with reasons.
2. **Negative controls** — every scripted eval carries a command proving it *can* fail. An eval that never fails proves nothing.
3. **End-state surfaces** — assertions run against merged state, fresh clone, deployed artifact, or a locally built artifact driven directly (rule 6's pr-scope `artifact-path`; admitted by PR #154, 2026-07-13) — never working-tree self-reports: driving a locally run artifact observes end-state behaviour, a self-report just quotes the diff.
4. **Oracle independence** — must not share an oracle with the implementation (same regex, same fixture, same test the implementation writes).
5. **Grader independence** — judgement evals graded by a fresh subagent that receives only `evals.json` + artifact references — never the implementation conversation. The orchestrator never hand-writes `result`. Extended by PR #153 (2026-07-13): an eval amended after a grader verdict returns to a fresh grader; the orchestrator never writes a per-eval `status` that flips an existing verdict (structurally backstopped by `grade-loop`'s regrade-on-amendment refusal — see [[task-evals-gate]]).
6. **Strongest surface** ([[pr_144-149_agentic-loop-hardening-from-loop-engineering|PR #145]], 2026-07-12) — if the task's goal state names something a human sees or interacts with (a UI, CLI output, a rendered artifact, a served endpoint), at least one P0 eval must exercise that surface directly — drive the running artifact (browser, CLI invocation, HTTP request), never only code-greps of merged state. At pr scope pre-merge this means the locally-run artifact (surface: `artifact-path` — a file path, a local CLI invocation, or a locally served endpoint; the same endpoint on the live post-merge instance is `deployed`; named by PR #154, 2026-07-13); at loop scope, the deployed surface. Writer-side only: no script can detect "user-facing," so this is enforced at generation and by review, not by either enforcement gate. Exemplar named in the skill: the run-output noise-strip loop (PRs #139-141), where merged-state greps passed while the live streaming window still leaked injected preamble text — only an in-browser eval across the streaming lifecycle caught it.

**Gameability self-check** (run once per eval, immediately before freezing): *"Can the implementer pass this by (a) editing the eval, (b) asserting on the working tree, (c) self-reporting, or (d) reusing its own oracle? Any yes → rewrite."* No partial pass — a failing eval is rewritten, not annotated.

## Tier rules (self-exemption defence)

Concrete predicates, same design rationale as agentic-loop Phase 2.6's "what named thing does this remove?" test:

- **Tier 0 (exempt, justified)** — single work-unit AND no outward/irreversible surface AND an existing test/verify-criterion already covers the goal state. Still a written artifact (`tier_justification` required) — the gates accept a justified exemption, never an absence. PR #154 (2026-07-13) reconciled this predicate with rule 6: anything rule 6 names (a UI, CLI output, a rendered artifact, a served endpoint) **is** an outward surface here, so a user-facing change is minimum tier 1 with rule 6's ≥1 P0 drive-the-artifact eval. This widens only the tier-0 test — tier 2's outward predicate stays scoped to its own parenthetical list, so user-facing alone does not escalate past tier 1.
- **Tier 1 (standard)** — 3–5 end-state evals, ≥1 negative control, P0/P1 split.
- **Tier 2 (full suite)** — ≥3 work-units (the line agentic-loop Phase 2.7/3 already draw) OR any irreversible/outward surface (publish, deploy, migration, data deletion, external send).

**`tier_justification` is required at every tier, not just tier 0** (owner directive, [[pr_7-10_task-evals-followups|PR #10]]) — tier 0 justifies the exemption itself; tier 1/2 must state which tier predicate fired. Both the writer (`post_evals::validate_structure` check 2, pr scope) and the loop-scope reader (`als_read_loop_evals_result`, which gained a distinct `UNJUSTIFIED` result) enforce this; a blank or whitespace-only justification blocks regardless of grading outcome, including retroactively against pre-existing GO artifacts written before the check existed. See [[task-evals-gate]] for the mechanism.

## Schema (schema_version 1)

Scope is `pr` or `loop`. Each eval carries an ID, `priority` (`P0` blocks the gate, `P1` doesn't), `mode` (`scripted` or `agent-run`), `surface`, an `assert` one-liner, a `cmd` or verifier instruction, a `negative_control` (required for scripted), `status`, and `evidence`. GO requires **all P0 evals pass**; P1 failures don't block but must be listed unresolved. See [[task-evals-gate]] for the full JSON shape and how the two enforcement seams consume it.

## Where evals.json lives

- **Loop scope** → the loop-state dir beside `progress.json` (path from `agentic_loop_path.sh`), outside the repo, never committed.
- **PR scope** → working material only. The durable artifact is the SHA-bound PR comment `scripts/post_evals.sh` posts (marker `coderails-eval-summary`).

## Enforcement wiring

Two components consume this skill's output, both live from day one (owner choice, no advisory phase-in):

- **`/merge` gate** (pr scope) — [[merge]] reads the artifact `/coderails:post-evals` posts, fail-closed, additive to the existing review-artifact gate. The evals themselves are frozen at the writing-plans freeze-after-stress-test step, well before merge; `/coderails:post-evals` at the plan's final task only grades and posts the already-frozen artifact (see [[writing-plans]]).
- **`loop_state_guard` gate** (loop scope) — [[loop_state_guard]] blocks `LOOP-STOP: complete` at ≥3 work-units without a passing loop-scope `evals.json`, fail-open when `work_units` is absent (legacy loop).

Full architecture: [[task-evals-gate]].

## Verifier agent contract (agent-run evals)

A fresh sonnet subagent grades judgement evals. Its prompt contains ONLY the `evals.json` content, artifact references, and the confidence-label contract — explicitly nothing else: not the implementation conversation, not the implementer's summary, not the orchestrator's opinion. Same "the author is the least able to see its own shims" principle as agentic-loop Phase 4b's clean-break gate. The verifier returns per-eval statuses; the orchestrator folds those into `evals.json` and stops there — it never computes or writes `result` itself, at either scope. Computing and stamping `result` is a separate, neutral step: `post_evals.sh` (pr scope) or **`post_evals.sh grade-loop`** (loop scope, [[pr_144-149_agentic-loop-hardening-from-loop-engineering|PR #144]], 2026-07-12). `grade-loop` also stamps a `grading` object (`by` + a checksum over the per-eval statuses + result) that [[loop_state_guard]] checks before accepting a `GO`/`TIER0` verdict — an unstamped or checksum-mismatched file now reads `UNSTAMPED` and blocks, distinct from `NO-GO`/`ABSENT`. Honest boundary: the stamp catches accidental drift (a status edited after grading without re-stamping), not deliberate tampering. PR #153 (2026-07-13) added the regrade-on-amendment backstop: the stamp records `amendments_at_grade`, and `grade-loop` refuses to re-grade a post-verdict amendment that lacks a `regraded_by` field naming the fresh grader — with plainly stated blind spots (a status flip with no accompanying amendment, an in-place amendment edit, or a pre-first-grade flip are all invisible to it and held by rule 5 + the Phase 13 audit alone). See [[task-evals-gate]] for the full mechanism.

## Failure modes encoded

- An eval whose oracle is the implementation's own test or regex (oracle non-independence).
- A scripted check with no negative control — passes trivially, proves nothing.
- Working-tree self-reports standing in for merged/deployed-state assertions.
- The implementer grading its own judgement evals instead of a fresh, context-blind verifier.
- A tier-justification-free artifact at **any** tier, not only tier 0 — silence is never an acceptable substitute for the artifact (extended from tier-0-only by [[pr_7-10_task-evals-followups|PR #10]]).
- Post-freeze eval edits made silently instead of recorded as `amendments`.
- Comment-spoofing: a marker-shaped artifact posted by an untrusted author — closed for both PR-gate readers by [[pr_7-10_task-evals-followups|PR #8]] (author-identity + association filter, fail-closed).

## Relationship to /workflow and agentic-loop

Not itself part of the `/workflow` chain directly — invoked as writing-plans' final task (pr scope) or agentic-loop's Phase 2.7c (loop scope). Sits at the same "gate before merge/completion" altitude as the review-artifact seam, but is independently sourced: an oracle-independent eval suite, not a code review.

## Source

`coderails/skills/task-evals/SKILL.md`

## See also

- [[task-evals-gate]] — design page: the dual-scope enforcement architecture (pr-scope PR-comment gate + loop-scope Stop-hook gate)
- [[post-evals]] — the `/coderails:post-evals` command that posts the pr-scope artifact
- [[merge]] — the pr-scope gate consumer
- [[loop_state_guard]] — the loop-scope gate consumer
- [[writing-plans]] — invokes this skill at the freeze-after-stress-test step (corrected from "final task" wording by [[pr_15-17_loop-hardening-registration-eval-freeze-ledger-dry|PR #15]]); the plan's actual final task only grades and posts
- [[agentic-loop]] — invokes this skill at Phase 2.7c (loop scope)
- [[review-artifact-seam]] — the predecessor truth-seam this design largely mirrors (marker SSOT, SHA-bound artifact, fail-closed gate)
- [[loop-progress-fields]] — consolidating page for `work_units`, the field the loop-scope eval gate's `>=3` threshold reads
- [[pr_1-4_task-evals-feature]] — the original cluster source record (PRs #1–4)
- [[pr_7-10_task-evals-followups]] — the follow-up cluster: wiki-first prerequisite, comment-spoofing/pagination closure, tier_justification everywhere (PRs #7–10)
- [[pr_15-17_loop-hardening-registration-eval-freeze-ledger-dry]] — PR #15: fixes the invocation-point wording that let freeze-before-build collapse into the final task
- [[pr_138_remove-specs-plans-tracking]] — removed this skill's design spec from repo tracking; this page's own content is now the durable record
- [[dashboard]] / [[pr_25_observability-dashboard]] — PR #25 (2026-07-06): first production demonstration of this gate paying for itself — the frozen Tier-2 suite (10 evals, GO) caught two real bugs (a launch-script false-success and a statically-baked empty config) that every review round had missed
- [[pr_144-149_agentic-loop-hardening-from-loop-engineering]] — PRs #144/#145 (2026-07-12) source record: `grade-loop` neutral grading + `UNSTAMPED` demotion, and rule 6 "Strongest surface"
