---
title: "Investigation: Why skills/dashboard/ is on TIER_GATE_PATH_DENYLIST"
type: investigation
created: 2026-07-21
last_updated: 2026-07-21
sources:
  - scripts/tier-gate/tier-gate-runner.sh
  - scripts/tier-gate/judge-prompt.md
  - skills/task-evals/SKILL.md
  - sources/pr_232_tier-review-gate.md
  - design/enforcement-model.md
  - skills/dashboard.md
tags: [tier-gate, denylist, self-edit-leash, dashboard, enforcement, security]
---

# Why `skills/dashboard/` is on `TIER_GATE_PATH_DENYLIST`

Answers a wiki query asking why the dashboard sits on the tier-gate path
denylist, what the self-edit leash protects against, and why the list is
not just `scripts/tier-gate/`.

## The constant

```
TIER_GATE_PATH_DENYLIST='^(skills/dashboard/|launchd/|scripts/tier-gate/|\.github/workflows/)'
```

(`scripts/tier-gate/tier-gate-runner.sh:768`)

## It is not primarily a self-edit list

The framing "self-edit denylist" is a slight misread of the code, and the
distinction is what answers the dashboard question.

The constant's own header comment (lines 760-768) says the list is paths
"that are **outward-facing or irreversible surfaces by construction** (tier-0's
own predicate names these as automatically disqualifying — see
judge-prompt.md)." That is its primary identity: a mechanical, model-free
restatement of the tier-0 exemption predicate. A tier-0 claim means "single
work-unit, no outward/irreversible surface, already covered by a test"
([[task-evals]] SKILL.md:67). Anything under these prefixes contradicts that
predicate by construction, so the claim can be refused without ever paying
for a judge call.

The regex has **two independent consumers**:

1. **`tg_prefilter`** (line 797) — the mechanical pre-filter. Blocks with
   `verdict=illegitimate`.
2. **The self-edit leash in `tg_gate_pr`** (line 918) — runs *earlier*, before
   any judging logic, and blocks with the more specific `verdict=self_edit`.
   Its comment states plainly that it "**Reuses** `TIER_GATE_PATH_DENYLIST` —
   the same paths `tg_prefilter`'s own denylist check would eventually hit"
   but greps it directly so the posted status distinguishes a leash violation
   from "diff too big" or "content doesn't match the claimed tier."

The leash comment names only `scripts/tier-gate/` as "the daemon's own
source." It never describes the dashboard as self-modifying source. The leash
has no list of its own — it borrows the outward-surface list wholesale, and
the dashboard rides along.

## Chronology confirms the borrow direction

- `cc5c570` (2026-07-17, "Fix tier-gate judge injection") introduced
  `tg_prefilter` **and** the denylist, with `skills/dashboard/` present from
  the first line of its existence. That commit's message describes Fix 2 as a
  "mechanical path-denylist + file/line-count cap before any model call" — no
  mention of a self-edit leash.
- `e8bdd6c` ("tier-gate: replace post-nothing with attest-all + self-edit
  leash") introduced the leash *later*, and `ae929ab` ("judge every tier")
  reworked it.

So the dashboard was denylisted for the outward-surface reason **before the
self-edit leash existed**, and was then swept into the leash when the leash
chose to reuse the existing regex rather than define a narrower one.

## Why the dashboard is an outward surface

`skills/dashboard/` is a 160-file Next.js application (`git ls-files` count),
not a markdown skill. It is a served web HUD with `POST /api/run`,
`/api/queue`, `/api/events`, and `/api/run/output` route handlers. Per
[[dashboard]]'s "Security posture" section, `POST /api/run` and the
workflow-audit Approve/Deny queue are an **unauthenticated command-execution
surface**, LAN-reachable since PR #179. Per [[task-evals]], anything "a human
sees or interacts with (a UI, CLI output, a rendered artifact, a served
endpoint)" is an outward surface, making such a change **minimum tier 1**. A
tier-0 claim on a dashboard diff is therefore false by definition — exactly
what a mechanical pre-filter should catch for free.

The other two entries fit the same pattern rather than a self-edit one:
`launchd/` (daemon lifecycle, an irreversible machine-state surface) and
`.github/workflows/` (CI execution, both outward and privileged).

## Why the shared list is defensible as a leash anyway (inferred)

This is synthesis, not something the code comments state. The dashboard
happens to sit **inside the enforcement trust domain**, so treating it under a
self-edit leash is not merely an accident of regex reuse:

- The dashboard's run route is the **sole set-site** of
  `CODERAILS_HEADLESS_RUN=1`, the flag that makes `check_confidence_labels.sh`
  and `check_verify_loop.sh` skip Stop-event enforcement
  ([[enforcement-model]], [[discipline-loop]]).
- `AGENTS.md` explicitly flags any PR introducing a **second** set-site as a
  security finding, not a legitimate extension.

So an agent editing `skills/dashboard/app/src/app/api/run/route.ts` could in
principle widen the discipline-hook exemption — a change to the enforcement
machinery a tier-0 exemption must never carry. But the documented reason the
path is on the list is the outward-surface predicate, not this.

## Consequence: PR #245

Because the leash blocks at every tier and cannot be tier-exempted, any PR
touching `scripts/tier-gate/` posts `verdict=self_edit` before judgement.
PR #245 hit exactly this and required human force-merge/close — correct by
design, not a bug. The same holds for a dashboard PR: the daemon will post
`self_edit` on it regardless of the claimed tier.

## Known ceiling

The tier is **self-declared** by the PR's own eval artifact. The denylist is
mechanically derived from the file list (never from defendant prose), so it
resists a forged tier marker — but it only covers these four prefixes. A
dishonest claim on a non-denylisted path still depends on the judge. See the
memory-parked "tier-gate SELF-DECLARED TIER BYPASS" arc.

## See also

- [[pr_232_tier-review-gate]] — source record for the gate, prefilter, and
  verdict-laundering closures
- [[enforcement-model]] — trust-boundary framing for the headless exemption
- [[dashboard]] — the unauthenticated exec surface and LAN posture
- [[task-evals]] — the tier predicates the denylist mechanises
