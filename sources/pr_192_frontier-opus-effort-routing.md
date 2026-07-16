---
title: "PR #192 — frontier role re-pointed to opus at xhigh; effort joins the Model stamp"
type: source
created: 2026-07-16
last_updated: 2026-07-16
origin: "PR #192, merged 2026-07-16 (90f359e)"
sources: []
tags: [source, agentic-loop, writing-plans, model-routing, capability-roles, effort-routing, cost-control]
---

# PR #192 — frontier → opus at `xhigh`; per-task effort routing

Owner-directed correction to PR #169's role→model table, driven end-to-end
through the full gate chain (push → review artifact → tier-0 eval artifact →
merge) in one session turn, at the owner's instruction, before three planned
improvement loops were allowed to start.

## What changed

Three files, lockstep (41 insertions):

1. **`skills/agentic-loop/SKILL.md` Phase 2.8** — the `frontier` row now
   resolves to **opus at `xhigh` effort**, not fable. Two new rule
   paragraphs:
   - *"`frontier` resolves to opus, never automatically to fable"* — fable
     escalation requires BOTH a named capability gap in the `Model:` stamp
     (what opus-at-xhigh cannot do) and the standard fallback-valve
     discipline.
   - *"Effort is part of the stamp"* — every stamp names role AND effort:
     `frontier` → opus `xhigh` (`max` = per-task escalation with named
     reason); `default` → sonnet `high` (may lower bounded exact-recipe
     tasks to `medium`; never investigations/reviews); `fast-mechanical` →
     haiku (no effort parameter). Effort tuning is the first lever, model
     escalation the second.
2. **`skills/writing-plans/SKILL.md`** — the mandatory `Model:` stamp now
   carries the effort level where the role's model supports one; second
   example added showing the frontier/effort form.
3. **`AGENTS.md`** — advisory-routing ceiling bullet updated: role + effort
   assignment, Phase 2.8 named as home of effort defaults and the
   fable-escalation rule.

## Why (the decision record)

- **Owner's rationale (verbatim intent):** fable and opus are both
  available; auto-choosing fable — the most powerful AND most expensive —
  "is not fair to the human". A cost decision of that size must never be
  made silently by the loop.
- **Vendor guidance agrees** (platform.claude.com "Choosing a model",
  fetched 2026-07-16): the model-selection matrix places complex agentic
  coding — multihour autonomous agents, large-scale refactoring, systems
  engineering — on **Opus**, with `xhigh` effort explicitly named "the best
  setting for most coding and agentic use cases"; **Fable 5** is positioned
  for next-generation intelligence at $10/$50 per M tokens (premium, ~2×
  opus). The guidance also states tuning effort is often a better lever
  than switching models — the direct origin of the effort-stamp rule (the
  owner extended the change to cover it mid-turn).

## Gate trail

- PR #192, 4 auto-commits (the repo's auto-commit hook committed each Edit;
  the crafted commit message was superseded — same content, hook-authored
  messages).
- Review artifact: no findings (inline pass; consistency sweep confirmed no
  stray fable/opus/effort references contradict the new text).
- Eval artifact: tier 0, GO computed by `post_evals.sh`. One amendment on
  the record: an E3 (suite-green) eval was REMOVED pre-grading because
  `validate-structure` correctly refused its negative control as identical
  to its check (vacuous control) — folded into E2's evidence instead. The
  validator catching a vacuous control in live use is itself evidence the
  eval gates work.
- Full hook suite green post-edit (exit 0).

## Impact

- Every future loop's `frontier` work runs at roughly half the token cost
  unless a stamp explicitly justifies fable — the justification burden now
  sits on the expensive choice, not the cheap one.
- Plans stamp effort per task; [[loop-cost-tracking]]'s per-model miner
  will show the shift in retro.json cost objects.
- The three in-flight improvement-loop plans (sandbox-workers, tier-review,
  workflow-audit-routine — session-local docs) needed NO edits: they stamp
  roles, not model names, so the table re-points them automatically —
  the role indirection working exactly as designed by [[pr_169_model-routing-step]].

## Cross-references

- [[pr_169_model-routing-step]] — the source this corrects: same table, new
  resolution for `frontier`, plus the new effort dimension.
- [[agentic-loop]] / [[writing-plans]] / [[enforcement-model]] — pages
  updated by this ingest.
