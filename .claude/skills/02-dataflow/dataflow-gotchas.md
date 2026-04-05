---
name: dataflow-gotchas
description: "Common DataFlow mistakes and misunderstandings. Use when DataFlow issues, gotchas, common mistakes DataFlow, troubleshooting DataFlow, or DataFlow problems."
---

# DataFlow Common Gotchas

## Quick Reference

- **Primary key MUST be `id`** -- not `user_id`, `model_id`, anything else
- **CreateNode = flat fields, UpdateNode = nested filter+fields** -- different patterns
- **Never set `created_at`/`updated_at`** -- auto-managed (auto-stripped with warning)
- **NOT an ORM** -- models are schemas, use workflow nodes not `User().save()`
- **Use connections, NOT `${}` template syntax** -- conflicts with PostgreSQL
- **Result access**: ListNode->`records`, CountNode->`count`, ReadNode->dict directly
- **Use Express for APIs**: `db.express.create()` is 23x faster than workflows

## #1: Auto-Managed Timestamp Fields (FIXED)

DataFlow auto-strips `created_at`/`updated_at` with a warning. Best practice: never include them.

```python
# BEST PRACTICE
workflow.add_node("ModelUpdateNode", "update", {
    "filter": {"id": id},
    "fields": data  # DataFlow sets updated_at automatically
})
```

## #2: Sync Methods in Async Context (DF-501)

```python
# WRONG - RuntimeError: DF-501
@app.on_event("startup")
async def startup():
    db.create_tables()  # Sync in async context!

# CORRECT
@asynccontextmanager
async def lifespan(app):
    await db.create_tables_async()
    yield
    await db.close_async()
app = Nexus(lifespan=lifespan)
```

| Sync Method       | Async Method            |
| ----------------- | ----------------------- |
| `create_tables()` | `create_tables_async()` |
| `close()`         | `close_async()`         |

Sync methods still work in sync context (CLI, scripts).

## #3: Docker/Async Deployment (FIXED)

`auto_migrate=True` now works in Docker/async. Uses sync DDL drivers (psycopg2/sqlite3) for table creation, async drivers (asyncpg/aiosqlite) for CRUD.

```python
# Zero-config Docker pattern
db = DataFlow("postgresql://...")  # auto_migrate=True default, works
```

In-memory SQLite uses URI shared-cache mode automatically. For manual control:

```python
db = DataFlow("postgresql://...", auto_migrate=False)
# Then use await db.create_tables_async() in lifespan
```

## #4: Empty Dict Truthiness Bug (FIXED)

Upgrade to latest DataFlow. Previously `{}` was treated as falsy in filters.

```python
# WRONG pattern in custom code
if filter_dict:          # Empty dict is falsy!
    process_filter()

# CORRECT
if "filter" in kwargs:   # Check key existence
    process_filter()
```

## #5: Primary Key MUST Be 'id'

```python
# WRONG
@db.model
class User:
    user_id: str  # FAILS

# CORRECT
@db.model
class User:
    id: str  # REQUIRED
    name: str
```

## #6: CreateNode vs UpdateNode Pattern

```python
# CreateNode: FLAT
workflow.add_node("UserCreateNode", "create", {
    "id": "user_001", "name": "Alice", "email": "alice@example.com"
})

# UpdateNode: NESTED filter + fields
workflow.add_node("UserUpdateNode", "update", {
    "filter": {"id": "user_001"},
    "fields": {"name": "Alice Updated"}
})
```

## #7: NOT an ORM

```python
# WRONG
user = User(name="John")  # Not instantiable
user.save()               # No save() method

# CORRECT
workflow.add_node("UserCreateNode", "create", {"name": "John"})
```

## #8: Template Syntax Conflicts with PostgreSQL

```python
# WRONG
"customer_id": "${create_customer.id}"  # Conflicts with PostgreSQL

# CORRECT: Use connections
workflow.add_connection("create_customer", "id", "create", "customer_id")
```

## #9: Nexus Integration

```python
# WRONG - dataflow_config does NOT exist in Nexus
nexus = Nexus(dataflow_config={"integration": db})

# CORRECT
nexus = Nexus(auto_discovery=False)  # CRITICAL: prevents blocking
nexus.register("create_product", workflow.build())
```

## #10: Result Access Patterns

| Node Type  | Access Pattern                          |
| ---------- | --------------------------------------- |
| ListNode   | `results["list"]["records"]` -> list    |
| CountNode  | `results["count"]["count"]` -> int      |
| ReadNode   | `results["read"]` -> dict or None       |
| CreateNode | `results["create"]` -> record           |
| UpdateNode | `results["update"]` -> record           |
| UpsertNode | `results["upsert"]["record"]` -> record |

## #11: soft_delete Auto-Filters

```python
@db.model
class Patient:
    id: str
    deleted_at: Optional[str] = None
    __dataflow__ = {"soft_delete": True}

# Default: excludes soft-deleted
workflow.add_node("PatientListNode", "list", {"filter": {}})

# Include deleted
workflow.add_node("PatientListNode", "all", {
    "filter": {}, "include_deleted": True
})
```

## #12: Sort/Order Parameters

All three formats work:

```python
"order_by": ["-created_at", "name"]                           # Prefix format
"sort": [{"field": "created_at", "order": "desc"}]            # Explicit format
"order_by": [{"created_at": -1}, {"name": 1}]                 # Dict format
```

## Historical Fixes (Upgrade to Latest)

- **String IDs**: Now fully supported (were converted to int)
- **VARCHAR(255)**: Now TEXT type (was truncating)
- **DateTime serialization**: ISO strings auto-converted
- **Multi-instance isolation**: Fixed (models no longer leak between instances)

<!-- Trigger Keywords: DataFlow issues, gotchas, common mistakes DataFlow, troubleshooting DataFlow, DataFlow problems, DataFlow errors, not working, DataFlow bugs -->
