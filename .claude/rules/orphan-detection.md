---
priority: 10
scope: path-scoped
paths:
  - "packages/**"
  - "src/**"
  - "**/tests/**"
---

# Orphan Detection Rules


<!-- slot:neutral-body -->

A class that no production code calls is a lie. Beautifully implemented orphans accumulate when a feature is built top-down — facade + accessor + documentation ship, but the wiring from the product's hot path to the new class never lands. The orphan keeps passing unit tests against itself, and the security/audit/governance promise it was supposed to deliver never executes once.

See `skills/16-validation-patterns/orphan-audit-playbook.md` for the Phase 5.11 post-mortem, the full detection protocol, and per-rule evidence.

## MUST Rules

### 1. Every `db.*` / `app.*` Facade Has a Production Call Site

Any attribute exposed on a public surface that returns a `*Manager`, `*Executor`, `*Store`, `*Registry`, `*Engine`, or `*Service` MUST have at least one call site inside the framework's production hot path within 5 commits of the facade landing. The call site MUST live in the same package as the framework, not in tests or downstream consumers.

```python
# DO — facade + hot-path call site in the same PR
async def list(self, model, ...):
    plan = await self._db.trust_executor.check_read_access(...)  # ← real call site

# DO NOT — facade ships, no hot-path call site
```

**Why:** Downstream consumers see the public attribute and build their security model around its documented behavior, silently bypassing protection because the framework never invokes the class on the actual data path. See playbook § Phase 5.11 for the 2,407-LOC trust-executor post-mortem.

### 2. Every Wired Manager Has a Tier 2 Integration Test

Once a manager is wired into the production hot path, its end-to-end behavior MUST be exercised by at least one Tier 2 integration test (real database, real adapter — `rules/testing.md` § Tier 2). Unit tests against the manager class in isolation are NOT sufficient.

```python
# DO — Tier 2 test against real infrastructure, asserts externally-observable effect
async def test_trust_executor_redacts_in_express_read(test_suite):
    rows = await db.express.list("Document")
    assert all(row["body"] == "[REDACTED]" for row in rows)

# DO NOT — Tier 1 in isolation proves the executor can redact, NOT that the framework calls it
```

**Why:** Unit tests prove the orphan implements its API. Integration tests prove the framework actually calls the orphan.

#### 2a. Crypto-Pair Round-Trip MUST Be Tested Through The Facade

Crypto wrappers that expose paired operations (`encrypt`/`decrypt`, `sign`/`verify`, `seal`/`unseal`, `wrap_key`/`unwrap_key`) MUST have at least one Tier 2 integration test that round-trips through the facade.

```python
# DO — round-trip through the facade
encrypted = db.crypto.encrypt(plaintext)
assert db.crypto.decrypt(encrypted) == plaintext

# DO NOT — two unit tests that mock each other's half (encrypt=GCM, decrypt=CBC drift, both pass)
```

**Why:** Crypto pairs are the manager-pattern at a smaller scale. Tier 2 round-trip tests are the only structural defense; no amount of Tier 1 coverage catches "encrypt uses GCM, decrypt uses CBC." See playbook § §2a.

### 3. Removed = Deleted, Not Deprecated

If a manager is found to be an orphan and the team decides not to wire it, it MUST be deleted from the public surface in the same PR — not marked deprecated, not left behind a feature flag, not commented out.

**Why:** Deprecation banners are easy to miss; consumers continue importing the symbol. Deletion is the only signal that survives a `pip install --upgrade`.

### 4. API Removal MUST Sweep Tests In The Same PR

Any PR that removes a public symbol (module, class, function, attribute) MUST delete or port the tests that import it, in the same commit.

```python
# DO — remove API + tests in one commit (D src/pkg/legacy.py + D tests/test_legacy.py)
# DO NOT — leave tests (ModuleNotFoundError at collection blocks every subsequent test)
```

**BLOCKED rationalizations:**

- "The tests will be cleaned up in a follow-up PR"
- "CI doesn't run those tests anyway"
- "The tests are obsolete; they don't need to move"
- "Integration tier is separate scope"
- "`pytest --collect-only` isn't part of CI"

**Why:** One orphan test import at collection blocks the ENTIRE suite (100 tests after it). See playbook § §4.

### 4a. Stub Implementation MUST Sweep Deferral Tests In Same Commit

Mirror of Rule 4. Any PR that implements a previously-deferred stub — replacing `NotImplementedError` / empty-body placeholder with a real impl — MUST delete or rewrite every test that asserts the deferred behavior in the same commit.

```python
# DO — M src/pkg/tracking.py (real impl) + D tests/.../test_track_deferral_names_phase
# DO NOT — implement, leave pytest.raises(NotImplementedError) (flips to "DID NOT RAISE")
```

**BLOCKED rationalizations:**

- "The deferral test was a scaffold; CI will surface it and we'll fix it then"
- "The new test covers it; the old one is obviously obsolete"
- "I'll clean up the scaffold tests in a follow-up"
- "The deferral test is in a different file, out of scope"
- "The Phase N naming means the test self-documents as obsolete"

**Why:** CI-late discovery blocks the release PR's matrix run at the worst moment. Rust equivalent: scaffold `#[should_panic]` on `todo!()` / `unimplemented!()`.

Origin: Session 2026-04-20 — kailash-ml 0.13.0, 5-job failure. See playbook § §4a.

### 4b. Error-Contract Refactor MUST Sweep Paired Tests In Same Commit

