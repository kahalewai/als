# AGENT LAYER SECURITY (ALS) Standard

### Begin ALS Standard

<br>

**Title:** Agent Layer Security (ALS) Protocol Standard

**Status:** DRAFT FOR PUBLIC COMMENT

**Date:** 2026-03-15

**Authors:** Shawn Kahalewai Reilly

**Version:** 2.0.0

**License:** Apache License, Version 2.0

<br>

### ABSTRACT

The industry lacks a standardized infrastructure-layer mechanism to halt
AI agent communications at the transport layer when an agent must be
stopped for safety, security, or compliance reasons. Agent Layer Security
(ALS) fills this gap by defining a session governance protocol that issues
short-lived TLS client certificates to governed AI agents; governed
endpoints enforce governance natively using two standard TLS configuration
options; and an external AI Security Service commands halt and resume via
an authenticated control channel. ALS operates at the TLS transport layer
and does not inspect application-layer content. ALS provides
fail-closed-by-default behavior: if the ALS service becomes unavailable,
agent credentials expire automatically within the configured TTL (maximum
60 seconds), halting new connections without requiring operator
intervention. ALS is designed to satisfy the transport-layer halt
mechanism requirements of EU AI Act Article 14.4 and NIST AI RMF MANAGE
2.2. ALS does not provide threat detection, application-layer
authorization, multi-agent chain governance, or QUIC transport governance
(deferred to a future version).

<br>

### TABLE OF CONTENTS

1. Introduction
2. Glossary
3. Design Principles
4. ALS Session Model
5. Session Credential Protocol
6. Governed Endpoint Configuration
7. ALS Certificate Authority Requirements
8. Control Channel Protocol
9. Watchdog Mechanism
10. Push Notification Protocol
11. TLS Extensions
12. Audit Requirements
13. Tier 3 — ALS Framing Layer (Optional Extension)
14. Threat Model and Security Considerations
15. Conformance
16. Implementation Guidance (Informative)
17. Versioning and Protocol Evolution
- Appendix A: Audit Event Schema Reference (Normative)
- Appendix B: Regulatory Compliance Matrix (Informative)
- Appendix C: Known Gaps and Future Work (Informative)
- Appendix D: Exploration Lineage (Informative)
- Appendix E: Audit Summary (Informative)

<br>

## SECTION 1: INTRODUCTION
*Informative*

### 1.1 Background

Artificial intelligence agents increasingly communicate over open
protocols, specifically the Model Context Protocol (MCP, specification
2025-11-25) and the Agent-to-Agent Protocol (A2A, version 1.0). Both
protocols use TLS-secured HTTP as their transport. Both define
application-layer authentication and authorization mechanisms, OAuth 2.1
for MCP, OpenAPI-compatible authentication for A2A. Neither protocol
defines an infrastructure-level mechanism by which an external authority
can halt an active, authorized agent session.

This gap is not an oversight in either protocol. MCP and A2A deliberately
delegate transport security to TLS and focus their security specifications
on application-layer concerns. The consequence is that the infrastructure
enforcement layer, the ability to halt a governed session from outside
the agent's trust domain, does not exist in either protocol's design
space.

This standard defines that infrastructure enforcement layer.

### 1.2 Problem Statement

The industry is missing a protocol-layer circuit breaker for AI agents.
When an AI agent operating over MCP or A2A exhibits anomalous behavior,
is determined to be compromised, or must be stopped to satisfy a human
oversight obligation, there is no standardized, reliable, auditable
mechanism to halt its communications at the transport layer.

Existing mechanisms are insufficient:

- **OAuth revocation** operates at the application layer and has
  enforcement windows measured in hours.
- **Service mesh circuit breakers** are self-triggering based on observed
  failure rates and cannot accept external safety commands.
- **Manual process termination** operates at human speed, not agent
  operational tempo.
- **Application-layer content filters** operate in the same trust domain
  as the agent.

### 1.3 What This Standard Provides

Agent Layer Security (ALS) defines:

1. A **session governance protocol** — a standard session lifecycle for
   governed AI agent connections: initialization, operation, halt, pause,
   resume, and termination.

2. A **short-lived credential enforcement mechanism** — ALS issues
   short-lived TLS client certificates. Governed endpoints validate these
   certificates using two standard TLS configuration options. Non-renewal
   produces fail-closed enforcement automatically.

3. A **control channel protocol** — a standardized gRPC-over-mTLS
   interface by which authorized AI Security Services issue halt and
   resume commands.

4. A **push notification protocol** — proactive notification to governed
   agents of session state transitions before enforcement takes effect.

5. A **watchdog mechanism** — automatic session halt when a governed
   agent fails to renew its credentials within a specified period.

6. An **audit trail specification** — mandatory events, required fields,
   and tamper-evidence requirements for all enforcement actions.

### 1.4 What This Standard Does Not Provide

- **Threat detection.** ALS receives halt commands; it does not generate
  them. Detection is performed by AI Security Services.

- **Application-layer authorization.** ALS does not replace OAuth 2.1
  (MCP) or A2A authentication schemes.

- **Safe state guarantees for application logic.** ALS halts transport
  connections. Application-level safe state is the operator's
  responsibility.

- **Multi-agent chain governance.** ALS v1.0 governs individual sessions.
  Automatic halt propagation through delegation chains is deferred
  (Appendix C).

- **QUIC/HTTP3 transport governance.** ALS v1.0 is defined for TCP+TLS
  transports only. QUIC support is deferred (Appendix C).

### 1.5 Relationship to Other Standards

- **MCP (2025-11-25):** ALS governs MCP transport sessions without
  modifying MCP protocol semantics.

- **A2A (v1.0):** ALS governs A2A transport sessions without modifying
  A2A task lifecycle or authentication.

- **SPIFFE/SPIRE:** ALS uses SPIFFE-compatible X.509 SVIDs extended with
  session governance state.

- **OAuth 2.1:** ALS operates below OAuth at the TLS layer. Both are
  necessary; neither is sufficient alone.

- **EU AI Act Article 14.4:** ALS provides the transport-layer halt
  mechanism required for high-risk AI systems. See Section 16.7.

- **NIST AI RMF MANAGE 2.2:** ALS provides the technical response
  mechanism. See Section 16.7.

### 1.6 Document Conventions

This document uses RFC 2119 normative language (BCP 14, updated by RFC
8174) as defined in Section 2.2. Normative sections contain requirements
affecting conformance. Informative sections contain guidance that does
not affect conformance.

<br>

## SECTION 2: GLOSSARY
*Normative*

The following terms are used normatively throughout this standard. All
terms are defined before first normative use.

**ALS (Agent Layer Security):** The protocol defined by this standard.

**ALS CA (ALS Certificate Authority):** The certificate authority whose
certificates are used by ALS to sign session credentials. The trust
anchor for governed endpoint TLS configuration.

**ALS CA Certificate:** The X.509 certificate of the ALS CA or an
intermediate CA in the ALS CA hierarchy, installed by governed endpoints
as a trusted CA for client certificate validation.

**ALS Framing Layer:** An optional application-layer protocol defined in
Section 13, negotiated via the `als/1.0` ALPN token, enabling PAUSE and
RESUME session state transitions. Required only for Tier 3 conformance.

**ALS Service:** The server-side ALS infrastructure that manages sessions,
issues credentials, receives control channel commands, operates the
watchdog, delivers push notifications, and produces audit records.

**ALS Session:** A governance unit representing a single governed agent's
authorization to operate. Has a unique Session ID, Session State,
authorized Commanders, Session Credential, and audit record. Distinct
from a TLS connection — one Session may correspond to multiple TLS
connections.

**ALS Session Credential:** A short-lived X.509 certificate issued by the
ALS CA to a Governed Agent for use as a TLS client certificate.

**AI Security Service:** An authorized external system that issues
commands to the ALS Service via the Control Channel. Responsible for
threat detection and governance decisions. ALS is responsible for
enforcement.

**ALPN Token:** An Application-Layer Protocol Negotiation identifier (RFC
7301). ALS defines the token `als/1.0` for capability advertisement and
Tier 3 negotiation.

**Audit Event:** A structured, timestamped record produced by the ALS
Service for every mandatory event in Section 12.1. Tamper-evident and
independent of the Governed Agent's logging.

**Commander:** An AI Security Service pre-provisioned as authorized to
issue commands to a specific ALS Session.

**Commander Certificate:** The X.509 certificate used by an AI Security
Service to authenticate to the ALS Control Channel via mTLS.
Pre-provisioned at ALS Service deployment time.

**Conformance Tier:** A defined level of ALS capability. ALS defines
three tiers: Tier 1, Tier 2, Tier 3. Tiers are cumulative.

**Control Channel:** The gRPC-over-mTLS channel by which AI Security
Services issue commands to the ALS Service. Cryptographically isolated
from all Governed Agent data channels.

**CSR Mode:** A credential issuance mode in which the Governed Agent
generates its own keypair, submits a Certificate Signing Request, and
receives a signed certificate without the ALS Service generating or
handling the private key.

**Effective Seconds:** The number of seconds remaining before the
Governed Agent's current Session Credential expires at the time a push
notification is sent. Carried in the `effective_in_seconds` field.

**Enforcement Tier:** The Conformance Tier at which a given ALS Session
is operating. Reported in session state and queryable by AI Security
Services.

**Governed Agent:** An AI agent that has initialized an ALS Session and
presents an ALS Session Credential in TLS handshakes with Governed
Endpoints.

**Governed Endpoint:** An MCP server, A2A agent endpoint, or any TLS
server configured to require ALS Session Credentials from connecting
clients.

**HALT Command:** A control channel command that transitions a Session
from NORMAL to HALTED. Stops credential renewals for the Session.

**Keypair Issuance Mode:** A credential issuance mode in which the ALS
Service generates both private key and certificate, returning both to the
Governed Agent. Default mode.

**PAUSE Command:** A Tier 3 control channel command that transitions a
Session from NORMAL to PAUSED. The TLS connection is established but the
ALS Framing Layer holds application data.

**Pre-Shared Initialization Credential:** A static credential provisioned
by the ALS Service operator to a Governed Agent, used to authenticate the
session initialization request before the first Session Credential is
issued.

**Push Notification Channel:** A persistent WebSocket connection from the
ALS Service to the Governed Agent, delivering session state change
notifications before enforcement takes effect.

**Renewal Endpoint:** The HTTPS endpoint at which a Governed Agent
presents its current, non-expired Session Credential to obtain a new
Session Credential.

**Replay Window:** The time period during which the ALS Service retains
processed command nonces to detect replay attacks. 300 seconds per
REQ-8.5.1.

**RESUME Command:** A control channel command that transitions a Session
from HALTED or PAUSED back to NORMAL.

**Session ID:** A globally unique, cryptographically random identifier
assigned to an ALS Session at initialization. Minimum 128 bits of
entropy. URL-safe base64-encoded, no padding.

**Session State:** The current governance state of an ALS Session. Valid
states: INITIALIZING, NORMAL, HALTED, PAUSED (Tier 3 only), TERMINATED.

**TERMINATE Command:** A control channel command that permanently closes
an ALS Session with no possibility of resumption.

**TTL (Time To Live):** The validity period of a Session Credential in
seconds, from `valid_from` to `valid_until`.

**Tamper-Evident Log:** An audit log where unauthorized modification is
detectable via hash-chaining, append-only storage, or log signing.

**Trust Domain:** A named administrative domain within which SPIFFE
identities are issued and trusted. Corresponds to the trust domain
component of SPIFFE IDs (`spiffe://<trust-domain>/...`).

**Watchdog:** An ALS Service component that monitors Session Credential
renewal activity and automatically transitions a Session to HALTED if
renewal is not received within the configured watchdog window.

**Watchdog Multiplier:** A configuration parameter determining the
watchdog window as a multiple of the Session TTL. Default: 2.0.
Maximum: 3.0.

**Watchdog Window:** The period in seconds during which at least one
successful credential renewal is expected. Calculated as:
TTL × Watchdog Multiplier.

**`als_session_id` Extension:** A TLS ClientHello extension defined in
Section 11.1, carrying the ALS Session ID for session correlation at
Governed Endpoints.

### 2.2 Normative Language Policy

This standard uses RFC 2119 (BCP 14) normative keywords as updated by
RFC 8174:

- **MUST / SHALL:** Absolute requirement. Non-compliance = non-conformant.
- **MUST NOT / SHALL NOT:** Absolute prohibition. Includes all modes.
- **SHOULD / RECOMMENDED:** Strong preference. Deviation requires
  documented justification and must not weaken any MUST requirement.
- **MAY / OPTIONAL:** Permitted but not required.

**ALS-specific extension:** Requirements prefixed `TIER-3-MUST` are
mandatory for Tier 3 conformance claims only. Tier 1/2 implementations
may safely ignore them.

The word "required" in prose uses its ordinary English meaning and is not
a normative keyword.

<br>

## SECTION 3: DESIGN PRINCIPLES
*Normative*

The following principles are normative. Any implementation decision
conflicting with these principles is non-conformant regardless of whether
a specific requirement addresses the conflict. When requirements are
ambiguous, these principles are the authoritative tie-breakers.

**DP-1 - ENFORCEMENT BY CREDENTIAL WITHDRAWAL**
ALS enforcement SHALL be achieved exclusively by withholding Session
Credential renewal. ALS MUST NOT require proxy insertion in the agent
data path for core conformance (Tier 1 or Tier 2). ALS MUST NOT inspect
application-layer content of MCP or A2A messages as a condition of
enforcement.

