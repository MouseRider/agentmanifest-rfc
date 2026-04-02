# Harness Selection Guide

The `FROM` directive is the most important decision in an AgentManifest file. This guide covers the major harnesses, their strengths, and the agent roles they’re best suited for.

-----

## How to Choose

Ask three questions:

1. **Does the agent need to be proactive, or reactive?**  
   Proactive agents (personal assistants, monitors) need heartbeat cycles and persistent memory. Reactive agents (coding agents, pipelines) run on demand and exit cleanly.
1. **Is the workflow shape known upfront, or emergent?**  
   If you can draw the workflow as a graph before the agent runs, LangGraph fits well. If the agent needs to invent its plan dynamically, you want a more open harness.
1. **Is this one agent or a team?**  
   If it’s a team with defined roles and handoffs, CrewAI’s mental model matches well. If it’s a single agent with sub-agent delegation, OpenClaw handles that natively.

-----

## Harness Profiles

### OpenClaw

**Best for:** Personal assistant, ops monitor, always-on generalist roles

OpenClaw is a general-purpose personal agent platform built for long-running, proactive, multi-domain work:

- Persistent memory and multi-session continuity
- Sub-agent delegation and orchestration
- Heartbeat and proactive behavior (periodic wake cycles, monitoring, initiative-taking)
- Rich tool integration: messaging, calendar, files, APIs
- Flexible channel configuration

The same harness can run a highly autonomous personal assistant (AUTONOMY high, MEMORY persistent) and a strictly-constrained ops monitor (GUARDRAILS read-only-by-default, MEMORY session-only). The harness configuration, not the harness itself, produces the behavioral difference.

**Less suited for:** Purely deterministic pipelines, structured multi-step workflows with a known graph shape.

```
FROM openclaw:latest
```

-----

### Claude Code / Codex CLI

**Best for:** Coding agent, PR reviewer, test runner

Purpose-built for software development with a harness that understands the software lifecycle:

- Deep git integration: read diffs, stage commits, understand branches
- Sandboxed terminal execution with file-system guardrails
- Test-driven workflows: run tests, iterate on failures, require green CI
- PR-native operation: trigger on PR events, report back to the PR

The harness gives the model structured understanding of the codebase and development workflow — not just a terminal. A general-purpose harness on coding tasks produces a capable but contextually naive agent. Claude Code produces an agent that operates the way a developer operates.

**Less suited for:** Personal assistant tasks, ops monitoring, anything outside the software development domain.

```
FROM claude-code:latest
```

-----

### LangGraph

**Best for:** Research pipeline, document processing, structured analysis

Treats everything as a directed graph. Nodes are operations, edges are transitions, cycles enable iteration:

- Strong for workflows where the shape is known upfront
- Excellent for research: gather → analyze → synthesize → verify → output
- Supports conditional branching and explicit loop control
- Human-in-the-loop checkpoints at specific graph nodes
- Native support for `CHECKPOINT` and `HUMAN_REVIEW` directives

LangGraph wants you to define the workflow structure before the agent runs. If you know the shape, that’s a feature. If the agent needs to invent its own plan dynamically, it’s a constraint.

**Less suited for:** Open-ended conversational agents, proactive always-on roles, tasks with highly variable workflow shape.

```
FROM langgraph:latest
```

-----

### CrewAI

**Best for:** Multi-agent teams, document workflows, role-based collaboration

Built for multi-agent collaboration with a mental model of a team:

- Role-based agent definitions with natural social structure
- Structured task handoffs between agents in a crew
- Shared goal and context propagation
- Sequential and parallel task execution models

If your problem looks like “a group of specialists working together toward a shared output,” CrewAI’s harness structure reinforces that naturally. It’s also a solid choice for single agents with a well-defined role inside a larger team composition.

**Less suited for:** Operational agents that need deterministic workflows, always-on monitoring, or roles that don’t map cleanly to team collaboration patterns.

```
FROM crewai:latest
```

-----

### AutoGen

**Best for:** Research refinement, multi-agent debate, iterative synthesis

Built around conversation as the coordination primitive:

- Multi-agent conversation loops: agents debate, refine, critique
- Human-in-the-loop via conversation injection
- Flexible for iterative research and synthesis
- Less opinionated about tool integration

Shines in research and synthesis contexts where you want agents to challenge each other and refine outputs iteratively. Less natural for operational agents that need deterministic workflows.

