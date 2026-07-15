---
title: "PR #184/#185/#186 — per-loop cost & model tracking (retro.json schema_version 2, dashboard cost tile)"
type: source
created: 2026-07-15
last_updated: 2026-07-15
sources: []
tags: [source, retro-json, phase-13, cost-tracking, token-usage, model-prices, dashboard, loop-cost-miner, schema-version-2]
---

# PR #184/#185/#186 — per-loop cost & model tracking

Three-PR cluster (2026-07-15) that closes the gap
[[retro-json-per-model-cost-tracking-gap_2026-07-15]] documented the same day:
coderails had no cost or per-model token tracking anywhere. This cluster adds
a fail-open token/cost miner, wires it into the agentic-loop Phase 13
teardown as `retro.json` `schema_version` 2, and rolls the frozen numbers up
into two new dashboard KPI tiles.

## PR metadata

| Field | Value | Merged |
|---|---|---|
| #186 `feat/loop-cost-miner` | `hooks/scripts/lib/loop_cost.sh` + `model_prices.json`; widens the retro `schema_version` gate to `>= 1` | 2026-07-15 18:45 |
| #184 `docs/loop-cost-skill` | `skills/agentic-loop/SKILL.md` Phase 13 + `AGENTS.md` lockstep | 2026-07-15 18:56 |
| #185 `feat/dashboard-cost-rollup` | `skills/dashboard/app/src/lib/collect/cost.ts` + `RailLeft.tsx` cost tiles | 2026-07-15 19:11 |

Landing order matters: #186 (the miner + the loosened gate) merged first, so
#184's skill instructions and #185's dashboard reader both build against a
`retro.json` shape that already existed on `main` by the time each merged.

## #186 — the miner (`hooks/scripts/lib/loop_cost.sh`)

`dc_mine_token_usage <session_id>`, a sourced-not-executed bash lib mirroring
`dc_mine_hook_blocks`'s fail-open idiom (`discipline_common.sh`). Resolution:
globs `~/.claude/projects/*/<session_id>.jsonl` (override
`CLAUDE_PROJECTS_DIR` for tests) to find the orchestrator's project dir
`<proj>`; the orchestrator transcript is that file, worker transcripts are
every `.jsonl` recursively under `<proj>/<session_id>/subagents/` (mirrors
`usage.ts`'s `listJsonlFiles` recursive descent — a flat glob would miss an
agent that itself spawned a nested `subagents/`).

Per line: keeps only `type=="assistant"` with a string `message.id`, a
`message.model` that isn't `<synthetic>`, and a `message.usage` object;
**dedupes by `message.id`** (first occurrence wins — streaming re-emits the
same id with an identical cumulative snapshot, same invariant `usage.ts`
documents). Sums per model: `input_tokens`, `output_tokens`,
`cache_read_input_tokens`, and cache-write split into
`cache_write_5m`/`cache_write_1h` (from `usage.cache_creation`'s
`ephemeral_5m_input_tokens`/`ephemeral_1h_input_tokens`, with a legacy flat
`cache_creation_input_tokens` falling entirely into `cache_write_5m` —
conservative, since 5m is the cheaper multiplier).

Prices from `hooks/scripts/lib/model_prices.json` (override
`CLAUDE_MODEL_PRICES_FILE`). A model present in transcripts but absent from
the price table still has its tokens counted; `usd_estimate` is 0 and the id
is appended to `unpriced_models` — never dropped, never crashed on.

**Fail-open, three nesting depths.** A two-stage parse (line-tolerant `jq -R
'fromjson? // empty'`, then `jq -s` aggregation) already dropped malformed
JSON lines. This PR additionally type-guards three deeper failure shapes that
would otherwise abort the whole `jq -s` aggregation and wipe every other
transcript's real numbers to a bare `{}`: a wrong-typed `message.usage` (not
an object), a wrong-typed `usage.cache_creation` (not an object), and a
wrong-typed numeric leaf inside an otherwise-valid usage object (e.g.
`input_tokens: "abc"`). Any single malformed line at any of these depths is
dropped/coerced instead of crashing the batch. On any error anywhere (no
`jq`, no glob hit, unreadable dir, missing price file) the function returns
`{}` — never nonzero, never blocks a caller.

`session_id` is sanitised before being globbed into a path — reuses
`als_sanitise_session_id`'s strip-`/`-and-collapse-`..` transform (sourcing
`loop_state_common.sh` if not already loaded) rather than duplicating it;
documented as defence-in-depth (the id is harness-owned, not
attacker-controlled), not a security boundary, same framing as that
helper's own comment.

Output shape: `prices_as_of`, `price_source`, `per_model` (per-model token
breakdown + `usd_estimate`), `total_tokens`, `total_usd_estimate`,
`models_used` (array), `unpriced_models` (array), `transcripts_scanned`
(count) — all siblings on one object.

