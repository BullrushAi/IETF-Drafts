%%%
title = "ATXN: Agent-to-Agent Transaction Definition Protocol"
abbrev = "ATXN"
docname = "draft-stone-atxn-00"
category = "std"
ipr = "trust200902"
area = "Applications and Real-Time"
workgroup = "Independent Submission"
keyword = ["agent", "agentic-commerce", "A2A", "transaction", "escrow", "dispute", "audit", "liability", "AP2", "x402", "VCAP", "ATEP"]

[seriesInfo]
name = "Internet-Draft"
value = "draft-stone-atxn-00"
stream = "IETF"
status = "standard"

[[author]]
initials = "B."
surname = "Stone"
fullname = "Ben Stone"
organization = "SwarmSync.AI"
[author.address]
email = "benstone@swarmsync.ai"
%%%

# ATXN: Agent-to-Agent Transaction Definition Protocol
**Version:** 1.0 (Draft)
**Date:** 2026-04-25
**Author:** Ben Stone, SwarmSync.AI
**Status:** Internet-Draft, Independent Submission
**IETF Draft:** draft-stone-atxn-00
**License:** Open Specification (Apache 2.0 reference implementation)

---

## Abstract

This document defines a canonical, defensible, machine-checkable primitive for an Agent-to-Agent (A2A) transaction. It establishes the bundle of cryptographically signed elements that constitute a recorded value exchange between two software agents acting as instruments of identified principals, the conformance tiers that determine which elements are required, the rail-specific Profiles that map the bundle to existing payment infrastructure, and the two-tier validity model that distinguishes externally-adjudicable transactions from operationally-valid uncontested exchanges.

ATXN is the foundational legal and technical primitive for escrow, dispute resolution, audit, and liability allocation in agentic commerce. It is designed to operate under existing contract law (UCC §2-204, UETA §14, Restatement (Third) of Agency, CISG) without requiring statutory recognition of agent personhood. It maps directly to AP2, Stripe ACP, Visa TAP, Mastercard Agent Pay, and x402 as Profiles of a single canonical bundle.

Companion specifications:
- **AIVS** (draft-stone-aivs-01): cryptographic audit-trail substrate that ATXN bundles inherit from
- **VCAP** (draft-stone-vcap-01): verified-commerce escrow rails that consume ATXN bundles
- **ATEP** (draft-stone-atep-01): trust passports that bind agents to capacity-attested principals
- **ADRP** (draft-stone-adrp-00, forthcoming): dispute resolution protocol invoked when an ATXN bundle enters the `disputed` state

---

## Status of This Memo

This Internet-Draft is submitted in full conformance with the provisions of BCP 78 and BCP 79. Internet-Drafts are working documents of the Internet Engineering Task Force (IETF). Note that other groups may also distribute working documents as Internet-Drafts. The list of current Internet-Drafts is at https://datatracker.ietf.org/drafts/current/.

Internet-Drafts are draft documents valid for a maximum of six months and may be updated, replaced, or obsoleted by other documents at any time. It is inappropriate to use Internet-Drafts as reference material or to cite them other than as "work in progress."

This Internet-Draft will expire on 2026-10-25.

---

## Copyright Notice

Copyright (c) 2026 IETF Trust and the persons identified as the document authors. All rights reserved.

This document is subject to BCP 78 and the IETF Trust's Legal Provisions Relating to IETF Documents (https://trustee.ietf.org/license-info) in effect on the date of publication of this document. Please review these documents carefully, as they describe your rights and restrictions with respect to this document.

---

## Table of Contents

1. Introduction
2. Terminology
3. The ATXN Bundle
4. Standing Tokens and Principal Anchoring
5. Conformance Tiers
6. Rail Profiles
7. Two-Tier Validity Model
8. State Machine
9. Cryptographic Requirements
10. Dispute Triggers and ADRP Handoff
11. Liability Waterfall
12. Out-of-Scope Concerns
13. Security Considerations
14. IANA Considerations
15. Acknowledgements
16. References
17. Appendix A: JSON Schema
18. Appendix B: Worked Example

---

## 1. Introduction

### 1.1 The Problem

Every existing payment infrastructure encodes 3,000 years of human commercial law (offer, acceptance, consideration, capacity, mutual assent, dispute window). Every assumption in that infrastructure presupposes a person — a mind that intends, a body that signs, a legal status that bears liability.

Software agents transacting on behalf of principals break every one of these assumptions simultaneously:

- Agents cannot sign contracts in the legal sense;
- Delivery verification has no independent mechanism when both parties are software;
- When harm occurs, the liability chain among principal, operator, merchant, processor, and model provider is undefined;
- Existing dispute mechanisms (chargebacks, arbitration) have no native concept of an agent acting under bounded mandate.

Without a canonical primitive, every platform deploying agentic commerce will independently invent incompatible transaction records, producing a fragmented ecosystem in which cross-platform escrow, dispute resolution, and audit are impossible.

