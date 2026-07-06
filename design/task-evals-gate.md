---
title: "Task-evals gate — dual-scope (pr + loop) enforcement over frozen, oracle-independent success evals"
type: design
created: 2026-07-06
last_updated: 2026-07-06
sources:
  - sources/pr_1-4_task-evals-feature.md
  - sources/pr_7-10_task-evals-followups.md
  - docs/coderails/specs/2026-07-03-task-evals-design.md
tags: [design, task-evals, evals-json, merge-gate, loop-gate, truth-seam, sha-bound, work-units, oracle-independence, comment-spoofing, tier-justification]
---

# Task-evals gate

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
  "tier": 0, "tier_justification": "<required when tier is 0>",
  "frozen_at": "<ISO8601>", "frozen_sha": "<base SHA at freeze>",
  "evals": [ { "id": "E1", "priority": "P0", "mode": "scripted",
    "surface": "merged-state | fresh-clone | artifact-path | deployed",
    "assert": "...", "cmd": "...", "negative_control": "...",
    "status": "pending | pass | fail", "evidence": "..." } ],
  "amendments": [], "result": null, "graded_at": null, "head_sha": "<SHA graded against>"
}
```

`eval_artifact::compute_go` (`scripts/lib/eval-artifact.sh`) is the ONE place `result` is derived: a pure `jq` predicate requiring every `.priority == "P0"` eval to have `.status == "pass"`. An eval with no `priority` field is simply excluded from the P0 gate by design (not a bug — `post_evals::validate_structure` check 7 is the layer that refuses a tier≥1 artifact with zero real P0 evals, keeping `compute_go` itself an unopinionated pure gate).

## The five anti-gaming rules (generation-time discipline)

Full detail on [[task-evals]]. Named here because they're what the two enforcement gates below are trusting was followed at generation time — the gates themselves cannot re-verify oracle independence or grader independence; they can only verify structural shape and P0 pass/fail:

1. Freeze-before-build
2. Negative controls
3. End-state surfaces
4. Oracle independence
5. Grader independence

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

### Structural refusals (`post_evals::validate_structure`, 7 checks)

1. File exists + valid JSON.
2. Tier 0 requires non-empty `tier_justification`.
3. Tier ≥1 scripted eval requires non-empty `negative_control`.
4. `negative_control` must not be vacuous relative to `cmd` — whitespace-normalised equality **plus** a word-bounded containment check (catches `"true; cmd"`, `"echo x && cmd"` wrapper tricks) without false-flagging a genuinely distinct control sharing a text prefix. This is a **structural** floor, not a semantic one — a genuinely-distinct-but-still-vacuous control (passes for unrelated reasons) still clears this check; semantic quality is the verifier/human review layer's job.
5. Any P0 eval requires non-empty `evidence`.
6. `head_sha` in the file must match the PR's current live head SHA.
7. Tier ≥1 requires at least one actual P0 eval in `.evals` — added in the review-fix round, closing a **vacuous-GO gap**: without this check, a tier-1+ artifact with an empty or P1-only `.evals` array would pass `compute_go`'s P0-only predicate vacuously (no P0 evals to fail = trivially "GO"). Tier 0 is exempt by design.

## Loop-scope gate

Lives entirely inside the pre-existing `loop_state_guard.sh` Stop hook (C1) — extended, not replaced or forked into a new script.

### New shared-lib readers (`loop_state_common.sh`)

- `als_read_work_units()` — reads `.work_units | length` off `progress.json` into `ALS_WORK_UNIT_COUNT`. **Fail-open**: absent file, absent/null `.work_units` (a legacy loop that predates this field), or malformed JSON all resolve to `0`. Absence must never itself cause a block — the same presence-not-provenance posture the rest of `loop_state_guard` already takes.
- `als_read_loop_evals_result()` — reads a sibling `evals.json` beside `progress.json` into `ALS_LOOP_EVALS_RESULT` ∈ `GO | TIER0 | NO-GO | ABSENT`. `ABSENT` covers no file, malformed JSON, or a non-`"loop"` scope — a stray pr-scope file left in the same loop-state dir must never satisfy the loop gate. **Missing `jq`** gets its own distinct fail-open branch and audit-log line (`evals=skipped reason=jq_missing`) — a different, explicitly-logged reason from "file genuinely absent."

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
- `GO` or `TIER0` → allow, logged.
- `NO-GO` or `ABSENT` → **block** (exit 2), pointing at `/coderails:task-evals` (or a justified tier-0 exemption).

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
| loop-scope Stop gate | `work_units >= 3`, `evals.json` result `NO-GO` | Block |
| loop-scope Stop gate | `work_units >= 3`, `tier_justification` blank/whitespace-only (any tier) | Block (`UNJUSTIFIED`, PR #10) |
| loop-scope Stop gate | `work_units >= 3`, `evals.json` result unrecognised value | Block (`reason=unrecognised_evals_result`, PR #10, fail-closed catch-all) |
| loop-scope Stop gate | `work_units >= 3`, `evals.json` result `GO` or justified `TIER0` | Allow |

The pr-scope gate is uniformly fail-closed (matching the review-artifact gate's posture: a missing dependency — `gh`, the artifact — always blocks). The loop-scope gate is deliberately **fail-open on absence** (missing `work_units` field, missing `jq`) but **fail-closed on presence-with-failure** (an `evals.json` that exists and says `NO-GO`, or is missing its justification, still blocks) — the asymmetry exists because `work_units` is a newly-added `progress.json` field that older, already-in-flight loops won't carry, and a hard requirement on a field that didn't exist when a loop started would wrongly punish legacy loops rather than catch a genuine gap. The `tier_justification` check (PR #10) is an exception to that leniency by owner directive — it fires retroactively even against pre-existing GO artifacts.

## Verifier agent contract (agent-run evals)

For judgement evals, a fresh sonnet subagent is spawned to grade. Its prompt contains ONLY the `evals.json` content, artifact references (PR number, clone path, deployed surface), and the confidence-label contract — explicitly nothing else, not the implementation conversation, not the implementer's summary, not the orchestrator's opinion of the outcome. Same "the author is the least able to see its own shims" principle as agentic-loop Phase 4b's clean-break gate (see [[agentic-loop]]). The assembly script computes `result`; the verifier never writes it directly.

## The honest ceiling — inherited, not re-solved

This gate proves a durable, SHA-bound (pr scope) or path-bound (loop scope), structurally-valid artifact **exists** with the right result. It does not — cannot — prove the underlying evals were run honestly, that a scripted `cmd` genuinely exercised the asserted surface, or that an `agent-run` verifier subagent was genuinely given a clean, uncontaminated context. This is the **identical** honest ceiling [[review-artifact-seam]] already documents for the review artifact: an auditable layer, not a tamper-proof barrier. `task-evals` inherits this ceiling rather than closing it — a pending owner decision, not resolved by this cluster. See [[enforcement-model]] for the general framework.

## Known gaps — status

- ~~**Comment-spoofing trust model** — inherited wholesale from the review-artifact gate; a sufficiently motivated cooperating agent could, in principle, post a structurally-valid artifact without the evals genuinely having been run as described.~~ **Closed for both PR-gate readers by [[pr_7-10_task-evals-followups|PR #8]]** (`62ad18d`, 2026-07-06): `scripts/lib/git-common.sh` now filters comments to the `gh`-authenticated login with `author_association == "OWNER"` before any marker matching, fail-closed on identity-fetch failure. **Scoped limitation, documented in code**: `OWNER` assumes a personally-owned repo; an org-owned repo's comments carry `MEMBER`/`COLLABORATOR` instead, so the gate fails closed there rather than passing. Widening the trust floor (e.g. a config allowlist for CI/service-account posters) remains a separate, not-yet-made owner decision — the flip-condition is recorded: a bot-account artifact poster would justify it.
- ~~**`gh` comment pagination** — the newest-wins scan does not paginate; a PR with enough comments to span pages could miss an older page's marker.~~ **Closed by the same [[pr_7-10_task-evals-followups|PR #8]]**: both readers now fetch via `gh api "repos/<repo>/issues/<n>/comments" --paginate`, replacing the `gh pr view --json comments` GraphQL call that was hard-capped at 100 comments.
- ~~**`install.sh` inventory gap** — does not yet name `scripts/lib/eval-artifact.sh` or `scripts/post_evals.sh` in its own audit surface the way it tracks other core scripts.~~ **Closed by [[pr_1-4_task-evals-feature|PR #3]]** (`9ecbeae5`, 2026-07-06) — `install.sh`'s script-chmod loop now lists both files (verified: `install.sh:332`). **A related gap in the same loop — `scripts/lib/review-artifact.sh` and `scripts/lib/config.sh` were also missing — closed by [[pr_7-10_task-evals-followups|PR #9]]** (`ffe5ccc`, 2026-07-06). The loop still has no `scripts/lib/*.sh` glob, so any future addition under `scripts/lib/` needs the same manual literal-list update repeated — a structural gap, not fully closed.
- **`tier_justification` now required at every tier, not just tier 0** — [[pr_7-10_task-evals-followups|PR #10]] (owner directive, `238f5e1`, 2026-07-06). See "tier_justification required at every tier" above for the mechanism.

## Owner decisions recorded

- **Hybrid grading** — scripted `cmd`/`negative_control` pairs for deterministic checks, `agent-run` mode with a fresh independent verifier for judgement calls, both supported in the same `evals` array.
- **Hook-enforced from day one** — the loop-scope gate ships live in the same PR that adds it; no separate advisory/observation-only period, unlike `enforce_pr_workflow`'s original transcript-gate-then-artifact-gate phase-in.
- **One schema, two seams** — pr scope and loop scope share `schema_version: 1` verbatim; the `scope` field is the only fork point, deliberately, so the gates can't drift on shape.
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
- [[review-artifact-seam]] — the predecessor design this pattern is largely mirrored from (marker SSOT, SHA-bound artifact, fail-closed gate, honest-ceiling framing)
- [[agentic-loop]] — Phase 2.7c (freeze loop-scope evals) and Phase 13 (unscored result reporting)
- [[writing-plans]] — mandatory final eval-gate task (pr scope)
- [[enforcement-model]] — the honest-ceiling framework this design sits within
- [[pr_1-4_task-evals-feature]] — the cluster source page
