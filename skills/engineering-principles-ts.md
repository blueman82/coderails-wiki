---
title: "Skill: engineering-principles-ts"
type: skill
created: 2026-07-06
last_updated: 2026-07-06
sources:
  - sources/pr_54_engineering-principles-vendoring.md
tags: [skill, engineering-principles, typescript, code-quality]
---

# Skill: engineering-principles-ts

TypeScript-specific coding standards and idioms. Invoked by [[engineering-principles]] for `.ts`/`.tsx` files, or directly.

Source: `coderails/skills/engineering-principles-ts/SKILL.md`
Invoked as: `coderails:engineering-principles-ts`

## Standards enforced

Strict mode always (`"strict": true`); no `any` — use `unknown` and narrow, or define proper types; explicit return types on exported functions; discriminated unions over type assertions. Optional chaining (`?.`) over nested null checks; nullish coalescing (`??`) over `||` for defaults; non-null assertion (`!`) only when safety is verified. Exhaustive `switch` checks — never a `default` that swallows cases; `assertNever` pattern for compile-time exhaustiveness. `const` assertions where applicable; type inference when obvious from assignment; object shorthand; arrow functions for callbacks; template literals over string concatenation. (verified: SKILL.md "TypeScript Idioms")

## Reduction patterns

Optional chaining over chained `&&` null guards; nullish coalescing over `||` defaults; object shorthand over `{ name: name }`; template literals over string concatenation; arrow callbacks over `function` callbacks; omit type annotation when inference is obvious. (verified: SKILL.md "Reduction Patterns")

## Imperative → declarative patterns

A mandatory scan-and-replace table for loop, conditional, accumulation, and promise anti-patterns (verified: SKILL.md "Imperative → Declarative Patterns"):

- **Loops**: `.map()`/`.filter()`/`.reduce()` over `for`+`push`; `.find()`/`.some()`/`.every()` over `for`+`break`; `.flatMap()` over nested `for` loops; `for...of`/`.forEach()`/`.entries()` over index-based `for`; `Object.fromEntries()` over loop-building objects; `.reduce()` over running totals; `.filter().length` over manual match counting; `.filter()` with a type predicate over manual type-narrowing loops.
- **Conditionals**: object lookup/`Map`/`Record<K, V>` over 5+ `if/else if` chains; extract-to-function or `switch` over 3+-deep nested ternaries; type guard functions (`value is T`) over repeated `typeof` checks; `.some()`/`.every()` over flag accumulation; guard clauses over 3+-deep nesting.
- **Accumulation**: spread (`[...a, ...b]`, `{ ...a, ...b }`) over manual array/object merge; `.join()` or template literal over string concat in loops; `Map`+`.reduce()` or `Object.groupBy()` (ES2024) over manual grouping; `[...new Set(items)]` over manual dedup.
- **Promises**: `Promise.all()` over sequential `await` in a loop for independent operations; `async`/`await` over `.then()` chains or callback nesting; `Promise.allSettled()` over manual error collection; `AbortSignal.timeout()`/`Promise.race()` over manual timeout wrappers.

## Checklist

No `any`; explicit return types on exported functions; optional chaining for null checks; nullish coalescing not `||` for defaults; discriminated unions for variants; exhaustive switch with `assertNever`; `as const` for literal types; object shorthand; template literals; arrow functions for callbacks; `.map`/`.filter`/`.reduce` over `for`+`push`; `.find`/`.some`/`.every` over `for`+`break`; `.flatMap` over nested loops; `Object.fromEntries` over loop-building objects; `Promise.all` over sequential awaits in loops; object/Map dispatch over long `if/else if`; guard clauses over deep nesting; `[...new Set()]` over manual dedup. (verified: SKILL.md "Checklist")

## Source

`coderails/skills/engineering-principles-ts/SKILL.md`

## See also

- [[engineering-principles]] — coordinator skill that dispatches here for `.ts`/`.tsx` files
- [[engineering-principles-python]] — sibling language skill
- [[engineering-principles-go]] — sibling language skill
- [[pr_54_engineering-principles-vendoring]] — the PR that vendored this skill from the external `strictcode-ts`
