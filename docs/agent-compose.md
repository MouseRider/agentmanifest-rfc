# Agent Compose

## Multi-Agent Coordination Above the Single-Agent Spec

A single AgentManifest defines a single agent. Coordinating multiple agents requires a layer above.

This document describes the `agent-compose` model — the natural analog to `docker-compose` — and the orchestration layer above it.

-----

## The Three Layers

```
┌─────────────────────────────────────────┐
│         Orchestration Layer             │
│    (agent-kubernetes equivalent)        │
│  Scheduling · Scaling · Health · Fleet  │
└─────────────────────────────────────────┘
                    ▲
┌─────────────────────────────────────────┐
│           agent-compose                 │
│  References AgentManifests · Defines    │
│  interfaces · Declares topology         │
└─────────────────────────────────────────┘
                    ▲
┌─────────────────────────────────────────┐
│         AgentManifest (this spec)       │
│      Single-agent definition            │
└─────────────────────────────────────────┘
```

-----

## agent-compose

An `agent-compose` file does three things:

1. References individual AgentManifest files by path or registry location
1. Defines inter-agent communication interfaces (API, message queue, shared storage)
1. Declares the coordination topology — which agents can call which others, and how decisions are made

Each agent retains its own AgentManifest — its own harness, model, guardrails, memory configuration. The compose file defines how they connect and coordinate, not how they behave internally.

-----

## Coordination Topologies

agent-compose supports three primary coordination topologies. They can be combined within a single compose file.

### Hierarchy

The most common pattern. A lead agent delegates to specialists; each specialist runs whatever harness suits its role. The coordinator doesn’t need to know which harness a subordinate uses — only which interface it exposes.

```yaml
# agent-compose.yaml — Customer Support Team

topology: hierarchy

agents:
  intake:
    manifest: ./agents/intake-classifier.agentmanifest
    role: specialist
    exposes:
      - classify-ticket

  resolver:
    manifest: ./agents/resolver.agentmanifest
    role: specialist
    receives-from: [intake]
    exposes:
      - resolve-ticket
      - draft-response

  escalation:
    manifest: ./agents/escalation.agentmanifest
    role: specialist
    receives-from: [resolver]
    exposes:
      - escalate-to-human

communication:
  transport: message-queue
  queue: rabbitmq://internal:5672/support

shared:
  knowledge: enterprise-support-kb@acme.internal
```

This is analogous to microservices: the orchestrator calls an interface. What harness runs behind that interface is an implementation detail.

### Council

For high-stakes decisions, a council routes a proposal to a set of agents for independent evaluation before any action is taken. No single agent’s judgment is final.

```yaml
topology: council

agents:
  proposer:
    manifest: ./agents/proposer.agentmanifest
  council:
    - manifest: ./agents/compliance-reviewer.agentmanifest
    - manifest: ./agents/context-checker.agentmanifest
    - manifest: ./agents/risk-assessor.agentmanifest

council_config:
  trigger: action-type=financial OR confidence < 0.7
  evaluation: independent   # agents evaluate without seeing each other's output
  quorum: all
  on_rejection: halt-and-alert
  on_timeout: escalate
```

`evaluation: independent` prevents anchoring — each reviewer reaches its own conclusion before any results are shared. Useful for: financial transactions, irreversible actions, external communications from autonomous agents, compliance-sensitive decisions.

### Consensus

A more flexible variant of council. Rather than requiring unanimous approval, agents reach a decision through structured agreement with configurable thresholds.

```yaml
topology: consensus

agents:
  council:
    - manifest: ./agents/reviewer-a.agentmanifest
      weight: 1.0
    - manifest: ./agents/reviewer-b.agentmanifest
      weight: 1.0
    - manifest: ./agents/senior-reviewer.agentmanifest
      weight: 2.0

consensus_config:
  method: weighted-majority   # options: majority, supermajority, unanimity, weighted-majority
  threshold: 0.6
  tie_resolution: escalate
  on_no_consensus: hold-for-human
```