Extension of Rule 4a. Any PR that changes the exception TYPE raised by a non-stub public API (e.g. `ValueError` → `IdentifierError`) MUST update every paired `pytest.raises(OldType, ...)` / `assert_matches!(err, OldError::...)` / `.unwrap_err()` assertion in the same commit.

```python
# DO — M audit_store.py (IdentifierError) + M test_audit_store.py (pytest.raises(IdentifierError))
# DO NOT — change error type, leave old pytest.raises (CI red across every matrix job)
```

**BLOCKED rationalizations:**

- "The new error type is a subclass; the old assertion still catches it"
- "CI will surface it; I'll fix the test once it goes red"
- "The paired test is in a different file, out of scope"
- "The old message substring is implementation detail; assertion can stay"
- "Only the security-relevant payloads matter; the type-change test is cosmetic"

**Why:** Identical failure pattern to §4a. This is the error-contract form of `rules/security.md` § "Multi-Site Kwarg Plumbing": grep every caller, patch every hit, same PR.

Origin: Session 2026-04-20 dialect-safety sweep — 7 matrix jobs red. See playbook § §4b.

### 5. Collect-Only Is A Merge Gate

`pytest --collect-only` across every test directory MUST return exit 0 before any PR merges. A collection error is a blocker in the same class as a test failure.

```bash
# DO — gate in CI / pre-commit / /redteam: pytest --collect-only tests/ packages/*/tests/ (exit 0)
# DO NOT — "we only run unit tests in CI, integration is manual"
```

**Why:** Collection failures are invisible in "unit-only CI" setups yet merge-blocking the moment someone runs the full suite locally.

### 5a. Collect-Only Gate Passes Per-Package, Not Combined Root Invocation

Rule 5 MUST NOT be interpreted as mandating a single combined invocation. Monorepos with sub-package test-only deps (`hypothesis`, `respx`, `pytest-benchmark`) CANNOT pass combined invocation from root venv because `python-environment.md` Rule 4 BLOCKS duplicating sub-package test deps in root `[dev]`.

```bash
# DO — per-package, each with its own [dev] extras installed
for pkg in packages/*/tests; do pytest --collect-only -q "$pkg" --continue-on-collection-errors; done
# DO NOT — combined from root (ModuleNotFoundError + ImportPathMismatchError)
```

**BLOCKED rationalizations:**

- "A single invocation is faster for CI"
- "We'll duplicate the test deps in root [dev] just for collection"
- "CI uses a different venv strategy so this doesn't matter locally"
- "Per-package collection is belt-and-suspenders"

**Why:** Per-package granularity matches dep-graph granularity. Rust equivalent: `cargo test -p <crate>` per-crate.

Origin: Session 2026-04-20 /redteam — 3 root causes, per-package succeeded for 9 sub-packages. See playbook § §5a.

### 6. Module-Scope Public Imports Appear In `__all__`

When a symbol is imported at module-scope into a package's `__init__.py` (not behind `_` / not lazy via `__getattr__`), it MUST appear in that module's `__all__` list unless the symbol is private.

```python
# DO — from pkg._device_report import DeviceReport + __all__ = [..., "DeviceReport"]
# DO NOT — imported but missing from __all__ (from pkg import * drops the advertised API)
```

**BLOCKED rationalizations:**

- "The symbol is reachable via `pkg.DeviceReport`, that's enough"
- "Nobody uses `from pkg import *`"
- "`__all__` is a convention, not a contract"
- "We'll clean up `__all__` in a follow-up"
- "The symbol is eagerly imported; the package re-exports it implicitly"

**Why:** `__all__` is the public-API contract: Sphinx, linters, `mypy --strict`, `from pkg import *` all read it as canonical. One-line addition in the same PR.

Origin: kailash-py PR #523/#529 (2026-04-19) — kailash-ml 0.11.0. See playbook § §6.

### 7. Public-API Removal MUST Sweep Consumer Trees With A Verifiable Grep

Any PR that removes a public symbol from a crate/package consumed by other in-repo trees (`bindings/*/`, `examples/`, `packages/*/`, `ffi/`, `clients/`) MUST `rg` the symbol across EVERY such tree AND migrate or delete each consumer in the same PR. The PR body MUST include a passing grep assertion (`! rg <symbol> bindings/ packages/ examples/ crates/`) as the commit-time claim that sweeping is complete.

```bash
# DO — enumerate, grep, migrate, assert-empty in PR body
! rg '<deleted_symbol>' bindings/ packages/ examples/ crates/
# DO NOT — removal lands, bindings left for "follow-up"
```

**BLOCKED rationalizations:**

- "Bindings are cross-crate, out of scope for a nexus PR"
- "Binding CI is separate; they'll re-check on their own cadence"
- "Local `cargo check -p <crate>` was clean; CI will catch the rest"
- "Bindings are generated; they'll auto-update"
- "The symbol was only used in one place, I'm sure I got it"

**Why:** Consumer trees share lockfiles; one unresolved import blocks every sibling release. Grep is O(1); broken `main` costs hours per re-run. The PR-body grep converts "I think I got them all" into a verifiable claim.

Origin: kailash-rs PR #427 (2026-04-19). See playbook § §7–§8.

## MUST NOT

- Land a `db.X` / `app.X` facade without the production call site in the same PR — **Why:** The PR review is the only structural gate before shipping.
- Skip the consumer check on the grounds that "downstream consumers will use it" — **Why:** Downstream use ≠ framework use. The hot path is the security boundary.
- Mark a wired manager as "fully tested" based on Tier 1 unit tests alone — **Why:** Tier 1 mocks the framework's call; it cannot detect "the framework never calls the manager in production."

<!-- /slot:neutral-body -->
