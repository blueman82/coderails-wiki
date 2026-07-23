---
title: "PR #269 — gate-time re-execution of eval commands (check 10)"
type: source
origin: "coderails PR #269 (merged 44ad17e, 2026-07-22)"
created: 2026-07-22
last_updated: 2026-07-23
sources: []
tags: [source, task-evals, gate, post-evals, smoke-run, re-execution, fabricated-evidence, honest-boundary]
---

# PR #269 — check 10, gate-time re-execution

> **Narrowed by [[pr_279_merge_time_smoke_reexecution|PR #279]]** (2026-07-23, next day). This
> page's own residual said check 10 only ever ran in the posting agent's own session at post
> time — never re-checked at merge, so a hand-written `smoke` object still passed the merge gate.
> #279 closes exactly that: it re-executes the same shape rules at merge, against the trusted
> head SHA, making the property binding rather than advisory. The residual PR #279 does *not*
> close — an unforgeable freeze-time stamp, or an attestor outside the agent's trust domain — is
> stated below and still stands. **Current state is [[task-evals-gate]].**

Merged 2026-07-22 at 21:25 UTC, two hours after [[pr_264_smoke_run_executor_and_check9|PR
#264]] (19:30). The two are one feature landed in two commits — read them together.

## What it closes

#264 shipped check 9 (the *shape* of recorded smoke evidence) plus a `smoke-run` executor, but
**nothing in the enforcement chain ever executed `smoke-run`**, and check 9 trusts numbers the
author types. Reproduced on main before this change: an artifact whose `cmd` is
`bash tests/this_file_was_never_created.sh` (real exit 127) carrying a hand-written
`"smoke": {"cmd_exit": 1, "negative_control_exit": 1}` passed `validate-structure` at **rc=0**.

> **An author who never ran the commands shipped green; only honest authors got checked.**

That is the sharpest statement of the whole arc: a gate that reads self-reported evidence
penalises exactly the population that complies with it.

## The mechanism

`post_evals::validate_structure` now carries **check 10** (`post_evals::validate_smoke_execution`).
At pr scope it **executes** every tier≥1 scripted eval's `cmd` and `negative_control` right
there at the gate, under the same 10s alarm wrapper `smoke-run` uses, and **judges only what it
observes — it never reads the typed `smoke` numbers at all**.

It refuses:

- `cmd` observed environmental — 126 / 127 / 142 / ≥128 — **including an empty `cmd`**
  (unresolvable by definition, fail closed; check 3 already refused an empty
  `negative_control`, but the empty-`cmd` case had no owner before this)
- `negative_control` observed environmental
- `negative_control` observed exiting 0 — vacuous at the gate, **whatever the recorded smoke
  claims**

It deliberately does **not** judge `cmd` polarity. See below.

## Adjudicating #264's "structural" claim

#264 said the codes "cannot be recomputed at the gate" because the feature is unbuilt at freeze
and built at merge. #269 found that **true for one dimension and false for two** — the useful
decomposition, and the reason this PR was possible at all:

| Dimension | Build-dependent? | Gated at merge? |
|---|---|---|
| `cmd` **polarity** (passes vs fails) | Yes — a cmd failing at freeze legitimately passes at merge | **No.** Gating it would contradict check 8 and block every honest freeze |
| **Resolvability** (both legs can execute at all) | No — build-independent at the moment of observation | **Yes**, now recomputed |
| `negative_control` **polarity** | No — check 9's own doctrine: the control "is defined to fail regardless of build state" | **Yes**, now recomputed |

The lesson generalises past this gate: *"can't be recomputed" is rarely true of a whole
artifact — decompose it, and usually only one dimension is genuinely build-dependent.*

One nuance stated honestly in the PR: resolvability is recomputed **at the gate, not as of
freeze**. A file absent at freeze can exist by merge (the implementation may create it), so
check 10 proves the eval is *executable at judgment time*, not that the freeze-time run
happened.

## Executing author-supplied strings at the gate — considered and accepted

Check 10 executes command strings out of a JSON file. The reasoning for accepting that
(verified: PR #269 body and the `validate_smoke_execution` header comment):

- **No privilege delta.** The strings come from the same principal that already runs arbitrary
  Bash in this environment — `smoke-run` executes these exact strings at freeze as the
  sanctioned flow, and the test gate runs whole suites. The gate runs them unprivileged, output
  discarded (an excerpt lands only in a refusal message), each capped by the 10s alarm.
- **The alternative fails open.** Statically resolving "the command's target path" means parsing
  shell — pipes, `&&`, quoting, env prefixes. Any parser shortfall becomes an
  accepted-without-checking path, which is [[pr_261_freeze_before_build_gate|PR #261]]'s
  fail-open lesson relocated. **Execution is the only judgment that cannot be confused about
  what the command does.**
- **Cost**: worst case 20s per scripted eval (two capped runs). Already implied by the smoke
  contract — a >10s cmd was already refused at freeze via a recorded 142.

## Residual, now narrowed to one dimension

- **Type a fake freeze-time `cmd_exit`.** An author whose commands resolve and whose control
  fails at gate time, but who never ran anything at freeze, still passes. That is exactly the
  polarity dimension, and it is demonstrably unrecoverable at merge. Closing it needs an
  unforgeable freeze stamp, or an attestor outside the agent's trust domain — the
  [[pr_232_tier-review-gate|tier-review daemon]] pattern.
- **An eval whose `cmd` resolves but checks nothing useful.** Semantic quality stays review
  territory, the same boundary check 4's structural floor already carries.
- **Create the missing script between freeze and merge.** The gate then sees a resolvable cmd
  and the freeze-time fabrication is invisible. The eval is at least genuinely executable at
  merge, which is what the gate can truthfully assert.
- **cwd/PATH tricks** so the cmd resolves at the gate while the "real" check differs. The gate
  runs from the same cwd contract as `smoke-run` (repo root, via the post-evals command), so
  this requires the author to sabotage their own artifact's commands in plaintext, visible to
  review.

## Scope and back-compat

Mirrors checks 8/9 exactly: **pr scope only**, tier 0 exempt, agent-run evals exempt, loop scope
untouched ([[loop_state_guard]] keeps its separate surface). Fail-closed `command -v jq` guard,
matching `validate_freeze`.

One side effect worth recording: **test fixtures that legitimately pass `validate_structure`
end-to-end now need genuinely resolvable commands** — because the gate really runs them. The old
`cmd: "run-a"` placeholder fixtures were themselves fabricated evidence that the pre-#269 gate
tolerated. The gate caught its own test suite.

## Verification (as reported in the PR)

Evals frozen at `frozen_sha` b951d98 (branch base), smoke recorded by the real `smoke-run`
executor, result computed by `compute-result` and never hand-written: **GO**.

| suite | result |
|---|---|
| `post_evals.test.sh` | PASS, 175 ok, 0 fail |
| `merge_evals_gate.test.sh` | PASS, 26 ok, 0 fail |
| `eval-artifact.test.sh` | PASS, 26 ok, 0 fail |
| `loop_state_guard_evals.test.sh` | PASS, 46 ok, 0 fail |

## See also

- [[pr_264_smoke_run_executor_and_check9]] — check 9 and the `smoke-run` executor
- [[pr_261_freeze_before_build_gate]] — check 8, whose freeze-before-build constraint is why
  `cmd` polarity must stay ungated
- [[pr_279_merge_time_smoke_reexecution]] — next day: makes this check's re-execution property
  binding at merge, narrowing the residual below
- [[task-evals-gate]] — the design page; current state of the check set
- [[task-evals]] — the skill, whose honest-boundary section was updated by this PR
