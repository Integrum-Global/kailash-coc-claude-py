---
name: validate-dataflow-patterns
description: "Validate DataFlow compliance patterns. Use when asking 'validate dataflow', 'dataflow compliance', or 'check dataflow code'."
---

# Validate DataFlow Patterns

> **Skill Metadata**
> Category: `validation`
> Priority: `MEDIUM`
> SDK Version: `0.9.25+`

## DataFlow Compliance Checks

```python
# ✅ CORRECT: Use @db.model decorator
from dataflow import DataFlow

db = DataFlow("sqlite:///app.db")

@db.model
class User:
    id: str
    email: str

# Auto-generates 11 nodes: UserCreateNode, UserReadNode, UserUpsertNode, UserCountNode, etc.

# ❌ WRONG: Manual node creation for database ops
# workflow.add_node("DatabaseExecuteNode", "create_user", {
#     "query": "INSERT INTO users..."
# })
```

## Validation Rules

1. **Use @db.model** - Not manual SQL
2. **Use generated nodes** - UserCreateNode, UserReadNode
3. **String IDs** - Required for all models
4. **No direct SQLAlchemy** - DataFlow handles it
5. **Verify writes persisted** - DataFlow silently ignores unknown parameters. Always read back after writes to confirm data was actually stored. Use the correct param names: `filter`/`fields` for UpdateNode, NOT `conditions`/`updates`.

### Silent No-Op Detection

DataFlow's UpdateNode silently ignores unknown parameter names. This produces false positives in tests:

```python
# ❌ WRONG: Uses wrong param names — silently writes nothing, returns success
workflow.add_node("UserUpdateNode", "update_user", {
    "conditions": {"id": user_id},  # WRONG — should be "filter"
    "updates": {"name": "New Name"}  # WRONG — should be "fields"
})

# ✅ CORRECT: Uses actual UpdateNode parameter names
workflow.add_node("UserUpdateNode", "update_user", {
    "filter": {"id": user_id},
    "fields": {"name": "New Name"}
})

# ✅ ALWAYS: Read back to verify the write persisted
workflow.add_node("UserReadNode", "verify_update", {
    "filter": {"id": user_id}
})
# Assert: verify_update result has name == "New Name"
```

<!-- Trigger Keywords: validate dataflow, dataflow compliance, check dataflow code, dataflow patterns -->
