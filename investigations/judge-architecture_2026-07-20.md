---
title: "Investigation: Judge Architecture in coderails"
type: investigation
created: 2026-07-20
last_updated: 2026-07-20
sources:
  - skills/workflow-audit.md
  - skills/workflow-audit/references/judge-contract.md
  - design/enforcement-model.md
  - sources/pr_27-39_workflow-audit-skill.md
  - sources/pr_232_tier-review-gate.md
tags: [judge, architecture, workflow-audit, tier-review, enforcement, decision-making]
---

# Judge Architecture in coderails

A judge in coderails is an independent decision-maker that evaluates structured input and returns a pass/fail or categorised verdict. Coderails uses judges in two contexts: one for workflow pattern discovery, one for tier-exemption honesty.

## The two judge roles

### 1. Workflow-audit judge (skill-proposal gate)

The [[workflow-audit]] skill's judge is a **fresh sonnet subagent** that examines n-gram clusters mined from transcripts and decides whether each cluster represents a candidate for a new skill.

**Input contract:** exactly two things:
- The cluster JSON from `cluster_ngrams.sh` (tool names, privacy-whitelisted command heads, counts, session IDs, n-gram lengths)
- A list of existing skill names and descriptions

**Fixed vocabulary:** The judge must constrain its reasoning to what appears in the cluster JSON alone. It cannot access transcript prose, file contents, or infer intent beyond whitelisted data — if a judgement would require information outside this scope, it must reject or note the limitation rather than guess.

**Output:** A `{verdict, reject_reason, task_summary, proposed_name, proposed_description}` object per cluster.

**Mandatory rejection criteria (checked in order):**
1. Project-specific convention (not generalisable)
2. Already covered by an existing skill (duplicate)
3. Tooling-mechanics artifact (the loop's own plumbing, not a user task)

**Endpoint:** The judge verdict is the sole in-session filter (as of 2026-07-15, the prior interactive `AskUserQuestion` gate was removed). Every `propose`-verdict goes to [[writing-skills]]' TDD process + PR gate; every `reject`-verdict is dropped. Queue-mode (opt-in) defers creation: Approve on a queue entry (checked/re-validated) spawns `run-builder.sh`, which authors the skill and opens a PR (still reviewed and merged by hand — Approve authorises creation, not landing an unseen diff). See [[pr_55-60-64-66-67_approve-build-runner]] for the builder pipeline.

**Key design principle:** Mechanical prefilter (scan + clustering) handles scale; model judgement handles interpretation. This tiering mirrors [[skill-testing-state_2026-06-26]]'s same pattern applied to a different axis.

### 2. Tier-review judge (tier-0 exemption gate)

The `scripts/tier-gate/` daemon is a **root-owned macOS LaunchDaemon** that judges whether an agent's self-classified `tier=0` eval-exemption claim is honest.

**Architectural difference from the workflow-audit judge:**
- Runs outside the agent's own uid/filesystem/trust-domain (uid 0 vs uid 501)
- Authenticates via the repo owner's Claude subscription (root-held `CLAUDE_CODE_OAUTH_TOKEN`)
- Routes all GitHub reads/writes through root-owned `curl`, never uid-501-writable `gh`
- Executes a root-owned `claude` binary with `HOME`/`cwd` pinned to root-owned paths

This is the **first coderails enforcement layer entirely outside the agent's own trust domain for a judgment call** — not just a filesystem write (sandboxed workers handle that), but an independent verdict that the agent cannot fabricate or bypass.

**Input:** Only `{claimed_tier, diff}` — deliberately excludes the PR's own `tier_justification` prose channel (deleted by in-PR fix `cc5c570`).

**Mechanical prefilter:** Before any judge call, a model-free check blocks obvious cases (path denylist + size cap).

**Output:** Posts a `tier-review` commit status (`verdict=legitimate`, `state=success`, tier token) to the PR, or posts nothing for a non-tier-0 claim.

**Enforcement gates:**
- `[[merge]]` step 4a requires `verdict=legitimate` + `state=success` from the exact configured machine-user login
- `[[enforce_pr_workflow]]` Gate 8 (config-keyed, opt-in via `tier_review.machine_user`) requires the same

Both local gates are **explicitly redundant-by-design** once a GitHub branch-protection ruleset is live (defence-in-depth for the pre-ruleset interim).

**Honest boundary:** The daemon is **available, not active** — dormant until the owner completes credentials population, a branch-protection ruleset, and a live-fire validation pass. It raises the cost of a dishonest tier-0 claim from free to expensive, not to impossible — an LLM judging a diff remains fallible, and it does not judge whether tier-1/2 evals were honest, nor does it judge code quality or correctness.

See [[pr_232_tier-review-gate]] for the full mechanism and the nine in-PR fixes (injection-channel deletion, subscription-auth rewire, root-curl routing, tautological test made to discriminate, etc.).

## The design law: why judges matter

From [[enforcement-model]]: **Hooks are mechanical enforcement. Slash commands are advisory.**

A check performed by the party with motive to pass it is not a check. This shapes where judges live:

- Workflow-audit's judge is a fresh subagent, not the orchestrator — independent evaluation of candidate skills
- Tier-review's judge runs outside the agent's own uid — the agent cannot control the verdict

Both judges sit at boundaries where self-grading would have created a gap: one for discovering new skill patterns, one for validating self-exemption claims.

## Cross-architecture principle

Both judges embody the same constraint: **fixed vocabulary, reject-on-ambiguity, no guessing.** The workflow-audit judge cannot infer intent beyond whitelisted transcript heads; the tier-review judge receives only diff + claimed tier, not justification prose. This forces honest limits: if a judgement requires information the judge doesn't have, it says no rather than inventing details.

## Related

- [[workflow-audit]] — the skill that uses the first judge
- [[enforce_pr_workflow]] — the hook that uses the second judge's output
- [[enforcement-model]] — the design law this architecture implements
- [[pr_232_tier-review-gate]] — the full tier-review daemon design and its own fixes
- [[pr_27-39_workflow-audit-skill]] — the workflow-audit implementation
- [[pr_55-60-64-66-67_approve-build-runner]] — the builder pipeline that turns workflow-audit proposals into PRs
