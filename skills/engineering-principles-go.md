---
title: "Skill: engineering-principles-go"
type: skill
created: 2026-07-06
last_updated: 2026-07-06
sources:
  - sources/pr_54_engineering-principles-vendoring.md
tags: [skill, engineering-principles, go, code-quality]
---

# Skill: engineering-principles-go

Go-specific coding standards and idioms. Invoked by [[engineering-principles]] for `.go` files, or directly.

Source: `coderails/skills/engineering-principles-go/SKILL.md`
Invoked as: `coderails:engineering-principles-go`

## Standards enforced

Accept interfaces, return structs — functions take interface parameters for flexibility but return concrete struct types for clarity; package-level organization over class-level thinking. Errors are values: handle or return them, never discard with `_ =`; wrap with `fmt.Errorf("context: %w", err)`; check and return early. Table-driven tests with `[]struct{ name string; ... }` and `t.Run(tt.name, ...)` subtests. Naming: short names (`i`, `n`, `err`) in small scopes; exported `PascalCase`, unexported `camelCase`; acronyms stay uppercase (`HTTPClient` not `HttpClient`). (verified: SKILL.md "Go Idioms")

## Reduction patterns

Early return over `if err != nil { ... } else { ... }`; wrap returned errors with context; remove purposeless empty structs; no discarded unused vars (`_ = f()`) — check the error; named returns for documentation value; no `TODO` comments — fix now or remove. (verified: SKILL.md "Reduction Patterns")

## Imperative → idiomatic patterns

Go is intentionally imperative (no map/filter/reduce), so this table targets Go-specific verbosity instead (verified: SKILL.md "Imperative → Idiomatic Patterns"):

- **Loops**: extract 3+-deep nested loops to functions; `strings.Builder`/`strings.Join` over `+=` string building; pre-allocate slices/maps with `make(..., len(src))` when size is known; `map[T]bool`/`map[T]struct{}` lookup over manual contains-loop; `slices.Reverse`/`slices.Contains`/`slices.Sort`+`slices.Compact` (1.21+) over manual equivalents; `min()`/`max()` builtins (1.21+) over manual tracking.
- **Conditionals**: `map[string]func()` dispatch over 5+ `switch`/`if-else` chains; `switch v := x.(type)` over repeated type assertions; guard clauses over 3+-deep nesting; short statement form (`if err := f(); err != nil`); direct boolean expression return over if/else returning true/false.
- **Concurrency**: `errgroup.Group` over manual goroutine+`WaitGroup` or manual channel collection; worker pool with semaphore channel over unbounded goroutine spawn; `sync.Map` for append-heavy read-many patterns; `context.Context` deadline or `default` case on every `select`.
- **Stdlib shortcuts**: middleware pattern for HTTP handler chains; struct tags for JSON field naming; `filepath.WalkDir` for file tree walks; exponential backoff+jitter for retries; `bytes.Buffer`; `filepath.Join`/`path.Join` over manual `/` concatenation.

## Checklist

All errors handled or returned (no `_ =`); errors wrapped with `%w`; early returns, no nested if/else; functions accept interfaces, return concrete structs; table-driven tests with subtests; no `TODO` comments; no purposeless empty structs; no 3+-deep nested loops; `strings.Builder`/`strings.Join` over `+=`; pre-allocated slices/maps when size known; `map[T]bool`/`slices.Contains` over manual contains loops; `errgroup.Group` over manual goroutine+WaitGroup; type switch over repeated assertions; map dispatch over long switch/if-else; guard clauses over deep nesting; `min()`/`max()` builtins (1.21+). (verified: SKILL.md "Checklist")

## Source

`coderails/skills/engineering-principles-go/SKILL.md`

## See also

- [[engineering-principles]] — coordinator skill that dispatches here for `.go` files
- [[engineering-principles-python]] — sibling language skill
- [[engineering-principles-ts]] — sibling language skill
- [[pr_54_engineering-principles-vendoring]] — the PR that vendored this skill from the external `strictcode-go`
