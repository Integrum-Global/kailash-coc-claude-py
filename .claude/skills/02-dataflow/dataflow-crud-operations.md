---
name: dataflow-crud-operations
description: "Use 11 auto-generated DataFlow nodes for CRUD operations. Use when DataFlow CRUD, generated nodes, UserCreateNode, UserReadNode, create read update delete, basic operations, or single record operations."
---

# DataFlow CRUD Operations

11 auto-generated workflow nodes for Create, Read, Update, Delete, List, Upsert, Count + 4 Bulk nodes.

## CRITICAL: CreateNode vs UpdateNode Patterns

| Node Type      | Pattern                  | Example                                                |
| -------------- | ------------------------ | ------------------------------------------------------ |
| **CreateNode** | **FLAT** fields          | `{"name": "Alice", "email": "a@ex.com"}`               |
| **UpdateNode** | **NESTED** filter+fields | `{"filter": {"id": 1}, "fields": {"name": "Updated"}}` |

```python
# CreateNode: FLAT individual fields
workflow.add_node("UserCreateNode", "create", {
    "name": "Alice",
    "email": "alice@example.com",
    "age": 30
})

# UpdateNode: NESTED filter + fields
workflow.add_node("UserUpdateNode", "update", {
    "filter": {"id": 1},
    "fields": {"name": "Alice Updated", "age": 31}
})
```

**Auto-managed fields** -- NEVER include `created_at` or `updated_at` in parameters.

## Core Pattern

```python
from dataflow import DataFlow
from kailash.workflow.builder import WorkflowBuilder
from kailash.runtime.local import LocalRuntime

db = DataFlow()

@db.model
class User:
    name: str
    email: str
    active: bool = True

# Generates: UserCreateNode, UserReadNode, UserUpdateNode, UserDeleteNode,
#   UserListNode, UserUpsertNode, UserCountNode,
#   UserBulkCreateNode, UserBulkUpdateNode, UserBulkDeleteNode, UserBulkUpsertNode

workflow = WorkflowBuilder()

workflow.add_node("UserCreateNode", "create", {
    "name": "Alice", "email": "alice@example.com"
})
workflow.add_node("UserReadNode", "read", {"filter": {"id": 1}})
workflow.add_node("UserUpdateNode", "update", {
    "filter": {"id": 1}, "fields": {"active": False}
})
workflow.add_node("UserDeleteNode", "delete", {"filter": {"id": 1}})
workflow.add_node("UserListNode", "list_users", {
    "filter": {"active": True}, "limit": 10
})

runtime = LocalRuntime()
results, run_id = runtime.execute(workflow.build())
```

## Node Reference

| Node                | Parameters                                                      | Notes                                |
| ------------------- | --------------------------------------------------------------- | ------------------------------------ |
| `{Model}CreateNode` | All model fields (flat)                                         | `return_id`, `validate` options      |
| `{Model}ReadNode`   | `filter` (by id or conditions)                                  | `raise_on_not_found` option          |
| `{Model}UpdateNode` | `filter` + `fields` (nested)                                    | `return_updated`, `validate` options |
| `{Model}DeleteNode` | `filter`                                                        | `soft_delete`, `hard_delete` options |
| `{Model}ListNode`   | `filter`, `order_by`, `limit`, `offset`, `fields`, `count_only` | MongoDB-style filters                |
| `{Model}UpsertNode` | Model fields + conflict resolution                              | --                                   |
| `{Model}CountNode`  | `filter`                                                        | Count matching records               |

### ListNode Filters (MongoDB-style)

```python
workflow.add_node("UserListNode", "list", {
    "filter": {"active": True, "age": {"$gt": 18}},
    "order_by": ["-created_at"],
    "limit": 10, "offset": 0,
    "fields": ["id", "name", "email"]
})
```

## Dynamic Values: Use Connections, NOT Template Syntax

```python
# WRONG: ${} conflicts with PostgreSQL
workflow.add_node("OrderCreateNode", "create", {
    "customer_id": "${create_customer.id}"  # FAILS
})

# CORRECT: workflow connections
workflow.add_node("OrderCreateNode", "create", {"total": 100.0})
workflow.add_connection("create_customer", "id", "create", "customer_id")
```

## Datetime Auto-Conversion

ISO 8601 strings automatically convert to datetime objects on all CRUD nodes.

```python
workflow.add_node("PythonCodeNode", "gen_ts", {
    "code": """
from datetime import datetime
result = {"registration_date": datetime.now().isoformat()}
    """
})
workflow.add_node("UserCreateNode", "create", {
    "name": "Alice", "email": "alice@example.com",
    "registration_date": "{{gen_ts.registration_date}}"  # ISO string -> datetime
})
```

Both datetime objects and ISO strings accepted. Formats: `2024-01-01T12:00:00`, with microseconds, `Z` suffix, or timezone offset.

## Complete CRUD Workflow Example

```python
workflow = WorkflowBuilder()
workflow.add_node("UserCreateNode", "create", {
    "name": "Alice", "email": "alice@example.com"
})
workflow.add_node("UserReadNode", "read", {"filter": {}})
workflow.add_connection("create", "id", "read", "filter.id")

workflow.add_node("UserUpdateNode", "update", {
    "filter": {}, "fields": {"active": False}
})
workflow.add_connection("read", "id", "update", "filter.id")

workflow.add_node("UserListNode", "list_inactive", {"filter": {"active": False}})

runtime = LocalRuntime()
results, run_id = runtime.execute(workflow.build())
created_user = results["create"]["result"]
inactive_users = results["list_inactive"]["result"]
```

## Soft Delete Pattern

```python
@db.model
class Customer:
    name: str
    email: str
    __dataflow__ = {'soft_delete': True}

workflow.add_node("CustomerDeleteNode", "soft_delete", {
    "filter": {"id": 123}, "soft_delete": True
})
workflow.add_node("CustomerListNode", "all", {
    "filter": {}, "include_deleted": True  # Include soft-deleted
})
```

## Troubleshooting

| Issue                             | Solution                                   |
| --------------------------------- | ------------------------------------------ |
| `Node 'UserCreateNode' not found` | Add `@db.model` decorator                  |
| `Missing required field`          | Provide value or add default to model      |
| `duplicate key`                   | Check existing record before create        |
| Missing `.build()`                | Always `runtime.execute(workflow.build())` |

<!-- Trigger Keywords: DataFlow CRUD, generated nodes, UserCreateNode, UserReadNode, UserUpdateNode, UserDeleteNode, UserListNode, create read update delete, basic operations, single record, DataFlow operations -->
