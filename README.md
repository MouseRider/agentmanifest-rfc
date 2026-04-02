# AgentManifest

**A declarative spec for AI agents — one spec per agent, the right harness per role.**

> Status: Design Proposal / RFC v0.3  
> Origin: Oleksandr Tsukanov, 2026-03-31  
> Feedback welcome — open an issue or start a discussion.

-----

## The Problem

Most declarative agent specs solve for *portability*: write one definition, run it on any framework. AgentManifest starts from a different question — not *where* an agent runs, but *what it needs to run well*.

A personal assistant and an ops monitor are not the same agent with different prompts. They need different execution environments — different memory models, different autonomy levels, different guardrail enforcement, different lifecycle behaviors. Forcing them into the same harness means one of them is always misconfigured.

The right goal isn’t *write once, run anywhere*. It’s *build each agent for what it actually does*.

**AgentManifest** is a declarative spec for doing exactly that. You describe what an agent needs to be. The tooling selects the right harness for the role and assembles the rest.

-----

## Core Insight: The Harness Is the Product

Two agents can run the same model and behave completely differently based on their harness — the execution environment that controls tool calls, guardrails, state management, autonomy, and lifecycle.

You can write “always ask for approval before deleting files” in a system prompt. The model will follow it until it doesn’t. A deterministic guardrail at the harness level enforces it unconditionally. Those are not the same thing.

AgentManifest makes harness selection and harness configuration a first-class part of the agent definition.

-----

## Quick Look

```
# AgentManifest — Personal Assistant
FROM openclaw:latest

MODEL claude-opus
ROLE personal-assistant

TOOLS browser, email, calendar, file-system, sub-agents
MEMORY persistent, cross-session
PERSONALITY ./soul.md

GUARDRAILS approval-for-external-sends, budget-cap-daily=5.00
AUTONOMY high
HEARTBEAT interval=30m, quiet-hours=23:00-08:00

CHANNELS telegram=in-out, email=in-out, twitter=out
SPENDING daily-cap=50.00, per-transaction-cap=20.00
IDENTITY did:web:agents.example.com:assistant

DEPLOY always-on
RESTART on-failure
```

```
# AgentManifest — Ops Monitor
# Same FROM harness. Completely different configuration.
FROM openclaw:latest

MODEL claude-haiku
ROLE ops-monitor

TOOLS file-system, ssh, docker, http, alerting
MEMORY session-only

GUARDRAILS strict-instructions, no-generative-output, read-only-by-default
AUTONOMY medium
HEARTBEAT interval=5m

ALERT_CHANNEL telegram-ops-thread
ON_ERROR alert-and-retry, max-retries=3

DEPLOY always-on
RESOURCES memory=256m
```

Same base harness. Completely different agent. The spec makes the differences explicit, auditable, and portable — without forcing both into a one-size-fits-all runtime.

-----

## How It Differs From Existing Approaches

|                      |Oracle Agent Spec          |Docker docker-agent            |gitagent                                    |Agentman           |**AgentManifest**                 |
|----------------------|---------------------------|-------------------------------|--------------------------------------------|-------------------|----------------------------------|
|Goal                  |Portability across runtimes|Declarative config, one runtime|Git-native agent definition, export anywhere|Container packaging|Role-appropriate harness per agent|
|Harness selection     |Abstracted away            |Fixed (one runtime)            |Adapter-based export                        |FastAgent/Agno only|First-class directive (`FROM`)    |
|Behavioral enforcement|Framework-dependent        |Prompt-based                   |RULES.md + compliance config                |Not in scope       |Deterministic, harness-compiled   |
|Multi-agent           |Single spec                |Coordinator model              |Inheritance + dependencies                  |Not in scope       |agent-compose layer               |
|Payment / commerce    |Not in scope               |Not in scope                   |Not in scope                                |Not in scope       |First-class directives            |
|Format                |YAML                       |YAML                           |File system structure                       |Dockerfile-like    |Dockerfile-like DSL               |
|Implementation        |Shipped                    |Shipped                        |Shipped (1k stars)                          |Shipped            |Design proposal / RFC             |

The key difference: gitagent, Oracle, and Docker all try to make the runtime invisible — write once, run anywhere. AgentManifest makes the opposite bet: the harness is the primary design decision, and different roles should run different runtimes optimized for their job.

**On gitagent specifically:** It’s excellent and moving fast. If your goal is git-native agent versioning, compliance artifacts, and framework portability, gitagent is worth using today. AgentManifest is solving a different problem — not “how do I define an agent portably” but “how do I declare that this agent should run on LangGraph and that one on Claude Code, and make both configurations auditable and composable.” The two could be complementary: a gitagent repo could reference an AgentManifest to declare its harness requirements.

-----

## Spec Layers

```
┌─────────────────────────────────────────────┐
│           Agent Orchestration               │
│    (agent-compose / kubernetes-equivalent)  │
└─────────────────────────────────────────────┘
                      ▲
┌─────────────────────────────────────────────┐
│              AgentManifest (this)               │
│   Declarative per-agent role definition     │
│   FROM harness + directives + constraints   │
└─────────────────────────────────────────────┘
                      ▲
┌─────────────────────────────────────────────┐
│           Agent Harnesses / Runtimes        │
│  OpenClaw · Claude Code · LangGraph         │
│  CrewAI · AutoGen · Custom                  │
└─────────────────────────────────────────────┘
                      ▲
┌─────────────────────────────────────────────┐
│              LLM Providers                  │
│   Anthropic · OpenAI · Gemini · Local       │
└─────────────────────────────────────────────┘
```

AgentManifest sits between the harness layer and the orchestration layer. It doesn’t replace harnesses — it selects them, configures them, and makes the configuration portable.

-----

## Repo Contents

- [`MANIFEST.md`](./MANIFEST.md) — Full specification, v0.3
- [`examples/`](./examples/) — Ready-to-read AgentManifest files for common agent roles
- [`docs/design-rationale.md`](./docs/design-rationale.md) — Why harness heterogeneity, not portability
- [`docs/harness-guide.md`](./docs/harness-guide.md) — Known harnesses and when to use them
- [`docs/marketplace.md`](./docs/marketplace.md) — The two-sided marketplace: open community components and commercial provider offerings
- [`docs/agent-compose.md`](./docs/agent-compose.md) — Multi-agent coordination above the single-agent spec
- [`CONTRIBUTING.md`](./CONTRIBUTING.md) — How to propose changes and harness resolver contributions

-----

## Status & Roadmap

This is a **design proposal**, not an implementation. Current scope:

- [x] Core spec syntax and directive set (v0.3)
- [x] Single-agent AgentManifest definition
- [x] Reference examples for 6 agent roles
- [x] Marketplace model (open vs. proprietary)
- [ ] agent-compose format specification
- [ ] Harness resolver interface definition
- [ ] Formal grammar / schema (YAML and DSL forms)
- [ ] Validator tooling
- [ ] Reference resolver for at least one harness

-----

## Contributing

This spec benefits from people who’ve run agents in production across different roles and frameworks. If you’ve hit the harness mismatch problem — or think this framing is wrong — open an issue.

See [`CONTRIBUTING.md`](./CONTRIBUTING.md) for how RFC discussion works here.

-----

## License

Apache 2.0. See [`LICENSE`](./LICENSE).
