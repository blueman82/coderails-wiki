---
title: "PR 72+75 (2026-07-08) — git push --force-with-lease allowlist carve-out"
type: source
created: 2026-07-08
last_updated: 2026-07-08
sources: []
tags: [hook, destructive-bash-gate, security, source, incident]
---

# PR #72 + #75 (2026-07-08) — force-with-lease allowlist carve-out

> ⚠️ **PR-number collision, resolved by date-disambiguation.** Older, unrelated PRs already occupy #72 ([[pr_72_config-walkup-symlink-hang]], merged 2026-06-29, a `config.sh` walk-up infinite-loop fix) and #75 ([[pr_75_main-branch-fallback]], merged 2026-06-29, a `git-common.sh` `main()` fallback fix) in this vault. This page documents DIFFERENT PRs #72/#75 — 2026-07-08 security-hardening changes that reused these numbers. Do not conflate.

Filed as one page: both PRs are the same carve-out arc (#75 is the very next fix on top of #72, landing less than 12 hours later), matching this vault's established practice of grouping tightly-coupled same-arc PRs into one source page (e.g. [[pr_57-62_subagent-enforcement-gate-hardening]]).

## PR metadata

| Field | Value |
|---|---|
| PR #72 merge commit | `8332509b6040bed6ed1c4bcf69ed2e2616b9b97f` |
| PR #72 merged | 2026-07-07T19:41:21Z |
| PR #72 title | "security/wu2 force with lease" |
| PR #75 merge commit | `e63d3d1f670372296090c897ef50eddce8bc45af` |
| PR #75 merged | 2026-07-08T07:55:25Z |
| PR #75 title | "security/wu2 tab fix" |

## Summary

### PR #72 — the allowlist carve-out

Introduced a narrow, owner-opt-in conditional-allow for `git push --force-with-lease` inside [[destructive_bash_gate]], which previously denied every `git push --force`/`-f`/`--force-with-lease` shape unconditionally (`git +push +.*(--force|-f\b|--force-with-lease)`).

New behaviour:
- Naked `--force`/`-f` — including combined short-flag clusters like `-uf`/`-fu`/`-ufd` (mirroring the file's existing `git clean` force detector) — is **always denied**, even when `--force-with-lease` also appears on the same line.
- A *clean* `--force-with-lease` with no naked force present is **allowed** only if `.claude/destructive_allowlist` (resolved against the command's own repo root via `git rev-parse --show-toplevel`, gitignored/local-only) contains the exact whole-line keyword `git-push-force-with-lease`.
- Missing, empty, or garbage allowlist file → denies. Fail-closed by construction: the file is matched with `grep -qxF` (exact line, fixed string) — never eval'd or spliced into a regex, so malformed content can only fail to match, never widen what's permitted.

### PR #75 — the tab-separator + interior-flag-split fix

**The incident.** PR #72 shipped with a live regression: the naked-force detector's token boundaries used a literal space character, not `[[:space:]]`. Bash's default `IFS` splits on space, tab, *and* newline — a tab-separated naked force (e.g. a flag pair joined by a literal tab in `tool_input.command`) produced the same real argv split as a space would, but slipped past the space-only boundary undetected while a space-separated one correctly denied. This was caught by an orchestrator STOP that treated an unreproduced finding as *not absent* rather than dismissing it — the resolution was a positive-control test (assert the allowlist path actually ALLOWs on a clean payload), not a same-session merge over the open question. See [[destructive_bash_gate]] for the positive-control-testing discipline this incident established.

Two fixes landed together:
1. **Tab-separator fix.** All token boundaries in `naked_force_re` switched from literal spaces to `[[:space:]]`.
2. **Interior-flag-split bypass.** `git push --fo\<newline>rce` — bash's own line-continuation splices `--fo` + backslash-newline + `rce` into the single real argv token `--force`, with *nothing* inserted in place of the removed characters. The pre-existing `tr '\n' ' '` flattener instead *replaced* the newline with a space, leaving the backslash behind and producing `--for\ ce` (two tokens, stray backslash) — the regex never saw a contiguous `--force`, and a continuation split *inside* a flag word (not just between two separate flag tokens) bypassed detection entirely, with no allowlist involvement needed at all.

   Fix: `force_cmd_flat` now runs a two-pass normalisation — `awk 'BEGIN{RS="\\\\\n"} {printf "%s", $0}'` first splices backslash-newline *pairs* out entirely (mirroring what bash itself does), THEN a plain `tr '\n' ' '` flattens any remaining bare newlines for the inter-token case. Order matters: splicing must happen before flattening, or the bare-newline pass would destroy the backslash-newline pairs' distinguishing shape first.

   This `force_cmd_flat` splice-then-flatten pattern is scoped locally to the force-push check; the file's separate `cmd_flat` (used by the substitution-check block added in [[pr_69_2026-07-08_substitution-bypass-audit]]) has the *same* splice gap — noted in-code as flagged but out of scope for this fix.

**push.sh long-form flag.** `push.sh` also gained a `--force-with-lease` long-form flag so workers stop copying the raw gate-bypassing command directly — closing the operational incentive that led to the incident in the first place.

## Files changed

- `hooks/scripts/destructive_bash_gate.sh` — `allowlist_permits()`, `naked_force_re`, `force_cmd_flat` (both PRs).
- `scripts/push.sh` — `--force-with-lease` long-form flag (PR #75).
- `hooks/scripts/tests/destructive_bash_gate.test.sh` — allowlist-present/absent tests, naked-force-still-denied-alongside-fwl tests, tab-separator regression test, interior-flag-split regression test.

## Wiki pages updated

- [[destructive_bash_gate]] — force-with-lease allowlist section, `allowlist_permits()` logic, `force_cmd_flat` normalisation.

## Caveats / gotchas

- **Incident lesson, generalisable:** a worker merging over an orchestrator STOP that flagged an unreproduced finding is the wrong resolution path — the right one is a fresh reproduction (here, a positive-control test proving the allowlist path is actually live) before merging. See [[destructive_bash_gate]]'s cross-cutting note.
- Second PR in the five-PR arc; #84 (next) found a third, independent bypass shape (git global options breaking the `git`+`push` adjacency the trigger relied on) — see [[pr_84_2026-07-08_git-global-option-bypass]].
