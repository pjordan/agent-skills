# Contributing

Thanks for your interest in contributing to agent-skills. This project is a collection of skills that help coding agents work more effectively during long-running development sessions.

## Before Proposing a New Skill — the Three Bars

Adding a skill is a durable commitment: once users depend on its operations, renaming or
restructuring is breaking. The triggering surface also isn't free — every extra skill's
description competes for model attention, and a plugin with too many overlapping descriptions
produces unreliable triggering. Realistic ceiling is roughly 5-7 skills per plugin.

**Default answer to "should we add a skill?" is no.** A proposal must clear all three bars:

1. **Disjoint triggering language.** The skill's natural trigger phrases do not overlap
   meaningfully with any existing skill's description. If the user's request could fire two
   skills, the triggering contract is already broken.
2. **Multi-step procedure that benefits from a checklist.** A skill is for work that has
   ceremony — preconditions, ordered steps, artifacts, rails. Single-shot transformations,
   one-line conventions, or "just remember to do X" are not skill-shaped.
3. **Awkward as any alternative.** A skill is the right shape only if it would be worse as
   a settings.json hook (for automation), a subagent definition (for encapsulated reasoning),
   a slash command, or a `CLAUDE.md` rule. State explicitly why the alternatives fall short.

If the proposal doesn't clear all three, don't open the PR — file an issue describing the
problem instead, or propose a smaller change to an existing skill.

## Adding a New Skill

1. Create a directory under `plugins/agent-skills/skills/` with your skill name (kebab-case):
   ```
   plugins/agent-skills/skills/my-skill/
   ├── README.md
   └── SKILL.md
   ```

2. Write your `SKILL.md` with YAML frontmatter:
   ```yaml
   ---
   name: my-skill
   description: >
     What the skill does and when to trigger it. Be specific about
     trigger phrases and contexts.
   ---
   ```

3. Follow these guidelines for the skill body:
   - Explain the *why* behind instructions, not just the *what*
   - Use the imperative form ("Create the file", not "You should create the file")
   - Include concrete examples with realistic file paths and content
   - Keep it under 500 lines — use `references/` files for overflow
   - Design for any LLM agent, not just Claude Code

4. Update the skills table in `README.md`

5. Add an entry to `CHANGELOG.md` under `## [Unreleased]`

## Improving an Existing Skill

- Read the skill's `SKILL.md` and understand its intent before making changes
- Test your changes by running the skill against a realistic codebase
- Describe what you changed and why in your PR

## Skill Quality Expectations

A good skill:
- Solves a real problem that agents encounter repeatedly
- Works with plain files and standard tools (no special dependencies)
- Has clear operations that an agent can follow without ambiguity
- Explains its reasoning so the agent can adapt to edge cases
- Is framework-agnostic where possible

## Pull Requests

- Keep PRs focused — one skill or one improvement per PR
- Write a clear description of what changed and why
- If adding a new skill, include a brief example of the skill in action

## Reporting Issues

Use the GitHub issue templates:
- **Bug report** — for skills that produce incorrect or broken output
- **Skill request** — for ideas about new skills to add
