# The AgentManifest Marketplace

AgentManifest is designed to support a two-sided marketplace of agent components. Every building block an AgentManifest references — harnesses, tools, knowledge bases, guardrail sets, context sources — can come from open community sources or from commercial providers. Both are first-class participants. The spec is neutral; it defines the interface that both sides plug into.

This creates a concrete commercial opportunity for providers who build and maintain domain-specific components: publish an AgentManifest-compatible package, and any developer building agents in your domain can reference it with a single line.

## Open vs. Commercial Components

Every component an AgentManifest references can exist in one of two forms: open or commercial.

**Open blocks** are freely available, forkable, and inspectable. Anyone can reference them. Anyone can improve them. Quality varies by community attention.

**Proprietary blocks** are backed by domain experts — paid, licensed, maintained by specialists. They cost money because they’re worth it: curated, current, and accountable.

This distinction applies to every layer of the stack.

-----

## Block Categories

### Harnesses

```
FROM openclaw:latest                              # open
FROM enterprise-harness:secops-v3@vendorco.io    # proprietary
```

Open harnesses are community-maintained and forkable. Proprietary harnesses are built by vendors with domain expertise — a financial services harness from a trading firm, a compliance harness from a legal tech company, a medical harness from a healthcare IT vendor.

### Tools / Skills

```
TOOLS github.com/openclaw/skills/browser          # open
TOOLS bloomberg-terminal@bloomberg.io             # proprietary
```

Open tool implementations are community-built. Proprietary tools come from vendors who own the underlying system — Bloomberg for financial data, Salesforce for CRM integration, proprietary internal APIs.

### Knowledge Bases

```
KNOWLEDGE github.com/legal-commons/wa-rcw-parsed  # open
KNOWLEDGE wa-realestate-pro@legalmind.io          # proprietary
```

**Example:** A paralegal agent for Washington State real estate law needs accurate, current knowledge of RCW statutes, local regulations, and case precedents.

- `KNOWLEDGE wa-realestate-pro@legalmind.io` — curated by practicing attorneys, kept current, structured annotations, paid subscription, accountable
- `KNOWLEDGE github.com/legal-commons/wa-rcw-parsed` — community-maintained parsed RCW documents, free, forkable, quality depends on contributor attention

A law firm deploying a client-facing agent might pay for the professional knowledge base. An independent developer building a research tool might start with the open version and contribute back. Both paths use the same syntax. The spec is neutral.

### Guardrail Sets

```
GUARDRAILS github.com/openclaw/guardrails/basic-safety  # open
GUARDRAILS soc2-compliant@compliance-firm.io            # proprietary
```

Open guardrail sets are community-maintained policy packs. Proprietary guardrail sets are audited and certified by compliance firms — relevant for regulated industries where “we used a guardrail” needs to hold up in an audit.

### Context Sources

```
CONTEXT github.com/opendata/news-feed     # open
CONTEXT licensed-market-data@refinitiv.io # proprietary
```

Public data feeds versus licensed proprietary data. The spec doesn’t care which — both are declared the same way and enforced the same way.

-----

## The Registry Model

This creates a marketplace dynamic analogous to Docker Hub.

Docker Hub enabled a thriving ecosystem of both open and commercial base images. An AgentManifest block registry could do the same for agent components:

- Harness maintainers ship resolver plugins
- Knowledge base providers publish AgentManifest-compatible packages
- Guardrail vendors certify their rule sets against the spec
- Open-source communities build free alternatives

The spec defines the interface. The registry hosts the blocks. The build tooling resolves references at build time.

**The supply chain questions that apply to npm and Docker apply here too:**

- Who signs harness resolver plugins?
- How do you audit a proprietary guardrail set?
- What’s the deprecation and versioning policy for a licensed knowledge base?
- How do you pin a proprietary block version for reproducible builds?

These are open problems. The spec creates the seam where they need to be solved — it doesn’t solve them today. Block registry governance is a future deliverable.

-----

## Version Pinning

As with any package ecosystem, pinning is strongly recommended for production agents:

```
# Unpinned — not recommended for production
KNOWLEDGE wa-realestate-pro@legalmind.io

# Pinned to a specific release
KNOWLEDGE wa-realestate-pro@legalmind.io:2026-Q1

# Pinned to a hash
KNOWLEDGE wa-realestate-pro@legalmind.io@sha256:abc123...
```

The pinning syntax follows the same pattern as `FROM` version tags and is resolved by the block registry at build time.
