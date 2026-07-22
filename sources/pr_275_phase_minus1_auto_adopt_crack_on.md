---
title: "PR 275 — Phase -1 auto-adopts improve-prompt under a full-autonomous envelope"
type: source
created: 2026-07-22
last_updated: 2026-07-22
sources:
  - skills/agentic-loop/SKILL.md
  - skills/agentic-loop/loop-state.md
tags: [agentic-loop, improve-prompt, crack-on, envelope, phase-minus-1, progress-json]
---

# PR 275 — Phase -1 auto-adopts improve-prompt under a full-autonomous envelope

<!-- Ingested by /wiki-ingest after merge. This is an immutable record of what changed. -->

## PR metadata

| Field | Value |
|---|---|
| PR number | #275 |
| Branch | `feat/crackon-auto-adopt-improve-prompt` |
| Merged | 2026-07-22 |
| Merge SHA | `b09c26a` |
| JIRA ticket | — |

## Summary

Under a "crack on" envelope the improve-prompt output was being lost entirely. This PR makes
[[agentic-loop]] Phase -1 keep running improve-prompt in a full-autonomous envelope, but
auto-adopt its output instead of asking.

The defect was never that the skill said to skip Phase -1. Phase -1 runs unless the prompt
explicitly opts out, and "crack on" is not an opt-out signal. The conflict was in Step 2, which
unconditionally fired an A/B/C `AskUserQuestion` (A = adopt improved, B = tweak, C = keep
original). A full-autonomous envelope forbids gates, so agents resolved the contradiction the only
way the text allowed — by dropping the whole phase, and the sharpened envelope with it.

Step 2 now branches on envelope class. Full-autonomous runs improve-prompt, writes the result to
`progress.json.authorising_prompt_raw`, appends the decision to `decisions_absorbed`, emits the
improved prompt as visible final text, and proceeds to Phase 0 on the next turn. Every other
envelope class keeps the A/B/C ask unchanged.

The governing principle: a full-autonomous envelope withdraws consent for the **gate**, not for
the **record**. Suppressing the approval is required; suppressing the improved prompt is not. The
improve-prompt output is worth more in autonomous operation, not less, because no human is
downstream to catch a vague envelope.

## Files changed

- `skills/agentic-loop/SKILL.md` — Phase -1 Step 2 rewritten to branch on envelope class; the
  Phase 0 handoff sentence extended to name the auto-adopted path; the `awaiting-input` category
  gloss corrected (it had cited the Phase -1 ask as an unconditional planned interaction point).
- `skills/agentic-loop/loop-state.md` — `decisions_absorbed` append-site list extended to include
  Phase -1, noting it appends only in a full-autonomous envelope.

## Wiki pages updated

- [[agentic-loop]] — phase table row -1, and the `authorising_prompt_raw` write rule
- [[improve-prompt]] — the skill whose output Phase -1 consumes

## Caveats / gotchas

**The step ordering is load-bearing, and the first draft got it backwards.** Delivery mechanism
(a) means *ending the turn* with the improved prompt as final text and no trailing tool call. The
original list said emit first, then write `progress.json`. Those writes are trailing tool calls,
which by the skill's own Delivery constraint make the emitted text invisible — so the list
defeated the exact guarantee the change exists to provide. The merged version does the writes
first and emits the prompt last, with Phase 0 beginning on the next turn. The turn break is a
rendering requirement, not an approval gate; the loop continues without input.

**Phase 2.5 is a near-mirror, not an exact one.** The auto-adopt shape follows Phase 2.5's
design-fork handling (auto-adopt, record, surface later, never stall), but Phase 2.5 writes only
to a file and carries no render-as-final-text constraint, so it completes inside one turn. Phase
-1 must also show the prompt to the user, which costs the turn break. The merged text states this
difference explicitly rather than claiming a clean mirror.

**`decisions_absorbed` phase values must stay strings.** The appended entry uses `{phase: "-1",
...}`. The dashboard's `readDecisions` filters on `typeof entry.phase === "string"`
(`skills/dashboard/app/src/lib/collect/sessions.ts`), so an unquoted numeric `-1` would be
silently skipped rather than erroring. 40/40 tests in `test/sessions.test.ts` pass with the
quoted form.

**Merged via `--admin`.** The `protect_main` ruleset requires a `tier-review` status check that
the tier-gate daemon posts only for tier-0 PRs — for anything else it posts nothing, leaving the
required check permanently pending. This PR had zero statuses on its head SHA and sat in that
deadlock, so it needed the owner-privileged merge path. This is the known required-status
deadlock, not a new failure.
