---
title: "PR 206, 208 — loop_state_common.sh docs accuracy + extraction robustness"
type: source
created: 2026-07-17
last_updated: 2026-07-17
sources: []
tags: [hook, loop_state_common, docs, reference, jq, robustness, malformed-transcript, als_extract_last_text, als_count_invocations, loop_stall_guard]
---

# PR #206, #208 — loop_state_common.sh: fixing the docs, then hardening the code

One theme, two PRs, same day: `hooks/scripts/lib/loop_state_common.sh` first got
documented correctly, then got a real correctness fix for a gap the corrected
documentation helped surface.

## PR metadata

| Field | Value |
|---|---|
| PR #206 | "docs: fix stale loop_state_common.sh function-inventory row in REFERENCE.md" |
| Merge SHA (206) | `f5725ece5535a5f29a4922d194f2c792647c9a4b` |
| Merged (206) | 2026-07-17T10:25:16Z |
| PR #208 | "Guard als_extract_last_text against non-object JSON lines" |
| Merge SHA (208) | `c98f7fa7ccc76ad9ec6fe652c0f4c939bcf6a7ed` |
| Merged (208) | 2026-07-17T10:29:43Z |

## Summary

### PR #206 — REFERENCE.md function-inventory row was stale

`docs/REFERENCE.md`'s function-inventory row for `loop_state_common.sh` listed
9 functions when the file defines 20, and omitted `als_gate_proofs_on_complete`
entirely — 0 hits anywhere in the doc, despite it being the proof-gate
`loop_stall_guard`'s Phase 13 teardown depends on (see [[loop_stall_guard]]'s
"Proof gate on `complete` (PR #198)" section). The row was rewritten to cover
all 20 functions, grouped by purpose (detection primitives, state readers,
per-hook entry gates, complete-declaration gates, reporting) — matching the
verified current source, not a stale earlier version. Same drift class as
[[pr_92_2026-07-08_reference-drift-and-lookalike-fp]], different file/row.

Two accuracy findings caught in review, before merge:
1. The proof-gate description omitted the **last-failed** blocking case —
   contradicting REFERENCE.md's *own* line 385, which already had it right.
   The new row now states the gate blocks on "any proof entry that is
   unexecuted **or last-failed**", consistent with [[loop_stall_guard]]'s own
   documented behaviour.
