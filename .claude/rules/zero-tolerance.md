# Zero-Tolerance Rules

## Scope

ALL sessions, ALL agents, ALL code, ALL phases. ABSOLUTE and NON-NEGOTIABLE.

## Rule 1: Pre-Existing Failures, Warnings, and Notices MUST Be Resolved Immediately

If you found it, you own it. Fix it in THIS run — do not report, log, or defer.

**Applies to** — "found it" includes, with equal weight:

- Test failures, build errors, type errors
- Compiler warnings, linter warnings, deprecation notices
- WARN/ERROR entries in the workspace's logs since the previous gate
- Runtime warnings emitted during the session (`DeprecationWarning`, `ResourceWarning`, `RuntimeWarning`)
- Peer-dependency warnings, missing-module warnings, version-resolution warnings

A warning is not "less broken" than an error. It is an error that the framework chose to keep running through. Both are owed.

**Process:**

1. Diagnose root cause
2. Implement the fix
3. Write a regression test
4. Verify with `pytest` (or the project's test command)
5. Include in current or dedicated commit

**BLOCKED responses:**

- "Pre-existing issue, not introduced in this session"
- "Outside the scope of this change"
- "Known issue for future resolution"
- "Reporting this for future attention"
- "Warning, non-fatal — proceeding"
- "Deprecation warning, will address later"
- "Notice only, not blocking"
- ANY acknowledgement, logging, or documentation without an actual fix

**Why:** Deferring broken code creates a ratchet where every session inherits more failures, and the codebase degrades faster than any single session can fix. Warnings are the leading indicator: today's `DeprecationWarning` is next quarter's "it stopped working when we upgraded".

**Mechanism:** The log-triage protocol in `rules/observability.md` MUST Rule 5 provides the concrete commands for scanning test runner output, build tool output, and `*.log` files. If `observability.md` is not loaded (e.g., editing a config file), the agent MUST still scan the most recent test runner and build tool output for WARN+ entries before reporting any gate as complete.

**Exceptions:**

- User explicitly says "skip this issue."
- Upstream third-party deprecation that cannot be resolved by updating or configuring the dependency in this session. Required disposition: pinned version with documented reason OR upstream issue link OR todo with explicit owner. Silent dismissal is still BLOCKED.

## Rule 2: No Stubs, Placeholders, or Deferred Implementation

Production code MUST NOT contain:

- `TODO`, `FIXME`, `HACK`, `STUB`, `XXX` markers
- `raise NotImplementedError`
- `pass # placeholder`, empty function bodies
- `return None # not implemented`

**No simulated/fake data:**

- `simulated_data`, `fake_response`, `dummy_value`
- Hardcoded mock responses pretending to be real API calls
- `return {"status": "ok"}` as placeholder for real logic

**Frontend mock data is a stub:**

- `MOCK_*`, `FAKE_*`, `DUMMY_*`, `SAMPLE_*` constants
- `generate*()` / `mock*()` functions producing synthetic data
- `Math.random()` used for display data

**Why:** Frontend mock data is invisible to Python detection but has the same effect — users see fake data presented as real.

## Rule 3: No Silent Fallbacks or Error Hiding

- `except: pass` (bare except with pass) — BLOCKED
- `catch(e) {}` (empty catch) — BLOCKED
- `except Exception: return None` without logging — BLOCKED

**Why:** Silent error swallowing hides bugs until they cascade into data corruption or production outages with no stack trace to diagnose.

**Acceptable:** `except: pass` in hooks/cleanup where failure is expected.

## Rule 4: No Workarounds for Core SDK Issues

When you encounter a bug in the Kailash SDK, file a GitHub issue on the SDK repository (`terrene-foundation/kailash-py`) with a minimal reproduction. Use a supported alternative pattern if one exists.

**Why:** Workarounds create a parallel implementation that diverges from the SDK, doubling maintenance cost and masking the root bug from being fixed.

**BLOCKED:** Naive re-implementations, post-processing, downgrading.

## Rule 5: Version Consistency on Release

ALL version locations updated atomically:

1. `pyproject.toml` → `version = "X.Y.Z"`
2. `src/{package}/__init__.py` → `__version__ = "X.Y.Z"`

**Why:** Split version states cause `pip install kailash==X.Y.Z` to install a package whose `__version__` reports a different number, breaking version-gated logic.

## Rule 6: Implement Fully

- ALL methods, not just the happy path
- If an endpoint exists, it returns real data
- If a service is referenced, it is functional
- Never leave "will implement later" comments
- If you cannot implement: ask the user what it should do, then do it. If user says "remove it," delete the function.

**Test files excluded:** `test_*`, `*_test.*`, `*.test.*`, `*.spec.*`, `__tests__/`

**Why:** Half-implemented features present working UI with broken backend, causing users to trust outputs that are silently incomplete or wrong.

**Iterative TODOs:** Permitted when actively tracked.
