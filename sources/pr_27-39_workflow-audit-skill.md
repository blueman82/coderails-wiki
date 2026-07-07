---
title: "PR #27, #30, #37, #39 — workflow-audit skill"
type: source
created: 2026-07-07
last_updated: 2026-07-07
sources: []
tags: [source, workflow-audit, skill-creator, privacy, transcripts, task-evals, agentic-loop]
---

# PR #27, #30, #37, #39 — workflow-audit skill

Ingested by the loop-boundary wiki agent after all four PRs merged to `main`. This is an immutable record of what changed; one source page for the cluster (not fragmented per-PR).

## PR metadata

| Field | Value |
|---|---|
| PR #27 | merge `cacbbf5` — `skills/workflow-audit/scripts/scan_transcripts.sh` + tests + 3 fixtures |
| PR #30 | merge `dbc0f6d` — `scripts/cluster_ngrams.sh` + tests + fixture + `references/judge-contract.md` |
| PR #37 | merge `4217800` — `skills/workflow-audit/SKILL.md` (orchestration) |
| PR #39 | merge `654df98` — `scripts/tests/e2e.test.sh` + `docs/REFERENCE.md` entry (skill count 28→29) |
| Repo | `blueman82/coderails` |
| Sub-project | 3 of 5 in the agentic-OS evolution sequence (observability → routines → **workflow-audit** → assistant-agent kernel integration → improvement loops) — see [[dashboard]] for sub-project 1 |

## Summary

Shipped `coderails:workflow-audit`, a new skill that mines Claude Code session transcripts for tool-use patterns that repeat across sessions, judges which are genuine candidates for a new skill, and — only after explicit owner approval at a hard-stop gate — creates each approved skill through the normal [[writing-skills]] TDD process and a full PR gate. Four stages: scan → cluster → judge → propose-and-stop.

The pipeline is deliberately **mechanical-prefilter then model-judge**, not pure model-driven end to end: a 73 MB / 143-session transcript corpus is infeasible to hand an LLM directly, so two bash scripts (`scan_transcripts.sh`, `cluster_ngrams.sh`) do the extraction and n-gram clustering, and only the resulting small cluster JSON goes to a fresh sonnet judge subagent. This mirrors [[skill-testing-state_2026-06-26]]'s tiering logic (mechanical enforcement where stakes/scale demand it, model judgement where interpretation is needed) but for a new axis: corpus scale rather than compliance stakes.

## Architecture

