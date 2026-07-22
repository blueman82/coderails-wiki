---
title: "Freeze-before-build is a mandatory rule, but compliance with it is never mechanically checked"
type: investigation
created: 2026-07-22
last_updated: 2026-07-22
sources:
  - skills/task-evals/SKILL.md
  - hooks/scripts/lib/loop_state_common.sh
  - skills/agentic-loop/teardown.md
  - skills/loop-retro-promotion/SKILL.md
tags: [investigation, task-evals, freeze-before-build, enforcement, self-reported, retro-json, standing-orders, honest-boundary]
---

# Freeze-before-build: mandatory rule, unverified compliance

## The question

Is there a mandatory requirement that agentic-loop task-evals be frozen at the beginning of a loop (freeze-before-build)? Is that requirement's *satisfaction* ever independently checked anywhere in coderails' own git history or wiki — or does compliance only ever get recorded in `retro.json` and `standing-orders.md` (self-reported, never externally checked)?

## Two separate claims — keep them apart

1. **Is freeze-before-build a real, mandatory rule?** Yes, heavily documented.
2. **Is compliance with that rule ever verified by anything other than the model's own word?** No — confirmed by exhaustive grep across `scripts/` and `hooks/`.

Git history and the wiki are rich on (1) — the rule's creation and refinement — and empty on (2). Conflating "the rule is well-documented" with "compliance is checked" is the trap here; they are not the same claim.

## (1) The rule is real and mandatory

[[task-evals]] rule 1 states it plainly: *"Freeze-before-build. Evals are generated and frozen (timestamp + base SHA) before implementation starts. Post-freeze edits are amendments with recorded reasons."* This is generation-time discipline enforced by prose instruction, not optional guidance — an eval that fails this rule "is not a valid eval" per the skill's own framing.

