# AgentManifest Specification

**Version:** 0.3  
**Status:** Design Proposal / RFC  
**Origin:** Oleksandr Tsukanov, 2026-03-31  
**Updated:** 2026-04-02

-----

## Overview

AgentManifest is a declarative specification for defining AI agents. A single AgentManifest file defines one agent: its base harness, model, role, capabilities, constraints, communications, payment permissions, and runtime lifecycle — in a portable, auditable, versionable format.

The syntax is intentionally Docker-like. `FROM` declares the base harness. Directives customize it. The build system assembles the rest.

For multi-agent coordination, see [`docs/agent-compose.md`](./docs/agent-compose.md).

-----

## Design Principles

### 1. Harness heterogeneity is a feature

Different agent roles need different execution environments. A coding agent and a personal assistant are not the same agent with different prompts. AgentManifest makes harness selection explicit rather than abstracting it away.

### 2. Deterministic enforcement over prompt-based enforcement

Guardrails, spending limits, channel permissions, and sandbox scope are enforced by the harness resolver at the execution layer — not by asking the model to be careful. Prompt-based rules are advisory. Harness-level rules are unconditional.

### 3. The dual-value pattern

Every directive supports two forms:

**Provider shorthand** — brief descriptor resolved by the harness:

```
MEMORY persistent, cross-session
```

**Full spec block** — explicit deployment definition:

```
MEMORY
  TYPE redis
  HOST memory.internal:6379
  TTL 30d
  HOOKS on-session-start=load, on-session-end=flush
```

Shorthand for convenience and prototyping. Full spec for production control. Both are valid at any directive.

### 4. The spec is the contract

An AgentManifest file should be readable by anyone — engineers, operators, auditors, and (eventually) other agents — and fully describe what the agent is authorized to do, how it behaves, and what it can access.

-----

## File Format

AgentManifest files use a custom DSL inspired by Dockerfile syntax. The recommended file extension is `.agentmanifest`.

A minimal valid spec requires only `FROM`, `MODEL`, and `ROLE`. All other directives are optional and default to the harness resolver’s baseline.

Comments begin with `#`. Blank lines are ignored. Directive names are uppercase. Values are case-insensitive unless referencing file paths or URIs.

-----

## Directive Reference

Directives are grouped into six categories.

-----

### 1. Agent Identity

#### `FROM`

Base harness identifier and optional version tag. This is the primary design decision — it determines which execution environment the agent runs in and which resolver handles the remaining directives.

```
FROM openclaw:latest
FROM claude-code:1.2
FROM langgraph:stable
FROM crewai:latest
FROM autogen:latest
FROM custom-harness:trading-v2
FROM enterprise-harness:secops-v3@vendorco.io
```

Version pinning is strongly recommended for production agents.

#### `MODEL`

Model alias or specific model identifier. Resolved by the build tooling against available providers. The harness may restrict which models are valid for a given `FROM`.

```
MODEL claude-opus
MODEL claude-sonnet
MODEL claude-haiku
MODEL gpt-4o
MODEL gpt-4o-mini
MODEL fast-reasoning
```

#### `ROLE`

Semantic role label. Informs harness defaults and behavioral profile. Not a prompt — a structural declaration used by the resolver to set baseline configuration.

```
ROLE personal-assistant
ROLE coding-agent
ROLE ops-monitor
ROLE trading-agent
ROLE research-pipeline
ROLE customer-support
```

#### `PERSONALITY`

Path to a persona or soul file that shapes the agent’s behavioral style and voice. Separate from `ROLE` — role is structural, personality is stylistic.

```
PERSONALITY ./soul.md
PERSONALITY ./personas/formal-assistant.md
```

#### `IDENTITY`

The agent’s verifiable identity on the wire. Used for audit logs, inter-agent authentication, access control, and payment attribution. Not a display name — a cryptographic or protocol-level identifier.

Supports W3C Decentralized Identifiers (DIDs), OAuth 2.0 client credentials, and API key references.

```
IDENTITY did:web:agents.example.com:assistant
IDENTITY oauth-client=agent-assistant-prod
IDENTITY apikey-ref=vault://agents/trading-prod
```

**Verifiable capabilities and SLAs**

Once an agent has a cryptographic identity, that identity can be bound to its declared capabilities — TOOLS, GUARDRAILS, AUTONOMY level, SPENDING limits. This makes capability claims checkable rather than advisory. A downstream agent or orchestrator can verify not just who an agent is, but what it is certified to do and within what constraints it operates.

This opens the door to meaningful behavioural SLAs — not just uptime and response time, but quality and compliance guarantees: “this agent is certified to operate within these guardrails under this identity.” As agent-to-agent trust infrastructure matures, identity-bound capability verification becomes the foundation for trustworthy multi-agent systems.

