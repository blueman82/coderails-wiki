---
title: "PR #169 — mandatory model-per-task routing step (Phase 2.8)"
type: source
origin: PR #169 (merged 2026-07-14T19:31:26Z, 229903a)
created: 2026-07-14
last_updated: 2026-07-14
sources: []
tags: [source, agentic-loop, writing-plans, enforcement-model, model-routing, capability-roles, phase-2.8]
---

# PR #169 — mandatory model-per-task routing step (Phase 2.8)

Three files changed (19+9, 71+19, 8+0 lines respectively): `AGENTS.md`,
`skills/agentic-loop/SKILL.md`, `skills/writing-plans/SKILL.md`. One atomic PR
so the three files never contradict each other on `main`.

## What shipped

1. **`skills/agentic-loop/SKILL.md` — new unconditional Phase 2.8, "Route:
   assign a model role per task."** Fires even for a <3-unit loop that skips
   Phase 2.7 entirely. Introduces **model-agnostic capability roles** —
   `fast-mechanical` (haiku), `default` (sonnet), `frontier` (fable, opus
   alternate) — as the durable vocabulary; the role→model table is the only
   thing a future model release touches ("a named-tier table went stale
   within a day of Fable 5's release"). Key rules:
   - **Investigations get `frontier` FIRST, not escalated-to** — a genuinely
     ambiguous investigation spawns at `frontier` from the start; a weak
     investigator burns wall-clock discovering it's out of depth, then a
     second run re-does the work at the stronger tier anyway. The one
     exception to `default`-first cost control everywhere else.
   - **Fallback valves live in the stamp, never improvised by a worker** — an
     escape hatch must be named in the `Model:` stamp/bullet up front; a
     worker picking its own fallback model is the exact failure this rule
     prevents.
   - **Escalation is safe by construction, not a correctness control** — PR
     gates (review, evals, hook-seam) are model-independent, so routing is a
     cost/latency decision only.
   - **Assignment set recorded ONCE** in `progress.json.decisions_absorbed`
     as a `{phase: "2.8", decision: "<task id: role, ...>"}` entry covering
     every task, not one entry per task.
   - All ~13 blanket sonnet/opus assertions across the skill reconciled:
     frontmatter description, Phases 2, 2.5, 3, 3a, 9, 10. Phase 2/2.5/9
     agents get their role assigned **inline**, at their own spawn point,
     using Phase 2.8's table — they spawn *before* 2.8 exists in the
     sequence (Phase 2, 2.5) or are loop-boundary ceremony rather than a
     build task (Phase 9), so 2.8's per-task routing doesn't reach them.
   - Phase 2.5's design-fork agent spawn was **inverted**: previously
     "sonnet recon, escalate the synthesis to opus only if the tradeoff is
     genuinely close" (cost-first, escalate-if-needed) → now `default` for a
     bounded choice, `frontier` **from the start** for a genuinely ambiguous
     investigation (the same investigations-first rule as above).

2. **`skills/writing-plans/SKILL.md` — mandatory `Model:` stamp on every
   task in every plan.** "What each task carries" grows from four to five
   mandatory elements: role (`fast-mechanical`/`default`/`frontier`) + a
   one-line rationale + an optional fallback valve. No loop-only carve-out —
   Gary: "so I am gonna need it." Role definitions, the table, and the
   tiering rationale stay in agentic-loop's Phase 2.8 (cross-referenced, not
   duplicated in this stamp).

3. **`AGENTS.md` — the enforcement-ceilings bullet reworded.** "`model:
   sonnet` for spawned workers is advisory, not hook-enforced" became
   "Model-role routing for spawned workers is advisory, not hook-enforced."
   Still **no enforcement hook** — deliberately: routing is cost/latency
   control, not a correctness gate, and a blunt model-gate hook can't
   distinguish Phase 2.8's sanctioned role-vs-role judgement call (bounded
   `default` vs. genuinely-ambiguous `frontier`-first) from a disallowed
   spawn without trusting a self-reported carve-out flag. The hook-coverage
   claim itself was made precise: the only `PreToolUse` matchers in
   `hooks/hooks.json` are `Bash` and `Write|Edit|MultiEdit` — the remaining
   registered events gate no tool calls at all.

## The Phase 2.8 number was reused, not newly minted

Phase 2.8 existed once before: pre-PR #86, it was the plan-writing phase,
merged into Phase 2.7 as sub-step 2.7b by PR #86 (2026-06-25-ish arc). That
merge is unrelated history and stays true — "Phase 2.7b (formerly 2.8)"
references elsewhere in the wiki describe *that* fact. PR #169 reuses the
now-vacant number **2.8** for a wholly different phase (model-role routing),
inserted between 2.7 and 3. The two Phase-2.8s share only the number. See
[[agentic-loop]]'s "2.7/2.8 merge note, and the 2.8 renumber" for the full
reconciliation.

## Design decisions from the loop

- **Placement as unconditional Phase 2.8**, not a sub-step, so a <3-unit
  loop still routes its 1-2 workers. Two alternatives rejected: a 2.7b
  sub-step (gated away exactly when a small loop needed it most, since 2.7b
  only fires at the ≥3-unit/dependency guard); a Phase 3 preamble (conflates
  "decide the routing once" with "consume it per spawn" — the same
  decide-once-consume-many shape Phase 2.5 already established for design
  forks, which argued for a dedicated phase instead).
- **ONE atomic PR** across all three files so `AGENTS.md`, `agentic-loop`,
  and `writing-plans` never disagree on `main` even transiently.

## Disambiguation: `fable` the model vs. `fable-mode` the skill

The `frontier` role's "currently" model is **Claude Fable 5** — an Anthropic
model, referenced here only by role-table value. This is unrelated to the
repo's separate `coderails:fable-mode` skill (a behavioural skill teaching
high-autonomy, self-verifying working style). The two share a name and
nothing else; PR #169 does not touch `fable-mode`.

## Provenance

- Handoff memory: `project_model_routing_skill_handoff.md`
- Manual precedent: dashboard-rethink session `0767967d`
- Loop session: `17c78b4e-830c-4410-a16a-38008babc7dc`

## See also

- [[agentic-loop]] — the parent skill page; new "Phase 2.8" section, reworded
  "Model-role routing is advisory" section, the 2.8-renumber reconciliation
- [[writing-plans]] — the fifth mandatory `Model:` element added to every
  plan task
- [[enforcement-model]] — the advisory-ceiling bullet reworded for the new
  vocabulary and precise hook-matcher claim
- [[pr_86_agentic-loop-hardening]] — the PR that created the *original*
  Phase 2.8 (plan-writing) and merged it into 2.7b; the number PR #169 reuses
