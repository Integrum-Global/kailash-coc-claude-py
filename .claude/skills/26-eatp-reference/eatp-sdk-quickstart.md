# EATP Standalone SDK -- Quick Start

The standalone EATP SDK (`pip install eatp`) provides cryptographic trust chains, delegation, and verification for AI agent systems. Apache 2.0, published by the Terrene Foundation.

## Installation

```bash
pip install eatp
# Dependencies: pynacl>=1.5, pydantic>=2.6, jsonschema>=4.21, click>=8.0
```

## Minimal 4-Operation Lifecycle

```python
import asyncio
from eatp import CapabilityRequest, TrustKeyManager, TrustOperations
from eatp.authority import (
    AuthorityPermission,
    AuthorityRegistryProtocol,
    OrganizationalAuthority,
)
from eatp.chain import ActionResult, AuthorityType, CapabilityType
from eatp.crypto import generate_keypair
from eatp.store.memory import InMemoryTrustStore


class SimpleAuthorityRegistry:
    """Minimal registry satisfying AuthorityRegistryProtocol."""

    def __init__(self):
        self._authorities = {}

    async def initialize(self):
        pass

    def register(self, authority: OrganizationalAuthority):
        self._authorities[authority.id] = authority

    async def get_authority(self, authority_id: str, include_inactive: bool = False):
        authority = self._authorities.get(authority_id)
        if authority is None:
            raise KeyError(f"Authority not found: {authority_id}")
        return authority

    async def update_authority(self, authority: OrganizationalAuthority):
        self._authorities[authority.id] = authority


async def main():
    # 1. Setup infrastructure
    store = InMemoryTrustStore()
    await store.initialize()

    key_mgr = TrustKeyManager()
    private_key, public_key = generate_keypair()  # PRIVATE first, public second
    key_mgr.register_key("key-acme", private_key)

    registry = SimpleAuthorityRegistry()
    registry.register(OrganizationalAuthority(
        id="org-acme",
        name="ACME Corp",
        authority_type=AuthorityType.ORGANIZATION,
        public_key=public_key,
        signing_key_id="key-acme",
        permissions=[
            AuthorityPermission.CREATE_AGENTS,
            AuthorityPermission.DELEGATE_TRUST,
        ],
    ))

    ops = TrustOperations(
        authority_registry=registry,
        key_manager=key_mgr,
        trust_store=store,
    )

    # 2. ESTABLISH -- create trust for an agent
    chain = await ops.establish(
        agent_id="agent-analyst",
        authority_id="org-acme",
        capabilities=[
            CapabilityRequest(
                capability="analyze_data",
                capability_type=CapabilityType.ACTION,
            ),
            CapabilityRequest(
                capability="read_reports",
                capability_type=CapabilityType.ACCESS,
            ),
        ],
        constraints=["audit_required"],
    )
    print(f"Established: {len(chain.capabilities)} capabilities")

    # 3. VERIFY -- check if agent can act
    result = await ops.verify(agent_id="agent-analyst", action="analyze_data")
    print(f"Verified: {result.valid} (level={result.level.value})")

    # 4. DELEGATE -- transfer capability to another agent
    delegation = await ops.delegate(
        delegator_id="agent-analyst",
        delegatee_id="agent-junior",
        task_id="task-q4-report",
        capabilities=["analyze_data"],
        additional_constraints=["no_pii_export"],
    )
    print(f"Delegated: {delegation.id}")

    # 5. AUDIT -- record action in immutable trail
    anchor = await ops.audit(
        agent_id="agent-analyst",
        action="analyze_data",
        resource="finance_db.quarterly_revenue",
        result=ActionResult.SUCCESS,
        context_data={"rows_processed": 1200},
    )
    print(f"Audited: {anchor.id}")


asyncio.run(main())
```

## AuthorityRegistryProtocol

The SDK does NOT provide a built-in registry (beyond examples). You must implement `AuthorityRegistryProtocol`:

```python
from eatp.authority import AuthorityRegistryProtocol

# Protocol requires these 3 methods:
class MyRegistry:
    async def initialize(self) -> None: ...
    async def get_authority(self, authority_id: str, include_inactive: bool = False) -> OrganizationalAuthority: ...
    async def update_authority(self, authority: OrganizationalAuthority) -> None: ...
```

All three methods are required. `update_authority()` is used by `CredentialRotationManager` during key rotation.

## Store Selection

| Store                | Use Case                        | Persistence                    | Dependencies   |
| -------------------- | ------------------------------- | ------------------------------ | -------------- |
| `InMemoryTrustStore` | Tests, development              | None (memory)                  | None           |
| `FilesystemStore`    | CLI tools, small deployments    | JSON files (`~/.eatp/chains/`) | None           |
| `PostgresTrustStore` | Production (via Kailash Kaizen) | PostgreSQL                     | kailash-kaizen |

```python
# In-memory (tests)
from eatp.store.memory import InMemoryTrustStore
store = InMemoryTrustStore()

# Filesystem (lightweight persistence)
from eatp.store.filesystem import FilesystemStore
store = FilesystemStore(base_dir="/path/to/chains")

# Both require: await store.initialize()
```

## Enforcement

The SDK is the Policy Decision Point (PDP). It returns verdicts; your application is the Policy Enforcement Point (PEP).

```python
from eatp.enforce.strict import StrictEnforcer, Verdict, EATPBlockedError

enforcer = StrictEnforcer()  # All args optional (on_held, held_callback, flag_threshold)

result = await ops.verify(agent_id="agent-001", action="spend_money")
verdict = enforcer.classify(result)

if verdict == Verdict.BLOCKED:
    # Your app must enforce this
    raise EATPBlockedError(agent_id="agent-001", action="spend_money", reason=result.reason)
elif verdict == Verdict.HELD:
    # Queue for human review
    await queue_for_review(result)
elif verdict == Verdict.FLAGGED:
    # Proceed but log for review
    log_warning(result)
# AUTO_APPROVED -- proceed normally
```

## Trust Postures

```python
from eatp.postures import TrustPosture, PostureStateMachine

# Create state machine
sm = PostureStateMachine(default_posture=TrustPosture.SUPERVISED)

# Posture hierarchy: BLOCKED < HUMAN_DECIDES < SUPERVISED < ASSISTED < FULL_AUTONOMY
print(TrustPosture.SUPERVISED < TrustPosture.FULL_AUTONOMY)  # True
```

## Secure Messaging

```python
from eatp.messaging.channel import SecureChannel
from eatp.messaging.signer import MessageSigner
from eatp.messaging.verifier import MessageVerifier
from eatp.messaging.replay_protection import InMemoryReplayProtection

channel = SecureChannel(
    sender_id="agent-a",
    receiver_id="agent-b",
    signer=MessageSigner(private_key=priv_key),
    verifier=MessageVerifier(public_keys={"agent-a": pub_key}),
    replay_protection=InMemoryReplayProtection(max_nonces=100_000),
)
```

## Key Rotation

```python
from eatp.rotation import CredentialRotationManager

rotation_mgr = CredentialRotationManager(
    key_manager=key_mgr,
    trust_store=store,
    authority_registry=registry,  # Must implement update_authority()
)

result = await rotation_mgr.rotate("agent-001")
```

## CLI

```bash
# Quickstart -- interactive trust setup
eatp quickstart

# Other commands via the CLI module
eatp --help
```

## For More Detail

- **API Reference**: `.claude/skills/26-eatp-reference/eatp-sdk-api-reference.md`
- **Patterns & Gotchas**: `.claude/skills/26-eatp-reference/eatp-sdk-patterns.md`
