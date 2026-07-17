---
title: "PR #194 + #198 — deferral gate and proof gate on LOOP-STOP: complete"
type: source
created: 2026-07-17
last_updated: 2026-07-17
sources: []
tags: [hook, agentic-loop, loop-stall-guard, complete-gate, deferral-gate, proof-gate, anti-gaming, work-units, proof-json]
---

# PR #194 + #198 — deferral gate and proof gate on `LOOP-STOP: complete`

Two new fail-closed conditions added to `loop_stall_guard.sh`'s `complete`
branch, both from the same one-day cluster (2026-07-16, session `a5289b41`):
four failure classes were caught by hand that day where every existing soft
control passed. Diagnosis: a rule *described* (in SKILL.md prose) is not a
rule *enforced* (by a hook). Both gates close that gap for a different class
of unproven/quietly-abandoned work.

## PR metadata

| Field | Value |
|---|---|
| PR number | #194 (deferral gate), #198 (proof gate) |
| Branch | `feat/deferral-gate`, `feat/proof-gate` |
| Merged | 2026-07-17 (both) |
| Merge SHA | `a1dc7238e195ea7d85267d250f759d7c86170059` (#194), `123b4910440a750bd32fdf9ff734fdf1eed1e487` (#198) |
| JIRA ticket | — |

## Summary

### PR #194 — `als_gate_work_units_on_complete` (the deferral gate)

Every `progress.json` `work_unit` must be `"done"` or `"dropped"` with a
non-empty `dropped_reason` before a loop can declare `LOOP-STOP: complete`.
Fails **open** at the file level (absent `jq`, absent/unset `ALS_PATH`,
absent/null/empty `work_units`, malformed `progress.json` — all allow,
since `work_units` is optional, same posture as the existing eval-threshold
read of the field). Fails **closed** per individual unit: any entry that
cannot be proven terminal (wrong status, or `dropped` with an empty/missing
`dropped_reason`) blocks. The allowlist is deliberately narrow —
`{done, dropped}` only, not the wider vocabulary (`merged`, `complete`, etc.)
seen historically, because widening it would rebuild the non-enforcement
this gate exists to remove.

Review found and fixed **two jq type-guard fail-open bypasses** before
merge: a non-object `work_unit` *value* (rather than a missing key) was
read as `status: null`, which is correctly not `"done"`/`"dropped"` — but
an earlier revision let a non-string `dropped_reason` slip past the
non-empty check via a type coercion. Both closed by explicit `type ==
"object"`/`type == "string"` guards before any field is read off a
work_unit entry.

### PR #198 — `als_gate_proofs_on_complete` (the proof gate)

A new enforcement **class**, not just a new check: every prior gate in this
hook (retro gate, deferral gate above) verifies artifact *presence/shape*
— a file exists and parses, a field carries the right value. This gate
instead **re-derives a verdict from the session transcript's raw Bash
`tool_use`/`tool_result` pairs** — exact trimmed-command match, last
execution wins, `is_error` decides pass/fail — checked against a
`proof.json` that was frozen earlier, at Phase 2.7e, by a **separate blind
agent** that saw only `progress.json.authorising_prompt_raw` (never the
plan, spec, or the orchestrator's own conversation). `proof.json`'s own
self-reported `.status` field is present in the schema but **never
consulted** for the verdict — a self-written `"status":"pass"` cannot
rescue a proof that was never actually run. That blind-authorship +
transcript-only-verdict combination is the anti-gaming design.

Scope is deliberately **orchestrator-session only**: a worker's Bash calls
never appear in the orchestrator's own transcript, so a proof satisfied
only inside a dispatched worker proves nothing to this gate — the same
grader-independence rationale [[task-evals-gate]] already established,
generalised here to proof *authorship* rather than grading.

Fails **open** on an absent `proof.json` (optional artifact, voluntarily
adopted, mirrors task-evals' posture). Fails **closed** on
malformed/unverifiable proofs, a missing/invalid `schema_version`, a
non-array `.proofs`, or a proof count exceeding a **100-proof cap**.

Review found and fixed four issues before merge:
1. **An O(proofs × executions) DoS-to-bypass route.** An earlier revision
   rescanned the full transcript once *per proof*. `loop_stall_guard` runs
   under a 15s `hooks.json` timeout, and a timed-out hook never exits 2 —
   the Stop proceeds **unblocked**. A model-writable, uncapped `proof.json`
   (reproduced at ~2,000 proofs × ~2,000 Bash calls ≈ 15s) could defeat the
   gate by making the *check itself* time out. Fixed with a single-pass
   command-match index (group executions by trimmed command, last wins —
   O(executions) to build, O(1) per-proof lookup) plus the 100-proof cap
   checked *before* any transcript mining starts.
2. **A full complete-gate-family bypass via one malformed transcript
   line.** `als_count_invocations`'s tolerant parse accepted any valid-JSON
   transcript line, including a non-object (bare array/number/string). A
   `.type` access on such a line threw, aborting the whole `jq` program —
   collapsing the invocation count to 0, which reads as "not a loop,"
   which skips every downstream gate including this one and the deferral
   gate. Fixed with an explicit `select(type == "object")` guard
   (`hooks/scripts/lib/loop_state_common.sh:169`) before `.type` is ever
   read, at both the top-level record and the nested content-array level.
3. **Two transient-jq fail-opens.** A `jq` failure reading `.proofs |
   length` (e.g. a truncated/mid-write `proof.json` racing the reader) was
   conflated with the legitimate "empty proofs, allow" case — both fell
   through to the same `return 0`. Fixed by numerically validating the
   count string (`''|*[!0-9]*` case match) *before* either comparison, so
   a jq failure now fails closed like every other unparseable-`proof.json`
   case in this gate.
4. **A background-launch gaming vector.** The transcript miner now
   excludes any Bash `tool_use` with `run_in_background: true` — a
   backgrounded launch's immediate tool_result is not a pass/fail outcome,
   so counting it as an "execution" would let a proof be satisfied by
   merely starting a process, not completing it. Phase 2.7e's proof
   command-authoring contract states this explicitly: proofs must run in
   the foreground.

**Trust boundary, stated explicitly in the gate's own source comment:**
this gate targets the *observed* failure class — honest self-deception and
lazy self-reporting (`"status":"pass"` written without ever running the
command) — not adversarial transcript forgery. A session that deliberately
appends forged `tool_use`/`tool_result` records to its own transcript file
(an ordinary writable file, not a sealed log) can defeat it. No
transcript-reading hook can close that; it is not this gate's claim.

## Design decisions recorded during this work

- **Q1 — YES.** `proof.json` is authored by a separate agent that sees
  only the authorising prompt. This generalises the existing
  [[task-evals]]/[[task-evals-gate]] grader-independence principle from
  *grading* to *authoring*: the same blind-spot argument (the orchestrator
  has already seen the plan by the time it could author proofs itself)
  applies to writing the proof set, not just judging it.
- **Q2 — NO.** `"blocked"` is not a valid terminal `work_unit` status for
  the deferral gate's allowlist. A loop that is blocked must declare
  `awaiting-input` or `hard-stop` (see [[loop_stall_guard]]'s
  `LOOP_STOP_VOCAB` table) — never `complete`. Widening the work_unit
  allowlist to accept `blocked` as terminal would let a stalled unit pass
  as finished, which is the exact non-enforcement this gate exists to
  remove.

## Files changed

- `hooks/scripts/lib/loop_state_common.sh` — both new `als_gate_*`
  functions (`als_gate_work_units_on_complete` at line 626,
  `als_gate_proofs_on_complete` at line 723), plus the
  `als_count_invocations` non-object-line guard fix (line 169)
- `hooks/scripts/loop_stall_guard.sh` — calls both gates in the `complete`
  branch (lines 66–67), after the existing retro gate
- `hooks/scripts/tests/loop_stall_guard.test.sh` — new test coverage for
  both gates, including the fail-open/fail-closed boundary cases and the
  DoS-cap regression
- `skills/agentic-loop/SKILL.md` — Phase 2.7e (blind proof author, the
  command-authoring contract: single self-contained command, absolute
  paths, foreground-only, exit-0-on-success); Phase 13 teardown step 1
  (run every proof verbatim, in the foreground, in the orchestrator's own
  session, before assembling the retro)
- `skills/agentic-loop/loop-state.md`, `skills/agentic-loop/teardown.md` —
  field spec and mechanics kept in lockstep with SKILL.md
- `AGENTS.md` / `README` / `REFERENCE` — kept in lockstep (repo-wide
  convention for any SKILL.md phase change)

## Wiki pages updated

- [[loop_stall_guard]] — new "Deferral gate on `complete`" and "Proof gate
  on `complete`" sections, joining the existing retro-gate section; now
  three `complete`-branch gates in call order (retro → work_units → proofs)
- [[loop-progress-fields]] — `work_units` gains a second consumer (the
  deferral gate, alongside the existing `>=3`-unit eval-threshold read)
- [[agentic-loop]] — Phase 2.7e and the Phase 13 teardown step 1 addition

## Caveats / gotchas

- The proof gate's transcript mining is **orchestrator-session-scoped by
  design**, not a limitation to fix later — a worker satisfying a proof in
  its own session is invisible to this gate on purpose, mirroring
  task-evals' grader-independence stance.
- Both gates share the file-level-fail-open-vs-entry-level-fail-closed
  asymmetry already established by the retro gate (mandatory artifact
  blocks on absence) vs. the eval-threshold read of `work_units`
  (optional, blocks on nothing when absent) — this cluster's two gates
  each pick one side of that asymmetry deliberately: `work_units` (file
  optional, entries strict) and `proof.json` (file optional, entries
  strict, PLUS a hard cap the retro gate doesn't need since retro.json
  isn't a model-inflatable array).
- `"blocked"` is explicitly excluded from the deferral gate's terminal
  vocabulary (Q2 above) — do not propose widening it without re-opening
  that decision.
