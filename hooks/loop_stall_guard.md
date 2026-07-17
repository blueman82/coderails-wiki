---
title: "Hook: loop_stall_guard.sh"
type: hook
created: 2026-06-25
last_updated: 2026-07-17
sources:
  - sources/session_2026-06-25_agentic-loop-upgrade-arc.md
  - sources/pr_49_gate-function-rename.md
  - sources/pr_76_harden-hook-stdin-read.md
  - sources/pr_96-98_mode-aware-install-argument-injection-guard-hook-owned-counter.md
  - sources/pr_86-107_2026-07-08_loop-lib-residuals.md
  - sources/pr_118-123_self-improving-loops.md
  - sources/pr_184_185_186_loop-cost-tracking.md
  - sources/pr_194_198_loop-complete-deferral-and-proof-gates.md
  - sources/pr_204_cost-reporter.md
  - sources/pr_206_208_loop-state-common-docs-and-robustness.md
  - sources/pr_224_231_233_235_loop-tooling-hardening.md
tags: [hook, agentic-loop, anti-stall, stop-hook, loop-stop, loop-state, hook-owned-counter, malformed-transcript, retro-gate, schema-version-2, cost-tracking, deferral-gate, proof-gate, work-units, anti-gaming, cost-reporter, reporter-not-gate, system-message, als_extract_last_text, withdrawn-proofs, als-pending-sysmsg]
---

# loop_stall_guard.sh (C2)

Stop hook that blocks an active, incomplete agentic loop from stopping without a `LOOP-STOP: <category> — <reason>` declaration. The declaration-based anti-stall layer (C2). Sits after `loop_state_guard` (C1) in Stop hook order.