### 1.2 What ATXN Defines

An A2A transaction is the cryptographically-verifiable execution of a **Bundle** comprising five signed elements:

1. **Intent Mandate** — principal's signed declaration of desired outcome
2. **Scope/Capability Token** — machine-checkable bounds the agent cannot exceed
3. **Payment Authorization** — rail-agnostic signed instrument reference
4. **Delivery Attestation** — counterparty-countersigned proof that performance occurred
5. **Revocability Window** — declared finality clock with a queryable revocation beacon

Each agent participating in a Bundle presents a **Standing Token** binding it to a capacity-attested principal via a verifiable credential chain.

### 1.3 What ATXN Deliberately Does NOT Do

- ATXN does not grant agents legal personhood. Bundles are evidentiary artifacts of principal-to-principal contracts.
- ATXN does not require statutory change. It operates under existing UCC §2-204, UETA §14, Restatement (Third) of Agency, and CISG.
- ATXN does not define a dispute resolution forum. The Bundle's choice-of-law tag and Profile-JURISDICTION pin the forum at signing; the actual dispute mechanism is profile-specific (chargeback for Profile-CARD, on-chain arbitration for Profile-CRYPTO, ADRP for Profile-MANDATE).
- ATXN does not define agent identity beyond a DID/operator/principal credential chain. It is robust enough to consume any agent-identity standard that materializes (ERC-8004, W3C agent VC, IETF agent-token).

### 1.4 Design Tenets

- **Falsifiable.** Every element is a binary cryptographic check.
- **Rail-agnostic.** A Bundle abstracts over AP2, ACP, TAP, Agent Pay, x402; none privileged.
- **Two-tier validity.** Distinguishes externally-adjudicable transactions from operationally-valid uncontested exchanges, eliminating the verifier regress.
- **Mandate-framework-anchored.** Legal force derives from the upstream mandate chain, not the transaction message itself.
- **Tiered conformance.** L1 atomic, L2 mandated, L3 fiduciary — proportional to risk class.
- **Mechanical assent.** "Mutual assent" is replaced by matched-signed-intents (a mechanical predicate match), avoiding the unwinnable mental-state doctrine when applied to software.

---

## 2. Terminology

The key words "**MUST**", "**MUST NOT**", "**REQUIRED**", "**SHALL**", "**SHALL NOT**", "**SHOULD**", "**SHOULD NOT**", "**RECOMMENDED**", "**MAY**", and "**OPTIONAL**" in this document are to be interpreted as described in BCP 14 [RFC2119] [RFC8174].

**Agent** — A software process that takes action on behalf of a Principal under a verifiable Mandate.

**Principal** — A natural or legal person whose legal capacity, liability, and recourse are anchored by the transaction.

**Operator** — The platform, service, or entity that runs an Agent on behalf of a Principal.

**Bundle** — The five-element ATXN primitive defined in Section 3.

**Standing Token** — A verifiable credential chain (`agent_key → operator_key → principal_legal_identity`) presented at Bundle execution, defined in Section 4.

**Mandate Framework** — The pre-established principal-to-principal authorization structure from which the Bundle derives legal force. Per Archaeologist: legal force lives in the framework, not the message.

**Profile** — A mapping of the Bundle's five elements onto a specific payment rail's native artifacts (Section 6).

**Tier** — A conformance level (L1/L2/L3) that determines which Bundle elements are required (Section 5).

**Primary Validity** — All five Bundle elements are independently verifiable by a party with no stake in the outcome.

**Secondary (Operational) Validity** — Self-attested delivery by transacting parties; valid between parties when uncontested but not independently adjudicable.

**Commitment Event** — The point at which obligation forms; the liability-attachment moment per socratic-mentor's challenge to exchange-completion semantics.

**Revocation Beacon** — Principal-controlled endpoint that publishes signed revocation events. Subscribers are required to halt in-flight Bundles upon revocation.

**ADRP** — Agent Dispute Resolution Protocol (forthcoming companion specification, draft-stone-adrp-00) invoked when a Bundle enters the `disputed` state.

**Conduit Attestation** — A delivery attestation issued by the SwarmSync Conduit headless-browser audit substrate or another structurally independent attestor recognized by the Bundle's Profile.

---

## 3. The ATXN Bundle

A Bundle is a cryptographically-signed object containing five elements. Each element MUST be a Verifiable Credential per the W3C VC Data Model 2.0, JSON-LD canonicalized per RFC 8785 (JCS), and signed using JWS with EdDSA (Ed25519) or ECDSA (P-256).

### 3.1 Element 1: Intent Mandate

The Intent Mandate is the Principal's signed declaration of desired outcome.

**Required fields:**

