---
title: "Hook: voice_announce.sh"
type: hook
created: 2026-07-07
last_updated: 2026-07-07
sources:
  - sources/pr_70-71_2026-07-07_dashboard-input-fix-and-voice-announcements.md
tags: [hook, agentic-loop, stop-hook, voice, observability, observe-only]
---

# voice_announce.sh

Stop hook that speaks a short macOS `say` announcement for an agentic loop's stopping-turn
outcome. Purely an observability/UX layer — it never blocks a Stop and cannot affect any
other hook's decision.

Source: `coderails/hooks/scripts/voice_announce.sh`
Shared lib: `coderails/hooks/scripts/lib/loop_state_common.sh` (`als_extract_last_text` /
`als_stable_last_text`, extracted from [[loop_stall_guard]] in the same PR)

## Event and mode

| Field | Value |
|---|---|
| Event | `Stop` |
| Mode | observe-only — **always exits 0** |
| Stop order | **1st** (before confidence/verify/loop_state_guard/loop_stall_guard/unregistered_loop_guard) |

## Why it runs first

Deliberate ordering, not incidental: this hook is observe-only and always exits 0, so
running first cannot affect any other Stop hook's block decision. Running it *after* a
hook that can exit 2 risks the runner short-circuiting before this hook ever fires — under
either parallel or sequential Stop-hook execution semantics, "first" is the only position
that guarantees the announcement always gets a chance to speak.

## The four announcement kinds

| Kind | Condition | Phrase |
|---|---|---|
| `complete` | `LOOP-STOP: complete` declared in the stopping turn | "Loop complete." |
| `waiting` | `LOOP-STOP: approval-gate` or `awaiting-input` declared | "Loop is waiting on you." |
| `stopped` | `LOOP-STOP: hard-stop` declared | "Loop has hit a hard stop." |
| `stall` | active + incomplete, text extracted successfully, but no valid `LOOP-STOP` line found in it | "Loop may have stalled." |

Uses the same `LOOP_STOP_VOCAB`-derived declaration regex as [[loop_stall_guard]] (C2), via
the shared lib — one vocabulary, two consumers (one blocks, one announces).

## Silence conditions (deliberately NOT a stall)

- No active agentic loop at all (no `progress.json`, no `agentic-loop` Skill invocation in
  the transcript) — the ordinary silent case.
- **Stable-text extraction itself comes back empty** — logged as `reason=extract_failed`,
  and explicitly *not* classified as `stall`. This is the review-caught fix: the first
  implementation conflated "nothing to read yet" (e.g. a transcript-flush race on a loop's
  very last, genuinely complete turn) with "read it and found no valid declaration," which
  would have spoken "stalled" for loops that had just completed cleanly.
- `say` not on `PATH` — logged distinctly (`reason=say_absent`), not claimed as a
  successful announcement.
- Debounce window still active for this session+kind (see below).

## Mechanics

- `speak()` launches `say` backgrounded and `disown`ed with all three streams redirected,
  so the hook returns in well under 1 second regardless of how long the actual speech
  takes — proven empirically against a naive 3-second-sleeping stub during review.
- **Per-session, per-kind debounce**: marker files live beside `progress.json` in the
  loop-state dir (never the repo), default window 60s, overridable via
  `CLAUDE_VOICE_DEBOUNCE_SECONDS`. A marker-write failure fails open **toward speaking**
  (never silently suppresses a real announcement) but logs `reason=debounce_write_failed`
  instead of claiming a clean debounce state.
- Reads its payload via the standard `IFS= read -r -d '' -t 5 input || true` convention
  (see [[pr_76_harden-hook-stdin-read]]).

## Shared-lib extraction (this PR)

`als_extract_last_text` / `als_stable_last_text` moved out of `loop_stall_guard.sh` into
`lib/loop_state_common.sh`, mirroring the earlier `discipline_common.sh` precedent used by
the confidence/verify-loop hooks. `loop_stall_guard.sh` was switched to call the shared
functions in the same PR — the byte-identical duplication between the two hooks is gone.

## Tamper-evidence counters

Adding this hook is a case the standing invariant tests are *designed* to catch if not
updated in lockstep — both were bumped correctly in the same PR:
`hooks_json_timeout_floor.test.sh`'s `EXPECTED_BACKSTOP_COUNT` 12→13, and
`exec_bit_invariant.test.sh`'s manifest gained `voice_announce.sh @ 100755`.

## Review findings (fixed pre-merge)

Two Critical: the `hard-stop` kind had no `case` arm at all (silently swallowed — no
announcement, no log line); and the empty-extraction-as-stall misclassification above.
Three Important: `say`-absent path previously logged as if it had announced successfully;
a failed debounce-write previously disabled debouncing silently instead of logging it; and
the first-in-Stop-array ordering constraint wasn't yet enforced/documented. 7/7 evals GO
including negative controls.

## Log output

Appends a `key=value` line to `$CLAUDE_DISCIPLINE_LOG`, e.g.
`hook=voice_announce session=<id> kind=<kind> announced=<0|1> reason=<...>` — distinct
`reason` values for `extract_failed`, `say_absent`, `debounce_write_failed`, and the
suppressed-by-debounce case.

## See also

- [[loop_stall_guard]] — shares the `LOOP_STOP_VOCAB` declaration regex and (as of this PR)
  the extraction helpers; C2 blocks on absence, this hook only announces
- [[loop_state_guard]] — C1; same active-loop detection primitives via the shared lib
- [[discipline-loop]] — full Stop hook composition and ordering
- [[hook-exit-codes]] — this hook is the one Stop hook that *always* returns the non-blocking code, by design
- [[pr_76_harden-hook-stdin-read]] — the stdin-read convention this hook follows
- [[pr_70-71_2026-07-07_dashboard-input-fix-and-voice-announcements]] — source record for this PR
