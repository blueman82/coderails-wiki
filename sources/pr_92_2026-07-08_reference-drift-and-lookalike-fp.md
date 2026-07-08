---
title: "PR 92 (2026-07-08) — REFERENCE.md gate drift fix + plugin_src lookalike false-positive fix"
type: source
created: 2026-07-08
last_updated: 2026-07-08
sources: []
tags: [hook, destructive-bash-gate, docs, false-positive, source]
---

# PR #92 (2026-07-08) — REFERENCE.md drift + plugin_src lookalike FP

> ⚠️ **PR-number collision, resolved by date-disambiguation.** An older, unrelated PR #92 (an exec-bit sweep) already exists in this vault as [[pr_92_exec-bit-sweep]]. This page documents a DIFFERENT PR #92 — 2026-07-08 docs/security-hardening changes that reused the number. Do not conflate.

## PR metadata

| Field | Value |
|---|---|
| PR number | #92 (2026-07-08 — see collision note above) |
| Merge commit | `652c5f8ab7782c804e162577da3726711d1c510f` |
| Merged | 2026-07-08T08:42:27Z |
| Title | "docs: fix REFERENCE.md gate drift + plugin_src lookalike FP" |

## Summary

Two independent fixes, both closing out the same-day gate-hardening cluster (#69, #72, #75, #84 above).

### (a) REFERENCE.md matrix drift

`docs/REFERENCE.md`'s `destructive_bash_gate.sh` Hook Activation Matrix row and its notes-list caveat were stale after the four preceding PRs — they omitted two now-live capabilities: command/process-substitution detection (backtick, `$(...)`, `<(...)`/`>(...)`) inside `push.sh`/`merge.sh`/`post_review.sh`/`post_evals.sh` free-text arguments, and the force-with-lease allowlist carve-out. Updated both to match the shipped gate exactly, and made the "no approval path" note precise about the one narrow carve-out that exists (verified against `docs/REFERENCE.md`'s current Hook Activation Matrix table, which now states this in full — see [[destructive_bash_gate]]).

### (b) `plugin_src` lookalike false-positive (reproduced and fixed)

`destructive_bash_gate.sh`'s `plugin_src` matcher (the pattern that blocks in-Bash writes to `skills/*/SKILL.md`/`commands/*.md` on main — see [[destructive_bash_gate]]'s branch-aware source-edit gate) had no left-anchor. It matched the substring `skills/[^/]+/SKILL\.md` or `commands/[^/]+\.md` **anywhere on the line**, including glued inside an unrelated word. Reproduced concretely: `tee xcommands/prep.md` and `tee not-skills/x/SKILL.md` false-positive DENYed on main even though neither is a real plugin path — `xcommands/` merely *contains* the substring `commands/`, it is not a `commands/` path segment.

**Fix.** Anchored `plugin_src` to a path/token boundary:

```bash
plugin_src='(^|[[:space:]/'"'"'"])(skills/[^/]+/SKILL\.md|commands/[^/]+\.md)([ '"'"'"]|$)'
```

The left side now requires start-of-string, whitespace, a quote, or a preceding `/` immediately before `skills/`/`commands/` — so a glued lookalike word doesn't match, while a genuinely nested real path like `vendor/skills/x/SKILL.md` still matches (the `/` before `skills/` there is a real path separator, not part of a directory name, so no false-negative was introduced). Mirrors `src_ext`'s existing right-anchor in the same file.

## Files changed

- `docs/REFERENCE.md` — Hook Activation Matrix row + notes-list caveat for `destructive_bash_gate.sh`.
- `hooks/scripts/destructive_bash_gate.sh` — `plugin_src` left-anchor.
- `hooks/scripts/tests/destructive_bash_gate.test.sh` — 4 regression tests (lookalike ALLOW, nested-real-path DENY still holds).

## Wiki pages updated

- [[destructive_bash_gate]] — `plugin_src` anchoring, REFERENCE.md now current.

## Caveats / gotchas

Test plan: reproduced the FP with targeted probes in a non-repo scratch `git init` *before* touching code (confirms the finding independent of any fix), then re-verified probes flip to ALLOW for lookalikes and stay DENY for genuine + nested real paths after the fix. Full suite green: 161/161 (157 existing + 4 new), zero `not ok`.

Fifth and closing PR in the arc. Unlike #69/#72+75/#84 (which each closed a bypass — a false ALLOW), this PR's (b) fix closes the opposite failure mode — a false DENY — reinforcing that narrowing a line-oriented ERE gate carries risk in both directions, not just the security-bypass direction. See [[destructive_bash_gate]] for the full arc summary.
