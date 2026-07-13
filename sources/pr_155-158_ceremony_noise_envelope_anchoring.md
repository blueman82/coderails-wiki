---
title: "PRs #155, #157, #158 — loop-scoped warn demotion, LOOP-STOP bundle warn-era prose, envelope-anchored eval authoring"
type: source
created: 2026-07-13
last_updated: 2026-07-13
sources: []
tags: [hook, discipline, loop-demotion, agentic-loop, task-evals, oracle-independence, warn-mode, ceremony-noise]
---

# PRs #155, #157, #158 — ceremony-noise reduction + envelope-anchored eval authoring

Three merged PRs from the same session (1cf76302, 2026-07-13), covering both the mechanism that
demotes discipline-hook blocks to warns inside an active loop, and the doc/skill updates that make
that mechanism's prose match its behaviour. PR #156 (concurrent session's work, the DNV-presence
check) is referenced but not duplicated here — see [[pr_156_dnv-presence-check]].

## PR #155 — loop-scoped warn demotion

| Field | Value |
|---|---|
| PR number | #155 |
| Branch | `feat/loop-scoped-warn-demotion` |
| Merged | 2026-07-13 |
| Merge SHA | `a36fa9d8df1c6f962cbeae3050dbf6701f7e5bf5` |
| JIRA ticket | — |

### Summary

The two discipline Stop hooks — `check_confidence_labels.sh` and `check_verify_loop.sh` — now
**demote** a would-be block to a model-visible warn on the `Stop` event, when the session is inside
an active, incomplete [[agentic-loop]]. The warn is emitted as `hookSpecificOutput.additionalContext`
with `exit 0`, not `exit 2`. `SubagentStop` (worker output) and non-loop sessions are unaffected —
they still hard-block exactly as before. (verified: `hooks/scripts/check_confidence_labels.sh:62-77`,
`hooks/scripts/check_verify_loop.sh:150-161, 225-238`)

**Motivating incident:** loop f87a0a2e forced roughly 69 turn regenerations in one crack-on loop — 49
confidence-label blocks and 20 verify-loop blocks. Each block cost a full model regeneration cycle;
the discipline itself was correct (every one of those responses genuinely needed a label or a DNV
tag), but a hard `exit 2` mid-loop is a different cost profile than the same signal delivered as a
correction the orchestrator can act on next turn without losing the current turn's work. (verified:
`skills/agentic-loop/SKILL.md` Phase 0.5, "Past failure" line, PR #157)

### The predicate: `als_loop_active_incomplete`

New shared-lib predicate in `hooks/scripts/lib/loop_state_common.sh`, deliberately built as a
**non-exiting** function (no `exit` calls, no logging) — callers own both, mirroring the existing
`als_gate_*` pair (`als_gate_require_active_loop` + `als_load_progress` + `als_gate_loop_complete`)
but as a predicate instead of an exiting gate. Truth table (verified:
`hooks/scripts/lib/loop_state_common.sh:340-364`):

- `invocations == 0` → INACTIVE (hook falls through to normal block/allow logic)
- `invocations > 0`, no `progress.json` yet, or absent/corrupt/foreign-owned file → ACTIVE (demote)
- `invocations > 0`, `status == "complete"`, not rearmed, session-owned → INACTIVE (no demotion — the
  loop is genuinely done)
- `invocations > 0`, `status == "complete"`, rearmed (`invocations > completed_marker`) → ACTIVE
- `invocations > 0`, `status == "complete"`, but `session_id` mismatch → ACTIVE (foreign-owned file
  can't certify this session's loop as complete)

The absent/corrupt/foreign-owned case reading ACTIVE is by design, not an oversight: it never falls
into the completed-and-owned exemption, so it demotes — and `loop_state_guard`/`loop_stall_guard`
block those same stops separately via their own gates regardless, so the stop is never left
unpoliced even though this predicate alone only demotes.

### Evaluation is lazy

Both hooks call the predicate **only once a block is imminent** — after the label/DNV check has
already determined a block would fire, not on every Stop event. A non-loop session never pays the
transcript-invocation scan; the cost of the demotion check is proportional to how often a block was
actually about to happen. (verified: both hook source files, comment preceding the demotion branch)

