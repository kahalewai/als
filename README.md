<div align="center">

![als](https://github.com/user-attachments/assets/3df546e8-122c-490a-9cad-5c8152cfee06)

Welcome to the Agent Layer Security (ALS) Security Standard Landing

![Status: Draft for Public Comment](https://img.shields.io/badge/Status-Draft%20for%20Public%20Comment-yellow)
![Version: 2.0.0](https://img.shields.io/badge/Version-2.0.0-blue)
![License: Apache 2.0](https://img.shields.io/badge/License-Apache%202.0-green)

<br>

</div>

<br>

## Intro

AI agents communicate autonomously over open protocols, execute actions asynchronously, and operate at machine speed, without a standardized mechanism to stop them. When an agent is compromised, exhibits anomalous behavior, or must be halted to satisfy a human oversight obligation, existing tools operate too slowly, too high in the stack, or from inside the agent's own trust domain. There is no infrastructure-level trip wire. There is no automatic cutoff. There is no circuit breaker to stop the connection. Until now...

<br>

> ALS is THE circuit breaker for AI agent communications

<br>

ALS (Agent Layer Security) is an open, vendor-neutral protocol standard that integrates with TLS and governs AI agent communications at the TLS transport layer; the infrastructure layer beneath the application, beneath OAuth, beneath the agent itself. When an AI Security Service issues a halt command, ALS withdraws the agent's transport credentials. Within seconds, every new connection attempt is rejected at the TLS handshake. The Agent has no control over this action. There is no active revocation infrastructure to protect. No fail-open condition possible. When ALS itself is unavailable, credentials expire automatically; the breaker trips by default.

<br>

ALS was designed from the ground up to satisfy the human oversight and halt mechanism requirements of the regulatory frameworks now governing enterprise AI:

* EU AI Act Article 14.4 - provides the transport-layer "stop button", with a tamper-evident audit trail satisfying Article 12 logging obligations
* NIST AI RMF MANAGE 2.2 - provides the standardized technical response mechanism for AI risk management
* NIST CSF 2.0 RS.MI - maps directly to containment, eradication, and recovery functions
* Zero Trust - enforces continuous verification at the transport layer; credentials are valid for ≤60 seconds, eliminating long-lived trust state

<br>

Tier 2 ALS conformance provides the technical foundation for enterprises to make regulatory compliance claims today, with two additional TLS configuration lines and no application code changes. You read that right, 2 additional lines, no new code.

<br>

ALS works with existing infrastructure, MCP, A2A, TLS 1.3, gRPC, OAuth 2.1, introducing no new cryptographic primitives, no new identity providers, and no proxy insertion into the agent data path. It is the first protocol standard designed to serve as the **enforcement layer for an open AI Security ecosystem**, where threat detection tools, compliance platforms, and human oversight systems converge on a single, standardized interface to stop an agent when it must be stopped.

<br>
<br>

## Why ALS?

Existing mechanisms for stopping AI agents are insufficient:

* OAuth revocation operates at the application layer with enforcement windows measured in hours
* Service mesh circuit breakers self-trigger on failure rates — they cannot accept an external safety command
* Manual process termination operates at human speed, not agent operational tempo
* Application-layer content filters operate inside the agent's trust domain and can be bypassed by the agent itself
* No existing mechanism provides a standardized, auditable, fail-closed transport-layer halt

ALS closes those gaps with an enforcement model that operates below the application layer, independent of the agent's cooperation, and independent of the AI Security Service's continued availability.

<br>
<br>

## How Does ALS Work?

**Step 1 - Agent initializes a governed session**

```
Agent → ALS Service: Initialize session
ALS Service → Agent: Session credential (short-lived TLS client cert, TTL ≤ 60s)
```

**Step 2 - Agent connects to governed endpoints using its credential**

```
Agent → MCP Server / A2A Endpoint: TLS ClientHello + ALS Session Credential
Endpoint: Validates cert chain → ALS CA. Valid = proceed. Expired = reject.
```

**Step 3 - Agent renews its credential on a continuous loop**

```
Agent → ALS Service: Renewal request (authenticated with current credential)
ALS Service: Checks session state → NORMAL: issue new credential | HALTED: refuse
```

**Step 4 - AI Security Service issues a halt command**

```
AI Security Service → ALS Control Channel (gRPC/mTLS): HALT session_id
ALS Service: Transitions session to HALTED. Ceases renewals. Sends push notification.
```

**Step 5 - Enforcement takes effect automatically**

```
Agent credential expires (within ≤ 60 seconds)
All subsequent connection attempts to governed endpoints: rejected at TLS handshake
No new connections possible until authorized RESUME command is issued
```

**Step 6 - Full audit trail is produced independently**

```
Every state transition, command, renewal, and enforcement action → tamper-evident audit log
Audit log is independent of the agent process and inaccessible to governed agents
```

<br>
<br>

## View the ALS Standard

For the complete formal specification, including definitions, requirements, and conformance criteria, see the full reference document:

[ALS-Standard-v2.0.0.md](https://github.com/kahalewai/als/blob/main/als_standard.md)

**Status:** Public Draft for Community Review

**Version:** v2.0.0

**License:** Apache License 2.0

**Date:** 2026-03-15

<br>
<br>

## How Do I Use ALS?

ALS defines three conformance tiers. Tiers are cumulative and designed for incremental adoption — each tier delivers independent value without requiring the other side of the connection to adopt first.

<br>

**Tier 1 - Session Governance (Agent-side)**

Instrument your agents with the ALS session lifecycle. No changes required to any endpoint or server.

* Agent calls `als.initializeSession()` at startup
* Agent uses the returned credential for all MCP/A2A connections
* Agent runs a continuous renewal loop
* Agent connects the push notification WebSocket
* ALS Service provides halt/resume command capability and full audit trail

> *Permitted claim: "ALS session governance and audit trail are active for this deployment."*

<br>

**Tier 2 - TLS-Enforced Governance (Endpoint-side)**

Apply two TLS configuration parameters to your MCP servers and A2A endpoints. No application code changes required.

```typescript
// Node.js — the complete Tier 2 server-side change
const server = https.createServer({
  cert: fs.readFileSync('./server.crt'),
  key:  fs.readFileSync('./server.key'),
  ca:   fs.readFileSync('./als-ca.crt'),   // ← Line 1
  requestCert: true,                        // ← Line 2
  rejectUnauthorized: true
});
```

```python
# Python (uvicorn) — the complete Tier 2 server-side change
uvicorn.run(app,
  ssl_certfile="./server.crt",
  ssl_keyfile="./server.key",
  ssl_ca_certs="./als-ca.crt",          # ← Line 1
  ssl_cert_reqs=ssl.CERT_REQUIRED        # ← Line 2
)
```

> *Permitted claim: "TLS-enforced governance is in effect. ALS enforcement is fail-closed. This deployment satisfies EU AI Act Article 14.4 transport-layer halt requirements."*

<br>

**Tier 3 - ALS Framing Layer (PAUSE/RESUME, Advanced)**

Negotiate the `als/1.0` ALPN token to enable the ALS Framing Layer. Supports PAUSE semantics — holding the TLS connection open while application data is suspended — enabling graceful safe-state procedures before halt enforcement takes effect.

> *Permitted claim: "ALS PAUSE capability is active. This deployment supports Category 1 stop semantics."*

<br>
<br>

## What the ALS Standard Defines

The ALS specification defines:

* Session governance protocol: session lifecycle for governed AI agent connections: initialize, operate, halt, pause, resume, terminate
* Short-lived credential enforcement: X.509 TLS client certificates with ≤60-second TTL; non-renewal = fail-closed enforcement
* Control channel protocol: gRPC-over-mTLS interface by which authorized AI Security Services issue commands
* Push notification protocol: proactive agent notification of session state transitions before enforcement takes effect
* Watchdog mechanism: automatic session halt when credential renewal ceases; cannot be disabled
* Three conformance tiers: Tier 1 (session governance), Tier 2 (TLS enforcement), Tier 3 (PAUSE/RESUME framing layer)
* Tamper-evident audit trail: mandatory events, required fields, retention requirements, and independence guarantees
* Threat model: named threats, mitigations, residual risks, and required complementary controls
* Conformance test suite: 40 test categories covering every normative requirement
* Regulatory compliance mapping: EU AI Act, NIST AI RMF, NIST CSF 2.0, IEC 61508

<br>
<br>

## What the ALS Standard Does NOT Do

Understanding the boundary is as important as understanding the capability:

* ALS does not detect threats. Detection is performed by AI Security Services. ALS receives halt commands and enforces them.
* ALS does not inspect application content. ALS operates at the TLS transport layer and does not parse MCP messages, A2A tasks, or any application-layer data.
* ALS does not replace OAuth or application-layer auth. ALS operates below OAuth at the TLS layer. Both are necessary; neither is sufficient alone.
* ALS does not guarantee in-flight operation rollback. ALS halts transport connections. Application-level safe state is the operator's responsibility.

<br>
<br>

## The ALS Ecosystem Vision

ALS is not just a protocol standard. It is the enforcement foundation for a new generation of AI Security tooling.

The ALS Control Channel defines a standardized gRPC "Command" interface by which any authorized AI Security Service can issue halt, pause, resume, and terminate commands to any ALS-governed session. This interface is intentionally open and interoperable.

**What this means for the ecosystem:**

```
┌─────────────────────────────────────────────────────────────┐
│                   AI SECURITY ECOSYSTEM                     │
│                                                             │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐   │
│  │  Threat      │  │  Compliance  │  │  Human Oversight │   │
│  │  Detection   │  │  Platform    │  │  Console         │   │
│  │  Tools       │  │              │  │                  │   │
│  └──────┬───────┘  └──────┬───────┘  └────────┬─────────┘   │
│         │                 │                   │             │
│         └─────────────────┴───────────────────┘             │
│                           │                                 │
│                    ALS CONTROL CHANNEL                      │
│                    (gRPC / mTLS)                            │
│                           │                                 │
│                    ┌──────▼───────┐                         │
│                    │  ALS SERVICE │                         │
│                    │ (Enforcement)│                         │
│                    └──────┬───────┘                         │
│                           │                                 │
│              TLS Credential Governance                      │
│                           │                                 │
│         ┌─────────────────┴─────────────────┐               │
│         ▼                                   ▼               │
│  ┌─────────────┐                   ┌─────────────────┐      │
│  │  AI Agent   │                   │ Governed        │      │
│  │  (Governed) │◄─────────────────►│ Endpoint        │      │
│  └─────────────┘                   │ (MCP/A2A/etc.)  │      │
│                                    └─────────────────┘      │
└─────────────────────────────────────────────────────────────┘
```

Any AI Security tool, threat detection platforms, behavioral monitors, policy engines, compliance systems, SIEM integrations, can become an ALS Commander. ALS provides the single, standardized enforcement interface they all converge on, so each tool doesn't need to solve the enforcement problem independently.

Examples

* External Prompt Injection Detection Tool detects a successful Prompt Injection, issues command to ALS to HALT the connection on the affected Agent.
* External Agent Action/Execution Tool detects an unauthorized delete command, issues command to ALS to HALT the connection on the affected Agent.
* External MCP Server Integrity Tool detects an unregistered change to MCP Server (Rug Pull), issues command to ALS to HALT the connection to the affected MCP Server.
* External Agent Gateway detects attempt to exfiltrate sensitive data (API Keys), issues command to ALS to HALT the connection on the affected Agent.
* Sensitive Environment requires Human-in-the-Loop (HITL) approval for an action, issues command to ALS to PAUSE the connection on the affected Agent

The vision: one enforcement standard, many security tools. Vendors build detection. ALS handles enforcement. The ecosystem interoperates.

<br>
<br>

## Regulatory Compliance

ALS was designed from the ground up to satisfy the transport-layer halt and human oversight requirements emerging from global AI regulation. Tier 2 conformance provides the technical foundation for the following regulatory claims:

<br>

**EU AI Act - Article 14.4 (Human Oversight, High-Risk AI Systems)**

| ALS Element | Article 14.4 Requirement |
|---|---|
| HALT Command (§8.3.2) | "Interrupt" the system |
| ALS Control Channel (§8.1) | "Stop button or similar procedure" |
| Fail-closed by TTL (DP-2) | Enforcement independent of agent cooperation |
| Push Notification + `effective_in_seconds` | "Stop safely" — controlled shutdown window |
| Tamper-evident Audit Trail (§12) | Article 12 logging requirements |

> Tier 2 is the minimum ALS conformance tier satisfying Article 14.4 for high-risk AI systems.
> Tier 1 alone is **not sufficient** for this claim.

<br>

**NIST AI RMF - MANAGE 2.2**

| ALS Element | MANAGE 2.2 Component |
|---|---|
| HALT Command | Respond — immediate risk response |
| RESUME Command | Recover — restoration after review |
| Watchdog (§9) | Automated detection and response |
| Audit Trail (§12) | Post-incident analysis capability |

<br>

**NIST CSF 2.0 - RS.MI (Incident Mitigation)**

| ALS Element | RS.MI Component |
|---|---|
| HALT Command | Containment |
| TERMINATE Command | Eradication |
| RESUME Command | Recovery |

<br>

**Zero Trust Alignment**

ALS enforces the Zero Trust principle of continuous verification at the transport layer. Session credentials are valid for ≤60 seconds. Every new TLS connection requires a currently-valid, ALS-CA-signed credential. There is no long-lived trust state that can be exploited. ALS governance is independent of the agent's own trust domain.

A full regulatory compliance matrix is available in the ALS Standard Appendix B

<br>
<br>


## Security

ALS was designed with a comprehensive threat model covering control channel threats, credential threats, data plane bypass threats, race conditions, and availability attacks.

* Full threat model is available in the ALS Standard Section 14
* Design principles are normative and serve as tie-breakers for all implementation decisions
* Fail-closed behavior is guaranteed by default, credential expiry requires no operator intervention and no active revocation infrastructure
* The ALS Control Channel is cryptographically isolated from all agent data channels

Key security properties:

| Property | Mechanism |
|---|---|
| Fail-closed by default | TTL-based credential expiry |
| Agent cannot modify own session state | Control Channel isolated from agent data path |
| Enforcement independent of agent | TLS handshake rejection at endpoint |
| Tamper-evident audit | Hash-chain, append-only, or log signing |
| Replay prevention | 300-second nonce window + timestamp validation |
| Watchdog cannot be disabled | No configuration option exists |

<br>
<br>

## Licensing

The ALS specification and all reference materials are released under the Apache License 2.0. The standard is open, royalty-free, and vendor-neutral. Organizations are encouraged to adopt, implement, reference, and build upon it.

<br>
<br>

## Future Work

Future ALS extensions planned:

* QUIC Extension - Governance for QUIC/HTTP3 agent transport
* Session Hierarchy Extension - Multi-agent chain governance with automatic halt propagation, developed in coordination with the A2A specification
* Graduated Enforcement - MONITORED and RESTRICTED session states for proportional response

<br>

## Community and Collaboration

ALS is an open initiative. The standard is published as a Draft for Public Comment and the community's input will shape its evolution.

The project welcomes:

* Feedback on the specification - technical review, edge case analysis, implementation experience
* AI Security Service integrations - if you're building detection or response tooling that should interoperate with ALS, let's collaborate
* Reference implementations - ALS SDK, ALS Service, conformance test harness
* Conformance test contributions - help complete and extend the TC-01 through TC-40 test suite
* Regulatory and compliance analysis - additional framework mappings, jurisdiction-specific guidance
* Ecosystem tooling - dashboards, policy engines, SIEM integrations, Commander implementations

The goal is to establish ALS as the standard enforcement interface for the AI Security ecosystem, so that every AI Security tool, compliance platform, and human oversight system has a single, reliable, interoperable mechanism to halt an agent when it must be stopped.

<br>
<br>
