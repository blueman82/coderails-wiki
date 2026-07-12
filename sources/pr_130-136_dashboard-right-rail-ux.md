---
title: "PRs #130-136 — Dashboard right-rail UX cluster"
type: source
origin: PR #130, #131, #132, #133, #135, #136 (merged 2026-07-10)
created: 2026-07-10
last_updated: 2026-07-10
sources: []
tags: [source, dashboard, ux, agentic-loop]
---

# PRs #130-136 — Dashboard right-rail UX cluster

Six independent PRs fixing UX/IA findings on the dashboard's right rail (Command Deck buttons + Run Output panel + PR Gates), identified via a prior critique session and re-confirmed against current source before implementation. Separate from the earlier ask-output data-extraction bug fix. Shipped as an [[agentic-loop]] run: brainstorming with a visual-companion mockup pass, `coderails:writing-plans` + `coderails:planning-sequence` stress-test, then 7 parallel workers (6 findings + 1 unrelated skill-edit PR, see [[pr_134_agentic-loop-retry-until-green]]).

## The six findings and their PRs

| PR | Finding | Merge commit | Files touched |
|---|---|---|---|
| #133 | Panel separation | 483ca87 | `hud.css` only |
| #130 | Input affordance | 16defd8 | `hud.css`, `RailRight.tsx` |
| #135 | Label wrapping | ce32b7c | `hud.css`, `RailRight.tsx` |
| #131 | Run-history structure | 5b26d74 | `hud.css`, `RailRight.tsx`, `OutputViewerPanel.tsx` |
| #132 | Output-viewer context | b1fd804 | `hud.css`, `OutputViewerPanel.tsx` |
| #136 | Button-state differentiation | aa41557 | `hud.css`, `RailRight.tsx`, `RailRight.test.tsx` (new), `package.json` |

1. **Panel separation** — `.hud-block` (shared across the ENTIRE dashboard, not just the right rail: also used by `RailLeft.tsx` and `AssistantLinkPanel.tsx`) gained a hairline border, subtle background wash, and a left rose-dim accent spine. Command Deck / Run Output / PR Gates were previously one undifferentiated scroll; the fix's blast radius correctly covers both rails, not just the three panels the original finding named.
2. **Input affordance** — `.hud-cmd-input` changed from an underline-only border to a full boxed field; buttons with `inputAllowed: true` gained a small "ARG" tag next to their label so the affordance is visible before any click.
3. **Label wrapping** — `.hud-cmd .hud-label` (deliberately scoped, not bare `.hud-label`, since that class is also used by `RunProgress.tsx` and `HudCallout.tsx`) gained `white-space: nowrap` + ellipsis truncation + a native `title` tooltip, replacing unpredictable two-line wrapping that broke the 2-column grid's row alignment.
4. **Run-history structure** — the uniform `·` glyph in both `RailRight.tsx`'s static finished-runs list and `OutputViewerPanel.tsx`'s clickable run-picker was replaced with a filled/hollow status glyph (◆ pass / ◇ fail), derived from `runResultLabel()`'s existing `"PASS" | "FAIL" | "RUNNING"` return type. The two implementations are *intentionally* duplicated (two real independent list components exist), with cross-reference comments in each file noting the sibling to keep in sync.
5. **Output-viewer context** — `OutputViewerPanel.tsx`'s `<pre>` output block gained a header bar (command name, outcome, duration, timestamp) above it, so a user can tell which run produced the text they're looking at. Markdown/ANSI rendering of the output text itself was explicitly deferred (out of scope) — the header fixes "which run is this," not "is this text formatted."
6. **Button-state differentiation** — the only finding requiring new component state. `ButtonUiState` gained a `lastOutcome: "completed" | "failed" | null` field, set when the `runs` SSE effect observes a button's run transition to ended, auto-clearing after ~1.5s via a `setTimeout` mirroring the existing `triggerShake` pattern. A bullet-flash CSS animation (green for success, intensified rose for failure) makes a completed run visible for the first time — previously a finished run reverted silently to idle with zero feedback.

## Process notes worth keeping

- **Design-agent scope drift caught, not accepted.** The mockup-generation agent's first pass substituted 2 unscoped "bonus" findings (deck-status legibility, gate scannability) in place of 2 of the original 6 — caught by the orchestrator comparing against its own source-verified findings list, not accepted at face value. The user was given a forced-choice ranking between the orchestrator's original six and the agent's severity-ranked eight, and chose the original six.
- **Test infrastructure gap, closed as part of #136.** No React Testing Library / jsdom existed in `skills/dashboard/app` before this cluster — `vitest.config.ts` was `environment: "node"` globally. #136 added `@testing-library/react` + a per-file `// @vitest-environment jsdom` docblock (not a global switch, to avoid risking the other 27 test files) and wrote the first component-mount tests for this codebase.
- **Base-staleness across concurrent sibling PRs.** #135's branch, left un-rebased while #130/#131/#132/#133 merged concurrently, would have silently reverted #130's work — caught by #135's own code review before merge, not by the plan. The orchestrator then proactively warned the two still-running siblings (#136, and the unrelated #134) about the same risk. Promoted as a repo-agnostic lesson: see the `standing-orders.md` entry SO-2 in the agentic-loop loop-state dir.
- **Eval design deliberately avoided sharing an oracle with its own tests.** #136's eval suite (tier 1, GO 4/4 P0 + 1/1 P1) checked `ButtonUiState`/CSS wiring via direct source grep rather than trusting the implementer's own 5 tests, and graded the test *file's* coverage with a fresh subagent blind to the implementation conversation — per [[task-evals]]'s anti-gaming rule 4 (oracle independence).

## See also

- [[dashboard]] — the parent skill page these six PRs extend
- [[agentic-loop]] — the loop this cluster ran under
- [[pr_134_agentic-loop-retry-until-green]] — the unrelated 7th PR that shipped in the same loop
- [[task-evals]] / [[task-evals-gate]] — the per-PR eval gate all seven PRs in this loop satisfied independently (a plan-writing correction: the original plan assumed one shared loop-scope eval, corrected during the `planning-sequence` stress-test pass since `merge.sh` gates per-PR, not per-loop)