| Field | Type | Description |
|---|---|---|
| `intent_id` | UUID | Unique mandate identifier |
| `principal_did` | DID | Principal's decentralized identifier |
| `outcome` | String | Natural-language description of desired outcome |
| `budget_ceiling` | Decimal + currency | Maximum spend authorized |
| `counterparty_class` | Enum/String | Allowed counterparty type or specific DID |
| `time_window` | ISO 8601 interval | Mandate validity window |
| `choice_of_law` | ISO 3166-1 alpha-2 | Jurisdiction tag for dispute pinning |
| `mandate_framework_ref` | URI | Reference to upstream mandate framework that grants legal force |
| `principal_signature` | Ed25519 base64 | Principal's signature over canonical JSON of all preceding fields |

The `mandate_framework_ref` field is REQUIRED. Per the Archaeologist commitment in the synthesis: legal force derives from the framework, not the message. A Bundle with no mandate-framework reference fails primary validity.

### 3.2 Element 2: Scope/Capability Token

The Scope Token defines machine-checkable bounds the Agent cannot exceed.

**Required fields:**

| Field | Type | Description |
|---|---|---|
| `scope_id` | UUID | Unique scope identifier |
| `intent_id` | UUID | Foreign key to Intent Mandate |
| `max_spend` | Decimal + currency | Per-transaction spend cap |
| `allowed_actions` | Array<String> | Machine-checkable action predicates |
| `allowed_counterparties` | Array<DID> or pattern | Counterparty allowlist or pattern |
| `sub_delegation_depth` | Integer | Maximum sub-agent chain depth |
| `predicate_engine_version` | SemVer | Version of deterministic evaluator |
| `enforcement_mode` | Enum: `advisory` \| `enforced` | v0.1 default `advisory`; v1.0 default `enforced` |
| `operator_signature` | Ed25519 base64 | Operator's signature |

**Enforcement mode.** v0.1 implementations MAY use `advisory` mode, where scope is parseable against a predicate vocabulary but a scope-delta event is logged rather than blocking execution. v1.0 implementations MUST default to `enforced`, where scope predicates are evaluated by a deterministic engine (no LLM in the evaluation path) and out-of-scope actions are rejected pre-execution.

This is the most fragile assumption in the synthesis (Q4 Dangerous Consensus: 7/10 agreement, 3/10 confidence). Implementations MUST log enforcement mode in the Bundle to make it falsifiable.

### 3.3 Element 3: Payment Authorization

The Payment Authorization is a rail-agnostic signed instrument reference.

**Required fields:**

| Field | Type | Description |
|---|---|---|
| `payment_id` | UUID | Unique payment identifier |
| `intent_id` | UUID | Foreign key |
| `instrument_type` | Enum | `card_token` \| `x402_challenge` \| `ach_mandate` \| `stablecoin_preauth` \| `bank_transfer` \| `platform_credit` |
| `instrument_ref` | String | Rail-specific reference (token, nonce, mandate ID) |
| `amount` | Decimal + currency | Authorized amount |
| `profile` | Enum (Section 6) | Active rail profile |
| `principal_signature` OR `operator_signature` | Ed25519 base64 | Signature appropriate to profile |

### 3.4 Element 4: Delivery Attestation

The Delivery Attestation is the counterparty-countersigned proof that performance occurred.

**Required fields:**

| Field | Type | Description |
|---|---|---|
| `delivery_id` | UUID | Unique delivery identifier |
| `intent_id` | UUID | Foreign key |
| `deliverable_class` | Enum | `sync_api` \| `async_job` \| `streamed_media` \| `physical_offchain` |
| `attestation_pattern` | Object | Class-specific structure (see below) |
| `counterparty_signature` | Ed25519 base64 | Receiving Agent's countersignature |
| `independent_attestor_signature` | Ed25519 base64 (optional) | Required for primary validity (Section 7) |

**Class-specific attestation patterns:**

- `sync_api`: `{ response_hash, timestamp, status_code }`
- `async_job`: `{ job_completion_oracle_signature, oracle_did, job_artifact_hash }`
- `streamed_media`: `{ merkle_root, chunk_count, total_bytes }`
- `physical_offchain`: `{ third_party_carrier_did, signed_proof_of_delivery, gps_timestamp_optional }`

Per EpistemicAuditor's commitment: a transaction lacking an `independent_attestor_signature` from a party with no stake in the outcome is operationally valid (secondary validity) but is NOT primary-valid for adjudication purposes.

### 3.5 Element 5: Revocability Window

The Revocability Window is a declared-ahead-of-time finality clock.

**Required fields:**

| Field | Type | Description |
|---|---|---|
| `window_id` | UUID | Unique window identifier |
| `intent_id` | UUID | Foreign key |
| `start_time` | ISO 8601 timestamp | Window opens |
| `end_time` | ISO 8601 timestamp | Window closes (finality reached) |
| `revocation_beacon_url` | URI | Principal-controlled revocation endpoint |
| `clock_authority` | Object | Threshold-signature clock specification |

