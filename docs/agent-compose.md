# Agent Compose

## Multi-Agent Coordination Above the Single-Agent Spec

A single AgentManifest defines a single agent. Coordinating multiple agents requires a layer above.

This document describes the `agent-compose` model — the natural analog to `docker-compose` — and the orchestration layer above it. Neither is fully specified yet; this is a design sketch, not a complete spec.

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
│  References AgentManifests · Defines        │
│  interfaces · Declares communication   │
└─────────────────────────────────────────┘
                    ▲
┌─────────────────────────────────────────┐
│         AgentManifest (this spec)           │
│      Single-agent definition            │
└─────────────────────────────────────────┘
```

-----

## agent-compose

An `agent-compose` file does three things:

1. References individual AgentManifest files by path or registry location
1. Defines inter-agent communication interfaces (API, message queue, shared storage)
1. Declares which agents can call which other agents

**Sketch of a customer support team:**

```yaml
# agent-compose.yaml — Customer Support Team

agents:
  intake:
    spec: ./agents/intake-classifier.agentmanifest
    exposes:
      - classify-ticket

  resolver:
    spec: ./agents/resolver.agentmanifest
    receives-from: [intake]
    exposes:
      - resolve-ticket
      - draft-response

  escalation:
    spec: ./agents/escalation.agentmanifest
    receives-from: [resolver]
    exposes:
      - escalate-to-human

communication:
  transport: message-queue
  queue: rabbitmq://internal:5672/support

shared:
  knowledge: enterprise-support-kb@acme.internal
```

Each agent retains its own AgentManifest — its own harness, model, guardrails, memory configuration. The compose file defines how they connect, not how they behave internally.

-----

## Key Design Decisions (Open)

### Harness heterogeneity in a team

The intake classifier might run on CrewAI. The resolver might run on LangGraph. The escalation agent might run on OpenClaw. Each runs the harness that makes it best at its job. The compose file coordinates them without requiring harness uniformity.

How inter-harness communication works at the implementation level is an open question. The current assumption is that agents expose interfaces (HTTP endpoints, message queue consumers) that are harness-agnostic at the protocol level.

### The lead agent doesn’t share a harness with subordinates

In a hierarchical team, the coordinator delegates through defined interfaces. Subordinate agents run whatever harness makes them best at their specialist task. The coordinator doesn’t need to know which harness a subordinate uses — only which interface it exposes.

This is analogous to microservices: the orchestrator calls an API. What runs behind the API is an implementation detail.

### Inter-agent protocol

The protocol for “a coding agent completes a task and reports back to its coordinator” needs definition. Candidate approaches:

- **A2A (Agent-to-Agent)** — Google’s emerging agent communication protocol
- **MCP (Model Context Protocol)** — Anthropic’s tool protocol, increasingly used for agent-to-agent calls
- **Custom HTTP** — Simple REST interfaces defined in the compose file

The spec doesn’t commit to a protocol today. The compose model assumes agents communicate through declared interfaces; the protocol is a resolver-level implementation detail.

### Circular delegation

Compose files should be a DAG (directed acyclic graph) — no circular delegation between agents. Enforcement is a compose validator concern.

-----

## Orchestration Layer

For org-level deployments — dozens or hundreds of agents across teams — an orchestration layer sits above agent-compose:

- Agents are the pods
- agent-compose files are the deployments
- The orchestrator handles scheduling, scaling, and health monitoring

This maps conceptually to Kubernetes but is not a spec deliverable today. The seam between agent-compose and an orchestration layer is where fleet management, cross-team versioning, rollouts, and observability infrastructure connect.

-----

## Cross-Harness Observability

When an ops monitor alerts, a coding agent spins up to fix something, and a personal assistant reports the outcome — you need coherent tracing across three different harnesses. That’s an observability problem that sits above the harness layer.

AgentManifest creates a clear seam where it needs to be solved. The `IDENTITY` directive in each AgentManifest gives every agent a verifiable identifier that can anchor traces, logs, and audit records across harness boundaries. The observability infrastructure itself is out of scope for this spec.

-----

## Status

`agent-compose` is a future deliverable. This document is a design sketch. If you have experience building multi-agent coordination layers, this is the right place to open a discussion.

See [`CONTRIBUTING.md`](../CONTRIBUTING.md).