2. `als_report_cost_on_complete` was labelled as printing "token cost" when it
   actually leads with a **USD** figure (`Loop cost: $<usd> (<tokens>
   tokens)...` — see [[loop_stall_guard]]'s cost-reporter behaviour matrix).
   Corrected to "mined cost — USD, tokens, price-staleness age".

Verified against origin/main (`git show origin/main:docs/REFERENCE.md`, line
433): the row now reads in full, listing all 20 functions and both corrected
descriptions.

### PR #208 — als_extract_last_text silently blinded by a non-object transcript line

Independent robustness fix, same file, filed right after the docs correction
made the function inventory visible enough to prompt closer scrutiny of each
entry.

`als_extract_last_text`'s stage-1 parse, `jq -R 'fromjson? // empty'`, emits
**any** valid JSON value per line — not only objects. A bare scalar or array
transcript line (e.g. `123`, `[1,2,3]`) survives stage 1 untouched and reaches
stage 2's `select(.type == "assistant")`, which **errors** on a non-object
(`Cannot index number with string "type"`). Because the whole pipeline is
wrapped in `2>/dev/null`, that crash silently collapses the **entire**
extraction to empty — even when a genuine assistant-text line sits right next
to the bad one in the same tail window.

**Fix:** one line, `| select(type == "object")` inserted immediately after the
`fromjson?` stage — filters out non-object values before the `.type` access
that was crashing on them. Plus 4 new tests in
`hooks/scripts/tests/loop_state_guard.test.sh`, poisoning the tail window both
before and after the genuine assistant-text line.

**User-facing consequence, verified at hook level.** Pre-fix,
[[loop_stall_guard]] (the hook that calls `als_extract_last_text` to read the
orchestrator's stopping-turn text) SPURIOUSLY BLOCKS a genuine `LOOP-STOP:
complete` declaration — "no LOOP-STOP declaration in your last message" — when
any poisoned line sits in the tail window, even though the declaration is
right there in the actual last message. Post-fix it parses correctly and
advances to the legitimate next gate (retro/work-units/proof).

**Failed CLOSED, not a security hole.** The blinding makes the hook see "no
declaration" and block — never "declaration satisfied" when it wasn't. This is
a robustness/correctness defect (a legitimate stop gets wrongly blocked), not
a fail-open bypass. A forced security-review pass (mandatory because the diff
touches `hooks/`) returned APPROVE: no fail-open path exists, and the
complete-only gates (retro/work-units/proofs) read `retro.json`/`progress.json`/
`proof.json` via independent `jq` calls, structurally insulated from whatever
`als_extract_last_text` returns.

**Nuance: `null` was actually harmless pre-fix.** jq's `null.type` returns
`null` without erroring — only booleans/strings/numbers/arrays hit the crash.
The real failure surface is narrower than "any non-object value," though the
fix (`select(type == "object")`) covers all of them uniformly rather than
special-casing `null`.

## Relationship to the sibling `als_count_invocations` fix (PR #198) — NOT a duplicate

`loop_state_common.sh` already had an almost-identical-sounding fix, from
[[pr_194_198_loop-complete-deferral-and-proof-gates|PR #198]]: a
`select(type == "object")` guard added to `als_count_invocations` (source
comment at `hooks/scripts/lib/loop_state_common.sh:169`, "SECURITY FIX,
reproduced full bypass"). **These are two separate functions, two separate
defects, fixed five days apart:**

| | `als_count_invocations` (PR #198) | `als_extract_last_text` (PR #208) |
|---|---|---|
| Purpose | Counts agentic-loop Skill invocations in the transcript — decides "is this a loop at all" | Extracts the last assistant text block — read by `loop_stall_guard` to find the `LOOP-STOP` line |
| Failure mode on a poisoned line | Count collapses to 0 → hook treats session as "not a loop" → **every** complete-gate (retro/work-units/proof) is skipped entirely | Extraction returns empty → hook sees "no declaration" → **blocks** a genuine `complete` stop |
| Direction of failure | Fail-OPEN (a full gate-family bypass — the Critical PR #198 fixed) | Fail-CLOSED (a spurious block — the correctness defect PR #208 fixed) |
| Severity class | Security (self-reported "SECURITY FIX" in source comment) | Robustness/correctness (forced security pass returned APPROVE) |

Same defect *class* (a two-stage jq pipeline where stage 1 admits any JSON
value but stage 2 assumes an object), same one-line fix shape, different
function, different blast radius, different day. Do not conflate — PR #198's
fix did not cover `als_extract_last_text`; it fixed a different call site.

## Deliberately out of scope: `discipline_common.sh`'s sibling function

`discipline_common.sh` (a **separate, deliberately independent** hook-family
lib — `loop_state_common.sh`'s own source comment states the two libs are not
shared "since loop_state_common.sh and discipline_common.sh are deliberately
independent") carries `dc_extract_last_text`, described in its own header
comment as having "the identical two-stage parse" as `als_extract_last_text`.
It almost certainly carries the same defect class. **PR #208 explicitly does
not touch it** — its own PR body states this is "a separate, already-tracked,
parked concern." This is parked for the jq-slurp round-2 arc (see the memory
handoff `project_jq_slurp_round2_handoff`, not yet a wiki source page as of
this ingest). Record the relationship here; do **not** present
`dc_extract_last_text` as fixed by this cluster.

## Files changed

- `docs/REFERENCE.md` (PR #206) — function-inventory row for `loop_state_common.sh`, line 433
- `hooks/scripts/lib/loop_state_common.sh` (PR #208) — one line added at `als_extract_last_text`, `select(type == "object")`
- `hooks/scripts/tests/loop_state_guard.test.sh` (PR #208) — 4 new tests (+57 lines)

## Wiki pages updated

- [[loop_stall_guard]] — documented the `als_extract_last_text` fix, its user-facing consequence (spurious block on poisoned tail window), and the relationship to the `als_count_invocations` fix already on that page
- [[discipline-loop]] — while cross-referencing the sibling `dc_extract_last_text` gap, found and flagged a **pre-existing wiki contradiction**: that page claimed `dc_extract_last_text` was "hardened in earlier PRs," but `origin/main`'s current `discipline_common.sh` shows no such guard exists — added a dated `⚠️ CONTRADICTION` note there rather than silently repeating the false claim

## Caveats / gotchas

- Do not conflate this fix with PR #198's `als_count_invocations` fix — see
  the comparison table above. Both live in the same file, both are
  `select(type == "object")` guards, both were reviewed as part of the same
  "malformed transcript line" defect family, but they are different functions
  fixed on different days for different failure directions.
- `dc_extract_last_text` in `discipline_common.sh` is NOT fixed by this
  cluster — explicitly parked, see above.
- PR #206's docs fix and PR #208's code fix are causally related (the
  docs-accuracy pass surfaced the function closely enough to prompt the
  robustness check) but independently mergeable and independently verifiable;
  ingested together here because they share one theme (this file, this day),
  not because one depends on the other functionally.
