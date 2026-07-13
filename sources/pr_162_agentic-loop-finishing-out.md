---
title: "PR #162 — agentic-loop finishing-out.md sibling extraction"
type: source
origin: PR #162 (merged 2026-07-13T18:11Z, b8fe61c)
created: 2026-07-13
last_updated: 2026-07-13
sources: []
tags: [source, agentic-loop, finishing-out, verification-before-completion, finishing-a-development-branch]
---

# PR #162 — agentic-loop finishing-out.md sibling extraction

Direct sequel to [[pr_134_agentic-loop-retry-until-green|PR #134]]: the same
progressive-disclosure pattern (a one-line markdown-link pointer left inline in
`SKILL.md`, detail moved to a new sibling file under `skills/agentic-loop/`) applied to
two loop-finish mechanics. Where PR #134 extracted `retry-until-green.md`, this PR
extracts `skills/agentic-loop/finishing-out.md` (40 lines).

## What shipped

Two wirings, both re-pointed at the new sibling file rather than newly invented:

1. **Orchestrator-level `verification-before-completion` at the Phase 13 `complete`
   declaration.** SKILL.md already had two [[verification-before-completion]]
   references (Phase 3/3a worker construction-discipline lines) — both discipline
   WORKERS' claims. Nothing previously disciplined the ORCHESTRATOR's own final
   completion claim. Now, immediately before the Phase 13 `LOOP-STOP: complete`
   declaration, the orchestrator applies verification-before-completion to itself:
   re-derive the claim from fresh evidence (each merged PR's `mergedAt` via `gh pr
   view`, the loop-scope eval `result` from `post_evals.sh grade-loop`, the
   wiki/sync-docs artifacts landed on `origin/main`) rather than recall.
   **Scoping decision:** this gates ONLY the Phase 13 `complete` declaration, not
   per-unit merge claims — Phase 12 already re-checks each merge independently, and
   applying VBC there too would duplicate that check. One aggregate check at loop
   end, not a repeat of the per-merge one.
2. **Per-unit worktree teardown reframed as [[finishing-a-development-branch]] Step 6.**
   Phase 4b's worktree-cleanup paragraph previously enumerated the cleanup commands
   inline (`cd` to main root, `git worktree remove`, `git worktree prune`, provenance
   check). PR #162 replaces that inline enumeration with a reference to
   finishing-a-development-branch's Step 6, now carried in the sibling file alongside
   the mechanics and provenance check. Adds one explicit caveat not previously
   stated inline: **never `git worktree remove` the worktree that is the shell's own
   cwd** — the command fails from inside the worktree being removed, so the
   orchestrator must `cd` to the main repo root first. This runs per-work-unit at
   Phase 4b, immediately after `/coderails:merge` confirms that unit's PR merged —
   still not deferred to Phase 9/13's loop-level teardown, which handles wiki/retro
   artifacts, not worktrees.

## Why a sibling file, not an inline SKILL.md edit

Same rationale as PR #134's `retry-until-green.md` extraction and the prior
`learned-failure-modes.md` precedent: `SKILL.md` is already past Anthropic's documented
500-line guidance, and growing it further trades hook-relevant density for exposition.
`finishing-out.md` bundles both loop-finish mechanics under one file rather than two,
since both are "how the loop finishes" content and neither is large enough alone to
justify a separate sibling.

## See also

- [[agentic-loop]] — the parent skill page this PR modifies
- [[pr_134_agentic-loop-retry-until-green]] — the precedent this PR directly follows:
  same sibling-file extraction pattern, same one-line-link idiom in SKILL.md
- [[verification-before-completion]] — the skill newly wired at the orchestrator level
  (Phase 13), distinct from its pre-existing worker-level (Phase 3/3a) wiring
- [[finishing-a-development-branch]] — the skill whose Step 6 mechanics the Phase 4b
  worktree-teardown paragraph now references instead of re-enumerating inline