1. **Scan** (`scan_transcripts.sh`) — walks `~/.claude/projects` (or a scoped subset), reads each session's `*.jsonl`, and emits one JSON line per session: `{session_id, project_slug, event_count, events:[{tool, head}]}`. Only `.type=="assistant"` records are read; `tool_use` items are extracted from `.message.content[]`. A structural privacy whitelist limits `head` to the first two whitespace tokens of a Bash command, the Skill name, or the Agent `subagent_type` — every other tool emits `{tool}` alone, never argument text or file content. See [[claude-code-transcript-schema_2026-07-07]] for the full record-type catalogue this scan sits on top of.
2. **Cluster** (`cluster_ngrams.sh`) — consumes that JSONL stream, builds `tool` or `tool:head` event strings per session, slides n=2..5 windows, and groups by `(n, ngram)` across sessions using `unique` on session ids (so one session repeating a pattern internally can't inflate distinct-session support). Emits `{scanned_sessions, clusters:[{ngram, n, count, sessions}], diagnostics:{below_threshold, truncated}}` for clusters meeting `--min-sessions` (default 3), capped at `--top` (default 50).
3. **Judge** — exactly one fresh sonnet subagent per run, given only the cluster JSON plus the `name`/`description` frontmatter of every existing `skills/*/SKILL.md`. The judge-contract (`references/judge-contract.md`) fixes its vocabulary to tool names, whitelisted heads, counts, session ids, and n-gram lengths — no transcript prose, no reconstructed intent beyond what those fields say. Three mandatory rejection criteria, checked in order: (1) project-specific convention rather than a generalisable task, (2) already covered by an existing skill, (3) a tooling-mechanics artifact of the loop's own plumbing (its own git/gh housekeeping) rather than a user-initiated task.
4. **Propose and stop** — a chart is shown to the owner (task summary, evidence count, sessions touched, proposed name/description, verdict). The **approval gate is a hard stop** via `AskUserQuestion` that explicitly overrides any standing loop autonomy — "crack on," "no human gates," or any full-autonomy envelope does not extend to skill creation from this skill. Zero approvals is a complete, successful run, not an escalation failure.
5. **Create** (only for approved candidates, one at a time) — invokes [[writing-skills]] for real RED/GREEN/REFACTOR, lands the new skill via its own branch + PR through the full gate sequence (`test_gate` → `pr-review-toolkit:review-pr` → security review → `post-review` → pr-scope evals → merge), and stops after each merge before starting the next. Never a direct commit to `main`, never written to a user's personal `~/.claude/skills`.

## Testing

Two unit test files (`scan_transcripts.test.sh`, `cluster_ngrams.test.sh`) plus one end-to-end test (`e2e.test.sh`, PR #39) that exercises the full `scan | cluster` pipeline over a synthetic multi-session corpus. The e2e suite's headline case is a **sentinel privacy negative control**: a fake secret (`SENTINEL_sk_live_99xyz`) is planted inside a Bash command's arguments, proven present in the raw fixture (so the test isn't vacuously passing), then asserted absent from both scan and cluster stdout — only the whitelisted two-token head (`"curl -H"`) survives. A second e2e property pins the no-approval-means-no-creation invariant structurally: the mechanical pipeline output alone can never trigger skill creation without the separate `AskUserQuestion` gate firing.

## Files changed

- `skills/workflow-audit/SKILL.md` — orchestration (scope mapping, size sanity, pipeline invocation, judge stage, proposal chart, approval gate, create step, wrap-up)
- `skills/workflow-audit/scripts/scan_transcripts.sh` + `scripts/tests/scan_transcripts.test.sh` + 3 fixtures (`fixture-small.jsonl`, `fixture-edge.jsonl`, `fixture-sentinel.jsonl`)
- `skills/workflow-audit/scripts/cluster_ngrams.sh` + `scripts/tests/cluster_ngrams.test.sh` + `fixtures/cluster-3sessions.jsonl`
- `skills/workflow-audit/references/judge-contract.md` — judge prompt template + schema + mandatory rejection criteria
- `skills/workflow-audit/scripts/tests/e2e.test.sh` — full-pipeline + sentinel negative control
- `docs/REFERENCE.md` — catalogue entry, skill count 28→29

## Wiki pages updated

- [[workflow-audit]] — new skill page
- [[claude-code-transcript-schema_2026-07-07]] — new investigation page cataloguing the transcript JSONL record types this skill's scan stage depends on
- [[writing-skills]] — cross-linked as the TDD process the create step invokes
- [[skill-testing-state_2026-06-26]] — cross-linked; same mechanical/model-judgement tiering logic, different axis (corpus scale vs. compliance stakes)
- [[dashboard]] — cross-linked as sub-project 1 in the same agentic-OS sequence this is sub-project 3 of

## Caveats / gotchas

- `workflow.config.yaml`'s `wiki_path` in the main repo is `null` at time of ingest — this wiki's location was resolved via the fallback path (`~/Github/coderails-wiki`, verified as a git repo) per the ingesting agent's instructions, not from config. Worth fixing the config's `wiki_path` if that null is unintentional. `(verified)`
- The judge's fixed vocabulary is a hard privacy boundary, not just a style guideline: `references/judge-contract.md` explicitly says the judge "cannot make that judgement and must reject or note the limitation, not guess" if a verdict would require knowing what a command's arguments meant beyond its whitelisted head.
- `--top 0` is treated as nonsensical input and floored to 1, rather than honoured as "give me zero clusters," to avoid a misleading `diagnostics.truncated: true` on an empty result.
- `--last-sessions` ranks by **in-file message timestamp**, never file `mtime` — this is the same gotcha the transcript-schema investigation page documents (a `last-prompt` record is a resume cache that bumps mtime without a timestamp of its own).
