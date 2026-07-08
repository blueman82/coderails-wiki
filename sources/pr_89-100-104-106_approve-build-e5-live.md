---
title: "PRs #89/#100/#104/#106 — Approve→build pipeline goes live: first real skill built + honest build feedback"
type: source
created: 2026-07-08
last_updated: 2026-07-08
sources: []
tags: [source, workflow-audit, dashboard, queue-contract, builder, skill-creator, verify-merged-pr, agentic-os, e5, live-fire]
---

# PRs #89/#100/#104/#106 — Approve→build pipeline goes live: first real skill built + honest build feedback

Ingested by the loop-boundary wiki agent after all four PRs merged to `main`.
One source page for the cluster (not fragmented per-PR). This is the
**live-fire close** of the Approve→build pipeline first wired in
[[pr_55-60-64-66-67_approve-build-runner]] (loop 2). That prior cluster left one
caveat explicit: *E5 — one real, manual, end-to-end build through the pipeline —
had not yet run.* This cluster ran it. Driving a genuine dashboard Approve click
through to a merged skill PR surfaced three production-only defects no automated
gate had caught (#89, #100, and the feedback gaps #106 addresses), and produced
the first artifact of the whole loop-2 thesis: a skill authored by a headless
`skill-creator` build (#104).

> NOTE: PR numbers are re-used across unrelated work in this repo's history. The
> `#89`/`#100` here are the **builder-repo-identity** and **launchd-env** fixes,
> distinct from the pre-existing wiki's `#89` ([[pr_89-91_skills-doc-frontmatter-injection]])
> and any other `#100`. The merge SHAs below are the reliable identifiers.

## PR metadata

| Field | Value |
|---|---|
| PR #89 (builder repo-identity) | "fix: builder repo-identity via `.claude-plugin/plugin.json` exact-name match" — merge `91d3802` |
| PR #100 (launchd env) | "dashboard launchd builder env" — exports `CODERAILS_BUILDER_REPO_PATH`/`_WRAPPER` + `~/.local/bin` on PATH — merge `a6f40ec`, merged 2026-07-08T13:24:47Z |
| PR #104 (first built skill) | "workflow-audit/skill verify-merged-pr" — merge `c30fcbf`, merged 2026-07-08T14:40:52+01:00 |
| PR #106 (build feedback) | "feat(dashboard): build-progress feedback + after-merge reconciliation" — merge `23a73bd`, merged 2026-07-08T15:02:49+01:00 |
| Repo | `blueman82/coderails` |
| Builds on | [[pr_55-60-64-66-67_approve-build-runner]]'s claim/spawn seam, `run-builder.sh` state machine, injection-fenced prompt, and `collectBuilds`/`AssistantLinkPanel` visibility |

## Summary

The pipeline documented in [[pr_55-60-64-66-67_approve-build-runner]] was fully
wired and mechanically hardened but never exercised end-to-end. Running one real
Approve click live (E5) turned three latent, production-only bugs into fixes and
produced the pipeline's first real output:

- **#89** — the builder identified its target repo by grepping a **nonexistent
  root `package.json`**; corrected to an exact-name match against
  `.claude-plugin/plugin.json`, the file that actually identifies the coderails
  plugin repo.
- **#100** — the launchd launcher didn't export the env the builder wrapper
  needs (`CODERAILS_BUILDER_REPO_PATH`, `CODERAILS_BUILDER_WRAPPER`) and didn't
  put `~/.local/bin` on `PATH`, so a launchd-spawned build couldn't find its repo
  or its tools. The launcher now exports both and extends `PATH`.
- **#104** — `skills/verify-merged-pr/` (SKILL.md + `evals/evals.json`), authored
  by a headless `skill-creator` session spawned from a genuine dashboard Approve
  click. **The first skill the Approve→build pipeline ever produced** — the
  loop-2 thesis proven end-to-end, not just wired. (The skill itself: guards
  against trusting a "PR is merged" claim before building on it — see
  [[verify-merged-pr]].)
- **#106** — build-progress feedback + after-merge reconciliation, so the panel
  stops lying during and after a 45-minute build (below).

The arc across the whole loop: **Approve went from flipping a status field to
running a real headless build to a merged skill PR, with honest live feedback.**

## Architecture — PR #106 (build feedback + reconciliation)

The prior cluster's panel showed a bare `building` for up to 45 minutes and could
stick on a stale `awaiting your merge` after the PR was already merged. #106 fixes
both, all layers TDD'd:

1. **Live phase.** The builder self-reports a coarse phase — one of
   `authoring | testing | pushing | opening_pr` — by writing a single word to
   `builds/<hash>/phase`. `buildPrompt` now takes `buildDir` to interpolate that
   concrete phase-file path into the prompt (`lib/build/prompt.ts`,
   `lib/build/spawn.ts`). The `collectBuilds` collector (`lib/collect/builds.ts`)
   **closed-set-validates the phase word before it ever reaches the client** —
   same reject-never-default discipline already applied to `state`. The panel
   renders `building` followed by the phase.
2. **Elapsed timer + heartbeat freshness.** The panel shows elapsed time from
   `startedAt` and a "last active Ns ago" heartbeat-freshness readout, so a live
   build is visibly progressing rather than an opaque spinner.
3. **After-merge reconciliation.** The builder writes `pr_url` **immediately**
   after `gh pr create` (not only as a final act), so the panel can join the
   build's PR against the dashboard's already-collected open-PR set. Once that PR
   leaves the open set (merged or closed), the panel shows **`PR resolved`**
   instead of a stale `awaiting your merge`.
4. **Null-guard against false resolution.** Reconciliation is **skipped** (falls
   back to `awaiting`) whenever the open-PR set is untrustworthy — gates not yet
   loaded, the poll failed, or a repo degraded to an error entry — so an open PR
   is **never** falsely marked resolved. Honesty bias: prefer showing a stale
   "awaiting" over a wrong "resolved".

## Key decisions

- **Live-fire is a distinct gate from wiring.** Every one of #89/#100 was a
  production-only defect (missing root `package.json`, missing launchd env) that
  the full test suite of [[pr_55-60-64-66-67_approve-build-runner]] passed
  cleanly over. The pipeline being "wired and tested" did not mean it worked; E5
  was the gate that mattered. `(verified — team-lead's brief names #89/#100 as
  live-E5-surfaced fixes)`
- **The builder's environment is the fragile seam.** Both #89 and #100 are about
  the builder correctly locating its repo and tools when spawned detached from a
  launchd context — not about the pipeline's logic. The state machine
  ([[pr_55-60-64-66-67_approve-build-runner]]) was already sound; what broke was
  identity and environment inheritance. `(inferred — both fixes touch repo-path /
  env resolution, not the wrapper state logic)`
- **Feedback honesty mirrors the pipeline's merge honesty.** Just as the pipeline
  refuses to let the headless builder merge its own PR, #106 refuses to *claim*
  resolution it can't trust — the open-PR-set null-guard is the same "never assert
  what you can't verify" bias applied to UI state. `(verified — #106 body: "an
  open PR is never falsely marked resolved")`
- **The first artifact validates the whole sub-project-3 premise.** [[workflow-audit]]
  mines transcripts for repeated tasks → proposes a skill → the dashboard Approve
  builds it. #104 (`verify-merged-pr`) is the first skill to traverse that entire
  chain from proposal to merged PR. `(verified — #104 is a `skill-creator`-authored
  skill merged from an Approve-spawned build)`

## Files changed

- `.claude-plugin/plugin.json` exact-name resolution in the builder (PR #89)
- launchd launcher env exports + `PATH` extension (PR #100)
- `skills/verify-merged-pr/SKILL.md`, `skills/verify-merged-pr/evals/evals.json` (PR #104, new)
- `skills/dashboard/app/src/components/AssistantLinkPanel.tsx` (PR #106 — phase, timer, heartbeat, PR-resolved reconciliation)
- `skills/dashboard/app/src/lib/build/prompt.ts`, `.../lib/build/spawn.ts` (PR #106 — `buildDir` → phase-file path)
- `skills/dashboard/app/src/lib/collect/builds.ts` (PR #106 — closed-set phase validation)
- `skills/dashboard/app/src/styles/hud.css` (PR #106)
- Test files (PR #106): `AssistantLinkPanel.test.ts`, `builderPrompt.test.ts`, `builds.test.ts`

## Wiki pages updated

- [[verify-merged-pr]] — **new** skill page for the first built skill
- [[dashboard]] — AssistantLinkPanel section extended: live phase, elapsed
  timer, heartbeat freshness, and the `PR resolved` after-merge reconciliation
  (with its untrustworthy-set null-guard)
- [[workflow-audit]] — pipeline-outcome note: the Approve→build loop has now
  produced its first real skill end-to-end (E5 done)
- [[pr_55-60-64-66-67_approve-build-runner]] — E5-pending caveat resolved,
  pointing here

## Caveats / gotchas

- **E5 is now DONE** — the prior cluster's headline caveat ("no proposal has yet
  been run through it end-to-end on a live approval") is resolved by this cluster.
  #104 is the concrete proof. `(verified — #104 is a real merged skill PR from an
  Approve-spawned build)`
- **The phase word is coarse, not a progress percentage** — `authoring/testing/
  pushing/opening_pr` is a four-state hint, not fine-grained progress. It exists
  to replace an opaque 45-minute spinner, not to precisely track build internals.
  `(verified — #106 body: "coarse phase")`
- **PR numbers `#89`/`#100` collide with unrelated earlier work** — always
  resolve via merge SHA (`91d3802` for repo-identity; `a6f40ec` for #100), not
  the bare number. `(verified — repo re-uses PR numbers, a
  pattern already flagged across prior clusters)`