### Fail-toward-blocking

The `jq` emission that produces the `additionalContext` warn runs first, and its own exit status
gates both the log line and the `exit 0`:

```bash
if jq -n --arg m "..." '{"hookSpecificOutput":{"hookEventName":"Stop","additionalContext":$m}}'; then
  # log would_block=1 warned=1 blocked=0, exit 0
fi
# falls through to the normal block path if jq failed
```

If `jq` fails (missing binary, malformed output), execution falls through to the pre-existing block
path below — never silently exits 0 with a log line that falsely claims `warned=1`. (verified:
`hooks/scripts/check_confidence_labels.sh:62-77`)

### Log-line accounting — the success metric for future loops

A demoted warn logs `would_block=1 warned=1 blocked=0`; a genuine pass logs neither; a genuine block
logs `blocked=1`. **The metric to mine for future loops' retros is the `blocked=1` line count, not
the flagged (`would_block=1`) count** — warns deliberately keep logging `would_block=1` every time the
underlying discipline issue is present, so the flagged count stays high by design and does not by
itself indicate a regression. A dropping `blocked=1` count with a steady or even rising `would_block=1`
count is the expected, healthy signature of this change working: the discipline pressure is still
present and visible in the log for retro mining, it just no longer costs a turn regeneration inside
an active loop. (verified: log-line format in both hook scripts; framing per the authorising session)

## PR #157 — LOOP-STOP bundle, warn-era prose

| Field | Value |
|---|---|
| PR number | #157 |
| Branch | `docs/loop-stop-bundle-warn-era` |
| Merged | 2026-07-13 |
| Merge SHA | `fe66400f9e441989074ce27000a548713db45af3` |
| JIRA ticket | — |
| Files changed | `skills/agentic-loop/SKILL.md` only (10 lines: 5 insertions, 5 deletions) |

### Summary

Doc-only PR rewriting `skills/agentic-loop/SKILL.md`'s Phase 0.5 and the Stop-conditions
"Declaring the stop" section so the skill's own prose matches PR #155's warn-era behaviour, which
had shipped without an accompanying skill-text update. Three changes:

1. **Phase 0.5's opening paragraph rewritten.** The pre-#157 text said the orchestrator "trips the
   user's stop hooks" and every block "costs a manual turn to clear" — accurate for the pre-#155
   world, stale afterward. The new text states the warn/block split explicitly: inside an active,
   incomplete loop the two hooks demote to a model-visible `additionalContext` warn rather than
   stopping the turn; outside a loop, and for worker output (`SubagentStop`), both hooks still block
   outright. "The discipline itself hasn't changed, the warn is the correction signal the
   orchestrator acts on next turn." (verified: diff `a36fa9d..fe66400`, `skills/agentic-loop/SKILL.md`)

2. **LOOP-STOP-as-final-line contract clarified.** The pre-#157 bullet described the declaration
   line as something emitted "in the SAME turn" as the confidence-label/DNV requirements, with no
   statement about *where* in the turn it must sit. The new text adds: the declaration must be the
   **FINAL line of the turn** — "that ending-line position is the contract this skill defines and the
   hook's category accounting assumes: when a turn carries more than one LOOP-STOP-shaped line (e.g.
   a quoted example), `loop_stall_guard` counts only the last one, so the last line must be the
   declaration that reflects the turn's actual outcome." The same clarification is mirrored into the
   Stop-conditions section's "Declaring the stop" paragraph. **This is a clarification of an existing
   contract, not a new rule** — `loop_stall_guard`'s tail-1 read of the declaration only determines
   *which* category among multiple matching lines gets counted; it does not reject a declaration
   placed earlier in the turn as invalid. The skill text is what defines the ending-line placement as
   the expected authoring contract; the hook's tail-1 behaviour is the reason that contract matters
   (a misplaced declaration risks the wrong line being counted, not an outright rejection).
   (verified: diff, both Phase 0.5 and Stop-conditions sections)

