---
title: "Evals-gate enforcement gap — 2026-07-08"
type: investigation
created: 2026-07-08
last_updated: 2026-07-08
sources: [sources/pr_95_slash-command-loop-detection.md]
tags: [investigation, task-evals, enforcement-model, systematic-debugging, workflow-uniformity, discipline-loop]
---

# Evals-gate enforcement gap — 2026-07-08

> Filed: 2026-07-08. Point-in-time snapshot — may be superseded.

## Question

Is the `task-evals` gate enforced uniformly across every entry workflow, or are there paths into implementation work that route around it entirely? This was prompted directly by [[pr_95_slash-command-loop-detection]]: that PR fixed a real bug (`als_count_invocations` undercounting slash-started loops) via a debugging-framed workflow, and no hook ever required a frozen `evals.json` for it.

## Evidence

- `grep -niE "eval" skills/systematic-debugging/SKILL.md` returns **zero hits** — the skill most naturally triggered by "there's a bug, fix it" makes no mention of `task-evals` anywhere in its trigger phrases, phases, or cross-references.
- `skills/systematic-debugging.md`'s own "See also" links only to [[verification-before-completion]] and [[test-driven-development]] — its full outbound chain is TDD → verify → verification-before-completion, with no branch toward evals.
- [[task-evals-gate]] (design page) documents exactly two hook-enforced consumers of frozen `evals.json`: a **pr-scope** gate in `scripts/merge.sh` (SHA-bound PR comment, checked before `gh pr merge`) and a **loop-scope** gate inside `loop_state_guard.sh` (`gate_loop_evals_required`, fires only when `work_units >= 3` inside an active agentic-loop).
- [[enforcement-model]]'s hook map confirms `enforce_pr_workflow.sh` gates `gh pr create`/`gh pr merge`/`git merge`/`git push` on prior `/review-pr` evidence — there is no equivalent hook gating any of those on prior `/coderails:post-evals` evidence for a single-PR (non-loop) path.
- [[writing-plans]] and [[subagent-driven-development]] both carry a **prose** (advisory, non-hook) instruction to freeze evals before implementation — confirmed by their SKILL.md content already cited on [[task-evals-gate]]'s "Owner decisions recorded" and cross-reference sections.
- [[agentic-loop]] Phase 2.7c freezes loop-scope evals and is the one path where the resulting `evals.json` is later hook-checked, but only once `work_units` reaches 3 — below that threshold the loop-scope gate allows unconditionally (`evals=skipped-below-threshold`, per [[task-evals-gate]]'s block-condition table).
- [[pr_95_slash-command-loop-detection]] itself: a debugging-framed single-PR fix. `enforce_pr_workflow` forced the review gate (a PR comment from `/pr-review-toolkit:review-pr` was required before merge and was in fact obtained — `code-reviewer` clean, `pr-test-analyzer` findings addressed). No hook forced or even checked for a `task-evals` artifact, and none was produced.

## Findings

**(verified)** Three entry workflows route to `task-evals` today, with different enforcement strength:
- `agentic-loop` — **hook-enforced**, but loop-scope only, and only above the 3-work-unit threshold (`loop_state_guard.sh`'s `gate_loop_evals_required`).
- `writing-plans` — **prose-only**: the plan template includes a "freeze evals" step, but nothing mechanically checks it ran.
- `subagent-driven-development` — **prose-only**: same shape, a Pre-Flight bullet per [[pr_15-17_loop-hardening-registration-eval-freeze-ledger-dry|PR #15]], not hook-checked.

**(verified)** One entry workflow has **no route to task-evals at all**:
- `systematic-debugging` — zero mention of evals anywhere in the skill. Its natural chain (TDD → verify → verification-before-completion) never touches the concept.

**(verified: `coderails/scripts/merge.sh` lines 69–90, read directly)** For a single-PR path outside `agentic-loop`, the only mechanically-enforced *invocation* gate is `review-pr` (via `enforce_pr_workflow`'s `PreToolUse` hook) — but the pr-scope `task-evals` *artifact* gate inside `merge.sh` itself is unconditional and fail-closed, with no config opt-out: the code has no `NO_CONFIG`/opt-out branch around it, and the header comment states outright "No fallback, no config opt-out (same posture as the review gate)." This mirrors the review-artifact gate immediately above it in the same function (lines 46–67), which is likewise unconditional inside `merge.sh`. The one config-sensitive line in the whole function (42–44) names only `enforce_pr_workflow` as conditionally inactive when `workflow.config.yaml` is absent — it never extends that carve-out to either artifact gate.

So the real shape of the gap is narrower than "no enforcement at all": **any merge routed through `scripts/merge.sh`** (i.e. via `/coderails:merge`) is blocked without a GO-result eval comment on the current head SHA, regardless of workflow.config.yaml presence — see the `eval_gate_rc` handling: `err "No coderails eval artifact for current head $sha — run /coderails:task-evals then /coderails:post-evals after /pr-review-toolkit:review-pr."` (line 87) or the NO-GO variant (line 85). The actual gap is upstream of `merge.sh`: **nothing forces the merge to go through `scripts/merge.sh` in the first place.** `enforce_pr_workflow.sh` (the `PreToolUse` hook) gates a raw `gh pr merge` Bash call on prior review evidence, but per [[task-evals-gate]] and [[enforcement-model]]'s hook map, no hook checks for eval evidence before that same raw `gh pr merge` call — a session that never invokes `/coderails:merge` and instead runs `gh pr merge` directly (once past the review-evidence gate) skips the eval check entirely, since the eval gate lives only inside the `merge.sh` script body, not in any hook. This is exactly the same "evidence not completion" / script-vs-hook asymmetry [[enforcement-model]] already documents for `enforce_pr_workflow` itself, one layer further in: `merge.sh` enforces evals whenever it runs, but nothing enforces that it runs.

**Consequence for PR #95 specifically (verified: `gh pr view 95 --json comments` against `blueman82/coderails`, run directly):** PR #95 has **zero PR comments** — no review-artifact marker, no eval-artifact marker, nothing. This settles the (a)/(b) question above: PR #95 was merged by a path that bypasses `scripts/merge.sh` entirely (a raw `gh pr merge`, or an equivalent, once whatever gate was actually active let it through), not via `/coderails:merge` with a satisfied eval gate. Since `merge.sh`'s own review-artifact and eval-artifact gates are both unconditional fail-closed checks against PR comments (verified above), and there are no comments to match against, `/coderails:merge` could not have been the path used — running it against this PR's actual history would have blocked on the review gate before ever reaching the eval gate. This confirms the gap is real and concrete for PR #95, not merely theoretical: whatever merged it skipped both artifact gates by not going through the script that enforces them, and no hook independently enforced either one on the raw merge call.

**(verified)** The concrete consequence on PR #95: a real, debugging-framed bug fix went through `enforce_pr_workflow`'s review gate (forced, and satisfied) but never touched `task-evals` (not forced by anything reachable from `systematic-debugging`). This is not a hypothetical — it is the exact path this PR took.

## Adversarial review

Not run — this investigation is filed as a first-pass finding, not yet stress-tested via `/coderails:planning-sequence` or an independent reviewer. The gate-mechanism question (whether `merge.sh`'s pr-scope eval check is unconditional or config-gated) is now resolved by direct source read (see Findings) — it is unconditional. The remaining open question for an adversarial pass is upstream: exactly which command/path was used to merge PR #95 if not `/coderails:merge` — this investigation shows *that* the artifact gates were bypassed (zero PR comments), not *how* (e.g. direct `gh pr merge` after some other review path, or a merge performed outside this repo's own hook-guarded session entirely).

## Resolution

Deferred. A Thread-B fix (closing the gap — likely either adding a `systematic-debugging` → `task-evals` cross-reference at minimum, or a hook-level check gating raw `gh pr merge` on eval evidence the same way `enforce_pr_workflow` already gates it on review evidence) is being handed off separately and is explicitly **not** part of [[pr_95_slash-command-loop-detection]]. No ticket filed yet for the follow-up (no JIRA project wired to this repo per [[pr_95_slash-command-loop-detection]]'s own metadata table).

## See also

- [[pr_95_slash-command-loop-detection]] — the PR whose own debugging-framed path surfaced this gap
- [[task-evals-gate]] — the dual-scope (pr + loop) gate design this investigation audits for coverage
- [[discipline-loop]] — the six-hook Stop-hook composition; the general enforcement-floor framing this gap sits inside
- [[enforcement-model]] — the Law ("hooks are mechanical enforcement, slash commands are advisory") and its existing documented ceilings (e.g. `enforce_pr_workflow`'s "evidence not completion")
- [[systematic-debugging]] — the skill confirmed to have zero evals cross-reference
- [[writing-plans]] · [[subagent-driven-development]] — the two prose-only (non-hook) evals entry points
