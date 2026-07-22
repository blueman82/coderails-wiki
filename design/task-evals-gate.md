---
title: "Task-evals gate — dual-scope (pr + loop) enforcement over frozen, oracle-independent success evals"
type: design
created: 2026-07-06
last_updated: 2026-07-22
sources:
  - sources/pr_261_freeze_before_build_gate.md
  - sources/pr_1-4_task-evals-feature.md
  - sources/pr_7-10_task-evals-followups.md
  - sources/pr_11-14_gate-hardening-followups.md
  - sources/pr_144-149_agentic-loop-hardening-from-loop-engineering.md
  - sources/pr_155-158_ceremony_noise_envelope_anchoring.md
  - sources/pr_138_remove-specs-plans-tracking.md
  - sources/pr_218_discriminating-check-gate.md
  - sources/pr_232_tier-review-gate.md
tags: [design, task-evals, evals-json, merge-gate, loop-gate, truth-seam, sha-bound, work-units, oracle-independence, comment-spoofing, tier-justification, trust-floor, viewerPermission, grade-loop, unstamped, discriminating-check, fixtures, tier-review, root-daemon]
---

# Task-evals gate

> The design spec formerly cited in this page's `sources:` (`docs/coderails/specs/2026-07-03-task-evals-design.md`) no longer exists — removed from repo tracking by [[pr_138_remove-specs-plans-tracking]] (2026-07-11). This page's own content is now the durable record.

