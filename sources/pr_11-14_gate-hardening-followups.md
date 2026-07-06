---
title: "PRs #11–14 (blueman82/coderails) — gate-hardening follow-ups: explicit NO-GO at every tier, push.sh staging fix, HOME-sandboxed install test, trust-floor permission check + merge.sh error split"
type: source
created: 2026-07-06
last_updated: 2026-07-06
sources: []
tags: [task-evals, evals-json, loop-gate, push, staging, install-sh, home-sandbox, trust-floor, comment-spoofing, viewerPermission, merge-gate, error-messages]
---

# PRs #11–14 — gate-hardening follow-ups

<!-- Ingested by /wiki-ingest after merge. This is an immutable record of what changed. -->

## PR metadata

| Field | Value |
|---|---|
| PR numbers | #11, #12, #13, #14 |
| Repo | `blueman82/coderails` |
| Branches | `task-followups/wu2-tier0-nogo`, `task-followups/wu3-test-home-sandbox`, `task-followups/wu1-push-staging`, `task-followups/wu4-wu5-trust-and-errors` |
| Base | `238f5e1` (PR #10, prior cluster) |
| Merged | 2026-07-06, sequentially (all same day) |
| Merge SHAs | #11 `a1c4803`, #12 `fdd29fd`, #13 `321bca3`, #14 `7c1dd19` (final `origin/main`) |
| JIRA ticket | — |

## Summary

Four independent work-units resolving owner-flagged follow-ups from the gate-hardening backlog: a loop-evals semantics gap (explicit `NO-GO` losing to the tier-0 exemption), a real recurring test-corruption bug (an unsandboxed install test rewriting the developer's actual `~/.claude/settings.json`), a staging-safety gap in `/coderails:push` (`git add -A` silently including untracked files), and a security-relevant trust-floor widening (the review/eval artifact gates' `OWNER`-association requirement replaced with a repo-permission check so the gates work on org-owned repos) plus a merge-error-message split (identity vs. permission vs. comments-fetch failure now produce distinct, actionable text). Each PR needed both a review artifact and an eval artifact before merge — the gate dogfooding its own extension, same discipline as the [[pr_7-10_task-evals-followups]] cluster before it.

## PR #11 — explicit NO-GO wins at every tier, including tier 0 (WU2, merge `a1c4803`)

`hooks/scripts/lib/loop_state_common.sh`'s `als_read_loop_evals_result()` gained a new branch, checked **before** the tier-0 exemption:

```sh
elif [ "$result" = "GO" ]; then ALS_LOOP_EVALS_RESULT="GO"
elif [ "$result" = "NO-GO" ]; then ALS_LOOP_EVALS_RESULT="NO-GO"
elif [ "$tier" = "0" ]; then ALS_LOOP_EVALS_RESULT="TIER0"
else ALS_LOOP_EVALS_RESULT="NO-GO"
```

Before this fix, a tier-0 `evals.json` with `result: "NO-GO"` explicitly recorded would still read `TIER0` (allow) — the tier-0 branch fired unconditionally on any tier-0 artifact regardless of what `result` said, because the old `elif` chain checked `tier == "0"` before ever inspecting `result` for the NO-GO case. Owner directive, stated inline in the source comment: **"an exemption justifies having no evals, not overriding a recorded failure."** A tier-0 artifact that never set `result` at all (the legitimate exemption case — no evals were run, tier justifies why) still correctly reads `TIER0`; only an artifact that *explicitly* recorded `NO-GO` now blocks like any other tier. (verified: `git diff 238f5e1..a1c4803 -- hooks/scripts/lib/loop_state_common.sh`)

The consumer, `loop_state_guard.sh`'s `gate_loop_evals_required`, is unchanged — it already treated any non-`GO`/non-`TIER0` result as block-worthy; the fix is entirely in what `als_read_loop_evals_result` returns.

Test coverage: `hooks/scripts/tests/loop_state_guard_evals.test.sh` extended (new cases for tier-0-with-explicit-NO-GO). Full evals-reader test file (`git-common.test.sh`'s sibling suite) reaches 29 cases total across the loop-evals reader.

## PR #12 — HOME-sandboxed install_mode_sweep.test.sh (WU3, merge `fdd29fd`)

**The bug this fixes was live and recurring during this very loop.** `hooks/scripts/tests/install_mode_sweep.test.sh` invokes the real `install.sh` non-interactively to verify its git-index-mode-aware exec-bit sweep (added by [[pr_96-98_mode-aware-install-argument-injection-guard-hook-owned-counter|PR #96]]). Before this fix, the test set `MEMORY_TARGET` to redirect memory-seeding writes into the temp tree, but `install.sh` also writes **unconditionally under `$HOME`** — `installed_plugins.json` scan, `~/.claude/commands` conflict scan, `settings.json`/`known_marketplaces.json` marketplace registration, `~/.claude/CLAUDE.md` append — none of which `MEMORY_TARGET` redirects. Because this repo's own `test_gate` `PreToolUse` hook runs the full test suite (`run_all.sh`) on **every** `git commit`, every worker commit made inside a worktree carrying the unfixed test corrupted the real developer machine's `~/.claude/settings.json` — 10 recorded events during this loop, all auto-repaired, but a real and repeatable corruption, not a one-off. Instruction-level "don't touch that file" exclusions given to workers were unenforceable, because the corruption happened inside a test the hook itself invoked outside worker awareness; the only real fix was sandboxing the test.

Fix: `HOME` is redirected to a freshly-`mktemp`'d sandbox (`$TMP_TREE/.home-sandbox`) for the duration of both `install.sh` invocations in the test (the git-tracked-tree run and the no-git-tree run). Anti-vacuity discipline — the sandbox isn't just proven inert, it's proven to be the actual mutation target:

- Pre-seeds the sandboxed `settings.json` and `known_marketplaces.json` with a stale `workflow-tools` marketplace key before the run, so the test exercises `install.sh`'s real `jq` stale-key-drop + coderails-registration mutation logic against the sandbox, not a no-op.
- Leaves the sandboxed `CLAUDE.md` absent before the run, covering the "file doesn't exist yet" append path (as opposed to the idempotent-append-if-present path).
- Before/after `cksum` guards on **all three** real-`$HOME` files the installer would otherwise touch (`settings.json`, `known_marketplaces.json`, `CLAUDE.md`) — asserted untouched (identical checksum) across both install.sh invocations in the test.
- Positive assertions that the sandbox mutation actually happened: stale key dropped, coderails path registered pointing at the sandbox tree, `CLAUDE.md` gains the `## Self-Checking Discipline` section.

`install.sh` makes no `gh` calls, so no auth/token state under `$HOME` was needed for the redirected runs (only `jq`/`git` on PATH). Suite: 24→25 (unchanged from this test's own count; the 29→30 net file-count bump for the whole cluster comes from PR #13's new `push_staging.test.sh`, not this PR).

## PR #13 — push.sh staging: `git add -A` → `git add -u` + untracked-files warning (WU1, merge `321bca3`)

`scripts/push.sh`'s commit step previously staged **everything** unconditionally (`git add -A`), silently including any untracked file in the working tree at commit time — a staging-safety gap for a PR that's supposed to touch only the files the branch's work concerns. Fixed to `git add -u` (tracked changes only), with an explicit warning listing any untracked files by name so the developer knows to `git add` them by hand if a new-file PR is intended:

```sh
git add -u
local untracked; untracked=$(git status --porcelain | grep '^??' | cut -c4- || true)
if [[ -n "$untracked" ]]; then
    warn "Untracked files not staged (run 'git add' explicitly to include them):"
    while IFS= read -r f; do warn "  $f"; done <<< "$untracked"
fi
if [[ -n $(git diff --cached --name-only) ]]; then
    ...commit...
fi
```

**Review-caught crash, fixed pre-merge:** the initial version crashed when staging a tracked-only change with zero untracked files present — `git status --porcelain | grep '^??'` under `pipefail` returns exit 1 when grep matches nothing, and without a `|| true` guard that propagated as a script failure. Fixed by appending `|| true` to the pipe (see the `git status --porcelain | grep '^??' | cut -c4- || true` line above — this is the post-fix form). The commit history shows this as a same-day follow-up commit ("Fix crash when staging a tracked-only change (no untracked files)", `865cab0`) after the initial "Address review findings" pass (`3055c0a`).

The commit block itself is also now conditional on `git diff --cached --name-only` being non-empty — a tracked-only push with no staged changes after `add -u` (e.g. only untracked files present, none force-added) no longer produces an empty/failing commit attempt.

New coverage: `hooks/scripts/tests/push_staging.test.sh` — first-ever dedicated staging test for `push.sh`, asserting tracked changes stage and commit, untracked files are warned-about and NOT staged, and the tracked-only-no-untracked path doesn't crash. `docs/REFERENCE.md`'s `scripts/push.sh` row updated to describe `git add -u` + the untracked-file warning instead of `git add -A`. Suite test-file count: 29→30 (this is the first new test file in the cluster; PRs #11/#12/#14 extended existing files).

## PR #14 — trust-floor: repo write-permission replaces OWNER-badge; merge.sh error-message split (WU4+WU5, merge `7c1dd19`)

Two independent fixes shipped in one PR, both touching `scripts/lib/git-common.sh` and `scripts/merge.sh`.

### WU4 — trust floor widened: OWNER-association conjunct removed, replaced with a permission check

**Security-relevant change, clean break with no compat path.** [[pr_7-10_task-evals-followups|PR #8]] had closed the review/eval artifact gates' comment-spoofing hole with a two-part trust rule: the comment author's login must match the `gh`-authenticated identity, **AND** the comment's `author_association` must be `OWNER`. That page's own known-gaps note flagged the scoped limitation explicitly: `OWNER` assumes a personally-owned repo; the same authenticated user's own comments on an **org-owned** repo carry `MEMBER`/`COLLABORATOR` instead, so the gate would fail closed there — a real usability gap for any org deployment, not a bug in the anti-spoof logic itself.

This PR removes the `author_association == "OWNER"` conjunct entirely and replaces it with a **repo-permission check**:

- New `pr::_trusted_permission()` — echoes the authenticated identity's `viewerPermission` on the current repo via `gh repo view "$(repo)" --json viewerPermission -q .viewerPermission`. Same per-subshell (not per-process) reuse-guard shape as the pre-existing `pr::_trusted_login()`. Returns non-zero (fail-closed) if the lookup itself fails.
- New `pr::_permission_is_write_or_better()` — `ADMIN`/`MAINTAIN`/`WRITE` pass; `READ`/`TRIAGE`/anything else does not.
- `pr::_trusted_comment_bodies()` now requires **both** conjuncts: login match (unchanged — this is the actual anti-spoof property, still fires regardless of permission level) **AND** `pr::_permission_is_write_or_better` on the looked-up permission. A successfully-resolved-but-insufficient permission (e.g. `READ`) is explicitly **not** treated as a fetch failure — the function echoes nothing and returns 0, so the caller sees the ordinary "no trusted comment found" path (exit 1 upstream), not a fail-closed exit 2. Only an actual API-call failure (identity fetch, permission fetch, or the comments fetch itself) is fail-closed.

New identity/reason-plumbing to support distinct error messages (see WU5 below): `pr::_trusted_comment_bodies` now emits a `TRUST_FETCH_FAIL_REASON=identity|permission|comments` line to stderr on failure (can't be a plain variable — every call site invokes it via `$(...)`, and a subshell assignment never survives back to the caller). A new wrapper, `pr::_trusted_comment_bodies_or_fail()`, recovers that stderr-carried reason into `PR_TRUST_FETCH_FAIL_REASON` in the **caller's own shell** (via a `mktemp` stderr-capture file, not a subshell) — both public readers (`pr::has_coderails_review_for_head`, `pr::has_coderails_eval_for_head`) now call this wrapper instead of the raw function, so the reason-capture plumbing lives in exactly one place. If `mktemp` itself fails, the wrapper sets reason `tempfile` and fails closed rather than silently losing the distinction.

**Independently verified: no surviving compat path.** The `OWNER` string and the `author_association` field are gone from `git-common.sh` entirely post-merge — this is a full replacement, not an additive relaxation sitting alongside the old check.

`INSTALLATION.md` gained a new bullet under Notes documenting the rule for operators: artifact comments are trusted only from the merging identity, which must hold write access on the repo — identical behaviour on personal and org-owned repos now. The bullet is explicit that this doesn't change what artifact comments *assert* — a compromised identity with write access can still post a false artifact; the gate proves *who* posted a marker, not that its contents are truthful (same honest-ceiling framing as [[review-artifact-seam]] and [[task-evals-gate]]).

### WU5 — merge.sh: distinct error messages for identity vs. permission vs. comments-fetch failure

Before this PR, both gates in `scripts/merge.sh` (review-artifact gate and eval-artifact gate) collapsed every `gh`-fetch failure (exit code 2 from the reader) into one generic message: `"GitHub fetch failed — could not fetch PR comments. Retry, or check gh auth/network."` Now each gate's exit-2 branch reads `PR_TRUST_FETCH_FAIL_REASON` (set by WU4's `pr::_trusted_comment_bodies_or_fail`) and produces one of three distinct messages:

```sh
case "${PR_TRUST_FETCH_FAIL_REASON:-}" in
    identity)   err "... could not resolve the authenticated identity (gh api user) ..." ;;
    permission) err "... could not resolve repo permission for the authenticated identity ..." ;;
    *)          err "... could not fetch PR comments ..." ;;  # comments/tempfile/unset fall through to the original generic message
esac
```

Applied identically to both the review-gate and eval-gate exit-2 branches in `merge.sh` (the eval-gate message additionally names "for the eval artifact gate" to distinguish which of the two stacked gates failed). The public reader exit-code contract (0=match, 1=no-match/NO-GO, 2=fetch-failed) is unchanged — this is purely a diagnostic-message refinement on the existing exit-2 path, not a new code path.

## Files changed (full cluster, base `238f5e1` → final `origin/main` `7c1dd19`)

`hooks/scripts/lib/loop_state_common.sh`, `scripts/lib/git-common.sh`, `scripts/merge.sh`, `scripts/push.sh`, `INSTALLATION.md`, `docs/REFERENCE.md`, plus test files: `hooks/scripts/tests/git-common.test.sh`, `hooks/scripts/tests/install_mode_sweep.test.sh`, `hooks/scripts/tests/loop_state_guard_evals.test.sh`, `hooks/scripts/tests/merge.test.sh`, `hooks/scripts/tests/merge_evals_gate.test.sh`, `hooks/scripts/tests/push_staging.test.sh` (new). 12 files, 643 insertions / 86 deletions total. (verified: `git diff --stat 238f5e1..origin/main`)

## Wiki pages updated

- [[task-evals-gate]] — new "Known gaps" closure note (trust floor now permission-based, not OWNER-badge); loop-evals block-condition wording tightened for explicit-NO-GO-at-tier-0
- [[review-artifact-seam]] — comment-spoofing section updated: OWNER-association replaced with permission check; scoped-limitation caveat removed (no longer applies)
- [[loop_state_guard]] — block-condition wording for the eval gate updated to name explicit-NO-GO-wins-at-tier-0
- [[merge]] — new error-message-split subsection under the artifact gates
- [[push]] — staging behaviour updated from `git add -A` to `git add -u` + untracked-file warning

## Caveats / gotchas

- The install-test HOME-sandbox fix (PR #12) closes a real, repeatedly-triggered corruption of the developer's actual `~/.claude/settings.json` during this very loop (10 auto-repaired events) — worth remembering as a cautionary example: a test that shells out to a real installer script needs every environment variable the installer writes through sandboxed, not just the one the test author thought of first (`MEMORY_TARGET` alone wasn't enough; `$HOME` itself also needed redirecting).
- The trust-floor change (PR #14 WU4) is a clean break, not additive — verify this before assuming any org-repo `OWNER`-based logic still exists anywhere in `git-common.sh`. See [[trust-floor]] for the consolidating concept page (SSOT for the mechanism itself remains [[merge]]).
- Reviewer-surfaced follow-ups not yet shipped (suggestion-tier, recorded here so they aren't lost): `push_staging.test.sh` coverage for a pre-staged-new-file case and a deleted-tracked-file case; a `loop_state` test case for `tier != 0` + justified + `result` absent; an `install_mode_sweep` NOGIT-run pre-seed case; a `merge.sh` test asserting the `tempfile` reason's exact error-branch message.
