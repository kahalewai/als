<div align="center">


# ALS Security Standard for MCP


Welcome to the ALS (Agent Layer Security) Security Specification Landing

![als](https://github.com/user-attachments/assets/233d5ae7-4fbf-4f29-9b53-25bc2aada558)

</div>

## Intro

As AI agents and multi-agent systems become more interconnected, they increasingly rely on shared protocols to communicate, coordinate, and act. MCP has emerged as a powerful foundation for agent communication, enabling interoperability and flexible agent-to-agent interaction. However, as these systems move from experimentation into production, a critical gap becomes apparent: a standardized security layer for agent interactions over MCP.

<br>

ALS (Agent Layer Security) addresses this need.

<br>

ALS is a production-ready, vendor-neutral security protocol designed to operate on top of MCP, providing explicit mechanisms for authorization, authenticity, trust, and audit. Rather than embedding security logic into prompts or ad-hoc guardrails, ALS defines a formal protocol that agents and platforms can rely on consistently and verifiably.

<br>

The AI industry currently lacks a shared, interoperable way to answer questions like:

* Who is authorized to perform a given action?
* Can the tools or artifacts being executed be trusted?
* What happens if something changes at runtime?
* How can actions be audited, revoked, or consented to?

<br>

ALS fills this gap by introducing a dedicated security layer for agent interactions, enabling safe, predictable, and governable AI systems without constraining innovation. The goal of ALS is simple: enable secure, auditable, trust-aware agent communication for real-world, production-scale AI systems.

<br>

## ALS Security Model

ALS provides a security and trust reference model for AI agent interactions, focusing on the authorization, verification, and monitoring of agent behavior at runtime. Instead of redefining how agents reason or communicate, ALS defines how security signals are attached to and enforced alongside those interactions.

<br>

ALS is built around a clear separation of concerns:

* MCP handles agent communication and messaging
* ALS enforces authorization, authenticity, and trust
* Runtime systems ensure continuous verification and revocation

<br>

For the complete ALS specification, including cryptographic requirements, token formats, manifests, reputation feeds, and conformance rules, see the full standard here:
[https://github.com/kahalewai/als/blob/main/als_standard.md](https://github.com/kahalewai/als/blob/main/als_standard.md)

<br>
<br>

| Component | Name                          | Responsibility                                 | Security Guarantee Provided                       |
| --------- | ----------------------------- | ---------------------------------------------- | ------------------------------------------------- |
| 1         | Capability Tokens             | Encode fine-grained agent authorization        | Prevents over-privileged or implicit actions      |
| 2         | Transport Binding             | Cryptographically binds tokens to sessions     | Blocks replay and misbinding attacks              |
| 3         | Signed Manifests              | Describe tools and required capabilities       | Ensures tool authenticity and intent transparency |
| 4         | Artifact Verification         | Validates runtime integrity of code and assets | Prevents supply-chain and tampering attacks       |
| 5         | Reputation Signals            | Provide trust and risk context                 | Enables informed allow, warn, or block decisions  |
| 6         | Continuous Runtime Monitoring | Re-verifies trust during execution             | Detects and stops mid-session compromise          |
| 7         | Revocation & Audit Mechanisms | Revoke authority and log actions               | Enables accountability and rapid containment      |

<br>

This model is implementation-agnostic and applies to single agents, multi-agent systems, agent marketplaces, tool ecosystems, and shared AI platforms. Security is treated as a first-class runtime concern, not a best-effort policy layered on top of probabilistic behavior.

<br>

ALS is designed to strengthen MCP, not duplicate MCP, or complicate MCP. ALS provides a new security protocol for safe interoperability at scale. Similar to how TLS is run on top of TCP at the transport layer, ALS runs on top of MCP at the application layer, and provides a new set of security controls specific to Agent communication over MCP connections.

<br>

## Contribute

ALS is a production-ready open security standard, developed to evolve through transparent, community-driven collaboration. This repository is the canonical home for the ALS specification, examples, and reference materials.

<br>

We invite AI platform builders, security engineers, protocol designers, researchers, and standards authors to help shape the future of secure agent communication.

<br>

Participation includes:

* Contributing clarifications, examples, and reference flows
* Proposing extensions aligned with ALS security principles
* Reviewing changes for cryptographic and protocol correctness
* Advancing interoperability and ecosystem adoption

<br>

Why contribute?

* Help define a foundational security layer for agent ecosystems
* Enable secure, auditable MCP-based systems
* Establish shared trust primitives for AI interoperability
* Influence the future of safe, scalable agent communication

<br>

How to get started:

1. Fork the repository.
2. Review the ALS specification and design principles.
3. Open a Pull Request with proposed improvements or additions.
4. Participate in open technical discussions and reviews.

<br>

By working together, we can ensure ALS becomes a widely adopted, interoperable security standard that enables trustworthy AI agent interactions across the entire ecosystem.

