---
title: "PRs #55/#59/#60/#64/#66/#67 — Approve-click → skill-creator builder pipeline (loop 2)"
type: source
created: 2026-07-07
last_updated: 2026-07-08
sources: []
tags: [source, workflow-audit, dashboard, queue-contract, builder, skill-creator, security, agentic-os]
---

# PRs #55/#59/#60/#64/#66/#67 — Approve-click → skill-creator builder pipeline (loop 2)

Ingested by the loop-boundary wiki agent after all six PRs merged to `main` in
sequence. One source page for the cluster (not fragmented per-PR). This is
loop 2 of the workflow-audit/dashboard queue-mode arc — it turns the
Approve-button click documented in
[[pr_43-44-46_workflow-audit-queue-seam]] from a bare status flip into a real,
mechanically-gated build pipeline that authors a skill via `skill-creator` and
opens a PR for the owner to review and merge by hand.

## PR metadata

| Field | Value |
|---|---|
| PR #55 (L2-WU1) | "dashboard: approve-path hardening + builder spawn seam" — merge `905795c`, merged 2026-07-07T13:34:49Z |
| PR #60 (L2-WU2) | "dashboard: builder wrapper + injection-fenced prompt template" — merge `dd925fe`, merged 2026-07-07T14:29:55Z |
| PR #64 (L2-WU2 follow-up) | "dashboard: fix cwd-relative wrapper path + mechanically deny merge tools" — merge `7f289c2`, merged 2026-07-07T14:54:13Z |
| PR #59 (L2-WU4) | "docs: approve-build runner spec + seam/SKILL.md reconciliation" — merge `f92e61a`, merged 2026-07-07T13:49:55Z |
| PR #66 (L2-WU3) | "approve builder/wu3 visibility" — merge `330c5a2`, merged 2026-07-07T18:22:44Z |
| PR #67 (L2-WU5) | "approve builder/wu5 approvals dir" — merge `de19d95`, merged 2026-07-07T18:35:26Z |
| Repo | `blueman82/coderails` |
| Spec | `docs/coderails/specs/2026-07-07-approve-build-runner.md` (PR #59) |
| Builds on | [[pr_43-44-46_workflow-audit-queue-seam]]'s `QueueFileEntry` envelope and Approve/Deny action path; [[pr_62_10_approvals-dir-move]]'s `queue/`/`approvals/` split |

## Summary

Closes the "honesty requirement" gap [[pr_43-44-46_workflow-audit-queue-seam]]
explicitly named: clicking Approve on a `workflow-audit:propose-skill` queue
entry now actually triggers a build, not just a `pending`→`approved` field
flip. Five work units, each its own PR (numbered non-sequentially as L2-WU1
through WU5 since they interleaved with other loop-2 work): a claim-and-spawn
seam in the API route (WU1), a bash wrapper owning the build lifecycle state
machine plus an injection-fenced prompt template (WU2, with a same-day
follow-up fixing a real path bug and hardening the merge-denial), a normative
spec doc (WU4), dashboard build-state visibility including a real XSS fix
(WU3), and a directory-path correction after a concurrent PR split `queue/`
into `queue/`+`approvals/` (WU5).

## Architecture

1. **WU1 — approve-path hardening + spawn seam** (PR #55): `resolveQueueEntry`
   gains a pending-only transition guard — throws `QueueActionError` on
   approve-after-deny or double-approve, and returns the updated entry so the
   caller gets the exact approved bytes without a second read. `POST
   /api/queue` maps that violation to HTTP 409. For an approved entry with
   `toolName === "workflow-audit:propose-skill"`, the route now also calls a
   new `claimAndSpawnBuild(entry, deps)` (`src/lib/build/spawn.ts`) and echoes
   its result under a `build` response field; deny and non-matching
   `toolName`s are unaffected. `claimAndSpawnBuild` validates `proposed_name`
   against `^[a-z0-9][a-z0-9-]{0,63}$`, claims the hash via a non-recursive
   `mkdirSync` (EEXIST-safe against a concurrent claim race), writes
   `snapshot.json`/`prompt.md`/`state.json` sidecar files into
   `~/.claude/coderails-dashboard/builds/<hash>/`, and spawns the (at that
   point not-yet-existing) wrapper script detached + unref'd via a
   dependency-injected `spawnImpl`.
2. **WU2 — builder wrapper + injection-fenced prompt** (PR #60):
   `skills/dashboard/scripts/run-builder.sh` is a bash 3.2 wrapper owning the
   whole build lifecycle state machine for one approved entry: a global lock
   (with dead-PID staleness handling), deterministic hash re-validation
   *before any LLM runs*, matching the queue producer's exact canonicalisation
   convention, worktree setup off an asserted absolute repo path, a `claude
   -p` spawn with heartbeat + wall-clock watchdog, and a terminal state
   derived from artifacts (`pr_url` presence) rather than exit code alone. A
   guaranteed-terminal-state `EXIT` trap ensures `state.json` always lands in
   a terminal state even if an unrelated command (e.g. `git fetch`) fails
   mid-script. `src/lib/build/prompt.ts` is a typed prompt template: all six
   snapshot-derived (judge-authored, potentially adversarial) fields are
   confined to a single `untrusted-proposal-data` fence; every other line is
   static authored prose (D4 delivery gates, no-merge terminal,
   transcript-mining prohibition). Wired into `spawn.ts`, replacing WU1's
   placeholder `prompt.md` string.
3. **WU2 follow-up — path fix + mechanical merge denial** (PR #64), same day:
   (a) `route.ts` had resolved the wrapper path via
   `join(process.cwd(), "..", "scripts", "run-builder.sh")` — a production
   Next.js server's cwd is not guaranteed to be the app root, the same
   "prod-prerender" failure shape already recorded in project memory. Fixed
   with `resolveDefaultWrapperPath()`, which walks upward from the module's
   own `__dirname` (stable regardless of `process.cwd()`) to find the sibling
   script, mirroring `collect/markerVersions.ts`'s existing find-repo-root
   technique; returns `null` — never a fabricated guess — if not found, and
   `route.ts` reports a distinct `wrapper_not_found` error instead of crashing
   the route module. (b) The prompt's "never merge" clause, previously only a
   soft instruction, is now mechanically enforced: the `claude` spawn also
   passes `--disallowedTools "Skill(coderails:merge)" "Bash(gh pr merge*)"
   "Bash(*merge.sh*)"`. **Empirically verified live** (outside the test suite)
   that these flags produce a `permission_denials` entry for both a
   `Skill(coderails:merge)` attempt and a `gh pr merge` attempt, and that
   `--dangerously-skip-permissions` does not bypass the deny-list.
4. **WU4 — spec doc** (PR #59): `docs/coderails/specs/2026-07-07-approve-build-runner.md`
   is the normative contract — trigger condition, claim/spawn seam (cited
   against shipped WU1 code, not asserted from the design alone), sidecar
   schema, wrapper/prompt contract (cited as committed design, WU2 not yet
   merged at PR #59's own merge time), owner-merges rationale, a
   concurrency/retry table including the `invalid_name` dead end, a threat-model
   honesty note, and the E5 manual eval. Also clean-break rewrites
   [[pr_43-44-46_workflow-audit-queue-seam]]'s "Honesty requirement" section —
   the old "approval today is a status flip, not a trigger" claim is now false
   — and updates `workflow-audit/SKILL.md` section 8 to name `skill-creator`
   as the authoring engine (owner directive) with `writing-skills`
   RED/GREEN/REFACTOR as the headless stop condition. The four eval-pinned
   section 7 `AskUserQuestion`-gate sentences were verified byte-for-byte
   unchanged before and after.
5. **WU3 — dashboard build-state visibility** (PR #66): a new `collectBuilds`
   collector (`src/lib/collect/builds.ts`) mirrors `queue.ts`'s closed-set
   validation discipline — an out-of-vocabulary or missing `state` is
   rejected, never defaulted, and any unreadable/malformed file degrades to
   "skip this entry," never throws. `BuildEntry.state` is one of `claimed |
   queued | running | pr_open | failed`. `Snapshot.builds` surfaces this to
   the panel, which hash-joins each queue entry to its build sidecar and
   renders one of building / awaiting-your-merge / failed / builder-dead (on
   stale heartbeat). **Real security fix included**: `AssistantLinkPanel.tsx`
   adds `safePrUrl()` — `prUrl` is builder-session-controlled data
   (`state.json`, read verbatim from `run-builder.sh`'s own `gh pr create`
   output), not dashboard-generated, and rendering it unvalidated into an
   `<a href>` would let a `javascript:`/`data:` scheme execute on click; only
   an `https:`-protocol URL (checked via `new URL(prUrl).protocol`) is ever
   linked, anything else falls back to the plain-text CTA.
6. **WU5 — approvals dir path fix** (PR #67): a concurrent PR (#62, see
   [[pr_62_10_approvals-dir-move]]) split the send-gate's approval directory
   from `queue/` into a new sibling `approvals/`, landing after this loop's
   spec/writer work had already been drafted against the old single-directory
   assumption. This PR updates `write_queue_entry.sh`,
   `workflow-audit/SKILL.md`, and both loop-2 spec docs so workflow-audit's
   own proposals are written to `approvals/` (where the dashboard's
   Approve/Deny UI and this builder pipeline actually read from), not the
   stale `queue/` path.

## Key decisions

- **Owner merges the skill PR — a click approves creation, not landing an
  unseen diff.** The Approve button on the dashboard authorises the *build
  attempt*; a human still reviews and merges the resulting PR through the
  normal gate sequence. This is named explicitly in the spec's rationale
  section and is why WU2's mechanical merge-denial exists — the headless
  builder session must never be able to complete the loop on its own even if
  its prompt or a future compromise tried to instruct it to.
- **The wrapper owns the state machine, not the LLM.** `run-builder.sh`
  computes lock acquisition, hash re-validation, terminal state, and heartbeat
  entirely in bash, outside the LLM's control — the spawned `claude -p`
  process only authors the skill; it cannot forge success by exiting 0, since
  terminal state is derived from artifact presence (`pr_url`), and it cannot
  merge, since `--disallowedTools` blocks every merge-shaped invocation
  independent of the prompt's own instructions.
- **Injection fencing over trust.** All six judge-authored (and therefore
  adversary-reachable, per [[workflow-audit]]'s own threat model) proposal
  fields are confined to a single labelled fence in the prompt template; every
  other instruction is static authored prose the fenced data cannot rewrite.
- **`--disallowedTools` is empirically proven to survive
  `--dangerously-skip-permissions`**, not merely assumed to — PR #64's test
  plan records a live (non-test-suite) verification that both a
  `Skill(coderails:merge)` attempt and a `gh pr merge` attempt each produced a
  `permission_denials` entry with the skip-permissions flag also present.
- **E5 (a real, manual, one-time build) is the loop-close gate**, per the
  team-lead's brief. ✅ **DONE 2026-07-08** — see
  [[pr_89-100-104-106_approve-build-e5-live]]. Running one live Approve click
  end-to-end surfaced three production-only defects no test caught (builder
  repo-identity, launchd env, and the opaque-build feedback gap) and produced
  the pipeline's first real skill, [[verify-merged-pr]] (PR #104). The pipeline
  is no longer just wired-and-tested; it has built and merged a skill.

## Files changed

- `skills/dashboard/app/src/app/api/queue/route.ts` (PRs #55, #64)
- `skills/dashboard/app/src/lib/build/spawn.ts` (PRs #55, #60, #64)
- `skills/dashboard/app/src/lib/collect/queueActions.ts` (PR #55)
- `skills/dashboard/app/src/lib/build/prompt.ts` (PR #60, new)
- `skills/dashboard/scripts/run-builder.sh` (PR #60, new; hardened PR #64)
- `skills/dashboard/app/src/lib/collect/builds.ts` (PR #66, new)
- `skills/dashboard/app/src/lib/collect/index.ts`, `src/hooks/useDashboardState.ts`, `src/components/AssistantLinkPanel.tsx`, `src/styles/hud.css`, `src/app/api/events/route.ts` (PR #66)
- `docs/coderails/specs/2026-07-07-approve-build-runner.md` (PR #59, new; path fix PR #67)
- `docs/coderails/specs/2026-07-07-workflow-audit-queue-seam.md` (PR #59 Honesty-requirement rewrite; path fix PR #67)
- `skills/workflow-audit/SKILL.md` (PR #59 section 8 authoring-engine sentence; path fix PR #67)
- `skills/workflow-audit/scripts/write_queue_entry.sh` + its test (PR #67, `queue/` → `approvals/`)
- Test files: `buildSpawn.test.ts`, `queueActions.test.ts`, `queueRoute.test.ts`, `builderPrompt.test.ts`, `runBuilder.test.ts`, `AssistantLinkPanel.test.ts`, `builds.test.ts`, `events.test.ts`, `useDashboardState.test.ts`

## Wiki pages updated

- [[workflow-audit]] — queue-mode section's "approval today is a status flip
  only" claim corrected to reflect the real build trigger
- [[dashboard]] — AssistantLinkPanel section extended with build-state
  visibility and the `safePrUrl` XSS fix
- [[pr_43-44-46_workflow-audit-queue-seam]] — superseded-claim note added
  pointing at this page (its own "Honesty requirement" section is now
  historically accurate-for-its-time, not current state)
- [[assistant-link-send-gate-architecture]] — no content change needed; its
  `approvals/` path description (from [[pr_62_10_approvals-dir-move]]) is
  unaffected by and consistent with this cluster's WU5 fix

## Caveats / gotchas

- **E5 (manual, one-time real build through the pipeline) has since run** —
  every claim above was, at this ingest, about the mechanism being wired and
  tested, not a proposal actually built and PR'd through it live. That gate
  closed on 2026-07-08: [[pr_89-100-104-106_approve-build-e5-live]] records the
  live run, the three production-only defects it surfaced, and the first built
  skill [[verify-merged-pr]]. `(verified — PR #104 is a real merged skill PR
  from an Approve-spawned build)`
- **WU2's follow-up (PR #64) fixes a bug that shipped in WU2 itself the same
  day** — the cwd-relative wrapper path — rather than being caught before
  merge. Recorded here rather than silently folded into "WU2" so the
  same-day self-correction is visible. `(verified — PR #64's own description
  names PR #60/L2-WU2 as what it's fixing)`
- **PR numbering is non-sequential and re-used across unrelated PRs in this
  repo's history** (as already flagged for the #43/#44/#46 cluster) — the
  work-unit labels (L2-WU1..WU5) are the more reliable ordering signal than
  the PR numbers themselves, since WU3 (#66) and WU5 (#67) merged hours after
  WU1/WU2/WU4 (#55/#59/#60/#64). `(verified — merge timestamps above)`
- **The merge-denial hardening (PR #64) is scoped to the specific tool
  patterns tested** (`Skill(coderails:merge)`, `Bash(gh pr merge*)`,
  `Bash(*merge.sh*)`) — it is a `--disallowedTools` allowlist-complement, not
  an exhaustive block of every conceivable merge-shaped command, matching the
  same "enumerated families, not exhaustive" honesty pattern documented for
  `destructive_bash_gate` in `AGENTS.md`. `(inferred — pattern-matched against
  the documented enforcement-ceiling convention; not separately confirmed
  against PR #64's diff for a broader denial list)`
