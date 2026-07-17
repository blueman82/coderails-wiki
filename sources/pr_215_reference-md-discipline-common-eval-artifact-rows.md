---
title: "PR 215 — Fix stale function-inventory rows in docs/REFERENCE.md"
type: source
created: 2026-07-17
last_updated: 2026-07-17
sources: []
tags: [docs, reference, discipline_common, eval-artifact, drift, function-inventory]
---

# PR #215 — Fix stale function-inventory rows in docs/REFERENCE.md

<!-- Ingested by /wiki-ingest after merge. This is an immutable record of what changed. -->

## PR metadata

| Field | Value |
|---|---|
| PR number | #215 |
| Merged | 2026-07-17T12:19:16Z |
| Merge SHA | `4e21216` (full: `4e212166b0a98e83c27acc5f90d8a5f8575cd946`) |
| Head SHA | `3843b16` |
| JIRA ticket | — |

## Summary

Two stale function-inventory rows in `docs/REFERENCE.md` were brought current,
grouped by purpose rather than flat name lists — **the same gap class**
[[pr_206_208_loop-state-common-docs-and-robustness|PR #206]] fixed the same day
for the `loop_state_common.sh` row. This is now the *third* row of the same
table caught carrying the same rot (`loop_state_common.sh` via #206; two more
rows via this PR) — the recurring pattern matters more than any single name:

- **`hooks/scripts/lib/discipline_common.sh`** — listed 3 of the file's 4
  functions, omitting `dc_mine_hook_blocks` entirely despite the doc naming it
  twice elsewhere in prose (`loop_cost.sh`'s row cross-references its fail-open
  idiom; `index.md` already documents its PR #118 origin — see
  [[pr_118-123_self-improving-loops]]). Added under a new **Log mining** group
  alongside the existing **Extraction** group (`dc_extract_last_text`,
  `dc_stable_text`, `dc_file_count`).
- **`scripts/lib/eval-artifact.sh`** — listed 3 of the file's 7 functions.
  Regrouped by purpose: **Construction** (`marker`, plus the private
  `_prefix` helper named as a leading-underscore private, not a peer of the
  public API), **Parsing** (`matches_marker`, `parse_result`, `parse_tier`),
  **Result derivation** (`compute_go`), **Provenance** (`grading_checksum`).

## A real defect caught in review

The first version of the `eval-artifact.sh` row claimed `matches_marker`
"delegates its shared prefix to the private `_prefix` helper" — worded to
imply `marker()` shares that delegation too. **False**: `eval_artifact::marker()`
(lines 12-21) builds its string with a standalone `printf` and never calls
`_prefix`. `_prefix`'s *only* caller anywhere in the file is `matches_marker()`
at line 40. Verified directly against source
(`grep -n "_prefix" scripts/lib/eval-artifact.sh` — the only hits are the
function definition, its header comment, and the line-40 call site).

This is the **same trap [[pr_206_208_loop-state-common-docs-and-robustness|PR #206]]
hit** — its row also contradicted another paragraph of the same file. A PR
whose entire purpose was doc accuracy asserted behaviour the code does not
have, and review is what caught it both times. Worth stating plainly: this gap
class keeps producing confidently-wrong prose about *how* functions relate to
each other, not just *whether* they're listed — a fabricated relationship
slips past casual inspection more easily than a missing name does. Fixed at
`3843b16`.

## Process note — freeze-before-build closed

The prior loop ([[pr_206_208_loop-state-common-docs-and-robustness|#206/#208]])
had to disclose a freeze-before-build violation on its own evals. This PR's
evals were frozen **before** the fix, smoke-run **and** negative-controlled at
freeze time, and one eval (E3) specifically encodes the review finding above —
it discriminates across three states: `origin/main` (pre-fix, exit 1),
the pre-fix branch head `0b3c12b` (exit 1), and the fixed head (exit 0). That
three-way discrimination is what makes E3 evidence the fix actually changed
behaviour, not just that a check was added and immediately made to pass.

## Files changed

- `docs/REFERENCE.md` (+2/-2 — two table rows only; docs-only change, no code
  touched)

## Wiki pages updated

- [[pr_206_208_loop-state-common-docs-and-robustness]] — linked as the sibling
  instance of the same gap class, not restated.

## Caveats / gotchas

- **`discipline_common.sh`'s `dc_extract_last_text` is deliberately untouched
  by this PR** and remains covered by the `⚠️ CONTRADICTION` note on
  [[discipline-loop]] (added by the #206/#208 ingest): that note's claim that
  the function was "hardened in earlier PRs" is still false as of this PR's
  merge (`origin/main`'s current `dc_extract_last_text` body carries **0**
  hits for a `select(type == "object")` guard, re-verified directly). It is
  owned by the parked jq-slurp round-2 arc (memory:
  `project_jq_slurp_round2_handoff`) as that arc's main item — out of scope
  here by design, not by oversight. The contradiction note itself was **not**
  edited, softened, or removed by this ingest.
- No code paths changed — this is a documentation-accuracy fix only. The
  `_prefix`/`matches_marker` relationship it now correctly describes was
  already true of the shipped code; only the prose was wrong.
