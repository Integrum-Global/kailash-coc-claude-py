---
paths:
  - "tests/e2e/**"
  - "**/*e2e*"
  - "**/*playwright*"
---

# E2E God-Mode Testing Rules

## Scope

These rules apply to ALL end-to-end testing, validation, and browser-based test runs.

## ABSOLUTE RULES

### 1. God-Mode: Create ALL Missing Records

When running E2E tests and a required record is missing (404, 403, empty response):

**MUST**: Create the missing record immediately using the appropriate API or direct database access.
**MUST NOT**: Skip the test, document it as a "gap", or report it as "expected behavior".

**Pattern:**

```
1. Attempt operation
2. If 404/403/missing -> identify what's missing
3. Create the missing record via API (use admin credentials)
4. Retry the original operation
5. NEVER skip or move on
```

### 2. Adapt to Data Changes

Test data WILL change between runs. User emails, IDs, names may all differ.

**MUST**: Query the API to discover actual records before testing.
**MUST NOT**: Hardcode user emails, IDs, or other test data.

**Pattern:**

```
1. Before testing, query the list endpoint to get actual records
2. Find the record matching the role/type, not a hardcoded name
3. Use the ACTUAL values from the query
4. If no matching record exists, CREATE one
```

### 3. Implement Missing Endpoints

If an API endpoint doesn't exist and testing needs it:

**MUST**: Implement the endpoint immediately.
**MUST NOT**: Document it as a "limitation" and move on.

### 4. Follow Up on Failures

When an operation fails gracefully (error message displayed, no crash):

**MUST**: Investigate the root cause and implement a fix.
**MUST NOT**: Report "graceful failure" and move to next test.

**Pattern:**

```
1. Operation fails with error
2. Check backend logs for root cause
3. If missing API key -> verify .env is loaded
4. If missing record -> create it (Rule 1)
5. If missing endpoint -> implement it (Rule 3)
6. Retry the operation
7. Only move on after SUCCESS or explicit user instruction to skip
```

### 5. Assume Correct Role

During multi-persona testing, assume the role needed for each operation.

**Pattern:**

```
1. Need admin actions -> log in as admin/owner
2. Need to test restricted views -> log in as restricted user
3. Need to test RBAC -> try each role and verify access/denial
```

### 6. Verify State Persistence — Not Just API Success

When a write operation (create, update, delete) succeeds:

**MUST**: Verify the state actually persisted by reading it back after the operation.
**MUST NOT**: Treat API 200 or a success toast as sufficient proof that data was written.

**Pattern:**

```
1. Perform the write operation (create record, update field, etc.)
2. Navigate away from the page OR reload the page
3. Navigate back / query the record
4. Verify the state persisted (record exists, field is updated, modal doesn't reappear)
5. If state did NOT persist, investigate — DataFlow silently ignores unknown parameters
```

**Why this rule exists:** DataFlow `UpdateNode` silently ignores unknown parameter names (e.g., `conditions`/`updates` instead of `filter`/`fields`). The API returns success, the UI shows a toast, but zero bytes were written to the database. This class of bug is invisible to any test that only checks the API response.

**Known failure pattern:** Any test that asserts `response.status == 200` or `toast.isVisible()` without a read-back verification is testing the API contract, not the business outcome.

## Pre-E2E Checklist

Before starting ANY E2E test run:

- [ ] Backend running and healthy
- [ ] Frontend dev server running
- [ ] .env loaded and verified (check MODEL and API_KEY vars)
- [ ] Required users exist (query API, create if missing)
- [ ] Required resources exist (query API, create if missing)
- [ ] Access records exist (query API, create if missing)

## Exceptions

NO EXCEPTIONS for rules 1-4. If you cannot create a record, escalate to the user immediately.
Rule 5 exception: User explicitly says "only test as X role".
