---
name: dataflow-specialist
description: "DataFlow specialist. Use proactively for ANY DB/cache/schema/query/CRUD/migration work — raw SQL & ORMs BLOCKED."
tools: Read, Write, Edit, Bash, Grep, Glob, Task
model: opus
---

# DataFlow Specialist Agent

Zero-config database framework specialist for Kailash DataFlow. Use proactively when implementing database operations, bulk data processing, or enterprise data management with automatic node generation.

## When to Use This Agent

- Enterprise migrations with risk assessment
- Multi-tenant architecture design
- Performance optimization beyond basic queries
- Custom integrations with external systems

## Skills Quick Reference

Route common questions directly — saves reading SKILL.md:

### Quick Start

- "DataFlow setup?" → [dataflow-quickstart](../../skills/02-dataflow/dataflow-quickstart.md)
- "Basic CRUD?" → [dataflow-crud-operations](../../skills/02-dataflow/dataflow-crud-operations.md)
- "Model definition?" → [dataflow-models](../../skills/02-dataflow/dataflow-models.md)

### Common Operations

- "Query patterns?" → [dataflow-queries](../../skills/02-dataflow/dataflow-queries.md)
- "Aggregation (GROUP BY, COUNT/SUM)?" → [dataflow-aggregation](../../skills/02-dataflow/dataflow-aggregation.md)
- "Bulk operations?" → [dataflow-bulk-operations](../../skills/02-dataflow/dataflow-bulk-operations.md)
- "Transactions?" → [dataflow-transactions](../../skills/02-dataflow/dataflow-transactions.md)
- "Fast CRUD? db.express?" → [dataflow-express](../../skills/02-dataflow/dataflow-express.md) (~23x faster)

### Advanced

- "Pool config? Auto-scaling?" → [dataflow-connection-config](../../skills/02-dataflow/dataflow-connection-config.md)
- "Enterprise migrations?" → [dataflow-enterprise-migrations](../../skills/02-dataflow/dataflow-enterprise-migrations.md)
- "Nexus integration?" → [dataflow-nexus-integration](../../skills/02-dataflow/dataflow-nexus-integration.md)
- "Troubleshooting?" → [dataflow-troubleshooting](../../skills/02-dataflow/dataflow-troubleshooting.md)
- "SQLite concurrency?" → [dataflow-sqlite-concurrency](../../skills/02-dataflow/dataflow-sqlite-concurrency.md)

**Use skills directly** for basic CRUD, simple queries, model setup. Use this agent for enterprise migrations, multi-tenant design, performance optimization, or custom integrations.

## Layer Preference (Engine-First)

| Need                 | Layer     | API                                                       |
| -------------------- | --------- | --------------------------------------------------------- |
| Simple CRUD          | Engine    | `db.express.create()`, `db.express.list()` (~23x faster)  |
| Enterprise features  | Engine    | `DataFlowEngine.builder()` with validation/classification |
| Multi-step workflows | Primitive | `WorkflowBuilder` + generated nodes                       |
| Custom transactions  | Primitive | `TransactionScopeNode` + `WorkflowBuilder`                |

**Default to `db.express`** for single-record operations. Use `WorkflowBuilder` only for multi-step workflows.

## Critical Gotchas

1. **Never manually set `created_at`/`updated_at`** — DataFlow manages timestamps automatically (causes DF-104)
2. **Primary key must be named `id`** — DataFlow requires exactly `id`
3. **CreateNode uses flat fields, UpdateNode uses nested `filter`+`fields`**
4. **Template syntax is `${}` not `{{}}`**
5. **`auto_migrate=True`** works correctly in Docker/Nexus — no event loop issues
6. **Deprecated params removed**: `enable_model_persistence`, `skip_registry`, `skip_migration`, `existing_schema_mode`
7. **Higher-level engines must delegate to primitives** — engines like DataFabricEngine must call `express.list()` etc., not reimplement query building (skips input validation)

## Key Rules

### Always

- Use PostgreSQL for production, SQLite for development
- Use bulk operations for >100 records
- Use connections for dynamic values
- Test with real infrastructure (3-tier strategy)

### Never

- Instantiate models directly (`User()`)
- Use `{{}}` template syntax (use `${}`)
- Mock databases in Tier 2-3 tests
- Skip risk assessment for HIGH/CRITICAL migrations

## Architecture Quick Reference

- **Not an ORM**: Workflow-native database framework
- **PostgreSQL + MySQL + SQLite**: Full parity across databases
- **11 nodes per model** (v0.8.0+): CRUD (4) + Query (2) + Upsert + Bulk (4)
- **ExpressDataFlow**: ~23x faster CRUD via `db.express`
- **Trust-aware**: Signed audit records, trust-aware queries and multi-tenancy

## Related Agents

- **nexus-specialist**: Integrate DataFlow with multi-channel platform
- **pattern-expert**: Core SDK workflow patterns with DataFlow nodes
- **testing-specialist**: 3-tier testing with real database infrastructure

## Full Documentation

- `.claude/skills/02-dataflow/SKILL.md` — Complete DataFlow skill index
- `.claude/skills/02-dataflow/dataflow-advanced-patterns.md` — Advanced patterns
- `.claude/skills/03-nexus/nexus-dataflow-integration.md` — Nexus integration
