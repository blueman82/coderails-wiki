---
title: "PR 84 (2026-07-08) — closes git-global-option force-push bypass in destructive_bash_gate"
type: source
created: 2026-07-08
last_updated: 2026-07-08
sources: []
tags: [hook, destructive-bash-gate, security, source]
---

# PR #84 (2026-07-08) — git-global-option force-push bypass

No prior PR #84 exists in this vault — no collision to disambiguate.

## PR metadata

| Field | Value |
|---|---|
| PR number | #84 |
| Merge commit | `7c4f12f02d67b29c3dab4049452ad0ac7c21a8c8` |
| Merged | 2026-07-08T08:30:02Z |
| Title | "Close git-global-option force-push bypass in destructive_bash_gate" |

## Summary

The [[destructive_bash_gate]] force-push trigger required a *contiguous* `git`+`push` token pair (`\bgit +push\b`-shaped). A git global option placed between the two words — `git -c KEY=VALUE push --force`, `git --no-pager push --force`, `git -C path push --force` — broke that adjacency, so the naked-force detector never even ran against the rest of the line: a naked force push with **no allowlist in play at all** sailed through as ALLOW.

**Fix.** Introduced an option-tolerant `git_push_re` fragment: `git`, followed by zero or more bounded git global-option tokens, then `push`.

```bash
git_opt_tok='(-[cC][[:space:]]+[^[:space:]]+|--[a-zA-Z][a-zA-Z-]*(=[^[:space:]]*)?|-[a-zA-Z]+)'
git_push_re="\\bgit\\b([[:space:]]+${git_opt_tok}){0,20}[[:space:]]+push\\b"
```

`git_opt_tok` covers the three shapes git global options take: `-c`/`-C` with a separate-token argument, a long option with an optional attached `=value`, and any other short flag. Bounded to 20 repetitions — git itself has no limit on repeated `-c`, so any fixed bound is a residual gap in principle, but 20 chained global options is far beyond any real invocation.

Applied **symmetrically** to both the trigger check and the naked-force-exclusion check (the allowlist carve-out from [[pr_72-75_2026-07-08_force-with-lease-allowlist]]) — so the allowlisted `--force-with-lease` path stays reachable when a global option is present, not just the naked-force deny path.

## Known residual gap (documented, not fixed)

A `-c`/`-C` value containing a quoted space (e.g. `-c "user.name=John Doe"`) is not matched by the single-token value arm — same class as the file's documented "quoted paths with spaces... remain uncaught" ceiling elsewhere (`AGENTS.md`), not something a line-oriented ERE can fix without quote-aware tokenising. Confirmed identical on pre-fix `main` (narrowed, not introduced, by this PR).

## Files changed

- `hooks/scripts/destructive_bash_gate.sh` — `git_opt_tok`, `git_push_re`, applied to both the trigger and the fwl-exclusion grep.
- `hooks/scripts/tests/destructive_bash_gate.test.sh` — 12 regression tests (RED confirmed: 6 fail against the pre-fix script when the fix is stashed).

## Wiki pages updated

- [[destructive_bash_gate]] — force-push trigger section now documents `git_push_re`'s option-tolerance.

## Caveats / gotchas

Test plan included an **independent standalone sweep** — a separate probe script, run in a non-repo temp dir, with payload phrasing distinct from the committed tests (12/12 pass, including the allowlist-present-ALLOW / allowlist-removed-DENY flip on the same payload) — plus negative controls confirming unrelated git subcommands with a global option (`git -c color.ui=always status`, `git --no-pager log -1`) still ALLOW, i.e. the widened trigger doesn't over-match ordinary commands. Full suite: 156/156 (144 existing + 12 new).

Third PR in the five-PR arc. Each of the three substitution/force-push fixes (#69, #72+#75, #84) found an *independent* bypass shape in the same gate — the pattern that motivates [[destructive_bash_gate]]'s cross-cutting note: narrowing a line-oriented ERE gate is a game of whack-a-mole, and every narrowing needs a fresh adversarial pass plus a positive-control test.
