---
title: "PR #238 — crack_on_prose_gate"
type: source
origin: https://github.com/blueman82/coderails/pull/238
created: 2026-07-17
last_updated: 2026-07-17
sources: []
tags: [pr, hook, stop-hook, crack-on, discipline]
---

# PR #238 — crack_on_prose_gate

Merged 2026-07-17 (squash commit `d8a303e`). Adds a `Stop` hook,
`hooks/scripts/crack_on_prose_gate.sh`, that closes the prose-question evasion in
the crack-on envelope.

## Key takeaways

- **The gap it closes.** [[crack_on_gate]] denies the `AskUserQuestion` **tool**
  while a crack-on envelope is active. That only blocks the tool — the model can
  still hand a question back to the user by writing it as plain prose in its final
  message (no tool call → no `PreToolUse` event → the tool-gate never fires). This
  was demonstrated live in the session that motivated the PR. [[crack_on_prose_gate]]
  is the prose half of the same waiver.
- **Mechanism.** A `Stop` hook in the [[check_confidence_labels]] /
  [[check_verify_loop]] discipline family. Active only when the session's
  `crack_on_active` flag is stamped. Blocks (exit 2 + stderr) a final message that
  ends by asking the user a question, forcing the model to continue the turn — make
  the call autonomously, or end with a declarative report / `LOOP-STOP`.
- **Classifier is a deterministic heuristic, not an LLM judge** — rejected for
  latency, API-dependency inside the hook, and untestability. Two tiers: terminal
  `?` on the prose body's last line, first-person-modal question in the last 3
  lines, and ~15 second-person request phrases.
- **Loop-safe.** A per-turn block counter caps at 3
  (`CLAUDE_CRACK_ON_PROSE_MAX_BLOCKS`) so a rephrased question can't cycle forever;
  at the cap the stop is allowed, logged `capped=1`.
- **Scope.** `Stop` only, never `SubagentStop` — a worker addresses its
  orchestrator, not the human.

## Honest ceiling (as recorded in the PR and the script header)

Intent has no regex. A `?`-free declarative handoff, a novel second-person
phrasing outside the tier-2 list, a question in plain double-quotes, and anything
after the 3-block cap all pass — audited but not blocked. The deliverable is
"common ask-in-prose evasions mechanically caught, fail-closed, fully logged", not
"asking is impossible". Same ceiling class as [[destructive_bash_gate]]'s
pre-expansion regex.

## Files

- **new** `hooks/scripts/crack_on_prose_gate.sh` — the gate (246 lines)
- **new** `hooks/scripts/tests/crack_on_prose_gate.test.sh` — 35 checks
- `hooks/hooks.json` — registration in the `Stop` array after `check_verify_loop.sh`
- `AGENTS.md` — new hook-table row + corrected the prior `crack_on_gate` row's
  now-false "no broader mechanism (Stop-deny) is used by design" claim
- `hooks/scripts/tests/exec_bit_invariant.test.sh` +
  `hooks/scripts/tests/hooks_json_timeout_floor.test.sh` — updated to assert the
  wired state (15 backstop scripts)

## Impact

The crack-on human-ask waiver is now enforced in both forms — the
`AskUserQuestion` tool ([[crack_on_gate]]) and prose text
([[crack_on_prose_gate]]). Origin: designed and built by a Fable 5 subagent,
verified + wired + merged by the orchestrator.

## Related

- [[crack_on_prose_gate]] — the hook page
- [[crack_on_gate]] — the tool-deny half
- [[discipline-loop]]