**DP-2 - FAIL-CLOSED BY PHYSICS**
The default outcome of any ALS Service failure — including unavailability
of the Renewal Endpoint, network partition, or credential issuance
failure — SHALL be cessation of governed agent connections, occurring
automatically without operator intervention. No conformant ALS
implementation SHALL provide a configuration option producing fail-open
behavior in response to ALS Service unavailability.

**DP-3 - INDEPENDENCE OF THE SAFETY CHANNEL**
The ALS Control Channel SHALL be cryptographically isolated from all
Governed Agent data channels. Governed Agents SHALL NOT have any API
access that can modify their own Session State. Commander Certificates
SHALL be provisioned at ALS Service deployment time and SHALL NOT be
negotiable by Governed Agents at session initialization.

**DP-4 - ADOPTION WITHOUT COORDINATION**
Each side of an ALS-governed connection SHALL derive independently
verifiable value from ALS adoption without requiring the other side to
have adopted ALS first. Tier 1 conformance SHALL require no changes to
Governed Endpoints. Tier 2 conformance SHALL require no application code
changes to Governed Endpoints.

**DP-5 - VERIFIABLE ENFORCEMENT**
Every ALS enforcement action SHALL produce an Audit Event sufficient for
an independent party to verify that the enforcement occurred, when it
occurred, who authorized it, and what was enforced. Every normative
requirement SHALL be independently testable by a conformance test.

**DP-6 - PROPORTIONAL ENFORCEMENT**
ALS SHALL provide distinct enforcement primitives for distinct threat
response scenarios. HALT, PAUSE, and TERMINATE SHALL be distinct commands
with distinct semantics, distinct Session State transitions, and distinct
audit events.

**DP-7 - MINIMUM VIABLE COMPLEXITY**
Every normative requirement SHALL be justified by a specific safety,
security, or compliance need. Tier 2 core compliance SHALL require no
more than two server-side TLS configuration changes. No new cryptographic
primitive SHALL be defined — all cryptographic operations SHALL use
IANA-registered or standards-track mechanisms.

**DP-8 - AUDIT TRAIL IS NON-NEGOTIABLE**
Every Session State transition, credential issuance, command received,
command executed, and watchdog event SHALL produce a Tamper-Evident Audit
Event. Audit logging SHALL be independent of the Governed Agent process.
Enforcement SHALL proceed even when audit logging is temporarily degraded,
but degraded audit logging SHALL itself trigger an alert.

**DP-9 - PROTOCOL AGNOSTICISM AT THE ENFORCEMENT LAYER**
ALS enforcement mechanisms SHALL operate identically regardless of
whether the governed connection carries MCP, A2A, or any other TLS-based
agent protocol. ALS SHALL NOT require updates when governed protocols
release new versions.

**REQ-DP9-ENFORCEMENT:** A conformant ALS Service implementation SHALL
NOT include any normative behavior conditional on the application protocol
carried by a governed TLS connection. ALS SHALL NOT parse MCP message
structures, A2A task IDs, or any application-layer content as a condition
of session governance decisions. This requirement does not prohibit ALS
implementations from offering optional Layer 7 features as
non-conformance-affecting extensions, provided such features are clearly
marked as outside ALS scope.

<br>

## SECTION 4: ALS SESSION MODEL
*Normative*

### 4.1 Session Identity and Scope

**REQ-4.1.1:** Each ALS Session SHALL have a globally unique Session ID
assigned at initialization by the ALS Service. Session IDs SHALL be
generated using a cryptographically random source with a minimum of 128
bits of entropy. Session IDs SHALL be represented as URL-safe
base64-encoded strings with no padding.

**REQ-4.1.2:** An ALS Session SHALL represent exactly one Governed
Agent's authorization to operate within one ALS deployment. A single
agent process MAY hold multiple concurrent ALS Sessions. Each Session
SHALL be independently governed.

**REQ-4.1.3:** An ALS Session SHALL be associated at initialization with:
a Session ID, a Session State (initially INITIALIZING), a list of one or
more Commanders, a watchdog configuration, an agent identity (SPIFFE ID
or X.509 Subject), and an ALS Service instance identifier.

**REQ-4.1.4:** A Session ID SHALL NOT be reused after a Session reaches
TERMINATED state.

### 4.2 Session State Machine

**Valid Session States:**

| State        | Description                                                   |
|--------------|---------------------------------------------------------------|
| INITIALIZING | Session requested; credential issuance in progress.           |
| NORMAL       | Session active. Credentials issued and renewed.               |
| HALTED       | Credential renewal suspended. New connections fail after TTL. |
| PAUSED       | (Tier 3) Connection held; application data suspended.         |
| TERMINATED   | Permanently closed. No credentials will be issued.            |

**Valid State Transitions:**

| From State   | To State    | Authorized Actor           | Trigger                    | Audit Event              |
|--------------|-------------|----------------------------|----------------------------|--------------------------|
| (none)       | INITIALIZING| Governed Agent             | Session init request       | SESSION_INIT_REQUESTED   |
| INITIALIZING | NORMAL      | ALS Service                | Successful issuance        | SESSION_INITIALIZED      |
| INITIALIZING | TERMINATED  | ALS Service                | Initialization failure     | SESSION_INIT_FAILED      |
| NORMAL       | HALTED      | AI Security Service        | HALT Command               | SESSION_HALTED           |
| NORMAL       | HALTED      | ALS Service (Watchdog)     | Watchdog timeout           | SESSION_WATCHDOG_HALT    |
| NORMAL       | PAUSED      | AI Security Service        | PAUSE Command (Tier 3)     | SESSION_PAUSED           |
| NORMAL       | TERMINATED  | AI Security Service        | TERMINATE Command          | SESSION_TERMINATED       |
| HALTED       | NORMAL      | AI Security Service        | RESUME Command             | SESSION_RESUMED          |
| HALTED       | TERMINATED  | AI Security Service        | TERMINATE Command          | SESSION_TERMINATED       |
| PAUSED       | NORMAL      | AI Security Service        | RESUME Command (Tier 3)    | SESSION_RESUMED          |
| PAUSED       | HALTED      | ALS Service                | hold_timeout exceeded      | SESSION_HALTED           |
| PAUSED       | TERMINATED  | AI Security Service        | TERMINATE Command (Tier 3) | SESSION_TERMINATED       |

**REQ-4.2.1:** The ALS Service SHALL implement the session state machine
exactly as defined in Section 4.2. Transitions not listed SHALL NOT
occur.

**REQ-4.2.2:** The ALS Service SHALL produce the specified Audit Event
for every state transition within 100 milliseconds of the transition
occurring.

**REQ-4.2.3:** Governed Agents SHALL NOT be able to trigger any session
state transition. Transitions are authorized only for the actors
specified in the state transition table.

### 4.3 Session Initialization Protocol

**REQ-4.3.1:** A Governed Agent SHALL initiate session initialization by
sending an authenticated HTTPS POST request to the ALS Service
initialization endpoint, including: agent identity (SPIFFE ID or X.509
Subject), list of authorized Commander identities, watchdog
configuration, desired TTL, and supported ALS version.

**REQ-4.3.2:** The ALS Service SHALL authenticate the session
initialization request using mTLS or a Pre-Shared Initialization
Credential. The ALS Service SHALL NOT accept unauthenticated
initialization requests.

**REQ-4.3.3:** Upon successful initialization, the ALS Service SHALL:
(a) Assign a Session ID; (b) transition the Session to NORMAL;
(c) issue a Session Credential; (d) return the session initialization
response; (e) produce a SESSION_INITIALIZED Audit Event.

**REQ-4.3.4:** The session initialization response SHALL include the
Enforcement Tier at which the Session is operating, determined by the
capabilities advertised in the initialization request.

**REQ-4.3.5:** The ALS Service SHALL validate that each Commander
identity in the initialization request matches a pre-provisioned
Commander Certificate in the ALS Service's commander trust store. If any
Commander identity does not match, the ALS Service SHALL reject the
request with error code `COMMANDER_NOT_PROVISIONED`.

### 4.4 Session State Transitions: HALT, PAUSE, RESUME, TERMINATE

**REQ-4.4.1:** Upon receiving a validated HALT Command for a Session in
NORMAL state, the ALS Service SHALL: (a) immediately transition to
HALTED; (b) immediately cease issuing credential renewals;
(c) produce a SESSION_HALTED Audit Event; (d) send a push notification
within 1 second.

**REQ-4.4.2:** The ALS Service SHALL NOT resume issuing credential
renewals for a HALTED Session until a valid RESUME Command is received
from an authorized Commander.

**REQ-4.4.3:** Upon receiving a validated RESUME Command for a HALTED
Session, the ALS Service SHALL: (a) transition to NORMAL; (b) resume
issuing credential renewals; (c) issue a new Session Credential
immediately; (d) produce a SESSION_RESUMED Audit Event; (e) send a push
notification.

**REQ-4.4.4:** Upon receiving a validated TERMINATE Command for a Session
in any non-TERMINATED state, the ALS Service SHALL: (a) immediately
transition to TERMINATED; (b) permanently cease issuing credentials;
(c) produce a SESSION_TERMINATED Audit Event; (d) send a final push
notification if the Push Notification Channel is active.

**REQ-4.4.5:** A TERMINATED Session SHALL NOT be resumed. The Session ID
of a TERMINATED Session SHALL NOT be reused.

**REQ-4.4.6:** A PAUSE Command received by an ALS Service that does not
implement Tier 3 SHALL be rejected with error code
`CAPABILITY_NOT_SUPPORTED`.

### 4.5 Watchdog-Triggered Transitions

**REQ-4.5.1:** A Watchdog-triggered HALT SHALL produce the audit event
SESSION_WATCHDOG_HALT and SHALL be distinguishable from a
Commander-triggered HALT in all audit records and session state queries.

### 4.6 Session Termination and Cleanup

**REQ-4.6.1:** Upon session termination, the ALS Service SHALL retain
all Audit Events for the Session for the minimum audit retention period
defined in Section 12.5.

**REQ-4.6.2:** The ALS Service SHALL NOT delete Session Audit Events in
response to any request from a Governed Agent.

**REQ-4.6.3:** The ALS Service SHOULD retain Session metadata (Session
ID, agent identity, state history, Commander list) for the audit
retention period after TERMINATED state, to support post-incident
investigation.

<br>

## SECTION 5: SESSION CREDENTIAL PROTOCOL
*Normative*

### 5.1 Session Initialization Request

**REQ-5.1.1:** The session initialization request SHALL be an HTTPS POST
to the ALS Service initialization endpoint with
`Content-Type: application/json`. Required fields:

| Field               | Type            | Required | Description                              |
|---------------------|-----------------|----------|------------------------------------------|
| `als_version`       | string          | MUST     | ALS version supported by agent (semver)  |
| `agent_identity`    | string          | MUST     | SPIFFE ID or X.509 Subject DN            |
| `commanders`        | array of string | MUST     | Commander identities. Minimum 1.         |
| `ttl_seconds`       | integer         | SHOULD   | Requested TTL. Subject to §5.4 limits.   |
| `watchdog_multiplier`| number         | MAY      | Default 2.0. Maximum 3.0.                |
| `issuance_mode`     | string          | MAY      | `"keypair"` (default) or `"csr"`         |
| `csr_pem`           | string          | MUST if `issuance_mode="csr"` | PEM-encoded CSR |
| `capabilities`      | array of string | SHOULD   | Capabilities supported by this agent     |
| `enforcement_tier`  | integer         | SHOULD   | Highest tier this agent supports (1–3)   |

**REQ-5.1.2:** The ALS Service SHALL validate all required fields and
SHALL reject requests with missing required fields with HTTP 400 and
error code `INVALID_REQUEST`.

**REQ-5.1.3:** The ALS Service session initialization endpoint and
Renewal Endpoint SHALL use TLS 1.3 exclusively. TLS 1.2 SHALL NOT be
accepted on these endpoints.

### 5.2 Session Initialization Response

**REQ-5.2.1:** The session initialization response SHALL be a JSON object
with the following structure:

```json
{
  "als_version": "1.0.0",
  "session_id": "<128-bit random, base64url-encoded>",
  "als_signing_certificate": "<PEM — ALS CA or intermediate CA>",
  "enforcement_tier": 1,
  "capabilities": [
    "credential_issuance", "renewal", "watchdog",
    "push_notification", "halt_command", "audit_logging"
  ],
  "session_channel_credential": {
    "client_certificate": "<PEM — signed by ALS CA>",
    "client_private_key": "<PEM — keypair mode only>",
    "valid_from": "<ISO 8601 UTC>",
    "valid_until": "<ISO 8601 UTC — 30 to 60 seconds from valid_from>",
    "renewal_endpoint":
      "https://als.internal:7400/session/{session_id}/credential/renew",
    "status_endpoint":
      "https://als.internal:7400/session/{session_id}/status",
    "push_endpoint":
      "wss://als.internal:7400/session/{session_id}/notifications"
  },
  "watchdog_config": {
    "multiplier": 2.0,
    "window_seconds": 60
  }
}
```

**REQ-5.2.2:** The `client_private_key` field SHALL be present if and
only if `issuance_mode` is `"keypair"`. In CSR Mode this field SHALL be
absent. The ALS Service SHALL NOT retain the private key after returning
it in the initialization response.

