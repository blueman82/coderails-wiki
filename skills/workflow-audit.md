---
title: "Skill: workflow-audit"
type: skill
created: 2026-07-07
last_updated: 2026-07-07
sources:
  - sources/pr_27-39_workflow-audit-skill.md
  - sources/pr_43-44-46_workflow-audit-queue-seam.md
tags: [skill, workflow-audit, skill-creator, privacy, transcripts, agentic-os, sub-project-3-of-5, queue-contract]
---

# Skill: workflow-audit

Mines Claude Code session transcripts for tool-use patterns that repeat across sessions, judges which are genuine candidates for a new skill, and — only after explicit owner approval — creates each approved skill through [[writing-skills]]'s TDD process and a full PR gate. Sub-project 3 of the 5-part agentic-OS evolution sequence (observability → routines → **workflow-audit** → assistant-agent kernel integration → improvement loops); see [[dashboard]] for sub-project 1.

Source: `coderails/skills/workflow-audit/SKILL.md`
Judge contract: `coderails/skills/workflow-audit/references/judge-contract.md`
Invoked as: `coderails:workflow-audit`

## Trigger phrases

"look at our last N sessions and pull out repeated tasks", "what do I do repeatedly that isn't a skill yet", "audit my workflows", "mine my transcripts for skill candidates", "turn my repeated tasks into skills".

## Pipeline

```
scan_transcripts.sh [--all-projects | --project <slug>] [--days N | --last-sessions N]
  | cluster_ngrams.sh --min-sessions 3 [--top K]
  → single fresh judge subagent (references/judge-contract.md)
  → proposal chart → AskUserQuestion approval gate (hard stop)
  → writing-skills TDD, one approved skill at a time, each its own PR
```

Full architecture, testing approach, and file list: [[pr_27-39_workflow-audit-skill]].

## Key design decisions

- **Mechanical prefilter, then model judge** — a 73 MB / 143-session corpus is infeasible to hand an LLM directly. Two bash scripts do scan + n-gram clustering; only the resulting small cluster JSON reaches a judge subagent. Same tiering logic as [[skill-testing-state_2026-06-26]] (mechanical where scale/stakes demand it, model judgement where interpretation is needed), applied to a new axis — corpus scale rather than compliance stakes.
- **Structural privacy boundary, not a redaction pass.** `scan_transcripts.sh` only ever extracts three whitelisted shapes: a Bash command's first two tokens, a Skill's name, or an Agent's `subagent_type`. Every other tool emits `{tool}` alone. This is enforced by construction (the jq filter never touches other fields), not by a downstream scrub step. See [[claude-code-transcript-schema_2026-07-07]] for the full transcript record-type catalogue this scan sits on top of.
- **Judge has a fixed vocabulary.** The judge subagent receives exactly two inputs — the cluster JSON and existing skill names/descriptions — and must reject or flag a limitation rather than guess if a verdict would need information beyond whitelisted heads, counts, and session ids.
- **Approval gate overrides loop autonomy.** The `AskUserQuestion` approval step is a hard stop that applies even inside an agentic-loop session authorised for full autonomy ("crack on", "no human gates", "self-merge"). Skill creation never proceeds on an earlier blanket authorisation or an inferred preference — only an explicit approval given in that interaction. Zero approvals is a complete, successful run.
- **Skills always land via the normal repo gates.** Approved candidates go through [[writing-skills]] RED-GREEN-REFACTOR and a full PR (`test_gate` → `pr-review-toolkit:review-pr` → security review → `post-review` → pr-scope evals → merge) one at a time — never a direct commit to `main`, never written into a user's personal `~/.claude/skills`.

## Mandatory judge rejection criteria

Checked in order; first match wins:

1. Project-specific convention, not a generalisable task (mirrors [[writing-skills]]'s own rule against project-specific skills).
2. Already covered by an existing skill (checked against the supplied name/description list).
3. Tooling-mechanics artifact of the loop's own plumbing (its own git/gh housekeeping) rather than a user-initiated task.

## Related

- [[pr_27-39_workflow-audit-skill]] — the ingest source page (full architecture, testing, files)
- [[claude-code-transcript-schema_2026-07-07]] — transcript JSONL schema catalogue this skill's scan stage relies on
- [[writing-skills]] — the TDD process invoked for each approved candidate
- [[skill-testing-state_2026-06-26]] — same mechanical/model-judgement tiering logic, prior investigation
- [[dashboard]] — sub-project 1 of the same agentic-OS sequence
