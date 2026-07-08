---
title: "Skill: verify-merged-pr"
type: skill
created: 2026-07-08
last_updated: 2026-07-08
sources:
  - sources/pr_89-100-104-106_approve-build-e5-live.md
tags: [skill, verify-merged-pr, workflow-audit, skill-creator, builder, agentic-os, first-built-skill]
---

# Skill: verify-merged-pr

Re-derives a "PR #N is merged" claim from the tools before you rely on it —
independently confirming the merge **state**, the **content** on `origin/main`,
and the **sibling PRs** the reporter may not have mentioned.

Source: `coderails/skills/verify-merged-pr/SKILL.md`
Evals: `coderails/skills/verify-merged-pr/evals/evals.json`
Invoked as: `coderails:verify-merged-pr`

## Provenance — the first Approve→build skill

This is the **first skill authored end-to-end by the dashboard Approve→build
pipeline** ([[pr_89-100-104-106_approve-build-e5-live|PR #104]]). A
[[workflow-audit]] proposal was Approved on the dashboard; a headless
`skill-creator` session ([[pr_55-60-64-66-67_approve-build-runner|the builder
pipeline]]) authored the skill and its evals and opened the PR; the owner merged
it by hand. It is the concrete proof that the loop-2 thesis works — proposal →
build → merged skill PR — not just that it was wired. `(verified — merge
`c30fcbf`)`

## When to use

- An agent, teammate, CI report, or session summary says a PR is merged /
  shipped / live / landed.
- You are about to build on, deploy, or hand off work that depends on the merge
  being real.
- A **headless builder or loop reports "done — PR merged"** and gives you one PR
  number — exactly the trust boundary the Approve→build pipeline itself lives on.

**Not for:** a merge you performed and watched complete this session, or a claim
about an open/draft PR (nothing merged to verify).

## The three checks

The claim is confirmed only when **all three** check out:

1. **State** — `gh pr view <N> --json state,mergedAt,mergeCommit,author,baseRefName`.
   Confirm `MERGED`, non-null `mergedAt`, and note the merge commit + the actual
   `baseRefName` (the PR may not target `main`).
2. **Content** — `git fetch origin <base>`, then `git merge-base --is-ancestor
   <oid> origin/<base>` **and** `git grep '<string unique to the change>'
   origin/<base>`. A merge marker is not the change; ancestry + grep proves the
   substantive content is on the branch you build from.
3. **Siblings** — `gh pr list --state merged --search "author:<login>
   sort:updated-desc"`. The reporter names one PR; sessions often land several.
   Enumerate every PR clustered tightly (minutes apart, same author) around the
   named one — **this is the check agents skip, and the point of the skill.**

## Key design decisions

- **The sibling check is the differentiator.** Verifying only the named PR
  confirms the claim while missing half the work that actually merged — a failure
  mode this repo hits directly, since its own loops land PR bursts and re-use PR
  numbers across unrelated work (see [[pr_89-100-104-106_approve-build-e5-live]]).
  `(verified — SKILL.md "Common Mistakes": "Verifying only the named PR")`
- **Presence is not behaviour.** The verdict must separate "content is on
  `origin/main`" from "the change works" — verifying the grep hits is not
  verifying runtime behaviour. `(verified — SKILL.md reporting rules)`
- **Fetch before you check.** Every content check fetches first; the local
  snapshot is assumed stale. `(verified — SKILL.md check 2)`

## See also

- [[pr_89-100-104-106_approve-build-e5-live]] — the source record; this skill is
  its headline artifact
- [[workflow-audit]] — the proposal engine that surfaced this skill candidate
- [[pr_55-60-64-66-67_approve-build-runner]] — the pipeline that built it
- [[verification-before-completion]] — the sibling discipline for verifying your
  own work before claiming completion; verify-merged-pr applies the same
  "re-derive, don't trust" stance to *someone else's* merge claim
- [[agentic-loop]] — the multi-PR orchestration skill this discipline applies to
  directly: an orchestrator relying on a worker's or subagent's "PR #N is
  merged" report sits on exactly the trust boundary this skill exists to close