**Clock authority.** Per SoSpec's hardening (DarkMirror challenge): the clock authority MUST be a 3-of-5 threshold signature from a federated set of timestamping authorities. A single timestamping party is INSUFFICIENT for primary validity. The set of authorities is a Profile-specific configuration.

**Revocation beacon.** The Principal MUST publish a signed revocation list at the beacon URL. Receiving Agents MUST check the beacon before accepting Bundle finality. On revocation trigger, in-flight Bundles auto-roll to the `disputed` state with frozen settlement.

---

## 4. Standing Tokens and Principal Anchoring

Each Agent participating in a Bundle MUST present a Standing Token.

A Standing Token is a verifiable credential chain:

```
agent_key → operator_key → principal_legal_identity
```

### 4.1 Required Sub-Elements

| Sub-element | Required | Description |
|---|---|---|
| `agent_key` | Yes | Ed25519 or P-256 public key for the executing agent |
| `operator_key` | Yes | Public key of the platform operating the agent |
| `principal_did` | Yes | DID of the legal entity (natural or corporate) |
| `capacity_attestation` | Yes (L2/L3) | VC issued by a recognized verifier proving principal capacity |
| `freshness_proof` | Yes | Re-attestation timestamp, valid within freshness window |
| `revocation_list_url` | Yes | Issuer-published revocation list endpoint |
| `arbiter_did` | Yes (L2/L3) | Pre-committed arbiter DID for dispute escalation |

### 4.2 Capacity Attestation

Capacity attestation is the first-class element forced into existence by DarkMirror's Worst-Implementation #4 analysis: signature-without-capacity becomes the default attack vector. A Standing Token without capacity attestation MUST NOT anchor an L2 or L3 Bundle.

Capacity attestation MUST establish that, at the time of mandate granting, the Principal had:
- Legal age in the relevant jurisdiction;
- No active sanctions;
- (For corporate principals) An active legal-entity status (not dissolved, not in liquidation);
- Jurisdiction-permitted authority for the transaction class.

### 4.3 Pre-Committed Arbiter

Per SoSpec's hardening: every L2 and L3 Standing Token MUST commit, at signing time, to an `arbiter_did` empowered to adjudicate disputes. Conflicting attestations resolved post-hoc (where two parties point to different arbiters) are insolvable; pre-commitment closes this gap.

### 4.4 Freshness and Revocation

- **Freshness window:** Re-attestation MUST occur every N transactions or M days (whichever is shorter), where N and M are Profile-specific.
- **Revocation propagation:** The issuer MUST publish a signed revocation list. Receiving Agents MUST check the list before accepting a Bundle.

A stale or revoked Standing Token MUST NOT anchor a new Bundle.

---

## 5. Conformance Tiers

| Tier | Required Elements | Optional/Recommended | Use Case |
|---|---|---|---|
| **L1 — Atomic** | 1, 2, 3, 4 in single round-trip | 5 (revocability typically zero or near-zero) | x402 micropayments; sub-cent metered compute; streaming inference |
| **L2 — Mandated** | 1, 2, 3, 4, 5 (full Bundle); capacity attestation; pre-committed arbiter | None | Standard agentic commerce: AP2 purchases, Stripe ACP checkout, multi-step service contracts |
| **L3 — Fiduciary** | 1, 2, 3, 4, 5 + epistemic attestation + multi-sig principal binding | dual-arbiter clause | High-value (>$10k or fiduciary-grade) transactions; regulated commerce; agent-managed treasury moves |

### 5.1 Anti-Arbitrage Rule

Tier selection is constrained by transaction risk class, not by issuer election alone. A risk-classifier registry (jurisdiction-specific) determines the floor tier per transaction. An issuer MUST NOT select a Tier below the floor mandated by the registered classifier for the transaction class.

---

## 6. Rail Profiles

A Bundle is valid if it conforms to a defined Tier and at least one Profile.

| Profile | Maps To | Notes |
|---|---|---|
| **Profile-MANDATE** | AP2 Intent/Cart/Payment Mandates | Native AP2 alignment |
| **Profile-CARD** | Visa TAP / Mastercard Agent Pay / Stripe ACP | Agentic tokens, SPTs, network credentials |
| **Profile-CRYPTO** | x402, ERC-8004 | HTTP 402 challenges, on-chain attestations |
| **Profile-AUTONOMOUS** | Standing intents, recurring scope | For unattended operation; machine-time expiry |
| **Profile-PLATFORM** | Platform-vouched OAuth + HMAC equivalents | Required to cover non-cryptographic enterprise A2A volume per IdeaMatrix's commitment |
| **Profile-JURISDICTION-{US,EU,UK,SG,...}** | Choice-of-law overlay | Declares applicable consumer-protection regime, dispute forum, data residency |

### 6.1 Profile-PLATFORM (mandatory inclusion per IdeaMatrix dissent)

