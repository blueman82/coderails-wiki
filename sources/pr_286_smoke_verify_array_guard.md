---
title: "PR #286 — fail closed when smoke_verify .evals is not a JSON array"
type: source
origin: "coderails PR #286 (merged 420004f, 2026-07-23)"
created: 2026-07-23
last_updated: 2026-07-23
sources: []
tags: [source, task-evals, gate, post-evals, smoke-run, merge-gate, fail-open, shape-guard, fabricated-evidence, honest-boundary]
---

# PR #286 — shape-guard `.evals` before the eval-index extraction trusts it

Merged 2026-07-23T17:10:29Z at `420004f` (branch `fix/smoke-verify-array-guard`). This is the fix
for the **pre-existing Critical [[pr_284_check10_hardening|PR #284]]'s review filed as a follow-up**:
*"`smoke_verify` fails open on a malformed non-array/non-object scalar `.evals` field... verified
pre-existing — byte-identical on `origin/main`... the fix is to shape-validate `.evals` is an array
before `smoke_verify` trusts it, fail-closed otherwise."* #286 is exactly that fix, plus the same
guard in the two sibling functions that share the defect.

## What it closes — a fail-open, one shape the gate never guarded

[[pr_279_merge_time_smoke_reexecution|`post_evals::smoke_verify`]] is the merge-time re-execution
gate: it re-runs a PR's scripted eval commands against the trusted head SHA to catch fabricated
evidence (a fake `cmd` resolves to 127, an honest one to its real exit code). It extracted the
scripted-eval **array indices** via a `jq to_entries` over `.evals`, then early-returned success when
extraction produced no indices — a legitimate case (an artifact whose only evals are `agent-run` has
no scripted `cmd` to re-run).

The defect: when `.evals` is a **scalar or string**, the `to_entries` jq errors to stderr with
**empty stdout**, so the "no indices → return 0" line fired and **passed the merge gate WITHOUT
re-executing anything**. A malformed eval artifact could merge with its evals never re-run — the exact
fabrication class this gate exists to prevent, one shape it did not guard.

Reachable via `scripts/merge.sh` and `hooks/scripts/enforce_pr_workflow.sh`, both of which extract the
embed via [[pr_279_merge_time_smoke_reexecution|`pr::coderails_eval_embed_for_head`]] — which
validates **only the marker line, never the embed's JSON shape** — so a forged malformed embed reaches
`smoke_verify` intact.

## The fix — a fail-closed type guard in THREE functions

A guard added after each function's tier-0 exemption, refusing unless `.evals` is a JSON array:

```sh
if ! jq -e '(.evals | type) == "array"' "$path" >/dev/null 2>&1; then
    printf 'post_evals: <fn>: .evals is not a JSON array (malformed or absent) — refusing.\n' >&2
    return 1
fi
```

Added to all three functions that iterate `.evals` (the PR body's prose names only two — it drops the
`validate_smoke`/check-9 guard; the diff adds all three):

| Function | Check | `scripts/post_evals.sh` |
|---|---|---|
| `post_evals::validate_smoke` | check 9 (freeze-time, recorded smoke) | ~231 |
| `post_evals::validate_smoke_execution` | check 10 (post-time, re-execution) | ~375 |
| `post_evals::smoke_verify` | merge-time re-execution | ~548 |

**Guards on TYPE, never on empty indices.** A valid array whose only evals are `agent-run`
legitimately has no scripted indices and must still be accepted — keying the refusal on "no indices"
would false-block that honest case. The guard fires only when `.evals` is not an array at all.

## The non-obvious detail — why the guard is load-bearing for the OBJECT shape

Check 7 (`validate_structure`: tier≥1 requires ≥1 P0 eval) already backstops the **scalar/string**
shapes on the live `validate_structure` chain — its `.evals[]?` finds no P0 in a scalar or string, so
check 7 refuses those first. But it does **NOT** backstop the **object** shape: `.evals[]?` iterates
an object's **VALUES**, so an object carrying a P0 among its values passes check 7 and reaches the
guard. For the object shape the new guard is the *only* thing that refuses it — load-bearing, not
belt-and-braces. (For `smoke_verify` at merge, which runs `merge` scope checks 1-9 before it, the same
reasoning holds; for the standalone reachability the guard stands on its own regardless.)

## The class — the third recurrence of #261's own generalization

This is the concrete recurrence of the lesson [[pr_261_freeze_before_build_gate|PR #261]] already
generalized on [[task-evals-gate]]: *"any check whose skip path keys on an empty read"* — an empty
`jq` result is indistinguishable from a legitimately-empty result, so the skip/early-return path reads
as compliance while verifying nothing. #261 was `frozen_sha` absent-vs-jq-missing; #286 is
`to_entries` jq-error → empty stdout → "no indices → return 0". (The wider eval-gate fail-open family
also includes [[pr_264_smoke_run_executor_and_check9|PR #264]]'s unwired executor, a different
mechanism — frame the class at #261's empty-read level, which #286 matches exactly, rather than as a
bare "third of a kind".)

## Verification (as reported in the PR)

- **6 regression checks** (marker `smoke_verify_rejects_non_array_evals`): scalar, string, object, and
  absent `.evals` each refuse with exit 1; a valid `agent-run`-only array is accepted with exit 0
  (proving the guard keys on type, not empty indices).
- **RED before the fix**: the scalar and string checks exit 0 against pre-fix code — the fail-open,
  reproduced.
- **Mutation-proven**: neutralising only the `smoke_verify` guard flips exactly the scalar and string
  checks red; restore returns green.
- **Reach-path test (SV9)**: composes the real embed extractor's fence-extraction awk with the real
  `smoke_verify` — proves a forged marker comment carrying `evals: 42` is preserved verbatim by the
  extractor (only the marker line is validated) and refused at the end of the chain `merge.sh` actually
  runs, not just when `smoke_verify` is called with a clean fixture.
- Full `post_evals` suite green; `merge_evals_gate`, `enforce_pr_workflow`, `merge_tier_review_gate`,
  and `merge` suites all green.

## Files changed

| File | +/− |
|---|---|
| `scripts/post_evals.sh` | +45 / −0 (three guards) |
| `hooks/scripts/tests/post_evals.test.sh` | +65 / −0 (checks S1-quater, SV8, SV9) |

## Honest boundary — unchanged from #279

Closes the **negligence + fabrication** class for the malformed-`.evals` shape at merge: a scalar,
string, or object `.evals` is now refused rather than silently skipped. It does **not** resist an
adversary who edits the enforcing hook itself — `hooks/scripts/` sits outside
`TIER_GATE_PATH_DENYLIST`, so `smoke_verify`'s own call sites in `enforce_pr_workflow.sh` are
agent-editable. Same writer-side boundary [[pr_279_merge_time_smoke_reexecution|#279]] already carries;
full closure against a genuinely adversarial agent still needs the [[pr_232_tier-review-gate|tier-review
daemon]] pattern, root-owned and outside the agent's write access. See [[enforcement-model]].

## See also

- [[pr_279_merge_time_smoke_reexecution]] — introduced `smoke_verify`; this PR hardens its `.evals`
  shape assumption
- [[pr_284_check10_hardening]] — the review that filed this Critical as a follow-up (byte-identical on
  main, not #284's)
- [[pr_261_freeze_before_build_gate]] — the empty-read fail-open class this is the concrete recurrence of
- [[pr_264_smoke_run_executor_and_check9]] — the sibling eval-gate fail-open (unwired executor,
  different mechanism)
- [[task-evals-gate]] — the design page; the shape guard is recorded in "Known gaps — status"
- [[merge]] / [[enforce_pr_workflow]] — the two consumers `smoke_verify` is reachable through
