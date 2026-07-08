---
title: "PRs #96–98 — evals-gate enforcement uniformity (hook-level gh pr merge gate, systematic-debugging routing, coverage-boundary docs)"
type: source
created: 2026-07-08
last_updated: 2026-07-08
sources:
  - hooks/scripts/enforce_pr_workflow.sh
  - hooks/scripts/tests/enforce_pr_workflow.test.sh
  - skills/systematic-debugging/SKILL.md
  - skills/task-evals/SKILL.md
  - docs/REFERENCE.md
  - AGENTS.md
tags: [source, task-evals, enforcement-model, enforce_pr_workflow, systematic-debugging, hook-gate, coverage-boundary, workflow-uniformity]
---

# PRs #96–98 — evals-gate enforcement uniformity (2026-07-08)

> ⚠️ **PR-number collision, unrelated to this page's content:** an earlier, different cluster of PRs #96/#97/#98 (merged 2026-07-03/2026-07-03/2026-07-05, in the repo's history before the `blueman82/coderails-dev`→`coderails` rename) is documented at [[pr_96-98_mode-aware-install-argument-injection-guard-hook-owned-counter]] — mode-aware `install.sh` sweep, the `post-review.md` command-substitution injection fix, and the `loop_stall_guard`-owned `loop_stop_counts` counter. That page's PR numbers were reused after the rename/renumbering; **this page documents a completely separate, later cluster** (merged 2026-07-08) with no content overlap. Do not conflate the two — check the `merged` date/SHA before citing either.

Three PRs merged same-day to `blueman82/coderails` main, closing the gap [[evals-gate-enforcement-gap_2026-07-08]] filed earlier the same day.

## PR #97 — Gate `gh pr merge` on the coderails eval artifact (merged `5b96690`, 2026-07-08T12:30:02Z)

`scripts/merge.sh` already required a SHA-bound `GO` eval-artifact PR comment before merging (via `pr::has_coderails_eval_for_head`), but `enforce_pr_workflow.sh`'s `PreToolUse` hook only ever checked for prior `/pr-review-toolkit:review-pr` evidence on `gh pr merge` — a raw `gh pr merge <N>` run outside `/coderails:merge` bypassed the eval gate entirely. **This is the exact mechanism [[pr_95_slash-command-loop-detection|PR #95]] shipped through with zero PR comments of either kind.**

**Fix — new `gate_eval_artifact_for_merge` function**, appended to the hook's existing gate chain (after `enforce_required_step`, before `exit 0`):

```
gate_has_command → gate_safe_passthrough → gate_in_scope → gate_config_present
  → gate_targets_main → gate_have_transcript → enforce_required_step
  → gate_eval_artifact_for_merge → exit 0
```

- Scoped to the `merge` subcommand only (`[ "$subcommand" = "merge" ] || return 0`) — `git_merge`/`git_push` are untouched by this gate.
- Runs strictly *after* the review-pr gate: `[ "$step_found" -eq 0 ] 2>/dev/null && return 0` skips the eval check if `enforce_required_step` already denied this invocation, so only one deny fires per call, and the network-dependent eval check never runs until the cheap transcript-only review check has already passed.
- The hook `cd`'s into the payload's `cwd` (a plain `cd` in-process, not a subshell) before calling `repo()`/`pr::*` helpers, because those helpers are CWD-dependent (`git remote get-url origin` takes no path arg) and a subshell would scope-lose the `PR_EVAL_TIER`/`PR_TRUST_FETCH_FAIL_REASON` globals the deny messages read afterward.
- Resolves the PR number from `$pr_num` (parsed earlier in the hook) or falls back to `pr::num "$(branch)"`; resolves head SHA via `pr::head_sha`; calls `pr::has_coderails_eval_for_head "$num" "$sha"`.
- **Exit-code handling mirrors `scripts/merge.sh`'s own eval gate exactly:**
  - `rc=2` (gh fetch failed) → deny, fail-closed, reason keyed off `PR_TRUST_FETCH_FAIL_REASON` (`identity` / `permission` / `tempfile` / default "could not fetch PR comments").
  - `rc≠0` (not 2) → deny; message is tier-aware NO-GO (`PR_EVAL_TIER` set) or "no coderails eval artifact for current head" (no marker at all).
  - `rc=0` → allow. Any tier's `GO` (including tier 0) satisfies it, same as `merge.sh`.
