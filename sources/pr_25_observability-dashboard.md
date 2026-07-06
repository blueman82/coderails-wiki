---
title: "PR 25 — observability dashboard"
type: source
created: 2026-07-06
last_updated: 2026-07-06
sources: []
tags: [source, dashboard, observability, agentic-os, task-evals, agentic-loop]
---

# PR 25 — observability dashboard

Ingested by /wiki-ingest after merge. This is an immutable record of what changed.

## PR metadata

| Field | Value |
|---|---|
| PR number | #25 |
| Branch | `observability/spec` |
| Merged | 2026-07-06 |
| Merge SHA | `fbc3ea0` |
| Repo | `blueman82/coderails` |
| Sub-project | 1 of 5 in the agentic-OS evolution sequence (observability → routines → workflow-audit → assistant-agent kernel integration → improvement loops) |

## Summary

Shipped a new `coderails:dashboard` skill: a Next.js/React Three Fiber web HUD plus a companion Obsidian command-centre plugin, both reading state the kernel already produces (session dirs, agentic-loop `progress.json`, `gh`-polled PR artifact markers, hook logs, wiki/memory mtimes) with no new services and no telemetry leaving the machine. Collectors cover sessions/loops, PR gate state, memory trail, and system health. A single `POST /api/run` endpoint, token- and Origin-guarded, spawns headless `claude -p` runs through one profile→flag mapping (`buildArgv`, read-only / standard / bypass) with two independent layers closing a flag-smuggling vector. The web HUD centrepiece is a network-sphere visualisation with a run-state hue sweep. The Obsidian plugin registers a file-native `agentic-os` code-block processor and writes intent files to a queue directory that is the deliberately frozen seam for the not-yet-built routines sub-project.

## Process

16 SDD tasks, each independently reviewed. Five fix loops: a lock TOCTOU (Critical), marker-grammar drift, `hooksFired` today-scoping, a single-SSE-provider requirement, and a flag-smuggling scanner finding (High). The frozen Tier-2 eval suite (10 evals; required because the PR is both ≥3 work-units and touches an outward/irreversible surface) caught **two production bugs that every review round missed**: the launch script reported false-success against a squatted port, and a statically-prerendered page baked an empty button config into the HTML (fixed by forcing the route dynamic). Both an eval artifact (GO 10/10) and a review artifact were posted SHA-bound on the PR; `/coderails:merge`'s two gates each consumed their respective artifact before allowing the merge. The loop this PR ran under registered a loop-scope `evals.json` too (GO 4/4) — see [[agentic-loop]].

## Files changed

- `skills/dashboard/SKILL.md` + `scripts/{start,stop}-dashboard.sh` — skill scaffold and launch/stop scripts
- `skills/dashboard/app/` — the Next.js app: `src/lib/argv.ts` (buildArgv, the single profile→flag mapping), `src/lib/collect/{sessions,prGates,memoryTrail,health}.ts` (collectors), `src/app/api/{events,run}/route.ts` (SSE stream + guarded run trigger), `src/components/sphere/{NetworkSphere,Scene,GridFloor,Fallback2D}.tsx` (R3F visualisation + 2D fallback), `src/lib/config.ts` (fail-fast config loader), `src/lib/runlog.ts` (JSONL run records), `test/*.test.ts` (vitest suites)
- `skills/dashboard/obsidian/` — native Obsidian plugin source + committed reproducible `dist/main.js`
- `docs/coderails/specs/2026-07-06-observability-dashboard-design.md` + plan doc + mockup/reference assets — spec and plan the implementation followed
- `docs/REFERENCE.md` — catalogue entry

## Wiki pages updated

- [[dashboard]] — new skill page
- [[agentic-loop]] — cross-linked as the loop this PR ran under (see agentic-loop's own page for its loop-scope eval gate mechanics, unchanged by this PR)
- [[task-evals]] / [[task-evals-gate]] — cross-linked as the gate this PR both used and validated with two real production catches
- [[unregistered_loop_guard]] — cross-linked; born from a gap in a prior session's own orchestration, relevant context for large multi-task sessions like this one

## Caveats / gotchas

- The literal string "v1" was banned from all files this plan created (owner mandate) — the one exempted case is the `EVAL_ARTIFACT_MARKER_VERSION`/`REVIEW_ARTIFACT_MARKER_VERSION` tokens, which must be sourced from the existing shell libs, never re-authored as a literal.
- The eval suite catching two bugs that reviews missed is the first concrete demonstration of task-evals earning its keep in production, not just in gate mechanics — worth citing if anyone questions the tier-2 ceremony's cost.
- The Obsidian queue-directory seam (`~/.claude/coderails-dashboard/queue/`) is intentionally provisional — it exists now so sub-project 2 (routines) has a stable contract to build a real runner against, with an interim direct-exec path in the meantime.
