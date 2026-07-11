---
title: "PRs #139-141 — dashboard ASK-button UX + run-output clean view"
type: source
created: 2026-07-11
last_updated: 2026-07-11
sources: []
tags: [source, dashboard, ux, run-output, streaming, agentic-loop]
---

# PRs #139-141 — dashboard ASK-button UX + run-output clean view

Ingested by `/wiki-ingest` after merge. Immutable record of what changed. Three independent PRs
shipped as one theme: making the dashboard's "ask" flow faster to use (Enter-to-submit) and its
answer faster to read (a clean projected view instead of raw `stream-json`, plus a readability
pass on the box rendering it).

## PR metadata

| PR | Title | Merge commit | Merged | Files touched |
|---|---|---|---|---|
| #139 | Enter-to-submit on the ask input | `60fd025` | 2026-07-11 | `RailRight.tsx`, `RailRight.test.tsx` (new tests) |
| #140 | Run-output clean view (`projectAssistantText`) | `43d4b89` | 2026-07-11 | `streamJson.ts`, `OutputViewerPanel.tsx`, `streamJson.test.ts`, `OutputViewerPanel.test.ts` |
| #141 | Run-output box readability | `e54d38d` | 2026-07-11 | `hud.css` |

## Summary

**#139 — Enter-to-submit.** `RailRight.tsx`'s `.hud-cmd-input` (the free-text box behind any
`inputAllowed: true` button, e.g. "ask") gained an `onKeyDown` handler: `Enter` without `Shift`
calls the same `handleClick(btn)` the ASK button itself calls; `Shift+Enter` is a no-op (reserved
for a future multi-line case, not exercised yet). Three lines of production code; two new tests
(`RailRight.test.tsx`) confirm submit-on-Enter and no-submit-on-Shift+Enter via `fireEvent.keyDown`
against a mocked fetch.

**#140 — clean view by default.** A new exported `projectAssistantText(raw: string): string` in
`streamJson.ts` projects a raw `claude -p --output-format stream-json` log down to just the
assistant's readable prose, non-throwing (same posture as the existing `parseStreamJsonLine`).
Precedence: (1) the last well-formed `type:"result"` line's `result` field, if it's a non-empty
string — last-wins if more than one such line ever appears, rather than first-wins, on the
reasoning that a later result is more likely the coherent final answer across multiple turns; (2)
else, concatenated `text_delta` values pulled out of `stream_event` → `content_block_delta` events
(covers a still-live run with no `result` line yet); (3) else, the raw input unchanged, so a run
that produced real output never renders an empty box. `input_json_delta` (tool-use argument
streaming) is explicitly ignored — only `text_delta` contributes to the concatenation.

`OutputViewerPanel.tsx` calls `projectAssistantText(output)` and renders the projected text by
default, behind a new `showRaw` boolean toggled by a `.hud-output-toggle` button. **The toggle
only renders when `isLive` is true** — see Design fact below for why.

**#141 — readability.** Pure CSS, no logic change: `.hud-output-viewer` font-size `9.5px → 12px`,
`max-height 220px → 320px`, `line-height 1.5 → 1.6`; a new `.hud-output-toggle` style (bordered
mono button, hover state picks up `--rose-dim`) gives the new toggle button from #140 a real look
instead of unstyled default button chrome.

## Design fact: two output paths, and why the raw toggle is live-only

The dashboard's Run Output panel has always had two independent data paths feeding the same
`<pre>` block (established by [[pr_80-82_dashboard-stream-run-output-viewer]], extended by
[[pr_124_dashboard-run-output-result-extraction]]):

- **Live path**: a still-running run streams through the SSE `"run-output"` event into
  `DashboardState.runOutput[runId]` — this buffer is genuinely raw `stream-json`, one JSONL line
  per hook/system/assistant event.
- **Settled path**: once a run ends, the panel fetches `GET /api/run/output`, which since PR #124
  already runs `extractResultText()` server-side and returns only the final result text — there is
  no raw JSONL sitting client-side for a settled run to toggle back to.