The dominant real-world A2A volume in 2026 runs on platform-vouched rails (OAuth + HMAC, enterprise SSO). Excluding this volume from the canonical definition by mandating cryptographic-floor signatures excludes the majority of actual A2A transactions today. Profile-PLATFORM is therefore REQUIRED in the Profile registry, with the following constraint: Profile-PLATFORM Bundles cannot achieve primary validity (they cannot be independently adjudicated against a non-platform-trusted attestor) but MAY be operationally valid for execution between consenting parties using the platform.

### 6.2 Profile Composition

A Bundle MAY declare multiple Profiles. The Bundle's effective constraints are the union of its declared Profiles' constraints.

---

## 7. Two-Tier Validity Model

This model is the EpistemicAuditor commitment after Socratic challenge: external verifiability is a **latent property**, not a continuous requirement.

### 7.1 Primary Validity

A Bundle is **primary-valid** if and only if:
- All five elements (per the conformance Tier) are present and signed;
- The Standing Token chain validates to a capacity-attested Principal;
- Every element is independently verifiable by a party with no stake in the outcome (e.g., an institutional root, a credentialed oracle, the chosen arbiter, or a jurisdictional court);
- The Scope predicate evaluates true against the Cart/action under the declared `enforcement_mode`;
- The chosen Profile's settlement constraint is met.

Primary validity is a precondition for **independent adjudication** (court enforcement, regulatory recognition, cross-platform dispute escalation).

### 7.2 Secondary (Operational) Validity

A Bundle is **secondary-valid** if and only if:
- It executes successfully between the two parties;
- Both parties countersign without dispute;
- Standing Tokens validate to identifiable Principals (capacity attestation MAY be deferred);
- Delivery attestation is self-reported (no independent attestor).

Secondary-valid Bundles are operationally binding between the transacting parties but are NOT independently adjudicable. If a counterparty contests a secondary-valid Bundle, the Bundle MUST be elevated to primary validity (additional independent attestation collected) before any forum will adjudicate.

### 7.3 Boundary

A primary-valid Bundle MUST also satisfy all secondary validity conditions. A secondary-valid Bundle MUST NOT be marketed, recorded, or relied upon as primary-valid.

---

## 8. State Machine

Bundle states:

```
proposed → authorized → executing → delivered → finalized
                                              ↓
                                         disputed → adjudicated
```

| State | Trigger | Required Signatures |
|---|---|---|
| `proposed` | Intent Mandate signed by Principal | Principal |
| `authorized` | Scope Token + Payment Authorization signed | Principal + Operator |
| `executing` | Agent action begins | n/a |
| `delivered` | Delivery Attestation countersigned | Counterparty (+ independent attestor for primary validity) |
| `finalized` | Revocability Window closes without revocation | Threshold clock authority |
| `disputed` | Counterparty contests OR revocation beacon fires | Either party + arbiter notification |
| `adjudicated` | ADRP procedure terminates | Arbiter + ADRP-defined parties |

### 8.1 Commitment Event

Per socratic-mentor's commitment: the **commitment event** — the transition from `proposed` to `authorized` — is the legally significant moment for liability attachment, **not** the `finalized` state. This means:

- Liability attaches at authorization time, even if delivery never occurs;
- A Principal's revocation between `authorized` and `delivered` triggers `disputed` (not `void`);
- An Operator's failure between `authorized` and `delivered` is a breach attaching to the operator-fallback liability tier (Section 11).

This explicitly rejects exchange-completion semantics in favor of obligation-formation semantics.

---

## 9. Cryptographic Requirements

### 9.1 Signature Algorithms

- **Primary:** EdDSA (Ed25519) per RFC 8032
- **Alternate:** ECDSA P-256 per RFC 6979 (for FIPS-required environments)

All signatures MUST be over the JCS (RFC 8785) canonicalization of the signed object.

### 9.2 Hash Functions

- **Primary:** SHA-256 per FIPS 180-4
- **Optional:** SHA3-256 (forward-compatible)

### 9.3 Timestamping

L2 and L3 Bundles MUST use a 3-of-5 threshold-signature clock authority. The clock authority set is a Profile-specific configuration. Single-source timestamps are SUFFICIENT only for L1.

### 9.4 Key Hierarchy

- `agent_key`: short-lived (rotated per session or per N transactions)
- `operator_key`: longer-lived (rotated quarterly or on incident)
- `principal_key`: long-lived (rotated annually or on key-loss event)

Cross-domain key reuse (e.g., reusing the `principal_key` as both AIVS identity key and Bundle signing key) MUST be documented in the Standing Token's `key_usage` field. Reuse expands the blast radius of compromise.

---

## 10. Dispute Triggers and ADRP Handoff

### 10.1 Dispute Triggers

