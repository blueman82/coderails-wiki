---
title: "PR #138 — remove docs/coderails/specs and docs/coderails/plans from tracking"
type: source
origin: PR #138 (merged 2026-07-11, b0a4487)
date: 2026-07-11
tags: [source, brainstorming, writing-plans, discipline-loop]
---

# PR #138 — remove docs/coderails/specs and docs/coderails/plans from tracking

> ⚠️ SUPERSEDES prior wiki/repo convention: `docs/coderails/specs/` and `docs/coderails/plans/` were documented across several places (`docs/REFERENCE.md`, `skills/brainstorming/SKILL.md`, `skills/workflow-audit/SKILL.md`) as **permanent, committed repo artifacts** written by `brainstorming` and `writing-plans`. Per an explicit owner decision on 2026-07-11 ("wipe coderails/plans and specs/ from repo, never track ever again"), this is no longer true. Both directories are gitignored; specs/plans are now session-local working documents only.

## What shipped

- Removed 20 previously-tracked files (14 spec/plan `.md` docs going back to 2026-06-26, 5 mockup/reference assets under `docs/coderails/specs/assets/`, plus one still-unmerged doc from earlier in the same session) from `main`.
- Added both directories to `.gitignore`.
- Updated `skills/brainstorming/SKILL.md`: the checklist and process-flow diagram no longer include a separate "Ensure isolated workspace" step for the spec-writing stage (that step existed specifically to protect the spec *commit* from landing on `main` — with nothing committed anymore, the rationale no longer applies). The "Documentation" section now instructs writing the spec to a session-local scratch path and explicitly NOT committing it; if a durable record is wanted, `coderails:handoff` or a wiki page is the suggested alternative.
- Updated `docs/REFERENCE.md` in three places (the `brainstorming` entry, the `writing-plans` entry, the Artifact-and-State-Locations table) to reflect the new ephemeral status, and removed a dangling self-referential link (the "Approve-click build runner" entry pointed at `docs/coderails/specs/2026-07-07-approve-build-runner.md`, one of the removed files — the paragraph already restates the contract inline, so the pointer was simply dropped).
- Fixed 8 comments across 7 dashboard source files (`AssistantLinkPanel.tsx`, `api/queue/route.ts`, `lib/collect/queue.ts`, `lib/collect/queueActions.ts`, `lib/collect/builds.ts`, `lib/runHue.ts`, `styles/hud.css`, `components/sphere/Fallback2D.tsx`, `components/sphere/NetworkSphere.tsx`) that cited the now-deleted spec/mockup files as normative sources. Contract-citing comments (queue shape, build sidecar schema) were redirected to the actual current source of truth — e.g. `queue.ts`'s own `QueueEntry` type definition, which is now the frozen contract since there's no separate spec file — rather than left dangling. Mockup-provenance comments (visual reference only, no behavioral contract) were softened to historical prose with no file path.

## Verification

- `hooks/scripts/tests/run_all.sh`: 38/38, run fresh both before and after the reference-fix commits.
- `skills/dashboard/app` test suite: 28/28 files, 418/418 tests — confirms the 8 comment-only edits to dashboard source introduced no regression.
- A frozen tier-1 `evals.json` (5 evals, 3 P0 + 2 P1) asserted the actual goal state post-merge rather than trusting the removal script: zero tracked files remain under either path (E1), both `.gitignore` entries are present (E2), and the exact set of files still mentioning either path matches an explicit 4-file allowlist of known-intentional references — no other file dangles a reference (E3, amended once after the scripted check correctly caught an over-strict first assertion). Result: GO.

## Process notes worth keeping

- **A `cd`-doesn't-persist environment gotcha caused a near-miss.** The orchestrator's first attempt at creating an isolated worktree for this removal silently ran against the wrong branch (its own session worktree) because a shell `cd` into a freshly-created worktree didn't survive to the next tool call in this environment — the harness resets cwd after each Bash invocation. Caught via `pwd`/`git branch --show-current` before any commit landed; nothing was lost, but the fix (use `git -C <path>` / absolute paths throughout, never rely on `cd` persisting) is worth remembering for any future multi-step git operation in this environment.
- **A review pass caught 3 real findings before merge**, all fixed: an orphaned "Spec written and committed" phrase left over in `brainstorming/SKILL.md`'s user-review-gate template after the no-commit policy was written above it; the 8 dashboard-comment dangling references (initially missed by the orchestrator's own manual grep, caught by an independent review agent's exhaustive search); and the `docs/REFERENCE.md` self-referential dangling link. Independent review found real gaps a same-session self-check had missed — the pattern this repo's own review-gate discipline exists to catch.

## See also

- [[brainstorming]] / [[writing-plans]] — the two skills whose write-behavior this PR changes
- [[review-artifact-seam]] / [[task-evals-gate]] — the two merge gates this PR satisfied (review + eval artifacts, both SHA-bound PR comments)
- [[assistant-link-send-gate-architecture]] — the queue contract now documented only in `queue.ts` itself (one of the dashboard source files this PR's reference-fix touched), since the spec file `2026-06-06-assistant-link-panel-design.md` it used to cite no longer exists
- [[pr_130-136_dashboard-right-rail-ux]] — the loop this cleanup ran inside, prompted by the user noticing the (unrelated, prior-session) spec/plan pair this loop had itself written and asking why they existed at all
