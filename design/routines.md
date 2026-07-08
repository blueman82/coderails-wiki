---
title: "Verified routines (scheduling convention)"
type: design
created: 2026-07-07
last_updated: 2026-07-08
sources:
  - sources/pr_36-41-33-53-65_verified-routines.md
  - sources/pr_88_93_dashboard-launchd.md
tags: [design, routines, cadence, artifact-gate, launchd, escalation, agentic-os, sub-project-2-of-5]
---

# Verified routines

The scheduling convention that turns a skill into a **routine**: a run that isn't done just because `claude` exited 0, but only when a specific artifact exists, is fresh enough, and satisfies a predicate. Shipped across [[pr_36-41-33-53-65_verified-routines|PRs #36/#41/#33/#53/#65]]; operator-facing doc is `docs/routines.md` (PR #65).

## Why "verified"

A bare cron job piping into `claude -p` proves nothing beyond "the process didn't crash." A routine's artifact gate — evaluated by [[dashboard-runner]] after a clean exit — is the actual verification: an exit-0 `claude` process that never wrote its expected artifact is a **failure**, not a success. This is the entire reason routines exist as a distinct concept from a plain scheduled button press `(verified, docs/routines.md)`.

## How a routine becomes a run: seeding

Routines don't have their own scheduling engine. The seed step (`seedDueRoutines()`, `skills/dashboard/runner/src/seed.ts`) writes an ordinary `Intent` into `queue/` for every due routine — `{ button, requestedAt, source: "scheduler" }` — using the exact same shape and directory any other producer uses. Once queued, [[dashboard-runner|the runner]] has no concept of "this came from a routine" versus a button press; it just executes buttons, and only checks for a matching `RoutineDef` after a clean exit to decide whether an artifact gate applies `(verified, skills/dashboard/runner/src/seed.ts` code comment)`.

## Due-ness mechanism

`isDue()` compares `now` against the routine's most recent recorded run (`readRuns()`, newest-first):

- **`nightly`** — due after **20 hours** since last run.
- **`weekly`** — due after **6.5 days** since last run.

Both thresholds are intentionally shorter than their nominal cadence so a routine delayed by a sleeping machine still fires on the next tick rather than sliding a full extra day `(verified, docs/routines.md)`. A negative elapsed time (clock skew, or a system clock corrected backward) is treated as **due** rather than compared numerically — a spurious extra run is recoverable, a routine silently wedged as permanently "not due" is not `(verified, skills/dashboard/runner/src/seed.ts` code comment)`.

**Duplicate-seed defense**: before writing a new intent, `seedDueRoutines` checks whether the routine's button already has a file in `queue/` or `processing/`. This check-then-write is not atomic — two concurrent seed passes for the same routine could both pass the check — but seeding only ever runs from the calendar-triggered plist, and launchd serialises a single calendar job's fires, so the race is accepted as narrow: worst case is one duplicate execution, bounded downstream by the runner's own atomic rename-based claim, not data loss `(verified, skills/dashboard/runner/src/seed.ts` code comment)`.

An unrecognised `cadence` string, or a routine whose `buttonRef` resolves to no button, doesn't crash seeding — it escalates a `runner-error` for that routine alone and the rest of the routines still get considered (same per-item failure-boundary discipline as the runner's own sweep).

## `RoutineDef` field contract

Routines live in the `routines` array of `~/.claude/coderails-dashboard.json`, validated by `validateRoutines()` at config-load time (`skills/dashboard/lib/src/config.ts`):

