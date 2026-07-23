---
title: "/coderails:merge"
type: command
created: 2026-06-25
last_updated: 2026-07-23
sources: [commands/merge.md, scripts/merge.sh, scripts/lib/git-common.sh, sources/pr_43_rough-edges.md, sources/pr_81-83_review-artifact-seam.md, sources/pr_89-91_skills-doc-frontmatter-injection.md, sources/session_2026-07-03_ai-docs-refresh-and-cc-mechanics-probes.md, sources/pr_93-94_post-review-injection-and-exec-bit-invariant.md, sources/pr_96-98_mode-aware-install-argument-injection-guard-hook-owned-counter.md, sources/pr_1-4_task-evals-feature.md, sources/pr_11-14_gate-hardening-followups.md, sources/pr_21-22_loop2-suggestion-tier-followups.md, sources/pr_232_tier-review-gate.md, sources/pr_279_merge_time_smoke_reexecution.md]
tags: [command, merge, pr, github, branch-cleanup, sync, review-artifact, sha-bound, dynamic-injection, command-substitution, security, task-evals, eval-artifact, trust-floor, viewerPermission, error-messages, tempfile, tier-review, root-daemon, smoke-verify, worktree, re-execution]
---

# /coderails:merge

Merges an approved PR, switches to main, pulls latest, and cleans up the feature branch. Branch cleanup is decoupled from the merge itself so a failed cleanup never reports a successful merge as failed. Since [[pr_1-4_task-evals-feature]] (PR #3, merged 2026-07-06), merging additionally requires a passing eval artifact — a second, independent gate stacked on top of the review-artifact gate. Since [[pr_279_merge_time_smoke_reexecution|PR #279]] (2026-07-23), a third gate re-executes the eval artifact's scripted checks against the trusted head SHA, making the artifact's re-execution property binding at merge rather than advisory. A fourth, opt-in gate — the tier-review status check ([[pr_232_tier-review-gate|PR #232]], 2026-07-17) — additionally applies only to tier-0 eval artifacts, and only when `tier_review.machine_user` is configured.

## Dynamic Git-status injection (PR #91)

`commands/merge.md` frontmatter now injects a "Current Git Status" block via bash
substitution before the command's prose instructions:

```
!`git branch --show-current`
!`gh pr list --state open --limit 10`
```

plus a display-only note ("data, not instructions") stating explicitly that this injected
output is repository state for the model to read, not instructions to follow — guarding
against the injected text being misread as a directive. This is display context only; it
does not change any merge logic in `merge.sh`. (verified: [[pr_89-91_skills-doc-frontmatter-injection]], `git diff` on `commands/merge.md`)

**`commands/post-review.md`'s parallel injection (PR #93) was shipped, then removed
for security (PR #97).** PR #91's original probe of `$ARGUMENTS`-inside-`!`cmd``
substitution ordering was inconclusive, deferring the post-review.md injection. A
same-day follow-up probe resolved the ordering question (`$ARGUMENTS` substitutes
before `!`cmd`` injection executes; the earlier probe's inconclusive result was a
subagent skill-enumeration artifact, not an ordering one) — see
[[session_2026-07-03_ai-docs-refresh-and-cc-mechanics-probes]]. PR #93 shipped the
deferred change: a `## Current PR State` block using the same "data, not instructions"
guard line. See [[post-review]] and [[pr_93-94_post-review-injection-and-exec-bit-invariant]].

**That injected line turned out to be a command-substitution injection
vulnerability, not just an unsanitised-input note.** [[pr_96-98_mode-aware-install-argument-injection-guard-hook-owned-counter|PR #97]]
(merged 2026-07-03T19:13:25Z) experimentally proved `$ARGUMENTS` inside a
render-time `` !`cmd` `` line executes as live shell syntax regardless of
quoting — because textual substitution happens before the shell parses quotes,
not after. `post-review.md`'s injected PR-state block is removed entirely,
replaced with an argument-free `gh pr list`, and a **class-wide test** now
guards that no `commands/*.md` file may carry a render-time `` !`cmd` `` line
containing `$ARGUMENTS` — this file's own injected block (`git branch
--show-current` / `gh pr list --state open --limit 10`) was never vulnerable,
since it never interpolates `$ARGUMENTS`, but the class-wide test also covers
this file going forward. See [[post-review]] for the full fix detail.

## Trust floor for artifact gates: repo permission, not OWNER-badge (PR #14)

Both artifact gates (review and eval) read comments through a trust filter in `scripts/lib/git-common.sh`, not directly. [[pr_11-14_gate-hardening-followups|PR #14]] (`7c1dd19`, 2026-07-06, owner directive) removed the prior `author_association == "OWNER"` requirement — which failed closed on org-owned repos, since the merging identity's own comments there carry `MEMBER`/`COLLABORATOR` instead — and replaced it with a repo-permission check: the comment author's login must match the authenticated identity (unchanged anti-spoof property) **and** that identity must hold write access or better (`ADMIN`/`MAINTAIN`/`WRITE`, via `gh repo view --json viewerPermission`) on the current repo. Works identically on personal and org-owned repos now. See [[review-artifact-seam]] and [[task-evals-gate]] for the full mechanism (both gates share this trust filter).

## Error-message split for fetch failures (PR #14, extended PR #21)

Both gates' exit-2 ("GitHub fetch failed") branches used to emit one generic message regardless of which underlying `gh` call failed. `merge.sh` now inspects `PR_TRUST_FETCH_FAIL_REASON` (set by the trust-filter wrapper) and produces a distinct message for an identity-lookup failure, a permission-lookup failure, or a comments-fetch failure — applied to both the review-artifact gate and the eval-artifact gate. The reader's public exit-code contract (0/1/2) is unchanged; this is purely a diagnostic refinement on the existing exit-2 path.

**`tempfile` case arm added ([[pr_21-22_loop2-suggestion-tier-followups|PR #21]], merged 2026-07-06, `d0e4c5a`).** `pr::_trusted_comment_bodies_or_fail` (`scripts/lib/git-common.sh:182-196`) sets `PR_TRUST_FETCH_FAIL_REASON="tempfile"` when its own `mktemp` call fails — this happens **before** any `gh` API call is attempted, so a message saying "GitHub fetch failed" would be actively wrong, not just imprecise. Before this PR the `case` had no `tempfile)` arm and fell into the generic `*)` branch, which does say "GitHub fetch failed." `merge.sh` now carries a dedicated `tempfile)` arm on **both** gates (review and eval), each phrased around the local mktemp/tmp-file cause rather than a network fetch, and pointing at `/tmp` disk space/permissions as the likely cause. TDD: Test 3d (`merge.test.sh`) and Test 4d (`merge_evals_gate.test.sh`), both asserting message-distinctness against the identity/fallback messages. See [[trust-floor]] for the reason vocabulary this extends.

## Invocation

```
/coderails:merge [pr-number | branch-name | auto]
/coderails:merge          # auto-detects PR from current branch
/coderails:merge 42       # merge PR #42
/coderails:merge feature/add-retry-logic
```

Default argument is `auto`: resolves the PR from the current branch name.

## What it does

**Via `merge.sh`:**

1. **PR resolution**: Maps the argument to a PR number. `auto` calls `pr::num` against the current branch; a numeric arg is used directly; a branch name calls `pr::num` against that branch.
2. **Approval check (conditional)**: If the repository has branch protection requiring PR reviews (`protected` check via GitHub API), the script reads `pr::review` and errors if the decision is not `APPROVED`. (verified: merge.sh:37–40)
3. **Review artifact gate** (added PR #82): Fetches the current PR head SHA via `pr::head_sha`. Then calls `pr::has_coderails_review_for_head <num> <sha>` which scans all PR comment bodies for an exact coderails marker matching the current head SHA. Two distinct failure modes, each with a distinct error message:
   - Exit 2 (GitHub fetch failed) → "GitHub fetch failed — could not fetch PR comments."
   - Exit 1 (no artifact) → "No coderails review artifact for current head — run /coderails:post-review."
   No local-file fallback. No progress.json fallback. (verified: `merge.sh` @ 503f6fa)
4. **Eval artifact gate** (added PR #3 of [[pr_1-4_task-evals-feature]], directly after the review-artifact gate, same `OPEN` branch, same head SHA already resolved in step 3): calls `pr::has_coderails_eval_for_head <num> <sha>`. Same rc-contract shape as the review gate, with a tier-aware NO-GO message:
   - Exit 2 (GitHub fetch failed) → "GitHub fetch failed — could not fetch PR comments for eval artifact."
   - Exit 1, `PR_EVAL_TIER` set (artifact found but NO-GO) → "Eval artifact for current head is NO-GO (tier N) — resolve failing P0 evals and re-run /coderails:post-evals."
   - Exit 1, no artifact found at all → "No coderails eval artifact for current head — run /coderails:task-evals then /coderails:post-evals after /pr-review-toolkit:review-pr."
   Fail-closed, no local fallback, no config opt-out — identical posture to the review gate. This is **additive**, not a replacement: both gates must pass. See [[task-evals-gate]].
4a. **Smoke-verify gate** (added [[pr_279_merge_time_smoke_reexecution|PR #279]], 2026-07-23, directly after step 4, same `OPEN` branch): the eval-artifact gate in step 4 only parses the marker comment's `result=GO` text — it never re-validates the artifact's content. This gate closes that: extracts the fenced JSON embed from the same trusted comment already matched in step 4 (`pr::coderails_eval_embed_for_head`), checks out the trusted head SHA into a detached `git worktree`, and calls `post_evals::smoke_verify` to re-execute every scripted eval's `cmd` and `negative_control` there — judging only what's actually observed running, never the recorded `smoke` numbers. Closes the gap where a hand-written smoke object for a `cmd` naming a script that never existed passed step 4 at rc=0. Tier 0 (empty `.evals`) is a fast no-op. Fail-closed on `gh` fetch failure, worktree-add failure, or any check-10-shaped refusal (environmental cmd/negative_control, or a negative_control observed passing) — same posture as steps 3/4, no bypass. See [[pr_279_merge_time_smoke_reexecution]] and [[task-evals-gate]] for the full mechanism, including the four bypasses found and closed in review.
4b. **Tier-review gate** (added [[pr_232_tier-review-gate|PR #232]], 2026-07-17; only when `PR_EVAL_TIER == "0"`, i.e. only for a tier-0 claim): config-keyed and inactive by default — a no-op unless `tier_review.machine_user` is set in `workflow.config.yaml`. When active, fetches the newest `tier-review` GitHub commit status for the head SHA (via `gh api .../statuses`) and requires all three: `state == success`, the status's `creator.login` exactly matches the configured machine user (a mismatch is treated as misconfiguration-or-forgery and never bypassed), and the status description contains `verdict=legitimate` (not just `state=success`, which closes a verdict-laundering path). Fail-closed on a `gh` fetch failure, same posture as steps 3/4. **Explicitly documented in the script's own comments as redundant-by-design once a GitHub branch-protection ruleset is live** — a defence-in-depth layer for the pre-ruleset interim, not the primary control; do not remove once the ruleset is active. See [[pr_232_tier-review-gate]] for the daemon that posts this status.
5. **Merge**: `gh pr merge <num> --merge`. This is a remote merge only — its failure aborts the script via `set -euo pipefail`. Branch cleanup is explicitly separate and non-fatal so a worktree collision never causes a merged PR to report as failed.
6. **Sync**: Checks out `main` and pulls `origin/main`.
7. **Branch cleanup (best-effort)**:
   - Deletes the remote branch via `git push origin --delete <head>`. Warns but continues if already gone.
   - Attempts `git branch -D <head>` locally. If this fails (because another worktree has the branch checked out), warns with the worktree path rather than erroring.
8. Shows the last 5 commits on main.

## Config fields read

`merge.md`/`merge.sh` read no config for the review-artifact or eval-artifact gates — those two remain unconditional, no config dependency. **This changed for one field with [[pr_232_tier-review-gate|PR #232]] (2026-07-17):** `merge.sh` now reads `tier_review.machine_user` (via `coderails::config_path` + a minimal single-purpose extractor, `coderails::_tier_review_machine_user` — not a generic YAML reader) to activate the tier-review gate at step 4a above. Absent config = the gate is inactive, matching every other install's prior behaviour exactly. See [[config-resolution]] for context on how the other three commands use the same config file.

## Scripts invoked

- `scripts/merge.sh` — full merge/sync/cleanup logic. Sourced helpers from `scripts/lib/git-common.sh`:
  - `require::repo` — blocks if remote is not a GitHub repository
  - `protected` — queries GitHub API for branch protection status
  - `pr::num` / `pr::state` / `pr::title` / `pr::review` / `pr::url` — PR introspection via `gh`
  - `pr::head_sha` — fetches current head SHA for the PR (added PR #82)
  - `pr::has_coderails_review_for_head` — exact marker match against PR comments (added PR #82; exit codes 0/1/2)
  - `pr::has_coderails_eval_for_head` — newest-artifact-wins marker match against PR comments for the eval gate (added PR #3 of the task-evals cluster; exit codes 0/1/2; sets `PR_EVAL_TIER` on a match, unset at entry so it can't leak from a prior call)
  - `pr::coderails_eval_embed_for_head` — same newest-wins marker match, but echoes the fenced `json` embed rather than reporting result/tier ([[pr_279_merge_time_smoke_reexecution|PR #279]]); feeds `post_evals::smoke_verify`
  - `pr::_trusted_comment_bodies_or_fail` — wraps the trust-filtered comment fetch both gate readers call; recovers `PR_TRUST_FETCH_FAIL_REASON` (`identity`/`permission`/`comments`/`tempfile`) into the caller's shell for the error-message split (added PR #14; `tempfile` reason's own `merge.sh` case arm added PR #21 — the reason was always set here, only the consumer was missing the arm)
  - `branch` / `main` — current and default branch detection
- `scripts/lib/review-artifact.sh` — marker SSOT, sourced transitively by `git-common.sh` (added PR #82)
- `scripts/lib/eval-artifact.sh` — marker SSOT for the eval artifact, sourced transitively by `git-common.sh` (added PR #3 of the task-evals cluster); also `eval_artifact::compute_go`, the sole place `result` is derived from per-eval P0 statuses
- `scripts/post_evals.sh` (sourced) — `post_evals::smoke_verify` ([[pr_279_merge_time_smoke_reexecution|PR #279]]): checks out the trusted head SHA into a detached `git worktree` and re-executes every scripted eval's `cmd`/`negative_control` there

## Preconditions

- `gh` on PATH and authenticated
- Remote must be a `github.com` repository — `require::repo` gate (verified: merge.sh:12)
- PR must exist for the target branch
- If branch protection is enabled: PR must have `APPROVED` review decision
- A coderails review artifact must exist on the PR matching the **current** head SHA (added PR #82 — see [[review-artifact-seam]])
- A coderails eval artifact must exist on the PR matching the **current** head SHA, with `result: GO` (or a justified tier-0 exemption) — added PR #3 of the task-evals cluster (see [[task-evals-gate]])
- If the PR's eval artifact is tier 0 AND `tier_review.machine_user` is configured: a `tier-review` status of `state: success`, posted by that exact machine user, carrying `verdict=legitimate` — added [[pr_232_tier-review-gate|PR #232]] (see step 4a above)
- PR must not be already closed (without merge)

## Chain position

Fourth (and last) in the chain. Called by [[workflow]] after ship-it authorisation, or standalone.

```
/prep  →  (code)  →  /push  →  /merge
                                ^^^^^^
```

After `/merge`, [[workflow]] runs `/wiki-ingest` and `/wiki-lint` if `config.wiki_path` is non-null — merge is the trigger for wiki update. If the worktree is still present after merge, [[workflow]] removes it via `git worktree remove` and `git branch -d` (separate from merge.sh's cleanup). (inferred: workflow.md:175–182)

## Design notes

The deliberate decoupling of branch cleanup from the merge step is a key design decision. Using `--delete-branch` on `gh pr merge` would delete the local branch too, which fails (and under `set -euo pipefail`, aborts the whole script) when another worktree holds the branch checked out. Separating and making cleanup non-fatal means a merged PR never reports as failed due to a worktree collision. (verified: merge.sh:52–56 inline comment)

The `protected` check uses the GitHub API directly rather than relying on `gh pr merge` to reject unapproved merges — this provides an explicit, user-readable error before the merge attempt. (inferred: merge.sh:37–40)

**Enforcement-gap notice (added PR #43):** When no `workflow.config.yaml` is present, `merge.sh` emits an informational `info` line before the artifact gate: "No workflow.config.yaml — enforce_pr_workflow hook is inactive, but the review artifact gate still applies." This confirms that the review artifact gate is unconditional — it does not require a config file. `enforce_pr_workflow` remains opt-in; the review artifact gate does not. (verified: `merge.sh` @ 503f6fa)

**Review artifact gate is unconditional (added PR #82):** Unlike `enforce_pr_workflow` (which no-ops without `workflow.config.yaml`), the review artifact gate runs on every merge regardless of config. It is built into `merge.sh` directly, not into the hook. The enforcement ceiling note from [[enforcement-model]] still applies: the gate checks that the artifact exists and matches the SHA; it does not verify the review was substantive. (verified: `merge.sh`, [[review-artifact-seam]])

**Eval artifact gate is also unconditional and additive (added PR #3 of the task-evals cluster, 2026-07-06):** Same unconditional posture as the review gate — no config opt-out, runs on every merge. It sits in the same `OPEN` branch directly after the review-artifact check, reusing the head SHA already resolved for that gate. Both gates must pass; neither substitutes for the other. Same honest-ceiling caveat: proves a structurally-valid, SHA-bound artifact with `result: GO` exists, not that the underlying evals were run honestly or that an `agent-run` verifier had a genuinely clean context. See [[task-evals-gate]].

## See also

- [[post-review]] — the command that creates the review artifact this gate checks
- [[post-evals]] — the command that creates the eval artifact the second gate checks
- [[review-artifact-seam]] — design page for the full truth-seam architecture (review artifact)
- [[task-evals-gate]] — design page for the dual-scope eval-gate architecture (pr + loop)
- [[task-evals]] — the skill that generates the evals.json posted as the eval artifact
- [[push]] — creates the PR that this command merges
- [[workflow]] — calls /merge in Phase 6, then wiki-ingest/lint
- [[config-resolution]] — merge previously read no config; since [[pr_232_tier-review-gate|PR #232]] (2026-07-17) it reads one field, `tier_review.machine_user` (see "Config fields read" above), the only workflow.config.yaml key merge.sh consumes
- [[repo-hosting]] — github.com remote requirement
- [[enforce_pr_workflow]] — PreToolUse hook that blocks `gh pr merge` unless `/pr-review-toolkit:review-pr` ran this session (NO_CONFIG opt-in; separate from the artifact gate)
- [[enforcement-model]] — the honest-ceiling framework both gates sit within
- [[pr_89-91_skills-doc-frontmatter-injection]] — PR #91 source record: Current Git Status injection added; post-review.md injection deferred pending an inconclusive substitution-ordering probe
- [[pr_93-94_post-review-injection-and-exec-bit-invariant]] — PR #93 ships the deferred post-review.md injection
- [[pr_96-98_mode-aware-install-argument-injection-guard-hook-owned-counter]] — PR #97 removes that injection for a command-substitution vulnerability; PR #96 makes install.sh's chmod sweep git-index-mode-aware
- [[pr_11-14_gate-hardening-followups]] — PR #14 widens the trust floor to a repo-permission check and splits the fetch-failure error messages; PR #11 fixes the loop-scope tier-0 NO-GO precedence
- [[pr_279_merge_time_smoke_reexecution]] — PR #279 (2026-07-23): the smoke-verify gate at step 4a, making checks 9/10's re-execution property binding at merge rather than advisory
- [[pr_232_tier-review-gate]] — PR #232 (2026-07-17): the fourth, opt-in tier-review gate at step 4b; the only workflow.config.yaml field merge.sh now reads
