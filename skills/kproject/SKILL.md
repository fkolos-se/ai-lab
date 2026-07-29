---
name: kproject
description: "Guide for kproject, a local, file-based issue workflow under a software project's .kproject/ directory. Trigger immediately when a message starts with `kproject`, optionally followed by an issue name and/or `-p PATH`; this selects (and, if needed, initializes) the active issue. Also use it for every follow-up request in an active kproject thread, including research, design, task planning, question resolution, or implementation. Always prefer this skill over ad-hoc editing for files under .kproject/issues/. Don't load in reaction of only kproject root mention, search through parent dirs for the first with .kproject subdir and consider it root instead."
---

# kproject

kproject is a lightweight, file-based issue workflow inside a project's `.kproject/` directory. An issue progresses through initialization, research/design/task planning, and implementation. Its durable records are plain Markdown files so a person or a future agent can resume the work with its context intact.

## Calling command and active context

Invoke the skill with:

```text
kproject [<issue_name>] [-p <project_path>]
```

`<issue_name>` is optional and selects the active issue. `-p <project_path>` is optional and selects the project root. Accept `-p` before or after the issue name. Treat the value following `-p` as the path and the remaining positional value as the issue name; never interpret a positional value as a project path. Treat stage selectors such as `--design` as actions, not issue names. Ask a focused question if `-p` has no value or multiple positional values make the issue name ambiguous.

Parse the command before taking action:

- `kproject add-animation -p /work/my-app` activates issue `add-animation` in `/work/my-app`.
- `kproject add-animation` activates that issue in the agent's project root.
- `kproject -p /work/my-app` selects a project but no issue.
- `kproject` uses the agent's project root but no issue.

Resolve the project root in this order: the explicit `-p` value, the agent's current project root (normally the current Git root), then the current working directory when outside Git. Validate that it exists and is a directory. If no root can be determined, ask for `-p`.

Accept an issue name as a slug or a quoted phrase with spaces. Reject raw names equal to `.` or `..` or containing a path separator. Normalize other names to a safe lowercase kebab-case slug by transliterating or dropping unsupported characters and collapsing separators. For example, convert `Add Animation to the Button` to `add-animation-to-the-button`. Tell the user whenever normalization changes the name, and ask for another name if normalization produces an empty slug.

On every invocation:

1. Resolve and validate the selected project root.
2. Read `${project_root}/.kproject/AGENTS.md` if present. It is optional, but its instructions apply for the rest of the thread alongside this skill.
3. Inspect `${project_root}/.kproject/issues/` if present. Determine each issue's recency from the latest modification of a file inside its folder, and inspect its stage files and task status. Do not create `.kproject/` merely to inspect it.
4. If an issue was named, make it the active issue. If its folder is absent, initialize it immediately (see below). If it exists, briefly orient the user to its available records and current task status.
5. If no issue was named and issues exist, ask the user with an interactive selector when available. Show up to five of the most recently updated issues and include `Create a new issue`; use the same choices in a compact numbered list when no selector UI is available. Do not choose an issue on the user's behalf. If no issues exist, ask for a new issue name directly.
6. If the same message also contains substantive context or a stage action, first establish the issue, record the input, and then perform the requested stage. Do not stop after initialization unless selection/initialization was the only requested action.

Keep the selected project root and active issue for follow-up messages in the thread. A later main `kproject` invocation establishes a fresh selection using the resolution rules above.

## Issue layout

```text
${project_root}/
└── .kproject/
    ├── AGENTS.md                         # optional, project-level instructions
    └── issues/
        └── ${issue_name}/
            ├── input.md                  # original user prompt, context, goals
            ├── problem.md                # researched problem and open questions
            ├── design.md                 # proposed solution and design questions
            ├── tasks.md                  # ordered implementation tasks and status
            └── files/                    # task-specific generated or source files
```

Use `files/` for artifacts belonging to the issue: user-requested reports, generated output, downloaded or extracted research material when appropriate, and planning inputs that should persist. Do not put ordinary production-code changes there.

## Stages

### 1. Initialization

Initialization is triggered by the main `kproject [<issue_name>] [-p <project_path>]` command when the named issue folder is missing.

Create `.kproject/`, `.kproject/issues/`, `.kproject/issues/${issue_name}/`, and its `files/` directory as necessary. Create a blank `input.md` in the issue folder. Do not create `problem.md`, `design.md`, or `tasks.md` during initialization. Confirm the initialized issue and selected project concisely.

