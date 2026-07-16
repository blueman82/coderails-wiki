---
title: "Queue contract cross-PR audit — workflow-audit, dashboard approval pipeline, send-gate"
type: investigation
created: 2026-07-07
last_updated: 2026-07-16
sources:
  - docs/coderails/specs/2026-07-06-assistant-link-panel-design.md  # deleted by PR #138 (2026-07-11); cited as historical provenance
  - sources/pr_138_remove-specs-plans-tracking.md
  - sources/pr_27-39_workflow-audit-skill.md
  - sources/pr_25_observability-dashboard.md
  - sources/pr_43-44-46_workflow-audit-queue-seam.md
  - sources/pr_62_10_approvals-dir-move.md
  - sources/pr_36-41-33-53-65_verified-routines.md
tags: [investigation, queue-contract, dashboard, workflow-audit, send-gate, assistant-agent, gaps, approvals]
---

> ✅ **UPDATE (2026-07-07, later same day):** The WU1/WU2/WU3 work this
> investigation flagged as "not yet started, not yet PR'd" (see "workflow-audit's
> current approval mechanism is NOT the queue — yet" below) has since shipped
> as PRs #43/#44/#46, closing Gap #2's queue-mode half. See
> [[pr_43-44-46_workflow-audit-queue-seam]] for the full record. The findings
> below are left as-written (an accurate point-in-time record); only this
> banner and the Gaps section's status lines are new.
>
> ⚠️ **PATH UPDATE (2026-07-07, still later same day):** The send-gate's own
> approval files (`gate/surfaces/queue.ts`, `DEFAULT_QUEUE_DIR`) moved from
> `~/.claude/coderails-dashboard/queue/` to a new sibling directory,
> `.../approvals/`, in [[pr_62_10_approvals-dir-move]] — fixing a collision
> where the routines runner sweep and workflow-audit's proposal writer (both
> still in `queue/`, item 2 below) would quarantine/reject a `QueueFileEntry`
> approval file as malformed input. Every `~/.claude/coderails-dashboard/queue/`
> path below describing **the send-gate's own writes/reads** (producer 1,
> `api/queue/route.ts`'s `DEFAULT_QUEUE_DIR`) is now stale by that one word;
> every `queue/` reference describing **workflow-audit's proposal path**
> (producer 2) is still accurate — that seam did not move. See
> [[pr_62_10_approvals-dir-move]] for the full record.
>
> ✅ **UPDATE (2026-07-07, later still):** Gap #3 below (PR #36 / the routines
> sub-project having no wiki page) is now **closed** — see
> [[pr_36-41-33-53-65_verified-routines]] and its four new pages
> ([[intent-queue-runner-contract]], [[dashboard-runner]], [[routines]],
> [[memory-consolidation]]). The routines *runner* this investigation
> described as the still-unbuilt consumer of WU3's consumption-seam contract
> is now built and documented. Left as-written below; only this banner and
> the Gap #3 status line are new.

# Queue contract cross-PR audit — workflow-audit, dashboard approval pipeline, send-gate

Filed in response to a wiki-query spanning three things that turned out to be
more entangled (and less documented) than the query assumed: the
`workflow-audit` skill, "the dashboard approval pipeline / queue contract (PR
#31)", and send-gate integration. Two of the three live in different
repos/wikis than expected, and PR #31 has **no wiki page at all** — this
investigation exists to close that gap and record what's actually true across
all three.

## The headline finding: one queue file format, two independent producers

`~/.claude/coderails-dashboard/queue/<hash>.json` is a single on-disk contract
(`QueueFileEntry`: `hash`, `toolName`, `toolInput: unknown`, `createdAt`,
`status: "pending"|"approved"|"denied"`) that is now written by **two
unrelated systems**, and read/resolved by **one** dashboard component that
treats `toolInput` fully opaquely (never destructured — `JSON.stringify` +
truncate only). That opacity is exactly what lets both producers share one
consumer safely:

