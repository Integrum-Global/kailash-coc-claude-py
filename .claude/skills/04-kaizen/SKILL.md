---
name: kaizen
description: "Kailash Kaizen — MANDATORY for ALL AI/LLM/agent work. Production AI framework with Delegate, BaseAgent, Signature. Use proactively when work touches LLM calls, prompts, completions, embeddings, chat, agent loops, tool use, RAG, vision/audio AI, or multi-agent. Also triggers for refactors introducing provider abstractions, adapter dispatch layers, LLM service classes, or LLMProviderService-style wrappers — Kaizen's Delegate already provides provider routing. Custom LLM services, provider abstraction layers, adapter dispatch code, openai/anthropic/litellm/langchain wrappers, hand-rolled provider dispatchers BLOCKED."
---

# Kailash Kaizen - AI Agent Framework

Production-ready AI agent framework built on Kailash Core SDK — signature-based programming, multi-agent coordination, and enterprise features.

## Quick Start

### Delegate (Recommended — 2 lines)

```python
from kaizen_agents import Delegate
delegate = Delegate(model=os.environ["LLM_MODEL"])
async for event in delegate.run("Analyze this data"):
    print(event)
```

### GovernedSupervisor (Governed Teams)

```python
from kaizen_agents import GovernedSupervisor
supervisor = GovernedSupervisor(model=os.environ["LLM_MODEL"], budget_usd=10.0)
result = await supervisor.run("Coordinate team analysis")
```

### BaseAgent (Custom Logic Only)

```python
from kaizen.core.base_agent import BaseAgent
from kaizen.signatures import Signature, InputField, OutputField
from dataclasses import dataclass

class SummarizeSignature(Signature):
    text: str = InputField(description="Text to summarize")
    summary: str = OutputField(description="Generated summary")

@dataclass
class SummaryConfig:
    llm_provider: str = "openai"
    model: str = os.environ.get("LLM_MODEL", "")

class SummaryAgent(BaseAgent):
    def __init__(self, config: SummaryConfig):
        super().__init__(config=config, signature=SummarizeSignature())

agent = SummaryAgent(SummaryConfig())
result = agent.run(text="Long text here...")
```

Note: The Agent API is deprecated since v0.5.0. Use Delegate instead.

### Pipeline Patterns (9 Composable)

```python
from kaizen_agents.patterns.pipeline import Pipeline

# Ensemble, Router, Blackboard, Parallel, Sequential,
# Supervisor-Worker, Handoff, Consensus, Debate
pipeline = Pipeline.ensemble(
    agents=[code_expert, data_expert, writing_expert],
    synthesizer=synthesis_agent,
    discovery_mode="a2a", top_k=3
)
result = pipeline.run(task="Analyze codebase", input="repo_path")
```

## Key Features

- **Signature-Based Programming**: Type-safe agent I/O with InputField/OutputField
- **BaseAgent**: Production foundation with error handling, audit, cost tracking
- **Multi-Agent**: Supervisor-worker, A2A protocol, 9 pipeline patterns
- **Multimodal**: Vision, audio, and text processing
- **Autonomy**: Hooks, Checkpoint, Interrupt, Memory, Planning, Meta-Controller
- **Distributed**: AgentRegistry for 100+ agents with O(1) capability discovery
- **Trust (v0.8.0)**: EATP cryptographic trust chains, TrustedAgent, secure messaging
- **Governance (v0.1.0)**: GovernedSupervisor with progressive disclosure, 7 PACT modules
- **L3 Primitives**: Envelope enforcement, scoped context, typed messaging, plan DAG

## Skill Files

### Getting Started

- [kaizen-quickstart-template](kaizen-quickstart-template.md) — Quick start with templates
- [kaizen-baseagent-quick](kaizen-baseagent-quick.md) — BaseAgent fundamentals
- [kaizen-signatures](kaizen-signatures.md) — Signature-based programming
- [kaizen-agent-execution](kaizen-agent-execution.md) — Execution patterns

### Agent Patterns

- [kaizen-agent-patterns](kaizen-agent-patterns.md) — Common design patterns
- [kaizen-chain-of-thought](kaizen-chain-of-thought.md) — Chain of thought reasoning
- [kaizen-react-pattern](kaizen-react-pattern.md) — ReAct pattern
- [kaizen-rag-agent](kaizen-rag-agent.md) — RAG agents
- [kaizen-config-patterns](kaizen-config-patterns.md) — Agent configuration strategies
- [kaizen-multi-agent-setup](kaizen-multi-agent-setup.md) — Multi-agent setup
- [kaizen-supervisor-worker](kaizen-supervisor-worker.md) — Supervisor-worker coordination
- [kaizen-a2a-protocol](kaizen-a2a-protocol.md) — Agent-to-agent communication
- [kaizen-agent-registry](kaizen-agent-registry.md) — Distributed coordination (100+ agents)

### Multimodal