- **`name`** — the routine's own identifier, must be unique.
- **`skillCommand` / `buttonRef`** — exactly one, never both, never neither. `buttonRef` reuses an existing button's `command`/`cwd`/`profile` — all three shipped routines take this path.
- **`foreignSkillPath`** (optional) — an absolute path to a skill living outside the coderails repo (e.g. `sync-docs`'s personal-plugin location). The runner checks this path exists *before* spawning `claude`, escalating `skill-missing` rather than spawning and letting it fail inside the sandbox.
- **`cadence`** — `"nightly"` or `"weekly"` only; see Due-ness above.
- **`expectedArtifact`** — `{ artifactPath, maxAgeSeconds, predicate }`, the artifact gate [[dashboard-runner]] evaluates. `artifactPath` supports `{date}`/`{runId}`/`{vault}` tokens. `predicate` is `exists`, `contains: marker`, or `json-field: path/value`.
- **`escalation`** — array drawn from `["notification", "vault-note"]`. An empty array means a routine fails silently except in the runlog — rarely what you want.

Worked example, the shipped `wiki-lint` routine (`examples/dashboard-config.json`):

```json
{
  "name": "wiki-lint",
  "label": "Wiki Lint (nightly)",
  "buttonRef": "wiki-lint",
  "cadence": "nightly",
  "expectedArtifact": {
    "artifactPath": "{vault}/log.md",
    "maxAgeSeconds": 129600,
    "predicate": { "kind": "contains", "marker": "## [{date}] lint" }
  },
  "escalation": ["notification", "vault-note"]
}
```

`maxAgeSeconds` should sit comfortably above the cadence interval — the nightly example above uses 129600s (36h); the two weekly routines use 691200s (8 days) — so a slightly-late run doesn't fail its own gate on staleness alone `(verified, docs/routines.md)`.

## The three shipped routines

All `"profile": "read-only"` (`examples/dashboard-config.json`, PR #53):

- **`wiki-lint`** (nightly) — gates on `{vault}/log.md` containing `## [{date}] lint`.
- **`sync-docs-weekly`** (weekly) — the `sync-docs` skill, referenced via `foreignSkillPath` (lives outside this repo).
- **`memory-consolidation-weekly`** (weekly) — see [[memory-consolidation]]; gates on that skill's own report file existing.

## launchd wiring

Two jobs, both installed idempotently by `launchd/install-routines.sh` / removed by `uninstall-routines.sh`:

- **`com.coderails.routine-sweeper.calendar`** — fires daily at 03:00, runs `bin/seed-and-sweep.sh` (seed, then sweep regardless of the seed step's own exit code — a seeding failure must not block already-queued intents from being processed).
- **`com.coderails.routine-sweeper.watch`** — fires on any write under `~/.claude/coderails-dashboard/queue`, runs `bin/sweeper.sh` (sweep only, no seed — a button press already wrote its own intent).

**Not portable.** Both plists hard-code this machine's absolute paths (checkout location, log path, `/opt/homebrew/bin/node`) at authoring time — `launchd`'s environment carries no `PATH` at all (confirmed via `launchctl print gui/$UID`), so both the plists and the bin scripts they invoke use absolute paths throughout rather than relying on shell PATH resolution. Moving the checkout, or handing this to another user's machine, requires hand-editing the plists first `(verified, docs/routines.md)`.

**Boot persistence — install copies the plists into `~/Library/LaunchAgents/` (2026-07-08, PR #85).** The install step doesn't merely `launchctl bootstrap` the plists from the repo directory — it first copies each plist to `~/Library/LaunchAgents/<label>.plist` (mode 0644) and bootstraps from *that* copy. This is load-bearing: a `launchctl bootstrap` from an arbitrary path survives only until logout/reboot, and `launchd` only auto-loads plists that live in `~/Library/LaunchAgents/`. Bootstrapping straight from the repo path silently unloads the entire routines system on the next reboot. **Discovered by the first live fire (2026-07-08):** the 03:00 run succeeded (3 intents seeded, all three routines green), the machine crashed and rebooted at 07:34, and afterwards `launchctl list` showed no `com.coderails` jobs and `~/Library/LaunchAgents/` held no `com.coderails` plists — the whole system had silently unloaded. Uninstall now boots out each label *and* removes the LaunchAgents copy, working against both the old repo-path-bootstrap install and the new copy-based one. Both scripts also `shopt -s nullglob` + guard the plist glob so an empty match exits with a clear error instead of crashing mid-loop under `set -euo pipefail` `(verified, docs/routines.md, launchd/install-routines.sh)`. Merge commit `866b5a9`; see [[pr_36-41-33-53-65_verified-routines]] for the original build.

**This fix's pattern was reused, not rediscovered, for the dashboard server itself ([[pr_88_93_dashboard-launchd]], same day):** the dashboard's own launchd agent (`skills/dashboard/scripts/*` → [[dashboard]]'s "Surviving reboots" section) adopts the identical copy-then-bootstrap-from-copy shape in its own `launchd/install-dashboard-agent.sh`, plus a follow-up fix for an uninstall-time `launchctl bootout` race (async unload, ~2s observed) that this routines system's own uninstaller does not need to handle the same way, since its jobs aren't typically uninstalled while actively mid-run the way a `KeepAlive` dashboard server is.

## Escalation channels: where to look when something fails

Four places, roughest signal to most precise, per `docs/routines.md`:

1. **`~/.claude/coderails-dashboard/routines/sweeper.log`** — both plists' stdout/stderr; first stop for "did the job even run."
2. **`~/.claude/coderails-dashboard/runs/runs.jsonl`** — one JSONL record per run (start + finish lines, folding in `exitCode`/`endedAt`), across every button and routine — the ground truth for "did this run, when, with what argv, what exit code."
3. **Vault run notes** — `<vault>/dashboard-runs/<routine>.md`, append-only, one section per run, green or red. **Not a wiki page** — see the caveat below.
4. **macOS notification** — fired synchronously via `osascript`. Transient; if missed, the vault note and runlog are the durable record.

## The security finding: bypass-profile routines run outside the hook safety net

**Empirically verified in-repo, 2026-07-07 (PR #65):** under `claude -p` (the non-interactive mode the runner always uses), `SessionStart`, `UserPromptSubmit`, and `Stop` hooks fire normally — but **`PreToolUse` hooks do not fire**. Confirmed directly: with a `test_gate` deny-trigger configured, a `-p` invocation ran `git commit` and it succeeded; the gate never engaged `(verified, docs/routines.md` and `skills/dashboard/runner/src/sweep.ts` code comment)`.

Concretely: `test_gate` and `enforce_pr_workflow` — the hooks that block untested commits and direct pushes to `main` in an interactive session — **do not protect a routine run**. A routine configured with `"profile": "bypass"` runs headless with neither a CLI tool allowlist (`read-only` gets `--allowedTools Read Grep Glob`; `bypass` gets none at all) nor the hook-based safety net an interactive terminal session has. **This is why all three shipped routines use `"profile": "read-only"`** — if a routine's skill needs to write files or run commands, its actions are gated only by the artifact check after the fact, never by a hook before the fact.

## Not a wiki page type: `dashboard-runs/` vault notes

The runner's `writeRunNote` (`type: routine-run` frontmatter) and the Obsidian plugin's direct-exec path (`status: running|done|failed`, no `type` field) both write into `<vault>/dashboard-runs/`, but neither is a wiki page under `AGENTS.md`'s schema — neither is ingested by `/wiki-ingest` nor checked by `/wiki-lint`, and neither should ever be linked via `[[wiki-links]]`. This distinction is now stated explicitly in `AGENTS.md`'s page-type table (PR #65's drift fix).

## See also

- [[intent-queue-runner-contract]] — the schema and lifecycle a routine's intent travels through once seeded
- [[dashboard-runner]] — the executor that evaluates a routine's artifact gate and escalates on failure
- [[memory-consolidation]] — one of the three shipped routines, a worked example
- [[dashboard]] — sub-project 1; owns the `ButtonDef`/`buildArgv` machinery a `buttonRef` routine reuses; its own launchd agent reuses this page's boot-persistence fix
- [[pr_36-41-33-53-65_verified-routines]] — the source record for this page
- [[pr_88_93_dashboard-launchd]] — the dashboard's own launchd agent + uninstall bootout-race fix, reusing this page's copy-then-bootstrap-from-copy pattern
