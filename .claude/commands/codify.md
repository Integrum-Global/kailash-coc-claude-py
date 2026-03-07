---
name: codify
description: "Load phase 05 (codify) for the current workspace. Create project agents and skills."
---

## Workspace Resolution

1. If `$ARGUMENTS` specifies a project name, use `workspaces/$ARGUMENTS/`
2. Otherwise, use the most recently modified directory under `workspaces/` (excluding `instructions/`)
3. If no workspace exists, ask the user to create one first
4. Read all files in `workspaces/<project>/briefs/` for user context (this is the user's input surface)

## Phase Check

- Read `workspaces/<project>/04-validate/` to confirm validation passed
- Read `docs/` and `docs/00-authority/` for knowledge base
- Output goes to `.claude/agents/project/` and `.claude/skills/project/`

## Workflow

### 1. Deep knowledge extraction

Using as many subagents as required, peruse `docs/`, especially `docs/00-authority/`.

- Ultrathink and read beyond the docs into the intent of this project/product
- Understand the roles and use of agents, skills, docs:
  - **Agents** — What to do, how to think about this, what can it work with, following procedural directives
  - **Skills** — Distilled knowledge that agents can achieve 100% situational awareness with
  - **`docs/`** — Full knowledge base

### 2. Create/Update agents

Create agents in `.claude/agents/project/`.

- Research how Claude subagents should be written, best practices, and how they should be used
- Reference `.claude/agents/_subagent-guide.md` for agent format
- Specialized agents whose combined expertise cover 100% of this codebase/project/product
- Use-case agents that can work across skills and guide the main agent in coordinating work best done by specialized agents

### 3. Create/Update skills

Create accompanying skills in `.claude/skills/project/`.

- Research how Claude skills should be written, best practices, and how they should be used
- Reference `.claude/guides/claude-code/06-the-skill-system.md` for skill format
- Do not create any more subdirectories
- Ensure single entry point for skills (`SKILL.md`) that references multiple skills files in the same directory
  - Skills must be as detailed as possible so agents can deliver most of their work just by using them
  - Do not treat skills as the knowledge base
    - Should contain the most critical information and logical links/frameworks between knowledge base content
    - Should REFERENCE instead of repeating the knowledge base in `docs/`

### 4. Red team the agents and skills

Validate that generated agents and skills are correct, complete, and secure.

## Agent Teams

Deploy these agents as a team for codification:

**Knowledge extraction team:**

- **deep-analyst** — Identify core patterns, architectural decisions, and domain knowledge worth capturing
- **requirements-analyst** — Distill requirements into reusable agent instructions
- **coc-expert** — Ensure agents and skills follow COC five-layer architecture (codification IS Layer 5 evolution)

**Creation team:**

- **documentation-validator** — Validate that skill examples are correct and runnable
- **intermediate-reviewer** — Review agent/skill quality before finalizing

**Validation team (red team the agents and skills):**

- **gold-standards-validator** — Ensure agents follow the subagent guide and skills follow the skill system guide
- **testing-specialist** — Verify any code examples in skills are testable
- **security-reviewer** — Audit generated agents/skills for prompt injection vectors, insecure patterns, or secrets exposure (codified artifacts persist across all future sessions)