- [kaizen-multimodal-orchestration](kaizen-multimodal-orchestration.md) — Multimodal coordination
- [kaizen-vision-processing](kaizen-vision-processing.md) — Vision/image processing
- [kaizen-audio-processing](kaizen-audio-processing.md) — Audio processing

### Advanced Features

- [kaizen-control-protocol](kaizen-control-protocol.md) — Bidirectional agent-client communication
- [kaizen-tool-calling](kaizen-tool-calling.md) — Autonomous tool execution
- [kaizen-memory-system](kaizen-memory-system.md) — 3-tier memory (Hot/Warm/Cold)
- [kaizen-checkpoint-resume](kaizen-checkpoint-resume.md) — Checkpoint & resume
- [kaizen-streaming](kaizen-streaming.md) — Streaming responses
- [kaizen-cost-tracking](kaizen-cost-tracking.md) — Cost monitoring
- [kaizen-structured-outputs](kaizen-structured-outputs.md) — Multi-provider structured outputs, type system

### Observability

- [kaizen-observability-hooks](kaizen-observability-hooks.md) — Lifecycle event hooks
- [kaizen-observability-tracing](kaizen-observability-tracing.md) — OpenTelemetry tracing
- [kaizen-observability-metrics](kaizen-observability-metrics.md) — Prometheus metrics

### Enterprise & Governance

- [kaizen-trust-eatp](kaizen-trust-eatp.md) — EATP trust infrastructure
- [kaizen-agent-manifest](kaizen-agent-manifest.md) — TOML manifest, deploy, FileRegistry
- [kaizen-composition](kaizen-composition.md) — DAG validation, schema compat, cost estimation
- [kaizen-catalog-server](kaizen-catalog-server.md) — MCP catalog server (11 tools)
- [kaizen-budget-tracking](kaizen-budget-tracking.md) — Budget tracking, posture integration
- [kaizen-agents-governance](kaizen-agents-governance.md) — GovernedSupervisor, 7 PACT modules
- [kaizen-agents-security](kaizen-agents-security.md) — Anti-self-modification, NaN defense

### L3 Autonomy Primitives

- [kaizen-l3-overview](kaizen-l3-overview.md) — 5 subsystems overview, L3Runtime
- [kaizen-l3-envelope](kaizen-l3-envelope.md) — Budget enforcement with gradient zones
- [kaizen-l3-context](kaizen-l3-context.md) — Scoped context with projections
- [kaizen-l3-messaging](kaizen-l3-messaging.md) — Typed inter-agent messaging
- [kaizen-l3-factory](kaizen-l3-factory.md) — Agent spawning, lifecycle tracking
- [kaizen-l3-plan-dag](kaizen-l3-plan-dag.md) — DAG task execution with gradient rules

## Provider Configuration (v2.5.0 — Explicit over Implicit)

Three fields, three purposes:

| Field                    | Purpose                                                    | Example                                      |
| ------------------------ | ---------------------------------------------------------- | -------------------------------------------- |
| `response_format`        | Structured output config                                   | `{"type": "json_schema", "json_schema": {}}` |
| `provider_config`        | Provider-specific operational settings                     | `{"api_version": "2024-10-21"}`              |
| `structured_output_mode` | `"explicit"` (recommended), `"auto"` (deprecated), `"off"` | `"explicit"`                                 |

```python
from kaizen.core.config import BaseAgentConfig
from kaizen.core.structured_output import create_structured_output_config

config = BaseAgentConfig(
    llm_provider="openai",
    model=os.environ["LLM_MODEL"],
    response_format=create_structured_output_config(MySignature(), strict=True),
    structured_output_mode="explicit",
)
```

- **Never** put structured output config in `provider_config` — use `response_format`
- **Never** rely on auto-generated structured output — set `structured_output_mode="explicit"`
- Azure canonical env vars: `AZURE_ENDPOINT`, `AZURE_API_KEY`, `AZURE_API_VERSION`

See [kaizen-config-patterns](kaizen-config-patterns.md) and [kaizen-structured-outputs](kaizen-structured-outputs.md) for details.

## Critical Rules

- Define signatures before implementing agents
- Extend BaseAgent for production (not Agent API — deprecated v0.5.0)
- Default to Delegate for autonomous agents, BaseAgent for custom logic
- LLM does ALL reasoning — no if-else routing (see `rules/agent-reasoning.md`)
- Test with real infrastructure (real infrastructure recommended in Tiers 2-3)
- Use `llm_provider="mock"` explicitly in unit tests
- Use `response_format` for structured output (not `provider_config`)
- Set `structured_output_mode="explicit"` for new agents

## Related Skills

- [01-core-sdk](../01-core-sdk/SKILL.md) — Core workflow patterns
- [02-dataflow](../02-dataflow/SKILL.md) — Database integration
- [03-nexus](../03-nexus/SKILL.md) — Multi-channel deployment
- [05-kailash-mcp](../05-kailash-mcp/SKILL.md) — MCP server integration
