---
title: "Skill: engineering-principles"
type: skill
created: 2026-07-06
last_updated: 2026-07-06
sources:
  - sources/pr_54_engineering-principles-vendoring.md
tags: [skill, engineering-principles, yagni, kiss, dry, fail-fast, ssot, law-of-demeter, code-quality, serena]
---

# Skill: engineering-principles

Proactively enforces six universal engineering principles (YAGNI, KISS, DRY, Fail Fast, SSOT, Law of Demeter) and dispatches to a language-specific sub-skill for idiom enforcement. Coordinator for the `engineering-principles` family.

Source: `coderails/skills/engineering-principles/SKILL.md`
Invoked as: `coderails:engineering-principles`

## Why this skill exists

Every other coderails skill polices process (did a plan get written, did a review happen); this one polices the code itself against six named principles, so violations get caught proactively — "after writing or modifying code in any file" — rather than only surfacing at review time. It vendors what was previously the global, out-of-plugin `strictcode` skill family, completing the self-containment goal for code-quality enforcement. (verified: SKILL.md "When to Activate"; sources/pr_54_engineering-principles-vendoring.md "Summary")

## Trigger

```
'Enforce engineering principles (YAGNI, KISS, DRY, Fail Fast, SSOT, Law of Demeter)
and language-specific coding standards (Python/Go/TypeScript) on code being written
or modified. Uses Serena LSP for call site analysis and reference counting. Use
PROACTIVELY when writing, modifying, or reviewing code. Triggers on code changes,
"enforce standards", "check principles", "apply standards", "code quality", or
explicit /engineering-principles command.'
```

Proactive activation (verified: SKILL.md "When to Activate"): after writing or modifying code in any file, when reviewing code changes before commit, or when implementing features/fixing bugs. Explicit activation: the user invokes `/engineering-principles` pointed at a file or directory.

## The six universal principles

Every code change is checked against these (verified: SKILL.md "Core Principles"):

| # | Principle | What to look for | Best tool |
|---|-----------|-------------------|-----------|
| 1 | **YAGNI** | Unused code, speculative features, dead branches | Serena `find_referencing_symbols` (LSP-precise); else Grep for call sites |
| 2 | **KISS** | Over-engineered abstractions, trivial classes | Serena `find_symbol` depth → single-method classes |
| 3 | **DRY** | Duplicated logic across files | Grep/Glob for repeated signatures or body fragments across files |
| 4 | **Fail Fast** | Late validation, deep nesting before error checks | Serena `find_symbol` body → nesting depth |
| 5 | **SSOT** | Duplicated state/config | Grep for the same config key/value in 2+ files |
| 6 | **Law of Demeter** | `a.b.c.d` chains | Serena `find_symbol` body → chain regex |

**Tool selection rule:** Serena for in-file structural analysis (LSP-backed, authoritative) when available; plain Grep/Glob for cross-file checks and as the fallback when Serena is absent. (verified: SKILL.md "Tool selection rule")

## Per-language dispatch

Phase 0 is mandatory and runs before any analysis (verified: SKILL.md "PHASE 0: Language Detection & Dispatch"):

| File extension | Invoke skill |
|---|---|
| `.go` | [[engineering-principles-go]] |
| `.py` | [[engineering-principles-python]] |
| `.ts`, `.tsx` | [[engineering-principles-ts]] |

The coordinator supplies universal principles and Serena-backed structural analysis; the language-specific skill supplies idioms, patterns, and before/after examples. (verified: SKILL.md "Phase 0" note)

## Serena (LSP) integration

Phase 1 activates Serena for the current project via `mcp__mcp-exec__execute_code_with_wrappers` with the `serena` wrapper; Phase 2 uses it for symbol overview, reference counting (YAGNI), symbol depth (Law of Demeter), class structure (KISS), and return-value contract verification (unhandled status/enum paths flagged as a "silent failure path"). **Graceful degradation:** if Serena is unavailable, the skill falls back entirely to file-level static analysis via Read/Grep/Glob — Serena enhances but is not required. (verified: SKILL.md "Phase 1", "Phase 2.5", "Graceful degradation")

Scout (semantic search) and SlimCode integration were deliberately stripped when this skill was vendored from `strictcode` — the DRY/SSOT/YAGNI rows were rewritten to a Grep/Glob fallback so the skill has no dependency outside coderails. Serena was retained because its degradation path already covered the "tool absent" case. (verified: sources/pr_54_engineering-principles-vendoring.md "What changed")

## Enforcement process

Four steps (verified: SKILL.md "Enforcement Process"):

1. **Detect language & dispatch** — read the file extension, invoke the matching language skill.
2. **Analyze** — Serena symbol overview/reference counting/depth analysis when available; Grep/Glob cross-file checks always; full static-analysis fallback if Serena is absent.
3. **Fix or flag** — fixes directly when LSP-confirmed safe (naming, missing type hints, redundant `else` after `return`, unused imports); flags to the user when confirmation is needed (deleting zero-reference functions, collapsing single-method classes, extracting duplicated code, breaking `a.b.c.d` chains).
4. **Report** — a structured summary: language detected, fixed count with line references, flagged count with LSP-verified issues.

## Rules

Never change external behavior or public APIs; never remove or simplify error handling; always preserve test coverage; when in doubt, flag instead of fix; apply the minimum change needed; don't add features, comments, or docstrings beyond what's needed; trust LSP data over heuristics for in-file analysis when available; always invoke the language-specific skill for idioms. (verified: SKILL.md "Rules")

## Where this skill is wired in

Three touchpoints beyond direct invocation, added when the family was vendored (verified: sources/pr_54_engineering-principles-vendoring.md "What changed"):

- **Planning** — [[writing-plans]]'s self-review gate gains an engineering-principles item; [[brainstorming]] gains a principle-vetting note.
- **Pre-push pre-flight** — [[push]] and [[workflow]] run `config.engineering_principles_skill` (default `/engineering-principles-python`) against changed files matching `config.engineering_principles_paths`, or any file with ≥20 lines changed. Config resolution and the `engineering_principles_paths: null` disable path are documented in [[config-resolution]].
- **PR review** — `workflow.md` Phase 3 step `2b` runs `/engineering-principles`, step `2c` runs `/simplify` (the built-in command, kept because `review-pr`'s own code-simplifier agent isn't guaranteed to run).

All three touchpoints are advisory, per [[enforcement-model]]: nothing mechanically blocks an implementation that skips the principle checks. (verified: sources/pr_54_engineering-principles-vendoring.md "Caveats / gotchas")

## Source

`coderails/skills/engineering-principles/SKILL.md`

## See also

- [[engineering-principles-python]] — Python-specific idioms and reduction patterns
- [[engineering-principles-go]] — Go-specific idioms and reduction patterns
- [[engineering-principles-ts]] — TypeScript-specific idioms and reduction patterns
- [[pr_54_engineering-principles-vendoring]] — the PR that vendored this family from the external `strictcode` skill
- [[pr_47_strictcode-skill-config]] — superseded predecessor (config keys renamed on vendoring)
- [[config-resolution]] — `engineering_principles_paths`/`engineering_principles_skill` config keys
- [[push]], [[workflow]] — pre-flight touchpoints
- [[writing-plans]], [[brainstorming]] — planning-stage touchpoints
- [[enforcement-model]] — why this skill's touchpoints are advisory, not hook-enforced
