---
title: "PR #76 — harden hook stdin read"
type: source
created: 2026-06-29
last_updated: 2026-06-29
sources: []
tags: [hook, stdin, defence-in-depth, orphan-process, resilience, bash]
---

# PR #76 — harden hook stdin read

<!-- Ingested by /wiki-ingest after merge. This is an immutable record of what changed. -->

## PR metadata

| Field | Value |
|---|---|
| PR number | #76 |
| Branch | `bug/harden-hook-stdin-read` (inferred) |
| Merged | 2026-06-29 |
| Merge SHA | `37459a3` |
| JIRA ticket | — |

## Summary

Replaced `input=$(cat)` with `IFS= read -r -d '' -t 5 input || true` in all 10 hook scripts. Added `hooks/scripts/tests/stdin_bounded_read.test.sh` (guard + fidelity tests, built TDD). Updated AGENTS.md hook conventions to document the new read pattern as canonical.

This is **independent defence-in-depth** for a latent stdin-blocking risk, NOT a fix for the fan-incident that triggered the investigation. The fan incident (21 orphaned `enforce_pr_workflow.sh` processes at 8-10% CPU for 3+ hours) was caused by an infinite loop in `scripts/lib/config.sh`'s walk-up, already fixed in commit `04b26c9` (PR #72). PR #76 closes a separate, latent risk in the stdin read path.

## The latent risk

