# ALS (Agent Layer Security) Standard

<br>

## Begin ALS Standard

**Status:** Production-Ready for Public Adoption

**Date:** 12/27/2025

**Version:** v1.0.0

**License:** Apache License 2.0

**Audience:** Security architects, AI engineers, AI platform builders, Security Standard authors

<br>

## Table of Contents

1. Introduction
2. Goals and Scope
3. Normative Language
4. Security Principles
5. Protocol Overview
6. Cryptography Requirements
7. Capability Tokens
8. Signed Manifests and Artifact Verification
9. Reputation Signals and Trust Data
10. Continuous Monitoring and Runtime Verification
11. Revocation Protocol
12. Key Management and Trust Anchors
13. Session and Message Flows
14. Audit, Consent, and Logging
15. Versioning and Error Handling
16. Privacy Considerations
17. Conformance and Interoperability
18. Message and Data Structures
19. Examples
20. Security Considerations
21. References

<br>

## 1. Introduction

Agent Layer Security (ALS) defines a standardized security protocol for AI agent interactions over MCP, aggregating authorization, authenticity, and trust signals into a single protocol. ALS operates on top of MCP, similar to how TLS operates on top of HTTP or TCP (except ALS operates at the Application Layer, not the Transport Layer). ALS is transport-agnostic and provides mechanisms for:

* Authorization via cryptographically bound capability tokens
* Authenticity via signed manifests and artifact hashes
* Trust awareness via reputation signals and continuous monitoring
* Auditability, revocation, and user consent

ALS is designed to operate over any secure transport (e.g., TLS, mTLS, Unix Domain Sockets), and is designed for real-time verification of agent interactions over MCP.

<br>

## 2. Goals and Scope

### 2.1 Goals

ALS SHALL:

1. Protect against token replay, confused deputy, and misbinding attacks.
2. Ensure artifact authenticity and integrity at runtime through signed manifests and hashes.
3. Enable continuous supply-chain monitoring, revoking tokens if manifests or artifacts  are modified during runtime.
4. Incorporate reputation signals for advisory or automatic blocking.
5. Provide audit, revocation, and consent mechanisms.

### 2.2 Scope

ALS SHALL specify:

* Token formats, transport-binding requirements, verification rules
* Manifest and artifact signing, format, verification, and monitoring
* Reputation signal ingestion and trust evaluation
* Continuous runtime verification and revocation procedures
* Session and message flows
* Audit and revocation standards

ALS SHALL NOT specify:

* Identity provisioning outside the token issuer
* Logging implementations or storage formats

<br>

## 3. Security Principles

ALS SHALL adhere to:

1. Defense in Depth: Multiple mechanisms SHALL be combined; no single mechanism is sufficient.
2. Transport Binding: All tokens SHALL be bound cryptographically to the session or transport channel.
3. Fine-Grained Capability Enforcement: Tokens SHALL enumerate explicit tool and resource capabilities.
4. Artifact Authenticity: Every artifact and manifest SHALL be signed; verification SHALL occur before execution and continuously during runtime.
5. Trust and Reputation Awareness: Clients SHALL optionally consider reputation signals for advisory or blocking purposes.
6. Audit and Consent: User consent SHALL be explicit for privileged actions; all actions SHALL be auditable.
7. Fail-Safe Defaults: Verification failures SHALL prevent execution unless explicitly overridden by policy.

<br>

## 4. Protocol Overview

ALS defines an integrated security protocol comprising:

1. Transport-Bound Capability Tokens (authorization)
2. Signed Manifests and Artifact Hashes (authenticity)
3. Reputation Signals and Trust Data  (trust)
4. Audit, Revocation, and Consent Mechanisms
5. Continuous Runtime Monitoring (runtime verification)

ALS SHALL ensure that all components are verified continuously; a change in a manifest or artifact SHALL trigger token revocation and prevent further execution.

<br>