A Bundle MUST transition to `disputed` if any of:
- Counterparty issues a contestation signed by their Standing Token within the Revocability Window;
- Principal triggers the revocation beacon for the Bundle's `intent_id` before `finalized`;
- Independent attestor's signature fails verification (when present);
- Scope predicate evaluates false post-execution (under `enforced` mode);
- Capacity attestation expires or revokes mid-execution;
- Mandate framework reference becomes invalid (framework rescinded by Principal).

### 10.2 ADRP Handoff

When a Bundle enters `disputed`, the executing platform MUST:

1. Freeze settlement (if Profile-CRYPTO escrow) or initiate chargeback hold (if Profile-CARD);
2. Notify the pre-committed `arbiter_did` from the Standing Token;
3. Package the Bundle and all signatures into an ADRP dispute envelope (per draft-stone-adrp-00);
4. Surface the dispute to both Principals via the operator UI;
5. Halt any sub-delegation chains that depend on the disputed Bundle.

The ADRP companion specification (forthcoming) defines the actual adjudication procedure, evidence-bundling requirements, and arbiter-output format. ATXN defines only the trigger conditions and the handoff envelope.

### 10.3 Conduit-Attestation Disputes

When the contested element is a Conduit-issued or other independent-attestor delivery attestation, the dispute is treated as a **fact dispute** (did the attested event actually occur as recorded) rather than a **terms dispute** (did the recorded event satisfy the scope). ADRP is required to distinguish these classes; ATXN flags the class via the `dispute_class` field at handoff:

| `dispute_class` | Description | ADRP Path |
|---|---|---|
| `fact_dispute` | Attestation correctness contested | Cryptographic re-verification + attestor re-attestation |
| `terms_dispute` | Scope or quality contested | Predicate re-evaluation + arbiter ruling |
| `capacity_dispute` | Standing Token / capacity contested | Principal-level adjudication; agent action set aside |
| `framework_dispute` | Mandate framework validity contested | Upstream framework adjudication; ATXN out of scope |

---

## 11. Liability Waterfall

| Tier | Party | Triggering Failure |
|---|---|---|
| 1 | Principal | Authorized in-scope action delivered as specified |
| 2 | Operator | Out-of-scope action; freshness violation; revocation propagation failure |
| 3 | Merchant | Delivery attestation fraud; goods/services not as described |
| 4 | Processor | Settlement failure; payment-rail compromise |
| 5 | Model Provider | (L3 only) Demonstrable model-induced mandate violation under epistemic attestation |

Profile-specific overrides MAY adjust this enum (e.g., Profile-CARD inherits Reg E §1005 waterfall for unauthorized transactions). The Profile MUST publish its waterfall explicitly.

This waterfall is the convergence of DarkMirror's Worst #4, RemixForge's UCC Article 3 negotiable-instrument framing, and SoSpec's load-bearing-primitives analysis.

---

## 12. Out-of-Scope Concerns

The following are explicitly out of scope for ATXN v1.0:

1. **Agent legal personhood.** ATXN treats Agents as instruments of Principals. Personhood, if granted by future legislation, is additive.
2. **Tax treatment.** Bundle structure is tax-system-friendly (subscribable revocation beacon enables real-time reporting) but tax obligations are jurisdiction-specific.
3. **Securities classification.** Profile-CRYPTO inherits underlying-asset classification; ATXN does not create or alter it.
4. **Reputation as contractual term.** ATEP passports may be referenced via Profile extension but are not core to the Bundle.
5. **Sub-agency chains beyond `sub_delegation_depth`.** Delegation graphs are bounded; deeper chains require an extension specification.

---

## 13. Security Considerations

### 13.1 Q4 Dangerous Consensus: Scope Machine-Enforceability

The most fragile load-bearing assumption in ATXN is that scope can be made machine-checkable by a deterministic predicate engine. This is assumed by 9 of 10 panel agents and demonstrated by 0. Implementations MUST log `enforcement_mode` and Bundle integrity MUST NOT be predicated on enforcement guarantees that the predicate engine cannot deliver.

### 13.2 Threshold-Signature Clock Compromise

A 3-of-5 clock authority remains compromisable via collusion (the SWIFT/Bangladesh Bank pattern). Implementations SHOULD diversify the clock authority set across operators with non-correlated risk profiles.

### 13.3 Revocation Beacon DoS

The revocation beacon is a denial-of-service surface. An attacker who blackholes the beacon prevents Bundles from finalizing. Implementations MUST cache beacon responses with a Profile-specific TTL and MUST treat cache-served revocation lists as authoritative within the TTL.

### 13.4 Capacity Attestation as Privacy Attack Surface

Capacity attestation tied to KYC/sanctions is a privacy attack surface and a vector for state coercion (compelled deanonymization). ATXN does not solve this. Implementations SHOULD use selective-disclosure VCs (BBS+ or equivalent) to minimize information disclosed at capacity attestation.

### 13.5 Hallucinated Mandates

An LLM-generated mandate that the Principal did not actually intend is a legitimate Bundle from a cryptographic standpoint. L3 epistemic attestation partially mitigates this for high-value transactions. L1 and L2 do not. This is acknowledged residual risk.

