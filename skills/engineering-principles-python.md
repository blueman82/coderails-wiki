---
title: "Skill: engineering-principles-python"
type: skill
created: 2026-07-06
last_updated: 2026-07-06
sources:
  - sources/pr_54_engineering-principles-vendoring.md
tags: [skill, engineering-principles, python, code-quality]
---

# Skill: engineering-principles-python

Python-specific coding standards and idioms. Invoked by [[engineering-principles]] for `.py` files, or directly.

Source: `coderails/skills/engineering-principles-python/SKILL.md`
Invoked as: `coderails:engineering-principles-python`

## Standards enforced

PEP 8 naming (`snake_case` functions/variables, `PascalCase` classes, `UPPER_CASE` constants); full type hints on all function signatures and class attributes, must pass Pyright strict; explicit imports (no star imports, no lazy imports, no `TYPE_CHECKING` blocks — fix the architecture instead of working around circular imports); `from __future__ import annotations` for forward references. (verified: SKILL.md "Python Idioms")

Idioms: EAFP (`try`/`except`) over LBYL (`if`-checks for existence/validity); context managers for all resources; f-strings over `.format()`/`%`; comprehensions over explicit loops where readable; no mutable default arguments. PEP 257 docstrings for public APIs only — no docstrings for private methods unless complex. (verified: SKILL.md "Python Idioms")

## Reduction patterns

Ternary over if/else assignment; walrus operator (`:=`) for assign-and-test; no redundant `else` after `return`; builtin generics (`list[int]`, `dict[str, int]`) over `typing.List`/`Dict`; `X | None` over `Optional[X]`; comprehensions over `for`+`append`; module-level function over single-method class. (verified: SKILL.md "Reduction Patterns")

## Imperative → declarative patterns

A mandatory scan-and-replace table for loop, conditional, and accumulation anti-patterns (verified: SKILL.md "Imperative → Declarative Patterns"):

- **Loops**: `itertools.product` for 3+ nested loops; `enumerate()` over manual counters; `zip()` over index-based parallel iteration; `any()`/`all()` over break-on-condition loops; dict/set comprehensions over loop-building; `sum()` over running totals; chained comprehensions over filter-then-transform; `itertools.chain.from_iterable()` over nested-loop flatten.
- **Conditionals**: dict dispatch over 5+ `if`/`elif` chains; `match`/`case` (3.10+) or visitor pattern over repeated `isinstance` checks; guard clauses over 3+-deep nesting.
- **Accumulation**: `"".join(parts)` over string concatenation in loops; `{**a, **b}`/`a | b` over manual dict merge; `collections.Counter` over manual counting; `pathlib.Path` over manual path strings; `itertools.groupby()`/`defaultdict(list)` over manual grouping.

## Output sanitization

Any text from AI models, user input, or external APIs must be treated as untrusted and sanitized at the point of insertion, not the point of collection (verified: SKILL.md "Output Sanitization Patterns"):

| Output format | Dangerous tokens | Sanitization |
|---|---|---|
| Slack mrkdwn | `<!channel>`, `<!here>`, `<!everyone>`, `<@U...>` | `re.sub(r'<[!@][^>]+>', '', text)` |
| HTML | `<script>`, `onclick=`, entity injection | `html.escape(text)` |
| SQL | `'; DROP TABLE`, `OR 1=1` | Parameterized queries only, never f-strings |
| Shell | `$(cmd)`, `` `cmd` ``, `; rm -rf` | `shlex.quote(text)` |
| JSON in templates | Unescaped quotes, newlines | `json.dumps(text)` |

Allowlist over blocklist for status/result dicts consumed by callers: `if status not in (good_values)` fails closed; `if status == "error"` does not. (verified: SKILL.md "Allowlist > blocklist")

## Checklist

Full type hints; builtin generics not `typing.List`/`Dict`/`Optional`; no star imports; no mutable defaults; f-strings; context managers; EAFP; no redundant else; comprehensions where readable; no 3+-deep nested loops; `enumerate`/`zip` over manual indexing; `any`/`all` over break-loops; `sum`/`max`/`min` over manual accumulation; `Counter`/`defaultdict` over manual counting/grouping; `"".join()` over loop concatenation; dict dispatch over long `if/elif`; guard clauses over deep nesting; untrusted text sanitized before output. (verified: SKILL.md "Checklist")

## Source

`coderails/skills/engineering-principles-python/SKILL.md`

## See also

- [[engineering-principles]] — coordinator skill that dispatches here for `.py` files
- [[engineering-principles-go]] — sibling language skill
- [[engineering-principles-ts]] — sibling language skill
- [[pr_54_engineering-principles-vendoring]] — the PR that vendored this skill from the external `strictcode-python`