`input.md` is the durable record of the user's prompt, including context, explanations, goals, constraints, supplied links, and requested deliverables. Keep it blank only when the selecting command contains no substantive issue input. If the initialization message includes context or a stage action, create the blank file first, then populate it before doing that work. Preserve the original request; append later clarifications as clearly dated or labeled additions rather than replacing it.

`input.md` starts blank by design. When it is first populated, read `assets/input.template.md` and use it as an adaptable structure.

### 2. Research, design, and tasks

Work with `problem.md`, `design.md`, and `tasks.md`. Use problem → design → tasks as the normal full-research sequence, but iterate: correct upstream records when new information changes them, then update affected downstream records.

Always process every data source the user supplies or references, including Jira tickets, GitHub repositories or issues, linked documents, APIs, logs, screenshots, and files. Record each source in `input.md`, inspect it before drawing conclusions or asking questions it can answer, and cite or identify it alongside relevant findings in the stage records. Also inspect relevant local code. If a source is inaccessible, keep its reference, record the access constraint, and ask only for the access or information genuinely needed. Never silently omit a source or claim to have inspected one that was unavailable.

#### `problem.md`

Research and describe the current situation, desired outcome, evidence, scope, acceptance criteria, and constraints. `Open Questions` is a first-class section: keep material unknowns explicit, actionable, and current. Do not bury uncertainty in prose or delete questions merely because planning continues.

Read `assets/problem.template.md` before first creating this file and adapt it to the issue rather than mechanically retaining irrelevant headings.

#### `design.md`

Develop the solution approach, technical design, specifications, alternatives, risks, dependencies, and decisions required for implementation. Keep a prominent `Open Questions` section for unresolved design decisions; it must explain what decision is needed, why it matters, and any known options or evidence.

Read `assets/design.template.md` before first creating this file and adapt it as needed.

#### `tasks.md`

Turn the settled design into ordered, stable-numbered implementation tasks. Each task must be self-contained enough that an engineer can implement it without re-deriving essential context. Include affected files and verification where known.

Read `assets/tasks.template.md` before first creating this file and adapt it as needed.

### 3. Implementation

Implement tasks from `tasks.md`. Update each task's status as it is completed, not only at the end. Run relevant tests and project-required verification for each finished task.

If new work is discovered, append a newly numbered task; never renumber existing tasks. If blocked, use an informative status such as `Blocked`, explain the blocker in the task, and tell the user before proceeding. Update `problem.md` or `design.md` when implementation exposes an incorrect assumption or a newly material decision.

## Stage selectors in a user prompt

Treat these selectors as the user asking to perform work in the corresponding stage. They may appear alongside an ordinary natural-language request.

| Selector | Requested work |
|---|---|
| `--problem` | Work on `problem.md` only. Research the issue and its supplied sources, then update the problem statement, acceptance criteria, and open questions without automatically producing a design. |
| `--design` | Work on `design.md` only. Research as needed, but do not automatically create or update tasks. |
| `--questions` | Process open questions, especially `design.md` → `Open Questions`. Research answerable questions first; ask the user only for material decisions evidence cannot settle. Record resolutions and update their downstream implications while retaining or clearly marking unresolved questions. |
| `--tasks` | Work on `tasks.md`. Derive or revise tasks from the available design; surface missing design information rather than inventing material decisions. Do not implement the tasks. |
| `--dev` | Implement from `tasks.md`: all unfinished tasks unless the user names specific task numbers. Require `tasks.md`; do not silently create a plan when it is absent. |

Without a selector, route clear natural-language requests as follows:

| User says something like... | Action |
|---|---|
| "research and plan this issue" | Full research sequence: `problem.md` → `design.md` → `tasks.md` |
| "work on the problem" / "define the issue" | `problem.md` only |
| "draft/revise the design" / "solution" | `design.md` only |
| "make the tasks" / "plan the work" | `tasks.md` only |
| "answer the open questions" | Open-question processing, prioritizing `design.md` |
| "perform the plan" / "implement the plan" | `--dev`: first unfinished task through the end |
| "do tasks 1, 2, 3" | `--dev`: exactly those task numbers |

If the requested action is unclear, or a material product/design decision cannot be discovered from code or provided sources, ask a focused question. Otherwise proceed using the active issue.

## Task status conventions

Each task in `tasks.md` retains its number for the issue's lifetime. Default statuses are `To Do` and `Done`; use `In Progress`, `Blocked`, or `Skipped` when those communicate real state better. A task's status must make it unambiguous whether it counts as finished, because "implement the plan" begins at the first unfinished task.
