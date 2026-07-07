---
title: "PR 36, 41, 33, 53, 65 — verified routines: intent-queue/runner contract + scheduled artifact-gated skill runs"
type: source
created: 2026-07-07
last_updated: 2026-07-07
sources: []
tags: [source, dashboard, routines, intent-queue, runner, launchd, memory-consolidation, agentic-os, sub-project-2-of-5]
---

# PR 36, 41, 33, 53, 65 — verified routines: intent-queue/runner contract + scheduled artifact-gated skill runs

Ingested by `/wiki-ingest` after merge. This is an immutable record of what changed. Five PRs, one loop, one theme: sub-project 2 of the agentic-OS evolution sequence (observability → **routines** → workflow-audit → assistant-agent kernel integration → improvement loops) — the piece [[dashboard]], [[assistant-link-send-gate-architecture]], and [[queue-contract-cross-pr-audit_2026-07-07]] all named as "still unbuilt" as of intake.

## PR metadata

| Field | Value |
|---|---|
| PR #36 | `routines/wu1-dashboard-lib`, merged `027c09f`, 2026-07-07 |
| PR #41 | `routines/wu2-runner`, merged `e322678`, 2026-07-07 |
| PR #33 | `routines/wu3-memory-consolidation`, merged `653146d`, 2026-07-07 |
| PR #53 | `routines/wu4-routines-launchd`, merged `4232393`, 2026-07-07 |
| PR #65 | `routines/wu5-docs`, merged `b6a40f5`, 2026-07-07 |
| Repo | `blueman82/coderails` |
| Sub-project | 2 of 5 in the agentic-OS evolution sequence |

## Summary

A **routine** is a scheduled skill run that isn't done just because `claude` exited 0 — it's done when a specific artifact exists, is fresh enough, and satisfies a predicate (the **artifact gate**). This cluster builds the whole chain end to end:

- **PR #36** — `@coderails/dashboard-lib`: the `Intent` schema (`button`, `input?`, `requestedAt`, `source`) and the `routines` section of `DashboardConfig` (`RoutineDef`: `cadence`, `expectedArtifact`, `escalation`), plus a compile-time `schema-compat.test.ts` that type-checks this package's `Intent` against the Obsidian plugin's own `IntentFile` so producer and consumer can't silently drift. Confirms the queue seam (`~/.claude/coderails-dashboard/queue/`) was already live from PR #25 — the Obsidian plugin already writes matching intent files — this PR gives that seam its first typed schema and its consumer-side contract.
- **PR #41** — `@coderails/dashboard-runner`: the one-shot sweeper. Claims a queued intent by atomic rename (`queue/<id>.json` → `processing/<id>.json`), spawns `claude` via the shared `buildArgv` profile→flag mapping, records the run, and — only if the claimed intent's button resolves to a `RoutineDef` — evaluates that routine's artifact gate and escalates on failure. Ships the per-intent failure boundary (one bad intent quarantines, doesn't crash the sweep), orphan recovery (a `processing/` file abandoned by a crashed sweep is reclaimed after 60 minutes as a `runner-error`), the artifact-gate predicate evaluator (`exists` / `contains` / `json-field`, with vault-root path-traversal defense), and the escalation mechanism (macOS notification + vault run note).
- **PR #33** — `memory-consolidation` skill: dedupes overlapping memories, flags stale/contradicted ones, refreshes `MEMORY.md`'s index. Writes its own durable report artifact (`~/.claude/coderails-dashboard/routines/memory-consolidation/report-{date}.md`) unconditionally, even when nothing changed — that unconditional write is what lets a routine gate on the report's existence rather than on `claude`'s exit code.
- **PR #53** — routine seeding + launchd wiring: `seedDueRoutines` (the scheduler) writes an `Intent` for every due routine into `queue/`, exactly like any other producer — the runner has no concept of "this came from a routine" once queued. Two launchd jobs (`com.coderails.routine-sweeper.calendar` at 03:00 daily, `com.coderails.routine-sweeper.watch` on queue-directory writes) drive seed+sweep and sweep-only respectively, installed/uninstalled idempotently by `launchd/install-routines.sh` / `uninstall-routines.sh`. Also ships `examples/dashboard-config.json` (the three shipped routines: `wiki-lint` nightly, `sync-docs-weekly`, `memory-consolidation-weekly`, all `"profile": "read-only"`) plus two scoped fixes to runner code from earlier work units.
- **PR #65** — `docs/routines.md`, the operator-facing guide, plus a drift audit that fixed two stale spots: `README.md`'s skill catalog didn't mention `memory-consolidation`, and `AGENTS.md`'s wiki page-type table had no entry for the vault run notes the runner writes (`dashboard-runs/<routine>.md`, `type: routine-run`) — now documented as explicitly **not** a wiki page type, excluded from `/wiki-ingest`/`/wiki-lint`.

