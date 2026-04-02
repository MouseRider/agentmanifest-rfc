# Design Rationale

## Why harness heterogeneity, not portability

This document explains the design decisions behind AgentManifest and how it differs from other declarative agent specifications. It’s the right place to start if you’re evaluating whether this approach makes sense or want to understand why certain choices were made.

-----

## The Portability Trap

The dominant approach to declarative agent specs in 2025–2026 is portability: define an agent once, run it on any framework. Oracle’s Open Agent Specification, Docker’s docker-agent, and similar projects are built around this goal. It’s a natural instinct — it’s what ONNX did for ML models, what OCI did for containers.

The problem: agent harnesses are not interchangeable runtimes.

A container image runs the same software on any OCI-compatible runtime. An agent’s *behavior* is inseparable from its execution environment. The harness determines:

- Whether memory persists across sessions or evaporates
- Whether guardrails are enforced by code or by prompting the model
- Whether the agent can take initiative or only responds
- Whether tool calls are retried, parallelized, or run sequentially
- Whether the agent has a concept of a “workflow” at all

Abstract these away behind a portability layer and you don’t get a portable agent. You get the lowest common denominator of every harness — which is to say, you get a worse agent everywhere.

The right question isn’t “which runtime can this agent run on?” It’s “which runtime makes this agent best at its job?”

-----

## The Two-Layer Model

Agent behavior has two distinct layers. Both matter. Most specs focus on one.

**Layer 1: Model traits**

Reasoning depth, speed, cost, context window, instruction following. This layer is well-understood — there are benchmarks, leaderboards, and mature model selection tooling.

**Layer 2: Harness traits**

Tool execution patterns (sequential, parallel, retry logic), deterministic guardrail enforcement, autonomy controls, state management, error recovery, sandboxing, behavioral profile. This layer is less understood, less documented, and has far more impact on production reliability than model selection.

Two teams using the same model can see dramatically different task completion rates based entirely on harness design. The harness is the product. The model is a replaceable component inside it.

AgentManifest addresses both layers. `MODEL` selects the model. `FROM` selects the harness. The remaining directives configure both. Neither is an afterthought.

-----

## Prompts vs. Harness Enforcement

This distinction deserves its own section because it’s the most important one.

Consider a trading agent. You don’t want it to exceed a per-transaction cap of $10,000. You have two options:

**Option A — Prompt-based enforcement:**

```
You must never execute a trade worth more than $10,000. Always check the position 
size before placing an order. If a trade would exceed this limit, do not proceed.
```

The model will follow this instruction the vast majority of the time. Until it reasons that a particular situation is exceptional. Until a context window gets crowded and the instruction loses prominence. Until a slightly unusual phrasing leads the model to classify the action differently.

**Option B — Harness enforcement:**

```
SPENDING per-transaction-cap=10000.00
```

The harness resolver compiles this into a pre-execution check. The model’s output is evaluated against the limit before any trade is placed. There is no “reasoning around” it. The limit is enforced unconditionally by code.

The same principle applies to prompt scaffolding and locale adaptation. If you switch the model in a running agent — from claude-opus to claude-haiku for cost reasons, or to a local model for privacy — the prompt structure that worked for one model may actively degrade performance on another. The `PROMPT_PROFILE auto` directive delegates this to the harness resolver: it knows which scaffolding strategy suits which model, and adapts without requiring the spec author to maintain model-specific prompt variants. Similarly, `LOCALE` lets the resolver inject context in the user’s language without the spec encoding language-specific prompt logic directly.

The same principle applies to file deletion approval, force-push prevention, read-only mode, output format restrictions, and any other constraint where “almost always” is not good enough.

AgentManifest’s `GUARDRAILS`, `SPENDING`, `SANDBOX`, and `CHANNELS` directives are all intended to be implemented as harness-level enforcement, not model-level instruction. The spec makes this distinction explicit: if a directive is listed under Capabilities & Constraints, it should be enforced by the resolver, not by the prompt.

-----

## Why the Docker Metaphor Works

Docker didn’t invent containers. It made containers usable by giving developers a declarative syntax (`FROM`, `RUN`, `COPY`, `EXPOSE`) that expressed intent clearly, composed predictably, and produced a reproducible artifact.

The Dockerfile metaphor translates well to agents:

- `FROM` selects a base image (harness) optimized for a use case
- Directives customize the base without reimplementing it
- The result is a portable artifact (the AgentManifest file) that produces consistent behavior when applied to the right resolver
- Base images (harnesses) can be versioned, pinned, and updated independently of the spec

The key difference from Docker: there is no single runtime that executes all agent types. A LangGraph agent and an OpenClaw agent are not the same thing in different packaging. They’re built differently, from the harness up. The spec acknowledges this rather than hiding it.

`agent-compose` extends the metaphor to multi-agent coordination, analogous to docker-compose. An orchestration layer above that maps to Kubernetes. The layer structure is intentional and maps to real operational concerns.

-----

## What This Is Not

**Not a portability layer.** If you want to write one agent definition and run it on LangGraph, CrewAI, and AutoGen interchangeably, AgentManifest is not the right tool. Oracle’s Agent Spec targets that use case. AgentManifest makes the opposite bet: the harness matters, so choose it deliberately.

**Not a replacement for harnesses.** OpenClaw, LangGraph, CrewAI, Claude Code — these are excellent harnesses for their respective domains. AgentManifest sits above them. It’s the spec that says “this agent should run on LangGraph” and lets you build that alongside “that agent should run on Claude Code” with the same workflow and tooling.

**Not an implementation.** This is a design proposal. No resolver implementations exist yet. The spec is the artifact; tooling comes later. This is intentional — getting the spec right matters more than shipping partial tooling early.

**Not complete.** The open questions section of [`MANIFEST.md`](../MANIFEST.md) is honest. Inter-agent protocol, version compatibility, cross-harness observability, block registry governance, and identity federation are all unsolved. This is v0.3 of a design proposal, not a finished standard.

-----

## Relationship to Existing Work

**Agentman** (github.com/yeahdongcn/agentman): Packages agents built on FastAgent or Agno into deployable containers using Docker-like syntax. AgentManifest builds on this foundation: same metaphor, extended to cover multi-harness composition and the full behavioral configuration of any agent.

**Oracle Open Agent Specification** (arxiv 2510.04173): Framework-agnostic portability spec. Explicitly solving the opposite problem from AgentManifest. Both are valid; they’re different bets on what developers need.

**Docker docker-agent**: Declarative YAML config for agents on a single runtime. Proves the appetite for declarative agent configuration. Doesn’t address harness selection.

**AgentSpec (ICSE ’26)**: Unrelated despite the name — a runtime enforcement framework for LLM agent safety. The `GUARDRAILS` directive in AgentManifest is inspired by similar motivations but operates at the spec layer rather than the enforcement layer.

**gitagent** (github.com/open-gitagent/gitagent): Git-native agent definition — your repository structure becomes your agent, with adapters to export to any framework. Excellent for versioning, compliance artifacts, and portability. Explicitly portability-first: the adapters let you run the same definition on CrewAI, Claude Code, OpenAI, and others. AgentManifest’s thesis is the opposite: the harness matters, so choose it deliberately rather than abstracting it away. The two are potentially complementary — a gitagent repo could embed an AgentManifest to declare harness requirements while gitagent handles versioning and git-native tooling.
