---
title: "Review-artifact truth seam — SHA-bound PR comment + fail-closed merge gate"
type: design
created: 2026-06-30
last_updated: 2026-07-06
sources:
  - sources/pr_81-83_review-artifact-seam.md
  - docs/coderails/specs/2026-06-30-progress-record-design.md
  - sources/pr_92_exec-bit-sweep.md
  - sources/pr_93-94_post-review-injection-and-exec-bit-invariant.md
  - sources/pr_7-10_task-evals-followups.md
  - sources/pr_11-14_gate-hardening-followups.md
tags: [design, review-artifact, merge-gate, truth-seam, sha-bound, enforcement, post-review, comment-spoofing, trust-floor, viewerPermission]
---

# Review-artifact truth seam

Introduced by [[pr_81-83_review-artifact-seam]] (merged 2026-06-30). Moves review truth from ephemeral chat output to a GitHub-visible, SHA-bound PR comment that `/merge` verifies fail-closed.

## The core problem

Before this design, `/pr-review-toolkit:review-pr` wrote its findings to the chat window only. `enforce_pr_workflow` gated merge on the *transcript event* that `review-pr` ran — not on any durable artifact. Two failure modes:

1. **No external record.** A review could produce blocking findings that the agent ignores before pushing again. The transcript disappears on compaction.
2. **No SHA binding.** A review run against an earlier push would satisfy the transcript gate even if a subsequent push changed the code under review.

The deeper insight from the design process: a self-authored local record (`review.ran = true`) cannot be review-truth. The agent both writes it and is judged on it, with no privilege boundary between writer and judge. This is the same enforcement ceiling documented for every other gate in [[enforcement-model]].

## The decision

Move truth out of the doer's writable file and onto a GitHub artifact that `/merge` verifies against live PR state.

**Three components:**

1. **`/coderails:post-review`** runs after `review-pr` and posts a machine-marked, SHA-bound comment to the PR. See [[post-review]].
2. **`scripts/lib/review-artifact.sh`** — the marker SSOT (25 lines). Both writer and reader source this file; one constructor, no drift.
3. **`scripts/merge.sh` gate** — before `gh pr merge`, fetches PR comments from GitHub and requires an exact marker match for the current head SHA. No local-file fallback. No progress.json fallback.

## Marker format

```
<!-- coderails-review-summary v1 pr=<N> head_sha=<sha> -->
```

Matching uses **exact string equality**, not substring grep. A line with any junk prefix or suffix fails. An unknown/future version never matches (fail-closed). (verified: `review-artifact.sh:17–19`)

## Fail-closed semantics

| Condition | Outcome |
|---|---|
| GitHub fetch fails (gh error) | Block with "GitHub fetch failed" |
| Fetch OK, no matching marker | Block with "run /coderails:post-review" |
| Fetch OK, exact marker found for current SHA | Allow merge |

Exit codes from `pr::has_coderails_review_for_head`: 0=match, 1=no-match, 2=fetch-failed. The distinct codes produce distinct, actionable error messages in `merge.sh`. (verified: `git-common.sh`, `merge.sh`)

