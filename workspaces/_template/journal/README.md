# Project Journal

The journal is the primary knowledge trail for this project. Every CO command that produces insights, decisions, or discoveries creates journal entries here. These entries prevent amnesia across sessions, document rationale for future reference, and build a compounding knowledge base that each session extends rather than rediscovers.

## Entry Types

| Type | Purpose | When Created |
|------|---------|-------------|
| **DECISION** | Record a choice with rationale, alternatives considered, and consequences | When making architectural, design, strategic, or scope decisions |
| **DISCOVERY** | Capture something learned — a concept, pattern, insight, or finding | When research, analysis, or exploration reveals new understanding |
| **TRADE-OFF** | Document a trade-off evaluation — what was gained and what was sacrificed | When balancing competing concerns (quality vs. speed, accuracy vs. interpretability, etc.) |
| **RISK** | Record an identified risk, vulnerability, or concern | When stress-testing, reviewing, or validating reveals potential problems |
| **CONNECTION** | Note a relationship between concepts, components, or findings | When cross-referencing reveals links that matter |
| **GAP** | Flag something missing that needs attention | When analysis reveals missing data, untested assumptions, or unresolved questions |

## Naming Convention

```
NNNN-TYPE-topic.md
```

- `NNNN` — four-digit sequential number, zero-padded (0001, 0002, ...)
- `TYPE` — one of the six types above
- `topic` — kebab-case brief description

Examples:

- `0001-DECISION-chose-event-driven.md`
- `0002-DISCOVERY-connection-pool-bottleneck.md`
- `0003-TRADE-OFF-latency-vs-consistency.md`
- `0004-RISK-single-point-of-failure.md`
- `0005-CONNECTION-auth-and-rate-limiting.md`
- `0006-GAP-missing-error-handling.md`

Always check the highest existing number before creating a new entry.

## Frontmatter Format

Every journal entry must include YAML frontmatter:

```yaml
---
type: DECISION | DISCOVERY | TRADE-OFF | RISK | CONNECTION | GAP
date: YYYY-MM-DD
project: [project name]
topic: [brief topic description]
phase: [which COC phase: analyze | todos | implement | redteam | codify | deploy]
tags: [list of relevant tags]
---
```

## Body Content

After frontmatter, write the insight in full. Each entry type has a recommended structure:

**DECISION** — Decision, Alternatives Considered, Rationale, Consequences
**DISCOVERY** — What Was Discovered, Why It Matters, Follow-Up
**TRADE-OFF** — Trade-Off, What Was Gained, What Was Sacrificed, Acceptable Because
**RISK** — Risk Identified, Likelihood and Impact, Mitigation, Follow-Up
**CONNECTION** — Connection, Components Linked, Why This Matters
**GAP** — What Is Missing, Why It Matters, How to Resolve

## Examples

### DECISION Entry

```yaml
---
type: DECISION
date: 2024-03-24
project: API Gateway
topic: Chose event-driven architecture over request-response
phase: plan
tags: [architecture, messaging, scalability]
---

## Decision
Adopted event-driven architecture for inter-service communication.

## Alternatives Considered
1. **Request-response (REST)** — Simpler but creates tight coupling between services
2. **GraphQL federation** — Good for client flexibility but adds complexity to the gateway layer

## Rationale
Event-driven decouples services temporally. Services can be deployed independently and failures are isolated. The trade-off is eventual consistency, which is acceptable for this use case.

## Consequences
- Need a message broker (chose NATS for simplicity)
- Must design for idempotency in all consumers
- Monitoring becomes more important (distributed tracing required)
```

### DISCOVERY Entry

```yaml
---
type: DISCOVERY
date: 2024-03-24
project: API Gateway
topic: Connection pooling bottleneck under load
phase: review
tags: [performance, database, connection-pool]
---

## What Was Discovered
Under sustained load (>500 req/s), the database connection pool exhausts before the rate limiter kicks in. The pool size of 20 is insufficient for the burst pattern.

## Why It Matters
This means the system fails silently — connections queue and timeout rather than returning errors. Users see slow responses, not error messages.

## Follow-Up
- Increase pool size to 50 and re-test
- Add connection pool monitoring to the dashboard
- Consider implementing connection pool circuit breaker
```

## Relationship to COC Phases

Journal entries are created throughout the CO lifecycle:

| Phase | Typical Entry Types |
|-------|-------------------|
| **Analyze** | DISCOVERY, GAP, CONNECTION |
| **Todos** | DECISION, TRADE-OFF, RISK |
| **Implement** | DECISION, DISCOVERY, RISK |
| **Redteam** | RISK, GAP, CONNECTION |
| **Codify** | DECISION, TRADE-OFF |
| **Deploy** | DECISION, RISK |

## Purpose

The journal serves multiple functions:

1. **Anti-amnesia**: Prevents re-discovering the same insights across sessions
2. **Decision documentation**: Provides rationale for choices that may be questioned later
3. **Knowledge compounding**: Each session builds on prior insights rather than starting fresh
4. **Audit trail**: Documents the reasoning process, not just the outcome