## 5. Cryptography Requirements

ALS SHALL use the following cryptography:

* Token Signing: MUST support ES256 (ECDSA P-256) or EdDSA (Ed25519)
* Manifest Signing: MUST support RSA-4096-PSS or EdDSA (Ed25519)
* Artifact Hashes: MUST use SHA-256; SHA-512 is RECOMMENDED for high-security deployments
* Transport Binding: MUST use mTLS or equivalent cryptographic proof-of-possession
* Keys MUST have a minimum lifecycle of 1 year and MUST be rotated or revoked as specified in Section 12

<br>

## 6. Capability Tokens

### 6.1 Purpose

Tokens convey authorization and SHALL be:

* Short-lived (MUST expire after TTL)
* Transport-bound (MUST include session binding information)
* Fine-grained (MUST enumerate allowed resources and actions)
* Revocation-aware (MUST be revoked if manifest/artifact changes)

### 6.2 Token Properties

* Short TTL - prevents long-lived exposure
* Proof-of-Possession binding - mTLS, DPoP, or other cryptographic binding
* Fine-grained capabilities - resource- and action-specific
* Revocation-aware - implementers must provide revocation checks

### 6.3 Token Verification

Implementations MUST:

1. Verify signature using issuer public key
2. Verify transport binding matches current session
3. Verify expiration
4. Verify requested capability is allowed
5. Revoke token if associated manifest or artifact changes during runtime

### 6.4 Allowed Token Types

* JWT (JWS) with `cnf` claim for proof-of-possession
* CWT for constrained environments
* Macaroons with attenuated caveats

### 6.5 Token Example

```
{
"iss": "als-issuer.local",
"sub": "client:alice",
"aud": "mcp-server:searchService",
"exp": 1730000000,
"jti": "session-9f7b",
"version": "1.1",
"capabilities": [
{ "resource": "filesystem:/home/alice/docs", "actions": ["read"] },
{ "tool": "searchFiles", "max_results": 20 }
],
"bound_to": {
"type": "mTLS",
"tls_thumbprint": "sha256:abcdef123456..."
}
}
```

<br>

## 7. Signed Manifests and Artifact Verification

### 7.1 Manifest Contents

Each tool or server SHALL publish a manifest including:

* `name`, `version`, `publisher`
* `requiredCapabilities`
* `artifact` information: `type`, `digest`
* `public_key` for signature verification
* `signature` over the manifest
* `version`: manifest version number

### 7.2 Runtime Artifact Verification

* Artifact digests SHALL be verified against the manifest
* Verification SHALL occur before execution and continuously during runtime
* Any mismatch SHALL revoke associated tokens and block execution

### 7.3 Example Manifest

```
{
"name": "searchFiles",
"version": "1.2.3",
"publisher": "[acme@example.com](mailto:acme@example.com)",
"requiredCapabilities": [
{"resource":"filesystem:/home/alice/docs","actions":["read"]}
],
"artifact": {
"type":"wasm",
"digest":"sha256:0f1e2d3c..."
},
"public_key": "-----BEGIN PUBLIC KEY-----\nMIIBIjANBgkq...\n-----END PUBLIC KEY-----",
"signature": "MEUCIQD...",
"version": "1.1"
}
```

<br>

## 8. Reputation Signals and Trust Data

### 8.1 Purpose

Reputation signals SHALL provide advisory or blocking information for actors and artifacts.

### 8.2 Feed Requirements

* Reputation entries MUST be signed and verifiable
* Clients MAY use scores to warn or block
* High-confidence negative scores SHOULD trigger automatic token revocation


### Example Feed Entry

```
{
"entity_id": "publisher:evilcorp@example.com",
"type": "publisher",
"score": -90,
"reason": "malware-in-artifact",
"first_seen": "2025-10-01T12:00:00Z",
"last_observed": "2025-11-01T08:00:00Z",
"confidence": 0.95,
"source_signatures": ["sig1...", "sig2..."]
}
```