3. **Past-failure figure corrected.** The pre-#157 text cited "~8 confidence/verify blocks in one
   run" as the motivating incident. The real figure — loop f87a0a2e's ~69 turn regenerations (49
   confidence-label + 20 verify-loop) — replaces it, with the incident now explicitly named as "the
   incident that motivated demoting these two hooks to warn-level inside an active loop." (verified:
   diff, Phase 0.5 closing paragraph)

## PR #158 — envelope-anchored eval authoring

| Field | Value |
|---|---|
| PR number | #158 |
| Branch | `docs/envelope-anchored-eval-authoring` |
| Merged | 2026-07-13 |
| Merge SHA | `6eb8afc670b520193f7f96eb10f1282d35528d8d` |
| JIRA ticket | — |
| Files changed | `skills/agentic-loop/SKILL.md` (5 lines), `skills/task-evals/SKILL.md` (1 line) |

### Summary

Extends [[task-evals]]'s rule 4 (oracle independence) with a **precedence rule** for loop scope: the
eval author's goal-state anchor is `progress.json`'s `authorising_prompt_raw` field — the post-Phase-0
envelope, exactly one canonical string, with no judgement call about which version of the prompt
counts as authoritative. `spec.md` does restate the loop's success criteria at Phase 2.7a, and
`plan.md` restates it per-task, but this is explicitly a precedence rule, **not a content denial**:
`spec.md`/`plan.md` supply constraints and concrete assertable surfaces useful for writing evals, and
their restated criteria never override the envelope's goal state as the anchor. `progress.json`'s
field is the canonical source; `spec.md`'s Phase-2.7a copy is a derived restatement, not an
independent authority. (verified: `skills/task-evals/SKILL.md` rule 4, diff `4fbb5fe..6eb8afc`)

### Why this closes a real gap

