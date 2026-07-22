---
title: "PR #264 — execute the freeze-time smoke-run (check 9 + the smoke-run executor)"
type: source
origin: "coderails PR #264 (merged 7096841, 2026-07-22)"
created: 2026-07-22
last_updated: 2026-07-22
sources: []
tags: [source, task-evals, gate, post-evals, smoke-run, negative-control, vacuous-pass, honest-boundary, freeze-before-build]
---

# PR #264 — execute the freeze-time smoke-run

> **Read with [[pr_269_gate_time_smoke_execution|PR #269]].** #264 shipped check 9 and the
> `smoke-run` executor, and named an open residual: nothing in the enforcement path ever ran
> the executor, so a hand-written `smoke` object of plausible shape passed. #269 merged the
> same day (two hours later) and closed most of that residual with check 10. This page records
> #264 as it stood at its own merge; **current state is [[task-evals-gate]]**.

## The gap

[[task-evals]]'s freeze-time smoke-run was a *prose mandate* with nothing recording or checking
its result. So an agent could freeze eval commands naming scripts that were only ever intended
to exist, and pair them with negative controls that passed without testing anything. Four real
failure instances from 2026-07-22 are the test set (verified: PR #264 body):

| # | Instance | What it looked like |
|---|---|---|
| a | Commands naming **five scripts that were never created** | The freeze recorded intent, not reality |
| b | A negative control whose file sat **outside git**, so the validator skipped and returned 0 | A *skip* that read as compliance |
| c | A control that "removed" `jq` by narrowing `PATH` to `/usr/bin:/bin` — where `jq` lives | Passed for the wrong reason |
| d | An assertion contradicted by reality | Caught only because that eval happened to be runnable at freeze |

Instance (b) is the same defect [[pr_261_freeze_before_build_gate|PR #261]] hit in its own
negative control, which is why that page forward-referenced this one.

## What shipped

**Check 9 (`post_evals::validate_smoke`)** — every pr-scope tier≥1 *scripted* eval must carry a
`smoke` object recording the observed exit codes of its `cmd` and `negative_control`. Absent or
non-numeric codes fail closed: "no recorded outcome" must never read as a compliant one.

**`smoke-run` (`post_evals::smoke_run`)** — a subcommand that *executes* those commands and
writes the `smoke` object itself, so the codes are **computed, never typed**. This is
[[task-evals]] rule 5 (a neutral script computes it, the author never hand-writes it) applied to
smoke evidence.

Both run commands through a 10s `perl alarm` wrapper (`post_evals::_run_recorded`), so 127
(not found), 142 (timeout sentinel) and signal deaths **fall out of a real run** instead of
being asserted.

## The load-bearing design constraint: shape, not polarity

This is the most transferable thing in the PR. Check 8 ([[pr_261_freeze_before_build_gate|PR
#261]], freeze-before-build) makes it mechanical that the feature is **not built at freeze** —
so `cmd` is *expected* to exit non-zero. **A gate keyed on `cmd` polarity would contradict
check 8 and block every honest freeze.**

So the two legs are judged differently:

- **`cmd` — judged on the SHAPE of its outcome, never its polarity.** 126 / 127 / 142 / ≥128
  means the command *never reached the artifact it claims to test*. A content failure (exit 1)
  is permitted and expected.
- **`negative_control` — judged on polarity AND shape.** It is defined to fail *regardless of
  build state*, so a control observed exiting 0 is vacuous by construction. But non-zero alone
  is insufficient: **a control that errors out on its own tooling is the same vacuous pass one
  level up** — the bug relocated, not fixed. So it must be non-zero AND not environmental.

That did-not-run vs ran-and-failed split is **the pass/skip/fail distinction applied only where
it is load-bearing** — deliberately *not* refactored through every check in the system. Instance
(b)'s root cause was exactly a skip being indistinguishable from a pass.

The environmental taxonomy (`post_evals::_is_environmental_rc`: 126, 127, 142, ≥128) is the same
one `validate_discriminating` applies to its `fixtures` legs — but that function keeps its own
inline checks deliberately, because it reports not-found/timeout/crash with three distinct
messages naming both legs' codes, which a shared boolean cannot express (verified:
`scripts/post_evals.sh` comment on `_is_environmental_rc`). If the taxonomy changes, both must.

## Why both halves were needed — the generalisable lesson

**A validator that merely READS recorded exit codes is insufficient, because the agent writes
those numbers.** An honest-but-optimistic author records the code they *expect* — `1`, "the
assertion fails until I build it" — never having run the command.

Proven empirically, not argued: driving failure instance (a) through check 9 alone returns
**rc=0**. It walks straight through, despite 24 green tests and passing mutation proofs. That is
why `smoke-run` exists. (verified: PR #264 body, "Verification")

**Recording is not judging.** `smoke_run` returns 0 even when what it observed is damning —
refusing is check 9's job. Keeping them apart means the recorded evidence is the same whether or
not anyone later gates on it.

## #264's honest boundary — and how #269 moved it

As merged, `post_evals::smoke_run` had **zero code callers**: the enforcement path
(`merge.sh` → `validate_structure` → `validate_smoke`) never executed it. #264 argued this was
*structural* rather than a wiring oversight — at freeze the feature is unbuilt and at merge it
is built, so the codes could not be recomputed at the gate. Verified empirically at the time: a
hand-written `smoke` object of plausible shape (`cmd_exit: 1`, control non-zero and
non-environmental) is not mechanically distinguishable from a recorded one and passes the full
enforcement path at rc=0.

So at #264's merge: **closed** = the honest-but-optimistic author, and the vacuous-control class
once the executor is used. **Open** = an author who never runs the commands at all — which is
what produced instance (a).

> ⚠️ **This boundary is superseded.** [[pr_269_gate_time_smoke_execution|PR #269]] adjudicated
> the "cannot be recomputed" claim as **true for one dimension and false for two**: `cmd`
> *polarity* is genuinely unrecomputable, but *resolvability* (both legs) and *control polarity*
> are build-independent and now ARE recomputed at the gate by check 10. Instance (a) is closed
> on main. Do not cite this section as current state.

## The excerpt cap — a constant measured, not chosen

The output-excerpt cap in `_run_recorded` was **measured against real output from this repo**
rather than picked freehand, and the measurement **found a defect**: a tail-only excerpt keeps a
test runner's verdict (last line — `post_evals.test.sh` emits 10886 chars, `PASS` last) but
discards a node stack trace's error, which sits on the **first** line followed by 900+ chars of
stack frames. That loses exactly the module-resolution tell (`ERR_MODULE_NOT_FOUND`)
[[task-evals]]'s SKILL.md names as the broken-instrument signature.

Fixed to keep **both ends** (`${out:0:250} [...] ${out: -250}`), locked with a test,
mutation-proven. Instance of the general lesson: *a freehand constant is an unverified claim
about the world; sample the real population before setting a threshold.*

## Scope

**pr scope only**, matching check 8's boundary — and stated as a deliberate choice, not a
technical limit (check 9 needs no repository, only the recorded outcome). Loop-scope artifacts
are gated by [[loop_state_guard]], a separate surface with its own callers, so extending the
smoke contract there stays its own decision rather than riding in as a side effect. Tier 0 is
exempt (its `evals` array is empty by definition) and agent-run evals are exempt (no `cmd` to
execute).

`smoke` carries **no `schema_version` bump**: it is additive, and loop-scope files tolerate its
absence exactly as before.

Instance (d) needed no new mechanism — the existing smoke-run caught it.

## Verification (as reported in the PR)

- Instances (a), (b) and (c) reproduce live at their real exit codes (127, 0, 0) and are each
  refused with a named reason once the executor has run.
- An honest freeze-before-build artifact still passes — the case proving the gate does not
  simply refuse everything.
- Before/after on one artifact: check 9 alone rc=0; executor overwrites `cmd_exit` 1 → 127;
  check 9 refuses.
- The residual is **proven, not asserted**: a hand-written object bypassing the executor passes
  at rc=0.
- Both refusals mutation-proven — neutralising each flips only its own tests.
- **No test in the executor suite hand-writes an expected exit code** — each asserts on what
  `smoke-run` observed, so a fixture cannot pass by restating the assumption in question.
- `post_evals` 160 checks green, plus `merge_evals_gate`, `loop_state_guard_evals`,
  `eval-artifact`, `discriminate`, `enforce_pr_workflow` and `merge`.

## See also

- [[pr_269_gate_time_smoke_execution]] — check 10, which closes most of the residual above
- [[pr_261_freeze_before_build_gate]] — check 8, the constraint that forces shape-not-polarity
- [[task-evals-gate]] — the design page; current state of all ten checks
- [[task-evals]] — the skill defining the smoke-run mandate and rule 5
- [[pr_218_discriminating-check-gate]] — the `fixtures` gate; a distinct defect class
  (formula can't discriminate), not the can't-execute class