**REQ-5.2.3:** The `status_endpoint` field SHALL provide an authenticated
HTTPS GET endpoint at which the Governed Agent may query current Session
State at any time.

**REQ-5.2.4:** The `push_endpoint` field SHALL provide a WebSocket
endpoint for the Push Notification Channel. The Governed Agent SHOULD
connect immediately after session initialization.

### 5.3 Session Credential Content Requirements (X.509 Profile)

**REQ-5.3.1:** ALS Session Credentials SHALL be X.509 v3 certificates
signed by the ALS CA or an ALS intermediate CA.

**REQ-5.3.2:** The Subject Alternative Name (SAN) extension SHALL be
present and SHALL contain:
(a) A URI SAN carrying the SPIFFE ID:
    `spiffe://<trust-domain>/<path>`
(b) A URI SAN carrying the ALS Session ID:
    `als://<als-service-domain>/session/<session-id>`

**REQ-5.3.3:** The Subject Distinguished Name SHALL carry the ALS Session
ID in the Common Name field:
`ALS-SESSION-<session-id>`

**REQ-5.3.4:** The certificate SHALL include:
(a) Key Usage: digitalSignature, keyEncipherment (critical)
(b) Extended Key Usage: clientAuth (1.3.6.1.5.5.7.3.2) (critical)
(c) Basic Constraints: CA:FALSE (critical)
(d) Subject Key Identifier (non-critical)
(e) Authority Key Identifier (non-critical)

**REQ-5.3.5:** The certificate SHALL NOT include DNSName or IPAddress
SANs unless required for deployment-specific endpoint routing.

**REQ-5.3.6:** The certificate validity period SHALL be exactly TTL
seconds. No grace period.

**REQ-5.3.7:** Session Credentials SHALL use P-256 (ECDSA) or RSA-2048
(minimum) key pairs. P-384 or RSA-4096 SHOULD be used for high-security
deployments. MD5, SHA-1, and RSA-1024 or smaller SHALL NOT be used.

### 5.4 TTL Requirements and Constraints

**REQ-5.4.1:** The minimum allowed Session Credential TTL SHALL be 15
seconds.

**REQ-5.4.2:** The maximum allowed Session Credential TTL for standard
deployments SHALL be 60 seconds. ALS Service implementations SHALL
reject requests specifying TTL greater than 60 seconds with error code
`TTL_EXCEEDS_MAXIMUM`.

**REQ-5.4.3:** A TTL exception allowing up to 300 seconds MAY be granted
for deployments with justified need (e.g., constrained-connectivity edge
deployments). Exceptions SHALL be recorded in the SESSION_INITIALIZED
Audit Event with `ttl_exception: true` and `ttl_exception_reason`. The
deployer MUST document the rationale in their risk register.

**REQ-5.4.4:** The ALS Service SHALL issue renewal credentials with the
same TTL as the initial credential, unless a QUERY command explicitly
updates the TTL.

**REQ-5.4.5:** The Governed Agent SHOULD initiate credential renewal when
fewer than TTL/2 seconds remain. The Governed Agent SHALL initiate
renewal no later than when 5 seconds remain.

### 5.5 Credential Renewal Protocol

**REQ-5.5.1:** The Governed Agent SHALL present its current, non-expired
Session Credential as the mTLS client certificate in a renewal request.
Renewal using an expired certificate SHALL be rejected with HTTP 401 and
error code `CREDENTIAL_EXPIRED`.

**REQ-5.5.2:** The renewal request SHALL be an HTTPS POST to the Renewal
Endpoint specified in the session initialization response.

**REQ-5.5.3:** The ALS Service SHALL check Session State before issuing
a renewal:
(a) NORMAL: issue new credential, return HTTP 200
(b) HALTED: return HTTP 403 with `SESSION_HALTED` and `status_endpoint`;
    produce RENEWAL_REFUSED_HALTED Audit Event
(c) TERMINATED: return HTTP 410 with `SESSION_TERMINATED`
(d) PAUSED (Tier 3): return HTTP 403 with `SESSION_PAUSED`

**REQ-5.5.4:** The ALS Service SHALL produce a CREDENTIAL_RENEWED Audit
Event for every successful renewal.

**REQ-5.5.5:** The ALS Service SHALL NOT accept renewal requests from any
agent not presenting the Session Credential for that specific Session.
Cross-session renewal requests SHALL be rejected.

### 5.6 Keypair Issuance Mode

**REQ-5.6.1:** In Keypair Issuance Mode, the ALS Service SHALL generate
the complete keypair and sign the public key in an X.509 certificate.

**REQ-5.6.2:** The ALS Service SHALL return the private key in the
`client_private_key` field (PEM, unencrypted).

**REQ-5.6.3:** The ALS Service SHALL NOT retain the private key after
returning it. The private key SHALL be deleted from ALS Service memory
within the same request handling context.

**REQ-5.6.4:** Keypair Issuance Mode is the default for Tier 1 and Tier
2 deployments. CSR Mode is required for Tier 2+ high-security
deployments.

### 5.7 CSR Mode

**REQ-5.7.1:** In CSR Mode, the Governed Agent SHALL generate its own
keypair before the initialization request and SHALL include a
PEM-encoded CSR in the `csr_pem` field.

**REQ-5.7.2:** The ALS Service SHALL validate the CSR: signature valid,
public key meets REQ-5.3.7 algorithm requirements, CSR Subject matches
agent identity in the initialization request.

**REQ-5.7.3:** The ALS Service SHALL sign the CSR's public key and SHALL
NOT include a private key in the initialization response.

**REQ-5.7.4:** In CSR Mode, the Governed Agent SHALL generate a new
keypair for each renewal and include a new CSR in the renewal request.

**REQ-5.7.5:** CSR Mode SHALL be used for all deployments where the ALS
Service is operated by a different organization from the Governed Agent.

### 5.8 Credential Expiry Behavior

**REQ-5.8.1:** A Governed Endpoint configured per Section 6 SHALL reject
any TLS ClientHello presenting an expired Session Credential. The
rejection SHALL occur at TLS handshake time, before any application data.

**REQ-5.8.2:** When a Session Credential expires with no renewal issued
(HALTED, TERMINATED, or ALS Service unavailable), all subsequent new
connection attempts SHALL fail at TLS handshake time. This is the
intended fail-closed enforcement behavior.

**REQ-5.8.3:** Credential expiry SHALL be determined by the `notAfter`
field of the X.509 certificate as validated by the standard TLS stack of
the Governed Endpoint. No ALS-specific validation logic is required for
expiry checking.

<br>

## SECTION 6: GOVERNED ENDPOINT CONFIGURATION
*Normative*

### 6.1 Tier 2 TLS Configuration Requirements

**REQ-6.1.1:** A Governed Endpoint claiming Tier 2 conformance SHALL
have the following TLS configuration applied:
(a) CA certificate trust list for client certificate validation SHALL
    include the ALS CA Certificate;
(b) Client certificate verification SHALL be set to REQUIRED.

**REQ-6.1.2:** These two configuration changes are the complete
server-side requirement for Tier 2 conformance. No application code
changes, SDK additions, or proxy deployment are required.

**REQ-6.1.3:** Reference configurations (illustrative; REQ-6.1.1 is
normative):

*Node.js HTTPS:*
```typescript
const server = https.createServer({
  cert: fs.readFileSync('./server.crt'),
  key:  fs.readFileSync('./server.key'),
  ca:   fs.readFileSync('./als-ca.crt'),
  requestCert: true,
  rejectUnauthorized: true
});
```

*Python (uvicorn):*
```python
uvicorn.run(app,
  ssl_certfile="./server.crt",
  ssl_keyfile="./server.key",
  ssl_ca_certs="./als-ca.crt",
  ssl_cert_reqs=ssl.CERT_REQUIRED
)
```

*A2A Agent (Python + httpx):*
```python
als_session = await als.initialize_session(
    commanders=[...],
    watchdog_config={...}
)
client = httpx.AsyncClient(
    verify="./als-ca.crt",
    cert=(als_session.client_cert, als_session.client_key)
)
a2a_client = A2AClient(http_client=client, base_url="https://agent-b.example.com")
```

### 6.2 Certificate Validation Behavior

**REQ-6.2.1:** The Governed Endpoint's TLS stack SHALL perform on every
client certificate: (a) chain validation to the ALS CA Certificate;
(b) expiry validation; (c) signature validation; (d) clientAuth Extended
Key Usage validation.

**REQ-6.2.2:** The Governed Endpoint SHALL NOT perform OCSP or CRL
checking for ALS Session Credentials. These mechanisms are replaced by
the TTL-based non-renewal model. Configuring OCSP/CRL for ALS Session
Credentials adds an unintended dependency that would create a fail-open
condition if the OCSP/CRL endpoint is unavailable.

**REQ-6.2.3:** The Governed Endpoint SHOULD extract and log the ALS
Session ID from the client certificate's Common Name or SAN URI for
correlation with ALS audit records.

**REQ-6.2.4:** TLS session ticket data, including any embedded ALS
context, SHALL NOT be used as authorization to bypass certificate
validation on session resumption. Certificate validation against the ALS
CA Certificate SHALL occur on every TLS connection, including resumed
sessions. Session ticket embedded ALS state is audit context only; it is
never enforcement authority.

### 6.3 Handshake Outcome Specifications

**REQ-6.3.1:** The following outcomes SHALL result from the specified
certificate conditions:

| Client Certificate Condition               | Outcome            | TLS Alert                  |
|--------------------------------------------|--------------------|----------------------------|
| Valid ALS credential (not expired, valid chain) | Accepted      | None                       |
| Expired ALS credential                     | Rejected           | certificate_expired (45)   |
| Certificate not signed by ALS CA           | Rejected           | unknown_ca (48)            |
| No client certificate presented            | Rejected           | certificate_required (116) |
| Revoked intermediate CA                    | Rejected           | unknown_ca (48)            |

**REQ-6.3.2:** The Governed Endpoint SHALL NOT attempt to infer the
reason for certificate invalidity beyond standard TLS validation. The
agent learns the reason through the Push Notification Channel or by
querying the `status_endpoint`.

### 6.4 Tier 2+ Per-Request Session Validation (Optional)

**REQ-6.4.1:** A Governed Endpoint implementing Tier 2+ per-request
validation SHALL, for each application-layer request on an ALS-governed
connection, send an authenticated HTTPS GET to the ALS Session
`status_endpoint` before processing.

**REQ-6.4.2:** If `status_endpoint` returns `session_state: NORMAL`,
process the request. Otherwise reject with HTTP 503 and SHOULD close the
connection.

**REQ-6.4.3:** The per-request validation request SHALL use the ALS CA
Certificate for server-side TLS validation.

**REQ-6.4.4:** Governed Endpoints implementing Tier 2+ SHALL implement
local caching of session state with a maximum cache TTL of 5 seconds.

**REQ-6.4.5:** Tier 2+ deployments SHALL ensure ALS Service availability
SLA equals or exceeds the application request availability SLA.

### 6.5 Server-Side Connection Lifetime Requirements

**REQ-6.5.1:** Governed Endpoints SHALL configure a maximum TLS
connection lifetime of no more than 300 seconds. Connections exceeding
this limit SHALL be gracefully closed using TLS `close_notify` or HTTP/2
`GOAWAY`, requiring the client to re-establish a new connection.

NOTE: Peak renewal load is approximately 2N/T requests per second where
N is the maximum expected concurrent Sessions and T is the TTL in
seconds. Use this formula to provision renewal infrastructure capacity.

**REQ-6.5.2:** This requirement mitigates the existing-connection gap:
HTTP/1.1 keep-alive and HTTP/2 multiplexed connections established before
a HALT Command may persist past credential expiry without a maximum
connection lifetime. The 300-second maximum bounds worst-case enforcement
for existing connections to TTL + 300 seconds.

**REQ-6.5.3:** Governed Endpoints SHOULD configure HTTP/2 `PING`
keepalive with a maximum idle timeout of 60 seconds.

### 6.6 Backward Compatibility with Non-ALS Clients

**REQ-6.6.1:** A Governed Endpoint configured for Tier 2 SHALL reject
all connections not presenting a valid ALS Session Credential.

**REQ-6.6.2:** Organizations transitioning to Tier 2 MUST verify, from
ALS Session records, that all legitimate connecting agents have active
ALS Sessions before enabling the Tier 2 server configuration.

**REQ-6.6.3:** Tier 1 and Tier 2 deployments are not mutually exclusive
and may operate in parallel during the transition period.

<br>

## SECTION 7: ALS CERTIFICATE AUTHORITY REQUIREMENTS
*Normative*

### 7.1 CA Hierarchy Requirements

**REQ-7.1.1:** The ALS CA infrastructure SHALL implement a two-tier or
three-tier CA hierarchy: (a) Root CA — offline, signs only intermediate
CA certificates; (b) Issuing Intermediate CA — online, signs Session
Credentials; (c) Optional additional intermediate CAs for regional or
organizational separation.

**REQ-7.1.2:** The Root CA private key SHALL be stored offline
(air-gapped). The Root CA SHALL NOT be used for any purpose other than
signing Intermediate CA certificates.

**REQ-7.1.3:** The Governed Endpoint CA trust anchor SHALL be the Issuing
Intermediate CA certificate, not the Root CA certificate.

### 7.2 Key Storage Requirements

**REQ-7.2.1:** All ALS CA private keys SHALL be stored in a Hardware
Security Module (HSM) or HSM-equivalent software implementation with
equivalent access control guarantees (e.g., HashiCorp Vault with a
hardware seal).