1. **assistant-agent's send-gate** (`gate/surfaces/queue.ts`,
   `DEFAULT_QUEUE_DIR` = the same path) — writes a `pending` entry when a
   gated Slack/Calendar send tool is called, then polls the same file for an
   externally-flipped `status`. This is one of three approval surfaces
   documented in [[capabilities/send-gate]] (assistant-agent-wiki) — terminal,
   Telegram, and this dashboard queue file.
2. **workflow-audit's queue-mode** (at time of this query: not yet built — see
   Gaps below; **shipped later the same day as PRs #43/#44/#46**, see
   [[pr_43-44-46_workflow-audit-queue-seam]]) — a *third* writer for
   skill-creation proposals, reusing the identical file shape rather than
   inventing a parallel mechanism.

**Nothing in either the design spec or the shipped code prevents hash
collision or cross-talk between the two producers** beyond the fact that
`hash` is a SHA-256 of canonicalised `toolInput`, so a workflow-audit proposal
and a send-gate tool-call would only collide if their inputs happened to
canonicalise identically — vanishingly unlikely, but not a designed invariant,
just an emergent one. Worth naming explicitly if a third producer is ever
added: the shared directory has no producer-namespacing (e.g. a prefix), just
accidental collision-avoidance via hash entropy.

## PR #31 — the Approve/Deny button (undocumented in this wiki until now)

`PR #31` ("feat/wu5 approve button", merged `6fa5e34`, 2026-07-06) is real,
merged, and **had no `pr_31_*.md` source page and no cross-reference from
either [[dashboard]] or [[workflow-audit]]** before this investigation. It
shipped exactly the piece the design spec (below) called "deferred":

- `skills/dashboard/app/src/lib/collect/queue.ts` — `collectQueue(queueDir,
  limit=50)`, modeled on `collectMemoryTrail`'s degrade-to-`[]`-on-any-error
  idiom, sorts newest-first.
- `skills/dashboard/app/src/lib/collect/queueActions.ts` —
  `resolveQueueEntry(queueDir, hash, decision)`, the **sole writer** of the
  approved/denied transition. Validates `hash` against `/^[0-9a-f]{64}$/`
  before any `join()` — explicit path-traversal defense, since
  `join(queueDir, hash + ".json")` alone does not stop a `../` segment.