**Comment-spoofing closed, `gh` pagination cap removed** ([[pr_7-10_task-evals-followups|PR #8]], 2026-07-06): `pr::has_coderails_review_for_head` now sources comment bodies from `pr::_trusted_comment_bodies`, which filters to the `gh`-authenticated login with `author_association == "OWNER"` before any marker matching, and fetches via a paginated `gh api .../issues/<n>/comments --paginate` call rather than the old `gh pr view --json comments` GraphQL fetch (previously hard-capped at 100 comments). This PR's primary target was the eval-artifact reader (`pr::has_coderails_eval_for_head`, see [[task-evals-gate]]) but both readers share the fetch helper, so this seam inherits the same fix. Scoped limitation: `OWNER` association assumes a personally-owned repo; an org-owned repo's comments would fail closed instead.

## The honest ceiling

The gate proves a durable, SHA-bound, auditable artifact **exists**. It validates the summary's structure (grammar: `## No findings` OR all three headings with bullets/None) but cannot assess whether the review was substantive. A cooperating-but-shallow reviewer can post a structurally-valid summary with no real findings. The gate is an **audit layer, not a tamper-proof barrier** — the same honest boundary as every other local gate. (verified: `AGENTS.md` enforcement-ceiling section)

## `progress.json.review` is cache, not authority

After posting the artifact, `/coderails:post-review` writes `progress.json.review` (URL, comment ID, head SHA) as an **index/cache** for observability and idempotency. The `/merge` gate reads from GitHub, never from `progress.json`. (verified: `post_review.sh`, `merge.sh`)

## Executability is now a codified invariant

[[pr_92_exec-bit-sweep]] (merged 2026-07-03) fixed a real permission-denied bug:
`scripts/post_review.sh` had mode `100644` in the git index despite being invoked
as a direct executable path (`./scripts/post_review.sh validate ...`) rather than
sourced or `bash`-wrapped — this had silently broken step 2 of [[post-review]]
(the validate-before-post abort gate). That PR flagged the gap that no test
asserted the mode bit stays `100755`.

**Gap closed same day by [[pr_93-94_post-review-injection-and-exec-bit-invariant|PR #94]]**:
`hooks/scripts/tests/exec_bit_invariant.test.sh` now asserts an expected-modes
manifest against the live git index for every tracked script under `scripts/` and
`hooks/scripts/`, plus a completeness scan that fails if a new script isn't
covered. Suite 23→24. The same PR normalised two more source-only libs
(`scripts/lib/git-common.sh`, `hooks/scripts/lib/agentic_loop_path.sh`) from
`100755`→`100644` that #92 didn't reach. A regression to `scripts/post_review.sh`'s
mode bit now fails the suite instead of silently reintroducing the
permission-denied bug.

## Deferred scope

Two items from the design spec were deferred by the planning-sequence stress-test:

- **Universal Stop guard** (`progress_record_guard.sh`) — would add ceremony tax to every non-loop run. Not implemented.
- **Universal-ledger enforcement** — same reason. The `/prep` Part 1b stub is present but optional/non-blocking.

These may be revisited once the gate is verified in production.

## Follow-up ordering constraint

`enforce_pr_workflow`'s review-pr transcript arm is expected to demote from block to nudge once this gate is live and verified — the durable artifact provides a stronger guarantee. This demotion MUST NOT happen before verification. A no-enforcement window between demotion and verification would allow unreviewed PRs to merge. (verified: `AGENTS.md` follow-up section)

## Architecture position

```
review-pr (external, chat only)
    ↓
/coderails:post-review         ← new: posts SHA-bound GitHub comment
    ↓                                  validates grammar before posting
progress.json.review (cache)           writes URL/id/sha as cache
    ↓
/merge gate                    ← new: fetches GitHub comments, exact SHA match
    ↓
gh pr merge
```

The chain is linear and fail-closed at each step. No local escape path.

## See also

- [[post-review]] — the `/coderails:post-review` command page
- [[merge]] — the updated `/coderails:merge` command with the artifact gate
- [[workflow]] — Phase 3 update (post-review before ship-it pause)
- [[agentic-loop]] — Phase 4b update (loop symmetry)
- [[enforce_pr_workflow]] — the predecessor transcript-based gate
- [[enforcement-model]] — the honest-ceiling framework this design sits within
- [[spec-plan-progress-artifact-chain]] — the predecessor artifact chain
- [[pr_81-83_review-artifact-seam]] — the cluster source page
- [[pr_93-94_post-review-injection-and-exec-bit-invariant]] — closes the exec-bit test gap; ships the post-review.md injection
- [[pr_7-10_task-evals-followups]] — closes the comment-spoofing hole and `gh` pagination cap shared by this seam's reader
- [[task-evals-gate]] — the sibling seam sharing the same fixed comment-fetch helper
