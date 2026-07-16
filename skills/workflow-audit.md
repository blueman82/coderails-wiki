---
title: "Skill: workflow-audit"
type: skill
created: 2026-07-07
last_updated: 2026-07-15
sources:
  - sources/pr_27-39_workflow-audit-skill.md
  - sources/pr_43-44-46_workflow-audit-queue-seam.md
  - sources/pr_55-60-64-66-67_approve-build-runner.md
  - sources/pr_89-100-104-106_approve-build-e5-live.md
tags: [skill, workflow-audit, skill-creator, privacy, transcripts, agentic-os, sub-project-3-of-5, queue-contract, builder]
---

# Skill: workflow-audit

Mines Claude Code session transcripts for tool-use patterns that repeat across sessions, judges which are genuine candidates for a new skill, and creates each `propose`-verdict skill through [[writing-skills]]'s TDD process and a full PR gate. The judge's propose/reject verdict is the only filter that decides what gets built — the in-session run no longer stops to ask the owner (see "No interactive approval gate" below; the removed `AskUserQuestion` step is what the durable contradiction flag on this page used to describe). Sub-project 3 of the 5-part agentic-OS evolution sequence (observability → routines → **workflow-audit** → assistant-agent kernel integration → improvement loops); see [[dashboard]] for sub-project 1.

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
  → proposal chart → (in-session: straight to create; queue-mode: write queue entry)
  → writing-skills TDD, one proposed skill at a time, each its own PR