**Less suited for:** Strict pipelines, ops roles, time-sensitive or latency-sensitive tasks.

```
FROM autogen:latest
```

-----

### Custom Harness

**Best for:** Domain-specific roles requiring non-standard enforcement

Some agent roles don’t fit any off-the-shelf harness well. Trading agents, medical agents, legal research agents with compliance requirements, and high-security ops agents often need custom harnesses with domain-specific guardrail enforcement, audit infrastructure, and integration with proprietary systems.

```
FROM custom-harness:trading-v2
FROM enterprise-harness:secops-v3@vendorco.io
FROM custom-harness:medical-claims-v1
```

A custom harness must implement the AgentManifest resolver interface to accept AgentManifest directives. See the resolver interface section in [`MANIFEST.md`](../MANIFEST.md).

-----

## Quick Reference

|Role              |Recommended `FROM`  |Key directives                                                       |
|------------------|--------------------|---------------------------------------------------------------------|
|Personal assistant|`openclaw:latest`   |HEARTBEAT, MEMORY persistent, CHANNELS, SPENDING                     |
|Ops monitor       |`openclaw:latest`   |GUARDRAILS read-only, MEMORY session-only, ALERT_CHANNEL             |
|Coding agent      |`claude-code:latest`|SANDBOX repo-scoped, GUARDRAILS test-before-commit, TRIGGER pr-opened|
|Research pipeline |`langgraph:latest`  |GRAPH, CHECKPOINT, HUMAN_REVIEW                                      |
|Multi-agent team  |`crewai:latest`     |ROLE, ESCALATION_PATH, SLA                                           |
|Iterative research|`autogen:latest`    |AUTONOMY medium, HUMAN_REVIEW                                        |
|Trading / finance |`custom-harness:*`  |GUARDRAILS hard-limits, SPENDING, SCHEDULE, IDENTITY                 |
|Customer support  |`crewai:latest`     |KNOWLEDGE, CHANNELS, ESCALATION_PATH, SLA                            |

-----

## Model-Aware Prompt Adaptation

The harness resolver is responsible for adapting its prompt scaffolding to the selected `MODEL`. This is a harness concern, not a spec author concern — you shouldn’t have to maintain separate prompt variants for Claude vs. GPT-4o vs. a local model.

The `PROMPT_PROFILE` directive controls this behavior. The default is `auto`, which lets the resolver apply the optimal scaffolding strategy for the declared model:

```
# Harness picks the right prompt structure for claude-opus automatically
MODEL claude-opus
PROMPT_PROFILE auto

# Explicit override for a reasoning-heavy research task
MODEL claude-opus
PROMPT_PROFILE verbose-reasoning

# Minimal scaffolding for a fine-tuned local model that doesn't benefit from CoT framing
MODEL llama-3-custom
PROMPT_PROFILE minimal
```

This matters most when:

- **Swapping models for cost or speed** — downgrading from claude-opus to claude-haiku for an ops monitor shouldn’t require rewriting the system prompt. `PROMPT_PROFILE auto` lets the resolver adapt.
- **Using local or fine-tuned models** — models tuned on specific instruction formats often perform worse with standard scaffolding. `minimal` or an explicit override avoids degrading their output.
- **Multi-model fallback** — if your harness supports model fallback, the resolver needs to adapt prompt scaffolding per model without the spec author defining each case.

### Locale and Language

`LOCALE` controls both output language and the language of injected context. A support agent serving Japanese users should receive context injections phrased in Japanese, not just produce Japanese output:

```
LOCALE ja-JP
PROMPT_PROFILE auto
```

The resolver handles adapting its internal scaffolding to the locale. `SYSTEM_PROMPT_LANG` can override the scaffolding language independently if the harness’s prompt engineering is maintained in a different language than the user-facing output.

-----

## Harness Resolver Status

|Harness       |Resolver status                    |
|--------------|-----------------------------------|
|openclaw      |Planned                            |
|claude-code   |Planned                            |
|langgraph     |Planned                            |
|crewai        |Planned                            |
|autogen       |Planned                            |
|custom-harness|Interface TBD — see CONTRIBUTING.md|

No resolvers are implemented yet. This is a design proposal. Resolver contributions are the most valuable thing this project needs. See [`CONTRIBUTING.md`](../CONTRIBUTING.md).