Useful for: moderation decisions on borderline cases, classification tasks where a single agent is inconsistent, any workflow where structured disagreement should surface before acting.

The conditions that trigger a council, the quorum required, and the fallback behavior are all declarable in the spec — not embedded in custom orchestration code.

### Combining Topologies

Topologies can be nested. A hierarchy can include a council at a specific decision point; a consensus step can gate a delegation.

```yaml
agents:
  coordinator:
    manifest: ./coordinator.agentmanifest
    role: lead
  analyst:
    manifest: ./analyst.agentmanifest
    role: specialist
  financial-council:
    topology: council
    members:
      - manifest: ./compliance.agentmanifest
      - manifest: ./risk.agentmanifest
    trigger: action-type=financial
    quorum: all

flow:
  coordinator -> analyst: delegate analysis
  analyst -> financial-council: gate on financial actions
  financial-council -> coordinator: return approval/rejection
```

-----

## Key Design Decisions

### Harness Heterogeneity in a Team

The intake classifier might run on CrewAI. The resolver might run on LangGraph. The escalation agent might run on OpenClaw. Each runs the harness that makes it best at its job. The compose file coordinates them without requiring harness uniformity.

How inter-harness communication works at the implementation level is an open question. The current assumption is that agents expose interfaces (HTTP endpoints, message queue consumers) that are harness-agnostic at the protocol level.

### Inter-Agent Protocol

The protocol for agent-to-agent communication needs definition. Candidate approaches:

- **A2A (Agent-to-Agent)** — Google’s emerging agent communication protocol
- **MCP (Model Context Protocol)** — Anthropic’s tool protocol, increasingly used for agent-to-agent calls
- **Custom HTTP** — Simple REST interfaces defined in the compose file

The spec doesn’t commit to a protocol today. The compose model assumes agents communicate through declared interfaces; the protocol is a resolver-level implementation detail.

### Circular Delegation

Compose files should be a DAG (directed acyclic graph) — no circular delegation between agents. Enforcement is a compose validator concern.

-----

## Identity in Compose Topologies

When agents carry `IDENTITY` directives backed by verifiable credentials, compose topologies gain an additional trust layer. A coordinator can verify that the specialist it’s delegating to is genuinely running the manifest it claims — same spec version, same guardrails in force, same identity.

Council and consensus patterns become stronger with verified identity. An audit trail of a council decision includes not just the outcome but the verified identity of each participating agent, the manifest version each was running, and the guardrails in force at the time. Without identity, a council is a group of agents. With identity, it’s an accountable decision-making structure with a verifiable record.

See [`docs/identity.md`](./identity.md) for the full identity model.

-----

## Cross-Harness Observability

When an ops monitor alerts, a coding agent spins up to fix something, and a personal assistant reports the outcome — you need coherent tracing across three different harnesses. That’s an observability problem that sits above the harness layer.

AgentManifest creates a clear seam where it needs to be solved. The `IDENTITY` directive in each AgentManifest gives every agent a verifiable identifier that can anchor traces, logs, and audit records across harness boundaries. The observability infrastructure itself is out of scope for this spec.

-----

## Orchestration Layer

For org-level deployments — dozens or hundreds of agents across teams — an orchestration layer sits above agent-compose:

- Agents are the pods
- agent-compose files are the deployments
- The orchestrator handles scheduling, scaling, and health monitoring

The topology declarations in agent-compose files become the architectural record of how decisions are made across the system — auditable independent of any specific harness implementation.

-----

## Status

The topology patterns (hierarchy, council, consensus) and the YAML format above are part of the AgentManifest v0.3 spec. Tooling — compose validator, reference resolver — is on the roadmap. If you have experience building multi-agent coordination layers, this is the right place to open a discussion.

See [`CONTRIBUTING.md`](../CONTRIBUTING.md).
