---
title: "PR #50 (2026-07-07) — comment_citation_gate.sh"
type: source
created: 2026-07-08
last_updated: 2026-07-08
sources: []
tags: [hook, pretooluse, comment-hygiene, citation]
---

# PR #50 (2026-07-07) — comment citations / WU2 gate

**⚠️ PR-number collision:** this PR is unrelated to the older
`sources/pr_50_planning-sequence-gate.md` (merged 2026-06-26, pre the
2026-07-05 repo recreation described in [[repo-hosting]]). This page uses a
date-qualified slug to disambiguate.

## PR metadata

| Field | Value |
|---|---|
| PR number | #50 |
| Title | comment citations/wu2 gate |
| Merge SHA | `6cdeb9c8d361f823b714d9c4f85daf9363bb66c8` |
| Merged | 2026-07-07T12:22:08Z |

## Summary

Adds `hooks/scripts/comment_citation_gate.sh`, a new `PreToolUse`
(`Write`/`Edit`/`MultiEdit`) hook that denies new code comments citing a
session-artifact label (`E#:`, `F# fix`, `CHANGE B#/C#`, `Task A#`, `TA-I#`,
"reviewer finding", `eval E#`, `WU#:`, `C2`, "per the plan/design/session") —
labels that only resolve inside the conversation that produced them and rot
the instant the session ends. `.md` files are out of scope entirely; `PR #NN`
is a deliberate, documented survivor since it resolves to a durable, checkable
GitHub artifact. Registered in `hooks/hooks.json` alongside
[[no_edit_on_main]] under the same `PreToolUse (Write|Edit|MultiEdit)`
matcher. A follow-up commit in the same PR (`312e91d`) registered the new
script in the exec-bit invariant test manifest.

## Files changed

- `hooks/scripts/comment_citation_gate.sh` (new)
- `hooks/hooks.json` (registration)
- exec-bit invariant test manifest (registration entry for the new script)

## Wiki pages updated

- [[comment_citation_gate]] — new hook page (this PR's coverage gap, closed
  2026-07-08 during a loop-lib residuals ingest pass, not at the time of this
  PR's own merge)
- `index.md` — new hooks-table row

## Caveats / gotchas

- **This PR's own body is a bare commit-log dump on GitHub, not a written
  description** — `gh pr view 50 --json body` returns the three commit
  subjects verbatim rather than PR prose. This source page's Summary is
  reconstructed from the hook script's own header comment and its
  `hooks.json` registration, not from a `gh`-fetched description.
- **This PR was never wiki-ingested at merge time** — the gap sat undetected
  until a full-vault wiki-lint health pass on 2026-07-08 flagged it (13/14
  hook scripts had wiki pages; this was the missing one). See
  [[comment_citation_gate]]'s "Wiki coverage gap" section for the lint
  finding this page closes.
- `AGENTS.md`'s own Part-1 hook event map also omits this hook — a
  source-side gap flagged in [[comment_citation_gate]] but not fixed by
  this ingest (editing `AGENTS.md`'s hook map is the maintainer's call).

## See also

- [[comment_citation_gate]] — the hook page this PR's content informs
- [[no_edit_on_main]] — sibling gate on the same PreToolUse event
- [[repo-hosting]] — the PR-number collision convention this page follows