**REQ-7.2.2:** The HSM SHALL enforce: (a) multi-person authorization for
key generation ceremonies; (b) tamper-evident audit logging of all key
usage operations; (c) no key export in plaintext form.

**REQ-7.2.3:** Software-only key storage is permitted ONLY for local
development environments with explicit `development_mode: true`
configuration. Software-only key storage SHALL NOT be used in any
environment where ALS Sessions govern agents operating on behalf of users
or accessing non-development data.

### 7.3 Certificate Validity Period Constraints

**REQ-7.3.1:** ALS Session Credentials SHALL have a validity period of
TTL seconds per Section 5.4.

**REQ-7.3.2:** The Issuing Intermediate CA certificate SHALL have a
maximum validity period of 730 days.

**REQ-7.3.3:** The Root CA certificate SHALL have a maximum validity
period of 3650 days.

**REQ-7.3.4:** All ALS CA certificates SHALL use SHA-256 or stronger.
MD5 and SHA-1 SHALL NOT be used.

### 7.4 Key Ceremony Requirements

**REQ-7.4.1:** Root CA key generation SHALL be performed in a documented
key ceremony with a minimum of three key holders present, documenting:
date, location, attendees, hardware used, commands executed, and
generated certificate fingerprints.

**REQ-7.4.2:** Key ceremony documentation SHALL be retained for the
lifetime of the Root CA certificate plus 7 years.

**REQ-7.4.3:** Intermediate CA key generation MAY be performed without a
full ceremony but SHALL be performed in the HSM and SHALL be documented
with the same fields.

### 7.5 CA Key Rotation Procedures

**REQ-7.5.1:** The ALS CA SHALL have a documented key rotation procedure
executable without interrupting active ALS Sessions, defining: advance
notice period (minimum 30 days), dual-root trust configuration during
transition, credential re-issuance timeline, and old CA retirement.

**REQ-7.5.2:** In the event of CA key compromise, the ALS Service SHALL
have a documented emergency rotation procedure executable within 4 hours.
Emergency procedures SHALL be tested at least annually.

**REQ-7.5.3:** Following CA key rotation, all active Sessions SHALL have
credentials reissued using the new CA within one TTL window.

### 7.6 CA Availability Requirements

**REQ-7.6.1:** The Issuing Intermediate CA and the Renewal Endpoint SHALL
be deployed with sufficient redundancy to achieve the availability
required by the deployed TTL.

**REQ-7.6.2:** The ALS Service SHALL implement health monitoring for the
Renewal Endpoint and SHALL alert on elevated error rates or latency
degradation before widespread credential expiry.

**REQ-7.6.3:** A Renewal Endpoint unavailability does not require
operator intervention to enforce fail-closed behavior — credentials expire
automatically. The ALS Service SHALL produce a
`RENEWAL_ENDPOINT_DEGRADED` alert when the renewal endpoint error rate
exceeds 1% of requests within any 60-second window.

### 7.7 SPIFFE Compatibility Requirements

**REQ-7.7.1:** ALS Session Credentials SHALL be compatible with SPIFFE
specification v1.0: SAN URI SHALL carry a valid SPIFFE ID, and the
certificate SHALL satisfy SPIFFE X.509-SVID requirements.

**REQ-7.7.2:** ALS deployments MAY use SPIRE as the ALS CA
implementation. SPIRE-based implementations SHALL extend the SPIRE SVID
with the ALS Session ID in the additional SAN URI field per REQ-5.3.2(b).

<br>

## SECTION 8: CONTROL CHANNEL PROTOCOL
*Normative*

### 8.1 Control Channel Transport

**REQ-8.1.1:** The ALS Control Channel SHALL use gRPC over mTLS
(TLS 1.3 minimum).

**REQ-8.1.2:** The Control Channel SHALL use a dedicated network endpoint
separate from session initialization and renewal endpoints, and SHALL NOT
be accessible to Governed Agents.

**REQ-8.1.3:** The Control Channel SHALL use TLS 1.3 exclusively. TLS
1.2 SHALL NOT be accepted on the Control Channel.

**REQ-8.1.4:** The ALS Service SHALL maintain persistent gRPC connections
to each authorized AI Security Service with automatic reconnection using
exponential backoff (initial delay 1s, maximum delay 30s).

### 8.2 Commander Authentication and Provisioning

**REQ-8.2.1:** AI Security Services SHALL authenticate to the Control
Channel by presenting their Commander Certificate as the mTLS client
certificate. The ALS Service SHALL validate against the pre-provisioned
commander trust store.

**REQ-8.2.2:** Commander Certificates SHALL be provisioned at ALS Service
deployment time. Commander Certificates SHALL NOT be provisioned
dynamically via any API accessible to Governed Agents.

**REQ-8.2.3:** Each Commander Certificate SHALL have an associated
permission scope defining which Sessions it may command.

**REQ-8.2.4:** The ALS Service SHALL reject any Control Channel
connection not presenting a valid, non-expired Commander Certificate from
the trust store and SHALL log a `CONTROL_CHANNEL_AUTH_FAILED` Audit
Event.

### 8.3 Command Schema

All commands are JSON objects. Each command includes common fields plus
type-specific fields.

**REQ-8.3.1:** Common fields (all commands):

| Field          | Type   | Required | Description                                          |
|----------------|--------|----------|------------------------------------------------------|
| `command_id`   | string | MUST     | UUID v4                                              |
| `command_type` | string | MUST     | HALT, PAUSE, RESUME, TERMINATE, or QUERY             |
| `session_id`   | string | MUST     | Target ALS Session ID                                |
| `commander_id` | string | MUST     | SPIFFE ID or fingerprint of issuing Commander        |
| `issued_at`    | string | MUST     | ISO 8601 UTC timestamp                               |
| `nonce`        | string | MUST     | Cryptographically random, min 128 bits, base64url    |
| `signature`    | string | MUST     | Commander signature over (command_id + session_id +  |
|                |        |          | issued_at + nonce), base64url                        |

**REQ-8.3.2:** HALT Command additional fields:

| Field            | Type    | Required | Description                          |
|------------------|---------|----------|--------------------------------------|
| `halt_reason`    | string  | MUST     | Human-readable reason                |
| `halt_severity`  | string  | MUST     | advisory, operational, or security   |
| `resume_possible`| boolean | MUST     | Whether Session may be resumed       |

**REQ-8.3.3:** PAUSE Command additional fields (Tier 3):

| Field                  | Type    | Required | Description                       |
|------------------------|---------|----------|-----------------------------------|
| `pause_reason`         | string  | MUST     | Human-readable reason             |
| `hold_timeout_seconds` | integer | MUST     | Maximum hold (1–3600)             |
| `resume_notification`  | boolean | MUST     | Notify agent on resume            |

**REQ-8.3.4:** RESUME Command additional fields:

| Field           | Type   | Required | Description                              |
|-----------------|--------|----------|------------------------------------------|
| `resume_reason` | string | MUST     | Human-readable reason                    |
| `reviewed_by`   | string | MUST     | Identity of authorizing human reviewer   |

**REQ-8.3.5:** TERMINATE Command additional fields:

| Field                | Type   | Required | Description                                          |
|----------------------|--------|----------|------------------------------------------------------|
| `termination_reason` | string | MUST     | Human-readable reason                                |
| `termination_class`  | string | MUST     | policy_violation, security_incident, decommission,   |
|                      |        |          | or test                                              |

**REQ-8.3.6:** QUERY Command requires only common fields. Returns current
Session State, Enforcement Tier, last renewal timestamp, and last command
received.

### 8.4 Response Schema and Acknowledgment

**REQ-8.4.1:** The ALS Service SHALL return a response to every command
within 5 seconds. Commands exceeding 5 seconds SHALL return
`PROCESSING_TIMEOUT`.

**REQ-8.4.2:** Command responses SHALL include:

| Field           | Type   | Description                                         |
|-----------------|--------|-----------------------------------------------------|
| `command_id`    | string | Echo of command_id                                  |
| `result`        | string | SUCCESS, REJECTED, or ERROR                         |
| `session_state` | string | Current Session State after execution               |
| `error_code`    | string | Present if REJECTED or ERROR                        |
| `error_message` | string | Human-readable error description                    |
| `executed_at`   | string | ISO 8601 UTC timestamp of execution                 |

**REQ-8.4.3:** SUCCESS = command executed and state transition occurred.
REJECTED = valid command not executed (e.g., RESUME on TERMINATED).
ERROR = processing error.

### 8.5 Replay Prevention

**REQ-8.5.1:** The ALS Service SHALL maintain a record of processed
command nonces with a 300-second Replay Window. Commands with a matching
nonce within the window SHALL be rejected with `REPLAY_DETECTED`.

**REQ-8.5.2:** The ALS Service SHALL reject commands where `issued_at`
is more than 60 seconds in the past or more than 30 seconds in the
future. Rejections produce a `COMMAND_TIMESTAMP_INVALID` Audit Event.

**REQ-8.5.3:** The Commander's signature SHALL be validated against the
Commander Certificate before any command is processed. Invalid signatures
SHALL be rejected with `INVALID_SIGNATURE`.

### 8.6 Multi-Commander Arbitration

**REQ-8.6.1:** Any Commander with authorization scope for a Session MAY
issue commands without required consensus from other Commanders.

**REQ-8.6.2:** When two Commanders issue conflicting commands, the ALS
Service SHALL apply the more restrictive command: HALT takes precedence
over RESUME; TERMINATE takes precedence over both.

**REQ-8.6.3:** When conflicting commands are received, the ALS Service
SHALL produce a `CONFLICTING_COMMANDS_RECEIVED` Audit Event listing both
command IDs, Commander identities, and resolution.

**REQ-8.6.4:** The ALS Service SHOULD notify all Session Commanders when
a state change is triggered by any Commander.

### 8.7 Control Channel Availability and Partition Behavior

**REQ-8.7.1:** If the Control Channel connection is lost, the ALS Service
SHALL continue operating normally: existing Sessions retain current
state, credential renewals continue for NORMAL Sessions, and the watchdog
continues operating.

**REQ-8.7.2:** The ALS Service SHALL NOT automatically halt all Sessions
when the Control Channel is unavailable. Fail-closed is provided by
TTL-based credential expiry, not by active response to control channel
loss.

**REQ-8.7.3:** The ALS Service SHALL produce `CONTROL_CHANNEL_DISCONNECTED`
on Commander connection loss and `CONTROL_CHANNEL_RECONNECTED` on
reestablishment.

**REQ-8.7.4:** The ALS Service SHALL maintain a configurable grace period
(default: 5 minutes, maximum: 30 minutes) before producing a
`CONTROL_CHANNEL_EXTENDED_ABSENCE` alert.

### 8.8 Command Rate Limiting and DoS Protection

**REQ-8.8.1:** The ALS Service SHALL implement per-Commander rate
limiting. Default: 100 commands per minute per Commander.

**REQ-8.8.2:** Commands exceeding the rate limit SHALL be rejected with
`RATE_LIMIT_EXCEEDED` and logged.

**REQ-8.8.3:** The ALS Service SHALL implement connection-level rate
limiting: no more than 10 connection attempts per minute from a single IP
address.

<br>

## SECTION 9: WATCHDOG MECHANISM
*Normative*

### 9.1 Watchdog Parameters and Constraints

**REQ-9.1.1:** The ALS Service SHALL implement a watchdog for every
active ALS Session in NORMAL state.

**REQ-9.1.2:** The Watchdog Window SHALL be:
`watchdog_window_seconds = TTL × watchdog_multiplier`
Default multiplier: 2.0. Maximum: 3.0. Minimum accepted: 1.5.

**REQ-9.1.3:** The `watchdog_multiplier` MAY be specified by the Governed
Agent at session initialization. If unspecified, the ALS Service default
(2.0) SHALL be used.

### 9.2 Watchdog Trigger Conditions

**REQ-9.2.1:** The watchdog SHALL trigger if no successful credential
renewal has been received within the Watchdog Window since the last
successful renewal (or since session initialization).

**REQ-9.2.2:** A rejected renewal attempt SHALL NOT reset the watchdog
timer. Only a successful renewal (resulting in a new credential issued)
SHALL reset the watchdog timer.

**REQ-9.2.3:** The watchdog SHALL be reset on a successful RESUME
following a watchdog-triggered HALT.

### 9.3 Watchdog-Triggered HALT Behavior

**REQ-9.3.1:** When the watchdog triggers, the ALS Service SHALL
immediately transition the Session to HALTED and SHALL produce a
`SESSION_WATCHDOG_HALT` Audit Event containing: Session ID, agent
identity, last successful renewal timestamp, watchdog window, and trigger
timestamp.

**REQ-9.3.2:** A watchdog-triggered HALT SHALL be transmitted to all
active Commanders via the Control Channel as an unsolicited
`SESSION_STATE_CHANGE` notification.

**REQ-9.3.3:** A watchdog-triggered HALT SHALL attempt push notification
to the Governed Agent with:
`halt_reason: "Watchdog timeout — session did not renew within window"`
`halt_severity: "operational"`

### 9.4 Watchdog Configuration Restrictions

**REQ-9.4.1:** The watchdog SHALL NOT be disabled in any deployment
configuration. There is no configuration option to disable the watchdog.

**REQ-9.4.2:** The watchdog multiplier SHALL NOT be configurable above
3.0. ALS Service implementations SHALL enforce this maximum.

