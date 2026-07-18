# Repository Guidelines

## Project Overview

This repository is a personal collection of reusable AI-agent prompts and skills. The current source content lives under `skills/`; each skill is self-contained and written primarily in Markdown.

## Repository Layout

- `README.md` gives the repository-level overview.
- `skills/<skill-name>/SKILL.md` is the entry point and instruction source for a skill.
- `skills/<skill-name>/assets/` contains templates and other files used by that skill.
- `.kproject/` contains local issue-workflow state. Do not edit it unless the request explicitly invokes or concerns the kproject workflow.

## Editing Skills

- Read a skill's entire `SKILL.md` and every directly relevant asset before changing it.
- Preserve the YAML frontmatter at the top of `SKILL.md`. Keep `name` aligned with the directory name and make `description` explicit about when the skill should trigger.
- Write operational instructions: state what the agent must inspect, create, update, validate, or ask rather than describing only the intended outcome.
- Keep terminology, filenames, workflow stages, selectors, and examples consistent across `SKILL.md` and its assets.
- When adding a referenced template or asset, use a path relative to the skill directory and verify that the file exists.
- Prefer adapting existing templates and conventions over introducing a second format for the same kind of record.
- Keep changes scoped to the requested skill. Do not rewrite unrelated wording or reformat untouched files.

## kproject-Specific Conventions

- The canonical issue records are `input.md`, `problem.md`, `design.md`, and `tasks.md`; their matching templates live in `skills/kproject/assets/`.
- If a workflow change affects the structure or meaning of one of those records, update both `skills/kproject/SKILL.md` and the relevant template in the same change.
- Task numbers are stable once assigned, and task statuses must clearly distinguish finished from unfinished work.
- Treat user-provided sources as required inputs: preserve their references and record whether they were inspected or inaccessible.

## Validation

There is currently no automated test suite or build step. Run the smallest useful checks for Markdown-only changes:

```bash
git diff --check
git diff -- AGENTS.md skills/
```

Then manually verify that:

- `SKILL.md` frontmatter is valid and remains the first content in the file;
- referenced assets and example paths exist;
- headings, selectors, filenames, and status values agree between instructions and templates;
- examples still demonstrate the behavior described by the surrounding text.

Report the checks run and note any validation that could not be performed.

## Git and Safety

- Inspect `git status --short` before editing and preserve existing user changes.
- Do not create commits, switch branches, or modify remotes unless explicitly requested.
- Never add credentials, tokens, private keys, or environment-file contents to the repository or command output.
