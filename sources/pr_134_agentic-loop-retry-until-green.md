---
title: "PR #134 — agentic-loop retry-until-green + tool wiring"
type: source
origin: PR #134 (merged 2026-07-10, 5a6c54d)
created: 2026-07-10
last_updated: 2026-07-10
sources: []
tags: [source, agentic-loop, discipline-loop]
---

# PR #134 — agentic-loop retry-until-green + tool wiring

Mid-loop skill edit, triggered by direct user feedback during the [[pr_130-136_dashboard-right-rail-ux]] loop: the user asked for a fully-autonomous agentic loop with zero human interruption ever, after finding the loop's scope-negotiation questions excessive. The orchestrator refused to remove agentic-loop's 4 hard-stops (verification failure surviving retry, disproven premise, genuine ambiguity, destructive/irreversible operation) even under explicit "never ask a human anything" pressure, named the specific risk (a runaway loop pushing through a broken test suite or force-pushing with nobody watching), and negotiated a narrower, safe version instead. See the `feedback_never_remove_hard_stops` memory for the full exchange.

## What shipped

1. **Retry-until-green** — a new mechanism in the stop-conditions section: a single failing test/lint/verification check is not, by itself, grounds to stop and ask. Diagnose → fix → re-verify in a bounded cycle (default 5 distinct attempts; re-trying an identical fix without a new diagnosis doesn't count as a new attempt). Only escalates to hard-stop #1 once the bound is exhausted and the failure still reproduces. Narrows how often the hard-stop fires; does not remove it.
2. **Tool wiring inside the retry cycle**, each surfaced to the user before implementing:
   - `coderails:systematic-debugging` — invoked when a failure's cause isn't obvious, so "diagnose" means actual investigation, not another ad hoc guess.
   - `coderails:dispatching-parallel-agents` — invoked when a verification failure turns out to be multiple independent broken things (2+ unrelated files/subsystems), rather than fixing them one at a time.
   - `coderails:verify` — Phase 5's existing "prove the symptom reproduces against SOT" instruction now explicitly points at this command as its mechanism, rather than restating the sources-only discipline inline.
   - `coderails:disconfirm` — new: once a fix is diagnosed, before implementing it, argue against the diagnosis (what would falsify it, what edge case breaks it). A genuine gap — nothing previously stress-tested a diagnosis before code got written against it. Explicitly skipped when the fix is a direct, mechanical application of an already-source-verified design (e.g. this same session's six dashboard findings).
3. **Phase 13 hardened** — the terminal self-audit is now explicitly stated as mandatory, singular, and never a mid-loop checkpoint: it runs exactly once, at the very end, immediately before the `complete` LOOP-STOP declaration.
4. **File-size split** — `skills/agentic-loop/SKILL.md` was already over Anthropic's documented 500-line guidance (`writing-skills/anthropic-best-practices.md`) before this session touched it; a prior 2026-06-25 slimming pass ([[pr_39_agentic-loop-slim-v2]]) had already found zero further passages to cut without deleting real autonomy-specific content. This edit's own additions were extracted to a new sibling file, `skills/agentic-loop/retry-until-green.md`, matching the existing `learned-failure-modes.md` progressive-disclosure precedent — leaving a short pointer inline rather than growing the main file further.

## Verification discipline during the edit

All six of the skill's byte-frozen regions (the ones `loop_state_guard`/`loop_stall_guard` depend on: frontmatter description, Phase -2, the Phase 0.5 LOOP-STOP bullet, the Phase 13 KPI/LOOP-STOP-counts bullet, the Stop-conditions LOOP-STOP declaration block, the Context-window-persistence lifecycle section) were confirmed byte-identical before and after via direct diff — not assumed safe because the edits were "nearby." The hook test suite (`hooks/scripts/tests/run_all.sh`) was re-run after every edit pass (4 separate edits across the session), 38/38 passing throughout.

## Process incident: branch contamination, caught by design

The PR itself was delayed by a genuine mistake: the orchestrator's own design-spec and implementation-plan documents (written earlier in the same session, on the same working branch) ended up mixed with these two skill files on one shared branch. A worker dispatched to ship just the skill-edit PR ran the plan's own precondition check (`git log main..HEAD --stat` must show ONLY the 2 expected files) — it correctly failed twice (the first fix attempt was itself contaminated by the same session's cwd-tracking confusion) before a genuinely clean, isolated worktree off fresh `origin/main` resolved it. The worker stopped and reported both times rather than guessing at a recovery — exactly the "stop-and-flag, not silent cherry-pick" behavior the plan specified. See the `standing-orders.md` entry SO-3 (agentic-loop loop-state dir) for the promoted lesson.

## See also

- [[agentic-loop]] — the parent skill page this PR modifies
- [[pr_130-136_dashboard-right-rail-ux]] — the loop this mid-loop edit shipped inside
- [[pr_39_agentic-loop-slim-v2]] — the prior file-size slimming pass this edit's extraction respects (did not reopen that decision, added new content via progressive disclosure instead)
- [[systematic-debugging]] / [[dispatching-parallel-agents]] — the two vendored skills newly wired into the retry cycle
- `feedback_never_remove_hard_stops` (memory) — the record of the "never ask a human anything" pushback and negotiation