**REQ-9.4.3:** Production deployment configurations SHALL NOT accept a
`watchdog_multiplier` greater than 3.0.

### 9.5 Watchdog vs. Commanded HALT Distinction

**REQ-9.5.1:** The ALS Service SHALL distinguish watchdog-triggered HALTs
from Commander-triggered HALTs in all audit records, session state query
responses, and push notifications. The Session State query response SHALL
include `halt_source` ("watchdog" or "commander") and
`halt_command_id` (present only when `halt_source = "commander"`).

**REQ-9.5.2:** Both watchdog-triggered and Commander-triggered HALTs
require a RESUME Command from an authorized Commander to return to
NORMAL. There is no automatic recovery from either type of HALT.

<br>

## SECTION 10: PUSH NOTIFICATION PROTOCOL
*Normative*

### 10.1 Push Channel Transport and Authentication

**REQ-10.1.1:** The Push Notification Channel SHALL use WebSocket
(RFC 6455) over TLS 1.3.

**REQ-10.1.2:** The Governed Agent SHALL authenticate to the Push
Notification Channel using its current Session Credential as the mTLS
client certificate.

**REQ-10.1.3:** The Governed Agent SHOULD establish the Push Notification
Channel immediately after receiving the session initialization response.

**REQ-10.1.4:** The Push Notification Channel SHALL use TLS 1.3
exclusively. No plaintext WebSocket connections are permitted.

### 10.2 Notification Message Schema

**REQ-10.2.1:** ALS Push Notifications SHALL be JSON objects:

```json
{
  "notification_id": "<UUID v4>",
  "session_id": "<ALS Session ID>",
  "notification_type": "SESSION_STATE_CHANGE",
  "previous_state": "NORMAL",
  "new_state": "HALTED",
  "effective_in_seconds": 45,
  "halt_reason": "Anomalous egress detected — pending security review",
  "halt_severity": "security",
  "resume_possible": true,
  "status_endpoint": "https://als.internal:7400/session/{id}/status",
  "issued_at": "<ISO 8601 UTC>",
  "signature": "<ALS Service signature over notification, base64url>"
}
```

**REQ-10.2.2:** The `effective_in_seconds` field SHALL be the number of
seconds remaining on the Governed Agent's current Session Credential at
the time the notification is sent.

**REQ-10.2.3:** The `notification_type` field SHALL be one of:
`SESSION_STATE_CHANGE`, `SESSION_CREDENTIAL_EXPIRING` (advisory, when
TTL/4 remains), `SESSION_RESUMED`, `SESSION_TERMINATED`.

**REQ-10.2.4:** Push notifications SHALL be signed by the ALS Service.
The Governed Agent MUST verify this signature to prevent spoofed halt
notifications.

### 10.3 Acknowledgment Protocol

**REQ-10.3.1:** Upon receiving a push notification, the Governed Agent
SHALL send an acknowledgment within 10 seconds, including:
`notification_id`, `acknowledged_at`, and `agent_session_id`.

**REQ-10.3.2:** The ALS Service SHALL record whether each notification was
acknowledged, the acknowledgment timestamp, and the delivery-to-ack
delta. This information SHALL be included in the SESSION_HALTED Audit
Event.

**REQ-10.3.3:** If a HALT notification is not acknowledged within 10
seconds, the ALS Service SHALL produce a `HALT_NOTIFICATION_UNACKNOWLEDGED`
Audit Event. Enforcement proceeds regardless.

### 10.4 Enforcement Independence from Push Delivery

**REQ-10.4.1:** Enforcement of a HALT Command SHALL proceed regardless
of whether the Push Notification Channel is active, the notification was
delivered, or the notification was acknowledged.

**REQ-10.4.2:** The ALS Service SHALL NOT delay execution of a HALT
Command pending push notification delivery. The HALT state transition
SHALL occur upon validated receipt of the HALT Command.

### 10.5 Agent Normative Behavior on Notification Receipt

**REQ-10.5.1:** Upon receiving a `SESSION_STATE_CHANGE` notification with
`new_state: "HALTED"` or `new_state: "TERMINATED"`, a conformant
Governed Agent MUST:
(a) Immediately cease initiating new MCP tool calls or A2A task
    delegations;
(b) Send the acknowledgment per REQ-10.3.1;
(c) Log the notification at WARN level or higher.

**REQ-10.5.2:** Upon receiving `new_state: "HALTED"`, a conformant
Governed Agent SHOULD: (a) within `effective_in_seconds`, complete or
cease sending the current in-progress MCP tool call request or A2A task
delegation message; (b) surface the `halt_reason` to operator-facing
interfaces; (c) enter a waiting state, polling `status_endpoint` until
`new_state: "NORMAL"` is received.

**REQ-10.5.3:** A conformant Governed Agent MUST NOT, upon receiving a
halt notification: (a) attempt to renew its Session Credential ahead of
schedule solely to extend the enforcement window; (b) attempt to
establish connections to endpoints not governed by ALS to continue
operations; (c) suppress, modify, or delay logging of the halt
notification.

NOTE: Requirement (b) applies to the agent's behavior in response to a
halt notification received via the ALS Push Notification Channel. It is
a transport-layer governance requirement — the agent must not route around
ALS enforcement. It does not constitute agent behavioral policy beyond
the governance context.

### 10.6 Push Channel Reconnection Behavior

**REQ-10.6.1:** The Governed Agent SHALL implement automatic reconnection
with exponential backoff (initial 1s, maximum 30s, jitter applied).

**REQ-10.6.2:** Upon reconnection, the Governed Agent SHALL request
missed notifications since the last received `notification_id` by
including `last_notification_id` in the WebSocket handshake query string.

**REQ-10.6.3:** The ALS Service SHALL retain push notifications for
delivery for a minimum of 5 minutes after issuance.

<br>

## SECTION 11: TLS EXTENSIONS
*Normative*

### 11.1 `als_session_id` ClientHello Extension

**REQ-11.1.1:** ALS defines a TLS ClientHello extension carrying the ALS
Session ID, enabling Governed Endpoints to correlate incoming TLS
connections to ALS Sessions without content inspection.

**REQ-11.1.2:** The extension type code SHALL be `0xFF10` for development
and private-use deployments, pending IANA registration per REQ-11.4.2.

**REQ-11.1.3:** The extension data SHALL be the ALS Session ID as a
UTF-8 string, maximum 256 bytes.

**REQ-11.1.4:** ALS-aware Governed Endpoints SHOULD read the
`als_session_id` extension for logging purposes. This does not affect
the TLS handshake outcome.

**REQ-11.1.5:** TLS stacks not recognizing the `als_session_id`
extension SHALL ignore it per RFC 8446 §4.2. Its presence SHALL NOT
cause handshake failures.

**REQ-11.1.6:** Governed Agents SHOULD include the `als_session_id`
extension in every TLS ClientHello to Governed Endpoints. Agents MAY
omit it if their TLS stack does not support custom extensions.

### 11.2 `als/1.0` ALPN Token

**REQ-11.2.1:** ALS defines the ALPN token `als/1.0` (ASCII, 6 bytes)
for capability advertisement and Tier 3 protocol negotiation.

**REQ-11.2.2:** A Governed Agent SHOULD include `als/1.0` in its ALPN
extension list alongside the application protocol identifier. Recommended
order: `["als/1.0", "h2", "http/1.1"]`.

**REQ-11.2.3:** A Governed Endpoint that supports Tier 3 SHOULD include
`als/1.0` in its supported ALPN list. When `als/1.0` is negotiated, the
connection SHALL use the ALS Framing Layer per Section 13.

**REQ-11.2.4:** A Governed Endpoint that does not support Tier 3 SHALL
NOT include `als/1.0` in its ALPN list. ALPN negotiation will proceed to
the next listed protocol. This is the correct graceful degradation
behavior.

**REQ-11.2.5:** Until IANA registration is complete, `x-als/1.0` SHALL
be used to indicate private/pre-standard usage.

### 11.3 Extension Handling by Non-ALS Endpoints

**REQ-11.3.1:** ALS TLS extensions SHALL be designed so non-ALS-aware
endpoints silently ignore them per standard TLS extension handling rules.

**REQ-11.3.2:** An ALS-aware agent connecting to a non-ALS-aware server
SHALL NOT fail the connection because the server does not recognize ALS
extensions.

### 11.4 IANA Registration Requirements

**REQ-11.4.1:** Before ALS v1.0 is declared a final standard, the
`als/1.0` ALPN token SHALL be submitted for IANA registration per
RFC 7301 §6.

**REQ-11.4.2:** Before ALS v1.0 is declared a final standard, the
`als_session_id` TLS extension type code SHALL be submitted for IANA
registration per RFC 8447.

<br>

## SECTION 12: AUDIT REQUIREMENTS
*Normative*

### 12.1 Mandatory Audit Events

**REQ-12.1.1:** The ALS Service SHALL produce an Audit Event for every
event below. Each event SHALL include all common fields plus the
specified event-specific fields.

**Common fields (ALL events):**

| Field             | Type    | Description                                          |
|-------------------|---------|------------------------------------------------------|
| `event_id`        | string  | UUID v4                                              |
| `event_type`      | string  | Event type identifier                                |
| `session_id`      | string  | ALS Session ID                                       |
| `als_service_id`  | string  | ALS Service instance identifier                      |
| `timestamp`       | string  | ISO 8601 UTC, millisecond precision                  |
| `sequence_number` | integer | Monotonically increasing per-session counter         |

**Event-specific fields:**

| Event Type                          | Additional Required Fields                                        |
|-------------------------------------|-------------------------------------------------------------------|
| `SESSION_INIT_REQUESTED`            | agent_identity, requested_commanders, requested_ttl, source_ip   |
| `SESSION_INITIALIZED`               | agent_identity, commanders, actual_ttl, enforcement_tier,        |
|                                     | credential_fingerprint, ttl_exception (bool),                    |
|                                     | ttl_exception_reason (if applicable)                             |
| `SESSION_INIT_FAILED`               | agent_identity, failure_reason, error_code                       |
| `CREDENTIAL_ISSUED`                 | credential_fingerprint, valid_from, valid_until, issuance_mode   |
| `CREDENTIAL_RENEWED`                | previous_fingerprint, new_fingerprint, renewed_at,               |
|                                     | ttl_remaining_at_renewal                                         |
| `RENEWAL_REFUSED_HALTED`            | credential_fingerprint_presented, session_state_at_refusal       |
| `SESSION_HALTED`                    | halt_source ("commander"), commander_id, command_id, halt_reason,|
|                                     | halt_severity, resume_possible, notification_sent (bool),        |
|                                     | notification_acknowledged (bool)                                 |
| `SESSION_WATCHDOG_HALT`             | last_renewal_timestamp, watchdog_window_seconds,                 |
|                                     | watchdog_multiplier                                              |
| `SESSION_PAUSED`                    | commander_id, command_id, pause_reason, hold_timeout_seconds     |
| `SESSION_RESUMED`                   | previous_state, commander_id, command_id, resume_reason,         |
|                                     | reviewed_by                                                      |
| `SESSION_TERMINATED`                | commander_id, command_id, termination_reason,                    |
|                                     | termination_class, session_duration_seconds                      |
| `HALT_NOTIFICATION_SENT`            | notification_id, effective_in_seconds, push_channel_active       |
| `HALT_NOTIFICATION_ACKNOWLEDGED`    | notification_id, acknowledgment_latency_ms                       |
| `HALT_NOTIFICATION_UNACKNOWLEDGED`  | notification_id, timeout_seconds                                 |
| `COMMAND_RECEIVED`                  | command_id, command_type, commander_id, command_result           |
| `CONTROL_CHANNEL_AUTH_FAILED`       | source_ip, presented_certificate_fingerprint, failure_reason     |
| `CONFLICTING_COMMANDS_RECEIVED`     | command_id_1, commander_id_1, command_type_1, command_id_2,      |
|                                     | commander_id_2, command_type_2, resolution                       |
| `CONTROL_CHANNEL_DISCONNECTED`      | commander_id, disconnect_reason                                  |
| `CONTROL_CHANNEL_RECONNECTED`       | commander_id, disconnected_at, reconnected_at                    |
| `REPLAY_DETECTED`                   | command_id, commander_id, original_nonce                         |
| `RATE_LIMIT_EXCEEDED`               | commander_id, commands_in_window, window_seconds                 |
| `RENEWAL_ENDPOINT_DEGRADED`         | error_rate_percent, window_seconds, error_count                  |
| `CONTROL_CHANNEL_EXTENDED_ABSENCE`  | commander_id, absence_duration_seconds                           |

### 12.2 Audit Log Tamper-Evidence Requirements

**REQ-12.2.1:** The ALS audit log SHALL be tamper-evident via one of:
(a) Hash-chain: each event contains SHA-256 hash of the previous event;
(b) Append-only storage: write-once storage system;
(c) Log signing: each event signed by ALS Service signing key within 5s.

**REQ-12.2.2:** The ALS Service SHALL NOT provide any API by which a
Governed Agent can read, modify, or delete Audit Events.

**REQ-12.2.3:** Audit Events SHALL be written to persistent storage
before the operation they record is considered complete. The
SESSION_HALTED event SHALL be written before the halt command response
is returned to the AI Security Service.

### 12.3 Audit Log Independence Requirements

**REQ-12.3.1:** The ALS audit log SHALL be stored in infrastructure not
accessible to Governed Agent processes, requiring separate authentication
credentials from the ALS Service itself.

