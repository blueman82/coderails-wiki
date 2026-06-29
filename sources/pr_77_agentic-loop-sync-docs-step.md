---
title: "PR #77 — agentic-loop: /sync-docs at loop boundary"
type: source
created: 2026-06-29
last_updated: 2026-06-29
sources: []
tags: [skill, agentic-loop, docs-drift, sync-docs, loop-boundary, in-tree-docs]
---

# PR #77 — agentic-loop: /sync-docs at loop boundary

<!-- Ingested by /wiki-ingest after merge. This is an immutable record of what changed. -->

## PR metadata

| Field | Value |
|---|---|
| PR number | #77 |
| Branch | `feat/agentic-loop-sync-docs-step` (inferred) |
| Merged | 2026-06-29 |
| Merge SHA | `cefb47c` |
| JIRA ticket | — |

## Summary

Added a "Docs-drift check — run `/sync-docs` at the loop boundary" subsection to Phase 9 of `skills/agentic-loop/SKILL.md`. The orchestrator now runs `/sync-docs` **once** at the loop boundary, after the cluster wiki ingest+lint. This is a SKILL PROCESS change only — no hook, no config, no script changes.

**The core distinction this PR encodes:** wiki ingest updates the external knowledge base; `/sync-docs` audits the repo's own in-tree docs (README.md, AGENTS.md, docs/REFERENCE.md, etc.) for drift against the just-merged code. These were previously conflated as "docs" at the loop boundary. PR #77 separates them explicitly. (verified: PR #77 diff)

## What changed

### Phase 9 — new `/sync-docs` subsection

A new paragraph was added to Phase 9 of `skills/agentic-loop/SKILL.md`:

```
**Docs-drift check — run `/sync-docs` at the loop boundary**

After the cluster wiki ingest+lint, the orchestrator runs `/sync-docs` ONCE at the loop
boundary. Wiki ingest updates the external knowledge base; `/sync-docs` is the
complement — it audits the repo's own in-tree docs (e.g. README.md, AGENTS.md,
docs/REFERENCE.md) for drift against the just-merged code.

Run it even without Serena (the `--semantic` backend) — omit `--semantic` for the
traditional file-comparison audit, which still catches drift. Do not skip `/sync-docs`
just because Serena isn't installed.

Delegate it to a spawned agent, same as the wiki step, to keep orchestrator context clean.

**Disposition of findings:** `/sync-docs` surfaces drift; the orchestrator must triage.
Fix only drift the loop's own PRs introduced. Surface pre-existing drift to the user
rather than silently folding unrelated doc fixes into the loop — that is scope creep.
This mirrors the loop's finding-triage discipline.
```

### Files changed

| File | Change |
|---|---|
| `skills/agentic-loop/SKILL.md` | Added `/sync-docs` subsection to Phase 9 |

## Key design decisions

**`/sync-docs` is delegated, not inline.** The orchestrator spawns an agent for it, the same pattern as the wiki step — keeps main context clean (the same "orchestrator never implements" rule from Phase 3). (inferred: consistent with Phase 3/3a delegation principle)

**Traditional audit (no `--semantic`) is sufficient.** The file-comparison audit catches real drift without requiring the Serena backend to be installed. Omitting `--semantic` is the correct default when Serena is absent — `/sync-docs` must not be skipped because of an optional backend.

**Scope discipline on findings mirrors finding-triage.** Only drift introduced by the loop's own PRs is fixed inline. Pre-existing drift is surfaced to the user (not silently folded in). This is not new policy — it reapplies the loop's existing finding-triage discipline ([[agentic-loop]] Phase 5) to the docs domain.

**Thematic distinction from PR #76.** PR #76 (hook stdin hardening) and PR #77 (agentic-loop sync-docs step) are independent changes ingested separately. They share only the merge date (2026-06-29).

## Wiki pages updated

- [[agentic-loop]] — Phase 9 description updated to include the `/sync-docs` step

## Caveats / gotchas

- This is advisory, not enforced. No hook gates `/sync-docs` execution. The agentic-loop skill is the instruction layer (the orchestrator must choose to invoke it); see [[enforcement-model]].
- `/sync-docs` without `--semantic` is a traditional file-comparison audit, not a semantic analysis. Semantically-equivalent but textually-different docs will pass the audit. That is acceptable — traditional audit still catches the common case (stale command names, removed flags, changed config keys).
- Pre-existing drift found by `/sync-docs` is NOT the loop's responsibility to fix. Surface it; don't fold it in. A loop that silently absorbs unrelated doc fixes violates its own scope discipline.
