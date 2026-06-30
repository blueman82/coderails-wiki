---
title: "PRs #81–83 — Independent-review truth seam: SHA-bound artifact + fail-closed merge gate"
type: source
created: 2026-06-30
last_updated: 2026-06-30
sources:
  - docs/coderails/specs/2026-06-30-progress-record-design.md
  - docs/coderails/plans/2026-06-30-progress-record-plan.md
  - commands/post-review.md
  - scripts/lib/review-artifact.sh
  - scripts/post_review.sh
  - scripts/lib/git-common.sh
  - scripts/merge.sh
  - commands/workflow.md
  - skills/agentic-loop/SKILL.md
  - commands/prep.md
  - AGENTS.md
tags: [source, review-artifact, merge-gate, post-review, truth-seam, sha-bound, workflow, agentic-loop]
---

# PRs #81–83 — Independent-review truth seam

A three-PR cluster delivering one design theme: move review truth from ephemeral chat output to a GitHub-visible, SHA-bound PR comment that `/merge` verifies fail-closed.

## PR metadata

| Field | Value |
|---|---|
| PR #81 | `Add progress-record design spec and implementation plan` — merged 2026-06-30T15:05Z |
| PR #82 | `feat: review-artifact engine — core artifact + merge gate (Tasks 0–5)` — merged 2026-06-30T15:33Z |
| PR #83 | `feat: wire /coderails:post-review into workflow + docs (PR-B)` — merged 2026-06-30T15:50Z |
| Branch cluster | `feature/review-artifact-engine` (PR #82) + `feature/post-review-wiring` (PR #83) |
| Related pages | [[review-artifact-seam]], [[post-review]], [[merge]], [[workflow]], [[agentic-loop]] |

---

## PR #81 — Design spec + implementation plan

**Files:** `docs/coderails/specs/2026-06-30-progress-record-design.md` (389 lines), `docs/coderails/plans/2026-06-30-progress-record-plan.md` (197 lines).

Committed 5 rounds of design review and a planning-sequence stress-test. Key decisions locked in the spec:

- Review truth moves from local `review.ran` boolean (rejected: doer-writable, ephemeral, not independently auditable) to a GitHub-visible SHA-bound PR comment.
- `progress.json.review` is a **cache/index** of the artifact, never the authority.
- `/merge` gates on GitHub reality (live fetch of PR comments), not any local file.
- The Stop guard enforces **presence + ownership + terminal status** of the progress record, NOT review truth. Truth lives in the PR artifact.
- Universal Stop guard (`progress_record_guard.sh`) and universal-ledger enforcement on non-loop runs were **deferred** — the planning-sequence stress-test found they would reintroduce ceremony tax without proportional benefit. The `/prep` stub is optional/non-blocking.

---

## PR #82 — Engine: core artifact + merge gate (Tasks 0–5)

**Files:** `scripts/lib/review-artifact.sh` (new), `scripts/post_review.sh` (new), `commands/post-review.md` (new), `scripts/lib/git-common.sh` (extended), `scripts/merge.sh` (extended), plus 4 test suites (179+181+61+53 lines).

### Task 0 — `scripts/lib/review-artifact.sh` (marker SSOT)

The marker is defined once and sourced by both writer (`/coderails:post-review` via `post_review.sh`) and reader (`/merge` gate via `git-common.sh`) — one constructor, no drift. (verified: review-artifact.sh)

```
<!-- coderails-review-summary v1 pr=<N> head_sha=<sha> -->
```

`review_artifact::matches_marker` uses **exact string equality**, not substring grep. A line with any junk prefix or suffix fails. An unknown/future version never matches (fail-closed). (verified: review-artifact.sh:17–19)

### Tasks 1–2 — `scripts/post_review.sh`

Two subcommands: `validate` (anti-placeholder grammar check) and `write-cache` (temp-file+mv to progress.json, warns+returns-0 if path absent).

**Grammar rule:** the summary body must satisfy one of:
- A line `## No findings` (for a clean review), OR
- All three headings in order (`## Critical`, `## Important`, `## Suggestions`), each followed by at least one bullet or the literal `None`.

This prevents hollow reviews from passing the gate without blocking correct "nothing found" results.

### Task 3 — `commands/post-review.md`

New `/coderails:post-review <PR#>` command. Posts via `gh api` (not `gh pr comment`) to capture URL and metadata. Anti-fabrication instruction: write `## No findings` if review-pr found nothing; never invent placeholders. (verified: commands/post-review.md)

### Task 4 — `scripts/lib/git-common.sh` additions

Two new functions sourcing `review-artifact.sh` via the BASH_SOURCE idiom:

- `pr::head_sha <num>` — fetches current `headRefOid` for the PR (empty on gh failure)
- `pr::has_coderails_review_for_head <num> <sha>` — scans all PR comment bodies line-by-line for an exact marker match; distinct exit codes: 0=match, 1=no-match, 2=fetch-failed

The distinct exit codes are load-bearing: `merge.sh` treats exit 2 (fetch-failed) as a different error message from exit 1 (no artifact), preserving actionable diagnostics.

### Task 5 — `scripts/merge.sh` gate

The merge gate runs **before `gh pr merge`** in the OPEN branch:

1. Fetch current head SHA via `pr::head_sha` — if empty, block with "GitHub fetch failed" message.
2. Call `pr::has_coderails_review_for_head` — exit 2 → "GitHub fetch failed"; exit 1 → "No coderails review artifact for current head — run /coderails:post-review".
3. Only if exit 0: proceed to `gh pr merge`.

No progress.json fallback. No local-file escape path. (verified: merge.sh @ 503f6fa)

Test suites: `review-artifact.test.sh`, `post_review.test.sh` (19 cases), `git-common.test.sh`, `merge.test.sh` (two distinct failure messages tested). Suite: 21/21 suites pass.

---

## PR #83 — Wiring: workflow + docs (Tasks 6–8)

**Files:** `commands/workflow.md`, `skills/agentic-loop/SKILL.md`, `commands/prep.md`, `AGENTS.md`, `docs/REFERENCE.md`, `README.md`.

### Task 6 — Workflow + agentic-loop wiring

`commands/workflow.md` Phase 3: inserted `/coderails:post-review <PR#>` after `review-pr`/simplify, before the Phase 4 ship-it pause. Chain now reads:

```
review-pr → post-review → (Phase 4 ship-it pause) → /merge
```

`skills/agentic-loop/SKILL.md` Phase 4b: added the matching post-review step (loop symmetry — same artifact gate applies whether the session is autonomous or interactive). (verified: SKILL.md @ 23194b3)

### Task 7 — Optional `/prep` ledger stub

`commands/prep.md` Part 1b: non-blocking. Writes a `progress.json` stub via `agentic_loop_path.sh`. Failure here does NOT abort `/prep`. This is the deferred universal-trace — present but optional. (verified: prep.md @ 23194b3)

### Task 8 — Documentation

`AGENTS.md` gains: `/coderails:post-review` in the workflow command architecture table and skills↔hooks seam section; enforcement-ceiling note (validates structure, not provenance; auditable not cryptographic); follow-up ordering constraint (see below). `REFERENCE.md` and `README.md` gain command table entries for `/coderails:post-review`.

---

## Key design decisions (the non-obvious stuff)

### The truth move: from local boolean to GitHub artifact

The rejected alternative was a doer-writable local `review.ran` boolean in `progress.json`. Three problems: (1) the agent both writes it and is judged on it with no privilege boundary; (2) it is local and ephemeral — no human can audit it post-session; (3) it does not carry SHA binding so a review on an earlier push would satisfy it.

The chosen approach: truth lives on GitHub as a SHA-bound PR comment. The `/merge` gate fetches from GitHub directly — it cannot be spoofed by editing any local file. (inferred: design spec)

### The honest ceiling

The gate proves a durable, SHA-bound, auditable artifact **exists**. It does NOT prove the review was substantive. The grammar validator catches completely hollow reviews (no headings, placeholder text) but cannot assess whether a reviewer genuinely found the right things. This is explicitly documented in `AGENTS.md` as an enforcement ceiling. (verified: AGENTS.md @ 23194b3)

### `progress.json.review` is cache only

The `write-cache` subcommand in `post_review.sh` records the artifact URL, comment ID, and head SHA in `progress.json.review` for observability and idempotency — not as the source of truth. The `/merge` gate never reads `progress.json`; it goes to GitHub. (verified: scripts/post_review.sh, scripts/merge.sh)

### Marker SSOT lib overrides spec bias

The design spec noted a "no new lib" bias, but `review-artifact.sh` was introduced anyway. Rationale: writer and reader must use identical marker construction and matching logic. Without a shared lib, two independent implementations diverge over time. The lib is intentionally minimal (25 lines, one file, one version constant, two functions). (inferred: PR #82 body)

### The ordering constraint on `enforce_pr_workflow` demotion

`enforce_pr_workflow`'s review-pr transcript arm is expected to demote from block to nudge once this gate is live and verified — because the durable artifact provides a stronger, independent guarantee. But this demotion MUST NOT happen before the gate is verified in production. A no-enforcement window between demotion and verification would allow unreviewed PRs to merge. (verified: AGENTS.md @ 23194b3)

### Deferred scope

The planning-sequence stress-test ruled out two originally-designed items:
1. **Universal progress.json Stop guard** (`progress_record_guard.sh`) — would reintroduce ceremony tax on every non-loop run. Deferred.
2. **Universal-ledger enforcement** — same reason. The `/prep` stub is optional/non-blocking.

---

## What changed in the system

| Component | Before PRs #81–83 | After |
|---|---|---|
| Review truth location | Ephemeral chat output only | SHA-bound GitHub PR comment |
| `/merge` gate | GitHub branch protection + `enforce_pr_workflow` transcript check | + review artifact gate (exact SHA match, fail-closed) |
| `/coderails:post-review` | Did not exist | New command: validates grammar, posts artifact, caches in progress.json |
| `scripts/lib/review-artifact.sh` | Did not exist | Marker SSOT (25 lines) |
| `scripts/post_review.sh` | Did not exist | validate + write-cache subcommands |
| `scripts/lib/git-common.sh` | pr::num, pr::state, pr::review, pr::title, pr::url | + pr::head_sha, pr::has_coderails_review_for_head (exit codes 0/1/2) |
| Phase 3 of `/workflow` | review-pr → ship-it pause | review-pr → post-review → ship-it pause |
| Phase 4b of agentic-loop | invoke review-pr | + invoke post-review after review-pr |
| `/prep` | Worktree + Jira only | + optional Part 1b: progress.json stub (non-blocking) |

---

## See also

- [[review-artifact-seam]] — design page for this architectural decision
- [[post-review]] — command page for `/coderails:post-review`
- [[merge]] — updated with the review artifact gate
- [[workflow]] — updated with post-review in Phase 3
- [[agentic-loop]] — updated with post-review in Phase 4b
- [[enforce_pr_workflow]] — the predecessor gate (transcript-based); demotion is a follow-up
- [[spec-plan-progress-artifact-chain]] — predecessor artifact chain design
