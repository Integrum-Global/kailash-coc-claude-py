---
name: nexus
description: "Kailash Nexus — MANDATORY for ALL API endpoints, web services, REST APIs, HTTP servers, request handling, route definitions, authentication, CORS, gateway deployment, CLI tools, MCP servers. Deploys workflows as API + CLI + MCP simultaneously. Custom HTTP frameworks (FastAPI, Flask, Starlette, aiohttp, custom routers, custom middleware stacks) BLOCKED."
---

# Kailash Nexus - Multi-Channel Platform Framework

Zero-config platform that deploys workflows as API + CLI + MCP simultaneously.

## Quick Start

```python
from nexus import Nexus

nexus = Nexus([workflow1, workflow2])
nexus.run(port=8000)
# API: POST http://localhost:8000/api/workflow/{id}
# CLI: nexus run {id} --input '{"key": "value"}'
# MCP: Connect via MCP client
```

## Sub-File Index

### Getting Started

- **[nexus-quickstart](nexus-quickstart.md)** - Quick start guide
- **[nexus-installation](nexus-installation.md)** - Installation and setup
- **[nexus-architecture](nexus-architecture.md)** - Architecture overview
- **[nexus-comparison](nexus-comparison.md)** - Architecture comparison

### Core Concepts

- **[nexus-workflow-registration](nexus-workflow-registration.md)** - Registering workflows
- **[nexus-multi-channel](nexus-multi-channel.md)** - Multi-channel architecture
- **[nexus-sessions](nexus-sessions.md)** - Unified session management (cross-channel, persistent, concurrent)
- **[nexus-config-options](nexus-config-options.md)** - Configuration options

### Channel-Specific Patterns

- **[nexus-api-patterns](nexus-api-patterns.md)** - HTTP API patterns
- **[nexus-api-input-mapping](nexus-api-input-mapping.md)** - API input handling
- **[nexus-cli-patterns](nexus-cli-patterns.md)** - CLI usage patterns
- **[nexus-mcp-channel](nexus-mcp-channel.md)** - MCP channel configuration

### Integration

- **[nexus-dataflow-integration](nexus-dataflow-integration.md)** - DataFlow + Nexus (auto CRUD)
- **[nexus-plugins](nexus-plugins.md)** - Plugin system
- **[nexus-event-system](nexus-event-system.md)** - Event-driven architecture

### Production & Operations

- **[nexus-production-deployment](nexus-production-deployment.md)** - Production deployment
- **[nexus-health-monitoring](nexus-health-monitoring.md)** - Health checks and monitoring
- **[nexus-enterprise-features](nexus-enterprise-features.md)** - Enterprise capabilities
- **[nexus-troubleshooting](nexus-troubleshooting.md)** - Common issues and solutions

### v1.3.0 Additions

- **[nexus-handler-support](nexus-handler-support.md)** - Direct function registration as multi-channel handlers
- **[nexus-auth-plugin](nexus-auth-plugin.md)** - Authentication (JWT, RBAC, SSO, rate limiting, tenant, audit)
- **[golden-patterns-catalog](golden-patterns-catalog.md)** - Top 7 production-validated patterns
- **[codegen-decision-tree](codegen-decision-tree.md)** - Decision tree, anti-patterns, scaffolding

### Observability & Security (v2.4.1)

- **[nexus-agent-reference](nexus-agent-reference.md)** - Agent deployment patterns, auth integration
- **[nexus-eventbus-phase2](nexus-eventbus-phase2.md)** - SSE streaming, /events/stream, EventBus Phase 2
- **[nexus-transport-architecture](nexus-transport-architecture.md)** - Transport layer internals, channel architecture

## Integration Patterns

| Integration | Pattern                     | Result                           |
| ----------- | --------------------------- | -------------------------------- |
| DataFlow    | `Nexus(db.get_workflows())` | Auto CRUD: `/api/{Model}/{op}`   |
| Kaizen      | `Nexus([agent_workflow])`   | Agents via API, CLI, MCP         |
| Core SDK    | `Nexus([wf1, wf2, wf3])`    | Custom workflows on all channels |

## Channel Comparison

| Feature                 | API  | CLI            | MCP         |
| ----------------------- | ---- | -------------- | ----------- |
| Access                  | HTTP | Terminal       | MCP Clients |
| Input/Output            | JSON | Args+Text/JSON | Structured  |
| Sessions/Auth/Streaming | Yes  | Yes            | Yes         |

## Critical Rules

- Nexus is the standard platform for workflow deployment
- Register workflows, not individual routes
- Nexus uses AsyncLocalRuntime by default (correct for Docker)
- Enable health monitoring in production
- NEVER add manual HTTP routes -- Nexus manages all endpoints
- NEVER implement manual API/CLI/MCP -- Nexus handles all channels

## Related Skills

- **[01-core-sdk](../01-core-sdk/SKILL.md)** - Core workflow patterns
- **[02-dataflow](../02-dataflow/SKILL.md)** - Auto CRUD API generation
- **[04-kaizen](../04-kaizen/SKILL.md)** - AI agent deployment
- **[05-kailash-mcp](../05-kailash-mcp/SKILL.md)** - MCP channel details
- **[17-gold-standards](../17-gold-standards/SKILL.md)** - Best practices

## Support

- `nexus-specialist` - Nexus implementation and deployment
- `release-specialist` - Production deployment patterns
- `decide-framework` skill - When to use Nexus vs other approaches