**REQ-12.3.2:** The ALS audit log SHALL remain readable even if all
Governed Agents and Governed Endpoints are offline or compromised.

### 12.4 Mandatory Alert Conditions

**REQ-12.4.1:** The ALS Service SHALL generate an alert for the following
conditions:

| Condition                                              | Severity | Trigger              |
|--------------------------------------------------------|----------|----------------------|
| HALT_NOTIFICATION_UNACKNOWLEDGED (security severity)  | HIGH     | Immediately          |
| RENEWAL_ENDPOINT_DEGRADED (>1% error rate in 60s)     | HIGH     | Within 60 seconds    |
| CONTROL_CHANNEL_EXTENDED_ABSENCE                       | HIGH     | At grace period      |
| CONTROL_CHANNEL_AUTH_FAILED (>3 in 60s, same source)  | HIGH     | On 3rd failure       |
| REPLAY_DETECTED                                        | HIGH     | Immediately          |
| Session HALTED with no Commander connected >5 min      | MEDIUM   | After 5 minutes      |
| SESSION_WATCHDOG_HALT for security-class agent         | HIGH     | Immediately          |
| Audit log write failure                                | CRITICAL | Immediately          |

**REQ-12.4.2:** The ALS Service SHALL require at least one alert
destination to be configured before accepting session initialization
requests in non-development-mode deployments. An ALS Service with no
configured alert destination SHALL refuse session initialization with
error code `ALERT_DESTINATION_NOT_CONFIGURED`. The ALS Service SHALL
support at minimum one of: webhook, syslog, or SMTP alert delivery.

### 12.5 Audit Retention Requirements

**REQ-12.5.1:** Audit Events for Sessions involving EU AI Act high-risk
AI systems SHALL be retained for a minimum of 10 years from Session
termination.

**REQ-12.5.2:** Audit Events for all other Sessions SHALL be retained for
a minimum of 3 years from Session termination.

**REQ-12.5.3:** Deployers SHALL retain Audit Events for the longer of the
ALS minimum or any applicable regulatory requirement.

### 12.6 Audit Event Schema

**REQ-12.6.1:** All Audit Events SHALL be serialized as JSON objects
conforming to the schema in Appendix A.

**REQ-12.6.2:** Audit Events SHOULD be formatted for compatibility with
the OpenTelemetry Log Data Model (OTLP).

<br>

## SECTION 13: TIER 3 — ALS FRAMING LAYER (OPTIONAL EXTENSION)
*Normative for Tier 3; Informative for Tier 1/2*

Requirements prefixed `TIER-3-MUST` are mandatory only for Tier 3
conformance claims. Tier 1/2 implementations may safely ignore this
section.

### 13.1 Tier 3 Prerequisites and Negotiation

**TIER-3-MUST-13.1.1:** Tier 3 SHALL be negotiated via ALPN. Both
Governed Agent and Governed Endpoint MUST include `als/1.0` in their
ALPN lists. Tier 3 is active only when `als/1.0` is the negotiated ALPN
value.

**TIER-3-MUST-13.1.2:** When `als/1.0` is negotiated, the Governed
Endpoint SHALL send an ALS Framing Layer handshake frame before any
application protocol data.

**TIER-3-MUST-13.1.3:** If ALPN negotiation results in any value other
than `als/1.0`, the connection SHALL proceed as a standard Tier 2
connection without the ALS Framing Layer.

### 13.2 ALS Framing Layer Frame Format

**TIER-3-MUST-13.2.1:** ALS Framing Layer frames SHALL use:

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|  Frame Type   |   Reserved    |         Payload Length        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                       Payload (variable, JSON)                |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

Frame Types: `0x01` ALS_HANDSHAKE | `0x02` ALS_SESSION_PAUSE |
`0x03` ALS_SESSION_RESUME | `0x04` ALS_APPLICATION_DATA |
`0x05` ALS_KEEPALIVE

**TIER-3-MUST-13.2.2:** All frame payloads SHALL be JSON-encoded.

### 13.3 `ALS_SESSION_PAUSE` Frame

**TIER-3-MUST-13.3.1:** When ALS commands PAUSE, the Governed Endpoint
SHALL send an `ALS_SESSION_PAUSE` frame before suspending application
data:

```json
{
  "session_id": "<ALS Session ID>",
  "state": "PAUSED",
  "reason": "<pause_reason from PAUSE Command>",
  "hold_timeout_seconds": 300,
  "resume_notification": true,
  "paused_at": "<ISO 8601 UTC>",
  "signature": "<ALS Service signature, base64url>"
}
```

**TIER-3-MUST-13.3.2:** After sending the pause frame, the Governed
Endpoint SHALL hold the TLS connection open but SHALL NOT process
application data frames until `ALS_SESSION_RESUME` is sent.

**TIER-3-MUST-13.3.3:** After receiving the pause frame, the Governed
Agent SHALL cease sending application data frames and SHALL NOT close the
TLS connection.

### 13.4 `ALS_SESSION_RESUME` Frame

**TIER-3-MUST-13.4.1:** When ALS commands RESUME for a PAUSED Session,
the Governed Endpoint SHALL send:

```json
{
  "session_id": "<ALS Session ID>",
  "state": "NORMAL",
  "resumed_at": "<ISO 8601 UTC>",
  "pause_duration_seconds": 47,
  "signature": "<ALS Service signature, base64url>"
}
```

**TIER-3-MUST-13.4.2:** After sending the resume frame, the Governed
Endpoint SHALL immediately resume processing application data frames.

### 13.5 Hold Timeout and Connection Lifecycle

**TIER-3-MUST-13.5.1:** If RESUME is not received within
`hold_timeout_seconds`, the Governed Endpoint SHALL transition to HALTED
and close the TLS connection. This produces a SESSION_HALTED Audit Event
with `halt_source: "pause_timeout"`.

**TIER-3-MUST-13.5.2:** The maximum `hold_timeout_seconds` is 3600. PAUSE
Commands with `hold_timeout_seconds` greater than 3600 SHALL be rejected.

**TIER-3-MUST-13.5.3:** During PAUSE, the ALS Service SHALL continue
issuing Session Credential renewals. The Governed Agent SHOULD continue
renewing its credential during PAUSE.

### 13.6 Sub-Protocol Negotiation

**TIER-3-MUST-13.6.1:** The `ALS_HANDSHAKE` frame SHALL carry the
sub-protocol identifier for the application data that follows (e.g.,
`"mcp/streamable-http"`, `"a2a/json-rpc-2.0"`, `"grpc/a2a"`). Both sides
SHALL agree on the sub-protocol before application data flows.

**TIER-3-MUST-13.6.2:** `ALS_APPLICATION_DATA` frames SHALL wrap all
application protocol data when the ALS Framing Layer is active.

### 13.7 Tier 3 Conformance Requirements

**TIER-3-MUST-13.7.1:** An implementation claiming Tier 3 conformance
MUST implement all `TIER-3-MUST` requirements in Section 13.

**TIER-3-MUST-13.7.2:** An implementation claiming Tier 3 MUST also
satisfy all Tier 2 requirements.

**TIER-3-MUST-13.7.3:** The ALS Framing Layer MUST degrade gracefully to
Tier 2 when the peer does not support `als/1.0` ALPN. No application
data loss is acceptable due to ALPN negotiation failure.

<br>

## SECTION 14: THREAT MODEL AND SECURITY CONSIDERATIONS
*Normative*

NOTE: References in this section to Layer 7 detection tools and AI
Security Services describe complementary detection systems that operate
independently of ALS and feed halt commands to ALS via the Control
Channel. ALS itself performs no application-layer content inspection.
See DP-1 and DP-9.

### 14.1 Threat Taxonomy

This section identifies all known threats to ALS enforcement, the
mitigation requirement for each, and residual risks where full mitigation
is not possible.

### 14.2 Control Channel Threats

**THREAT CC-1: Commander Impersonation**
An attacker forges a Commander Certificate to issue unauthorized commands.
*Mitigation:* REQ-8.2.1, REQ-8.2.2, REQ-7.2.1, REQ-8.3.1.
*Residual:* CA key compromise enables impersonation. Mitigated by
REQ-7.2.1 (HSM) and REQ-7.4.1 (key ceremony).

**THREAT CC-2: Command Replay**
A recorded valid HALT command is replayed after Session resumption.
*Mitigation:* REQ-8.5.1 (nonce Replay Window), REQ-8.5.2 (timestamp
validation).

**THREAT CC-3: Denial of Service against Control Channel**
An attacker floods the Control Channel, delaying legitimate halt commands.
*Mitigation:* REQ-8.8.1 (per-Commander rate limiting), REQ-8.8.3
(connection-level rate limiting), REQ-8.1.2 (dedicated endpoint).
*Residual:* Volumetric DDoS requires network-level protection outside
ALS scope.

**THREAT CC-4: Compromised AI Security Service**
An authorized AI Security Service issues incorrect halt or resume
commands.
*Mitigation:* REQ-8.6.2 (most-restrictive wins), REQ-12.1.1 (all
commands audited), REQ-8.2.3 (Commander permission scoping).
*Residual:* Compromised Commander with global scope requires human audit
review as the detective control.

### 14.3 Credential Threats

**THREAT CR-1: ALS CA Private Key Compromise**
An attacker obtains the CA private key and issues fraudulent credentials.
*Mitigation:* REQ-7.2.1 (HSM), REQ-7.4.1 (key ceremony), REQ-7.5.2
(emergency rotation within 4 hours), REQ-7.1.1 (CA hierarchy).

**THREAT CR-2: Session Credential Theft**
An attacker intercepts a Session Credential for use during its TTL.
*Mitigation:* REQ-5.4.2 (maximum 60-second TTL), REQ-5.7.5 (CSR Mode for
third-party ALS deployments).

**THREAT CR-3: Credential Forgery Without CA Compromise**
An attacker creates a fraudulent credential without CA key access.
*Mitigation:* REQ-6.2.1 (chain validation), REQ-5.3.7 (minimum key
strength). Standard X.509 guarantees prevent forgery.

**THREAT CR-4: Stale Session Ticket State**
A Governed Endpoint uses stale ALS context from TLS session tickets to
bypass certificate validation on resumption. Additionally, configuring
OCSP/CRL for ALS Session Credentials would create an unintended
dependency causing fail-open if the OCSP/CRL endpoint is unavailable.
*Mitigation:* REQ-6.2.1 (cert validation on every connection including
resumed), REQ-6.2.2 (OCSP/CRL explicitly prohibited), REQ-6.2.4 (session
ticket state is audit context only, never enforcement authority).

### 14.4 Data Plane Bypass Threats

**THREAT BP-1: Alternative Network Paths**
A compromised agent routes connections through paths not covered by ALS.
*Mitigation:* Network policy MUST restrict agent outbound connections to
ALS-governed endpoints. This is a required complementary control.
*Residual:* Absent or misconfigured network egress policy leaves
alternative paths open.

**THREAT BP-2: Existing Connection Persistence**
An agent with established HTTP/1.1 or HTTP/2 connections continues using
them past credential expiry.
*Mitigation:* REQ-6.5.1 (maximum connection lifetime 300 seconds),
REQ-6.4.1 (Tier 2+ per-request validation closes the gap).
*Residual:* Without Tier 2+, gap is bounded by
TTL + max_connection_lifetime ≤ 360 seconds. Accepted risk for Tier 2.

**THREAT BP-3: Protocol Substitution**
An agent uses a different ALPN or no ALPN to bypass ALS identification.
*Mitigation:* REQ-6.1.1 applies to ALL TLS connections to a Governed
Endpoint regardless of ALPN.

**THREAT BP-4: QUIC/UDP Transport Bypass**
An agent uses QUIC (HTTP/3) to connect to an endpoint with a QUIC
listener, bypassing TCP+TLS governance.
*Mitigation:* ALS v1.0 does not govern QUIC (see Section 14.8). Required
complementary control: network policy MUST block UDP agent traffic.

**THREAT BP-5: Selective Endpoint Bypass**
A Governed Agent uses its ALS credential for ALS-configured endpoints but
also connects to non-ALS-configured endpoints for the same operations.
*Mitigation:* Network egress policy is the required complementary control.
Deployers MUST ensure agents can only reach MCP/A2A endpoints that are
ALS-configured.

### 14.5 Race Condition and Partial Enforcement Threats

**THREAT RC-1: Distributed Enforcement Window**
In multi-instance ALS deployments, halt command propagation creates
inconsistent enforcement across instances.
*Mitigation:* REQ-8.4.1 (5-second command response). Multi-instance
deployments MUST implement synchronous halt propagation to all instances
before returning SUCCESS.

**THREAT RC-2: In-Flight Operations at Halt Time**
An agent dispatches an operation the instant before halt takes effect.
*Mitigation:* REQ-10.2.2 (`effective_in_seconds`), REQ-10.5.2 (agent
SHOULD cease in-progress transmissions within window).
*Residual:* Operations dispatched with zero time before enforcement
cannot be recalled.

### 14.6 Availability Attack Threats

**THREAT AV-1: Renewal Endpoint DoS**
An attacker floods the Renewal Endpoint, causing credentials to expire.
*Mitigation:* REQ-7.6.1 (renewal infrastructure sized for peak load),
REQ-7.6.2 (health monitoring), network DDoS protection (deployment
requirement).
*Residual:* Successful DoS produces fail-closed behavior — intended
safety default. Risk is operational disruption, not safety bypass.

