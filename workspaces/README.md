# Workspaces

This is the human-facing work surface of the COC setup. All project work happens here.

## Structure

```
workspaces/
  instructions/          # Reference templates (DO NOT edit directly)
  <project-name>/        # One directory per sprint/session/run
    instructions/        # Copied from reference, customized per project
    01-analysis/         # Research, value propositions, platform analysis
    02-plans/            # Implementation plans, architecture decisions
    03-user-flows/       # Detailed user workflow storyboards
    04-validate/         # Validation results, red team findings
    05-gaps/             # Identified gaps and remediation plans
    todos/               # Task tracking (active/, completed/)
    docs/                # Project documentation and authority docs
    src/                 # Backend codebase
    apps/                # Frontend codebases (web/, mobile/)
```

## How It Works

### 1. Create a workspace

Create a new directory under `workspaces/` for each sprint, session, or project run. Copy `workspaces/instructions/` into it.

```
workspaces/my-saas-app/
  instructions/          # Copied from workspaces/instructions/
```

### 2. Follow the phases in sequence

Each phase has an instruction file. Go through them in order — do not skip or reorder.

| Phase | File                                     | Output                                               | Gate              |
| ----- | ---------------------------------------- | ---------------------------------------------------- | ----------------- |
| 01    | `instructions/01-analyze.md`             | `01-analysis/`, `02-plans/`, `03-user-flows/`        | Human review      |
| 02    | `instructions/02-todos.md`               | `todos/active/`                                      | Human approval    |
| 03    | `instructions/03-implement.md`           | `src/`, `apps/`, `docs/`                             | All tests passing |
| 04    | `instructions/04-validate.md`            | `04-validate/`                                       | Red team sign-off |
| 05    | `instructions/05-create-agent-skills.md` | `.claude/agents/project/`, `.claude/skills/project/` | Human review      |

### 3. Iterate within and between phases

- Phase 03 (implement) is designed to be repeated until all todos move from `todos/active/` to `todos/completed/`
- Phase 04 (validate) feeds back into 03 — gaps found trigger implementation fixes
- Each iteration documents its findings in the workspace

## Key Principles

**The workspace is institutional memory.** It accumulates decisions, rationale, and state across sessions. Agents read it to restore context. It prevents amnesia.

**Modification protocol.** If phase instructions need changing during a session, update the workspace's `instructions/` file first, get human confirmation, then act. Never act on undocumented process changes.

**Reference templates are read-only.** `workspaces/instructions/` contains the canonical templates. Each project gets its own copy to customize.

**`.claude/` is infrastructure, not workbench.** The agents, skills, rules, and hooks are the cognitive infrastructure underneath. Do not modify them during project sessions — except for `project/` subdirectories in phase 05.