Verification: 47 checks in `loop_cost.test.sh` (dedupe, worker inclusion incl.
nested-subagent recursion, fail-open, multi-model split + rollup, cache
5m/1h pricing, legacy flat cache-write fallback, unpriced-model handling,
path-traversal neutralised, all three whole-batch-wipeout guards, stage-1
garbage-line tolerance). Also run against a real session
(`0767967d-90a4-459c-a552-529a19c512eb`, the [[dashboard-rethink-complete|dashboard-rethink loop]]):
65 transcripts, 4 models, `unpriced_models: []`, `total_usd_estimate: ~$409`.

### `model_prices.json` — the dated price table

```json
{
  "prices_as_of": "2026-06-24",
  "price_source": "https://www.anthropic.com/pricing — ... cache-write/cache-read
    multipliers (1.25x/2x write, 0.1x read) are Anthropic's documented prompt-caching
    economics, applied to each model's base input rate. claude-sonnet-5 uses its INTRO
    price ($2.00/$10.00) rather than standard ($3.00/$15.00) — see sonnet_intro_until.",
  "per_mtok": {
    "claude-opus-4-8":   { "input": 5.00,  "output": 25.00, "cache_read": 0.50, "cache_write_5m": 6.25,  "cache_write_1h": 10.00 },
    "claude-sonnet-5":   { "input": 2.00,  "output": 10.00, "cache_read": 0.20, "cache_write_5m": 2.50,  "cache_write_1h": 4.00,  "sonnet_intro_until": "2026-08-31" },
    "claude-haiku-4-5":  { "input": 1.00,  "output": 5.00,  "cache_read": 0.10, "cache_write_5m": 1.25,  "cache_write_1h": 2.00 },
    "claude-fable-5":    { "input": 10.00, "output": 50.00, "cache_read": 1.00, "cache_write_5m": 12.50, "cache_write_1h": 20.00 }
  }
}
```

`sonnet-5`'s intro rate ($2/$10 through 2026-08-31) is a **known staleness
trap** — this file must be hand-bumped to $3/$15 after that date; nothing
in the codebase does this automatically. No on-disk `claude-api` skill
reference file existed in this repo/plugin to cite instead (searched
`skills/`, `commands/`, `hooks/`, `~/.claude/`) — the price table cites the
public pricing page URL directly.

### The retro `schema_version` gate widens to forward-compatible

`hooks/scripts/lib/loop_state_common.sh`'s `als_gate_retro_on_complete`
(hosted in [[loop_stall_guard]]) previously required an **exact**
`schema_version` 1. Widened to accept **any `schema_version >= 1`**
(non-numeric, absent, or `< 1` still block) — this is what lets #184's Phase
13 bump the schema to 2 without the gate needing a matching same-PR change.
`loop_stall_guard.test.sh` gained negative/positive controls: 1/2/99 all
allow (forward-compat proof), 0/absent/non-numeric all still block.

## #184 — Phase 13 write contract, `retro.json` schema_version 2

`skills/agentic-loop/SKILL.md` Phase 13 step 1 gains a **cost-mining
sub-step**, same step, run after the existing fields are assembled:

1. Source `hooks/scripts/lib/loop_cost.sh`, run `dc_mine_token_usage
   <session_id>`.