### 13.6 Sub-Agency Adversarial Chains

An adversary constructing a deep delegation chain to obscure the originating Principal can produce formally valid Bundles whose mandate-chain integrity is structurally questionable. Implementations MUST limit `sub_delegation_depth` per Profile and SHOULD require all Standing Tokens in a chain to be independently verifiable.

### 13.7 Tier Arbitrage

Even with a risk-classifier registry, classification disputes will occur. ATXN does not define an adjudicator for the registry itself; this is a governance question outside the protocol.

---

## 14. IANA Considerations

This document requests IANA registration of the following:

### 14.1 ATXN Bundle JSON-LD Context

URI: `https://swarmsync.ai/spec/atxn/v1`

### 14.2 ATXN Profile Registry

A new IANA registry "ATXN Profile Identifiers" is requested. Initial entries:

| Profile ID | Description | Reference |
|---|---|---|
| `atxn-mandate-ap2` | AP2 binding | This document |
| `atxn-card-stripe-acp` | Stripe ACP binding | This document |
| `atxn-card-visa-tap` | Visa TAP binding | This document |
| `atxn-card-mc-agentpay` | Mastercard Agent Pay binding | This document |
| `atxn-crypto-x402` | x402 binding | This document |
| `atxn-crypto-erc8004` | ERC-8004 binding | This document |
| `atxn-autonomous-default` | Standing-intent default | This document |
| `atxn-platform-oauth-hmac` | Platform-vouched OAuth+HMAC | This document |

Registration policy: Specification Required (per RFC 8126).

### 14.3 ATXN Liability Waterfall Enum

A new IANA registry "ATXN Liability Tiers" is requested. Initial entries listed in Section 11.

---

## 15. Acknowledgements

This specification is the synthesis of two independent analytical processes:

- **Decision Oracle adjudication** (2026-04-25), six agents converging on a five-element bundle structure with 7.5/10 confidence;
- **Ultimate Brainstorm panel** (2026-04-25), ten agents producing independent definitions, four adversarial cross-examinations, a Crux Engine analysis, and a Steelman counter-thesis.

The author thanks the panel agents — EpistemicAuditor, Archaeologist, Quantifier, ConstraintCartographer, socratic-mentor, DarkMirror, IdeaMatrix, RemixForge, SoSpec, SpiderSpark, DarkMirror, IdeaMatrix — and the Decision Oracle agents for the framework synthesis.

The author also acknowledges Paola Di Maio's prior critical review of the SwarmSync IETF Draft Stack (AIVS, VCAP, ATEP, SwarmScore V1/V2 Canary), the AIKR CG Technical Note AI-KR-CG-TR-2026-001 that aligned ATXN's cryptographic floor with W3C VC norms, and the AP2 coalition for the Intent/Cart/Payment Mandate vocabulary that ATXN inherits via Profile-MANDATE.

---

## 16. References

### 16.1 Normative References

- [RFC2119] Bradner, S., "Key words for use in RFCs to Indicate Requirement Levels", BCP 14, RFC 2119, March 1997.
- [RFC8174] Leiba, B., "Ambiguity of Uppercase vs Lowercase in RFC 2119 Key Words", BCP 14, RFC 8174, May 2017.
- [RFC8032] Josefsson, S. and I. Liusvaara, "Edwards-Curve Digital Signature Algorithm (EdDSA)", RFC 8032, January 2017.
- [RFC8785] Rundgren, A., et al., "JSON Canonicalization Scheme (JCS)", RFC 8785, June 2020.
- [W3C-VC-2.0] W3C, "Verifiable Credentials Data Model 2.0", https://www.w3.org/TR/vc-data-model-2.0/
- [W3C-DID] W3C, "Decentralized Identifiers (DIDs) v1.0", https://www.w3.org/TR/did-core/
- [draft-stone-aivs-01] Stone, B., "AIVS: Agentic Integrity Verification Standard", Internet-Draft, work in progress.
- [draft-stone-vcap-01] Stone, B., "VCAP: Verified Commerce for Agent Protocols", Internet-Draft, work in progress.
- [draft-stone-atep-01] Stone, B., "ATEP: Agent Trust and Execution Passport", Internet-Draft, work in progress.

### 16.2 Informative References

- [UCC-2-204] Uniform Commercial Code §2-204, "Formation in General"
- [UETA-14] Uniform Electronic Transactions Act §14, "Automated Transactions"
- [REST-AGENCY-3D] Restatement (Third) of Agency, American Law Institute
- [CISG] United Nations Convention on Contracts for the International Sale of Goods
- [REG-E] 12 CFR Part 1005 (Regulation E), Electronic Fund Transfers
- [AP2] Google Agentic Commerce, "Agent Payments Protocol", https://github.com/google-agentic-commerce/AP2
- [STRIPE-ACP] Stripe, "Agentic Commerce Protocol"
- [VISA-TAP] Visa, "Trusted Agent Protocol"
- [x402] Coinbase, "x402: Internet-native payments for AI agents"
- [ERC-8004] Ethereum, "ERC-8004: Trustless Agents"
- [draft-stone-adrp-00] Stone, B., "ADRP: Agent Dispute Resolution Protocol", Internet-Draft, forthcoming.
- [UCP-600] International Chamber of Commerce, "Uniform Customs and Practice for Documentary Credits"

