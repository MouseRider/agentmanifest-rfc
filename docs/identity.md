# Agent Identity in AgentManifest

The `IDENTITY` directive assigns a cryptographic identity to an agent — immutable per manifest version, verifiable by external systems.

```
IDENTITY did:web:agents.example.com:assistant
```

Identity in AgentManifest is not a label. It’s a binding point. A verifiable identity is what allows an agent to be trusted by systems that require an accountable, auditable party on the other end of a transaction or access request.

-----

## What Verifiable Identity Enables

### Behavioral SLAs

Once an agent’s identity is tied to a specific manifest version, its capability claims become verifiable. You can assert — and prove — that this agent, as deployed from this manifest version, has specific guardrails enforced and operates within declared constraints.

“This agent will not take write actions without approval” stops being a prompt-level claim and becomes an auditable property of the deployment. The manifest version is the record; the identity links it to a running instance.

### Wallet and Payment Binding

An agent with a stable cryptographic identity can be issued a wallet — a spending account scoped to that identity, not to a human user or a shared service credential.

```
IDENTITY did:web:agents.example.com:purchasing-agent
SPENDING daily-cap=500.00, per-transaction-cap=100.00, currency=USD
```

The `SPENDING` directive declares the limits. The wallet enforces them at the infrastructure level. The agent can initiate purchases, pay for API calls, settle invoices, and interact with payment rails — within the bounds declared in its manifest, under an identity that’s traceable back to a specific spec version.

If something goes wrong, the audit trail is complete: which agent, which manifest version, which guardrails were in force, what it spent, and when. The identity is the thread that connects all of it.

This is meaningfully different from embedding payment credentials in configuration or routing all agent spending through a shared service account. Each agent has its own financial identity, with its own spending envelope, independently auditable.

### OAuth and API Credential Binding

Rather than embedding API keys in configuration files or passing credentials through prompts, the harness can resolve access rights from the agent’s verified identity at runtime.

An agent identity can be:

- The subject of an OAuth client credential (`client_id` scoped to that agent, not a human)
- The principal for a service account in enterprise identity systems (Azure AD, Google Workspace, AWS IAM)
- A member of a permissioned data feed or licensed API with per-agent access controls

This makes credential management tractable at scale. When an agent is retired or its manifest is updated to a new version, access can be revoked or reissued by identity — without hunting down where credentials were embedded.

### Permissioned Data and Knowledge Access

Knowledge bases and data feeds in the marketplace model can scope access by agent identity. A financial data provider can issue access to a specific agent identity, with usage tracked and billed per-agent rather than per-organisation. A legal knowledge base can grant access to a paralegal agent while denying it to a general-purpose assistant running in the same system.

```
KNOWLEDGE bloomberg-terminal@bloomberg.io  # access granted to this agent identity
KNOWLEDGE wa-realestate-pro@legalmind.io
```

The harness presents the agent’s identity when resolving these blocks. The provider enforces access controls on their side.

-----

## Inter-Agent Trust

In an agent-compose topology, agent identity enables trust verification between agents.

A coordinator agent delegating to a specialist can verify that the specialist is genuinely running the manifest it claims — same spec version, same guardrails in force, same identity. This is the difference between trusting that an agent *says* it’s a compliance reviewer and verifying that it *is* the compliance reviewer manifest, unchanged, with the expected guardrails active.

In council and consensus topologies, identity makes the audit trail meaningful. A council decision record includes:

- The verified identity of each participating agent
- The manifest version each was running at the time
- The guardrails declared in each manifest
- The independent evaluation each agent produced

Without identity, a council is just a group of agents. With identity, it’s an accountable decision-making structure with a verifiable record.

-----

## Identity Federation

When multiple agent-compose systems interact — or when agents from different organisations need to establish trust — identity federation becomes relevant.

Open questions in this space (not resolved in v0.3):

- How does an agent-compose file express which agent identities are permitted to call which other agents?
- How do you revoke an agent’s identity without taking down its harness?
- How does cross-organisation agent identity verification work when agents are calling external APIs or other organisations’ agents?

These are open design questions. The v0.3 spec establishes the identity primitive; the federation layer is future work.

-----

## DID and Identity Standards

AgentManifest uses W3C Decentralized Identifiers (DIDs) as the identity format, specifically `did:web` as the default method. This is intentional: `did:web` is resolvable without a blockchain, uses standard HTTPS infrastructure, and is straightforward for organisations to issue and manage.

Other DID methods are supported by harness resolvers. The spec is neutral on method as long as the identity is:

- Verifiable (can be checked by external systems)
- Immutable per manifest version (changing the manifest produces a new version, not a mutation of the existing identity)
- Resolvable to a DID document that includes the public key material needed for verification

-----

## Status

The `IDENTITY` directive is part of the AgentManifest v0.3 spec. Harness-level identity resolution, wallet binding, and OAuth integration are design-complete but not yet implemented in any reference resolver. Feedback on the identity model is welcome — open an issue in the repo.