2. **Fold-in is a split, not a copy.** Write the miner's returned object
   verbatim as `retro.cost` (carrying its own nested `schema_version` 1,
   independent of the retro's own version), then lift its `models_used`
   array **out** to top-level `retro.models_used` — `models_used` lives at
   retro top-level only, never duplicated inside `cost`. This split is what
   bumps the retro's own `schema_version` from 1 to 2; the `cost`/
   `models_used` fields do not exist under `schema_version` 1.
3. **Fail-open, same guarantee as the miner.** On any miner error both
   `retro.cost` and `retro.models_used` end up empty (`{}`/`[]`), and a
   `complete` declaration proceeds exactly as it would with populated
   values — `loop_stall_guard` checks the retro's presence and
   `schema_version`, never the cost field's correctness, so a miner failure
   cannot stall a loop.
4. **Pricing is computed once, here, and frozen.** `cost.per_model[*].
   usd_estimate` and `cost.total_usd_estimate` are priced a single time at
   teardown, stamped with `cost.prices_as_of`/`cost.price_source`. This is
   a load-bearing contract for #185: nothing downstream may re-price — the
   dashboard must sum the stored `usd_estimate` values as written and must
   never re-derive them from token counts against a live price table. A
   future "current price" toggle on the dashboard would be undoing this
   decision, not adding a feature.

**Human-facing reporting, not just a stored artifact.** The Phase 13
self-audit's "Artifacts produced" bullet gains a new sibling — **Loop
cost** — the per-model token + dated-USD breakdown from `retro.json`'s
`cost` field, printed to the human WITH a price-staleness age ("prices as
of `<cost.prices_as_of>`, N days old"). A `complete` loop must print this,
the same way the other Phase 13 facts are printed, not merely write it to
disk.

`AGENTS.md`'s skills↔hooks seam section and its hook-event-map table for
`loop_stall_guard.sh` are updated in lockstep — see
[[loop_stall_guard]] and the "third instance of the same seam" paragraph in
`AGENTS.md`'s working guide.

## #185 — dashboard cost rollup (`collect/cost.ts`)

`collectLoopCost(baseDir, now)` walks the same `~/.claude/agentic-loop`-shaped
tree `collectLoops` already reads (`<baseDir>/<slug>/<sessionId>/`), reading
each loop dir's `retro.json` if present. **This function never re-prices** —
per #184's frozen-pricing contract, it only sums the `total_usd_estimate`
each `retro.json` already carries.

`readLoopCost()` degrades a loop to "excluded" (not a thrown error) whenever:
no `retro.json`, `retro.cost` isn't an object, or `total_usd_estimate` isn't
a number — covers both an in-flight loop (no retro yet) and a fail-open empty
`cost: {}` from the miner.

Two rolling windows, same shape as `usage.ts`'s tiles but a **different
denominator** — completed loops only, not every local transcript regardless
of completion:
- **`week`** — rolling 7 days from `now` (`nowMs - 7*24*60*60_000` through
  `nowMs`).
- **`month`** — current **calendar** month (`[monthStart, monthStart+1)` by
  month boundary, not a fixed 30-day window) — a loop from last calendar
  month is excluded even if within 30 days of `now`.

`CostBucket.usd`/`tokens` are `number | null` — `null` means **zero
completed loops fell in this window**, distinct from a real `$0` spend (a
completed loop with an empty/unpriced `cost: {}` is a genuine zero and stays
a number), mirroring the existing `UsageTotals | null` pattern in `usage.ts`
so the rendered tile can't confuse "nothing to report" with a misleading
`$0.00`.

`deriveSharedPricesAsOf()` only surfaces a staleness note (`prices as of
<date>`) when **every** summed loop's `retro.json` carries the identical
`prices_as_of` — a mix of dates (e.g. spanning the sonnet-5 intro-rate
cutoff) has no single honest answer, so the bucket omits the date rather than
picking one arbitrarily.

### `RailLeft.tsx` — two new KPI tiles

`health.ts`'s `HealthTile.key` union gains `"costWeek" | "costMonth"`, both
computed by a shared `costTile()` helper alongside the existing `usage5h`/
`usageWeek`/`hooksFired`/`lintFindings` tiles. `costTile()` renders
`bucket.usd === null` as `unavailable: no completed loops in this window`
(not `$0.00`); otherwise the formatted USD value plus a note combining the
token count and scope label ("N tokens · completed loops only"), with the
shared `prices_as_of` staleness suffix appended when derivable. `RailLeft`
labels the two tiles "Cost (Week)" / "Cost (Month)" in `KPI_LABELS` and
renders `tile.note` under the value specifically for these two keys (the
other four tiles don't show a note in the UI).

## Cross-cutting notes

- **This cluster is the direct resolution of
  [[retro-json-per-model-cost-tracking-gap_2026-07-15]]**, filed the same
  day. That investigation's headline finding — "there is no cost or
  per-model tracking anywhere in coderails" — is no longer true as of this
  merge; see that page's own "Superseded" framing going forward, and treat
  its cross-PR-constraints section (ownership model, no-verdict rule,
  presence-only gate) as the design contract this cluster followed, not an
  open gap anymore.
- **Model-routing attribution is still not closed.** The investigation's
  constraint #4 — "no hook enforces actual model identity at spawn time, so
  cost attributed to Phase 2.8's role assignment is attributing to declared
  intent, not verified fact" — is unaffected by this cluster. The miner
  reads `message.model` directly off each transcript line (verified fact,
  per-message), which is actually a *stronger* join than the investigation
  anticipated needing — but this is a different axis: it tells you which
  model **produced tokens in a session**, not whether a worker's Phase 2.8
  role assignment was honoured. No reconciliation between the two exists.
- **Ownership model:** `retro.cost`/`retro.models_used` are
  orchestrator-written (via the Phase 13 teardown step), not hook-owned —
  same category as `decisions_absorbed`, not `loop_stop_counts`. See
  [[loop-progress-fields]] for the established pattern this follows.
- **Fourth independent seam on the retro/evals schema family.** Following
  [[task-evals-gate]]'s "one schema, three seams" framing (now four,
  counting `cost.ts`): `retro.json`'s `cost` field is written once by Phase
  13, read once by the dashboard's `cost.ts` — both must stay in lockstep
  with the miner's actual output shape, with no schema-version-checked
  enforcement beyond `loop_stall_guard`'s presence/parse gate.

## See also

- [[retro-json-per-model-cost-tracking-gap_2026-07-15]] — the investigation this cluster closes
- [[agentic-loop]] — Phase 13 teardown, the skill this cluster edits
- [[loop_stall_guard]] — hosts the widened `schema_version >= 1` retro gate
- [[dashboard]] — the six-panel dashboard; SYSTEM VITALS gains two tiles here
- [[loop-progress-fields]] — the field-ownership pattern (`retro.cost`/`retro.models_used` are orchestrator-owned)
- [[task-evals-gate]] — "one schema, N seams" framing this cluster extends
- [[pr_169_model-routing-step]] — the still-unclosed model-identity-at-spawn gap this cluster does not address