Introduced by [[pr_1-4_task-evals-feature]] (PRs #1–4, merged 2026-07-06). Adds a second, independent enforcement axis alongside [[review-artifact-seam]]: instead of gating on *review having happened*, this gates on *frozen, oracle-independent success evals having passed* — at two distinct scopes, pr and loop, sharing one schema.

## The core problem

Every existing coderails verification loop is self-verification: the same agent (or process) that implements a change also writes the criteria it will be judged against, runs them, and grades itself. `writing-plans` verify-criteria are authored by the process that then implements against them. Agentic-loop workers verify their own artifact before reporting. Phase 4b reviews code quality, not goal attainment. Phase 13's self-audit is explicitly unscored. The review-artifact gate ([[review-artifact-seam]]) proves review *happened* on the correct SHA — not that the task's actual goal state was reached.

The one place this pattern was already broken was the hand-written public-readiness suite (E0–E10): negative controls, end-state assertions against fresh surfaces, independent GO/NO-GO gating, evals defined independently of task self-verification. `task-evals` generalises that one-off pattern into a repeatable skill with a frozen schema and two hook-enforced consumers.

## The decision

Generate and **freeze** a tiered `evals.json` before implementation starts (freeze-before-build), then gate on it at two scopes:

- **pr scope** — a SHA-bound PR comment, gate lives in `scripts/merge.sh`, mirrors [[review-artifact-seam]] almost exactly.
- **loop scope** — a sibling file next to `progress.json`, gate lives in the existing `loop_state_guard.sh` Stop hook, extended rather than given a new script.

Both consume the identical `schema_version: 1` shape (`scope: "pr" | "loop"` is the only structural fork point), so the two gates can never drift on what a valid eval object looks like even though they were built as separate work-units (WU2 pr-scope, WU3 loop-scope) with independently-appropriate fail-open/fail-closed postures.

## Schema (schema_version 1)

```json
{
  "schema_version": 1, "scope": "pr | loop", "task_ref": "<PR#/branch or loop ordinal>",
  "tier": 0, "tier_justification": "<required at every tier: tier 0 = why the exemption is legitimate; tier 1/2 = which tier predicate fired>",
  "frozen_at": "<ISO8601>", "frozen_sha": "<base SHA at freeze>",
  "evals": [ { "id": "E1", "priority": "P0", "mode": "scripted",
    "surface": "merged-state | fresh-clone | artifact-path | deployed",
    "assert": "...", "cmd": "...", "negative_control": "...",
    "status": "pending | pass | fail", "evidence": "..." } ],
  "amendments": [ { "eval": "E1", "when": "<ISO8601>", "why": "<reason>", "regraded_by": "<fresh grader run — required only for post-verdict amendments>" } ],
  "result": null, "graded_at": null, "head_sha": "<SHA graded against>",
  "grading": { "by": "post_evals.sh grade-loop", "checksum": "<sha256 over statuses+result>", "amendments_at_grade": 0 }
}
```

`eval_artifact::compute_go` (`scripts/lib/eval-artifact.sh`) is the ONE place `result` is derived: a pure `jq` predicate requiring every `.priority == "P0"` eval to have `.status == "pass"`. An eval with no `priority` field is simply excluded from the P0 gate by design (not a bug — `post_evals::validate_structure` check 7 is the layer that refuses a tier≥1 artifact with zero real P0 evals, keeping `compute_go` itself an unopinionated pure gate).

**Check 8 — freeze-before-build, added by [[pr_261_freeze_before_build_gate|PR #261]] (2026-07-22).** Until then, rule 1 (freeze before implementation) was the one anti-gaming rule with no mechanism: `frozen_sha` was written into every artifact and read by nothing — `grep -rn frozen_sha hooks/ scripts/` returned zero hits. Check 8 (`post_evals::validate_freeze`) requires `frozen_sha` to be an ancestor of the branch's merge-base with the default branch. It **refuses** a `frozen_sha` pointing at one of the branch's own commits (froze after building) and refuses one git can't resolve; it **skips** when the field is absent (back-compat — every prior artifact omits it) or the file sits outside a work tree. Pr scope only, matching where the merge gate reads. A *disclosed* late freeze passes via explicit prose in `tier_justification` or an amendment `why` — deliberately not a boolean, since a flag can be set silently while a sentence is visible to a reader; the substring match (`freeze`/`frozen`) is an acknowledged coarse heuristic. Same PR fixed a **fail-open**: with `jq` absent the `frozen_sha` read returns empty, which is indistinguishable from "field absent" and took the skip path, so the gate passed while verifying nothing — now guarded explicitly on `command -v jq`. Generalises to any check whose skip path keys on an empty read.

`grading` is optional and additive — added by [[pr_144-149_agentic-loop-hardening-from-loop-engineering|PR #144]], 2026-07-12. Only `post_evals.sh grade-loop` (loop scope) writes it; pr-scope files and every pre-existing reader tolerate its absence. See "Loop-scope gate: the `grade-loop` stamp" below. It is write-time provenance, absent at freeze — PR #153 (2026-07-13) added `amendments_at_grade` to the stamp and moved the SKILL.md schema block to show the frozen shape only, after review caught that a file frozen literally from a grading-bearing template would false-refuse its first grade under the regrade-on-amendment backstop (below).

## The six anti-gaming rules (generation-time discipline)

Full detail on [[task-evals]]. Named here because they're what the two enforcement gates below are trusting was followed at generation time — the gates themselves cannot re-verify oracle independence, grader independence, or strongest-surface coverage. **Rule 1 is the exception as of [[pr_261_freeze_before_build_gate|PR #261]]**: it is now mechanically checked at pr scope (check 8 above), so the "structural shape and P0 pass/fail only" framing below no longer covers the whole set. Rules 2-6 remain generation-time discipline:

1. Freeze-before-build — **no longer discipline-only.** Check 8 (pr scope) verifies `frozen_sha` is an ancestor of the branch base, refusing a freeze that happened after implementation began unless the lateness is disclosed in prose. Loop scope still trusts the rule.
2. Negative controls
3. End-state surfaces
4. Oracle independence — extended by [[pr_155-158_ceremony_noise_envelope_anchoring|PR #158]] (2026-07-13) with a loop-scope **precedence rule**: the eval author's goal-state anchor is `progress.json`'s `authorising_prompt_raw` — the post-Phase-0 envelope, exactly one canonical string, no judgement call about which version of the prompt counts. `spec.md` restates the loop's success criteria at Phase 2.7a and `plan.md` restates it per-task, but this is precedence, not content denial: `spec.md`/`plan.md` supply constraints and concrete assertable surfaces useful for writing evals, and their restated criteria never override the envelope as the anchor. `progress.json`'s field is canonical; `spec.md`'s Phase-2.7a copy is a derived restatement. Wired into [[agentic-loop]] at four points: the Phase -2 stub schema comment, the mid-loop re-stub carry-forward rule (now covers `authorising_prompt_raw` alongside `loop_stop_counts`), an explicit Phase 2.7c cross-reference, and Phase 13's `retro.json` `envelope` field (now sourced from the named field, not "verbatim from `progress.json`" generically).
5. Grader independence — extended by PR #153 (2026-07-13): an eval amended after a grader verdict returns to a fresh grader; the orchestrator never writes a per-eval `status` that flips an existing verdict. Structurally backstopped by `grade-loop`'s regrade-on-amendment refusal (below), the one rule 2-5 exception to "generation-time discipline only".
6. **Strongest surface** ([[pr_144-149_agentic-loop-hardening-from-loop-engineering|PR #145]], 2026-07-12) — if the goal state names something a human sees or interacts with (a UI, CLI output, a rendered artifact, a served endpoint), at least one P0 eval must exercise that surface directly rather than only grepping merged state. A writer-side generation rule only — no gate can detect "this goal state is user-facing," so unlike rules 2-5 there is no corresponding structural check in either enforcement gate below. PR #154 (2026-07-13) named the pr-scope surface explicitly — `artifact-path`, defined as a locally built or locally run artifact (a file path, a local CLI invocation, or a locally served endpoint; the same endpoint on the live post-merge instance is `deployed`) — and reconciled the tier-0 exemption: anything rule 6 names **is** an outward surface for the tier-0 predicate, so a user-facing change is minimum tier 1 (tier 2's outward predicate stays scoped to its own parenthetical list). The same PR aligned rule 3 to admit a locally built artifact driven directly, distinguishing driving from working-tree self-reports.

## PR-scope gate

### Marker format

```
<!-- coderails-eval-summary v1 pr=<N> head_sha=<sha> result=<GO|NO-GO> tier=<0|1|2> -->
```

### Matching: literal prefix, not interpolated regex

`eval_artifact::matches_marker` matches via **literal prefix string-equality** through `head_sha=<sha> result=`, mirroring `review_artifact::matches_marker`. This was a review-round fix, not the original design: an earlier version built an anchored regex by interpolating the untrusted `pr`/`head_sha` values directly (`^<!-- ... pr=$pr head_sha=$head_sha result=(GO|NO-GO) tier=[0-2] -->$`), which meant a PR whose number or SHA carried regex metacharacters (e.g. `pr="1|.*"`) could widen the alternation and cause a false match. The fix: build a **literal** prefix via `printf`, compare with a bash `case` glob (`"$prefix"*`), and only defer to `parse_result`/`parse_tier`'s anchored regex — which is safe because it never interpolates `pr`/`head_sha` — to validate the remaining `result=/tier=` grammar. (verified: `scripts/lib/eval-artifact.sh`, review-fix commit `8291764`)

### Newest-artifact-wins

`pr::has_coderails_eval_for_head` scans **all** matching marker lines for the PR/SHA and uses the **last** (newest) match's result/tier — not the first. This closed a critical review finding: without it, a stale `GO` comment could permanently outrank a later `NO-GO` re-grade posted against a fix that regressed, for the identical head SHA. Symmetric to `review_artifact`'s own reader, which faced the same question and made the same choice.

### Merge gate placement

`scripts/merge.sh` gains the eval-gate check directly after the existing review-artifact gate, in the same `OPEN` branch, before `gh pr merge`. Same rc-contract shape as the review gate: `0` = match found with `GO`, `1` = no match or `NO-GO`, `2` = `gh` fetch failed (distinct error message: "GitHub fetch failed" vs "no coderails eval artifact"/"NO-GO — resolve failing P0 evals"). Fail-closed, no local-file fallback, no config opt-out — identical posture to [[review-artifact-seam]].

**Second, hook-level consumer added 2026-07-08 ([[pr_96-98_evals-gate-uniform-enforcement_2026-07-08|PR #97]]):** `enforce_pr_workflow.sh`'s `PreToolUse` hook gains `gate_eval_artifact_for_merge`, applying the identical `pr::has_coderails_eval_for_head` check (same rc semantics: 0/GO allow, non-zero NO-GO/absent deny, rc=2 fetch-failure fail-closed) directly to a raw `gh pr merge <N>` Bash call, not just inside `merge.sh`'s script body. This closes the gap [[evals-gate-enforcement-gap_2026-07-08]] identified: previously nothing forced a merge to actually route through `merge.sh`, so this gate is config-dependent (inactive under `NO_CONFIG`, same as the rest of `enforce_pr_workflow`) where `merge.sh`'s own gate is config-independent — full mechanism and gate-ordering detail on [[enforce_pr_workflow]], not duplicated here.

### Structural refusals (`post_evals::validate_structure`, 7 checks)

1. File exists + valid JSON.
2. Every tier requires a non-blank `tier_justification` (owner directive, [[pr_7-10_task-evals-followups|PR #10]]: tightened from tier-0-only to all tiers — see "tier_justification required at every tier" below).
3. Tier ≥1 scripted eval requires non-empty `negative_control`.
4. `negative_control` must not be vacuous relative to `cmd` — whitespace-normalised equality **plus** a word-bounded containment check (catches `"true; cmd"`, `"echo x && cmd"` wrapper tricks) without false-flagging a genuinely distinct control sharing a text prefix. This is a **structural** floor, not a semantic one — a genuinely-distinct-but-still-vacuous control (passes for unrelated reasons) still clears this check; semantic quality is the verifier/human review layer's job.
5. Any P0 eval requires non-empty `evidence`.
6. `head_sha` in the file must match the PR's current live head SHA.
7. Tier ≥1 requires at least one actual P0 eval in `.evals` — added in the review-fix round, closing a **vacuous-GO gap**: without this check, a tier-1+ artifact with an empty or P1-only `.evals` array would pass `compute_go`'s P0-only predicate vacuously (no P0 evals to fail = trivially "GO"). Tier 0 is exempt by design.

## Discriminating-check gate ([[pr_218_discriminating-check-gate|PR #218]], 2026-07-17)

A second, distinct pr-scope freeze-time gate, additive to the seven structural refusals above — added at `/coderails:post-evals` Step 3b, between structural validation and result computation. Catches a defect class rule 2 (negative controls) does not: a scripted check whose `cmd` and `negative_control` are both present and textually distinct, but whose formula is **itself broken** — incapable of ever passing (false alarm) or ever failing (vacuous). Motivating real instance: loop 8b69e779's awk formula split `"39/39 suites passed"` under `-F'[ /]'` and landed `$(NF-2)` on the literal word `"suites"`, exiting 1 unconditionally — a genuine pass and a genuine fail produced identical exit codes.

### Mechanism

An eval may carry an optional `fixtures` object: `{ good, bad, formula? }` — `good` is sample input that should make the check pass, `bad` is sample input that should make it fail, `formula` is the verdict-stage command (defaults to the text after the last top-level pipe in `cmd` when omitted; text-position split, not shell-aware, so a quoted pipe forces an explicit `fixtures.formula`). `post_evals::validate_discriminating` (`scripts/post_evals.sh`) pipes `good` and `bad` into the formula via `post_evals::_run_formula` (a `perl alarm`-wrapped 10s-timeout `bash -c`) and requires opposite exit codes: `good_rc == 0 && bad_rc != 0`. Both fixtures producing the same exit code (both 0, both non-zero) rejects as non-discriminating, by eval id, with a distinct message per direction.

**Env-guard**: exit 127 (command not found), exit 142 (the function's own timeout sentinel, 128+SIGALRM), exit 126 (permission denied), and any exit ≥128 (signal deaths — 137=SIGKILL, 139=SIGSEGV, ...) are all reported as distinct environmental-failure messages, never conflated with a discrimination verdict — a formula that crashes on the bad-fixture leg would otherwise fall through into the accept path (`good_rc=0 && bad_rc≠0`) and read as a legitimate "bad correctly fails" result. The 142 check runs before the ≥128 check so the timeout message stays distinct (142 also satisfies ≥128; ordering here is load-bearing per the source comment). The ≥128 broadening (126 and general signal deaths) was a same-day follow-up fix (`335a382`), not the original PR.

**Fail-closed on required-pair and malformed shape**: `fixtures` present but not a JSON object rejects with a distinct "must be an object" message (a bare string/number would otherwise silently degrade every field extraction to `""` and misreport as non-discriminating). `good` and `bad` are required **together** — an author supplying `good`+`formula` but omitting `bad` gets `bad=""` by jq's default, and proving discrimination against an empty string nobody wrote is the unsafe accept direction; both omission directions reject explicitly rather than silently accepting. This pairing requirement was also a same-day follow-up fix, closing a gap the original PR's own design left open.

### Grandfathering — explicit, load-bearing scope limit

An eval with no `fixtures` field is validated **exactly as it was before this gate existed** — zero behaviour change, `validate_discriminating` returns 0 immediately if no scripted eval in the file carries `fixtures`. `fixtures` is opt-in per eval, never retroactive: freezing this gate did not retroactively validate any pre-existing `evals.json`, and an author who never adds `fixtures` gets no discrimination proof at all from this layer.

### Honest boundary — do not overclaim

This gate validates only checks that **carry** `fixtures` — it does not retroactively validate the ~46% of the corpus that already has scripted checks without fixtures (a figure re-derived after an earlier, badly broken classifier in the building loop had claimed 91% and used that number to conclude the whole feature was unbuildable — see "Design history" below), and it does nothing for prose/judgement-mode evals. Even where `fixtures` is present, a pass proves only that the formula **can discriminate between these two specific synthetic inputs** — it proves nothing about whether the formula tests the **right** claim, whether `cmd` and `fixtures.formula` stay in sync after later edits, or whether the fixtures are representative of real pass/fail states. This closes the "never fails" (or "never passes") class of defect; it is not a general correctness proof of the check, and it is a narrower claim than the seven pr-scope structural refusals above, which apply to every scripted eval unconditionally.

### Design fork: formula-against-fixtures vs. real-surface execution

Rejected alternative: run the check against the real target surface at freeze time. Rejected because at freeze a not-yet-built surface and a genuinely broken check both exit non-zero **indistinguishably** — a real-surface gate could not tell "this check is broken" apart from "this feature legitimately doesn't exist yet." Piping synthetic `fixtures.good`/`fixtures.bad` text into the formula sidesteps that ambiguity: the formula is exercised against known inputs with known expected outcomes, independent of whether the real implementation exists.

### Design history: a wrongly-concluded hard-stop, overturned

An earlier point in the building loop concluded the feature was "unbuildable," premised on a measured claim that ~91% of the corpus's `negative_control` values looked like prose rather than machine-checkable commands (near-zero surface for a fixtures gate to validate). That 91% came from a broken classifier; re-measured, the real figure is closer to 46%. A `/coderails:disconfirm` pass plus an adversarial fable-5 red-team review overturned the stop by challenging the classifier itself. Recorded as a general caution: a hard-stop premised on a measured corpus statistic warrants the same "is this measurement itself broken" scrutiny as any other claim before it's allowed to kill a feature.

Relationship to rule 2 (negative controls): rule 2 and the pre-existing structural check (`validate_structure` check 4, vacuous-relative-to-cmd) both operate on **text** — do `cmd` and `negative_control` differ as strings. This gate operates on **behaviour** — does the check's formula actually produce different verdicts on different real inputs. A check can satisfy rule 2 and check 4 (textually distinct control) while still failing this gate (formula behaviourally identical on pass and fail input), which is exactly the loop-8b69e779 shape.

## Loop-scope gate

Lives entirely inside the pre-existing `loop_state_guard.sh` Stop hook (C1) — extended, not replaced or forked into a new script.

### New shared-lib readers (`loop_state_common.sh`)

- `als_read_work_units()` — reads `.work_units | length` off `progress.json` into `ALS_WORK_UNIT_COUNT`. **Fail-open**: absent file, absent/null `.work_units` (a legacy loop that predates this field), or malformed JSON all resolve to `0`. Absence must never itself cause a block — the same presence-not-provenance posture the rest of `loop_state_guard` already takes.
- `als_read_loop_evals_result()` — reads a sibling `evals.json` beside `progress.json` into `ALS_LOOP_EVALS_RESULT` ∈ `GO | TIER0 | NO-GO | UNJUSTIFIED | UNSTAMPED | ABSENT`. `ABSENT` covers no file, malformed JSON, or a non-`"loop"` scope — a stray pr-scope file left in the same loop-state dir must never satisfy the loop gate. **Missing `jq`** gets its own distinct fail-open branch and audit-log line (`evals=skipped reason=jq_missing`) — a different, explicitly-logged reason from "file genuinely absent." **Explicit NO-GO wins at every tier, including tier 0** ([[pr_11-14_gate-hardening-followups|PR #11]], owner directive, 2026-07-06): the `elif` chain now checks `result == "NO-GO"` **before** the `tier == "0"` exemption branch. Before this fix, a tier-0 artifact that explicitly recorded `result: "NO-GO"` still read `TIER0` (allow) — the tier-0 branch fired on tier alone, without ever inspecting `result` for that case. "An exemption justifies having no evals, not overriding a recorded failure" (owner directive, stated inline in the source comment). A tier-0 artifact that never sets `result` at all — the legitimate exemption case — still correctly reads `TIER0`; only an artifact that explicitly records `NO-GO` now blocks like any other tier.

### Loop-scope gate: the `grade-loop` stamp and `UNSTAMPED` ([[pr_144-149_agentic-loop-hardening-from-loop-engineering|PR #144]], 2026-07-12)

Before this PR, nothing prevented the orchestrator itself from hand-writing `result: "GO"` into a loop-scope `evals.json` — the reader checked shape and the `result` field's value, not who computed it. `scripts/post_evals.sh` gained a **`grade-loop`** subcommand: it validates the loop-variant structure, computes `result` via the same `eval_artifact::compute_go` SSOT the pr-scope gate already uses, and atomically stamps `result`, `graded_at`, and `grading: {by, checksum}` (a sha256 over the per-eval statuses + the computed result).

`als_read_loop_evals_result()` now demotes an otherwise-valid `GO`/`TIER0` verdict to **`UNSTAMPED`** when `.grading.by`/`.grading.checksum` are absent, blank, or the checksum recomputed against the file's own `.result` doesn't match what's stored. `loop_state_guard.sh` blocks on `UNSTAMPED`, pointing at `grade-loop`. **Honest boundary, stated inline in the source**: the stamp catches accidental drift (a status hand-edited after grading without re-stamping), not deliberate tampering — a motivated forger could still fabricate a matching checksum. `NO-GO`/`UNJUSTIFIED` are untouched by this check (already block; an unstamped rejection is not a forgery risk since there's nothing to gain by faking a NO-GO).

A side effect worth stating precisely: `grade-loop` always writes `.result` — even a tier-0 exemption with an empty `.evals` array computes `GO` (vacuously true under `compute_go`'s P0-pass predicate with zero P0 evals). Because the reader's `elif` chain checks `.result` before `.tier`, a **stamped** tier-0 file now reads `GO`, not `TIER0` — `TIER0` as a terminal outcome only occurs for an **unstamped** tier-0 file (no `.result` at all).

### Regrade-on-amendment backstop (PR #153, 2026-07-13)

Closes the gap the L2/A1 incident (loop f87a0a2e) exposed: a fresh grader returned NO-GO, the orchestrator amended the eval (a genuine harness bug, recorded per rule 1) — then hand-set the flipping `status: pass` itself, and `grade-loop` computed GO from that orchestrator-written status. The stamp bound statuses+result but carried no amendment history, so it structurally could not catch the flip.

`grade-loop` now stamps `grading.amendments_at_grade` (the `.amendments` count at grade time) and, on a later invocation where the array has grown past the stamped count, **refuses** (exit 1, nothing written) unless every post-verdict amendment carries a non-blank *string* `regraded_by` naming the fresh grader run. Fail-closed throughout: malformed `.amendments` (non-array, or non-object entries) refuses rather than grades; non-string `regraded_by` counts as unattested; prior-verdict detection keys on grade residue (`.grading // .graded_at // .result`), so regenerating the file while any residue remains still refuses — the sanctioned path is amending the graded file in place. Test coverage includes two mutation-proven gaps closed at the merge gate: a mixed attested/unattested batch refuses (an all-unattested predicate mutation survived the original suite), and the slice boundary at the stamped index exempts pre-grade amendments (a whole-array-scan mutation survived).

Honest boundary, stated in the SKILL.md and the source: the gate keys on amendment **count growth after a grade-loop stamp**. A status flipped with no accompanying amendment, an existing amendment edited or replaced in place, a flip folded in before the first grade-loop run, and a hand-edited `amendments_at_grade` stamp (it sits outside the checksum canon) are all invisible to it — held by the rule-5 text and the Phase 13 audit alone. The attestation is verified to *exist*, not to be *true* — same refusal-or-recorded-lie boundary as the checksum itself. The reader side (`als_read_loop_evals_result`) is untouched; defence-in-depth: a refused flip leaves per-eval statuses mismatching the stale checksum, so the loop guard demotes GO/TIER0 to `UNSTAMPED` anyway.

### tier_justification required at every tier (owner directive, PR #10)

Originally only tier 0 required a non-empty `tier_justification`. [[pr_7-10_task-evals-followups|PR #10]] extended the requirement to **every tier**: tier 0 justifies the exemption itself; tier 1/2 must now state which tier predicate fired. `als_read_loop_evals_result()` gained a distinct result value, **`UNJUSTIFIED`**, checked before `result`/`tier` are even inspected — a blank (or whitespace-only, after trim) `tier_justification` short-circuits straight to `UNJUSTIFIED` regardless of grading outcome. This closes a bypass the source comment names explicitly: `eval_artifact::compute_go` never inspects `tier_justification` at all, so without this reader-side check a GO-graded loop artifact with no justification would have silently satisfied the gate. `UNJUSTIFIED` is kept distinct from `NO-GO` so the guard's block message names the actual defect. The guard also added a `*` catch-all for any unrecognised result value, logging `reason=unrecognised_evals_result` and failing closed. The writer side (`post_evals::validate_structure` check 2, pr scope) was extended the same way, with trim-then-check replacing mere non-empty-check so a whitespace-only justification doesn't pass either. **Behaviour flip**: pre-existing GO loop artifacts written before this check existed, lacking `tier_justification`, now block — a deliberate, explicitly-logged breaking change to already-graded artifacts, not new-artifact-forward only.

### Gate ordering: an intentional exact deviation

`gate_loop_evals_required()` (new, local to `loop_state_guard.sh`) must run **before** the shared `als_gate_loop_complete` in the guard's own gate chain:

```
als_gate_no_transcript → als_gate_stop_loop → als_gate_require_active_loop
  → als_load_progress → gate_loop_evals_required → als_gate_loop_complete
  → gate_present_and_owned → block_state_failure
```

`als_gate_loop_complete` **exits 0 directly** the instant `status == "complete"` AND not re-armed AND session-owned — it IS the off-switch for the rest of the chain, with no way to signal "checked, still active" back to a caller placed after it. So `gate_loop_evals_required` re-checks the identical three conditions locally, rather than restructuring the shared function to support an insertion point. This is a documented, reviewed exact deviation from how the C1/C2 guards otherwise compose gates as clean top-to-bottom skip chains — the source comment spells out why. (verified: `hooks/scripts/loop_state_guard.sh` lines 66–74)

### Block condition

When the three loop-complete conditions hold AND `ALS_WORK_UNIT_COUNT >= 3`: reads the loop evals result.
- `GO` or a justified `TIER0` (no `result` recorded) → allow, logged.
- `NO-GO` (including an explicit `NO-GO` at tier 0 — [[pr_11-14_gate-hardening-followups|PR #11]]) or `ABSENT` → **block** (exit 2), pointing at `/coderails:task-evals` (or a justified tier-0 exemption).

Below the 3-unit threshold: allow unconditionally, logged `evals=skipped-below-threshold`. `loop_stall_guard.sh` (C2) is untouched — this entire gate lives in C1.

## Fail-closed / fail-open matrix

| Layer | Condition | Outcome |
|---|---|---|
| pr-scope merge gate | `gh` fetch fails | Block: "GitHub fetch failed" |
| pr-scope merge gate | No matching marker for current SHA | Block: "no coderails eval artifact" |
| pr-scope merge gate | Marker found, `result=NO-GO` | Block: "NO-GO — resolve failing P0 evals" |
| pr-scope merge gate | Marker found, `result=GO` (newest match) | Allow |
| loop-scope Stop gate | `work_units` absent (legacy loop) | Allow (fail-open) |
| loop-scope Stop gate | `jq` missing | Allow (fail-open, distinct log reason) |
| loop-scope Stop gate | `work_units >= 3`, no/malformed/non-loop-scope `evals.json` | Block (`ABSENT`) |
| loop-scope Stop gate | `work_units >= 3`, `evals.json` result `NO-GO` (any tier, including tier 0 — PR #11) | Block |
| loop-scope Stop gate | `work_units >= 3`, `tier_justification` blank/whitespace-only (any tier) | Block (`UNJUSTIFIED`, PR #10) |
| loop-scope Stop gate | `work_units >= 3`, `evals.json` result unrecognised value | Block (`reason=unrecognised_evals_result`, PR #10, fail-closed catch-all) |
| loop-scope Stop gate | `work_units >= 3`, `evals.json` result `GO`/justified `TIER0`, valid `grade-loop` stamp | Allow |
| loop-scope Stop gate | `work_units >= 3`, `evals.json` result `GO`/`TIER0` but `grading` absent or checksum mismatch | Block (`UNSTAMPED`, PR #144) |

The pr-scope gate is uniformly fail-closed (matching the review-artifact gate's posture: a missing dependency — `gh`, the artifact — always blocks). The loop-scope gate is deliberately **fail-open on absence** (missing `work_units` field, missing `jq`) but **fail-closed on presence-with-failure** (an `evals.json` that exists and says `NO-GO`, or is missing its justification, still blocks) — the asymmetry exists because `work_units` is a newly-added `progress.json` field that older, already-in-flight loops won't carry, and a hard requirement on a field that didn't exist when a loop started would wrongly punish legacy loops rather than catch a genuine gap. The `tier_justification` check (PR #10) is an exception to that leniency by owner directive — it fires retroactively even against pre-existing GO artifacts.

## Verifier agent contract (agent-run evals)

For judgement evals, a fresh sonnet subagent is spawned to grade. Its prompt contains ONLY the `evals.json` content, artifact references (PR number, clone path, artifact path or local endpoint, deployed surface), and the confidence-label contract — explicitly nothing else, not the implementation conversation, not the implementer's summary, not the orchestrator's opinion of the outcome. Same "the author is the least able to see its own shims" principle as agentic-loop Phase 4b's clean-break gate (see [[agentic-loop]]). The assembly script computes `result`; the verifier never writes it directly. Folding applies to fresh grader output only: an eval amended after a grader verdict goes back to a fresh grader, and the post-verdict amendment records who re-graded in `regraded_by` (PR #153 — see the backstop above).

## The honest ceiling — inherited, not re-solved

This gate proves a durable, SHA-bound (pr scope) or path-bound (loop scope), structurally-valid artifact **exists** with the right result. It does not — cannot — prove the underlying evals were run honestly, that a scripted `cmd` genuinely exercised the asserted surface, or that an `agent-run` verifier subagent was genuinely given a clean, uncontaminated context. This is the **identical** honest ceiling [[review-artifact-seam]] already documents for the review artifact: an auditable layer, not a tamper-proof barrier. `task-evals` inherits this ceiling rather than closing it — a pending owner decision, not resolved by this cluster. See [[enforcement-model]] for the general framework.

**One specific slice of this ceiling — the tier-0 exemption itself — gained an opt-in closer.** Tier 0's `tier_justification` is self-written by the same party it exempts; nothing in this gate, and nothing in [[task-evals]]'s own generation rules, ever reviewed whether a `tier=0` claim was honest — that stayed true until [[pr_232_tier-review-gate|PR #232]] (2026-07-17). Where a project opts in (`config.tier_review.machine_user` set), a separate root-owned daemon (`scripts/tier-gate/`) judges the PR's claimed tier against its real, capped diff — never the `tier_justification` prose itself, which the judge does not read at all — via a subscription-authenticated Claude call outside the agent's own trust domain, then posts a `tier-review` commit status that both [[merge]] and [[enforce_pr_workflow]] additionally check before allowing a tier-0 merge. This narrows the ceiling for tier-0 specifically; it does not touch the ceiling for tier 1/2 evals (a scripted `cmd` genuinely reflecting reality, or a verifier subagent genuinely being given clean context) at all — see [[pr_232_tier-review-gate]] for the full mechanism, its own honest boundary (available, not yet active in this repo), and why it raises the cost of a dishonest tier-0 from free to expensive rather than claiming impossibility.

## Known gaps — status

- ~~**Comment-spoofing trust model** — inherited wholesale from the review-artifact gate; a sufficiently motivated cooperating agent could, in principle, post a structurally-valid artifact without the evals genuinely having been run as described.~~ **Closed for both PR-gate readers by [[pr_7-10_task-evals-followups|PR #8]]** (`62ad18d`, 2026-07-06): `scripts/lib/git-common.sh` now filters comments to the `gh`-authenticated login with `author_association == "OWNER"` before any marker matching, fail-closed on identity-fetch failure.
- ~~**Scoped limitation: `OWNER` association assumes a personally-owned repo; fails closed on org repos**~~ — **Closed by [[pr_11-14_gate-hardening-followups|PR #14]]** (`7c1dd19`, 2026-07-06, WU4, owner directive): the `author_association == "OWNER"` conjunct is **removed entirely** (clean break — independently verified no surviving compat path). Trust now requires login match with the merging identity (unchanged anti-spoof property) **AND** that identity holding write access or better (`ADMIN`/`MAINTAIN`/`WRITE` via `gh repo view --json viewerPermission`) on the current repo — new `pr::_trusted_permission()` / `pr::_permission_is_write_or_better()` in `scripts/lib/git-common.sh`. Works identically on personal and org-owned repos now. A resolvable-but-insufficient permission (e.g. `READ`) is treated as "no trusted comment found" (not a fetch failure); an actual lookup failure fails closed, same posture as the identity-fetch failure. `INSTALLATION.md` documents the rule for operators. This is the flip that the prior bullet's own "not-yet-made owner decision" note anticipated — it landed as a permission check, not the config-allowlist alternative that same note floated (that alternative was rejected as unneeded once the permission-check shape was chosen; see [[pr_11-14_gate-hardening-followups]]). The honest ceiling above (artifact truthfulness) is explicitly **not** touched by this change — it proves *who* posted a marker, not that its contents are truthful.
- ~~**`gh` comment pagination** — the newest-wins scan does not paginate; a PR with enough comments to span pages could miss an older page's marker.~~ **Closed by the same [[pr_7-10_task-evals-followups|PR #8]]**: both readers now fetch via `gh api "repos/<repo>/issues/<n>/comments" --paginate`, replacing the `gh pr view --json comments` GraphQL call that was hard-capped at 100 comments.
- ~~**`install.sh` inventory gap** — does not yet name `scripts/lib/eval-artifact.sh` or `scripts/post_evals.sh` in its own audit surface the way it tracks other core scripts.~~ **Closed by [[pr_1-4_task-evals-feature|PR #3]]** (`9ecbeae5`, 2026-07-06) — `install.sh`'s script-chmod loop now lists both files (verified: `install.sh:332`). **A related gap in the same loop — `scripts/lib/review-artifact.sh` and `scripts/lib/config.sh` were also missing — closed by [[pr_7-10_task-evals-followups|PR #9]]** (`ffe5ccc`, 2026-07-06). The loop still has no `scripts/lib/*.sh` glob, so any future addition under `scripts/lib/` needs the same manual literal-list update repeated — a structural gap, not fully closed.
- **`tier_justification` now required at every tier, not just tier 0** — [[pr_7-10_task-evals-followups|PR #10]] (owner directive, `238f5e1`, 2026-07-06). See "tier_justification required at every tier" above for the mechanism.
- ~~**PR-scope gate enforced only inside `scripts/merge.sh`'s own body — nothing forced a merge to route through it, so a raw `gh pr merge` bypassed the eval check entirely** (documented same-day in [[evals-gate-enforcement-gap_2026-07-08]], the mechanism [[pr_95_slash-command-loop-detection|PR #95]] shipped through).~~ **Closed by [[pr_96-98_evals-gate-uniform-enforcement_2026-07-08|PR #97]]** (`5b96690`, 2026-07-08): `enforce_pr_workflow.sh` now gates raw `gh pr merge <N>` on the identical SHA-bound artifact check, config-dependent (`NO_CONFIG`-gated) where `merge.sh`'s own gate is config-independent — see "Merge gate placement" above. Companion invocation-path gap (`systematic-debugging` had no route to `task-evals` at all) closed prose-level by the same cluster's PR #96.

## Owner decisions recorded

- **Hybrid grading** — scripted `cmd`/`negative_control` pairs for deterministic checks, `agent-run` mode with a fresh independent verifier for judgement calls, both supported in the same `evals` array.
- **Hook-enforced from day one** — the loop-scope gate ships live in the same PR that adds it; no separate advisory/observation-only period, unlike `enforce_pr_workflow`'s original transcript-gate-then-artifact-gate phase-in.
- **One schema, two seams — now three consumers** — pr scope and loop scope share `schema_version: 1` verbatim; the `scope` field is the only fork point, deliberately, so the gates can't drift on shape. As of [[pr_144-149_agentic-loop-hardening-from-loop-engineering|PR #148/#149]] (2026-07-12), the dashboard's `sessions.ts` `readEvalsFrozen` is a **third, independent consumer** of the loop-scope shape — read-only, non-enforcing, but still capable of *displaying* a wrong verdict if its logic drifts from the bash SSOT. This is exactly what happened between #148 and #149: the first cut's `grade-loop`-stamp check required only `.grading.checksum` non-empty, not `.grading.by` — weaker than the bash reader's `[ -z "$stamped_by" ] || [ -z "$stamped_checksum" ]` check — so a partially-stamped artifact could show "frozen" on the dashboard while the hook would correctly block it as `UNSTAMPED`. #149 fixed the TS reader to require both fields non-empty. The dashboard reader deliberately skips checksum *recomputation* (KISS, since it can't block anything) but must still track the bash reader's exact boolean shape for presence checks — the two-seams framing above undersells the drift risk now that a third, purely-cosmetic-but-user-facing reader exists on the same schema.
- **Newest-artifact-wins** — a stale `GO` must never outrank a fresher `NO-GO` for the same SHA (pr scope); mirrors the same design question the review-artifact reader already answered the same way.

## Architecture position

```
task-evals (skill)                         task-evals (skill, scope: loop)
    ↓ freeze-before-build                       ↓ freeze-before-build (Phase 2.7c)
evals.json (pr scope, working material)     evals.json (loop scope, beside progress.json)
    ↓
/coderails:post-evals                       loop_state_guard.sh reads it directly
    ↓ validates structure, computes GO/NO-GO    ↓ als_read_loop_evals_result
SHA-bound PR comment (marker SSOT)              ↓
    ↓                                        gate_loop_evals_required
scripts/merge.sh gate (after review gate)       ↓ (work_units >= 3 required)
    ↓                                        BLOCK on NO-GO/ABSENT, allow on GO/TIER0
gh pr merge
```

## See also

- [[task-evals]] — the skill that generates and freezes the evals this gate consumes
- [[post-evals]] — the command that validates and posts the pr-scope artifact
- [[merge]] — the updated command with the additive eval-artifact gate
- [[loop_state_guard]] — the extended C1 hook carrying the loop-scope gate
- [[loop-progress-fields]] — consolidating page for `work_units`, the field the loop-scope `>=3`-work-unit threshold this gate applies is keyed off
- [[review-artifact-seam]] — the predecessor design this pattern is largely mirrored from (marker SSOT, SHA-bound artifact, fail-closed gate, honest-ceiling framing)
- [[agentic-loop]] — Phase 2.7c (freeze loop-scope evals) and Phase 13 (unscored result reporting)
- [[writing-plans]] — mandatory final eval-gate task (pr scope)
- [[enforcement-model]] — the honest-ceiling framework this design sits within
- [[pr_1-4_task-evals-feature]] — the original cluster source page (PRs #1–4)
- [[pr_7-10_task-evals-followups]] — the follow-up cluster closing comment-spoofing/pagination and adding tier_justification-everywhere (PRs #7–10)
- [[pr_11-14_gate-hardening-followups]] — the gate-hardening cluster: explicit NO-GO wins at tier 0 (#11), HOME-sandboxed install test (#12), push.sh staging fix (#13), trust-floor widened to a permission check + merge.sh error-message split (#14)
- [[trust-floor]] — consolidating concept page for the trust-floor/OWNER-permission model this gate's comment-fetch reader relies on (SSOT for the mechanism is [[merge]]); extended with the `tempfile` failure-reason case by PR #21
- [[pr_21-22_loop2-suggestion-tier-followups]] — Loop 2 follow-up: merge.sh's `tempfile` case arm (shared fetch helper, both gates) + test-coverage completions for `loop_state_guard_evals.test.sh` (final-else NO-GO fixture) and `install_mode_sweep.test.sh`
- [[pr_96-98_evals-gate-uniform-enforcement_2026-07-08]] — closes the "nothing forces the merge through `merge.sh`" gap: adds a second, hook-level pr-scope gate on raw `gh pr merge` (PR #97) plus a `systematic-debugging` → `task-evals` invocation cross-reference (PR #96)
- [[evals-gate-enforcement-gap_2026-07-08]] — the investigation that identified the gap the above cluster closes
- [[pr_144-149_agentic-loop-hardening-from-loop-engineering]] — PRs #144-149 (2026-07-12): `grade-loop` neutral grading + `UNSTAMPED` demotion (#144), rule 6 "Strongest surface" (#145), the dashboard's third-seam `readEvalsFrozen` drift (#148/#149)
- [[pr_155-158_ceremony_noise_envelope_anchoring]] — PR #158 (2026-07-13): rule 4 "Oracle independence" extended with the loop-scope `authorising_prompt_raw` precedence rule, wired into [[agentic-loop]] at Phase -2/-1/2.7c/13
- [[dashboard]] — the skill whose `sessions.ts`/`RailLeft.tsx` are the third schema consumer named above
- [[pr_232_tier-review-gate]] — PR #232 (2026-07-17): root-daemon tier-0 honesty judge, opt-in, closes the tier-0-specific slice of the honest ceiling above
- [[pr_218_discriminating-check-gate]] — PR #218 (2026-07-17): the discriminating-check gate — `fixtures`-based freeze-time proof that a scripted check's formula can both pass and fail, closing the class rule 2's text-only negative-control check cannot
