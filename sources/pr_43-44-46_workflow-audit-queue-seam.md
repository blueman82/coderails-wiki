---
title: "PR #43, #44, #46 — workflow-audit × dashboard queue-mode integration"
type: source
created: 2026-07-07
last_updated: 2026-07-07
sources: []
tags: [source, workflow-audit, dashboard, queue-contract, routines, agentic-os]
---

# PR #43, #44, #46 — workflow-audit × dashboard queue-mode integration

Ingested by the loop-boundary wiki agent after all three PRs merged to `main`
in sequence. One source page for the cluster (not fragmented per-PR). Note:
these PR numbers coincidentally reuse numbers already taken by unrelated,
earlier hook-hardening PRs in this same repo — see [[pr_43_rough-edges]],
[[pr_44_no-edit-plugin-source]], [[pr_46_gate-git-push-on-main]] for those
(different feature, different merge SHAs, no relation to this cluster).

## PR metadata

| Field | Value |
|---|---|
| PR #43 (WU1) | "workflow audit dashboard/wu1 queue writer" — merge `f1212a31`, merged 2026-07-07T10:56:34Z |
| PR #44 (WU2) | "dashboard: readable workflow-audit proposal preview (WU2)" — merge `0468bc90`, merged 2026-07-07T11:09:15Z |
| PR #46 (WU3) | "workflow audit dashboard/wu3 seam doc" — merge `675e63bc`, merged 2026-07-07T11:17:36Z |
| Repo | `blueman82/coderails` |
| Builds on | The generic `QueueFileEntry` envelope frozen by sub-project 4's design spec (`2026-07-06-assistant-link-panel-design.md`) and PR #31's Approve/Deny button — see [[queue-contract-cross-pr-audit_2026-07-07]] |

## Summary

