---
name: kproject
description: "Guide for working with kproject, a local, file-based issue workflow that lives inside a software project's own repo under a .kproject/ directory. Trigger this skill immediately whenever a message starts with 'kproject' followed by a path (e.g. 'kproject /path/to/project/root') — that establishes the active project root for the rest of the thread. Also use it for any follow-up request in an already-active kproject thread, such as creating a new issue, researching or planning an issue (drafting ISSUE.md, SOLUTION.md, or PLAN.md), being asked to work on 'the issue file', 'the solution file', or 'the plan', being asked to perform, implement, or execute the plan, or being asked to do specific numbered plan items. Always prefer this skill over ad-hoc file editing whenever files under .kproject/issues/ (per issue) are involved."
compatibility: "Requires an agent with read/write access to the project's local filesystem (e.g. Claude Code, a Filesystem MCP connector, or a container filesystem tool)."
---

# kproject

kproject is a lightweight, file-based issue workflow that lives inside a software project's own repository, under `.kproject/`. Each issue moves through three phases — initializing, researching/planning, and implementing — and every phase's output is a plain Markdown file that a human and an agent can both pick up later. This skill defines how to operate that workflow: which files to create, in what order, and how to route the user's natural-language requests to the right action.

This skill assumes some other tool already gives you read/write access to the project's files (a filesystem connector, Claude Code, a bash shell, etc.) — kproject itself is just the convention for *what* to read and write and *when*.

## Activating a project

The skill is invoked explicitly, with a message like:

> kproject /path/to/project/root

Treat the given path as the **active project root** for the rest of the thread. If the user later sends another `kproject <path>` with a different path, switch the active root to that new one. If a later request is ambiguous about which project it applies to (more than one root mentioned, unclear which is current), ask — don't guess at a filesystem path.

As soon as a project is activated:

1. Confirm the path exists and is a directory.
2. Look for `${project_root}/.kproject/AGENTS.md`. It's optional — if it's missing, move on. If it exists, read it and treat it exactly like a project-level `AGENTS.md`: it carries the project's own conventions, constraints, and instructions, and applies for the rest of the thread alongside this skill, not instead of it.
3. Look for `${project_root}/.kproject/issues/`. If it exists, list what's there so you have situational awareness — which issues exist, which of ISSUE.md / SOLUTION.md / PLAN.md each one has, and roughly how far each PLAN.md has progressed. If `.kproject/` or `.kproject/issues/` don't exist yet, that's fine — this project simply hasn't used kproject before, and both get created the first time an issue is initialized.

Give the user a brief orientation after activation (which project, what issues already exist, whether an AGENTS.md was found) rather than silently proceeding — but keep it short, this isn't a report.

## Project layout

```
${project_root}/
└── .kproject/
    ├── AGENTS.md                 # optional, project-level instructions for the agent
    └── issues/
        └── ${issue_name}/
            ├── ISSUE.md           # the problem, written during research
            ├── SOLUTION.md        # the concept + specs, written during research
            ├── PLAN.md            # the implementation plan, written during planning
            └── ...                # anything else the agent or user adds along the way
```

`${issue_name}` is a kebab-case slug (lowercase, hyphen-separated). If the user gives a name that isn't already a clean slug ("Add Animation to the Button"), convert it (`add-animation-to-the-button`) and mention the slug you used so they can correct it if they'd rather have something else.

## The three phases

### 1. Initialization — creating the issue

Triggered by requests like "create a new issue called `add-animation`" or "start a new issue for X."

This phase only creates the empty issue directory: `.kproject/issues/${issue_name}/`. Create `.kproject/` and `.kproject/issues/` too if this is the project's first issue. Don't create ISSUE.md, SOLUTION.md, or PLAN.md here — those belong to research, even if the user hands you a one-line description along with the name. Hold onto anything they told you at this point; it's the seed for ISSUE.md once research starts.

Confirm the issue was created, and either move into research if that's clearly what they want next, or ask.

### 2. Research (aka Plan) — ISSUE.md → SOLUTION.md → PLAN.md

Triggered by requests like "research and plan the issue," "let's work on the issue file," "draft the solution," or similar. This phase produces the three research files, in order, but it's a genuinely iterative process, not a template-filling exercise:

1. **ISSUE.md** — Understand and describe the problem. Read the relevant parts of the codebase first (don't ask the user things you can find out yourself), then interview them for the parts that aren't in the code: intent, constraints, priorities, what "done" looks like. Write the problem description and Acceptance Criteria once there's enough to make them concrete and testable — vague acceptance criteria make the later plan hard to scope.
2. **SOLUTION.md** — Once the problem is settled, work out how to solve it: the approach, the technical design, and enough specification (interfaces, data shapes, files touched, key decisions) that another engineer could pick it up without asking you anything. Interview the user again where there are real design decisions to make (trade-offs, library choices, scope calls) — don't silently decide things that materially change the shape of the solution.
3. **PLAN.md** — Break the solution into an ordered, numbered list of tasks. Each task should be small and self-contained enough that a junior engineer could pick up any single one and implement it without re-deriving context from the rest of the plan.

Treat "in this order" as the default path, not a one-way gate. If, while writing SOLUTION.md, you or the user realize ISSUE.md was incomplete or wrong, go back and fix it before continuing — the same goes for PLAN.md surfacing a gap in SOLUTION.md. The user may also explicitly ask to revisit an earlier file ("let's revise the issue file after all"); handle that the same way — update it, then check whether anything downstream needs to change as a result.

A request that names one file specifically — "the issue file," "the solution," "the plan" — means work on that file alone, not the full sequence:

- "issue" / "issue file" → `ISSUE.md`
- "solution" / "solution file" / "concept" → `SOLUTION.md`
- "plan" / "plan file" → `PLAN.md`

Read `assets/ISSUE.template.md`, `assets/SOLUTION.template.md`, or `assets/PLAN.template.md` before writing the corresponding file for the first time, and use it as a starting structure. It's a scaffold, not a form: drop sections that don't apply to a given issue and add sections a particular problem genuinely needs.

### 3. Implementation (aka Develop) — working the plan

Triggered by requests like "perform the plan," "implement the plan," or "do plan items 1, 2, 3."

- **"Perform/implement the plan"** (no items named): start from the first item that isn't `Done` and work forward to the end of the list, implementing each one and updating its status as you finish it. If you hit a blocker on an item, mark it accordingly (see conventions below) rather than silently skipping it, and tell the user before moving on.
- **Specific items named** ("do items 1, 2, 3"): implement exactly those, regardless of the status of items in between. Update the status of each one you touch.

Update PLAN.md as you finish each item, not just at the end — status should always reflect the true current state, since it's what tells you (or a future thread) where "the first undone item" actually is. If the project has tests or its AGENTS.md specifies a verification step, run it as part of finishing each item.

## Plan item and status conventions

Each item in PLAN.md keeps a stable number for the life of the issue — that's how the user refers back to it ("do items 1, 2, 3"). If new tasks come up mid-implementation, append them with new numbers at the end rather than renumbering the list, even if they'd logically slot in earlier.

Default statuses: `To Do` and `Done`. Use situation-specific ones when they're actually informative — `In Progress`, `Blocked`, `Skipped` — rather than forcing everything into the two defaults. Whatever status you use, keep it unambiguous about whether the item counts as finished, since "first undone item" and "perform the plan" depend on that.

## Quick reference: request → action

| User says something like... | Action |
|---|---|
| `kproject /path/to/project` | Activate that project root — see Activating a project |
| "Create a new issue called `X`" | Initialization: create `.kproject/issues/x/` |
| "Research and plan the issue" | Research phase, full sequence: ISSUE.md → SOLUTION.md → PLAN.md |
| "Let's work on the issue file" | Research phase, ISSUE.md only |
| "Draft/revise the solution" | Research phase, SOLUTION.md only |
| "Let's do the plan" (before one exists) | Research phase, PLAN.md only |
| "Perform the plan" / "implement the plan" | Implementation, from first undone item to the end |
| "Do plan items 1, 2, 3" | Implementation, exactly those items |

If a request doesn't cleanly match one of these and it's not obvious from context which issue or phase it refers to, ask — don't guess at which files to touch.
