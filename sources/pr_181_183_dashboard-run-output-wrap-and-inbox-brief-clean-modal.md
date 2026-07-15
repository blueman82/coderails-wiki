---
title: "PR #181, #183 (coderails) + PR #27 (assistant-agent) — run-output modal wrap fix + clean inbox-brief modal output"
type: source
origin: pr
created: 2026-07-15
last_updated: 2026-07-15
sources:
  - skills/dashboard/app/src/styles/hud.css
  - skills/dashboard/app/test/hudMarkdownWrap.test.ts
  - examples/dashboard-config.json
  - tasks/inbox-brief.md (assistant-agent repo)
tags: [source, dashboard, run-output, inbox-brief, modal, rachel, telegram, cross-repo]
---

# PR #181, #183 (coderails) + PR #27 (assistant-agent) — run-output modal wrap fix + clean inbox-brief modal output

Two independent, same-day post-ship fixes to the [[dashboard]]'s run-output modal
([[RunOutputOverlay]], shipped by the `dashboard-rethink` loop's t10/t12 — see
[[dashboard-run-output-rendering-gap_2026-07-15]] for the investigation that found the
wiki hadn't caught up to that modal's existence yet). Grouped as one ingest because both
are "modal follow-ups," not because they share a mechanism.

## 1. PR #181 (coderails, merged @ ff99692) — long lines now wrap

`(verified)` Before this PR, `.hud-markdown pre` (the fenced-code-block style inside the
modal's rendered markdown) had only `overflow-x: auto` — a long unbroken line or wide
code block overflowed the modal horizontally instead of wrapping, per an owner report.
Fix: three CSS declarations added to that one rule in
`skills/dashboard/app/src/styles/hud.css`:

```css
white-space: pre-wrap;
overflow-wrap: anywhere;
word-break: break-word;
```

`overflow-x: auto` is kept as a harmless fallback for anything that still can't wrap.
`hudMarkdownWrap.test.ts` pins these declarations directly against the stylesheet source
(regex-matched, anchored so it can't match the sibling `.hud-markdown pre code { ... }`
rule) rather than asserting wrap behavior in a live DOM — the test file's own comment
notes jsdom has no layout engine (`scrollWidth`/`clientWidth` both read back `0` for an
overflowing element), so whether a long line visibly wraps in a real browser was checked
separately, not provable from this test alone.

## 2. PR #27 (assistant-agent, merged @ ef2783c) + PR #183 (coderails, merged @ 0e87e16) — clean inbox-brief modal output

**Problem.** Pressing the dashboard's INBOX BRIEF button showed Rachel's raw execution
trace in the run-output modal — tool calls, reasoning, intermediate text — not the actual
brief Gary's Telegram received. The two views had diverged even though both exist to
answer "what did the sweep find."

**Why a one-file fix couldn't reach it.** There are **two LLM layers** in this button's
call path (see [[dashboard]]'s "thin dispatcher" note on this button, PR #175-176): the
dashboard's `run` route spawns an outer `claude -p` agent (per the hardcoded
`spawnImpl("claude", ...)` boundary), whose prompt tells it to Bash-shell into
`bin/rachel "Read tasks/inbox-brief.md and follow it."`. The run-output modal renders the
**outer** agent's final `result` field — Rachel's own stdout is relayed text inside that
outer agent's transcript, not the modal's data source. Editing `tasks/inbox-brief.md`
alone (teaching Rachel to emit something cleaner) cannot change what the modal shows,
because the outer dispatcher agent still narrates/relays rather than becoming the brief
verbatim. Both ends of the chain needed a coordinated edit — this is why the fix is two
PRs across two repos, not one.

**Mechanism ("A-refined").**

- **PR #27** (`tasks/inbox-brief.md`, assistant-agent): one line added to the existing
  delivery-confirmation step. Once Rachel has confirmed delivery (exit 0 and
  `[notify] sent.` from `bridge/notify.ts` — see [[capabilities/inbox-brief]]), she must
  end her turn with a single final line, exactly: `BRIEF FILE: <absolute path to the
  scratch file she wrote and sent>` — no code fence, no bullet, no leading whitespace, so
  the line is machine-findable by its literal `B` prefix. On a delivery failure or a
  skipped sweep, she must NOT print this line (stating the failure/skip plainly instead),
  so a stale file is never advertised as current.
- **PR #183** (`examples/dashboard-config.json`, coderails): the inbox-brief button's
  `command` was rewritten so the **outer** `claude -p` agent is instructed: run the
  `bin/rachel` command as before, then read the file named on the `BRIEF FILE:` line and
  make its **entire final message** exactly that file's verbatim contents — nothing
  before it, nothing after, no preamble, no code fence. If no `BRIEF FILE:` line appears
  (delivery failed or the sweep was skipped), it instead reports Rachel's stated outcome
  in a single line. This closes the gap the sentinel-only fix couldn't: the outer agent's
  `result` field — what the modal actually renders — now IS the brief file's bytes, not a
  summary or relay of them.

**Verified live.** Pressing the button produced run `5cfc5ee0d5a32a82`, whose modal
output matched the `/tmp/inbox-brief-<ts>.txt` bytes Telegram received, whitespace-
normalised, with zero trace of Rachel's execution — confirming the mechanism closes the
gap end-to-end, not just in the two diffs read separately.

## Why this belongs next to the wrap fix

Both changes touch the same modal component and landed the same day as immediate,
observed-in-use follow-ups to the `dashboard-rethink` loop's t10/t12 work — one a
rendering defect (long lines), one a data-source defect (wrong content entirely). No
shared code path between them; grouped here as a ship-day cluster, not a shared mechanism.

## Files changed

- `skills/dashboard/app/src/styles/hud.css` (coderails, PR #181) — 3 lines added to
  `.hud-markdown pre`
- `skills/dashboard/app/test/hudMarkdownWrap.test.ts` (coderails, PR #181) — new, 38 lines
- `examples/dashboard-config.json` (coderails, PR #183) — inbox-brief button's `command`
  string rewritten (1 line changed)
- `tasks/inbox-brief.md` (assistant-agent, PR #27) — 1 line added to the delivery step

## See also

- [[dashboard]] — the skill page; Run Output viewer section updated for this cluster and
  to close the staleness the 2026-07-15 investigation flagged
- [[dashboard-run-output-rendering-gap_2026-07-15]] — the investigation that found the
  wiki's Run Output section 5 PRs stale (retired inline `<pre>` viewer, raw/clean toggle
  that no longer exists); this ingest is what closes that gap
- [[capabilities/inbox-brief]] (assistant-agent-wiki) — the sweep's own page, updated with
  the `BRIEF FILE:` sentinel contract
- [[pr_175-176_crack-on-gate-and-inbox-brief-button]] — the inbox-brief button's original
  build (thin-dispatcher shape, profile-bypass-can-only-spawn-`claude` boundary) this
  cluster builds on without changing
- [[pr_163-168_dashboard-rethink]] — the loop whose t10/t12 tasks shipped the modal these
  two PRs are following up on (not itself updated by this ingest — that PR range is
  #163-168; the modal/markdown work is #171-174, covered by the dashboard.md rewrite below)
