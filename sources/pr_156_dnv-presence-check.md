---
title: "PR 156 — DNV-presence check closes omission-beats-honesty inversion"
type: source
created: 2026-07-13
last_updated: 2026-07-13
sources: []
tags: [hook, check_verify_loop, did-not-verify, discipline, turn-scoped, jq-hardening]
---

# PR 156 — DNV-presence check closes omission-beats-honesty inversion

## PR metadata

| Field | Value |
|---|---|
| PR number | #156 |
| Branch | `hooks/dnv-presence-check` |
| Merged | 2026-07-13 |
| Merge SHA | `4fbb5fe978a0697b288a03a9cce22d8328486f19` |
| JIRA ticket | — |

## Summary

`check_verify_loop.sh` gains a presence check: on the `Stop` path, if the current
TURN edited `>= 3` files and the response has no `## Did Not Verify` header at all,
the hook blocks (`exit 2`). This closes an inversion the prior version had —
a response that omitted the DNV section entirely passed silently, while an honest
section carrying one untagged bullet blocked. Omission was cheaper than honesty.
(verified: `hooks/scripts/check_verify_loop.sh:46-56`, header comment added by this PR)

The presence block is keyed on the **header's presence**, not on zero bullets — a
compliant prose-only section ("nothing outstanding") has the header but no
bullets and still passes, the same honesty boundary the pre-existing bullet-tagging
check already used. (verified: check_verify_loop.sh:47-52)

Two supporting changes shipped in the same PR:

1. **`file_count` re-scoped session-cumulative → TURN-scoped.** `dc_file_count()`
   in `hooks/scripts/lib/discipline_common.sh` now finds the last "genuine user"
   record (a `user`-type record whose `message.content` is a non-empty string, or
   an array containing a text block — a tool-result-only array does not count) and
   only counts `Write`/`Edit`/`MultiEdit` targets after that cutoff. This honours
   the CLAUDE.md self-checking-discipline wording, which is per-response ("after
   any response that edits files"), not per-session. Falls back to counting the
   whole transcript if no genuine user record exists (covers test fixtures).
   (verified: discipline_common.sh:11-24)
2. **`dc_file_count` hardened with the same per-line tolerant jq parse** the text
   extractors already used (`jq -R 'fromjson? // empty' | jq -s ...` — stage 1
   drops a malformed/truncated line, stage 2 aggregates over what's left). Before
   this PR, `dc_file_count` used a bare `jq -s` slurp, meaning a single malformed
   JSONL line anywhere in the transcript would zero the file count for the entire
   turn (fail-closed in the wrong direction — it would make a real multi-file edit
   look like zero files and silently skip the presence check). This closes the
   jq-slurp fragility family for the last of the three known instances (the other
   two — `dc_extract_last_text` and `unregistered_loop_guard`'s dispatch-turn
   count — were fixed earlier; see [[jq_slurp_round2_handoff]] equivalent memory).
   (verified: discipline_common.sh:20-24, 26)

## Loop-scope wiring

The presence block is wired through the SAME `als_loop_active_incomplete` demotion
predicate PR #155 added for the bullet-tagging path (that PR is a **different,
concurrent session's work** — referenced here, not duplicated): on a `Stop` event
inside an active, incomplete agentic loop, a presence violation demotes from a
hard block to a model-visible `additionalContext` warn
(`[discipline-warn(loop)] ...`) instead of `exit 2`. `SubagentStop` never
demotes — worker output stays fully block-enforced. The presence check cannot
fire on `SubagentStop` at all regardless: `file_count` is never computed on that
path (it's always `0`), so the `>= 3` gate never trips. (verified:
check_verify_loop.sh:150-161, header comment lines 46-56)

The PR's own reasoning for wiring through the existing demotion predicate rather
than adding an in-loop hard block: after #155's bullet-path demotion merged, a
hard presence-block in-loop would have been the only remaining in-loop hard block
in the file, inconsistent with the merged demotion arc. (verified: header comment
lines 52-56)

## Files changed

- `hooks/scripts/check_verify_loop.sh` — presence-check block, header comment rewrite
- `hooks/scripts/lib/discipline_common.sh` — `dc_file_count` turn-scoping + jq hardening
- `hooks/scripts/tests/check_verify_loop.test.sh` — new tests
- `hooks/scripts/tests/discipline_common.test.sh` — new tests

## Wiki pages updated

- [[check_verify_loop]] — presence check, turn-scoped file_count, loop demotion wiring
- [[discipline-loop]] — Stop hook composition and jq-slurp-family closure note

## Caveats / gotchas

- The presence check and the bullet-tagging check are genuinely different code
  paths sharing one predicate (`als_loop_active_incomplete`) for the loop
  demotion decision — don't conflate "wired through the same demotion" with
  "the same check."
- Empty/prose-only DNV section under a present header is compliant — the same
  honesty boundary as an all-tagged bullet list. Do not read the presence check
  as requiring non-empty content.
- `file_count` turn-scoping changes what "3 files" means: a prior turn's edits in
  the same session no longer count toward a later turn's presence gate. This is a
  deliberate narrowing (matches CLAUDE.md's per-response wording), not a
  regression — a session that edited 5 files across turns 1-2 and then has a
  pure-conversation turn 3 will not be nagged for a missing DNV section on turn 3.