`input=$(cat)` reads stdin until EOF. Normally Claude Code closes stdin when it invokes a hook, so EOF arrives promptly. But if the **parent process dies** (crash, OOM, SIGKILL) without closing stdin, the hook becomes an orphan — reparented to PID 1, where the `hooks.json` `timeout` field can no longer kill it. In that state, `input=$(cat)` blocks forever: the hook leaks a process per failed turn and can spin indefinitely on its own internal logic. (verified: PR #76 body)

## The fix

```bash
IFS= read -r -d '' -t 5 input || true
```

Each part of this idiom is load-bearing (verified: PR #76 body, AGENTS.md hook conventions):

| Component | Why it matters |
|---|---|
| `IFS=` | Prevents leading/trailing whitespace stripping |
| `-r` | No backslash escaping |
| `-d ''` | Read until NUL-byte or EOF, preserving full multi-line JSON payloads (Stop-hook `last_assistant_message` carries multi-line content) |
| `-t 5` | Times out after 5 seconds if no EOF arrives — the in-process backstop |
| `\|\| true` | Mandatory — `read -d ''` returns exit code 1 on **normal** EOF; without `\|\| true` the script aborts under `set -e` |

### macOS and timeout portability

- macOS bash 3.2 rejects fractional `-t` values — the timeout must be an integer.
- `timeout`/`gtimeout` are absent on stock macOS — using them would silently fail-open every deny-first hook (exit 127 treated as allow), making them unsuitable.

## The fail-open design: why it is safe

When `-t 5` fires (no EOF within 5 seconds), `input` is empty → jq yields empty → the gate stands aside (exit 0 = ALLOW). This is deliberate:

- The timeout fires **only when the parent process is dead**.
- If the parent is dead, there is no live tool-execution pipeline to honour a deny.
- Fail-open coincides exactly with "nothing to gate" — a dead-parent hook cannot meaningfully enforce anything.

This fail-open-on-stall is the correct posture for a PreToolUse enforcement hook. Do NOT add `set -e` or flip the empty-input branch to a deny. (verified: AGENTS.md hook conventions)

## The 5-second timeout: invariant

The 5-second timeout is deliberately `<=` the smallest `hooks.json` `timeout` value (also 5 seconds). This ensures the in-process backstop and the harness timeout never disagree — the process backstop always fires first or simultaneously. (verified: PR #76 body)

**Invariant guard:** A guard test asserting `min(hooks.json timeout) >= 5` was identified as a known gap by the PR #76 review (comment-analyzer S1, silent-failure-hunter I2). **This gap was closed by [[pr_78_hooks-json-timeout-floor]] (PR #78, merged 2026-06-29)**, which added `hooks/scripts/tests/hooks_json_timeout_floor.test.sh`. The invariant is now machine-checked on every test run.

## Security review

A 6-agent review (code-reviewer, comment-analyzer, security, code-simplifier, silent-failure-hunter, pr-test-analyzer) confirmed the change is safe. Security verdict: **CLEAN 9/10**. (verified: PR #76 body)

## Files changed

| File | Change |
|---|---|
| `AGENTS.md` | Hook conventions updated to document the `read -r -d '' -t 5` pattern |
| `hooks/scripts/check_confidence_labels.sh` | `input=$(cat)` → `IFS= read -r -d '' -t 5 input \|\| true` |
| `hooks/scripts/check_verify_loop.sh` | Same |
| `hooks/scripts/destructive_bash_gate.sh` | Same |
| `hooks/scripts/discipline_catchup.sh` | Same |
| `hooks/scripts/enforce_pr_workflow.sh` | Same |
| `hooks/scripts/inject_context.sh` | Same |
| `hooks/scripts/loop_stall_guard.sh` | Same |
| `hooks/scripts/loop_state_guard.sh` | Same |
| `hooks/scripts/no_edit_on_main.sh` | Same |
| `hooks/scripts/test_gate.sh` | Same |
| `hooks/scripts/tests/stdin_bounded_read.test.sh` | New — guard + fidelity tests (TDD: 2 guard FAILs, 4 fidelity OKs pre-fix; 6/6 PASS after; full suite 17/17) |

## Test approach (TDD)

**Guard test**: holds the write-end of a named pipe open in a background process (no EOF ever sent), runs `test_gate.sh` and `check_confidence_labels.sh` against the read-end, asserts each exits within 8 seconds (5s timeout + 3s slack). Uses background-process + bounded-poll — no `timeout` dependency.

**Fidelity test**: pipes a normal multi-line JSON payload and confirms allow/deny decisions are unchanged, proving `read -d ''` preserves the full payload.

**RED state** (against old `input=$(cat)`): 2 guard FAILs (hooks hung), 4 fidelity OKs.
**GREEN state** (after fix): 6/6 PASS. Full suite: 17/17 suites pass. (verified: PR #76 body)

## Context: fan incident vs. PR #76

The fan incident that motivated the investigation:
- **21 orphaned `enforce_pr_workflow.sh` processes** spinning at 8–10% CPU for 3+ hours, driving fans.
- **Root cause**: infinite loop in `scripts/lib/config.sh::coderails::config_path()` on the NO_CONFIG path (walk-up did not terminate on symlink-resolved paths). Already fixed in commit `04b26c9` (PR #72). See [[pr_72_config-walkup-symlink-hang]].
- PR #76 is independent: it addresses a separate path (stdin block on orphaned hook), not the config walk-up loop.

Do NOT conflate the fan incident root cause with PR #76. They share the surface symptom (orphaned hook processes) but are mechanically distinct.

## Wiki pages updated

- [[check_confidence_labels]] — stdin read convention note added
- [[check_verify_loop]] — stdin read convention note added
- [[test_gate]] — stdin read convention note added
- [[destructive_bash_gate]] — stdin read convention note added
- [[discipline_catchup]] — stdin read convention note added
- [[no_edit_on_main]] — stdin read convention note added
- [[enforce_pr_workflow]] — stdin read convention note added
- [[inject_context]] — stdin read convention note added
- [[loop_state_guard]] — stdin read convention note added
- [[loop_stall_guard]] — stdin read convention note added
- [[discipline-loop]] — convention reference added

## Caveats / gotchas

- The `|| true` is mandatory, not a style choice. `read -d ''` exits 1 on normal (non-timeout) EOF. Without it, a script running with `set -e` would abort on every successful read.
- The 5-second bound is an in-process backstop, not the primary defence. The primary defence against an orphaned hook causing harm is that a dead parent means there's nothing to gate.
- **Invariant guard closed (PR #78)**: the known gap (no automated `min(hooks.json timeout) >= 5` guard) was closed by [[pr_78_hooks-json-timeout-floor]] (PR #78). The guard lives in a dedicated test file (`hooks_json_timeout_floor.test.sh`), not in `stdin_bounded_read.test.sh`, as it tests the hooks.json configuration rather than the read idiom itself.
