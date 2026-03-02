# Kailash Vibe CC Setup

This repository is the **COC (Cognitive Orchestration for Codegen) setup** for Claude Code — providing agents, skills, rules, and hooks for Kailash SDK development. All projects using this setup inherit these capabilities through the `.claude/` directory.

## Absolute Directives

These override ALL other instructions. They govern behavior before any rule file is consulted.

### 1. Framework-First

Never write code from scratch before checking whether the Kailash frameworks already handle it.

- Instead of direct SQL/SQLAlchemy/Django ORM → check with **dataflow-specialist**
- Instead of FastAPI/custom API gateway → check with **nexus-specialist**
- Instead of custom MCP server/client → check with **mcp-specialist**
- Instead of custom agentic platform → check with **kaizen-specialist**

### 2. .env Is the Single Source of Truth

All API keys and model names MUST come from `.env`. Never hardcode model strings like `"gpt-4"` or `"claude-3-opus"`. Root `conftest.py` auto-loads `.env` for pytest.

See `rules/env-models.md` for full details.

### 3. Implement, Don't Document

When you discover a missing feature, endpoint, or record — **implement or create it**. Do not note it as a gap and move on. The only acceptable skip is explicit user instruction.

See `rules/e2e-god-mode.md` and `rules/no-stubs.md` for enforcement details.

## Rules Index

| Concern                               | Rule File               | Scope                           |
| ------------------------------------- | ----------------------- | ------------------------------- |
| Agent orchestration & review mandates | `rules/agents.md`       | Global                          |
| E2E god-mode testing                  | `rules/e2e-god-mode.md` | `tests/e2e/**`, `**/*e2e*`      |
| API keys & model names                | `rules/env-models.md`   | `**/*.py`, `**/*.ts`, `.env*`   |
| Git commits, branches, PRs            | `rules/git.md`          | Global                          |
| No stubs, TODOs, or placeholders      | `rules/no-stubs.md`     | Global                          |
| Kailash SDK execution patterns        | `rules/patterns.md`     | `**/*.py`, `**/*.ts`, `**/*.js` |
| Security (secrets, injection)         | `rules/security.md`     | Global                          |
| 3-tier testing, no mocking Tiers 2-3  | `rules/testing.md`      | `tests/**`, `**/*test*`         |

## Agents

### Analysis & Planning

- **deep-analyst** — Failure analysis, complexity assessment
- **requirements-analyst** — Requirements breakdown, ADR creation
- **sdk-navigator** — Find patterns before coding
- **framework-advisor** — Choose Core SDK, DataFlow, Nexus, or Kaizen

### Framework Specialists

- **dataflow-specialist** — Database operations, auto-generated nodes
- **nexus-specialist** — Multi-channel platform (API/CLI/MCP)
- **kaizen-specialist** — AI agents, signatures, multi-agent coordination
- **mcp-specialist** — MCP server implementation

### Core Implementation

- **pattern-expert** — Workflow patterns, nodes, parameters
- **tdd-implementer** — Test-first development
- **intermediate-reviewer** — Code review after changes
- **gold-standards-validator** — Compliance checking

### Testing & Release

- **testing-specialist** — 3-tier strategy with real infrastructure
- **documentation-validator** — Test code examples
- **git-release-specialist** — Git workflows, CI, releases

## Skills Navigation

For SDK implementation patterns, see `.claude/skills/` — organized by framework (`01-core-sdk` through `05-kailash-mcp`) and topic (`06-cheatsheets` through `28-coc-reference`).

## Critical Execution Rules

```python
# ALWAYS: runtime.execute(workflow.build())
runtime = LocalRuntime()
results, run_id = runtime.execute(workflow.build())

# String-based nodes only
workflow.add_node("NodeType", "node_id", {"param": "value"})

# Docker/FastAPI: AsyncLocalRuntime
# CLI/Scripts: LocalRuntime
# Return structure is always (results, run_id)
```

## Kailash Platform

| Framework    | Purpose                                | Install                        |
| ------------ | -------------------------------------- | ------------------------------ |
| **Core SDK** | Workflow orchestration, 115+ nodes     | `pip install kailash`          |
| **DataFlow** | Zero-config database operations        | `pip install kailash-dataflow` |
| **Nexus**    | Multi-channel deployment (API+CLI+MCP) | `pip install kailash-nexus`    |
| **Kaizen**   | AI agent framework                     | `pip install kailash-kaizen`   |

All frameworks are built ON Core SDK — they don't replace it.