- `skills/dashboard/app/src/app/api/queue/route.ts` — `POST /api/queue`,
  token + Origin/Host guarded (mirrors `/api/run`'s pattern exactly),
  `DEFAULT_QUEUE_DIR = ~/.claude/coderails-dashboard/queue`.
- `AssistantLinkPanel.tsx` — renders only the "pending queue" slice of the
  4-item ASSISTANT.LINK panel spec (item 3, "Sends + approvals log" is
  partially covered; the JSONL audit-log half of item 3, plus items 1/2/4,
  are still unbuilt — see Gaps).

This closes the round-trip the design spec (next section) left as "deferred
work" — the queue contract is now fully bidirectional in code, not just on
the writer side.

## The normative contract: `docs/coderails/specs/2026-07-06-assistant-link-panel-design.md`

> ⚠️ **File removed since this investigation was filed.** `docs/coderails/specs/`
> was gitignored and its contents deleted by [[pr_138_remove-specs-plans-tracking]]
> (2026-07-11), so the spec called "normative" below no longer exists in the repo.
> This page is a point-in-time record and is left as written; for the durable
> contract see [[intent-queue-runner-contract]] and
> [[assistant-link-send-gate-architecture]].

This spec (sub-project 4 of 5 in the agentic-OS sequence) is the **source of
truth for the `QueueFileEntry` shape**, and it says so explicitly and
unusually directly: at the time it was written, assistant-agent's
`gate/surfaces/queue.ts` did not yet exist, so *this coderails-repo spec
document is normative over the assistant-agent implementation* — "WU2's
`queue.ts` must conform to the shape defined here, not the reverse... this
file should be updated only if the owner explicitly re-resolves the contract,
not silently patched to match an implementation that drifted."

Cross-checked against the actual shipped `gate/surfaces/queue.ts`
(assistant-agent repo) field-by-field: **matches verbatim** — same five
fields, same names, same types, same status enum. No drift found `(verified)`.

The spec also defines the full 4-item ASSISTANT.LINK panel (tasks,
email-last-checked, sends+approvals log, routine-runs slot) — only item 3's
pending-queue half is built (via PR #31's `AssistantLinkPanel`). See Gaps.

## workflow-audit's current approval mechanism is NOT the queue — yet

`skills/workflow-audit/SKILL.md` (as shipped by PR #27/#30/#37/#39, see
[[pr_27-39_workflow-audit-skill]] and [[workflow-audit]]) uses a **synchronous
in-session `AskUserQuestion`** hard-stop gate today, not the dashboard queue.
The pending task list surfaced at the start of this query —

- WU1: queue-mode proposal output for workflow-audit
- WU2: dashboard collector/tile rendering + Approve wiring
- WU3: approved-entry consumer → writing-skills create path

— is a **not-yet-started, not-yet-PR'd** effort to give workflow-audit a
*second* approval path that writes into the same queue directory PR #31 (and
send-gate) already use, so an owner can approve a skill-creation proposal
from the dashboard instead of only in a live session. `AskUserQuestion`
appears to remain the primary/default gate; queue-mode is additive, per the
task names ("proposal output" / "Approve wiring" / "consumer"), not a
replacement. This is inferred from the task list plus the fact that
`SKILL.md`'s "Approval gate — hard stop, no exceptions" section only
describes `AskUserQuestion` today (verified by reading the file directly, no
mention of queue/dashboard mode as of `654df98`).

## Where each piece actually lives (repo/wiki mismatch to watch for)

The query bundled three things that live in genuinely different places —
worth naming so a future query doesn't waste a pass assuming co-location:

| Piece | Lives in | Wiki page |
|---|---|---|
| `workflow-audit` skill | `coderails` repo | [[workflow-audit]] + [[pr_27-39_workflow-audit-skill]] (this wiki) |
| Dashboard skill + queue collector/API/button (PR #25, #31) | `coderails` repo (`skills/dashboard/`) | [[dashboard]] + [[pr_25_observability-dashboard]] (this wiki) — **PR #31 itself had no page until this investigation** |
| Queue contract spec (normative) | `coderails` repo, `docs/coderails/specs/2026-07-06-assistant-link-panel-design.md` | not yet a dedicated wiki page (see Gaps) |
| send-gate | `assistant-agent` repo (`gate/sendGate.ts`, `gate/surfaces/queue.ts`) | `capabilities/send-gate.md` in the **assistant-agent-wiki**, a separate sibling vault — not in coderails-wiki at all |
| "routines" sub-project 2 (`@coderails/dashboard-lib`, PR #36) | `coderails` repo | [[intent-queue-runner-contract]] + [[dashboard-runner]] + [[routines]] + [[memory-consolidation]] — closed, see Gap #3 |

## Gaps found (things the plan assumes but aren't enforced/documented in code)

1. ~~**PR #31 has no source page in coderails-wiki.**~~ **CLOSED 2026-07-07**
   — [[pr_31_assistant-link-approve-button]] filed (alongside
   [[pr_28_assistant-link-queue-contract-and-panel-spec]] for PR #28), closing
   the process gap this investigation originally flagged (the button/run
   model's *design* was documented in [[dashboard]], but the queue-specific
   Approve/Deny piece itself had no source record until now).
2. ~~**WU1/WU2/WU3 queue-mode work not yet PR'd.**~~ **CLOSED 2026-07-07** —
   shipped as PRs #43 (writer), #44 (dashboard render), #46 (consumption-seam
   contract). See [[pr_43-44-46_workflow-audit-queue-seam]]. The PR #36
   routines/dashboard-lib gap immediately below is still open — the routines
   *runner* itself (the consumer of the seam WU3 defines) remains unbuilt.
3. ~~**PR #36 ("routines/wu1 dashboard-lib", merged `027c09f`) has no wiki page
   either.**~~ **CLOSED 2026-07-07** — the full sub-project 2 cluster
   (PRs #36/#41/#33/#53/#65) shipped and is now documented:
   [[pr_36-41-33-53-65_verified-routines]] (source page),
   [[intent-queue-runner-contract]] (the `Intent`/`RoutineDef` schema and
   queue lifecycle), [[dashboard-runner]] (the sweeper, artifact gate,
   escalation taxonomy), [[routines]] (cadence/config/launchd), and
   [[memory-consolidation]] (one shipped routine's skill). The runner that
   [[pr_43-44-46_workflow-audit-queue-seam]]'s WU3 consumption-seam contract
   describes is this same runner — it is now built, though WU3's specific
   consumption of *approved workflow-audit proposals* is not independently
   confirmed here to be wired up (the runner executes routine buttons; it
   was not verified during this closure to special-case
   `workflow-audit:propose-skill` queue entries).
4. **ASSISTANT.LINK panel is only 1 of 4 items built**, and even that item
   (3, "sends + approvals log") only has the pending-queue half — the JSONL
   audit-log tail (`gate/auditLog.ts`) rendering half of item 3 is not
   confirmed built in the code reviewed here. Items 1 (tasks), 2 (email last
   checked), and 4 (routine-runs) remain explicitly deferred per the spec's
   own language, each blocked on a different unbuilt dependency (an empty
   `tasks/*.md` convention, an unbuilt secretary state file, and sub-project 2
   respectively).
5. **No producer-namespacing in the queue directory.** As above — three
   producers now (send-gate, PR #31's own generic path, and
   workflow-audit queue-mode as of [[pr_43-44-46_workflow-audit-queue-seam]])
   share one flat directory keyed only by content hash. Not a bug today, but a
   documented assumption (hash entropy avoids collision) rather than an
   enforced invariant (e.g. a producer-tagged filename prefix). Worth a code
   comment or spec note if a fourth producer is ever added.
6. **No stale-entry cleanup exists.** The spec explicitly scopes this out
   ("A future cleanup routine (unbuilt, out of scope here) may prune stale
   `pending` files past some age") — confirmed still true, no cleanup script
   found in either repo during this investigation. Reaffirmed as a non-goal in
   [[pr_43-44-46_workflow-audit-queue-seam]]'s WU3 spec too.
7. **`workflow.config.yaml`'s `wiki_path` is still `null`** in the main repo
   (noted originally in [[pr_27-39_workflow-audit-skill]]'s caveats) — this
   wiki's location keeps being resolved via fallback path convention, not
   config. Repeated here because it's the second investigation in a row to
   note the same unresolved null.

## Superseded / one-directional decisions worth remembering

- The design spec is explicit that it is **not** to be "silently patched to
  match an implementation that drifted" — if `gate/surfaces/queue.ts` ever
  diverges from the spec's `QueueFileEntry` shape, that is a bug in the
  assistant-agent side, and the fix is to correct the code, not update the
  spec, unless the owner explicitly re-resolves the contract. This is an
  unusual and deliberate one-way normativity — worth citing precisely if a
  future PR proposes "fixing" the spec to match observed drift instead of
  fixing the drift.
- The Approve/Deny button was explicitly deferred in the spec pending PR #25
  merging first; PR #31 is that deferred follow-up, confirmed delivered.

## Related

- [[dashboard]] — sub-project 1; button/run security model, Obsidian
  command-centre queue-writing interim path
- [[workflow-audit]] — sub-project 3; current `AskUserQuestion` gate plus, as
  of [[pr_43-44-46_workflow-audit-queue-seam]], the queue-mode second surface
  this investigation originally found pending (WU1–WU3, not yet PR'd as of
  this writing — since closed)
- [[pr_25_observability-dashboard]] / [[pr_27-39_workflow-audit-skill]] /
  [[pr_43-44-46_workflow-audit-queue-seam]] — source records for the two
  skills plus this cluster's queue-mode follow-up
- [[pr_36-41-33-53-65_verified-routines]] — source record for sub-project 2
  (routines), which closed Gap #3 below
- assistant-agent-wiki `capabilities/send-gate.md` — the gate that
  originates one of the two queue producers (separate vault, not
  cross-linkable via `[[...]]` from here)
