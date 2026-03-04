# Workspaces Scope

Work only within `workspaces/<project-name>/`. Read `.claude/` freely but never write to it except `.claude/agents/project/` and `.claude/skills/project/` during phase 05.

## Phase Contract

Follow phases in order. Each phase has a human gate — do not proceed without approval.

1. **01-analyze** — Research, analyze, plan. Output: `01-analysis/`, `02-plans/`, `03-user-flows/`
2. **02-todos** — Break plans into todos. Output: `todos/active/`. **Stop for human approval.**
3. **03-implement** — Build iteratively until all todos complete. Output: `src/`, `apps/`, `docs/`
4. **04-validate** — Red team with Playwright/Marionette MCP. Feeds back to 03.
5. **05-create-agent-skills** — Codify project knowledge into `.claude/agents/project/`, `.claude/skills/project/`

## Modification Protocol

To change how a phase works: update the workspace's `instructions/` file first, get human confirmation, then act. Never modify documentation or instructions outside the current workspace.

## Codebase Locations

- Backend: `src/`
- Web frontend: `apps/web/`
- Mobile frontend: `apps/mobile/`
- Authority docs: `docs/00-authority/` (includes project-level `CLAUDE.md`)
- Todos: `todos/active/`, `todos/completed/`
