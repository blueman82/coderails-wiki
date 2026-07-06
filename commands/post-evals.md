---
title: "/coderails:post-evals"
type: command
created: 2026-07-06
last_updated: 2026-07-06
sources:
  - commands/post-evals.md
  - scripts/lib/eval-artifact.sh
  - scripts/post_evals.sh
  - sources/pr_1-4_task-evals-feature.md
tags: [command, post-evals, eval-artifact, sha-bound, merge-gate, task-evals]
---

# /coderails:post-evals

Added by [[pr_1-4_task-evals-feature]] (WU2, PR #3, merged 2026-07-06). Validates and posts a machine-marked, SHA-bound eval summary as a durable GitHub PR comment. Creates the pr-scope artifact that [[merge]] verifies before merging, additive to the existing review artifact gate. Structurally and procedurally mirrors [[post-review]] — same marker-SSOT pattern, same `gh api` posting convention, same duplicate-guard, but for `evals.json` rather than a hand-written review summary.

## Invocation

```
/coderails:post-evals <PR#>
```

Run after `/coderails:task-evals` has produced an `evals.json` for this PR — typically as the plan's final task (see [[writing-plans]]'s mandatory eval-gate task) or directly on request.

## What it does

Seven sequential steps (verified: `commands/post-evals.md`):

1. **Step 0 — Argument gate.** Verify the argument is a plain PR number (digits only, non-empty) **by inspection** — never by pasting it into a shell command. Stop if empty or non-numeric. Ships this way from day one (post-evals.md was written after PR #97 closed the equivalent injection class in `post-review.md`, so it never carried the vulnerable render-time pattern).
2. **Locate the evals.json** — working material, not a fixed path; `/coderails:task-evals` (pr scope) doesn't mandate a fixed location, so it's found wherever the invoking workflow placed it.
3. **Resolve head SHA** — `gh pr view "$ARGUMENTS" --json headRefOid -q .headRefOid`.
4. **Validate structure** — `./scripts/post_evals.sh validate-structure <path> "$ARGUMENTS" "$HEAD_SHA"`. Non-zero exit **aborts** — do not post. See the 7 structural refusals below.
5. **Compute result + read tier** — `./scripts/post_evals.sh compute-result <path>` (never hand-written) and `jq -r '.tier'`.
6. **Build marker, write summary** — sources `scripts/lib/eval-artifact.sh`, calls `eval_artifact::marker`. Summary body: per-eval pass/fail split by priority, plus any `amendments` verbatim. The prose summary is deliberately **not grammar-gated** — the JSON's structural guarantees are what the merge gate relies on, not the comment wording (contrast with [[post-review]]'s grammar-checked summary, a deliberate difference: `task-evals`' floor is the schema, not prose structure).
7. **Post via `gh api`** (not `gh pr comment`) — checks for an existing artifact comment matching this PR+SHA first (skip + report existing URL if found, avoiding duplicate artifacts), otherwise posts and captures the returned URL/id/author/created metadata.

Report step: prints the posted comment URL and the computed result/tier.

## Current PR State injection

`commands/post-evals.md`'s frontmatter injects `- Open PRs: !`gh pr list --state open --limit 10`` — the argument-free convention `merge.md` always used and `post-review.md` adopted only after PR #97's fix. `post-evals.md` was authored after that fix landed, so it carries the safe pattern natively; it was never exposed to the command-substitution injection class described on [[post-review]] and [[pr_96-98_mode-aware-install-argument-injection-guard-hook-owned-counter]].

## Structural refusals (`post_evals::validate_structure`, `scripts/post_evals.sh`)

Seven checks, first-failure-wins:

1. File exists and is valid JSON.
2. Every tier requires a non-blank `tier_justification` (owner directive, [[pr_7-10_task-evals-followups]]: tightened from tier-0-only to all tiers) — tier 0 justifies the exemption itself; tier ≥1 justifies which tier predicate fired.
3. Tier ≥1 scripted eval must have non-empty `negative_control`.
4. `negative_control` must not be vacuous relative to `cmd` — whitespace-normalised equality, plus a word-bounded containment check (catches `"true; cmd"`/`"echo x && cmd"` wrapper tricks without false-flagging a genuinely distinct control that happens to share a text prefix).
5. Any P0 eval must carry non-empty `evidence`.
6. `head_sha` in the file must match the PR's current live head SHA.
7. Tier ≥1 requires at least one actual P0 eval in `.evals` — closes a vacuous-GO gap where an empty or P1-only `.evals` array would otherwise pass `eval_artifact::compute_go`'s P0-only predicate vacuously. Tier 0 is exempt (its `tier_justification` stands in for evals).

## Marker format

```
<!-- coderails-eval-summary v1 pr=<N> head_sha=<sha> result=<GO|NO-GO> tier=<0|1|2> -->
```

Built by `eval_artifact::marker()` in `scripts/lib/eval-artifact.sh` — the same file `pr::has_coderails_eval_for_head` in `git-common.sh` uses for matching. Matching is **literal prefix string-equality** through `head_sha=<sha> result=`, not a regex built from interpolated `pr`/`head_sha` — closes a metacharacter-injection path a review round found. See [[task-evals-gate]] for the full detail.

## Scripts used

- `scripts/lib/eval-artifact.sh` — marker SSOT + `eval_artifact::compute_go` (sourced, not run)
- `scripts/post_evals.sh` — `validate-structure` and `compute-result` subcommands

## Chain position

Sits between `task-evals` (or a plan's final task) and `/merge`:

```
/coderails:task-evals  →  /coderails:post-evals  →  /coderails:merge
```

Without a valid GO (or justified tier-0) artifact on the current head SHA, [[merge]] blocks with an actionable error pointing to this command — additive to, not a replacement for, the pre-existing review-artifact block.

## Preconditions

- `gh` on PATH and authenticated
- Remote must be a `github.com` repository
- An `evals.json` for this PR must already exist, produced by `/coderails:task-evals`
- `head_sha` in the evals.json must match the PR's current live head — a stale artifact from an earlier push fails validation

## Honest ceiling

Same posture as [[post-review]]: this command validates artifact structure and P0 pass/fail, not whether the evals were genuinely oracle-independent or whether an `agent-run` verifier was genuinely given a clean context. The five anti-gaming rules are generation-time discipline in [[task-evals]]; this command is a structural floor at post-time, not a re-verification of the generation discipline. See [[task-evals-gate]] and [[enforcement-model]].

## See also

- [[task-evals]] — the skill that produces the `evals.json` this command validates and posts
- [[task-evals-gate]] — design page for the full dual-scope enforcement architecture
- [[merge]] — the command that gate-checks this artifact before merging
- [[post-review]] — the structurally-analogous predecessor command for review artifacts
- [[review-artifact-seam]] — the design page this pattern was largely mirrored from
- [[pr_1-4_task-evals-feature]] — the cluster source record
