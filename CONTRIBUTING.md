# Contributing

## Governance

This repo was created to get the idea moving, not to own it. If a working group,
standards committee, or community organization wants to take over stewardship of
AgentManifest — whether that's the IETF, a foundation, or an informal community
committee — that's explicitly welcome and encouraged.

The goal is for this spec to become something the industry builds on, not something
one person controls. If you represent or want to organize such a group, open an
issue tagged `governance` and let's make it happen.

---

AgentManifest is a design proposal, not a finished standard. Contributions that matter most right now:

1. **Harness resolver implementations** — without these, the spec is theoretical
2. **Corrections to the spec** — directives that don't make sense, missing cases, conflicting semantics
3. **New harness profiles** — if you run agents in production on a harness not covered here
4. **agent-compose design** — the multi-agent coordination layer is a sketch; it needs people who've built multi-agent systems
5. **Real-world examples** — AgentManifests for roles you've actually deployed

---

## RFC Process

This repo uses GitHub Issues and Discussions for RFC-style feedback.

**For small corrections** (typos, clarifications, obvious spec errors):  
Open a PR directly.

**For substantive changes** (new directives, harness resolver interface, agent-compose format):  
Open an Issue first with the `rfc` label. Describe the problem, the proposed change, and any tradeoffs. Discussion happens in the issue before any PR.

**For new harness profiles:**  
Open an Issue with the `new-harness` label. Include: what the harness is, what role it's best for, which AgentManifest directives it handles natively, and which require shimming or can't be supported.

---

## Harness Resolver Contributions

A harness resolver is the adapter between an AgentManifest file and a running agent
on a specific harness. This is the highest-value contribution.

A conformant resolver must:

1. Accept a valid AgentManifest file as input
2. Map all supported directives to harness-native configuration
3. Enforce `GUARDRAILS`, `SPENDING`, and `SANDBOX` at the harness execution layer — not via prompts
4. Emit a resolver manifest listing: directives handled, directives ignored (with warnings), and any non-default harness mappings
5. Ignore unknown directives with a warning rather than a fatal error (forward compatibility)

The resolver interface is not yet formally specified. If you're building a resolver,
open an Issue tagged `resolver-interface` to participate in the interface design
before implementing.

**Target resolvers (highest priority):**
- openclaw
- claude-code / codex-cli
- langgraph
- crewai
- autogen

---

## What's Out of Scope

- Implementing a new agent harness (this spec works with existing harnesses)
- Model fine-tuning or model selection tooling
- Replacing or competing with Oracle's Agent Spec, Docker's docker-agent, or Agentman

---

## Code of Conduct

Be direct and specific. Critique the spec, not the person. Bring production experience when you have it.

---

## License

By contributing, you agree that your contributions will be licensed under CC0 1.0 Universal.