Git history on origin/main shows the rule being actively built and refined:
- **[[pr_15-17_loop-hardening-registration-eval-freeze-ledger-dry|PR #15]]** (`0030dfd`) fixed the invocation-point wording that had let freeze-before-build collapse into the plan's "final task" — previously mis-worded as "writing-plans' final task," which produced a real unfrozen 17-unit execution in practice. Corrected to "writing-plans' freeze-after-stress-test step."
- **`088f7d8`** (2026-07-17) added the mandatory **freeze-time smoke-run**: every scripted eval's `cmd` and `negative_control` must be executed once, for real, immediately before freezing — closing the gap where a broken instrument (reporter-loading error, module-resolution failure) could sit undetected behind a passing negative control.
- **`9885643`** (PR #218) added the optional **discriminating-check gate** (`fixtures`), catching a formula that can never actually fail or pass regardless of input — evidenced by a real incident (loop `8b69e779`'s awk formula exiting 1 unconditionally).
- Merge commit `80f5969` — PR #196, `fix/pr-scope-evals-freeze-gap` — closed a related pr-scope freeze gap.

All three of these are documentation/prose changes to `SKILL.md` (or to jq validation logic that checks *shape*, not *timing* — see below). None of them add a mechanism that checks *when* the freeze actually happened relative to implementation.

## (2) Compliance is never mechanically checked — confirmed empirically

Grepped every `.sh` script under `scripts/` and `hooks/` (the only places coderails' actual gates live — `post_evals.sh`, `loop_state_guard.sh`/`loop_stall_guard.sh` and their shared lib `loop_state_common.sh`) for `frozen_at` and `frozen_sha`:

```
$ grep -rn "frozen_at\|frozen_sha" scripts/ hooks/
(no output — zero matches)
```

`frozen_at`/`frozen_sha` appear **only** in skill markdown (`task-evals/SKILL.md`, `agentic-loop/SKILL.md`, `subagent-driven-development/SKILL.md`, etc.) — i.e., only in the prose instructions telling the model what to do, never in code that could check it.

What the actual gate code *does* check, confirmed by reading `post_evals::validate_structure` and `als_read_loop_evals_result`/`als_gate_work_units_on_complete`/`als_gate_proofs_on_complete` in `loop_state_common.sh`:
- Presence of `evals.json` / `proof.json`.
- `tier_justification` non-blank at every tier.
- `schema_version >= 1` (or `>= 2` for the proof gate's disposition requirement).
- `result`/`grading` checksum stamp consistency (catches accidental post-grade drift, not deliberate tampering — documented honest boundary).
- For the proof gate specifically: that a frozen proof command was **actually executed in the orchestrator's own transcript** and didn't error (a stronger check than most, but the code's own header is explicit that this "verifies a command RAN... it CANNOT verify it was the RIGHT command," and a session that appends forged transcript records defeats it entirely — again a named, accepted honest boundary).

None of these check that `frozen_at` timestamp / `frozen_sha` predates the first implementation-touching tool call. The proof gate's own source comment even says the proof choice is "time-stamped (frozen at Phase 2.7e, before implementation, per SKILL.md)" — but that clause describes what the *prose instructs*, not something the gate itself re-derives or verifies. No code diffs `frozen_sha` against any later commit; no code compares `frozen_at` against transcript event timestamps.

## The user's suspected mechanism, verified directly

The question named `retro.json` and `standing-orders.md` as the only places compliance might be recorded. Checked both:

- **`retro.json`** (schema documented in [[agentic-loop]]'s teardown detail page, `skills/agentic-loop/teardown.md`) carries an `evals` field — but it's scoped to `result` / `amendments` / unresolved P1s. **There is no field asserting that freeze-before-build happened**, let alone one capturing when. The schema's own stated design principle is explicit: *"The schema has no `verdict` field — raw and unscored is structural, not an oversight: the retro records what happened, it does not grade it."* So `retro.json` isn't even a self-graded compliance record for this specific rule — the freeze-timing fact isn't represented in it at all.
- **`standing-orders.md`** is confirmed to be a per-repo, **outside-git runtime artifact** living in the loop-state directory beside `retro.json` (`<repo-key-dir>/*/`, per [[loop-retro-promotion]]'s SKILL.md) — described there as "the overlay": a place repo-specific recurring lessons accumulate across loops, mined later by the `loop-retro-promotion` pipeline for repo-agnostic promotion into `learned-failure-modes.md`. It is not a coderails source file, not something reviewed at merge, and not itself subject to any gate.

So the premise holds, and is if anything starker than stated: it's not just that compliance is "self-reported in retro.json" — `retro.json`'s schema doesn't even have a slot for this particular fact. Both named files are self-authored, outside-repo, loop-generated artifacts describing the loop's own conduct, consumed later only by a scheduled mining pipeline (`loop-retro-promotion`) that itself runs unreviewed and un-gated.

## Bottom line

- Freeze-before-build is a real, actively-maintained, mandatory generation rule (task-evals rule 1), refined across at least three PRs (#15, freeze-gap #196, the smoke-run commit, and the discriminating-check gate #218).
- No script or hook anywhere in coderails checks that a `frozen_at`/`frozen_sha` stamp actually predates implementation — confirmed by an exhaustive grep returning zero hits outside prose.
- The two files the question names as the suspected self-report channel were checked directly: `retro.json`'s schema has no field for this fact at all, and `standing-orders.md` is an outside-git, per-repo overlay artifact, not a governed record.
- This is the same class of honest boundary the codebase names repeatedly for its other gates (tier-0 self-exemption, the proof gate's "verifies RAN, not RIGHT," the retro-gate's presence-not-honesty check) — freeze-before-build simply has no gate at all, mechanical or otherwise, checking its timing claim.

## See also

- [[task-evals]] — the skill defining the rule (rule 1, gameability self-check, freeze-time smoke-run)
- [[task-evals-gate]] — the dual-scope enforcement architecture; documents what IS checked (structure, tier_justification, grading checksum) and is silent on freeze timing
- [[loop_stall_guard]] — hosts the retro gate, deferral gate, and proof gate; all three check artifact presence/shape, none check freeze timing
- [[loop-retro-promotion]] — confirms `standing-orders.md`'s nature as an outside-git per-repo overlay
- [[pr_15-17_loop-hardening-registration-eval-freeze-ledger-dry]] — PR #15, the invocation-point wording fix
- [[pr_218_discriminating-check-gate]] — PR #218, closes a related but distinct gap (formula can't discriminate), not the freeze-timing gap
