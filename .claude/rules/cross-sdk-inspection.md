---
paths:
  - "**/src/**"
  - "**/tests/**"
---

# Cross-SDK Issue Inspection

## Scope

These rules apply to ALL bug fixes, feature implementations, and issue resolutions in BOTH BUILD repos (kailash-rs and kailash-py).

## MUST Rules

### 1. Cross-SDK Inspection on Every Issue

When an issue is found or fixed in ONE BUILD repo, you MUST inspect the OTHER BUILD repo for the same or equivalent issue.

**Why:** Bugs in shared architecture (trust plane, DataFlow, Nexus) almost always exist in both SDKs — fixing only one leaves users of the other SDK hitting the same issue.

**kailash-rs issue found → inspect kailash-py**:

- Does the Python SDK have the same bug?
- Does the Python SDK need the equivalent feature?
- File a GitHub issue on `terrene-foundation/kailash-py` if relevant.

**kailash-py issue found → inspect kailash-rs**:

- Does the Rust SDK have the same bug?
- Does the Rust SDK need the equivalent feature?
- File a GitHub issue on `esperie-enterprise/kailash-rs` if relevant.

### 2. Cross-Reference in Issues

When filing a cross-SDK issue, MUST include:

- Link to the originating issue in the other repo
- Tag: `cross-sdk` label
- Note: "Cross-SDK alignment: this is the [Rust/Python] equivalent of [link]"

**Why:** Without cross-references, the same bug gets fixed independently with different approaches, causing semantic divergence between SDKs that violates EATP D6.

### 3. EATP D6 Compliance

Per EATP SDK conventions (D6: independent implementation, matching semantics):

- Both SDKs implement features independently
- Semantics MUST match (same API shape, same behavior)
- Implementation details may differ (Rust idioms vs Python idioms)

**Why:** Semantic divergence between SDKs means code ported from Python to Rust (or vice versa) silently changes behavior, breaking user trust in the platform's cross-language promise.

### 3a. Structural API-Divergence Disposition