#140's `showRaw` toggle button is therefore gated on `isLive` (`{isLive && <button
className="hud-output-toggle">...}`) — for a settled run, rendering a raw/clean toggle would be a
no-op control (there's nothing raw to reveal), so it's hidden entirely rather than shown disabled
or shown as dead weight. `OutputViewerPanel.test.ts` encodes this as an explicit assertion:
*"does not render the raw/clean toggle for a settled run ... there is no raw JSONL client-side to
toggle to."* **This asymmetry was caught by independent PR review, not present in the original
design** — the two-path split itself predates this cluster, but the toggle's live-only scoping is
this cluster's own contribution, verified by a dedicated negative test rather than left implicit.

## Tests

`streamJson.test.ts` gained 9 cases for `projectAssistantText`: delta-concatenation-only (no
result line), result-line preferred over deltas, result wins across multiple assistant turns,
malformed/unparseable lines skipped without throwing, an incomplete trailing partial line handled
without loss, full raw-passthrough when nothing parses as assistant text, empty-string input,
`input_json_delta` ignored, and result field empty/whitespace falls back to deltas rather than
blanking output. `OutputViewerPanel.test.ts` gained 2 cases: default-clean-view for a live run
(asserts projected prose present, raw JSONL markers absent, toggle present), and toggle-absent for
a settled run. `RailRight.test.tsx` gained 2 cases for Enter/Shift+Enter, described above.

## Files changed

- `skills/dashboard/app/src/components/RailRight.tsx` — `onKeyDown` handler (3 lines)
- `skills/dashboard/app/src/components/RailRight.test.tsx` — 2 new tests + `findInput` helper
- `skills/dashboard/app/src/lib/streamJson.ts` — `projectAssistantText()` (43 lines)
- `skills/dashboard/app/src/components/OutputViewerPanel.tsx` — `showRaw` state, `displayedOutput`
  projection, conditional toggle button (live-only)
- `skills/dashboard/app/test/streamJson.test.ts` — 9 new `projectAssistantText` tests
- `skills/dashboard/app/test/OutputViewerPanel.test.ts` — 2 new tests
- `skills/dashboard/app/src/styles/hud.css` — `.hud-output-toggle` new rule, `.hud-output-viewer`
  font-size/max-height/line-height bump

## Wiki pages updated

- [[dashboard]] — Run Output viewer section extended with the clean-view default, the toggle, and
  the live-vs-settled asymmetry it depends on; new "ASK-button UX + clean output view" subsection
- [[pr_80-82_dashboard-stream-run-output-viewer]] — no content change needed; this cluster builds
  on its `streamJson.ts` module without altering its documented behavior
- [[pr_124_dashboard-run-output-result-extraction]] — this cluster's design-fact section is the
  settled-path half of the same asymmetry that page documents for the server-side extraction

## Caveats / gotchas

- `projectAssistantText` and `extractResultText` (PR #124, `api/run/output/route.ts`) are two
  separate functions solving related but distinct problems: `extractResultText` runs server-side
  on a settled run's full log file and returns only the last result line (or raw fallback);
  `projectAssistantText` runs client-side on whatever `output` string the panel already has
  (either the live SSE buffer or the already-extracted settled text) and additionally knows how to
  fall back to concatenated deltas for a still-streaming run. Calling `projectAssistantText` on an
  already-extracted settled string is a harmless no-op in practice — a plain result string with no
  `stream-json` lines fails `parseStreamJsonLine` line-by-line and falls through to the raw-input
  return — but the two functions are not interchangeable and should not be merged without checking
  both call sites.
- Shift+Enter is a reserved no-op, not a documented multi-line-input feature — the input is a
  single-line `<input>`, not a `<textarea>`; Shift+Enter currently just fails to submit rather than
  inserting a newline.
- `Enter`-to-submit was added only to the ask input's existing `onKeyDown` prop path in
  `RailRight.tsx`; no equivalent was added to any other `inputAllowed` button context (there is
  currently only the one "ask" button configured, per [[dashboard]]'s button/run model section).
