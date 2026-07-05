---
title: "PR 92 — executable-bit sweep (scripts/post_review.sh permission-denied fix)"
type: source
created: 2026-07-03
last_updated: 2026-07-05
sources:
  - scripts/post_review.sh
  - commands/post-review.md
  - hooks/scripts/tests/run_all.sh
  - sources/pr_96-98_mode-aware-install-argument-injection-guard-hook-owned-counter.md
tags: [source, exec-bit, file-mode, post-review, hardening]
---

# PR 92 — executable-bit sweep (scripts/post_review.sh permission-denied fix)

## PR metadata

| Field | Value |
|---|---|
| PR number | #92 |
| Merged | 2026-07-03 |
| Merge SHA | `4cb38b5` (`4cb38b54aa83aef0e9359f5ff348f743b934014d`) |
| JIRA ticket | — |

## Summary

`scripts/post_review.sh` was mode `100644` (not executable) in the git index while
[[post-review]] (`commands/post-review.md:48` and `:125`) invokes it as a direct
path — `./scripts/post_review.sh validate ...` and `./scripts/post_review.sh
write-cache ...`. This produced a real permission-denied failure, observed when a
review artifact briefly posted with unvalidated content because step 2 (validate,
[[post-review]]'s abort-on-fail gate) couldn't execute. The PR flips the mode bit
to `100755` for that one file, plus normalises 8 already-behaviourally-fine test
files (`hooks/scripts/tests/*.test.sh`) from `100644` to `100755` for consistency
with their 17 already-755 peers — those 8 are invoked via `bash "$test_file"` in
`run_all.sh`, so the mode bit wasn't strictly required there, only cosmetic
drift. Zero content changes; every touched blob is byte-identical pre/post
(reviewer-verified via `git diff --stat` showing only mode changes).

## Files changed (mode only, 100644 → 100755)

| File | Why it mattered |
|---|---|
| `scripts/post_review.sh` | **Real bug.** Invoked as `./scripts/post_review.sh` (direct path exec), not sourced or `bash`-wrapped |
| `hooks/scripts/tests/cli_antipatterns.test.sh` | Cosmetic — invoked via `bash "$test_file"` |
| `hooks/scripts/tests/config.test.sh` | Cosmetic |
| `hooks/scripts/tests/hooks_json_timeout_floor.test.sh` | Cosmetic |
| `hooks/scripts/tests/merge.test.sh` | Cosmetic |
| `hooks/scripts/tests/post_review.test.sh` | Cosmetic |
| `hooks/scripts/tests/post_review_command.test.sh` | Cosmetic |
| `hooks/scripts/tests/review-artifact.test.sh` | Cosmetic |
| `hooks/scripts/tests/stdin_bounded_read.test.sh` | Cosmetic |

## Audited, left at 100644 (correct as-is — do not re-flag)

- `hooks/scripts/lib/discipline_common.sh`, `scripts/lib/config.sh`,
  `scripts/lib/review-artifact.sh` — only ever `source`d, never exec'd as a path.
- `install.sh`, `uninstall.sh` — always invoked as `bash install.sh` /
  `bash uninstall.sh` per README.md/INSTALLATION.md, never as a direct path.
  (verified: grep of README.md/AGENTS.md/INSTALLATION.md finds no claim that
  these lack exec bits or must be run via `bash` *because* of file mode — the
  existing "invoked via `bash`" language is just the documented calling
  convention, already consistent with the audit.)

## Wiki pages updated

- [[post-review]] — added an exec-bit caveat under Preconditions
- [[review-artifact-seam]] — added a one-line note that `post_review.sh`'s
  executability is now a codified invariant, not an accident

## Caveats / gotchas

- This is the second time this repo's git history has needed an exec-bit fix
  (distinct from any prior install.sh chmod-derivation work referenced in
  [[self-containment]] — that concerned files `install.sh` sets executable at
  install time, not files' modes in the source tree). No test in
  `hooks/scripts/tests/run_all.sh` currently asserts `scripts/post_review.sh`
  carries the executable bit — a regression (e.g. a future `git update-index
  --chmod=-x`, or a Windows checkout re-adding the file) would reintroduce the
  same permission-denied failure silently until a live `/post-review` run hit
  it. No automated guard was added in this PR; flagged here as a known gap,
  not fixed.
  **Closed same day by [[pr_93-94_post-review-injection-and-exec-bit-invariant|PR #94]]**
  — `exec_bit_invariant.test.sh` now guards this mode bit (and every other
  tracked script's) against regression.
- PR #94's own source page separately flagged a second, distinct gap:
  `install.sh`'s unconditional post-install chmod sweep re-adds `+x` on disk at
  install time regardless of git index mode — a drift between the invariant
  test (checks the index) and the installer (ignores it).
  **Closed by [[pr_96-98_mode-aware-install-argument-injection-guard-hook-owned-counter|PR #96]]**
  (merged 2026-07-03, 127a149) — the sweep is now index-mode-aware
  (`100755`→`+x`, `100644`→`-x`, untracked→legacy `+x`), with a fix round
  closing a Critical the mode-read introduced (installer died silently on
  non-git checkouts under `set -euo pipefail`).
