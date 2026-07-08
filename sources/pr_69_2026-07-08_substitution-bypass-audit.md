---
title: "PR 69 (2026-07-08) — closes process-substitution and multi-line substitution bypasses in destructive_bash_gate"
type: source
created: 2026-07-08
last_updated: 2026-07-08
sources: []
tags: [hook, destructive-bash-gate, security, source]
---

# PR #69 (2026-07-08) — substitution-bypass audit

> ⚠️ **PR-number collision, resolved by date-disambiguation.** An OLDER, unrelated PR #69 (merged 2026-06-29, a `no_edit_on_main` deny-message reword) already exists in this vault as [[pr_69_2026-06-29_no-edit-message-worktree]]. This page documents a DIFFERENT PR #69 — a repo re-numbering (or a distinct fork) reused the number for a 2026-07-08 security-hardening change. Do not conflate the two; each is filed under its own date-disambiguated slug.

## PR metadata

| Field | Value |
|---|---|
| PR number | #69 (2026-07-08 — see collision note above) |
| Merge commit | `ebfee7b937872cde0034babdbb1fb2b81266398a` |
| Merged | 2026-07-07T19:03:36Z |
| Title | "security/wu1 subst audit" |

## Summary

Closed two live command-substitution bypasses in [[destructive_bash_gate]]'s workflow-script substitution check (the block that denies a backtick/`$(...)` inside a `push.sh`/`merge.sh`/`post_review.sh`/`post_evals.sh` free-text argument):

1. **Process substitution.** `<(...)` and `>(...)` execute their body eagerly, exactly like `$(...)`, but contain neither a backtick nor a literal `$(` — invisible to a detector that only checked for those two. Confirmed bypass: `bash scripts/push.sh "note" <(touch pwned)` ran with zero backticks or `$(` anywhere on the line. Fix: widened `subst_re` to `` `|\$\(|<\(|>\( `` at every site that scans for substitution characters.
2. **Multi-line substitution.** A script mention on one physical line and its own live substitution on a *different* physical line — via a heredoc with an unquoted delimiter (still expands `$(...)` inside it) or ordinary backslash line-continuation (which bash joins into one logical command before executing) — let the `before_script`/`from_script` scoping logic silently miss the substitution, because `sed`'s and `grep`'s `.` never cross a newline. Fix: `cmd_flat=$(echo "$cmd" | tr '\n' ' ')` flattens embedded newlines to spaces before any scoping logic runs, so the whole logical command is visible to the single-line regex the way bash would see it.

**Conservative tradeoff, documented in the code:** flattening with a bare `tr '\n' ' '` (rather than the backslash-newline-pair-aware `awk` approach PR #75 later used for the force-push flattener) means a *quoted-delimiter* heredoc (`<<'EOF'` — normally substitution-inert) is now also denied if it contains substitution-looking characters, because flattening can't distinguish quoted from unquoted heredoc bodies. Accepted as an over-block, not a false-negative — the gate's failure direction stays fail-closed.

## Files changed

- `hooks/scripts/destructive_bash_gate.sh` — widened `subst_re`; added `cmd_flat` newline-flattening before the substitution-scoping block (see [[destructive_bash_gate]] "Command-substitution detection" for the current logic).
- `hooks/scripts/tests/destructive_bash_gate.test.sh` — regression tests for both bypass shapes.

## Wiki pages updated

- [[destructive_bash_gate]] — substitution-detection section updated to describe `<(`/`>(` coverage and `cmd_flat` newline-flattening.

## Caveats / gotchas

This is the first PR in a five-PR adversarial-hardening arc on the same gate (#69, #72+#75, #84, #92 — all 2026-07-08). Each fix narrowed a regex; each narrowing risked reopening an adjacent hole, which is exactly what happened next (see [[pr_72-75_2026-07-08_force-with-lease-allowlist]] and [[pr_84_2026-07-08_git-global-option-bypass]]). See [[destructive_bash_gate]] for the cross-cutting theme note on adversarial regex narrowing + positive-control testing.
