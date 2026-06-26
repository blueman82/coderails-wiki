---
title: "Skill-testing state in coderails — 2026-06-26"
type: investigation
created: 2026-06-26
last_updated: 2026-06-26
sources: [sources/pr_55_remove-dead-evals.md, sources/pr_56_writing-skills-using-coderails-example.md]
tags: [investigation, skill-testing, evals, skill-creator, decision]
---

# Skill-testing state in coderails — 2026-06-26

> Filed: 2026-06-26. Point-in-time snapshot — may be superseded.

## Question

Does coderails have automated skill testing? And what was the orphaned `skills/planning-sequence/evals/evals.json` actually for — dead, used, or half-baked? The file's presence implied a test harness existed; this investigation determined whether it does.

## Evidence

- `skills/writing-skills/testing-skills-with-subagents.md` — describes the **manual** skill-testing ritual (TDD applied to skills: RED = run pressure scenario without the skill, GREEN = write skill, REFACTOR = close rationalization loopholes). Manual, by hand, at creation time.
- `skill-creator` (an **external** `claude-plugins-official` plugin, NOT part of coderails — zero skill-creator code in the repo) ships two distinct eval systems:
  - **Trigger eval** — `scripts/run_eval.py` (310 lines, re-verified on disk). Schema `{query, should_trigger}`. Tests whether a skill's *description* causes it to fire. Parallel `claude -p` workers, trigger-rate threshold.
  - **Quality/behavioural eval** — `evals/evals.json` schema `{skill_name, evals:[{id, prompt, expected_output, files, expectations[]}]}` (defined in `references/schemas.md`), run by an **executor agent + grader agent** (Benchmark/Improve modes). The grader scores the `expectations[]` array.
- coderails repo: grep across `*.sh`/`*.py`/`*.js`/`*.md`/`*.json` for the eval file / its path / any loader → the only hit was the file's own contents. Nothing reads it.
- `hooks/scripts/tests/` — 14 test files + `run_all.sh`. The mechanically-enforced layer (hooks) IS regression-tested.

## Findings

- **coderails has ZERO automated skill testing** — neither triggering nor behavioural. `(verified)`
- coderails' lone `planning-sequence/evals.json` was skill-creator's **quality-eval** schema, dormant, and missing the `expectations[]` field the grader actually scores. It matched skill-creator's quality format (shared lineage) but NOT `run_eval.py`'s trigger format. `(verified)`
- The bottleneck for building a runner is the **eval corpus, not the runner**: one eval file across 27 skills. `(verified)`
- Only ~4–6 *discipline* skills are pressure-sensitive (a one-word edit can flip compliance): [[test-driven-development]], [[planning-sequence]], [[verification-before-completion]], [[receiving-code-review]], [[agentic-loop]]. The wiki/reference/engineering-principles skills have no loophole to test. `(inferred)`
- Stakes are already correctly tiered: hooks (must-hold invariants) are tested; skills are advisory by design — the lower-stakes half. See [[enforcement-model]], [[discipline-loop]]. `(verified)`

## Adversarial review

The reasoning initially leaned on a false pillar — a claim that the eval file "already plugs into run_eval.py today." Re-verification killed it on two counts: nothing auto-runs the file, and the schemas don't match (`query`/`should_trigger` vs `prompt`/`expected_output`). The conclusion survived on the two strong legs (corpus=1, stakes-tiering). Lesson: verify *wiring + schema*, not just *file existence*, before claiming something is "already automated."

## Resolution

- **Deleted** the dormant eval file ([[pr_55_remove-dead-evals]]) — it was a decoy implying infra that didn't exist.
- **Decided NOT to build a behavioural skill-eval runner now.** A runner over a corpus of one is theatre; most skills aren't pressure-sensitive; the tested-hooks / advisory-skills tiering already covers the high-stakes half.
- **Candidate future fix** (deferred, not built): a reminder *hook* (PostToolUse on Write/Edit, mirroring [[no_edit_on_main]]) that fires when a discipline `SKILL.md` is edited and nudges to capture/run an eval — growing the corpus mechanically rather than relying on the maintainer remembering. Build the runner once the corpus earns it.

## See also

- [[writing-skills]] — the skill that authors skills; skill testing is part of its remit
- [[enforcement-model]] · [[discipline-loop]] — why advisory skills are the lower-stakes half
- [[pr_55_remove-dead-evals]] · [[pr_56_writing-skills-using-coderails-example]]