**Identity immutability**

An IDENTITY directive binds a cryptographic identity to a specific agent definition. That binding should be treated as immutable — if the manifest changes in any meaningful way (different TOOLS, different GUARDRAILS, different MODEL), the identity should change too. Reusing the same identity across materially different definitions breaks the trust anchor: you can no longer verify what an agent is by checking its identity, because what it is has changed underneath.

The practical implication: treat AgentManifest versions the way you treat signed software releases. A new version with different capabilities is a new identity. The old identity remains valid for the old definition. Revocation of an identity should not require taking down the harness — it should invalidate the binding between that identity and its capability claims.

-----

### 2. Capabilities & Constraints

#### `TOOLS`

Comma-separated list of tools the agent is permitted to use. The harness resolver maps these labels to actual implementations. Tools not listed are implicitly off-limits.

```
TOOLS git, terminal, file-system, browser, test-runner
TOOLS browser, email, calendar, file-system, sub-agents, web-search
TOOLS file-system, ssh, docker, http, alerting
TOOLS market-data-api, broker-api, position-tracker, risk-calculator
```

Full spec form (for individual tool configuration):

```
TOOL browser
  ALLOW_DOMAINS *.internal, docs.company.com
  DENY_DOMAINS *.social, *.gaming
  TIMEOUT 30s
```

#### `GUARDRAILS`

Deterministic enforcement rules compiled by the harness resolver. Not prompt instructions — harness-level constraints that execute unconditionally regardless of model output.

Format: `rule-name` or `rule-name=value`.

```
GUARDRAILS test-before-commit, no-force-push, require-green-ci
GUARDRAILS strict-instructions, no-generative-output, read-only-by-default
GUARDRAILS hard-stop-loss=2pct, max-position-size=5pct, no-margin, no-options
GUARDRAILS tone-policy=./support-guidelines.md, no-refunds-without-approval
```

Harness resolvers define which guardrail labels they support. Custom guardrail definitions can reference policy files.

#### `AUTONOMY`

Autonomy level controlling approval gates and initiative-taking behavior.

|Value                   |Behavior                                                         |
|------------------------|-----------------------------------------------------------------|
|`high`                  |Acts without asking unless hitting a guardrail                   |
|`high-within-guardrails`|High autonomy within declared constraints; hard stops at limits  |
|`medium`                |Acts on routine tasks; asks on ambiguous or high-impact decisions|
|`supervised`            |Asks for approval at defined checkpoints                         |
|`low`                   |Asks before most actions                                         |

```
AUTONOMY high
AUTONOMY medium
AUTONOMY supervised
AUTONOMY high-within-guardrails
```

#### `MEMORY`

Memory configuration for state persistence between turns and sessions.

```
MEMORY persistent, cross-session
MEMORY session-only
MEMORY append-only-audit-log
MEMORY none
```

Full spec form:

```
MEMORY
  TYPE redis
  HOST memory.internal:6379
  TTL 30d
  SCOPE cross-session
  HOOKS on-session-start=load, on-session-end=flush
```

#### `SANDBOX`

Scope restricting the agent’s execution environment access.

```
SANDBOX repo-scoped
SANDBOX read-only
SANDBOX network-isolated
SANDBOX none
```

#### `KNOWLEDGE`

Knowledge base reference — open registry path or proprietary provider. See [`docs/marketplace.md`](./docs/marketplace.md) for the full open/proprietary model.

```
KNOWLEDGE github.com/legal-commons/wa-rcw-parsed
KNOWLEDGE wa-realestate-pro@legalmind.io
KNOWLEDGE enterprise-support-kb@acme.internal
```

#### `CONTEXT`

Context source configuration.

```
CONTEXT tsvc-enabled
CONTEXT repo-root=./
CONTEXT window=128k
```

#### `LOCALE`

Language and regional locale for the agent’s output and context injection. Affects more than just output language — the harness resolver uses locale to adapt prompt phrasing, date/number formatting, and culturally appropriate response patterns for the target audience.

```
LOCALE en-US
LOCALE ja-JP
LOCALE pt-BR
LOCALE fr-FR
```

Full spec form for multi-locale agents (e.g. a support agent serving multiple regions):

```
LOCALE
  DEFAULT en-US
  DETECT from-user-input
  SUPPORTED en-US, ja-JP, fr-FR, pt-BR
  FALLBACK en-US
```

When `DETECT from-user-input` is set, the harness resolver infers the user’s locale from their message and switches context injection language accordingly — without changing the agent’s operational configuration.

#### `PROMPT_PROFILE`