Before this PR, an eval author working at loop scope had no stated rule for which of several
plausible-looking "goal state" sources to derive evals from — the original user prompt (verbatim, but
possibly superseded by Phase -1's improve-prompt step), `spec.md`'s Phase 2.7a restatement, or
`plan.md`'s per-task restatement. A restatement drifting from the original intent (even
unintentionally, through normal plan-writing paraphrase) could silently become the oracle the evals
were derived from — which is exactly the kind of implementation-adjacent oracle rule 4 already
warns against for the non-loop case ("derive evals from the task's goal state, not its implementation
steps"). Anchoring on one specific field closes the ambiguity structurally rather than relying on the
eval author's judgement call every time.

### Wiring into the agentic-loop skill (four touch points)

1. **Phase -2 stub schema comment.** `authorising_prompt_raw`'s inline comment now notes: "Phase -1
   updates this if an improved prompt is adopted" — flagging that the field is not fixed at stub time.
2. **Phase -2 mid-loop re-stub rule.** The existing rule that `loop_stop_counts` carries forward
   verbatim on a mid-loop re-stub (recovery after a restart) is extended: `authorising_prompt_raw`
   must also carry forward verbatim. "A re-stub refilled from conversation memory instead of the
   prior file's value would silently drift the eval author's canonical anchor" — the same
   memory-vs-durable-record concern the `decisions_absorbed` array and `loop_stop_counts` carry-forward
   rules already apply elsewhere in this skill.
3. **Phase -1 (sharpen the authorising prompt).** On adopting an improved envelope (outcome A —
   improved prompt adopted, or B — user tweak applied), the orchestrator must now update
   `progress.json.authorising_prompt_raw` to the adopted text, keeping the field the canonical
   post-Phase-0 envelope. Outcome C (proceed with the original prompt unchanged) needs no update —
   the Phase -2 stub already wrote the original verbatim.
4. **Phase 2.7c cross-reference.** The sub-step that freezes loop-scope `evals.json` now states
   explicitly: "The eval author anchors goal state on `progress.json`'s `authorising_prompt_raw`, per
   `task-evals`'s oracle-independence rule" — pointing the loop-scope freeze step at the anchor rule
   rather than leaving it implicit.
5. **Phase 13 teardown enrichment.** The `retro.json` assembly step's `envelope` field, previously
   described as "verbatim from `progress.json`", is now specified as "verbatim from `progress.json`'s
   `authorising_prompt_raw`" — naming the exact field rather than the file generically.

(verified: diff `4fbb5fe..6eb8afc`, `skills/agentic-loop/SKILL.md` Phase -2/-1/2.7c/13 sections)

### Enrich-at-Phase-0 wording aligned

The pre-existing "Lifecycle, enforced by the `loop_state_guard` Stop hook" section's "Enrich at
Phase 0" bullet — "record the envelope verbatim" — is now "record the envelope verbatim in
`authorising_prompt_raw`", naming the field consistently with the rest of the changes above.

## Deploy and verification

Installed plugin cache refreshed to version **1.1.5** at commit `1ac2166` — version lockstep with
`.claude-plugin/marketplace.json` was repaired by a concurrent session's telemetry PR (#159); this
loop's own PRs (#155/#157/#158) did not touch the version field. Live-verified post-deploy: the
installed `check_confidence_labels.sh`/`check_verify_loop.sh` demote correctly on a Stop probe inside
an active loop, and `SubagentStop` still blocks unconditionally.

## Loop evals — GO

The tier-2 frozen loop-scope `evals.json` suite was graded GO by `post_evals.sh grade-loop`. One
recorded amendment: eval L7's fresh-clone surface had been specified assuming a `git archive` export
(no `.git` directory), which meant two git-dependent suite files could not run against it as written.
The eval was reimplemented as a true `git clone` instead, and the reimplementation was proven green
independently before the amendment was folded back into the frozen suite (rule-5 grader-independence
discipline: an eval amended after a verdict returns to a fresh grader, never a self-flip).

### Design decisions and accepted residuals

- **Anomaly-line surfacing was explicitly skipped** (YAGNI, owner decision) — not deferred as a gap,
  a deliberate scope cut for this loop.
- **Cosplay-loop escape, accepted residual.** A loop that never reaches a genuine `complete`
  declaration holds its discipline hooks at warn-level indefinitely — in principle, a session could
  stay "in an active loop" to keep discipline at warn without ever finishing real work. This is
  contained, not closed: `SubagentStop` (worker output) still blocks unconditionally regardless of
  loop state, and the demoted warns are still loudly logged (`would_block=1 warned=1 blocked=0`) for
  retro mining — a cosplay loop's log signature (blocks that never happen, warns that never resolve)
  is visible to a human or a future retro pass even though no hook currently detects the pattern
  automatically.
- **Bundle-line compliance is measured, not guaranteed.** PR #157's FINAL-line contract for the
  `LOOP-STOP` declaration (bundled with confidence labels + DNV content) is a documented authoring
  contract, not a hook-enforced one beyond `loop_stall_guard`'s existing tail-1 category read. A
  turn that fails to bundle all three is not separately caught — the discipline log's presence/
  absence of labels and DNV tags on loop-scoped turns is the retro-mining signal for this, same as
  the warn-vs-block metric above.

## Related

- [[pr_156_dnv-presence-check]] — concurrent session's PR (#156), the DNV-presence check wired
  through the same `als_loop_active_incomplete` predicate PR #155 introduced
- [[discipline-loop]] — the composed hook chain both #155 and #156 extend
- [[check_confidence_labels]] — hook detail page, already reflects the #155 demotion
- [[check_verify_loop]] — hook detail page, already reflects the #155/#156 demotion
- [[agentic-loop]] — the skill PR #157 and #158 both edit
- [[task-evals]] / [[task-evals-gate]] — the skill and design page rule 4 (oracle independence) now
  extends
- [[pr_144-149_agentic-loop-hardening-from-loop-engineering]] — the prior loop-engineering cluster
  this session's motivating incident (loop f87a0a2e) and this session's own mechanism build on
- [[pr_159_retire-catchup-add-telemetry]] — concurrent session's PR (#159): version-lockstep fix that
  brought the installed cache to 1.1.5, catchup-hook retirement, telemetry