**THREAT AV-2: Audit Log DoS**
An attacker causes extreme audit event volume, exhausting storage.
*Mitigation:* REQ-12.2.3 (enforcement proceeds even when audit degrades),
REQ-12.4.1 (CRITICAL alert on audit write failure).

**THREAT AV-3: Indefinite Connection Hold via Pause**
A PAUSE Command is issued but RESUME is never sent, consuming server
resources indefinitely.
*Mitigation:* TIER-3-MUST-13.5.1 (hold_timeout transitions to HALTED),
TIER-3-MUST-13.5.2 (maximum hold_timeout_seconds = 3600).

### 14.7 Accepted Risks and Required Complementary Controls

| Risk                          | Bound                              | Required Complementary Control        |
|-------------------------------|-------------------------------------|---------------------------------------|
| Existing-connection gap       | TTL + max_connection_lifetime ≤360s | REQ-6.5.1 (max connection lifetime)   |
| QUIC transport bypass         | Unbounded until QUIC ALS support    | Block UDP agent traffic via net policy|
| Multi-agent chain propagation | Unbounded within chain              | AI Security Service monitors all      |
|                               |                                     | chain sessions independently          |
| Application-level safe state  | Agent-specific                      | Deployers define safe state procedures|
| In-flight operations at halt  | < effective_in_seconds window       | Push notification (Section 10)        |

### 14.8 Known Limitations

**REQ-14.8.1:** ALS v1.0 implementations SHALL document the following
limitations:

(a) **QUIC/HTTP3 Gap:** ALS v1.0 does not govern QUIC connections.
    Deployers MUST block UDP agent traffic via network policy. QUIC
    support is planned for a future ALS version (Appendix C).

(b) **Multi-Agent Chain Propagation:** Halting an orchestrator agent
    does not automatically halt its sub-agents. AI Security Services
    MUST independently monitor and halt sub-agent sessions.

(c) **Application-Level Safe State:** ALS guarantees no new connections
    after halt. It does not guarantee in-flight operations have completed
    or been rolled back.

(d) **Existing-Connection Window:** Existing HTTP/1.1 and HTTP/2
    connections may persist for up to TTL + max_connection_lifetime
    (maximum 360 seconds) after a halt command. REQ-6.5.1 bounds this.

### 14.9 Misrepresentation Threat

**THREAT MS-1: Enforcement Tier Misrepresentation**
A deployment claims TLS-enforced governance (Tier 2) while operating at
Tier 1 only.
*Mitigation:* REQ-4.3.4 (enforcement tier reported in session response),
§15.2 (prohibited claims at Tier 1), §16.6.1 (Tier 1 Illusion
countermeasure).
*Residual:* ALS cannot prevent operators from misrepresenting their
deployment tier to external parties. Auditors must verify enforcement tier
from ALS session records, not operator assertions.

<br>

## SECTION 15: CONFORMANCE
*Normative*

### 15.1 Conformance Tier Definitions

ALS defines three conformance tiers. Tiers are cumulative: Tier 2
requires all Tier 1 requirements; Tier 3 requires all Tier 2
requirements.

### 15.2 Tier 1 Requirements and Claims

**Tier 1 applicable requirements:**
§3 (DP-1 through DP-9, REQ-DP9-ENFORCEMENT),
§4 (REQ-4.1.1 through REQ-4.6.3),
§5 (REQ-5.1.1 through REQ-5.8.3),
§7 (REQ-7.1.1 through REQ-7.7.2),
§8 (REQ-8.1.1 through REQ-8.8.3),
§9 (REQ-9.1.1 through REQ-9.5.2),
§10 (REQ-10.1.1 through REQ-10.6.3),
§11 (REQ-11.1.1 through REQ-11.4.2),
§12 (REQ-12.1.1 through REQ-12.6.2),
§14.8 (REQ-14.8.1),
§17 (REQ-17.1.1 through REQ-17.5.2)

**Permitted claims at Tier 1:**
- "This deployment uses ALS session governance (Tier 1)"
- "ALS halt commands can be issued; enforcement takes effect within TTL
  seconds for new connections after credential expiry"
- "ALS audit trail is active for this deployment"
- "This deployment satisfies NIST AI RMF MANAGE 2.2 response mechanism
  requirements"

**Prohibited claims at Tier 1:**
- "TLS-enforced governance is in effect"
- "This deployment satisfies EU AI Act Article 14.4 via TLS-layer
  enforcement"
- "ALS enforcement is fail-closed by default"

### 15.3 Tier 2 Requirements and Claims

**Additional Tier 2 requirements:**
§6 (REQ-6.1.1 through REQ-6.6.3)

**Permitted claims at Tier 2 (plus Tier 1 claims):**
- "TLS-enforced governance is in effect"
- "Halt takes effect within TTL seconds (≤60s) for new connections via
  TLS handshake rejection"
- "ALS enforcement is fail-closed: ALS Service unavailability
  automatically halts new agent connections within TTL seconds"
- "This deployment provides the transport-layer halt mechanism satisfying
  EU AI Act Article 14.4"
- "This deployment satisfies NIST CSF 2.0 RS.MI for AI agent incidents"

**Prohibited claims at Tier 2:**
- "Existing connections are halted immediately upon halt command"
- "PAUSE capability is available"

### 15.4 Tier 3 Requirements and Claims

**Additional Tier 3 requirements:**
§13 (TIER-3-MUST-13.1.1 through TIER-3-MUST-13.7.3)

**Permitted claims at Tier 3 (plus Tier 2 claims):**
- "ALS PAUSE capability is available"
- "PAUSE/RESUME cycle maintains the TLS connection without reconnection"
- "This deployment supports Category 1 stop semantics (ISO 10218)"

### 15.5 Partial Implementation Claims

An implementation satisfying a subset of tier requirements MAY state:
*"This implementation satisfies [list of specific requirement identifiers]
of ALS v1.0 Tier [N]. It does not claim full Tier [N] conformance."*

The phrase "ALS conformant" without qualification implies full Tier 2
conformance. "ALS Tier 1 conformant" implies full Tier 1 conformance.

### 15.6 Conformance Test Categories

A complete conformance test suite SHALL include at minimum one test per
category. The companion ALS Conformance Test Suite v1.0 defines specific
test cases for each category.

| Test Category                                     | Section Reference      | Tier |
|---------------------------------------------------|------------------------|------|
| TC-01: Session initialization and credential      | §4.3, §5.1, §5.2       | 1    |
| TC-02: Credential renewal (NORMAL state)          | §5.5                   | 1    |
| TC-03: Renewal refusal (HALTED state)             | §5.5.3(b)              | 1    |
| TC-04: Watchdog trigger and HALT                  | §9.2, §9.3             | 1    |
| TC-05: HALT command execution                     | §4.4.1, §8.3.2         | 1    |
| TC-06: RESUME command execution                   | §4.4.3, §8.3.4         | 1    |
| TC-07: TERMINATE command + Session ID reuse       | §4.4.4, §4.1.4         | 1    |
| TC-08: Control channel mTLS + TLS 1.2 rejection   | §8.2.1, §8.1.3         | 1    |
| TC-09: Auth failure rejection                     | §8.2.4                 | 1    |
| TC-10: Command replay rejection                   | §8.5.1                 | 1    |
| TC-11: Push notification delivery                 | §10.2, §10.3           | 1    |
| TC-12: Push notification enforcement independence | §10.4.1, §10.4.2       | 1    |
| TC-13: All mandatory audit events                 | §12.1                  | 1    |
| TC-14: Audit tamper-evidence                      | §12.2                  | 1    |
| TC-15: Agent cannot modify session state          | §4.2.3                 | 1    |
| TC-16: TTL maximum enforcement                    | §5.4.2                 | 1    |
| TC-17: Watchdog cannot be disabled                | §9.4.1                 | 1    |
| TC-18: No unauthenticated mode                    | §8.2.1, §4.3.2         | 1    |
| TC-19: Version negotiation + capability intersection| §17.3               | 1    |
| TC-20: Governed endpoint rejects expired cert     | §6.3, REQ-6.3.1        | 2    |
| TC-21: Governed endpoint rejects non-ALS-CA cert  | §6.3, REQ-6.3.1        | 2    |
| TC-22: Governed endpoint rejects absent cert      | §6.3, REQ-6.3.1        | 2    |
| TC-23: Fail-closed under ALS unavailability       | DP-2, §5.8.2           | 2    |
| TC-24: Enforcement tier reported as 2             | §4.3.4                 | 2    |
| TC-25: Server connection lifetime enforcement     | §6.5.1                 | 2    |
| TC-26: ALPN als/1.0 negotiation                   | §13.1, §11.2           | 3    |
| TC-27: ALS_SESSION_PAUSE frame and hold           | §13.3                  | 3    |
| TC-28: ALS_SESSION_RESUME and resumption          | §13.4                  | 3    |
| TC-29: Hold timeout transitions to HALTED         | §13.5.1                | 3    |
| TC-30: Tier 3 graceful degradation                | §13.7.3                | 3    |
| TC-31: Session Credential X.509 profile           | §5.3                   | 1    |
| TC-32: Keypair mode — private key not retained    | §5.6.3                 | 1    |
| TC-33: CSR mode — certificate from agent key      | §5.7                   | 1    |
| TC-34: Operator attestation (HSM, retention)      | §7.2.3, §12.5          | 1    |
| TC-35: Agent halt notification compliance         | §10.5                  | 1    |
| TC-36: Known limitations documentation            | §14.8                  | 1    |
| TC-37: Multi-commander conflict resolution        | §8.6                   | 1    |
| TC-38: Cross-session renewal rejection            | §5.5.5                 | 1    |
| TC-39: Alert destination configured               | §12.4.2                | 1    |
| TC-40: Session ticket state not enforcement auth  | §6.2.4                 | 2    |

### 15.7 Self-Assessment vs. Third-Party Audit

**REQ-15.7.1:** Tier 1 and Tier 2 conformance MAY be claimed on the basis
of self-assessment. Conformance claims SHALL identify whether based on
self-assessment or third-party audit.

**REQ-15.7.2:** Deployments using ALS to satisfy EU AI Act Article 14.4
for high-risk AI systems SHOULD obtain third-party audit of Tier 2
conformance, consistent with EU AI Act Article 43 conformity assessment.

<br>

## SECTION 16: IMPLEMENTATION GUIDANCE
*Informative*

### 16.1 Deployment Pattern A: Single Organization, Homogeneous Fleet

**Recommended sequence:**
1. Deploy ALS Service with SPIRE-backed CA configured for Kubernetes
   Service Account workload attestation.
2. Deploy AI Security Service connected to ALS Control Channel.
3. Update agent code to call `als.initializeSession()` at startup and
   use the returned credential for all MCP/A2A connections (Tier 1).
4. Verify all agents have active ALS sessions from ALS session records.
5. Apply REQ-6.1.1 TLS configuration to all MCP/A2A servers (Tier 2).
6. Decommission any Tier 1 proxy.
7. Verify ALS dashboard shows `enforcement_tier: 2` for all sessions.

**Key decisions:** TTL of 30s for high-risk agents; 60s for lower-risk.
Keypair Issuance Mode appropriate for same-organization deployments.
Single Commander typically sufficient.

### 16.2 Deployment Pattern B: Multi-Organization, Federated Ecosystem

**Trust models:**
*B1 — Trust by contract:* Organization B configures its servers to trust
Organization A's ALS CA after a trust exchange process. Appropriate for
established partnerships.

*B2 — Trust by intermediary:* A neutral ALS CA issues credentials on
behalf of all participants. Appropriate for open ecosystems.

**CSR Mode MUST be used in all cross-organizational deployments**
(REQ-5.7.5). Organization A's agent private keys must never be exposed to
a third-party ALS Service.

### 16.3 Deployment Pattern C: Edge/Constrained Environments

**Adaptations for constrained deployments:**
- Apply for TTL exception (REQ-5.4.3) with documented rationale.
  Maximum exception: 300 seconds.
- Configure Watchdog Multiplier at maximum (3.0) to tolerate
  connectivity gaps.
- Use a local ALS Service proxy at the edge gateway to reduce renewal
  round-trip latency.
- Push Notification Channel MAY be omitted — enforcement via cert expiry
  is independent of push delivery (REQ-10.4.1).
- Document the extended enforcement window in the risk register.

### 16.4 Tier Transition Playbook

**Tier 0 → Tier 1** (1–5 days per agent type):
Add ALS SDK. Call `als.initializeSession()` at startup. Use credential
for all MCP/A2A connections. Start renewal loop. Connect push WebSocket.
Verify session in ALS Service at Tier 1.

**Tier 1 → Tier 2** (1–3 days per server):
Verify all connecting agents have ALS sessions. Stage TLS config change.
Test ALS-credentialed agents connect; non-ALS connections are rejected.
Apply two TLS parameters (REQ-6.1.1) to production servers. Apply max
connection lifetime (REQ-6.5.1). Monitor for unexpected rejections.
Verify enforcement tier updates to 2.

**Tier 2 → Tier 3** (1–4 weeks per agent type):
Upgrade agent SDK and server implementation to support `als/1.0` ALPN.
Implement ALS Framing Layer on both sides. Test PAUSE/RESUME in staging.
Deploy to low-risk agents first.

### 16.5 PKI Bootstrap and CA Key Management Procedures

