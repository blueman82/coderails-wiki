---
title: "PR #47 — feat(init): add strictcode_skill config for multi-language strictcode"
type: source
created: 2026-06-26
last_updated: 2026-06-26
sources: []
tags: [source, init, push, workflow, strictcode, config]
---

# PR #47 — feat(init): add strictcode_skill config for multi-language strictcode

<!-- Ingested by /wiki-ingest after merge. This is an immutable record of what changed. -->

## PR metadata

| Field | Value |
|---|---|
| PR number | #47 |
| Branch | `feature/init-strictcode-skill` |
| Merged | 2026-06-26 |
| Merge SHA | `f00ed54` |
| JIRA ticket | — |

## Summary

PR #47 promotes `strictcode_skill` from a hardcoded `/strictcode-python` invocation to a configurable field in `workflow.config.yaml`. The `/coderails:init` scaffolder now prompts for the skill with language auto-detection: `go.mod` present → `/strictcode-go`; `package.json` + `.ts` files present → `/strictcode-ts`; otherwise → `/strictcode-python`. Users can override or answer "none" to disable strictcode entirely. Both [[push]] and [[workflow]] now read `config.strictcode_skill` (defaulting to `/strictcode-python` if absent/null) instead of calling `/strictcode-python` directly. `/coderails:workflow`'s `allowed-tools` frontmatter was expanded to pre-authorise `/strictcode-go` and `/strictcode-ts` alongside `/strictcode-python`. The change is fully backward-compatible: existing configs without `strictcode_skill` continue to run `/strictcode-python` as before. (verified: `gh pr diff 47`)

Note: PR #47's squash commit introduced a duplicate `strictcode_skill` bullet in `init.md`. PR #48 removed the duplicate; the canonical `init.md` has a single `strictcode_skill` field. (inferred: brief + diff review)

## Files changed

| File | What changed |
|---|---|
| `commands/init.md` | Step 5: new `strictcode_skill` prompt added with language auto-detection logic; example YAML gained `strictcode_skill: "/strictcode-python"` |
| `commands/push.md` | Pre-flight: "run `/strictcode-python`" → "run `config.strictcode_skill` (default: `/strictcode-python`)" |
| `commands/workflow.md` | Phase 3 pre-flight: same wording update; `allowed-tools` frontmatter: added `SlashCommand(/strictcode-go)` and `SlashCommand(/strictcode-ts)` |

## Wiki pages updated

- [[init]] — `strictcode_skill` added to collected config fields and YAML example
- [[push]] — pre-flight section updated; `config.strictcode_skill` added to config fields table
- [[workflow]] — Phase 3 pre-flight updated; `config.strictcode_skill` added to config fields table
- [[config-resolution]] — `strictcode_skill` noted as a new config field with auto-detection default

## Caveats / gotchas

- **Backward-compatible**: absent or null `strictcode_skill` falls back to `/strictcode-python`. Existing `workflow.config.yaml` files without this field continue to work unchanged. (verified: diff — default clause present in push.md and workflow.md updates)
- **`allowed-tools` expansion**: projects using `/strictcode-go` or `/strictcode-ts` will no longer hit permission prompts during `/coderails:workflow` because the skill is pre-authorised in the frontmatter. This does not affect standalone `/coderails:push` (which reads config at runtime, not via `allowed-tools`).
- **Auto-detection is a prompt default, not enforcement**: `/coderails:init` detects the likely skill and proposes it; the user can override at the prompt. The skill is recorded in config; nothing validates at runtime that the skill matches the project's language.
- **Duplicate bullet in PR #47 squash**: the `strictcode_skill` bullet appears twice in the raw `gh pr diff 47` output. PR #48 (`de-duplicate strictcode_skill bullet`) corrected this. The wiki documents the canonical (post-#48) state.