When the sibling SDK reports a bug at an API surface that this SDK does not expose (e.g. kailash-rs#424 `execute_raw(sql, params)` vs kailash-py `execute_raw(sql)`), the disposition MUST include BOTH of the following in the same PR:

1. A **Tier 2 test through the sibling-binding path** that exercises the real surface this SDK uses (e.g. Python Express `bulk_create` with shrinking-arity test matrix) to prove the issue does not reproduce here.
2. A **structural-invariant test locking the signature** — a test that asserts the current signature arity / kwarg shape / return type, so a future refactor toward the sibling SDK's shape fails loudly at test time rather than silently reopening the divergence.

```python
# DO — close the issue with both tests landed in the same PR
@pytest.mark.integration
async def test_execute_raw_sibling_binding_does_not_reproduce(db):
    # Exercises the real Express bulk_create path that kailash-rs#424 hit
    for batch_size in (1, 10, 100, 1000):
        rows = [{"id": i, "x": i} for i in range(batch_size)]
        await db.express.bulk_create("Row", rows)

def test_execute_raw_signature_invariant():
    """Lock the arity: Python's execute_raw takes (sql) only, not (sql, params)."""
    import inspect
    sig = inspect.signature(AsyncSQLDatabaseNode.execute_raw)
    positional = [p for p in sig.parameters.values() if p.kind != inspect.Parameter.VAR_KEYWORD]
    assert len(positional) == 2, "execute_raw(self, sql) — do not add params positional"

# DO NOT — close the sibling issue with a "not applicable" comment only
# gh issue close kailash-py#525 --comment "Python doesn't expose execute_raw(sql, params); closing"
# ↑ no test guards against a future refactor silently adding params; the divergence reopens invisibly
```

**BLOCKED rationalizations:**

- "Python doesn't have this API, so nothing to test"
- "The sibling tested it, we don't need to"
- "A structural test is over-engineering for a one-line divergence"
- "We'll add the guard when someone tries to reshape the API"

**Why:** "Not applicable" closes the ticket but leaves zero structural defense against a future refactor that drifts toward the sibling shape; the binding-path Tier 2 test proves the bug doesn't reproduce today, and the signature-invariant test proves it cannot reproduce tomorrow by a silent API change. Together they convert "closed without test" (the prior failure mode) into two grep-able assertions that survive every refactor.

Origin: kailash-py issue #525 / PR #528 (2026-04-19) — closed a structural cross-SDK divergence ticket with two tests (binding-path + signature invariant) rather than a "not applicable" comment.

### 4. Inspection Checklist

When closing any issue, verify:

- [ ] Does the other SDK have this issue? (check or file)
- [ ] If feature: is it in the other SDK's roadmap?
- [ ] If bug: could the same bug exist in the other SDK?
- [ ] Cross-reference added to both issues if applicable

**Why:** Closing without cross-SDK verification is the primary cause of feature drift — the checklist is the last gate before an issue is forgotten.

### 5. Cross-SDK Symbol Citations MUST Be Grep-Verified

When a template, skill, rule, or guide authored in one BUILD repo cites a concrete symbol from the sibling SDK (an import path, class name, method name, error type, env var name, or file path), the citation MUST be verified against the sibling SDK's working tree BEFORE the commit lands. If the symbol cannot be confirmed to exist, cite the abstract pattern instead — never invent the literal API.

Evidence of verification MUST appear in the commit body in the following format:

```
Verified against kailash-py:
- DialectManager.get_dialect → packages/kailash-dataflow/src/dataflow/adapters/dialect.py:446
- InvalidIdentifierError → packages/kailash-dataflow/src/dataflow/adapters/exceptions.py
```

```python
# DO — cite a verified Python symbol from a kailash-py-targeted skill
# (grep confirmed: DialectManager.get_dialect exists at the cited path)
from dataflow.adapters.dialect import DialectManager
from dataflow.adapters.exceptions import InvalidIdentifierError

dialect = DialectManager.get_dialect("postgresql")
```

```rust
// DO — cite a verified Rust symbol from a kailash-rs-targeted artifact
// (grep confirmed: BlockingFinalizerProtector exists at the cited path)
use kailash_core::runtime::BlockingFinalizerProtector;
```

```markdown
# DO NOT — cite a symbol whose existence in the sibling SDK was not grep-checked

A kailash-py rule references `BlockingFinalizerProtector` from kailash-rs as a
recommended pattern — but a grep against the kailash-rs working tree returns
zero matches. The cited symbol does not exist; the rule recommends a phantom API.
```

**BLOCKED rationalizations:**

- "The API looks like it should exist"
- "The name is standard Python / standard Rust"
- "The skill describes intent, not literal API"
- "We'll verify later"
- "The sibling SDK is on the roadmap to add this"
- "The example is illustrative, not load-bearing"

**Why:** Cross-SDK citations that are not grep-verified produce phantom APIs that downstream consumers wire into their code, only to discover at runtime that the cited symbol never existed. The bug-fix-side cross-SDK rule (MUST Rules 1–4 above) covers issue inspection; this rule covers the more common template / skill / rule authorship path, where invented APIs are even harder to detect because they look plausible. Single-grep verification at authorship time costs seconds and prevents the multi-session debugging cost that "I'll verify later" guarantees.

Origin: kailash-rs codify cycle (2026-04-19) — multiple cross-SDK API miscitations in skill authorship surfaced as HIGH red-team findings; precedent in earlier `kailash-coc-claude-rs#52` (Bedrock preset cited before SDK shipped).

## Examples

```
# Issue #52 in kailash-rs: per-request API key override
# → Filed kailash-py#12 as cross-SDK alignment
gh issue create --repo terrene-foundation/kailash-py \
  --title "feat(kaizen): per-request API key override" \
  --label "cross-sdk" \
  --body "Cross-SDK alignment with esperie-enterprise/kailash-rs#52"
```

## Automation

When the Claude Code Maintenance workflow is active, the fix job prompt
includes cross-SDK inspection as Phase 4.5 (between codify and commit).
When paused, this must be done manually.