Source: `coderails/hooks/scripts/loop_stall_guard.sh`
Shared lib (Gates 1–4): `coderails/hooks/scripts/lib/loop_state_common.sh` (extracted PR #49; `als_gate_*` functions shared with [[loop_state_guard]])

## Event and mode

| Field | Value |
|---|---|
| Event | `Stop` |
| Mode | block (exit 2) |
| Timeout | 15s (hooks.json) |
| Stop order | 4th (last Stop hook; after confidence, verify, loop_state_guard) |

## What it enforces

**Declaration presence + valid category.** When an agentic loop is active and incomplete, the orchestrator's stopping turn must contain a line matching `LOOP-STOP: <category> — <reason>` where `<category>` is from the fixed vocabulary. It does NOT judge whether the reason is legitimate (honest boundary: forces categorised declaration, cannot force honesty).

## The LOOP-STOP declaration

```
LOOP-STOP: <hard-stop|approval-gate|awaiting-input|complete> — <reason>
```

**Vocabulary** (defined once in `lib/loop_state_common.sh` as `LOOP_STOP_VOCAB`, consumed by both the regex and the block message):

| Category | Maps to |
|---|---|
| `hard-stop` | Verification failure, premise disproven, ambiguous out-of-envelope decision, unauthorised destructive op |
| `approval-gate` | Named risk boundary the envelope flagged for sign-off (pause-then-proceed) |
| `awaiting-input` | Planned interaction point inside the loop (Phase -1 improve-prompt ask, Phase 1 plan confirmation) |
| `complete` | All authorised work done — **MUST also set `progress.json status: "complete"` and run Phase 13** |

The `complete` ⇒ teardown coupling is load-bearing: a `LOOP-STOP: complete` tag in text only satisfies the current turn's gate. If the model declares `complete` in text but leaves `status: in-progress`, every later stop still demands a tag and C1 still treats the loop as active. The two actions are atomic in SKILL.md.

## Logic: skip gates (cheap first)

Gates 1–4 are implemented as named `als_gate_*` functions in `loop_state_common.sh` (shared with [[loop_state_guard]] — extracted PR #49, formerly byte-identical between the two guards). Gates 5–6 are local to this script.

1. **`als_gate_no_transcript`** — allow.
2. **`als_gate_stop_hook_active`** — allow (avoid stop-loop).
3. **`als_gate_not_a_loop`** — allow (not a loop). Same structured `jq` detection as C1, via `loop_state_common.sh`. This gate's underlying transcript parse is the same one made malformed-line-tolerant by [[pr_86-107_2026-07-08_loop-lib-residuals|PRs #91/#107]] — see [[loop_state_guard]]'s "Malformed-transcript tolerance" section for the fix detail; this hook shares the fix via the common lib, nothing local changed.
4. **`als_load_progress` / done-and-not-rearmed check** — allow if `status == "complete"` AND not re-armed AND session-owned (shared off-switch with C1).
5. **Last assistant message contains a valid `LOOP-STOP` line** — allow (regex built from `LOOP_STOP_VOCAB`; category must be followed by a non-alphanumeric char or end-of-line so "completed" doesn't match "complete"). On a valid declaration, three `complete`-only sub-gates run in order before the allow: `als_gate_retro_on_complete` → `als_gate_work_units_on_complete` (PR #194) → `als_gate_proofs_on_complete` (PR #198) — see their own sections below. Each can itself `exit 2` and block, so "allow" here means "declaration present AND every `complete`-only sub-gate passed." A fourth `complete`-only step, `als_report_cost_on_complete` (PR #204), then runs after all three gates and before `bump_loop_stop_count` — it is a reporter, not a gate, and never affects "allow" (see its own section below).
6. **BLOCK (exit 2)** — active + incomplete + no valid declaration.

## Block message design

The block message contains the **exact copy-paste tag template** built from `LOOP_STOP_VOCAB`, stating the `complete` ⇒ teardown coupling. The model copies the template; it never reconstructs the format. (Same lesson as C1's path-in-block-message: remove any derivation the model must do.)

## Multi-hook thrash risk

A stopping turn inside an active loop now carries three co-requirements: confidence labels, DNV resolution, and `LOOP-STOP`. The agentic-loop skill's Phase 0.5 bundles all three into one "stop-ceremony" note so the model emits them together rather than thrashing one hook while satisfying another.

## Phase 13 stall metric

`progress.json` gains a `loop_stop_counts` object (`{hard-stop, approval-gate, awaiting-input, complete}`). Phase 13 surfaces it. The `awaiting-input` count is the primary **avoidable-stall** signal — if a model rubber-stamps `awaiting-input` to escape the gate, it shows up in its own audit rather than hiding behind a technically-valid tag.

**Counter is now hook-owned, sole-writer (PR #98, merged 2026-07-05).** This
hook — not the orchestrator — increments `loop_stop_counts.<category>` after
validating the declared category, via a tmp-file `jq` read-modify-write (same
pattern as `scripts/post_review.sh`'s `write_cache`). Previously the counter had
**two writers** (this hook's presence check plus orchestrator prose asking the
agent to self-maintain the field) that raced under concurrent Stop-hook
invocations, undercounting 2 loops in a recorded session. `SKILL.md` now states
the field is HOOK-OWNED in 5 places — the orchestrator reads it as-is and
carries it forward verbatim on any progress.json rewrite, same treatment as
`completed_marker`. The write is best-effort and never fatal: a missing
`progress.json`, malformed JSON, absent `jq`, or a failed `mv` is logged and
swallowed, and the hook still exits 0 to let the declared stop through. See
[[spec-plan-progress-artifact-chain]] for the full race analysis and
[[pr_96-98_mode-aware-install-argument-injection-guard-hook-owned-counter]] for
the fix detail (including four standing-invariant tests added in a post-review
round: no-clobber, jq-absence fail-open, last-declaration-wins tie-break,
degraded-filesystem safety).

## Log output

Appends a `key=value` line to `$CLAUDE_DISCIPLINE_LOG`:
`hook=loop_stall_guard session=<id> invocations=<n> declared=<0|1> blocked=<0|1>`

## Known limitations

- Cannot force the declared reason or category to be truthful; a model can rubber-stamp `awaiting-input`. The Phase 13 KPI is the auditable counter-pressure.
- Four Stop hooks (confidence, verify, C1, C2) firing on the same stopping turn is the highest ceremony cost in the system; the main thing to watch in Phase 13 is avoidable-stall counts from stop-ceremony thrash. (A fifth Stop hook, [[unregistered_loop_guard]], was added PR #17 — it only fires on *unregistered* loops and is a nudge, not a co-requirement of this ceremony, so it does not add to this specific thrash risk.)

## Retro gate on `complete` (PR #119)

This hook additionally hosts the **retro gate** (`als_gate_retro_on_complete` in the shared lib). On a `LOOP-STOP: complete` declaration only, it blocks (exit 2) unless a parseable `retro.json` sits beside `progress.json` — the Phase 13 write contract. Category is lowercase-normalised before the check (a case-insensitive upstream match feeding a case-sensitive compare was the Critical caught in review — `Complete`/`COMPLETE` had bypassed the gate entirely). **Block-before-bump:** the gate call runs before `bump_loop_stop_count`, so a blocked `complete` never increments the counter. Fail-open when jq is absent (matches `bump_loop_stop_count`), with a `retro_gate=skipped_no_jq` breadcrumb; fail-closed when `ALS_PATH` is unset. Presence + parse only — provenance/content fidelity is not checkable here, same honest boundary as every other guard. See [[pr_118-123_self-improving-loops]].

**Gate widened to `schema_version >= 1`, forward-compatible (PR #186, 2026-07-15).** Originally required an **exact** `schema_version` 1; now accepts any value `>= 1` (non-numeric, absent, or `< 1` still block). This is what let Phase 13 bump the retro's own schema to 2 — see below — without needing a matching same-PR gate change. `loop_stall_guard.test.sh` carries both directions as explicit controls: schema_version 1/2/99 all allow (forward-compat proof), 0/absent/non-numeric all still block. See [[pr_184_185_186_loop-cost-tracking]].

**`retro.json` is now `schema_version` 2 as of the same cluster** — Phase 13's teardown write contract gained a cost-mining sub-step ([[loop_cost]]'s `dc_mine_token_usage`) that writes `retro.cost` (the miner's frozen per-model token/USD breakdown) and lifts its `models_used` array to top-level `retro.models_used`. This gate itself is unchanged by that addition beyond the version-check widening above — it still checks presence + `schema_version`, never the cost field's correctness, so a miner failure (which fails open to empty `cost`/`models_used`) cannot stall a loop. See [[agentic-loop]]'s Phase 13 section for the full field contract and [[pr_184_185_186_loop-cost-tracking]] for the source record.

## Deferral gate on `complete` (PR #194)

`als_gate_work_units_on_complete` in the shared lib, called immediately
after the retro gate above. On a `LOOP-STOP: complete` declaration only,
every `progress.json` `work_unit` must be `"done"` or `"dropped"` with a
non-empty `dropped_reason` — an offender list of unit ids is built and
printed in the block message so the model knows exactly which units to
finish or explicitly drop.

**Fail-open at the file level, fail-closed per unit** — this is the
opposite asymmetry from the retro gate on purpose: `work_units` is
optional (a trivial or legacy loop may never populate it, so its *absence*
must never block), but once an entry exists it must be provably terminal,
so a malformed or ambiguous entry blocks rather than passing by default.
Fails open on: absent `jq`, unset/missing `ALS_PATH`, absent/null/empty
`work_units`, or a `progress.json` that doesn't parse. The allowlist is
deliberately narrow — `{done, dropped}` only, not the wider vocabulary
(`merged`, `complete`, etc.) seen in historical loops; widening it would
rebuild the non-enforcement this gate exists to remove. `"blocked"` is
explicitly NOT a terminal status here — a blocked unit must resolve to
`awaiting-input` or `hard-stop` at the loop-declaration level, never ride
through as an implicitly-finished work unit (recorded design decision,
[[pr_194_198_loop-complete-deferral-and-proof-gates]]).

Review found and fixed two jq type-guard fail-open bypasses before merge:
a non-object `work_unit` value and a non-string `dropped_reason` could
each slip past the terminal check via type coercion rather than an
explicit missing-field read. Both closed with `type == "object"`/`type ==
"string"` guards before any field is read off an entry. See
[[loop-progress-fields]] for `work_units`' full schema and its other
consumer (the eval-threshold gate in [[loop_state_guard]]).

## Proof gate on `complete` (PR #198)

`als_gate_proofs_on_complete` in the shared lib, called after the
deferral gate above. A new enforcement **class**, distinct from every
other gate in this hook: instead of checking artifact presence/shape, it
**re-derives a pass/fail verdict from the session transcript's raw Bash
`tool_use`/`tool_result` pairs** (exact trimmed-command match, last
execution wins, `is_error` decides) and checks that verdict against a
`proof.json` frozen earlier — at Phase 2.7e — by a **separate blind
agent** that saw only `progress.json.authorising_prompt_raw`. `proof.json`
carries its own `.status` field but the gate **never consults it** — a
proof cannot be marked passing by self-report; only a matching, non-error
transcript execution satisfies it. This is the anti-gaming design:
blind authorship (Phase 2.7e) generalises [[task-evals]]'s
grader-independence principle from grading to *authoring*, and
transcript-only verdicts close the loophole a self-written `"status":
"pass"` would otherwise open.

**Orchestrator-session scope is deliberate.** A dispatched worker's Bash
calls never appear in the orchestrator's own transcript, so a proof
satisfied only inside a worker's session proves nothing to this gate —
proofs MUST be run by the orchestrator itself, in the foreground, per
Phase 13 teardown step 1 in [[agentic-loop]]. A backgrounded Bash launch
(`run_in_background: true`) is excluded from the transcript miner
entirely, closing a gaming route where merely starting a process (not
completing it) could count as satisfying a proof.

**Fail-open on absent `proof.json`** (optional, voluntarily adopted, same
posture as task-evals). **Fail-closed** on: malformed JSON, missing/invalid
`schema_version` (`>= 1` required), a non-array `.proofs`, an unparseable
proof count, or a proof count exceeding a **100-proof cap**.

Review found and fixed three issues before merge, beyond the type-guard
class above:
1. **O(proofs × executions) DoS-to-bypass.** An earlier revision rescanned
   the whole transcript once per proof. This hook runs under a 15s
   `hooks.json` timeout, and a timed-out hook never exits 2 — the Stop
   proceeds unblocked. A model-writable, uncapped `proof.json` (~2,000
   proofs × ~2,000 Bash calls ≈ 15s, reproduced) could defeat the gate by
   making the check itself time out, not by satisfying it. Fixed with a
   single-pass command-match index (executions grouped by trimmed command,
   last wins, built once) plus the 100-proof cap checked *before* any
   transcript mining starts — O(proofs + executions) instead of
   O(proofs × executions).
2. **A full complete-gate-family bypass via one malformed transcript
   line.** `als_count_invocations`'s tolerant parse accepted any
   valid-JSON line, including a non-object. A `.type` access on such a
   line threw, collapsing the invocation count to 0 — which reads as "not
   a loop," skipping every downstream Stop-hook gate, this one and the
   deferral gate included. Fixed with an explicit `select(type ==
   "object")` guard before `.type` is read, both at the top-level record
   and inside nested content arrays (`hooks/scripts/lib/loop_state_common.sh:169`).
3. **Two transient-jq fail-opens**, closed by numerically validating the
   proof-count string before either downstream comparison, instead of
   letting a jq read failure fall through to the same `return 0` as the
   legitimate "zero proofs" case.

**Trust boundary (stated explicitly in the gate's own source comment):**
this gate defends against the *observed* failure class — honest
self-deception and lazy self-reporting — not adversarial transcript
forgery. A session that deliberately appends forged
`tool_use`/`tool_result` records to its own transcript file can defeat it;
no transcript-reading hook can close that, and this gate makes no stronger
claim.

## Proof withdrawal — `withdrawn_proofs` (PR #224)

`proof.json` may also carry a `withdrawn_proofs` array — a way to say "I ran
this proof, it failed, and I'm not fixing it, here's why" instead of the
only two prior options (satisfy the proof, or never declare `complete`).
Mined by the **same** single-pass `$exec_index` the `.proofs` check above
builds, not a second scan — same DoS-to-bypass mitigation.

A `withdrawn_proofs` entry must have **actually executed** in-session, its
**last execution's `is_error` strictly `true`** (stricter than `.proofs`,
which tolerates `is_error: null`; a withdrawal claims a *witnessed* failure,
so `null`/`false` both fail), a **non-empty `withdrawn_reason` and `cmd`**,
and **no double-dip** against `.proofs` (the same id can't be both pending
and withdrawn). The cap from the proof gate above is **combined**:
`len(.proofs) + len(.withdrawn_proofs) <= 100`, not 100 each — both arrays
feed the same mining pass, so splitting the cap per-array would recreate the
~100×100 timeout shape the cap exists to rule out.

**Edge case fixed by this PR:** the pre-withdrawal gate's `.proofs`
absent/null path was a bare `return 0` ("nothing to prove"). That's wrong
once `withdrawn_proofs` exists — a loop whose only proof was withdrawn
produces exactly `.proofs` absent + `withdrawn_proofs` populated, which the
old early-return would let sail through unvalidated. The true "nothing to
prove" allow now requires **both** arrays empty; either one alone still runs
the full mining pass.

Fail-closed throughout, same posture as `.proofs` — a withdrawal that fails
any check blocks `exit 2`. Only the final human-visible message (naming
which ids were successfully withdrawn) inherits a never-block posture, and
only because it runs after every validation already passed.

## `ALS_PENDING_SYSMSG` — one merged systemMessage, not two colliding ones (PR #224)

This hook's `als_gate_proofs_on_complete` (withdrawal notices, above) and
`als_report_cost_on_complete` (below) both need to tell the human something
on a `complete` declaration, but two top-level `{systemMessage: ...}` JSON
objects concatenated on one hook's stdout is not valid as a single document
under a whole-buffer parse. Both functions now append their text to a
shared global accumulator, `ALS_PENDING_SYSMSG` (defined in
`loop_state_common.sh`, appended via `als_append_pending_sysmsg`,
newline-joined) instead of emitting JSON directly. `loop_stall_guard.sh`'s
call site emits the single merged `{systemMessage: ...}` only after both
gates have had their chance to append — the only place either message
reaches the human's terminal.

## Cost reporter on `complete` (PR #204) — the one step here that never blocks

`als_report_cost_on_complete` in the shared lib, called after the proof gate
above and before `bump_loop_stop_count`. Unlike every other function on this
branch, it is named `..._report_...` deliberately: **every path returns 0.**
It exists because `skills/agentic-loop/teardown.md` already said, in bold,
that a `complete` loop "must print" its cost — prose that loop `0d3fb487`
read, then silently skipped anyway (and fabricated an explanation for the
omission). Prose can't enforce prose; this closes that gap mechanically,
independent of what the model chooses to do.

**Deliberately inverts this file's fail-toward-blocking idiom.** Every other
gate here (and `check_verify_loop.sh`/`check_confidence_labels.sh` elsewhere)
runs its check and lets a failure exit non-zero. This function does the
opposite on purpose: `dc_mine_token_usage`
([[loop_cost]]:7-12, see
[[pr_184_185_186_loop-cost-tracking]]) is contractually fail-open to `{}` and
"must never block a caller" — a legitimately empty `.cost` on an
otherwise-valid, already-finished loop is expected, not an error state. A
fail-closed reporter would deadlock a loop that has ALREADY cleared the
retro/work_units/proof gates above it — strictly worse than the
unrecorded-cost bug it exists to fix.

**Behaviour matrix**, keyed off `retro.json`'s `schema_version` and `.cost`
(the retro is already proven present/parseable by the retro gate above, so
this function doesn't re-check that):

| Condition | Output |
|---|---|
| `schema_version < 2` (legacy, pre-cost-miner) | silent |
| `schema_version >= 2`, `.cost` populated | `Loop cost: $<usd> (<tokens> tokens), prices as of <date>, N days old` |
| `schema_version >= 2`, `.cost` non-empty but missing a required field | `cost recorded but incomplete (missing <field(s)>)` |
| `schema_version >= 2`, `.cost == {}` (miner failed open) | `cost unavailable (miner returned no data)` |
| `schema_version >= 2`, `.cost` absent (teardown skipped the mining step) | `cost not recorded` |

The four non-legacy rows are deliberately distinct messages — `schema_version`
is the discriminator between "legacy loop, nothing to report" and "sv2 loop,
teardown skipped a step it should have run," since both leave `.cost` absent.
Collapsing any two into one message, or into a silent return, would recreate
the exact failure this reporter exists to close.

**Anti-fabrication, not just anti-silence.** Every branch follows
"visibly-wrong beats plausibly-fabricated": a non-scalar cost field is
selected to empty rather than rendered as a multi-line `jq -r` pretty-print
blob; `printf '%.2f'` on a non-numeric input is gated behind a numeric-string
check first (it otherwise silently prints `$0.00` — verified); a
`prices_as_of` staleness date is gated on the exact `YYYY-MM-DD` shape before
`date` parses it (macOS `date -j -f "%Y-%m-%d"` silently accepts trailing
garbage like `"2026-06-24FORGED"` — verified) and prints the raw string
otherwise; the final message is stripped to printable+space/tab characters
before reaching the terminal. The discipline-log entry records only the
outcome class (`reported`/`miner_failed_open`/`cost_absent`/
`cost_incomplete`), never the interpolated message body.

**New precedent: the first coderails hook to use `systemMessage`.** Every
prior hook in this repo delivers to the model via `additionalContext`
(model-visible only — Claude Code's own docs state it never appears as a
chat message). A Stop hook's stdout otherwise lands in the debug log,
invisible to the human. `systemMessage` is the channel that actually reaches
the human's terminal — verified empirically with a live smoke test, rendering
as `Stop says: <msg>`. See [[hook-exit-codes]] for the fuller channel-mechanics
treatment. See [[pr_204_cost-reporter]] for the full source record.

**Emission mechanism changed by PR #224** (see the `ALS_PENDING_SYSMSG`
section above): this function no longer emits its own `{systemMessage: ...}`
JSON directly — it appends its message text to the shared
`ALS_PENDING_SYSMSG` accumulator via `als_append_pending_sysmsg`, and
`loop_stall_guard.sh`'s call site emits the single merged JSON after both
this function and the proof gate's withdrawal-notice path have run. The
behaviour-matrix table below is unchanged; only how the resulting string
reaches the human's terminal changed.

## `als_extract_last_text` malformed-line fix (PR #208) — the extraction path, not the count path

The declaration-detection gate (step 5 above) depends on `als_extract_last_text`
to read the orchestrator's actual last message text and check it against the
`LOOP-STOP` regex. Before PR #208, a single non-object JSON line anywhere in
the tail window (`jq -R 'fromjson? // empty'` admits any valid JSON value, not
only objects) crashed the downstream `select(.type == "assistant")` stage
(`Cannot index number with string "type"`), and the function's blanket
`2>/dev/null` silently collapsed the **whole** extraction to empty — even with
a genuine `LOOP-STOP: complete` line sitting right next to the bad one.
**Verified consequence:** this hook then reported "no LOOP-STOP declaration in
your last message" and blocked a stop that should have been allowed. Fixed
with one line, `select(type == "object")`, inserted right after the
`fromjson?` stage. Fails CLOSED (spurious block), not open — a forced
security-review pass (mandatory, diff touches `hooks/`) returned APPROVE, and
the complete-only gates above read their own JSON artifacts independently, so
they were never at risk from this bug either way.

**Not the same fix as `als_count_invocations`'s `select(type == "object")`
guard** (line 169, added by [[pr_194_198_loop-complete-deferral-and-proof-gates|PR #198]],
self-labelled "SECURITY FIX" in its own source comment) even though both are
one-line guards of the identical shape in the identical file. `als_count_invocations`
decides whether a session is a loop at all — a poisoned line there collapsed
the count to 0, causing `als_gate_require_active_loop` to treat the session as
"not a loop" and skip the retro/work-units/proof gates entirely (fail-OPEN, a
full gate-family bypass). `als_extract_last_text` only decides whether *this
turn's* declaration parses — a poisoned line there caused a false block
(fail-CLOSED). Same defect class, different function, different failure
direction, five days apart. See [[pr_206_208_loop-state-common-docs-and-robustness]]
for the full comparison.

`discipline_common.sh`'s sibling `dc_extract_last_text` carries the same
two-stage shape (documented in `als_extract_last_text`'s own source comment)
and almost certainly the same defect — explicitly **not** fixed by PR #208,
parked for the jq-slurp round-2 arc.

## Stdin read convention (PR #76)

This hook reads its payload via `IFS= read -r -d '' -t 5 input || true`. See [[pr_76_harden-hook-stdin-read]] for the full convention and the fail-open rationale.

## See also

- [[loop_state_guard]] — C1: presence/ownership guard; shares loop-active detection and the active-window off-switch
- [[unregistered_loop_guard]] — new sibling Stop hook (PR #17): covers loops that never reached C1/C2 at all because they never registered
- [[hook-exit-codes]] — this hook blocks via plain `exit 2` on `Stop`, the mechanism this page's table documents
- [[agentic-loop]] — the skill that must emit `LOOP-STOP` declarations; Phase 0.5 is the stop-ceremony note
- [[spec-plan-progress-artifact-chain]] — the two-hook loop-state guard architecture; why C1 speaks before C2
- [[discipline-loop]] — full hook composition (5 Stop hooks + 2 SubagentStop hooks, 14 hooks total across all events, as of PR #17)
- [[enforcement-model]] — hooks vs. commands
- [[pr_96-98_mode-aware-install-argument-injection-guard-hook-owned-counter]] — PR #98: this hook becomes the sole writer of loop_stop_counts
- [[pr_86-107_2026-07-08_loop-lib-residuals]] — PRs #91/#107: malformed-transcript tolerance in the shared `loop_state_common.sh` parse this hook's `als_gate_not_a_loop` depends on
- [[pr_184_185_186_loop-cost-tracking]] — PR #186: widens the retro gate to `schema_version >= 1`; PR #184: the cost-mining sub-step that bumps the retro to `schema_version` 2
- [[pr_194_198_loop-complete-deferral-and-proof-gates]] — PR #194: the deferral gate (`work_units` terminal-status check); PR #198: the proof gate (transcript-derived verdicts against a blind-authored `proof.json`) and the `als_count_invocations` non-object-line fix that closed a full complete-gate-family bypass
- [[loop-progress-fields]] — `work_units`' full schema; now has two consumers (the eval-threshold gate here documented and the deferral gate above)
- [[pr_204_cost-reporter]] — PR #204: `als_report_cost_on_complete`, the non-blocking reporter that mechanically prints the loop's cost via `systemMessage` — the fourth complete-only step, after the three gates above
- [[pr_206_208_loop-state-common-docs-and-robustness]] — PR #206: corrected REFERENCE.md's `loop_state_common.sh` function-inventory row (was 9/20 functions, missing `als_gate_proofs_on_complete` entirely); PR #208: the `als_extract_last_text` malformed-line fix documented above, and its comparison against the earlier `als_count_invocations` fix (PR #198)
- [[pr_224_231_233_235_loop-tooling-hardening]] — PR #224: `withdrawn_proofs` (proof withdrawal) documented above, and the `ALS_PENDING_SYSMSG` shared accumulator that changed how this hook and the cost reporter emit `systemMessage`