Declares how the harness resolver should adapt its prompt scaffolding and context injection to the selected model. Different models respond differently to the same instruction phrasing — Claude, GPT-4o, Gemini, and local models each have distinct optimal prompting conventions, chain-of-thought patterns, and few-shot structures.

`PROMPT_PROFILE` externalizes this adaptation to the harness rather than forcing the spec author to maintain model-specific prompt variants.

```
PROMPT_PROFILE auto
PROMPT_PROFILE conservative
PROMPT_PROFILE verbose-reasoning
PROMPT_PROFILE minimal
```

|Value              |Behavior                                                                                           |
|-------------------|---------------------------------------------------------------------------------------------------|
|`auto`             |Harness resolver selects the optimal profile for the declared `MODEL` — default                    |
|`conservative`     |Terse, directive instructions; minimal scaffolding; suits strong instruction-following models      |
|`verbose-reasoning`|Explicit chain-of-thought scaffolding, step-by-step framing; suits reasoning-heavy tasks           |
|`minimal`          |Bare prompt, no injected scaffolding; for models or use cases where harness framing degrades output|

Full spec form for explicit per-model overrides:

```
PROMPT_PROFILE
  DEFAULT auto
  OVERRIDE claude-opus: verbose-reasoning
  OVERRIDE gpt-4o-mini: conservative
  OVERRIDE llama-3: minimal
  SYSTEM_PROMPT_LANG en
  FEW_SHOT_EXAMPLES ./prompts/examples.md
```

The `SYSTEM_PROMPT_LANG` sub-directive controls the language used for injected system prompt scaffolding independent of `LOCALE` — relevant when the agent operates in a non-English locale but the harness’s internal prompt engineering is maintained in English. The resolver handles translation of scaffolding to match `LOCALE` unless overridden here.

This directive interacts with `LOCALE`: when both are set, the resolver adapts both the prompt *structure* (PROMPT_PROFILE) and the prompt *language* (LOCALE) simultaneously, injecting model-appropriate scaffolding in the user’s language.

-----

### 3. Communications & Channels

#### `CHANNELS`

Declares which communication channels the agent is authorized to use, and in which direction.

|Mode    |Meaning                             |
|--------|------------------------------------|
|`in`    |Receives/monitors only — no outbound|
|`out`   |Posts/broadcasts only — no inbound  |
|`in-out`|Full bidirectional                  |

Shorthand form:

```
CHANNELS telegram=in-out, email=in-out, twitter=out, slack=in-out
```

Full spec form:

```
CHANNEL telegram
  MODE in-out
  ACCOUNT @my-agent-bot
  ALLOW @alex, @trusted-group

CHANNEL twitter
  MODE out
  PURPOSE publish-updates

CHANNEL email
  MODE in
  FILTER from=*@company.com
```

Supported channel types: `telegram`, `slack`, `email`, `sms`, `phone`, `twitter`, `linkedin`, `discord`. Channels not declared are implicitly off-limits.

-----

### 4. Payment & Commerce

Payment directives enforce financial boundaries at the harness level. An agent cannot exceed its spending caps by reasoning around them — the harness enforces limits unconditionally.

#### `PAYMENT`

Payment systems and rails the agent is authorized to initiate payments through.

```
PAYMENT stripe, corporate-card
PAYMENT broker-api
```

Full spec form:

```
PAYMENT
  RAIL stripe
  ACCOUNT acct-agent-prod
  CURRENCY USD
  AUTH require-mfa-above=500.00
```

#### `SPENDING`

Hard spending limits. Values are enforced by the harness resolver.

```
SPENDING daily-cap=50.00, per-transaction-cap=20.00
SPENDING daily-cap=25000.00, per-transaction-cap=10000.00
```

Full spec form:

```
SPENDING
  DAILY_CAP 50.00
  PER_TRANSACTION_CAP 20.00
  APPROVAL_THRESHOLD 100.00
```

#### `RECEIVE`

Payment types the agent is authorized to accept.

```
RECEIVE invoices, reimbursements
RECEIVE settlement-proceeds
RECEIVE invoices, subscriptions
```

#### `WALLET`

Crypto wallet configuration.

```
WALLET crypto=0x...abc
WALLET crypto=disabled
```

#### `HUMAN_APPROVAL`

Conditions requiring human approval before action — financial or otherwise.

```
HUMAN_APPROVAL above-threshold=10000
HUMAN_APPROVAL always
HUMAN_APPROVAL never
```

-----

### 5. Runtime & Lifecycle

#### `DEPLOY`

Deployment mode.

|Value                      |Behavior                                     |
|---------------------------|---------------------------------------------|
|`always-on`                |Persistent process; restarts on failure      |
|`on-demand`                |Spawned by trigger; exits when task completes|
|`always-on-during-schedule`|Persistent within SCHEDULE windows           |