## Architecture: intent producers, one runner

Every run — dashboard button, Obsidian command, or scheduled routine — starts as a small JSON **intent** file in `~/.claude/coderails-dashboard/queue/`. Producers only ever write intent files; they never invoke `claude` directly under this contract (the Obsidian plugin's interim direct-exec behaviour, predating this cluster, is a documented exception, not a pattern to copy). The runner is the **sole** consumer and the sole invoker of `claude` for a queued or scheduled run — this is stated as a permanent design rule in `skills/dashboard/lib/README.md`, even though the Obsidian plugin doesn't yet honor it fully.

A routine's `cadence` is `"nightly"` (due after 20h since last run) or `"weekly"` (due after 6.5 days) — both intentionally shorter than nominal so a routine delayed by a sleeping machine still fires on the next tick. `expectedArtifact.artifactPath` supports `{date}`/`{runId}`/`{vault}` template tokens; `predicate` is `exists`, `contains: marker`, or `json-field: path/value`. `escalation` is drawn from `["notification", "vault-note"]` — no dedup exists: a permanently misconfigured routine re-escalates in full on every calendar fire, forever, by design (silent swallowing judged worse than repeated noise).

## Security finding (empirically verified, documented in PR #65)

Under `claude -p` (the runner's invocation mode), `PreToolUse` hooks **do not fire** — confirmed directly: a `test_gate` deny-trigger did not stop a `git commit` under `-p`. `test_gate` and `enforce_pr_workflow` do not protect a routine run. A `"profile": "bypass"` routine runs headless with neither a tool allowlist nor the hook safety net. All three shipped routines use `"profile": "read-only"` for exactly this reason — see [[routines]] and [[intent-queue-runner-contract]] for the full mechanism.

## Files changed (representative, not exhaustive)

- `skills/dashboard/lib/src/intent.ts`, `config.ts`, `test/schema-compat.test.ts` (PR #36)
- `skills/dashboard/runner/src/{sweep,exec,escalate,artifactGate,runlog}.ts`, `bin/sweeper.sh`, `bin/main.ts` (PR #41)
- `skills/memory-consolidation/SKILL.md` (PR #33)
- `skills/dashboard/runner/src/seed.ts`, `bin/seed-and-sweep.sh`, `launchd/*.plist`, `launchd/install-routines.sh`/`uninstall-routines.sh`, `examples/dashboard-config.json` (PR #53)
- `docs/routines.md`, `README.md`, `AGENTS.md` (PR #65)

## Wiki pages updated

- [[dashboard]] — routines/Obsidian-plugin section updated to point at the now-shipped runner instead of describing it as an unbuilt dependency
- [[assistant-link-send-gate-architecture]] — routine-runs ASSISTANT.LINK panel slot cross-referenced as no longer purely a dependency gap
- [[queue-contract-cross-pr-audit_2026-07-07]] — Gap #3 (PR #36 / routines sub-project 2 had no wiki page) marked closed
- New pages: [[intent-queue-runner-contract]] (the schema + lifecycle design page), [[dashboard-runner]] (the sweeper), [[routines]] (the scheduling convention — cadence, artifact gates, escalation, launchd), [[memory-consolidation]] (the skill)

## Caveats / gotchas

- **Not portable.** Both launchd plists hard-code this machine's absolute paths (checkout location, log path, `/opt/homebrew/bin/node`) at authoring time — `launchd`'s environment carries no `PATH`. Moving the checkout or handing this to another user's machine requires hand-editing the plists first.
- **No dedup on escalation**, deliberately — see Architecture above.
- **`dashboard-runs/` vault notes are not wiki pages.** Two note conventions live side by side in that folder: the runner's `writeRunNote` (`type: routine-run` frontmatter) and the Obsidian plugin's direct-exec path (`status: running|done|failed`, no `type` field). Neither is ingested or linted by the wiki skills — this is now stated explicitly in `AGENTS.md`.
- **The queue directory has no producer-namespacing.** As already flagged in [[queue-contract-cross-pr-audit_2026-07-07]], the routines seeder is now a *third* writer into `~/.claude/coderails-dashboard/queue/` alongside the send-gate and workflow-audit's queue-mode proposals, sharing collision-avoidance via hash entropy only, not an enforced invariant.