- `sourcing`: the hook now sources `scripts/lib/git-common.sh` at the top (pulls in `eval-artifact.sh` + `review-artifact.sh` transitively); `git-common.sh`'s colour vars are `_GIT_COMMON_COLORS_LOADED`-guarded so double-sourcing (merge.sh also sources it in-process elsewhere) is safe.
- **Documented residual, unchanged by this PR:** `git_merge`/`git_push` have no PR number to resolve a SHA-bound artifact against, so they remain review-gated only — this is the same boundary [[task-evals-gate]] and the investigation already named, now also stated in `docs/REFERENCE.md` (PR #98, below).
- NO_CONFIG posture is unaffected — `gate_config_present` (Gate 4) still stands aside before any of this new gate's code ever runs, in any repo without `workflow.config.yaml`.

**Tests** (`hooks/scripts/tests/enforce_pr_workflow.test.sh`): a global `gh` mock (`MOCKGH_DIR`, a fake executable placed first on `PATH` for the whole suite) now backs every `gh` call this hook's new gate makes — `pr list --json number`, `pr view --json headRefOid`, `api user`, `repo view --json viewerPermission`, `api repos/.../comments`. The default mock auto-satisfies the eval gate (`GO tier=1`) so every pre-existing review-pr-only `gh pr merge` ALLOW case in the file is unaffected without modification. A new dedicated "EVAL ARTIFACT GATE" section adds cases for: no marker → deny; `GO` tier 1 → allow; `GO` tier 0 → allow; `NO-GO` tier 2 → deny with tier named in the message; `MOCK_GH_COMMENTS_FAIL=1` → rc=2 fail-closed deny with a retry hint; `NO_CONFIG` → allow; and gate-ordering (the review-pr gate fires and denies first, before the eval gate ever runs). Full suite green: **93 ok, 0 FAIL** (up from 86 pre-existing cases).

`AGENTS.md`'s `enforce_pr_workflow.sh` hook-table row updated in lockstep to describe the new block condition.

## PR #96 — route `systematic-debugging` fixes to `task-evals` (merged `4e727b2`, 2026-07-08T12:30:22Z, commit `8395498`)

Closes the *other* half of the gap [[evals-gate-enforcement-gap_2026-07-08]] identified: `systematic-debugging` had **zero** route to `task-evals` at all — a debugging-framed fix (exactly [[pr_95_slash-command-loop-detection|PR #95]]'s own shape) never produced an `evals.json` in the first place, so PR #97's new hook gate would have had nothing to check even if it had existed sooner. This is a reciprocal cross-reference, both directions:

- **`skills/systematic-debugging/SKILL.md`**, Phase 4 ("Fix the root cause, not the symptom") gains a new **Step 0**, before the pre-existing Step 1 (Create Failing Test Case):
  > **Freeze success evals if the fix will carry a PR** — if this fix is a code change that will carry a PR, invoke `/coderails:task-evals` (scope: `pr`) now, before writing the fix, to freeze the success evals — same freeze-before-build rule as the test in step 1 below. Subject to task-evals' own tier rules: a trivial fix meeting the tier-0 predicate takes the tier-0 exemption, not a skip.

  Also adds a "Related skills" bullet: `coderails:task-evals` - Freeze PR-scope success evals before implementing a fix that will carry a PR (Phase 4, Step 0).

- **`skills/task-evals/SKILL.md`**: the skill's own invocation contract — "This skill is invoked at three points" — becomes **four points**, with a new bullet: "**systematic-debugging** — pr scope, frozen before the fix is implemented, when a debugging fix will carry a PR."

This is prose-only (advisory), the same enforcement strength `writing-plans` and `subagent-driven-development` already had per [[task-evals-gate]] — it is **not** hook-enforced that a debugging session actually invokes `task-evals`. What PR #97 changes is that *if* a debugging-framed fix skips this new Step 0 and is merged via `gh pr merge` (with `workflow.config.yaml` present), the merge itself is now blocked absent an eval artifact — the two PRs close complementary halves of the same gap (invocation-path prose nudge + merge-path mechanical gate), neither alone sufficient.

## PR #98 — document the coverage boundary (merged `d9ea84e`, 2026-07-08T13:03:59Z, commit `1c261a2`)

Docs-only. `docs/REFERENCE.md`:
- The `enforce_pr_workflow.sh` hook-table row updated to describe the new eval-artifact block condition on `gh pr merge` (mirrors the `AGENTS.md` update from PR #97, same wording pattern).
- New bullet in the hook-notes section, **"Eval-gate coverage boundary"**: the coderails eval artifact is enforced at exactly two points — `/coderails:merge` via `scripts/merge.sh` (config-independent, no opt-out) and raw `gh pr merge <N>` via this hook (config-dependent — inactive under `NO_CONFIG`, same as the rest of `enforce_pr_workflow`). It is explicitly **not** enforced on raw `git merge`/`git push` to main/master (the hook has no PR number to resolve a SHA-bound artifact against — those stay review-gated only) or in any `NO_CONFIG` repo. Stated as a **documented residual, accepted not closed** — matching the framing in `docs/coderails/specs/2026-06-26-subagent-enforcement-and-gate-hardening-design.md`.

This is an honest-boundary statement, not a further closure — it records exactly what PR #97 did and did not cover, so a future reader doesn't mistake the hook-level gate for total coverage.

## The overall arc

[[evals-gate-enforcement-gap_2026-07-08]] (filed the same morning, prompted by [[pr_95_slash-command-loop-detection|PR #95]]'s own bypass) identified two independent holes in one afternoon of investigation:

1. **Enforcement-path hole**: PR-scope `task-evals` was enforced only *inside* `scripts/merge.sh`'s script body — nothing forced a merge to actually go through that script. A raw `gh pr merge` (once past the pre-existing review-pr hook gate) skipped the eval check entirely. **Closed by PR #97** — the identical check now also runs as a hook gate on the raw command itself.
2. **Invocation-path hole**: `systematic-debugging`, the skill most naturally triggered by "there's a bug, fix it," had zero cross-reference to `task-evals` anywhere in its trigger phrases or phases — so a debugging-framed fix never produced an eval artifact to begin with. **Closed (prose-level) by PR #96.**

PR #98 documents the resulting boundary honestly: `git merge`/`git push` to main and any `NO_CONFIG` repo remain outside both the pre-existing review gate's and the new eval gate's reach for this hook — an accepted, named residual, not a further fix.

## See also

- [[evals-gate-enforcement-gap_2026-07-08]] — the same-day investigation this cluster closes (Thread-B fix)
- [[task-evals-gate]] — the dual-scope design page; this cluster adds a third hook-enforced consumer to the pr-scope gate
- [[enforce_pr_workflow]] — the hook page this cluster's PR #97 material is now primarily documented on
- [[enforcement-model]] — the hook-map page whose `enforce_pr_workflow` row needed the same update
- [[pr_95_slash-command-loop-detection]] — the PR whose own merge path surfaced the gap this cluster closes
- [[systematic-debugging]] — the skill gaining the new Step 0 cross-reference to task-evals
- [[pr_96-98_mode-aware-install-argument-injection-guard-hook-owned-counter]] — the unrelated, earlier cluster that reused these same PR numbers (see collision note at top of this page)
