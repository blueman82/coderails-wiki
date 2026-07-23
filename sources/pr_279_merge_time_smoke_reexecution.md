---
title: "PR #279 — merge-time smoke re-execution against the trusted head SHA"
type: source
origin: "coderails PR #279 (merged afa2f5b, 2026-07-23)"
created: 2026-07-23
last_updated: 2026-07-23
sources: []
tags: [source, task-evals, gate, post-evals, smoke-run, re-execution, merge-gate, fabricated-evidence, honest-boundary, worktree, bypass]
---

# PR #279 — binding gate-time re-execution at merge

Merged 2026-07-23T06:54:06Z at `afa2f5b`. Completes an arc [[pr_264_smoke_run_executor_and_check9|PR
#264]] and [[pr_269_gate_time_smoke_execution|PR #269]] left with a named residual — read those two
first.

## What it closes

Check 10 (`post_evals::validate_smoke_execution`, #269) re-executes every scripted eval's `cmd` and
`negative_control` and judges the observed shape, never the recorded numbers — but it ran **only**
inside `post_evals::validate_structure` at **post time, in the posting agent's own session**
(`/coderails:post-evals`). Advisory, not binding.

The merge gate (`pr::has_coderails_eval_for_head`, `scripts/lib/git-common.sh:243`, verified) only
parses the trusted marker comment for `result=GO` and a matching head SHA — it never re-validates
the `evals.json` content itself. So an agent could hand-write a `smoke` object of plausible shape
(`{"cmd_exit":1,"negative_control_exit":1}`) for a `cmd` naming a script that never existed, post it,
skip running `/coderails:post-evals` honestly (or run it and then edit the artifact after), and the
merge path accepted it at rc=0 — the exact fabrication class checks 9/10 exist to catch, just never
actually enforced at the point that matters.

> This is #269's own residual, verbatim: *"an author whose commands resolve and whose control fails
> at gate time but who never ran anything at freeze still passes... closing it needs an unforgeable
> freeze stamp, or an attestor outside the agent's trust domain."* #279 does not build that attestor.
> It closes a narrower, still-real gap: the same re-execution check 10 already performs, but run a
> second time at merge, against the trusted commit, by the merging process rather than the posting
> agent.

## The mechanism

Adds `post_evals::smoke_verify <embed_json_path> <head_sha>` (`scripts/post_evals.sh:457`, verified).
Called with the **embed** extracted from the same trusted marker comment the eval-artifact gate
already matched (`pr::coderails_eval_embed_for_head`, new function, `scripts/lib/git-common.sh:286`)
— never a local `evals.json` file, which the caller (running at merge time, possibly from a
different checkout) might not have at all.

Steps, in order:
1. Tier 0 → return 0 immediately (empty `.evals`, nothing to re-execute).
2. Collect scripted-eval **array indices** (not `id`s — see Bypass 2/3 below).
3. `git worktree add --detach` the trusted `head_sha` into a fresh `mktemp -d` directory. Fetches
   the SHA first if not already in the local object store (`git fetch origin "$head_sha"`) — the
   merge hook can fire from a checkout that never had the PR branch.
4. For every scripted eval, re-run `cmd` and `negative_control` inside that worktree via
   `post_evals::_run_recorded`, judging only what's observed — the same shape rules check 10
   applies (environmental cmd/negative_control refuses; a negative_control observed exiting 0
   refuses), never the typed `smoke` numbers.
5. Removes the worktree on every exit path.

Wired into **both** merge consumers:
- `scripts/merge.sh:133-149` (verified) — a new "Smoke-verify gate" block directly after the
  existing eval-artifact gate, before `gh pr merge`.
- `hooks/scripts/enforce_pr_workflow.sh` — new `gate_smoke_verify()` (verified, line 536), called
  from the same chain as `gate_eval_artifact_for_merge` at line 519:
  `gate_smoke_verify "$num" "$sha" && gate_tier_review_status "$num" "$sha"` — i.e. **before**
  [[pr_232_tier-review-gate|Gate 8's tier-review status check]], not after.

## Why NOT a call into check 10 / `validate_structure`

`smoke_verify` does not call `post_evals::validate_structure`'s `merge` scope through check 10 — it
runs its **own** re-execution loop, reusing only the primitives (`_run_recorded`,
`_is_environmental_rc`). Two reasons, both stated in the source comment (verified,
`scripts/post_evals.sh:448-455` region):

- Checks 1-9 already ran at post time (structural validation — not the re-execution property) and
  re-imposing them at merge added failure modes unrelated to fabrication: check 2
  (`tier_justification`) and check 6 (`.head_sha` vs trusted sha) both false-blocked a genuine,
  resolvable P4 acceptance fixture during the building loop's own verification, for reasons
  unrelated to whether the `cmd` is real.
- Check 10 re-executes in the **caller's own cwd** under a hardcoded 10s alarm. Calling it from the
  merge gate would be wrong twice: wrong directory (defeats worktree isolation and the priming
  defence below) and wrong timeout (next section).

`validate_structure` gained a new **`merge` scope**: runs checks 1-9 (identical to `pr` scope) and
stops before check 10. `smoke_verify` is the re-execution that replaces check 10 at the merge
boundary — a second, purpose-built call site judging the same shape rules, not a duplicate rule set.

## Timeout: a separate, measured constant

`POST_EVALS_SMOKE_VERIFY_TIMEOUT` defaults to **120s** (verified, `scripts/post_evals.sh:24`),
distinct from check 10's hardcoded 10s alarm. Measured against real eval-shaped commands in this
repo before picking the number: eval-artifact 114ms, `merge_evals_gate` 1.6s, `post_evals` 10.4s,
`discriminate` 21.5s. The freeze-time 10s alarm would false-fail the `discriminate` suite outright.
Check 10's own 10s default in `_run_recorded` is untouched — a new optional timeout/cwd parameter
was added to `_run_recorded` instead, defaulting to the old behaviour for every pre-existing caller.
The same discipline recurs across this wiki's gate history: sample the real population a threshold
filters, rather than pick a round number freehand.

## Worktree-at-head-SHA: load-bearing twice

1. **Directory correctness.** At merge, the cwd may be `main`, not the PR head — a bare re-execution
   there would false-127 on a script that only exists on the PR branch, unrelated to whether the
   artifact is honest.
2. **Neutralises priming.** An agent could plant a stub script locally that was never committed, so
   a real `cmd` resolves in the agent's own working tree but wouldn't exist for anyone else.
   Checking out the exact trusted SHA into a fresh worktree means only what was actually committed
   at that SHA can run.

## No surface exemption (a bypass found and closed in review, not shipped as a design choice)

**Important nuance the PR body's summary softens**: the merged code carries **no** `surface`
exemption at all — verified directly in `scripts/post_evals.sh`'s `smoke_verify` header comment,
which states explicitly *"NO SURFACE EXEMPTION. Every scripted eval is re-executed, regardless of
its `surface` field."* An earlier draft did skip `surface: deployed`/`surface: fresh-clone` (neither
runs in a local detached worktree) — but `surface` is written by the same posting agent this gate
exists to distrust, and nothing in checks 1-9 constrains it. That made the exemption
attacker-writable: a scripted eval with a fabricated `cmd` plus `surface: "deployed"` would dodge
re-execution entirely and still pass at rc=0 — the exact "hand-written smoke object for a script
that never existed" class this whole gate exists to close. The draft's exemption also diverged from
check 10, which has no surface filter at all. The axis that legitimately excludes an eval here is
**`mode`**, not `surface`: a scripted eval has a runnable `cmd` by definition and must be verified;
`agent-run` evals (no `cmd`) are excluded by the mode filter and graded by the verifier subagent
instead — unaffected, since `smoke_verify` only runs at pr/merge scope, never loop scope.

## Four bypasses found in review, closed pre-merge — one root pattern

All four (plus two adjacent hardening fixes) reduce to the same defect class: **gate authority
resting on a field the gated party controls.**

1. **Surface exemption** (above) — `surface` is agent-written; keying re-execution skip on it let a
   fabricated `cmd` dodge the gate entirely. Fixed: key on `mode`, never `surface`.
2. **Empty-`id` bypass** — the re-execution loop originally keyed on extracting a list of eval
   `id`s. An eval with `id: ""` produced a blank line that a skip-empties loop silently dropped,
   meaning that eval's `cmd`/`negative_control` were never re-executed at all.
3. **Duplicate-`id` bypass** — same root cause: two evals sharing one `id` would run one eval's `cmd`
   twice while the other's never ran, again letting a fabricated eval hide behind a legitimate one's
   identifier.
   Both 2 and 3 closed by the same fix: **iterate by array index**, never by `id` — `id` is only
   used for human-readable error messages now, never to select which eval executes (verified,
   `scripts/post_evals.sh:539-546` comment block and the `to_entries | ... | .key` index-collection
   idiom at line ~490).
4. **Git-fetch gap** — a legitimate merge could fail closed (false negative) if the trusted
   `head_sha` wasn't already in the local object store and the merge hook fired from a checkout
   without the PR branch fetched. Fixed: `smoke_verify` fetches the SHA first
   (`git fetch origin "$head_sha"`) before attempting the worktree checkout; a fetch failure itself
   still fails closed (return 1), never a silent skip-verification fallback.
5. **cd-failure fail-open** — an adjacent hardening fix: a failed `cd` into the worktree previously
   surfaced as a generic rc that wasn't classified as environmental, risking a fabricated command's
   real failure mode being misread as a legitimate discrimination result rather than "never ran."

Bypasses 2/3 are the most structurally interesting: they are the **same defect class** the surface
exemption was, discovered independently — gate authority must never rest on a field the gated party
controls. This is now the second time this exact lesson has recurred inside this one PR (surface,
then id), and it echoes [[pr_218_discriminating-check-gate]]'s env-guard ordering fix and
[[pr_269_gate_time_smoke_execution]]'s general "decompose the can't-be-recomputed claim" move — a
recurring genre in this gate's history, not a one-off.

## Verification (as reported in the PR)

- **Acceptance test** (`post_evals.test.sh`, "smoke_verify: BEFORE/AFTER acceptance"): a real
  throwaway git repo, one committed pair of honest scripts, and a hand-written smoke object of the
  previously-accepted shape (`{"cmd_exit":1,"negative_control_exit":1}`) whose `cmd` names a script
  never committed at that head SHA. `smoke_verify` refuses it (exit 1, stderr names the eval) —
  **before this PR, that identical artifact satisfied the merge path at rc=0** via the marker's
  `result=GO` text alone. A companion fixture with real, committed, honestly-failing scripts is
  accepted (exit 0).
- Direct unit coverage added for `pr::coderails_eval_embed_for_head` (`git-common.test.sh`):
  newest-comment-wins, no-match, gh-fetch-failure, and untrusted-comment rejection — this function
  selects which artifact gets re-executed, so it earns its own tests rather than being exercised
  only indirectly through `merge.sh`'s stubs.
- Wiring proven at **both** merge consumers, not just `smoke_verify` in isolation:
  `merge.test.sh`, `merge_evals_gate.test.sh`, and `enforce_pr_workflow.test.sh` each got a
  negative-path case (smoke-verify stub fails → merge blocks with a named reason).
- Full affected suites reported green, run directly (not trusted from a sub-agent's own report):
  `post_evals`, `merge`, `merge_evals_gate`, `merge_tier_review_gate`, `enforce_pr_workflow`,
  `git-common`.
- [[post-evals]]'s structural-guarantees prose corrected the stale "checks 1-7" to "checks 1-10 in
  `post_evals::validate_structure` at post time, and `post_evals::smoke_verify`'s gate-time
  re-execution of every scripted eval at merge" — verified directly against the merge commit's own
  diff (`git diff afa2f5b^1 afa2f5b -- commands/post-evals.md` and the PR's actual `baseRefOid`,
  `e5c26fa`, which still reads "checks 1-7"): this genuinely is #279's own fix, landed in its head
  commit `baa5bbd`, 2026-07-23. (A `git log -S"checks 1-10"` search on this string alone returns no
  hits before this PR, confirming the wording is new here, not a renumbering of pre-existing text —
  an earlier check of this claim wrongly concluded otherwise from a `git log --oneline` pass that
  matched the unrelated "checks 1-7 in `post_evals::validate_structure`" phrase in commits `196009e`/
  `aad9acd`, without checking that PR #279's own base ref still carried that exact stale phrase.)

## Honest boundary — unchanged from #264/#269, narrowed further

This closes the **negligence + fabrication** class at merge: an agent that skips
`/coderails:post-evals`, or hand-writes a `smoke` object after the fact, is now caught at merge by
re-execution against the trusted commit.

It does **not** resist an adversary who edits the enforcing hook itself — `hooks/scripts/` is not in
`TIER_GATE_PATH_DENYLIST` (verified: the denylist regex covers `scripts/tier-gate/`,
`skills/dashboard/`, `launchd/`, and `.github/workflows/` — see
[[tier-gate-path-denylist-dashboard_2026-07-21|the denylist investigation]] — `hooks/scripts/` is
absent), so `smoke_verify`'s own call sites in `enforce_pr_workflow.sh` are themselves
agent-editable. Same writer-side boundary the `tier_justification` and fixtures gates already
carry — see [[enforcement-model]]. Full mechanical closure against a genuinely adversarial
agent needs an attestor outside the agent's own trust domain: the
[[pr_232_tier-review-gate|tier-review daemon]] pattern, root-owned and outside the agent's write
access. That closure is evaluated and deliberately deferred, not attempted here.

## Provenance

Built by a dispatched sub-agent from a detailed brief (worktree, TDD, mutation-proof every refusal).
A follow-up pass independently re-verified the diff line-by-line against the brief, re-ran all six
affected suites directly rather than trusting the sub-agent's report, added the direct
`pr::coderails_eval_embed_for_head` tests, added the three negative-path wiring tests, and corrected
this description where an earlier draft claimed `smoke_verify` calls `validate_smoke_execution`
directly — it does not; it runs its own re-execution loop against `merge` scope, for the reasons
under "Why NOT a call into check 10" above.

## See also

- [[pr_264_smoke_run_executor_and_check9]] — check 9 and the `smoke-run` executor (freeze time)
- [[pr_269_gate_time_smoke_execution]] — check 10 (post time, advisory) and its named residual, which
  this PR narrows
- [[task-evals-gate]] — the design page; checks 1-10 plus this merge-time re-execution
- [[merge]] — `scripts/merge.sh`'s new smoke-verify gate block
- [[enforce_pr_workflow]] — the hook's new `gate_smoke_verify`, ordered before Gate 8
- [[pr_232_tier-review-gate]] — the attestor-outside-trust-domain pattern this PR's honest boundary
  points to as the only further mechanical closure