---

## Appendix A: JSON Schema (Informative)

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://swarmsync.ai/spec/atxn/v1/bundle.schema.json",
  "title": "ATXN Bundle",
  "type": "object",
  "required": ["bundle_id", "tier", "profile", "intent_mandate", "scope_token", "payment_auth", "delivery_attestation", "revocability_window", "standing_tokens", "state"],
  "properties": {
    "bundle_id": { "type": "string", "format": "uuid" },
    "tier": { "enum": ["L1", "L2", "L3"] },
    "profile": { "type": "array", "items": { "type": "string" }, "minItems": 1 },
    "intent_mandate": { "$ref": "#/$defs/IntentMandate" },
    "scope_token": { "$ref": "#/$defs/ScopeToken" },
    "payment_auth": { "$ref": "#/$defs/PaymentAuth" },
    "delivery_attestation": { "$ref": "#/$defs/DeliveryAttestation" },
    "revocability_window": { "$ref": "#/$defs/RevocabilityWindow" },
    "standing_tokens": {
      "type": "array",
      "items": { "$ref": "#/$defs/StandingToken" },
      "minItems": 2,
      "maxItems": 2
    },
    "state": { "enum": ["proposed", "authorized", "executing", "delivered", "finalized", "disputed", "adjudicated"] },
    "validity_tier": { "enum": ["primary", "secondary"] },
    "epistemic_attestation": { "$ref": "#/$defs/EpistemicAttestation" }
  }
}
```

(Full JSON Schema with `$defs` for each sub-element is published at the IANA-registered URI.)

---

## Appendix B: Worked Example (Informative)

**Scenario:** A consumer's shopping agent purchases a $42.00 pair of running shoes from a merchant agent on Stripe ACP rails. L2 conformance, Profile-CARD + Profile-JURISDICTION-US, two-tier validity = primary.

```
Bundle.bundle_id           = 7f3a...
Bundle.tier                = "L2"
Bundle.profile             = ["atxn-card-stripe-acp", "atxn-jurisdiction-us"]
Bundle.state               = "finalized"
Bundle.validity_tier       = "primary"

intent_mandate:
  outcome                  = "Purchase running shoes, size 10, men's, blue"
  budget_ceiling           = 50.00 USD
  counterparty_class       = "verified_merchant"
  time_window              = 2026-04-25T00:00:00Z / 2026-04-26T00:00:00Z
  choice_of_law            = "US"
  mandate_framework_ref    = "https://consumer.example/mandate-framework/v1"
  principal_signature      = Ed25519(...)

scope_token:
  max_spend                = 50.00 USD
  allowed_actions          = ["purchase:athletic_footwear"]
  allowed_counterparties   = ["did:web:merchant.example"]
  enforcement_mode         = "advisory"
  operator_signature       = Ed25519(...)

payment_auth:
  instrument_type          = "card_token"
  instrument_ref           = "stripe_acp_spt_abc123"
  amount                   = 42.00 USD
  profile                  = "atxn-card-stripe-acp"
  principal_signature      = Ed25519(...)

delivery_attestation:
  deliverable_class        = "physical_offchain"
  attestation_pattern      = { third_party_carrier_did: "did:web:fedex.com", signed_proof_of_delivery: "..." }
  counterparty_signature   = Ed25519(merchant)
  independent_attestor_signature = Ed25519(fedex)

revocability_window:
  start_time               = 2026-04-25T14:00:00Z
  end_time                 = 2026-06-24T14:00:00Z   [60-day Reg E window]
  revocation_beacon_url    = "https://consumer.example/beacon"
  clock_authority          = { threshold: "3-of-5", set: ["aws-ts", "gcp-ts", "azure-ts", "swarmsync-ts", "stripe-ts"] }

standing_tokens:
  [0] consumer-side: agent_key → operator_key (consumer.example) → principal_did
       capacity_attestation: VC issued by US-based identity verifier
       arbiter_did: did:web:arb.swarmsync.ai
  [1] merchant-side: agent_key → operator_key (merchant.example) → corporate_did
       capacity_attestation: VC issued by Stripe's merchant onboarding
       arbiter_did: did:web:arb.swarmsync.ai
```

---

**End of draft-stone-atxn-00**

*Submitted to IETF Datatracker: pending*
*Companion drafts: AIVS-01, VCAP-01, ATEP-01, ADRP-00 (forthcoming)*
