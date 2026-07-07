---
title: "Intent-queue/runner contract"
type: design
created: 2026-07-07
last_updated: 2026-07-07
sources:
  - sources/pr_36-41-33-53-65_verified-routines.md
tags: [design, dashboard, intent-queue, runner, routines, agentic-os, sub-project-2-of-5]
---

# Intent-queue/runner contract

The on-disk contract that lets every trigger of a `claude` run — a dashboard button, an Obsidian command, or a scheduled routine — funnel through one queue and one executor, instead of each surface spawning `claude` itself.

## Context

The queue directory (`~/.claude/coderails-dashboard/queue/`) was already live from [[pr_25_observability-dashboard|PR #25]] — the Obsidian plugin was already writing intent files there on every button press. What was missing was a typed schema for those files and a consumer that actually claimed and executed them; until [[pr_36-41-33-53-65_verified-routines|this cluster]], the plugin's own direct-exec path was the only thing that ran `claude`. This page documents the schema and lifecycle `@coderails/dashboard-lib` (PR #36) gives that seam, and the executor [[dashboard-runner]] (PR #41) that consumes it.

## The rule

**Producers only ever write intent files; they never invoke `claude` directly under this contract.** The runner (`skills/dashboard/runner`) is the sole consumer and the sole invoker of `claude` for a queued or scheduled run. This is stated as a permanent design rule in `skills/dashboard/lib/README.md` — a rule for every future surface wired to this queue, not a description of what's true today.

**Known exception, not a precedent:** the Obsidian plugin's `pressButton` (`obsidian/src/exec.ts`) currently ALSO direct-execs `claude` itself after writing the intent file — it predates this cluster and does not yet wait for the runner to claim and execute the queued intent. The lib README is explicit that this is transitional producer behaviour on the way to the runner becoming sole executor, not something a new producer should copy `(verified, skills/dashboard/lib/README.md)`.

## The `Intent` schema

```typescript
interface Intent {
  button: string;       // matches a ButtonDef.name or a routine's own trigger name
  input?: string;       // optional freeform text, never parsed as a flag
  requestedAt: number;  // epoch-ms (Date.now()), NOT an ISO string
  source: "web" | "obsidian" | "cli" | string;
}
```

`parseIntent(raw: unknown): Intent` (`skills/dashboard/lib/src/intent.ts`) validates and returns a typed `Intent`, throwing `IntentValidationError` on anything malformed — `button` and `source` must be strings, `requestedAt` a finite number, `input` a string when present `(verified)`. A compile-time `test/schema-compat.test.ts` type-checks this `Intent` against the Obsidian plugin's own `IntentFile` type on every build, so producer and consumer schemas cannot silently drift apart `(verified, skills/dashboard/lib/README.md)`.

`IntentFile.source` is presently the single literal `"obsidian"` on the wire (the only merged producer at the time PR #36 landed); `Intent.source`'s wider string union exists to admit future producers (`"web"`, `"cli"`) — the scheduler (PR #53) is the second producer to actually use it, writing `source: "scheduler"` `(verified, skills/dashboard/runner/src/seed.ts)`.

## Lifecycle: queue → processing → archive/quarantine

A queued intent file moves through exactly these directories, atomically claimed by rename — never copied, never edited in place `(verified, skills/dashboard/lib/README.md)`:

1. **`queue/<runId>.json`** — written by a producer. Not yet claimed. Directory created with mode `0o700` by producers.
2. **`processing/<runId>.json`** — claimed by the runner via `fs.renameSync`. The atomicity of a same-filesystem rename is what prevents two runner instances from double-claiming the same intent — a racing sweeper's rename simply fails because the source is already gone, and that failure is treated as "nothing to do here," not an error `(verified, skills/dashboard/runner/src/sweep.ts)`.
3. **On success: `archive/<runId>.json`** — never deleted, subject to a configurable retention prune (not yet built).
4. **On malformed input (fails `parseIntent`, or names a button the config doesn't have): `quarantine/<runId>.json`** — the sweep continues past it rather than crash-looping.

**Orphan recovery:** a file left in `processing/` for more than 60 minutes (`ORPHAN_THRESHOLD_MS`) is assumed to belong to a sweep that crashed before archiving/quarantining it — a healthy sweep processes an intent in seconds to minutes. `recoverOrphans()` moves it to `quarantine/`, records a failed run, and escalates a `runner-error`, all before the main claim loop runs each sweep `(verified, skills/dashboard/runner/src/sweep.ts)`.

## NOTE: this is a separate directory from `approvals/`

**`~/.claude/coderails-dashboard/approvals/` is a sibling directory, not part of this lifecycle.** [[pr_62_10_approvals-dir-move|PR #62]] (a later, independent change, same day) moved the send-gate's own pending-approval files out of `queue/` into `approvals/` specifically because this queue's own consumers — the routines runner's sweep and workflow-audit's `propose-skill` writer, both still writing/reading `queue/` — would quarantine or reject a `QueueFileEntry` approval file as malformed input, since it doesn't parse as an `Intent`. Do not conflate the two: `queue/`+`processing/`+`archive/`+`quarantine/` is this contract's intent lifecycle; `approvals/<hash>.json` is the send-gate's own file-per-approval store, described in [[assistant-link-send-gate-architecture]].

## Config: buttons and routines share one file

`~/.claude/coderails-dashboard.json` holds both `buttons` (`ButtonDef`, interactive/user-triggered, owned by `skills/dashboard/app/src/lib/config.ts`) and `routines` (`RoutineDef`, scheduled/artifact-gated, owned by `@coderails/dashboard-lib`) under one `DashboardConfig`. A `RoutineDef` names either a `skillCommand` directly or a `buttonRef` pointing at an existing `ButtonDef` — never both — validated by `validateRoutines()` at config-load time, which throws `ConfigError` on a duplicate routine name, on defining zero or both of `skillCommand`/`buttonRef`, or on a `buttonRef` matching no button `(verified, skills/dashboard/lib/src/config.ts)`. See [[routines]] for the full field-by-field contract (`cadence`, `expectedArtifact`, `escalation`).

## `buildArgv`: imported, not owned here

The one profile→flag mapping (`skills/dashboard/app/src/lib/argv.ts`, merged via PR #25) is not re-implemented anywhere in this contract. The Obsidian plugin imports it directly (esbuild-bundled); [[dashboard-runner]] imports the same function. This is the same discipline the button/run security model in [[dashboard]] already established for the web UI.

## Known caveats / edge cases

- **Not portable.** Nothing in the schema itself is machine-specific, but the launchd wiring that seeds routines into this queue (see [[routines]]) hard-codes absolute paths at authoring time.
- **No producer-namespacing in `queue/`.** Multiple producers (Obsidian, scheduler, and previously the send-gate before PR #62 split it out) share one flat directory keyed only by the intent's own filename (`<runId>.json` for the runner's producers) — collision-avoidance is by construction (each producer generates its own `runId`), not an enforced invariant. See [[queue-contract-cross-pr-audit_2026-07-07]] for the pre-existing version of this same observation about the (now-separated) approvals traffic.
- **`schema-compat.test.ts`'s `tsconfig.json` needs `"DOM"` in `lib`** purely because it type-imports through `obsidian/src/render.ts`, which uses DOM types — `dashboard-lib` itself has no runtime DOM dependency and never emits code (`noEmit: true`) `(verified, skills/dashboard/lib/README.md)`.

## See also

- [[dashboard-runner]] — the sole executor that claims intents from this queue
- [[routines]] — the scheduling convention that seeds routine intents into this same queue
- [[dashboard]] — sub-project 1; the Obsidian plugin and web button model that produce most intents
- [[assistant-link-send-gate-architecture]] — the sibling `approvals/` directory this contract is NOT
- [[pr_43-44-46_workflow-audit-queue-seam]] — the third (pre-existing, separate) writer into `queue/`, for skill-creation proposals
- [[queue-contract-cross-pr-audit_2026-07-07]] — flagged this sub-project's absence from the wiki before this cluster closed the gap
- [[pr_36-41-33-53-65_verified-routines]] — the source record for this page
