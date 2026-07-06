---
title: "PRs #1–4 (blueman82/coderails) — task-evals feature: game-resistant success-evals discipline + dual-scope enforcement"
type: source
created: 2026-07-06
last_updated: 2026-07-06
sources: []
tags: [task-evals, evals-json, anti-gaming, merge-gate, loop-gate, eval-artifact, post-evals, agentic-loop, writing-plans, teamcreate-purge, doc-drift, sync-docs]
---

# PRs #1–4 — task-evals feature

<!-- Ingested by /wiki-ingest after merge. This is an immutable record of what changed. -->

## PR metadata

| Field | Value |
|---|---|
| PR numbers | #1, #2, #3, #4 |
| Repo | `blueman82/coderails` |
| Branches | `task-evals/wu1-skill`, `task-evals/wu3-loop-gate`, `task-evals/wu2-pr-pipeline`, `task-evals/wu4-wiring` |
| Merged | 2026-07-06 (all four, same day) |
| Merge SHAs | #1 `0fc8350`, #2 (loop-gate) `5a5df44`, #3 (pr-pipeline) `3b4be2f`, #4 `72261f7` |
| JIRA ticket | — |

## Summary

A new skill, `task-evals`, and two enforcement seams that consume it. The problem statement (from `SKILL.md`'s own "Why this skill exists"): every existing coderails verification loop is a **self-verification** loop — the same agent that implements also writes its own verify-criteria, runs them, and grades itself. `writing-plans` verify-criteria are written by the process that then implements against them; agentic-loop workers verify their own artifact; Phase 13 self-audits are explicitly unscored. `task-evals` generalises the one place this pattern was already broken — the hand-written public-readiness suite (E0–E10) — into a reusable discipline: **freeze a tiered, game-resistant `evals.json` before implementation starts**, then gate merge (pr scope) or loop completion (loop scope) on it.

Four PRs, each a distinct work unit (WU1–WU4), landed the same day:

- **PR #1 (WU1 — skill + design)**: `skills/task-evals/SKILL.md` + `docs/coderails/specs/2026-07-03-task-evals-design.md`. Defines five anti-gaming rules, a tier 0/1/2 predicate ladder, the eval anatomy, and the schema (`schema_version: 1`).
- **PR #2 (WU3 — loop-scope gate)**: `hooks/scripts/lib/loop_state_common.sh` + `hooks/scripts/loop_state_guard.sh` + `loop_state_guard_evals.test.sh`. Blocks `LOOP-STOP: complete` at ≥3 work-units without a passing loop-scope `evals.json` beside `progress.json`.
- **PR #3 (WU2 — pr-scope pipeline)**: `scripts/lib/eval-artifact.sh` (marker SSOT) + `scripts/post_evals.sh` (writer/validator) + `git-common.sh` reader + `scripts/merge.sh` second gate + 4 test files. Fail-closed SHA-bound PR-comment artifact, mirroring the existing review-artifact seam.
- **PR #4 (WU4 — wiring, 8 docs files)**: wires both gates into `agentic-loop`, `writing-plans`, `merge.md`/`workflow.md`, `AGENTS.md`, `README.md`, `docs/REFERENCE.md`. Also purges stale `TeamCreate`/`TeamDelete` tool references across the repo (the actual mechanism is named-teammate `Agent` spawns + shared `TaskCreate`/`TaskUpdate` + `SendMessage`) down to one legitimate user-phrasing trigger in `agentic-loop`'s own frontmatter.

Notably, **PR #4 was itself the new gate's first customer**: a tier-1 pr-scope eval artifact was frozen (5/5 evals) before PR #4's own docs edits began, and graded GO before merge — the eval-gate pipeline validating itself on the PR that wires it in.

## Design: the five anti-gaming rules (WU1)

Every eval `task-evals` generates must satisfy all five, or it is not a valid eval (verified: `skills/task-evals/SKILL.md`):

1. **Freeze-before-build** — evals are generated and frozen (`frozen_at` + `frozen_sha`) before implementation starts. Post-freeze edits are recorded `amendments`, visible and auditable.
2. **Negative controls** — every scripted eval carries a command proving the check *can* fail. An eval that has never failed proves nothing.
3. **End-state surfaces** — assertions run against merged state, fresh clone, or deployed artifact — never a working-tree self-report.
4. **Oracle independence** — an eval must not share its oracle with the implementation (same regex, same fixture, same test the implementation writes).
5. **Grader independence** — judgement (`agent-run`) evals are graded by a fresh subagent given only `evals.json` + artifact references, never the implementation conversation. The orchestrator never hand-writes `result`; a neutral assembly script computes it.

A mandatory **gameability self-check** runs once per eval before freezing: *"Can the implementer pass this by (a) editing the eval, (b) asserting on the working tree, (c) self-reporting, or (d) reusing its own oracle? Any yes → rewrite."*

## Tier rules

- **Tier 0 (exempt, justified)** — single work-unit AND no outward/irreversible surface AND an existing test/verify-criterion already covers the goal state. Still a written artifact (`tier_justification` required) — gates accept a justified exemption, never silence.
- **Tier 1 (standard)** — 3–5 end-state evals, ≥1 negative control, P0/P1 split.
- **Tier 2 (full suite)** — ≥3 work-units (same line agentic-loop Phase 2.7/3 already draw) OR any irreversible/outward surface (publish, deploy, migration, data deletion, external send).

## Schema (schema_version 1)

```json
{
  "schema_version": 1,
  "scope": "pr | loop",
  "task_ref": "<branch/PR# for pr scope; session loop ordinal for loop scope>",
  "tier": 0,
  "tier_justification": "<required when tier is 0>",
  "frozen_at": "<ISO8601>",
  "frozen_sha": "<base SHA at freeze>",
  "evals": [
    {
      "id": "E1", "priority": "P0", "mode": "scripted",
      "surface": "merged-state | fresh-clone | artifact-path | deployed",
      "assert": "<goal-state assertion>", "cmd": "<command>",
      "negative_control": "<command proving the check can fail>",
      "status": "pending | pass | fail", "evidence": "<cmd + exit code + output>"
    }
  ],
  "amendments": [ { "eval": "E1", "when": "<ISO8601>", "why": "<reason>" } ],
  "result": null, "graded_at": null, "head_sha": "<SHA graded against>"
}
```

GO requires **all P0 evals pass**. P1 failures don't block but must be listed unresolved — visible debt, not silently dropped. `eval_artifact::compute_go` (`scripts/lib/eval-artifact.sh`) is the ONE place `result` is derived — a pure jq predicate over `.evals[] | select(.priority=="P0")`.

## Two independent gates, one shared schema

Both seams read the same `evals.json` shape but at different scopes, and were built as separate work-units (WU2, WU3) with independent fail-open/fail-closed postures suited to their own hook.

### PR-scope: SHA-bound PR-comment artifact (WU2, PR #3)

Mirrors the existing [[review-artifact-seam]] almost exactly:

- **Marker** (`scripts/lib/eval-artifact.sh`): `<!-- coderails-eval-summary v1 pr=<N> head_sha=<sha> result=<GO|NO-GO> tier=<0|1|2> -->`
- **Matching**: literal prefix string-equality through `head_sha=<sha> result=`, never a regex built by interpolating untrusted `pr`/`head_sha` values — closing a metacharacter-injection path a first-round review found (`pr="1|.*"` could otherwise widen an anchored regex's alternation). Only after the literal prefix matches does the code defer to `parse_result`/`parse_tier`, whose remaining `result=(GO|NO-GO) tier=[0-2] -->$` grammar is a **safe** anchored regex (it never interpolates the untrusted values).
- **Newest-artifact-wins**: `pr::has_coderails_eval_for_head` scans ALL matching marker lines for the PR and uses the LAST (newest) match — a critical review finding closed a stale-GO-overrides-newer-NO-GO gap where an earlier passing comment could outrank a later failing re-grade for the same SHA.
- **`scripts/merge.sh` gate**: added directly after the existing review-artifact gate, in the same `OPEN` branch, before `gh pr merge`. Same rc-contract shape: 0 = match/GO, 1 = no-match or NO-GO, 2 = `gh` fetch failed. Fail-closed, no local fallback, no config opt-out — identical posture to the review gate.
- **`post_evals.sh` structural refusals** (7 checks, first-failure-wins): (1) file exists + valid JSON; (2) tier-0 requires non-empty `tier_justification`; (3) tier≥1 scripted eval must have non-empty `negative_control`; (4) `negative_control` must not be vacuous relative to `cmd` — whitespace-normalised equality check PLUS a word-bounded containment check (catches `"true; cmd"`, `"echo x && cmd"` wrapper tricks without false-flagging a genuinely distinct control that happens to share a text prefix); (5) any P0 eval must carry non-empty `evidence`; (6) `head_sha` in the file must match the PR's current live head SHA; (7) tier≥1 requires **at least one actual P0 eval** in `.evals` — added in the review-fix round, closing a vacuous-GO gap where an empty or P1-only `.evals` array passed `compute_go`'s P0-only predicate vacuously.

### Loop-scope: work-unit-threshold Stop-hook gate (WU3, PR #2)

Lives entirely in the existing C1 hook, `loop_state_guard.sh`, extended rather than given a new script:

- **New reader**: `als_read_work_units()` reads `.work_units | length` off `progress.json` into `ALS_WORK_UNIT_COUNT`. **Fail-open**: absent file, absent/null `.work_units` (a legacy loop predating this field), or malformed JSON all resolve to `0` — absence must never itself trigger a block.
- **New reader**: `als_read_loop_evals_result()` reads a sibling `evals.json` beside `progress.json` into `ALS_LOOP_EVALS_RESULT` ∈ `GO | TIER0 | NO-GO | ABSENT`. `ABSENT` covers no file, malformed JSON, or a non-`"loop"` scope (a stray pr-scope file left in the same dir must never satisfy the loop gate). **Distinct fail-open path for missing `jq`**: logs `evals=skipped reason=jq_missing` and returns without erroring — a separate audit trail from the "file genuinely absent" case.
- **New gate function**: `gate_loop_evals_required()` in `loop_state_guard.sh` — re-checks the same three conditions `als_gate_loop_complete` checks (`status == "complete"` AND not re-armed AND session-owned), because it **must run BEFORE** the shared `als_gate_loop_complete`, which exits 0 directly the instant those three hold. A comment in the source spells out why this ordering is a deliberate exact deviation from how the shared gate functions are normally composed: `als_gate_loop_complete` has no way to signal "checked, still active" back to a caller placed after it, so the new gate duplicates the three-condition check locally rather than restructuring the shared function.
- When those three hold AND `work_units >= 3`: reads the loop evals result. `GO`/`TIER0` → allow (logged, not blocked). `NO-GO`/`ABSENT` → **block** (exit 2) with a message pointing at `/coderails:task-evals` (or a justified tier-0 exemption).
- Below the 3-unit threshold: allow unconditionally, logged `evals=skipped-below-threshold`.
- `loop_stall_guard.sh` (C2, the anti-stall hook) is **untouched** — this gate lives entirely in C1.

## Verifier agent contract (agent-run evals)

For judgement evals, a fresh sonnet subagent is spawned to grade. Its prompt carries ONLY `evals.json` + artifact references (PR number, clone path, deployed surface) + the confidence-label contract — explicitly never the implementation conversation, the implementer's summary, or the orchestrator's opinion of the outcome. Same "the author is the least able to see its own shims" principle as agentic-loop Phase 4b's clean-break gate. The verifier returns per-eval status + evidence; the assembly script (`post_evals.sh` for pr scope, a direct `evals.json` update for loop scope) computes `result` — **the verifier never writes `result` directly**.

## Wiring (WU4, PR #4)

- **agentic-loop Phase 2.7c** (new sub-step, alongside `spec.md`/`plan.md`): invoke `/coderails:task-evals` (scope: `loop`) to freeze the loop's end-state evals. Two independent triggers stated explicitly: reaching Phase 2.7 at all (≥3 work-units) is tier-2-eligible on unit count alone; an irreversible-surface trigger can independently apply even to a <3-unit loop that reached 2.7 via the cross-unit-dependency clause. Also produces per-work-unit pr-scope eval refs that travel into worker prompts verbatim — same "a ref recorded only in `progress.json` and absent from the worker's own prompt does not exist for that worker" rule Phase 3 already applies to disposition.
- **agentic-loop Phase 13**: gains a "Loop-scope eval result" reporting bullet — unscored, alongside the disposition-violation bullet, with the same "no record found for a ≥3-unit loop is an audit failure, not a pass" framing already applied to the disposition record.
- **`progress.json` `work_units` field documented as a shape**: a JSON object keyed by unit id, each entry carrying at least a `status`. This is the field `als_read_work_units` reads `.length` off of — SKILL.md and `AGENTS.md` both now state that keeping it populated for every tracked work-unit is required for the loop-scope gate to see the right count.
- **writing-plans**: gains a mandatory **final eval-gate task** — every plan ends with one task invoking `/coderails:task-evals` (scope: `pr`). Explicitly *not* folded into the last implementation task (the one deliberate exception to writing-plans' own "fold setup/config/docs into the task whose deliverable needs them" rule) — the eval-gate task must run after all other tasks' code exists, since freeze-before-build means evals are frozen before implementation starts but graded against the finished result at the end.
- **`merge.md`/`workflow.md`**: document the eval-artifact gate as **additive to, not a replacement for**, the review-artifact gate — same fail-closed rc semantics, same `OPEN`-branch placement.
- **`AGENTS.md`**: gains a paragraph under the skills↔hooks seam convention naming both new gates explicitly (see the [[merge]] and [[loop_state_guard]] page updates below for the exact text).
- **`docs/REFERENCE.md`**: new `/coderails:post-evals` command-reference row; `/coderails:merge` row updated to "Requires a coderails review artifact AND a coderails eval artifact."
- **TeamCreate/TeamDelete purge**: every prose reference to the dead `TeamCreate`/`TeamDelete` tools rewritten to describe the actual mechanism — named-teammate `Agent` spawns, a shared task list via `TaskCreate`/`TaskUpdate` with `blockedBy` dependencies, and `SendMessage` for coordination. `docs/REFERENCE.md` additionally records that Claude Code v2.1.178 removed the `TeamCreate`/`TeamDelete` tools outright. One quoted user-phrasing trigger survives verbatim in each of `agentic-loop`'s frontmatter `description` and `docs/REFERENCE.md` ("if the user has explicitly named `TeamCreate` in their prompt...") — because the trigger is describing what a *user might type*, not what the orchestrator invokes; `REFERENCE.md`'s catalogue entry for the skill was also updated with the eval-gate cross-reference.

## Files changed

**PR #1**: `skills/task-evals/SKILL.md`, `docs/coderails/specs/2026-07-03-task-evals-design.md`

**PR #2**: `hooks/scripts/lib/loop_state_common.sh`, `hooks/scripts/loop_state_guard.sh`, `hooks/scripts/tests/loop_state_guard_evals.test.sh`

**PR #3**: `scripts/lib/eval-artifact.sh`, `scripts/post_evals.sh`, `scripts/lib/git-common.sh`, `scripts/merge.sh`, `commands/post-evals.md`, `install.sh`, 6 test files (`eval-artifact.test.sh`, `exec_bit_invariant.test.sh`, `git-common.test.sh`, `merge.test.sh`, `merge_evals_gate.test.sh`, `post_evals.test.sh`)

**PR #4**: `AGENTS.md`, `README.md`, `commands/merge.md`, `commands/workflow.md`, `docs/REFERENCE.md`, `skills/agentic-loop/SKILL.md`, `skills/task-evals/SKILL.md`, `skills/writing-plans/SKILL.md`

## Wiki pages updated

- New: [[task-evals]] (skill page)
- New: [[post-evals]] (command page)
- New: [[task-evals-gate]] (design page — the dual-scope seam architecture)
- Updated: [[loop_state_guard]], [[merge]], [[workflow]], [[agentic-loop]], [[writing-plans]]

## Caveats / gotchas

- **Trust model inherited, not re-solved.** The eval-artifact gate is a GitHub PR comment, same as the review artifact it mirrors — it proves an SHA-bound, structurally-valid artifact **exists**, not that the underlying evals were run honestly or that the verifier subagent was genuinely independent. This is the identical honest ceiling [[review-artifact-seam]] already documents for its own artifact; `task-evals` inherits it rather than closing it. Owner decision on tightening this further is pending — not resolved by this cluster.
- **`gh` comment pagination** is a known, un-hardened gap in the newest-wins scan (same gap exists in the review-artifact reader it mirrors) — a PR with enough comments to paginate could have an older page's marker missed. Not fixed in this cluster.
- ~~**`install.sh`** does not yet reference `scripts/lib/eval-artifact.sh` or `scripts/post_evals.sh` by name in its own inventory/audit surface the way it does other core scripts — flagged, not fixed.~~ **Closed by PR #3** (`9ecbeae5`, 2026-07-06) — see addendum below.
- **Hook-enforced from day one was a deliberate owner choice**, not a phase-in: `loop_state_guard.sh`'s work-unit threshold gate ships live in the same PR that adds it, with no separate "advisory period." Contrast with `enforce_pr_workflow`'s original transcript-evidence gate, which predated its own SHA-bound artifact successor by weeks.
- **Hybrid grading is intentional**: scripted `cmd`/`negative_control` pairs for deterministic checks, `agent-run` mode with a fresh independent verifier for judgement calls — the schema supports both in the same `evals` array, and `post_evals.sh`'s structural refusals apply per-mode (e.g. check 3/4 only fire on `mode == "scripted"`).

## Addendum: PRs #5–#6 (follow-up doc drift + TeamCreate purge)

Two small follow-up PRs, merged the same day as the WU1–WU4 cluster above, both docs-only.

### PR #5 (`task-evals/wu6-doc-drift`, merge SHA `574ecc3`)

A `sync-docs` audit pass found the WU1–WU4 rollout had left drift in its own reference material: stale REFERENCE tables, a skill-count figure, the README catalogue, and a `/coderails:workflow` cross-reference. Title: "task evals/wu6 doc drift". Body (verbatim): "docs: fix task-evals rollout drift — REFERENCE tables, skill count, README catalogue, workflow ref (sync-docs audit)". A review finding on this PR additionally corrected the **pr-scope `evals.json` location row** in `docs/REFERENCE.md` — the table had the wrong path for where a pr-scope eval artifact's working file lives before it's posted.

### PR #6 (`docs/teamcreate-final-purge`, merge SHA `7aab163`)

The owner made a final call to remove the **last** literal `TeamCreate` trigger-phrase strings from coderails prose — including the two survivors WU4 (PR #4, above) deliberately preserved as "this is what a user might type" quotes in `agentic-loop`'s frontmatter `description` and `docs/REFERENCE.md`. Title: "docs/teamcreate final purge". Body (verbatim): "docs: remove last TeamCreate trigger-phrase strings (owner decision — full purge)". Those two remaining occurrences are now rewritten to natural equivalents — "create a team", "team of agents", "spawn a team" — completing the purge WU4 had left at "one legitimate user-phrasing trigger" by owner decision at the time; PR #6 revisits that decision and closes it out fully.

Net effect: the TeamCreate/TeamDelete purge described under "Wiring (WU4, PR #4)" above is now complete with **zero** remaining literal-string references anywhere in coderails prose, not "one legitimate survivor."