#### `TRIGGER`

Events that spin up an on-demand agent.

```
TRIGGER pr-opened, pr-review-requested
TRIGGER api-call, schedule
TRIGGER webhook=https://internal.example.com/trigger
```

#### `HEARTBEAT`

Periodic proactive wake cycle. Lets always-on agents take initiative on a schedule rather than only reacting to inputs.

```
HEARTBEAT interval=30m
HEARTBEAT interval=5m, quiet-hours=23:00-08:00
```

#### `SCHEDULE`

Operational schedule constraints. Agent is active only within declared windows.

```
SCHEDULE market-hours-only, timezone=America/New_York
SCHEDULE weekdays=09:00-18:00, timezone=Europe/London
```

#### `RESTART`

Restart policy on failure.

```
RESTART on-failure
RESTART always
RESTART never
```

#### `RESOURCES`

Resource constraints passed to the harness runtime.

```
RESOURCES memory=256m
RESOURCES memory=1g, timeout=30m
RESOURCES memory=512m, latency-target=low
```

#### `ON_ERROR`

Error handling behavior when a tool call or execution step fails.

```
ON_ERROR ask-human
ON_ERROR alert-and-retry, max-retries=3
ON_ERROR fail-fast
```

-----

### 6. Observability & Escalation

#### `ALERT_CHANNEL`

Channel for operational alerts separate from the agent’s primary communication channels.

```
ALERT_CHANNEL telegram-ops-thread
ALERT_CHANNEL slack=#alerts-prod
```

#### `SLA`

Service level expectations communicated to the harness for monitoring and alerting.

```
SLA response-time=2m
SLA uptime=99.9pct
```

#### `ESCALATION_PATH`

Where to route issues the agent cannot resolve autonomously.

```
ESCALATION_PATH human-agent
ESCALATION_PATH pagerduty=P0-oncall
```

#### `HUMAN_REVIEW`

Workflow checkpoints requiring human review before the agent proceeds. Used in pipeline-style harnesses.

```
HUMAN_REVIEW at=verify
HUMAN_REVIEW at=output, timeout=24h
```

#### `CHECKPOINT`

Workflow checkpoints for state save and resumption. Used in pipeline-style harnesses.

```
CHECKPOINT after=gather, after=synthesize
```

#### `GRAPH`

Explicit workflow graph for pipeline-style harnesses. Defines nodes and transitions.

```
GRAPH {
  gather → analyze → synthesize → verify → output
}
```

-----

## Harness Resolver Contract

A harness resolver is the adapter that takes an AgentManifest file and instantiates a running agent on a specific harness. Each `FROM` target requires a resolver implementation.

A conformant resolver must:

1. Accept all directives defined in this spec
1. Map supported directives to harness-native configuration
1. Enforce `GUARDRAILS`, `SPENDING`, and `SANDBOX` directives deterministically — not via prompts
1. Ignore unknown directives with a warning (not a fatal error) to allow forward compatibility
1. Emit a resolver manifest documenting which directives it handled, which it ignored, and which it mapped to non-default harness behavior

The resolver interface specification is a future deliverable. See [`CONTRIBUTING.md`](./CONTRIBUTING.md) for how to propose resolver implementations.

-----

## Known Harnesses

See [`docs/harness-guide.md`](./docs/harness-guide.md) for a full breakdown of supported harnesses, their strengths, and appropriate use cases.

|`FROM` target       |Best fit                                                            |
|--------------------|--------------------------------------------------------------------|
|`openclaw:latest`   |Personal assistant, ops monitor — always-on, proactive, multi-domain|
|`claude-code:latest`|Coding agent — git-native, sandboxed, test-driven                   |
|`langgraph:latest`  |Research pipeline — structured graph workflows, human-in-loop       |
|`crewai:latest`     |Team-based workflows — role-based multi-agent collaboration         |
|`autogen:latest`    |Iterative refinement — multi-agent debate and synthesis             |
|`custom-harness:*`  |Domain-specific — trading, medical, legal, high-compliance roles    |

-----

## Version History

See [`CHANGELOG.md`](./CHANGELOG.md).

-----

## Related Documents

- [`docs/design-rationale.md`](./docs/design-rationale.md) — Why this approach over portability-first alternatives
- [`docs/harness-guide.md`](./docs/harness-guide.md) — Harness selection guide
- [`docs/marketplace.md`](./docs/marketplace.md) — Open and proprietary block ecosystem
- [`docs/agent-compose.md`](./docs/agent-compose.md) — Multi-agent coordination
- [`examples/`](./examples/) — Working AgentManifest examples
