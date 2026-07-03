---
title: "/coderails:post-review"
type: command
created: 2026-06-30
last_updated: 2026-07-03
sources:
  - commands/post-review.md
  - scripts/lib/review-artifact.sh
  - scripts/post_review.sh
  - sources/pr_81-83_review-artifact-seam.md
  - sources/pr_92_exec-bit-sweep.md
tags: [command, post-review, review-artifact, sha-bound, merge-gate, workflow]
---

# /coderails:post-review

Added by [[pr_81-83_review-artifact-seam]] (PR #82/83, merged 2026-06-30). Posts a machine-marked, SHA-bound review summary as a durable GitHub PR comment. Creates the artifact that `/coderails:merge` verifies before merging.

## Invocation

```
/coderails:post-review <PR#>
```

Run immediately after `/pr-review-toolkit:review-pr` completes, before `/coderails:merge`.

## What it does

Seven sequential steps: (verified: `commands/post-review.md`)

1. **Write summary** — agent writes findings from the just-completed `review-pr` run to a temp file. Anti-fabrication rule: if `review-pr` found nothing, write `## No findings`; never invent placeholders.
2. **Validate** — runs `./scripts/post_review.sh validate <file>`. If validation fails, **abort** — do not post. (See grammar below.)
3. **Resolve head SHA** — `gh pr view <PR#> --json headRefOid -q .headRefOid`.
4. **Build marker** — sources `scripts/lib/review-artifact.sh` (the SSOT), builds the marker line via `review_artifact::marker <PR#> <sha>`. Prepends marker to summary body.
5. **Post via `gh api`** (not `gh pr comment`) — uses `repos/<owner>/<repo>/issues/<PR#>/comments` endpoint to capture the returned URL, comment ID, author, and creation time as metadata.
6. **Best-effort cache write** — resolves progress.json path via `agentic_loop_path.sh`; writes `progress.json.review` block (URL, id, sha). A missing progress.json is not an error — the PR artifact is the authority.
7. **Report** — prints the posted comment URL.

## Grammar rule (validated by `post_review.sh validate`)

The summary body must satisfy one of:

- A bare line `## No findings` (clean review with no findings), OR
- All three headings **in order**: `## Critical`, `## Important`, `## Suggestions`, each followed by at least one bullet (`- …`) or the literal line `None` if that section is empty.

Any other structure fails validation and blocks posting. This prevents completely hollow reviews from creating an artifact the merge gate then accepts. (verified: `scripts/post_review.sh`)

## Marker format

The marker is the first line of the posted comment:

```
<!-- coderails-review-summary v1 pr=<N> head_sha=<sha> -->
```

Constructed by `review_artifact::marker()` in `scripts/lib/review-artifact.sh` — the same function `pr::has_coderails_review_for_head` in `git-common.sh` uses for matching. One SSOT, no drift.

## Scripts used

- `scripts/lib/review-artifact.sh` — marker SSOT (sourced, not run)
- `scripts/post_review.sh` — `validate` and `write-cache` subcommands
- `hooks/scripts/lib/agentic_loop_path.sh` — resolves progress.json path (best-effort, not required)

## Chain position

Sits between `review-pr` and `/merge` in [[workflow]] Phase 3 (and [[agentic-loop]] Phase 4b):

```
/pr-review-toolkit:review-pr  →  /coderails:post-review  →  /coderails:merge
```

Without a valid artifact on the current head SHA, [[merge]] blocks with an actionable error message pointing to this command.

## Preconditions

- `gh` on PATH and authenticated
- Remote must be a `github.com` repository
- Must run in the same session as the `review-pr` it summarises (anti-fabrication depends on in-session context)

## Honest ceiling

This command validates structure, not provenance. A cooperating-but-shallow reviewer can write a structurally-valid summary that passes validation without substantive findings. The grammar check is a floor against completely hollow artifacts, not a proof of review quality. See [[review-artifact-seam]] and [[enforcement-model]].

## See also

- [[review-artifact-seam]] — the design page for the full truth-seam architecture
- [[merge]] — the command that gate-checks the artifact before merging
- [[workflow]] — the orchestrator that chains review-pr → post-review → merge
- [[agentic-loop]] — Phase 4b, which adds post-review for loop symmetry
- [[enforce_pr_workflow]] — the predecessor transcript-based gate