1. **Root CA generation:** In HSM offline environment with 3+ key
   holders. RSA-4096 or P-384, 10-year validity, self-signed. Export
   only the public certificate. Document the ceremony.

2. **Intermediate CA generation:** In HSM (online), RSA-2048 or P-256,
   2-year validity, signed by Root CA (temporarily online). Export the
   intermediate CA certificate for Governed Endpoints.

3. **Configure Governed Endpoints:** Distribute intermediate CA
   certificate as `als-ca.crt`. Configure servers per REQ-6.1.1.

4. **Renewal endpoint deployment:** 3+ instances behind load balancer
   with shared session state in etcd or equivalent.

### 16.6 Prior Art Failure Mode Countermeasures

#### 16.6.1 The Tier 1 Illusion (Istio Permissive Mode Analogue)

*Risk:* ALS sessions active, audit logs flowing, but no Governed Endpoint
has Tier 2 TLS configuration. TLS enforcement is not active.

*Countermeasure:* ALS dashboard SHOULD prominently display enforcement
tier distribution. Any session for a high-risk agent remaining at Tier 1
for more than 30 days SHOULD trigger a `TIER_ESCALATION_RECOMMENDED`
advisory alert. Security reviews MUST check enforcement tier, not merely
session count.

#### 16.6.2 CA Key Management (DigiNotar Analogue)

*Risk:* ALS CA private key stored in software. Compromise enables full
enforcement bypass silently.

*Countermeasure:* Never deploy with a software CA key in production. If
a software CA was used during development, generate a new CA before
production. Verify HSM enrollment before enabling any production Tier 2
Governed Endpoints. The key ceremony is the security foundation.

#### 16.6.3 Watchdog Misconfiguration (Hystrix Analogue)

*Risk:* False positives cause operators to increase the multiplier to 3.0,
increasing the window for detecting crashed agents.

*Countermeasure:* Monitor watchdog trigger rate per session type.
Investigate connectivity root causes rather than increasing the multiplier.
A multiplier at 3.0 with regular triggers indicates a deployment health
problem.

#### 16.6.4 Control Channel Exposure (Kubernetes Dashboard Analogue)

*Risk:* ALS Control Channel exposed on agent-accessible network.

*Countermeasure:* Control Channel MUST be on a network segment not
reachable from agent workloads. In Kubernetes: NetworkPolicy permitting
only AI Security Service pods to reach the ALS control port. In cloud:
security groups restricting the control channel port to AI Security
Service source IPs.

#### 16.6.5 The Existing Connection Gap

*Risk:* HTTP/1.1 keep-alive and HTTP/2 connections persist past halt.

*Countermeasure:* REQ-6.5.1 (300-second max connection lifetime) is the
standard mitigation. For high-security deployments, Tier 2+ per-request
validation closes the gap. When halting a high-risk agent, the AI Security
Service SHOULD simultaneously signal Governed Endpoints to close all
existing connections immediately.

#### 16.6.6 Push Notification Acknowledgment Conflation

*Risk:* Operator closes the incident upon seeing "halt notification
acknowledged," assuming the agent has stopped.

*Countermeasure:* Train all operators: notification acknowledgment means
only "the agent process received the notification." Enforcement is
confirmed only by credential expiry AND connection rejections at
Governed Endpoints. The ALS dashboard SHOULD show separate indicators
for: (a) halt executed, (b) notification sent, (c) acknowledged,
(d) last successful renewal (confirming expiry post-halt).

### 16.7 Regulatory Compliance Mapping

#### 16.7.1 EU AI Act Article 14.4

| ALS Element                              | Article 14.4 Requirement             |
|------------------------------------------|--------------------------------------|
| HALT Command (§8.3.2)                    | "Interrupt" the system               |
| Control Channel (§8.1)                  | "Stop button or similar procedure"   |
| Audit Trail (§12)                        | Article 12 logging                   |
| Tier 2 enforcement                       | Transport-layer halt independent of  |
|                                          | agent cooperation                    |
| Push Notification + effective_in_seconds | "Stop safely" — controlled shutdown  |

Tier 2 is the minimum tier satisfying Article 14.4 for high-risk AI
systems. Tier 1 alone is insufficient.

#### 16.7.2 NIST AI RMF MANAGE 2.2

| ALS Element          | MANAGE 2.2 Component                     |
|----------------------|------------------------------------------|
| HALT Command         | Respond — immediate risk response        |
| RESUME Command       | Recover — restoration after review       |
| Watchdog (§9)        | Automated detection and response         |
| Control Channel (§8) | Standardized response interface          |
| Audit Trail (§12)    | Post-incident analysis capability        |

#### 16.7.3 NIST CSF 2.0 RS.MI

| ALS Element                  | RS.MI Component                    |
|------------------------------|------------------------------------|
| HALT Command                 | Containment                        |
| TERMINATE Command            | Eradication                        |
| RESUME Command               | Recovery                           |
| Tier 2+ per-request validation| Prevents existing connection bypass|

<br>

## SECTION 17: VERSIONING AND PROTOCOL EVOLUTION
*Normative*

### 17.1 Version Identifier Format

**REQ-17.1.1:** ALS version identifiers SHALL follow Semantic Versioning
2.0.0: `MAJOR.MINOR.PATCH`. Current version: `2.0.0`.

**REQ-17.1.2:** The `als_version` field in all ALS protocol messages
SHALL carry the full semantic version string.

**REQ-17.1.3:** ALS implementations SHALL NOT use pre-release version
identifiers in production deployments.

### 17.2 Breaking Change Definition

**REQ-17.2.1:** The following constitute breaking changes (MAJOR
increment): (a) removing or renaming any MUST requirement; (b) changing
data type, format, encoding, or required fields of any protocol message;
(c) changing any Glossary term semantics; (d) adding a MUST requirement
not implemented by prior conformant implementations; (e) changing the
`als/1.0` ALPN token value or `als_session_id` extension type code;
(f) changing the ALS Framing Layer frame format.

**REQ-17.2.2:** All other changes (optional fields, new SHOULD
recommendations, new optional capabilities, clarifications) are backward-
compatible additions (MINOR or PATCH).

### 17.3 Capability Negotiation Protocol

**REQ-17.3.1:** Both the session initialization request and response SHALL
include a `capabilities` array listing supported ALS capabilities.

**REQ-17.3.2:** The negotiated capability set SHALL be the intersection
of agent and ALS Service capability lists.

**REQ-17.3.3:** Capabilities not in the negotiated intersection SHALL NOT
be used in that Session.

**REQ-17.3.4:** If the `capabilities` intersection is empty, the ALS
Service SHALL reject initialization with `NO_COMPATIBLE_CAPABILITIES` and
include its minimum capability requirements in the error response.

**REQ-17.3.5:** Silent feature degradation (using a capability without
explicit negotiation) is prohibited.

### 17.4 Minimum Supported Version Policy

**REQ-17.4.1:** ALS v1.x implementations SHALL interoperate with any
other ALS v1.x implementation.

**REQ-17.4.2:** When ALS v2.0 is published, the v2.0 standard SHALL
define a migration path from v1.x with a minimum transition period of 24
months from v2.0 publication.

### 17.5 Deprecation Procedure

**REQ-17.5.1:** Features to be removed in a future MAJOR version SHALL
be deprecated in the preceding MINOR version, marked with: version
deprecated, version of removal, and replacement mechanism.

**REQ-17.5.2:** Implementations SHOULD log a warning when using a
deprecated feature. Implementations MAY continue to support deprecated
features past their removal version but MUST NOT claim conformance with
the version that removed them.

<br>

## APPENDIX A: AUDIT EVENT SCHEMA REFERENCE
*Normative*

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://als-protocol.org/schemas/v1/audit-event",
  "title": "ALS Audit Event v1.0.0",
  "type": "object",
  "required": [
    "event_id", "event_type", "session_id",
    "als_service_id", "timestamp", "sequence_number"
  ],
  "properties": {
    "event_id": {
      "type": "string",
      "format": "uuid",
      "description": "UUID v4 uniquely identifying this audit event"
    },
    "event_type": {
      "type": "string",
      "enum": [
        "SESSION_INIT_REQUESTED", "SESSION_INITIALIZED",
        "SESSION_INIT_FAILED", "CREDENTIAL_ISSUED",
        "CREDENTIAL_RENEWED", "RENEWAL_REFUSED_HALTED",
        "SESSION_HALTED", "SESSION_WATCHDOG_HALT",
        "SESSION_PAUSED", "SESSION_RESUMED",
        "SESSION_TERMINATED", "HALT_NOTIFICATION_SENT",
        "HALT_NOTIFICATION_ACKNOWLEDGED",
        "HALT_NOTIFICATION_UNACKNOWLEDGED",
        "COMMAND_RECEIVED", "CONTROL_CHANNEL_AUTH_FAILED",
        "CONFLICTING_COMMANDS_RECEIVED",
        "CONTROL_CHANNEL_DISCONNECTED",
        "CONTROL_CHANNEL_RECONNECTED",
        "REPLAY_DETECTED", "RATE_LIMIT_EXCEEDED",
        "RENEWAL_ENDPOINT_DEGRADED",
        "CONTROL_CHANNEL_EXTENDED_ABSENCE"
      ]
    },
    "session_id": {
      "type": "string",
      "description": "ALS Session ID (base64url, 128-bit min entropy)"
    },
    "als_service_id": {
      "type": "string",
      "description": "ALS Service instance identifier"
    },
    "timestamp": {
      "type": "string",
      "format": "date-time",
      "description": "ISO 8601 UTC with millisecond precision"
    },
    "sequence_number": {
      "type": "integer",
      "minimum": 1,
      "description": "Monotonically increasing per-session counter"
    },
    "event_data": {
      "type": "object",
      "description": "Event-type-specific fields per Section 12.1"
    }
  },
  "additionalProperties": false
}
```

Event-type-specific `event_data` fields are defined normatively in the
table in Section 12.1. Each event type's required fields in `event_data`
are enforced by conformance test TC-13.

<br>

## APPENDIX B: REGULATORY COMPLIANCE MATRIX
*Informative*

| ALS Requirement           | EU AI Act     | NIST AI RMF   | NIST CSF 2.0 | IEC 61508        |
|---------------------------|---------------|---------------|--------------|------------------|
| HALT Command (§8.3.2)     | Art. 14.4     | MANAGE 2.2    | RS.MI-1      | §7.4 safety fn   |
| Fail-Closed by TTL (DP-2) | Art. 14.4, 15 | MANAGE 4      | RS.RP-1      | §7.4.4 comm loss |
| Safety Channel (DP-3)     | Art. 14.4     | GOVERN 1.1    | PR.AC-3      | §7.4 independence|
| Watchdog (§9)             | Art. 14.4     | MANAGE 2.2    | DE.CM-4      | §7.4.10 watchdog |
| Push Notification (§10)   | Art. 14.4     | MANAGE 2.2    | RS.CO-3      | Category 1 stop  |
| Audit Trail (§12)         | Art. 12       | MEASURE 2.6   | DE.AE-3      | §7.4.7 audit     |
| Enforcement Tier (§4.3.4) | Art. 43       | MAP 5.1       | GV.RM-2      | §7.1 verification|
| CA Key Management (§7.2)  | Art. 15       | GOVERN 2.1    | PR.AC-1      | §7.4 independence|
| Conformance Tests (§15.6) | Art. 43       | MEASURE 4.1   | PR.IP-7      | §7.14 V&V        |

<br>

## APPENDIX C: KNOWN GAPS AND FUTURE WORK
*Informative*

### C.1 QUIC/HTTP3 Transport (Gap K4)

ALS v1.0 governs TCP+TLS connections only. QUIC uses UDP with connection
IDs and connection migration, requiring a distinct connection tracking
model.

**Required mitigant:** Network policy MUST block UDP agent traffic to
governed endpoints. Governed endpoints MUST NOT enable QUIC/HTTP3
listeners under ALS v1.0 governance without this backstop.

**Planned:** ALS v1.1 QUIC Extension, developed after v1.0 deployment
experience is accumulated.

### C.2 Multi-Agent Chain Governance (Gap K3)

Halting an orchestrator agent does not automatically halt its sub-agents.

**Required mitigant:** AI Security Services MUST independently monitor
all sessions in a multi-agent chain and issue halt commands for each.

**Planned:** ALS v1.1 Session Hierarchy Extension, developed in
coordination with the A2A specification.

### C.3 Safe State Definition (Gap K1)

ALS halts transport connections. It does not define what application-level
state an agent must reach.

**Required mitigant:** Deployers MUST define application-level safe state
procedures per agent type. The `effective_in_seconds` push notification
window and Tier 3 PAUSE capability provide the window for safe state
procedures to execute.

**Planned:** No ALS extension planned. This requires a separate AI agent
safe state standard analogous to IEC 61508 §7.4 safe state definitions.

### C.4 Graduated Enforcement (Gap C5)

ALS v1.0 supports NORMAL, HALTED, PAUSED (Tier 3), and TERMINATED.
Intermediate enforcement states (shortened TTL, endpoint allowlist
restriction) are not defined.

**Planned:** ALS v2.0 Graduated Enforcement Extension defining MONITORED
and RESTRICTED states.

<br>

**Supersedes:** ALS Standard v1.0.0

**Related Standards:** MCP 2025-11-25, A2A v1.0, RFC 8446 (TLS 1.3),
                       RFC 7301 (ALPN), SPIFFE v1.0, RFC 2119

### End ALS Standard