Gives the [[workflow-audit]] skill a second, asynchronous approval surface on
the [[dashboard]]'s existing approval queue, alongside its original synchronous
in-session `AskUserQuestion` gate (unchanged, still the primary path). Three
work units, each its own PR: a tested writer script (WU1), a readable render
branch in the dashboard's existing panel (WU2), and a normative consumption-seam
contract for the not-yet-built routines runner (WU3). This closes Gap #2 named
in [[queue-contract-cross-pr-audit_2026-07-07]] ("WU1/WU2/WU3... not yet
started, not yet PR'd") — that investigation's open item is now shipped.

## Architecture

1. **WU1 — the writer** (`skills/workflow-audit/scripts/write_queue_entry.sh`,
   PR #43): reads one judge-contract verdict object on stdin; on
   `verdict:"propose"` writes exactly one `<hash>.json` `QueueFileEntry` file
   and prints the bare hex hash to stdout; on any other verdict (including
   `"reject"`) writes nothing and exits 0 silently — the script itself refuses
   to persist a reject-verdict row even if called on one directly. `toolInput`
   is built by explicit `jq -n` field construction from exactly six
   D2-whitelisted fields (`cluster_ngram`, `count`, `sessions`, `task_summary`,
   `proposed_name`, `proposed_description`) — a structural whitelist, not a
   filter, so a stray field on the piped verdict is dropped, never copied
   through. `toolName` is hardcoded to `"workflow-audit:propose-skill"`.
   `hash = sha256(jq -S -c <toolInput>)`, documented as canonically equivalent
   to assistant-agent's `sendGate.ts` `hashInput`/`sortKeysDeep` recipe for
   this flat (no-nested-object) shape, with one named divergence (`jq`
   backslash-`u`-escapes DEL/codepoint 127 where JS `JSON.stringify` emits it
   raw — accepted as exceedingly unlikely in LLM-generated prose, not worked
   around). Queue dir created `0700`, entry file `0600`. `SKILL.md` gained a
   new "5. Queue-mode output (optional)" section wiring this in: additive to
   the interactive gate (section 7), never a replacement; every invariant from
   section 7 — stale-approval caution, zero-approvals-is-success, D2-whitelist
   content only — explicitly restated as applying to a queue entry too.
2. **WU2 — the render branch** (`AssistantLinkPanel.tsx` + `hud.css`, PR #44):
   a type-guarded render branch recognises `toolName ===
   "workflow-audit:propose-skill"` and displays the proposal readably
   (proposed name / description / task summary / session count) instead of
   the previous opaque `JSON.stringify`-and-truncate fallback. Any other
   `toolName` still falls back to the prior opaque render — the branch is
   additive, not a replacement of the general case. The Approve/Deny buttons
   and `POST /api/queue` path (both from PR #31) are untouched by this PR;
   only the *preview* changed, not the action.
3. **WU3 — the seam contract** (`docs/coderails/specs/2026-07-07-workflow-audit-queue-seam.md`,
   PR #46): the normative consumption contract for the not-yet-built routines
   runner. Filter: `status === "approved" && toolName ===
   "workflow-audit:propose-skill"`. Mandatory re-validation before acting:
   recompute `sha256(jq -S -c <toolInput>)` (or the equivalent
   `sortKeysDeep`+`JSON.stringify`+sha256 recipe) over the entry's own
   `toolInput` and compare against the stored `hash` — a mismatch is a
   distinct logged rejection (`hash_mismatch:<hash>`), never a soft warning,
   never grounds to proceed on a "close enough" match. An entry that fails
   `parseQueueEntry` (missing field, wrong type, unrecognised status) is a
   separately logged rejection (`unparseable_entry:<filename>`) — both
   categories are rejections, not creates, and neither is swallowed silently.
   Only on successful re-validation does the runner drive
   `coderails:writing-skills`'s full RED/GREEN/REFACTOR process for the
   proposed skill, landed via its own branch and the full gate sequence —
   never a direct commit to `main`, never written to a user's personal
   `~/.claude/skills`.

## The "Honesty requirement" — read before assuming a pipeline exists

WU3's spec has an explicit section by this name, because the natural reading
of "workflow-audit now has a dashboard Approve button" overstates what's
built. **As of this cluster, clicking Approve on a
`workflow-audit:propose-skill` queue entry changes only that entry's `status`
field, in place, from `"pending"` to `"approved"`. Nothing else happens.** No
skill is created, no branch opens, no PR is filed, by any code that exists
today — the Approve button's wiring and the underlying `resolveQueueEntry` /
`POST /api/queue` path are unchanged by this cluster; they already existed,
generically, for every `toolName`, from PR #31. Skill creation only happens
once the routines runner performs the filter → re-validate → create sequence
above. An approved entry today is a recorded, hash-bound approval sitting in
`~/.claude/coderails-dashboard/queue/`, waiting — a deliberate seam-design
decision (the integration's own loop spec, decision C, "Consumption:
seam-only"), not an unfinished accident. Zero approvals, or approvals that sit
unconsumed indefinitely, are both valid, complete states: no nag, no retry, no
implicit timeout converts a stale approval into a create.

## Key decisions

- **Reuse the existing `QueueFileEntry` envelope verbatim, no new schema.**
  The generic five-field envelope (`hash`, `toolName`, `toolInput: unknown`,
  `createdAt`, `status`) frozen by sub-project 4's design spec is reused
  as-is; this cluster adds exactly one new `toolName` value
  (`"workflow-audit:propose-skill"`), not a parallel mechanism. Mirrors the
  same envelope send-gate already uses — see
  [[queue-contract-cross-pr-audit_2026-07-07]]'s "one queue file format, two
  independent producers" finding, now a third producer joining the same two.
- **Producer is a tested helper script invoked post-judge, never the frozen
  pipeline scripts.** `write_queue_entry.sh` sits downstream of
  `scan_transcripts.sh` / `cluster_ngrams.sh` / the judge subagent — those
  three stay untouched; the D2 privacy whitelist they already enforce is
  re-applied at the queue-writing boundary via the same six-field structural
  whitelist, not re-derived independently.
- **Seam-only consumption, no stopgap poller, pre-authorised by the owner's
  loop handoff.** WU3 explicitly declines to build any poller or consumer
  process — "no live process is waiting at the moment a dashboard Approve
  click happens, so there is nothing for a poller to poll until the routines
  runner exists." This mirrors the same interim-seam pattern the dashboard's
  Obsidian plugin used before the routines sub-project existed (see
  [[dashboard]]'s "Obsidian command centre" section).
- **D2/D5 invariants carried into queue mode verbatim, not re-derived.** The
  section 5 addition to `SKILL.md` explicitly restates that every pinned
  invariant from the interactive-gate section (stale-approval caution,
  zero-approvals-is-success, D2-whitelist-only content, never committed to a
  repo) applies identically to a queue entry — queue mode is a second
  surface for the same gate, not a weaker one.

## Files changed

- `skills/workflow-audit/scripts/write_queue_entry.sh` + `scripts/tests/write_queue_entry.test.sh` + `scripts/tests/fixtures/queue-proposal.json` (PR #43)
- `skills/workflow-audit/SKILL.md` — new "5. Queue-mode output (optional)" section (PR #43)
- `skills/dashboard/app/src/components/AssistantLinkPanel.tsx` + `src/styles/hud.css` + `app/test/AssistantLinkPanel.test.ts` (PR #44)
- `docs/coderails/specs/2026-07-07-workflow-audit-queue-seam.md` (PR #46)

## Wiki pages updated

- [[workflow-audit]] — queue-mode section added
- [[dashboard]] — AssistantLinkPanel readable-render branch cross-referenced
- [[queue-contract-cross-pr-audit_2026-07-07]] — Gap #2 marked closed, third producer noted

## Caveats / gotchas

- The routines runner itself is still unbuilt — WU3 is a contract for code
  that doesn't exist yet, not an implementation. `(verified — spec's own
  "Consumption seam contract... not built by this document" framing)`
- The `hash` canonicalisation has one documented, accepted divergence between
  `jq -S -c` and JS `sortKeysDeep`+`JSON.stringify` for a literal DEL
  (codepoint 127) byte — judged unreachable in practice for LLM-generated
  text, not worked around. `(verified — script header comment)`
- This cluster's PR numbers (#43, #44, #46) collide with unrelated pre-existing
  wiki source pages using the same numbers for an earlier, different feature
  (hook-hardening rough-edges / no-edit-plugin-source / git-push-gate). No
  content relationship between the two; flagged here only to prevent a future
  reader conflating them. `(verified — checked existing sources/ directory
  before naming this page)`
