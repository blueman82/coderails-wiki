---
title: "Session: verify-loop hardening + cache-trap discovery"
type: source
origin: direct-edit session (no PR)
created: 2026-05-31
last_updated: 2026-05-31
sources: []
tags: [source, session, hook, check_verify_loop, install-cache, exit-codes]
---

# Session 2026-05-31 — verify-loop hardening

A direct-edit session (no PR) that reworked `check_verify_loop.sh`, surfaced the
install/cache three-copy trap, and established the per-event exit-2 blocking
semantics. Recorded as a session source because the changes were committed
directly to the working tree, not through a reviewed PR.

## Changes to `check_verify_loop.sh`

1. **Deleted the weak "did you act?" check.** The hook formerly asked whether any
   Read/Grep/Bash tool call appeared after the DNV section. It carried a jq
   operator-precedence bug (`select(.type=="assistant" and …) | test(…)` piped a
   boolean into `test`, throwing on any non-text transcript entry; the error was
   swallowed by `2>/dev/null` and coerced to 0 → block). Both the weak check and
   its buggy jq were removed. (verified: confirmed against the script this session)
2. **Source-token detector is now the sole block trigger.** A DNV bullet naming a
   `file.ext` or `file:line` blocks; anything else passes. (verified)
3. **String-coercion extraction.** Each assistant entry is reduced to a joined
   text string before list-building, so a trailing non-text entry can no longer
   shadow a real DNV block (`last` can't pick a number/null). (verified)
4. **File-edit threshold lowered `< 3` → `< 1`.** Single-file sessions are now in
   scope; the old threshold is why the hook never fired during a 2-file session.
   (verified: check_verify_loop.sh)
5. **Renamed cryptic `Px` labels to plain gate comments.** No behaviour change;
   comments only. (verified)
6. **Meta-bullet exclusion added.** A DNV bullet whose *leading clause* is a
   scope/completion statement ("nothing outstanding", "scoped out", "covered
   above", "did not enumerate") no longer blocks even when it names a file. The
   marker is anchored to `^-` so meta-words appended mid-sentence can't dodge a
   genuine claim. (verified: logic-replication test + live cache confirms)

## Why the meta-exclusion was needed

Token-matching can't tell "I did not check `prep.md:96`" (a deferred claim, should
block) from "covered above against `prep.md:96`" (a scope note, should pass). Three
false-positive blocks occurred this session before the exclusion. The fix narrows
the false-positive surface without weakening enforcement of genuine claims — the
anti-dodge anchoring is the load-bearing detail. See [[check_verify_loop]].

## Cross-cutting discoveries

- **Install/cache three-copy trap** — see [[install-and-cache-trap]] and
  [[install-cache-trap_2026-05-30]]. Editing the repo never changes the running
  hook until a `/plugin` reinstall repopulates the cache; `/reload-plugins` alone
  does not re-sync.
- **Per-event exit-2 blocking semantics** — see [[hook-exit-codes]]. Established
  which hook events block on `exit 2` and why coderails uses two block mechanisms.

## Impact

`check_verify_loop.sh` rewritten; [[check_verify_loop]] updated; [[hook-exit-codes]]
created. No command, skill, or install-script changes.
