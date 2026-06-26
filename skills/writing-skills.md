---
title: "writing-skills"
type: skill
created: 2026-06-25
last_updated: 2026-06-26
sources: [sources/pr_19-30_self-containment-and-hardening.md, sources/pr_56_writing-skills-using-coderails-example.md]
tags: [skill, meta, skill-authoring, plugin-development]
---

# writing-skills

Meta-skill for authoring new coderails skills: frontmatter conventions, trigger-phrase design, phase decomposition, and failure-mode encoding.

## Trigger phrases

When the user wants to create a new skill; "write a skill", "author a skill", "new skill for coderails". Meta — about the plugin itself.

## Relationship to /workflow

Outside the workflow chain. This is a plugin-development skill, not a project-delivery skill.

## Key phases / steps

1. Identify the failure mode the skill exists to prevent (work backwards from the problem).
2. Draft trigger phrases that Claude would actually encounter.
3. Define phases — each must have a verifiable output.
4. Write the `SKILL.md` in `coderails/skills/<name>/SKILL.md`.
5. Register the skill in the plugin manifest.

## Failure modes encoded

- Writing a skill that describes what to do without encoding what mistake it prevents.
- Trigger phrases so broad that the skill fires when it shouldn't.
- Phases with no verifiable exit criteria ("do the thing" is not a phase).
- Forgetting to register in the manifest (skill exists but is never loaded).

## Skill testing

The skill ships a `testing-skills-with-subagents.md` reference and an `examples/CLAUDE_MD_TESTING.md` worked campaign (added PR #56): skill testing is TDD applied to prose — RED (run a pressure scenario without the skill, watch the agent rationalize), GREEN (write the skill), REFACTOR (close loopholes). This is **manual** and done at authoring time.

There is **no automated skill testing in coderails** — neither triggering nor behavioural. The only `evals.json` was deleted as a dormant decoy (PR #55). The decision not to build a runner, and the candidate reminder-hook that would grow an eval corpus mechanically, are recorded in [[skill-testing-state_2026-06-26]].

## Source

`coderails/skills/writing-skills/SKILL.md` (+ `testing-skills-with-subagents.md`, `examples/CLAUDE_MD_TESTING.md`)

## See also

[[skill-testing-state_2026-06-26]] — what skill-testing exists, what's external, why coderails has no runner  
[[self-containment]] — context: these skills were vendored so coderails could grow its own skill library  
[[wiki-ingest]] — when you write a new skill, ingest it into the wiki