```

Full architecture, testing approach, and file list: [[pr_27-39_workflow-audit-skill]].

## Key design decisions

- **Mechanical prefilter, then model judge** — a 73 MB / 143-session corpus is infeasible to hand an LLM directly. Two bash scripts do scan + n-gram clustering; only the resulting small cluster JSON reaches a judge subagent. Same tiering logic as [[skill-testing-state_2026-06-26]] (mechanical where scale/stakes demand it, model judgement where interpretation is needed), applied to a new axis — corpus scale rather than compliance stakes.
- **Structural privacy boundary, not a redaction pass.** `scan_transcripts.sh` only ever extracts three whitelisted shapes: a Bash command's first two tokens, a Skill's name, or an Agent's `subagent_type`. Every other tool emits `{tool}` alone. This is enforced by construction (the jq filter never touches other fields), not by a downstream scrub step. See [[claude-code-transcript-schema_2026-07-07]] for the full transcript record-type catalogue this scan sits on top of.
- **Judge has a fixed vocabulary.** The judge subagent receives exactly two inputs — the cluster JSON and existing skill names/descriptions — and must reject or flag a limitation rather than guess if a verdict would need information beyond whitelisted heads, counts, and session ids.
- **No interactive approval gate — the judge's verdict is the filter.** The in-session run has no `AskUserQuestion` step: every `propose`-verdict candidate goes straight to the create step, and every `reject`-verdict one is dropped. Nothing is created that the judge rejected, and zero `propose` verdicts is a complete, successful run — not a failure to escalate past. Queue-mode (below) is the one path that defers creation to a human: it writes a queue entry instead of building, and the dashboard Approve-click is that entry's sole creation trigger.
  > **RESOLVED 2026-07-16 (lint) — source change, was the durable CONTRADICTION flag.** This bullet previously claimed an `AskUserQuestion` hard-stop that fired "even under crack on", faithful to the old `workflow-audit/SKILL.md` §7 "Approval gate — hard stop, no exceptions". That gate conflicted with [[crack_on_gate]], which denies **every** `AskUserQuestion` call while a session's crack-on flag is stamped (bare `tool_name = "AskUserQuestion"` matcher, no carve-out). The conflict is now moot: the source was rewritten (SKILL §7 → "Proceed to creation", commit `a6d0142`, 2026-07-15, on the deliberately-named `worktree-workflow-audit-no-approval-gate` branch — "no `AskUserQuestion`, no waiting on the owner"), removing the interactive gate entirely. So `crack_on_gate.sh` no longer suppresses any workflow-audit call — there is nothing left to suppress. The owner-decision-owed resolution was to narrow the source claim, and that has happened. Note kept (not deleted) so the durable flag's disappearance is explained rather than silent. The 2026-07-15 rewrite still lacks its own `sources/` ingest page — see the lint suggestions.
- **Skills always land via the normal repo gates.** Proposed candidates go through [[writing-skills]] RED-GREEN-REFACTOR and a full PR (`test_gate` → `pr-review-toolkit:review-pr` → security review → `post-review` → pr-scope evals → merge) one at a time — never a direct commit to `main`, never written into a user's personal `~/.claude/skills`.

## Queue-mode output (the human-approval surface)

Section 5 of `SKILL.md` (added by [[pr_43-44-46_workflow-audit-queue-seam]],
PRs #43/#44/#46) gives each `verdict:"propose"` judge output an asynchronous
surface on the [[dashboard]]'s existing approval queue. Queue-mode is opt-in and
mutually exclusive with in-session creation: in queue-mode a `propose` verdict is
written as a queue entry *instead* of being built in-session, and the dashboard
Approve-click is that entry's sole creation trigger. Since the in-session path no
longer has its own `AskUserQuestion` gate (see above), this queue is now the only
place a human gates what gets built. `write_queue_entry.sh` writes one
`QueueFileEntry` per proposal (`toolName: "workflow-audit:propose-skill"`, six
D2-whitelisted fields, sha256 hash binding, `0700`/`0600` perms), reusing the
generic queue envelope verbatim rather than a new schema. The pinned invariants —
stale-approval caution, zero-approvals-is-success, D2-whitelist-only content,
never committed to a repo — all apply to a queue entry.

**Approval now triggers a real build (updated 2026-07-07).** The paragraph
above described the state as of [[pr_43-44-46_workflow-audit-queue-seam]];
[[pr_55-60-64-66-67_approve-build-runner]] (loop 2, same day) closed that gap.
Clicking Approve on a `workflow-audit:propose-skill` queue entry now claims
the hash, re-validates it, and spawns a wrapper-owned `claude -p` session
(`skills/dashboard/scripts/run-builder.sh`) that authors the skill via
`skill-creator` and `writing-skills`' RED/GREEN/REFACTOR process, then opens a
PR — never merges it directly (mechanically enforced via
`--disallowedTools`, not just a prompt instruction). The owner still reviews
and merges the resulting PR by hand; a click approves *creation*, not landing
an unseen diff. See [[pr_55-60-64-66-67_approve-build-runner]] for the full
claim/spawn/wrapper/prompt-fencing architecture. **E5 (a real, manual,
one-time build through the pipeline) is now DONE (2026-07-08).** The pipeline
has built and merged its first real skill — [[verify-merged-pr]] — from a
genuine Approve click; the live run surfaced three production-only defects no
test caught (builder repo-identity, launchd env, opaque-build feedback). The
full loop is proven end-to-end, not just wired. See
[[pr_89-100-104-106_approve-build-e5-live]].

## Mandatory judge rejection criteria

Checked in order; first match wins:

1. Project-specific convention, not a generalisable task (mirrors [[writing-skills]]'s own rule against project-specific skills).
2. Already covered by an existing skill (checked against the supplied name/description list).
3. Tooling-mechanics artifact of the loop's own plumbing (its own git/gh housekeeping) rather than a user-initiated task.

## Related

- [[pr_27-39_workflow-audit-skill]] — the ingest source page (full architecture, testing, files)
- [[pr_43-44-46_workflow-audit-queue-seam]] — the queue-mode integration source page (writer, dashboard render, consumption-seam contract)
- [[pr_55-60-64-66-67_approve-build-runner]] — the builder pipeline that turns Approve into a real skill-creator build + PR; supersedes the "status flip only" claim above
- [[pr_89-100-104-106_approve-build-e5-live]] — the live-fire close of that pipeline (E5 done): first built skill [[verify-merged-pr]] + honest build feedback
- [[verify-merged-pr]] — the first skill this loop built end-to-end (Approve → headless `skill-creator` build → merged PR)
- [[claude-code-transcript-schema_2026-07-07]] — transcript JSONL schema catalogue this skill's scan stage relies on
- [[writing-skills]] — the TDD process invoked for each proposed candidate
- [[skill-testing-state_2026-06-26]] — same mechanical/model-judgement tiering logic, prior investigation
- [[dashboard]] — sub-project 1 of the same agentic-OS sequence; hosts the queue/AssistantLinkPanel this skill now writes into
- [[queue-contract-cross-pr-audit_2026-07-07]] — the investigation that first flagged this queue-mode work as pending/not-yet-PR'd; now closed by this cluster
