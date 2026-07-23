---
title: "PR #284 — check 10 hardening: whitespace cmds, duplicate ids, forking timeouts"
type: source
origin: "coderails PR #284 (merged 46550b4, 2026-07-23; supersedes the closed #277)"
created: 2026-07-23
last_updated: 2026-07-23
sources: []
tags: [source, task-evals, gate, post-evals, check-10, whitespace-bypass, process-group, fail-open, tier-review]
---

# PR #284 — check 10 hardening (three fixes)

Follow-up to [[sources/pr_269_gate_time_smoke_execution]]. Review of #269 found that its
gate-time re-execution (check 10) shipped three defects live on main; #284 closes them. (First
attempted as #277, which was rebased cleanly but could not be pushed without a force-push the
destructive-command guardrail blocks — see the fresh-branch note below.)

## The three fixes

1. **Whitespace-only cmd bypass (the CRITICAL).** A `cmd` of `"   "` executed as a no-op,
   exited 0, was not environmental, and — since cmd polarity is deliberately ungated — was
   **accepted**. A P0 check that verifies nothing passed the gate this whole layer exists to
   protect. Now `cmd` and `negative_control` are trimmed (the file's existing
   `gsub("^\\s+|\\s+$"; "")` idiom) before the empty-check and refused if blank. Adversarial
   review confirmed the trim handles NBSP / em-space / ideographic-space; zero-width chars
   aren't stripped but aren't valid bash no-ops either, so they land on 127/refused.

2. **Duplicate-id under-execution.** Check 10 looked eval commands up by id (`jq ... select(.id
   == $id)`), which returns only the *first* match — so a second eval sharing an id was never
   executed. Now iterates by **array index**, executing every eval.

3. **Unbounded forking timeout.** The 10s `perl alarm` only signalled the immediately-exec'd
   process. A command that forked a grandchild (`bash script.sh`, `npm test`) left the
   grandchild orphaned, reparented to PID 1, still holding the output pipe open — so the
   `out=$(...)` capture blocked until the *grandchild* exited (measured **31s** against a 10s
   cap). `_run_recorded` now `setpgrp`s the child into its own process group and the alarm
   handler `kill`s the whole negative-PGID group. Bounded near 10s; the 142 timeout sentinel
   is preserved. The sibling `_run_formula` was correctly left untouched — it discards output,
   so no orphan can hold a pipe open there.

## Tier: the judge's ruling, accepted not gamed

#284 first claimed **tier 1**. The tier-review daemon returned `verdict=illegitimate`: three
independent fixes meet its **tier-2 "≥3 work-units" predicate**. The tier-1 claim was
under-claiming. Rather than re-argue the ruled verdict (which reads as gaming) or force-merge
past the attestor, the claim was **re-tiered to 2** with one P0 eval per fix (whitespace,
duplicate-id, fork-bound) and re-judged `legitimate`. A worked example of the tier-review gate
([[sources/pr_232_tier-review-gate]]) doing exactly its job — catching an honest-but-wrong
tier claim and forcing it right.

## The gate caught its own PR's evals

While freezing #284's evals, its negative controls initially **exited 0** (they printed
"PASS"). Check 10 refused them — *"a control that passes proves nothing."* They were rewritten
as polarity witnesses that exit 1. The anti-vacuous-control rule from
[[sources/pr_218_discriminating-check-gate]] firing on the very PR that hardens check 10.

Two evals (the no-regression suite, and the fork-bound test) had to move `scripted → agent-run`
because each runs >10s and check 10's own gate-time cap would kill them as 142 — the fork-bound
eval verifies the very timeout that would kill it. Same handling as [[sources/pr_282_tier_gate_error_retry|#282]]'s E4.

## Fresh-branch-not-force-push

#277 was rebased onto current main cleanly, but pushing the rebased branch needed a force-push
the destructive-command guardrail blocks. Instead of editing the guardrail (a decision reserved
for the human) or deleting+recreating the branch (which auto-closed #280 earlier), the net
3-file diff — which applied cleanly on main — was put on a **fresh branch off current main** and
pushed with a plain non-force push. The guardrail never fired and was never touched. General
pattern worth reusing whenever a rebased branch meets a force-push guardrail.

## Known residual (pre-existing, filed separately)

Review surfaced a Critical that is **not** #284's: `smoke_verify` fails open on a malformed
non-array/non-object scalar `.evals` field. Reachable via `scripts/merge.sh` and
`hooks/scripts/enforce_pr_workflow.sh`, both of which extract the eval embed from the trusted
marker comment (validating only the marker line, never the embed's JSON shape) and hand it to
`smoke_verify`, bypassing checks 1-9. Verified pre-existing — `smoke_verify` is byte-identical
on `origin/main` and #284 touches neither file. Filed as a follow-up; the fix is to
shape-validate `.evals` is an array before `smoke_verify` trusts it, fail-closed otherwise.