<br>

## 9. Continuous Monitoring and Runtime Verification

ALS implementations MUST:

1. Re-verify manifest signatures and artifact digests periodically (recommended every 5 minutes or configurable)
2. Re-assess reputation scores for critical artifacts
3. Immediately revoke tokens if verification fails
4. Log all verification events for audit

<br>

## 10. Revocation Protocol

ALS specifies runtime token revocation:

* Tokens MUST be checked against revocation lists or push notifications
* Revocation events MAY be triggered by:

  * Manifest or artifact mismatch
  * Reputation score threshold breach
  * Key compromise
 
<br>

* Clients and servers MUST enforce fail-safe blocking for revoked tokens

<br>

## 11. Key Management and Trust Anchors

* Implementations MUST maintain trusted root store
* Keys MUST be rotated at intervals ≤ 1 year
* Compromised keys MUST be revoked and published to trust anchors

<br>

## 12. Session and Message Flows

### 12.1 Session Establishment

1. Client establishes secure transport
2. Client fetches manifest and artifact
3. Client verifies signature and digest
4. Client evaluates reputation signals

### 12.2 Tool Invocation

1. Client requests **transport-bound capability token**
2. Client sends `tools/call` with token
3. Server verifies **token signature, binding, capability, and runtime manifest verification**
4. Server executes tool
5. Both client and server log operation for audit
6. Continuous monitoring occurs throughout session

### 12.3 Audit Flow

* Log includes: timestamp, tool, capability, actor identity (hashed if sensitive), transport session ID, verification results

<br>

## 13. Audit, Consent, and Logging

* Audit: All actions SHALL be logged, including token hash, manifest hash, session ID, timestamp, and verification results
* Revocation: Tokens SHALL be revoked if manifests or artifacts change or reputation signals indicate compromise
* Consent: Clients MAY present human-readable prompts for  HITL (Human-in-the-Loop) workflows

<br>

## 14. Versioning and Error Handling

* Tokens, manifests, and reputation feeds MUST include version numbers
* Implementations MUST define error codes for:

  * Token invalid/expired/revoked
  * Manifest invalid/mismatch
  * Artifact mismatch
  * Reputation block

<br>

## 15. Privacy Considerations

* Minimize telemetry and do not expose client identities
* Reputation signals SHOULD use hashed or anonymized identifiers
* Optional telemetry MAY aggregate data for risk scoring

<br>

## 16. Conformance and Interoperability

* Implementations MUST provide test vectors for tokens, manifests, reputation feeds
* Conformance tests MUST verify: signature verification, token binding, runtime revocation, and error handling

<br>

## 17. Message and Data Structures

### 17.1 Capability Token

* iss: issuer
* sub: subject
* aud: audience
* exp: expiration
* jti: unique identifier
* capabilities: array of {resource, actions}
* bound_to: transport binding information

### 17.2 Manifest

* name
* version
* publisher
* requiredCapabilities
* artifact: {type, digest}
* public_key
* signature

### 17.3 Reputation Feed Entry

* entity_id
* type
* score
* reason
* first_seen / last_observed
* confidence
* source_signatures

<br>

## 18. Security Considerations

ALS MUST protect against:

* Token theft (short TTL, transport binding, revocation)
* Replay attacks (PoP-bound tokens)
* Confused-deputy attacks (fine-grained capabilities)
* Supply-chain compromise (continuous manifest/artifact verification, token revocation)
* Reputation poisoning (signed feeds, multi-validator verification)

Continuous monitoring and runtime revocation are required for supply-chain defense.

<br>

## 19. References

* RFC 2119 — Key words
* JWT — JSON Web Token
* CWT — CBOR Web Token, RFC 8392
* TLS 1.3 — RFC 8446
* SHA-2 family hash functions
* Macaroons — Google Research Publication

## End ALS Standard
