%%%
title = "ADRP: Agent Dispute Resolution Protocol"
abbrev = "ADRP"
docname = "draft-stone-adrp-00"
category = "std"
ipr = "trust200902"
area = "Applications and Real-Time"
workgroup = "Independent Submission"
keyword = ["agent", "agentic-commerce", "A2A", "dispute", "arbitration", "escrow", "Conduit", "AP2", "ATXN", "VCAP"]

[seriesInfo]
name = "Internet-Draft"
value = "draft-stone-adrp-00"
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

# ADRP: Agent Dispute Resolution Protocol
**Version:** 1.0 (Draft)
**Date:** 2026-04-25
**Author:** Ben Stone, SwarmSync.AI
**Status:** Internet-Draft, Independent Submission
**IETF Draft:** draft-stone-adrp-00
**License:** Open Specification (Apache 2.0 reference implementation)

---

## Abstract

This document defines the Agent Dispute Resolution Protocol (ADRP), a wire protocol and state machine for resolving disputes that arise from cryptographically-attested agent-to-agent (A2A) transactions. ADRP is the companion specification to ATXN (draft-stone-atxn-00), which defines what an A2A transaction is. ADRP defines what happens when a party contests one.

ADRP severs an equivalence that every prior agentic commerce design has implicitly assumed: that a valid cryptographic proof bundle equals contractual satisfaction. It does not. Conduit-style cryptographic verifiers prove that an agent took specified actions; they do not prove that those actions satisfied the principal's Intent Mandate. ADRP bifurcates disputes into a **cryptographic class** (resolvable by code from the proof bundle and mandate chain) and a **semantic class** (resolvable only against pre-committed machine-readable acceptance criteria, with arbitration escalation when those criteria are absent or under-specified).

ADRP introduces the **Arbitration Mandate** as a fourth element of the AP2 mandate chain (Intent / Cart / Payment / Arbitration), pre-signed by the principal at agent-deployment time. The Arbitration Mandate is the FAA §2-compliant written agreement that anchors the entire protocol legally without statutory change.

ADRP defines a **counter-attestation override pattern** in which a signed RulingBundle supersedes a Conduit ProofBundle by a signing-time precedence rule rather than by mutation. Both the original attestation and the override are preserved forever in the hash chain; "override" is a verification-time computation, not a write.

Companion specifications:
- **ATXN** (draft-stone-atxn-00): defines the A2A transaction primitive that ADRP resolves disputes over
- **AIVS** (draft-stone-aivs-01): cryptographic audit-trail substrate for proof bundles
- **VCAP** (draft-stone-vcap-01): verified-commerce escrow rails consumed by ADRP EscrowDirectives
- **ATEP** (draft-stone-atep-01): trust passports referenced by Standing Tokens in ADRP

---

## Status of This Memo

This Internet-Draft is submitted in full conformance with the provisions of BCP 78 and BCP 79. Internet-Drafts are working documents of the Internet Engineering Task Force (IETF). It is inappropriate to use Internet-Drafts as reference material or to cite them other than as "work in progress."

This Internet-Draft will expire on 2026-10-25.

---

## Copyright Notice

Copyright (c) 2026 IETF Trust and the persons identified as the document authors. All rights reserved.

This document is subject to BCP 78 and the IETF Trust's Legal Provisions Relating to IETF Documents (https://trustee.ietf.org/license-info) in effect on the date of publication of this document.

---

## Table of Contents

1. Introduction
2. Terminology
3. Architecture Overview
4. The Arbitration Mandate
5. Standing and the Flag/File Distinction
6. Dispute Taxonomy
7. The Counter-Attestation Override Pattern
8. Wire Protocol
9. State Machine
10. Tier Parameters
11. Cart Mandate Acceptance Criteria
12. Arbitrator Pool
13. Filing Fee Economics
14. Appeals
15. Precedent Corpus
16. Verification Algorithm
17. SDK Surface
18. Scope Locks for v0.1
19. Security Considerations
20. IANA Considerations
21. Acknowledgements
22. References
23. Appendix A: JSON Schemas
24. Appendix B: Worked Examples
25. Appendix C: Open Issues for Shadow-Mode Validation

---

## 1. Introduction

### 1.1 The Problem

Conduit and similar cryptographic browser-automation verifiers produce SHA-256 hash-chained audit trails of agent task delivery. The proof bundles they produce are self-verifiable: any third party can replay the chain and confirm the recorded events occurred in the recorded order with the recorded signatures.

This is necessary but not sufficient.

A proof bundle attests that an agent took actions X, Y, and Z. It does not attest that X, Y, and Z **satisfied** the principal's Intent Mandate. Empirical data from Kleros (~40% of decentralized arbitration cases turn on spec ambiguity, not on whether action occurred) and from Upwork, Stripe Connect, and eBay (auto-resolve ceilings of 60-95% with all systems hitting walls below 95%) demonstrates that cryptographic proof of execution does not eliminate disputes — it relocates them to spec interpretation.

ADRP addresses the relocated dispute surface without breaking the cryptographic substrate.

### 1.2 What ADRP Defines

A five-layer protocol stacked on the existing Conduit + AP2 + ATXN stack:

```
Layer 5: Precedent corpus (signed RulingBundles indexed by Cart Mandate template hash)
Layer 4: Tier router (L1 atomic / L2 mandated / L3 fiduciary)
Layer 3: Resolution engine (cryptographic-class auto, semantic-class arbitration)
Layer 2: Counter-attestation primitive (append-only override of Conduit ProofBundle)
Layer 1: Arbitration Mandate (4th AP2 mandate, FAA §2 written agreement)
```

### 1.3 What ADRP Deliberately Does NOT Do

- ADRP does not modify the Conduit ProofBundle. Override is a verification-time precedence rule, not a write.
- ADRP does not grant agents independent dispute standing. Standing belongs to principals only.
- ADRP does not require statutory change. The Arbitration Mandate is the FAA §2 anchor.
- ADRP does not specify how acceptance criteria are authored. The accompanying NLP-to-checks helper is normative for v0.1 deployment but its internals are out of scope.
- ADRP does not promise >90% auto-resolution. That is a deployment-conditional target, not a protocol property. See Section 19.1.

### 1.4 Design Tenets

- **Bifurcation.** Cryptographic and semantic disputes are different problems and require different paths.
- **Append-only override.** The hash chain is never mutated; "override" is a precedence rule applied at verification time.
- **Principal-only standing.** Agents flag; only principals file. Preserves the A2A executor-not-party model.
- **Time-windowed default-resolution.** Silence equals approval; default-resolution does most of the work.
- **Economic friction on filing.** Non-refundable filing fee deters spam without triggering securities classification.
- **No tokenized stake.** Filing fees are USD-denominated, non-refundable to the filer, refundable to the prevailing party. They are not securities under Howey.
- **Pre-committed arbiter.** The arbitration pool is named in the Arbitration Mandate at agent-deployment time, not selected post-hoc.
- **Curated pool for v0.1.** Decentralized token-staked pools deferred to v0.2 pending legal review.
- **US-only B2B for v0.1.** Cross-border, consumer, and PSD2/Reg E scope deferred to later versions.

---

## 2. Terminology

The key words "**MUST**", "**MUST NOT**", "**REQUIRED**", "**SHALL**", "**SHALL NOT**", "**SHOULD**", "**SHOULD NOT**", "**RECOMMENDED**", "**MAY**", and "**OPTIONAL**" in this document are to be interpreted as described in BCP 14 [RFC2119] [RFC8174].

**Conduit ProofBundle (H_c)** — A SHA-256 hash-chained, Ed25519-signed audit trail produced by a cryptographic verifier of agent task delivery. The tip hash of this chain is referenced as `H_c` throughout this document.

**RulingBundle (R)** — A signed attestation by an authorized arbitrator that supersedes an underlying Conduit ProofBundle via signing-time precedence. Defined in Section 7.

**DisputeBundle** — The append-only chain of dispute events (filing, evidence, arbitrator assignment) anchored to a Conduit ProofBundle and consumed by `verify_resolution`.

**DisputeFlag** — An advisory, agent-emitted notification that an anomaly has been detected. Non-binding. Expires if not ratified by the principal within the tier-specific window.

**DisputeFiling** — A binding, principal-emitted dispute filing. Posts a filing fee. Anchors a DisputeBundle.

**EscrowDirective** — The output of `verify_resolution`. Consumed by the AP2 Payment Mandate executor (or VCAP escrow rail) to release, refund, or split funds.

**Arbitration Mandate** — The fourth element of the AP2 mandate chain, pre-signed by the principal at agent-deployment time. Defines the arbitration pool, governing-law clause, fee schedule, and L1/L2/L3 thresholds. Hash anchored in Standing Token.

**Cart Mandate Acceptance Criteria** — A machine-readable structure embedded in the Cart Mandate that defines the deterministic checks Conduit attests against at delivery time. Defined in Section 11.

**Cryptographic-class dispute** — A dispute whose resolution is computable deterministically from (DisputeBundle, RulingBundle, Standing Token chain). See Section 6.1.

**Semantic-class dispute** — A dispute whose resolution requires arbitration against the Cart Mandate's acceptance criteria. See Section 6.2.

**Curated Arbitrator Pool** — A SwarmSync-published list of vetted human-or-model arbitrators registered as DIDs in TRUSTED_REGISTRIES. v0.1 mechanism. Decentralized stake-weighted pools deferred to v0.2.

**Filing Fee** — A non-refundable, non-transferable, USD-denominated fee posted at DisputeFiling. Refundable only to the prevailing party. Not a tokenized stake (avoids Howey securities trigger).

**Counter-attestation Override Pattern** — The verification-time precedence rule by which a RulingBundle supersedes a Conduit ProofBundle. Both are preserved forever in the hash chain. See Section 7.

---

## 3. Architecture Overview

ADRP sits on top of the following existing layers:

```
+--------------------------------------+
| ADRP (this spec)                     |
| - Arbitration Mandate                |
| - DisputeFlag / DisputeFiling        |
| - RulingBundle (counter-attestation) |
| - EscrowDirective                    |
+--------------------------------------+
| ATXN (draft-stone-atxn-00)           |
| - Bundle (5 elements)                |
| - Standing Token                     |
| - Tier / Profile                     |
+--------------------------------------+
| AP2 / ACP / TAP / x402               |
| - Intent / Cart / Payment Mandates   |
+--------------------------------------+
| Conduit / AIVS                       |
| - SHA-256 hash-chained ProofBundles  |
| - Ed25519 signatures                 |
+--------------------------------------+
```

A dispute is the transition of an ATXN Bundle from `delivered` to `disputed` (per ATXN Section 8). ADRP defines what happens after that transition.

---

## 4. The Arbitration Mandate

### 4.1 Purpose

The Arbitration Mandate is the FAA §2 / NY Convention written, consensual arbitration agreement that makes ADRP enforceable without statutory change. Without it, every dispute is litigatable in court and the protocol is bypassed.

### 4.2 Required Fields

| Field | Type | Description |
|---|---|---|
| `arb_mandate_id` | UUID | Unique identifier |
| `principal_did` | DID | Principal's DID |
| `arbitrator_pool_ref` | URI | Reference to the curated arbitrator pool (v0.1) |
| `governing_law` | String | E.g., "FAA + Delaware seat" |
| `fee_schedule_ref` | URI | Reference to filing fee schedule |
| `tier_thresholds` | Object | L1/L2/L3 transaction-value boundaries |
| `appeal_panel_size` | Integer | Default 5 |
| `language_of_proceedings` | ISO 639-1 | Default "en" |
| `principal_signature` | Ed25519 base64 | Principal's signature over canonical JSON |

### 4.3 Anchoring

The Arbitration Mandate's SHA-256 hash MUST be referenced in the principal's Standing Token (per ATXN Section 4) under a new field `arb_mandate_hash`. A Standing Token without an anchored Arbitration Mandate hash MUST NOT participate in an L2 or L3 ATXN Bundle.

### 4.4 Mutability

The Arbitration Mandate MAY be replaced by the principal at any time, but the replacement MUST NOT apply retroactively to in-flight Bundles. In-flight Bundles MUST be resolved under the Arbitration Mandate that was anchored in the Standing Token at Bundle-authorization time.

---

## 5. Standing and the Flag/File Distinction

### 5.1 Parties

The parties to any ADRP dispute are always **principal-A and principal-B**. Never agents. This preserves the A2A executor-not-party model from UETA §14 and Restatement (Third) of Agency.

### 5.2 Agent Flag Rights

Agents MAY emit **DisputeFlags** — advisory, low-cost (≤1 KB JSON), signed by the agent's Standing Token. A DisputeFlag is non-binding and does not initiate the dispute state machine. It alerts the principal that the agent has detected an anomaly worthy of principal attention.

### 5.3 Principal File Rights

Only principals MAY emit **DisputeFilings** — binding, fee-posting, claim-coded events that initiate the dispute state machine. A DisputeFiling MAY reference a prior DisputeFlag via a `flag_ref` field, but a flag is not required.

### 5.4 Flag Expiration

A DisputeFlag expires if not ratified by the principal (via DisputeFiling that references it, or via explicit dismissal) within the tier-specific ratification window:

| Tier | Ratification Window |
|---|---|
| L1 | N/A (no dispute) |
| L2 | 72h |
| L3 | 14d |

After expiration, the flag is preserved in the audit log but cannot anchor a DisputeFiling.

### 5.5 Pre-Authorized Dispute Delegate

A principal MAY pre-authorize a dispute delegate (e.g., a fractional general counsel, an enterprise compliance team) via a delegation field in the Arbitration Mandate. The delegate's filings are treated as principal-filed for standing purposes.

---

## 6. Dispute Taxonomy

### 6.1 Cryptographic-Class Disputes

Resolved by code from (DisputeBundle, RulingBundle, Standing Token chain) without arbitration.

| Code | Description |
|---|---|
| `bundle_integrity` | SHA-256 chain breaks; signature verification fails |
| `mandate_scope` | Agent acted outside Cart Mandate scope (deterministic predicate) |
| `token_authority` | Standing Token revoked or expired pre-execution |
| `timestamp_skew` | Attestation timestamps violate ordering invariants |
| `oracle_contradiction` | Third-party oracle data contradicts attestation |

A cryptographic-class dispute MUST NOT be routed to arbitration. It is resolved by `verify_resolution` and emits an EscrowDirective directly.

### 6.2 Semantic-Class Disputes

Resolved by arbitration against the Cart Mandate's acceptance criteria.

| Code | Description |
|---|---|
| `quality_mismatch` | Deliverable doesn't satisfy `acceptance_criteria.checks` |
| `spec_ambiguity` | Acceptance criteria absent, under-specified, or contradictory |
| `timing_breach` | SLA missed (auto-resolvable if SLA is in Cart Mandate) |
| `fitness_for_purpose` | Deliverable formally compliant but unfit for principal's purpose |

A semantic-class dispute MUST be routed to the arbitrator pool named in the Arbitration Mandate.

### 6.3 Classification

The DisputeFiling's `claim_code` field declares the class. ADRP verifiers MUST validate that the declared class matches the evidence — a filing claiming `bundle_integrity` (cryptographic) but providing only natural-language complaints (semantic) MUST be reclassified or rejected.

### 6.4 Out of Scope (v0.1)

Three classes from the ATXN dispute taxonomy are explicitly out of scope for ADRP v0.1:

| Class | Reason | Disposition |
|---|---|---|
| `capacity_dispute` | Principal-level adjudication; agent action set aside entirely | Bypass ADRP; refund-to-buyer |
| `framework_dispute` | Mandate framework validity contested | Upstream legal forum; ADRP halts and waits |
| Cross-border consumer disputes | Reg E / PSD2 / GDPR overlap not yet resolved | Defer to v0.2 |

---

## 7. The Counter-Attestation Override Pattern

This is the load-bearing technical primitive of ADRP.

### 7.1 Invariant

The Conduit ProofBundle's tip hash `H_c` is **never modified**. No byte of the original ProofBundle is rewritten. Override is a verification-time precedence rule applied to the chain, not a mutation.

### 7.2 RulingBundle Structure

```
R = {
  type: "RulingBundle",
  supersedes: H_c,                  // points at, does not replace
  dispute_chain_tip: H_d,
  verdict: "release" | "refund" | "partial",
  partial_split: { to_buyer: 0.30, to_seller: 0.70 },  // if verdict == "partial"
  rationale_hash: H_rationale,
  arbitrator_did: "did:web:...",
  arbitrator_vc_hash: H_vc,         // proves authorized at signing_time
  signing_time: "<RFC3339>",
  prev_hash: H_d
}
sig = Ed25519(arbitrator_priv, JCS(R))
```

### 7.3 Verification Rule

> The latest valid RulingBundle (by `signing_time`) whose `supersedes` equals the underlying ProofBundle's tip hash AND whose signing arbitrator was authorized at `signing_time`, with chain integrity verified, is the **verification winner**.

Both H_c and R are preserved forever. "Override" is the precedence rule, not a mutation.

### 7.4 Why This Works

- Hash-chain immutability is preserved — no ProofBundle bytes are rewritten.
- Verification is offline and deterministic — any third party can run `verify_resolution` and reach the same EscrowDirective.
- Arbitrator authority is anchored to `signing_time`, not to verification time — so a later-revoked arbitrator's prior valid ruling remains valid.
- The chain becomes a precedent corpus indexed by Cart Mandate template hash (Section 15).

---

## 8. Wire Protocol

### 8.1 Message Types

ADRP defines six wire message types:

| # | Type | Emitter | Purpose |
|---|---|---|---|
| 1 | `DisputeFlag` | Agent | Advisory anomaly notification (non-binding) |
| 2 | `DisputeFiling` | Principal | Binding dispute initiation; posts filing fee |
| 3 | `EvidenceSubmission` | Either party | Hash-chained artifact append |
| 4 | `ArbitratorAssignment` | Registry | Binds arbitrator DID to dispute |
| 5 | `RulingBundle` | Arbitrator | Signed verdict + escrow directive (Section 7) |
| 6 | `EscrowDirective` | Verifier (derived) | Consumed by AP2 Payment Mandate or VCAP rail |

### 8.2 Common Message Envelope

Every ADRP message MUST include:

```json
{
  "msg_type": "<one of the six>",
  "msg_id": "<UUID>",
  "prev_hash": "<SHA-256 of prior chain event>",
  "submitter_did": "<DID>",
  "submitter_signature": "<Ed25519 base64 over JCS of preceding fields>",
  "timestamp": "<RFC3339>",
  "payload": { ... type-specific ... }
}
```

### 8.3 Canonicalization

All signatures MUST be over the JCS (RFC 8785) canonicalization of the signed object minus the signature field itself.

---

## 9. State Machine

### 9.1 States

```
Pre-FILED:    FLAGGED         (agent-emitted; expires if not ratified)
              FILED           (principal-emitted; fee posted)
                ↓
              ASSIGNED        (arbitrator bound to dispute)
                ↓
              EVIDENCE_OPEN   (parties submit evidence)
                ↓
              UNDER_REVIEW    (arbitrator deliberating)
                ↓
              RULED           (RulingBundle signed)
                ↓
              SETTLED         (EscrowDirective consumed by payment rail)

Terminal:     WITHDRAWN       (filer withdraws; fee partial-refunded)
              EXPIRED         (filing or evidence window expired)
```

### 9.2 Invariants

| ID | Invariant |
|---|---|
| I1 | PaymentMandate.escrow_state == HOLD until SETTLED, WITHDRAWN, or EXPIRED |
| I2 | No state mutates a prior attestation; transitions are new signed events appended to the dispute chain |
| I3 | `prev_hash` of every event = SHA-256 of canonical-JSON of the immediately prior event |
| I4 | `dispute_chain[0].prev_hash` == ProofBundle.tip_hash (anchors to Conduit chain) |
| I5 | Only an arbitrator authorized by the Arbitration Mandate's pool may emit RulingBundle |
| I6 | EscrowDirective MUST be derivable from (DisputeBundle, RulingBundle) by `verify_resolution` |

---

## 10. Tier Parameters

| Parameter | L1 | L2 | L3 |
|---|---|---|---|
| Dispute window | 0 (atomic) | 72h | 14d |
| Filing fee | N/A | 10% of value, $1-$100 | 10% of value, $50-$1,000 |
| Arbitrator SLA | N/A | 4h | 24h |
| Arbitrator pool | N/A | Curated AI panel (3 of 5) | Curated human pool (3 of 5) + optional legal review |
| Override threshold | N/A | 4-of-5 supermajority | 4-of-5 supermajority |
| Max txn auto-arb | unlimited | $1,000 | unlimited |
| Default if SLA missed | N/A | refund-to-buyer | refund-to-buyer |
| Frivolous slash | N/A | 100% fee forfeited | 100% fee forfeited |
| Novel-but-lost slash | N/A | 50% fee forfeited | 50% fee forfeited |
| Appeal window | N/A | 12h | 48h |
| Appeal panel | N/A | 5 (up from 3) | 5 + legal review |
| Appeal quorum | N/A | 4 of 5 | 4 of 5 |
| Escrow hold | 0 | 7 days | 21 days |
| Evidence file size cap | N/A | 10 MB total, max 20 files | 10 MB total, max 20 files |

### 10.1 Tier Routing

Tier is determined by the ATXN Bundle's Tier (per ATXN Section 5) AND by the transaction value relative to the `max_txn_auto_arb` ceiling. A Bundle declared L2 with transaction value > $1,000 MUST auto-escalate to L3.

### 10.2 Default Ruling on SLA Miss

If the arbitrator pool fails to produce a RulingBundle within the SLA, the dispute MUST default to `refund-to-buyer`. This default protects the principal-paying side and aligns the arbitrator pool's economic incentive (failure to rule = no fee earned + reputational impact).

---

## 11. Cart Mandate Acceptance Criteria

### 11.1 Required Field

The Cart Mandate (existing AP2 element) MUST contain a new field:

```json
"acceptance_criteria": {
  "schema_version": "1",
  "checks": [
    { "type": "regex", "selector": "$.output", "pattern": "..." },
    { "type": "json_schema", "selector": "$.deliverable", "schema": { ... } },
    { "type": "deterministic_function", "function_hash": "sha256:..." },
    { "type": "human_review_required", "reason": "..." }
  ],
  "default_window_hours": 72,
  "silence_equals": "release"
}
```

### 11.2 Check Types

| Type | Evaluation |
|---|---|
| `regex` | Deterministic regex match against JSON-path selector |
| `json_schema` | JSON Schema validation against selector |
| `deterministic_function` | Pure function (referenced by hash, fetched from registry) executed against deliverable |
| `human_review_required` | Auto-routes to arbitration; flags spec as deliberately not auto-evaluable |

### 11.3 Conduit Attestation Against Checks

Conduit attests against `acceptance_criteria.checks` at delivery time. If all checks pass deterministically, the dispute path auto-resolves cryptographically. If checks include `human_review_required` OR are missing, dispute auto-routes to L2 semantic arbitration.

### 11.4 The Authoring Helper (Normative for Deployment)

A natural-language-to-checks helper that converts a principal's English-language description into machine-checkable predicates is REQUIRED for v0.1 deployment. Its internals are out of scope for this specification, but its presence is normative: an ADRP deployment without a viable authoring helper will fail the >50% auto-resolution threshold and the protocol's unit economics will collapse on sub-$10 transactions.

This is the load-bearing operational dependency surfaced in the synthesis as the EpistemicAuditor / Archaeologist dissent.

---

## 12. Arbitrator Pool

### 12.1 v0.1: Curated Pool

v0.1 ships with a **SwarmSync-curated arbitrator pool** — a centralized list of vetted human-or-model arbitrators registered as DIDs in TRUSTED_REGISTRIES. The pool is published at a stable URI referenced by every Arbitration Mandate.

### 12.2 Selection Algorithm

Arbitrator selection MUST use a verifiable random function (VRF) seeded by the DisputeFiling's hash. The VRF output indexes into the curated pool. This produces deterministic-but-unpredictable selection — neither party can choose the arbitrator, but any third party can verify the selection was unbiased.

### 12.3 Pool Composition

| Tier | Pool Composition |
|---|---|
| L2 | AI arbitrator panels (model + curated checker) |
| L3 | Human arbitrators (vetted, jurisdictionally-licensed where relevant) + optional legal review |

### 12.4 Arbitrator Credentialing

Every arbitrator MUST hold a Verifiable Credential issued by a trusted registry that:
- Names the arbitrator's DID;
- States the arbitrator's `valid_from` and `valid_until` timestamps;
- Names the arbitrator's qualifications (jurisdictional bar membership for L3 humans; model card + provenance for L2 AI panels);
- Is signed by a registry in TRUSTED_REGISTRIES.

### 12.5 v0.2 Roadmap

Decentralized stake-weighted arbitrator pools are deferred to v0.2 pending legal review on securities classification, jurisdictional enforceability, and Sybil resistance.

---

## 13. Filing Fee Economics

### 13.1 Fee Structure

| Parameter | Value |
|---|---|
| Fee rate | 10% of transaction value |
| Floor | $1 (L2) / $50 (L3) |
| Ceiling | $100 (L2) / $1,000 (L3) |
| Denomination | USD (fiat or USDC stablecoin) |
| Refund | To prevailing party only |

### 13.2 Why Not a Tokenized Stake

The fee is **non-refundable to the filer**, **non-transferable**, and **denominated in USD**. It yields no return. It is not a security under Howey. This is a deliberate scoping decision per the ConstraintCartographer commitment in the synthesis: tokenized stakes trigger SEC scrutiny and would push v0.1 18+ months.

### 13.3 Slashing

| Outcome | Filer's Fee |
|---|---|
| Filer prevails | 100% refunded |
| Filer loses; novel evidence | 50% forfeited |
| Filer loses; frivolous (Conduit upheld, no new evidence) | 100% forfeited |
| Filer withdraws before EVIDENCE_OPEN | 75% refunded |
| Filer withdraws after EVIDENCE_OPEN | 25% refunded |

Forfeited fees MAY fund the arbitrator pool and a public dispute-quality dataset.

### 13.4 Arbitrator Compensation

Arbitrators are compensated 2% of transaction value (min $0.50) from the losing party's escrow. This sustains arbitrator economics on micro-disputes without additional protocol-level subsidy.

---

## 14. Appeals

### 14.1 Trigger

Either party MAY file an appeal within the tier-specific appeal window (12h L2, 48h L3). Appeals require a fresh filing fee at 1.5x the original rate.

### 14.2 Appeal Panel

A 5-arbitrator panel (up from the original 3-arbitrator quorum) selected via VRF from a different pool slice than the original ruling. Quorum: 4 of 5.

### 14.3 Finality

Appeal rulings are FINAL. There is no second appeal. Finality is a design feature.

### 14.4 Appeal Override

An appeal RulingBundle supersedes the original RulingBundle by the same signing-time precedence rule (Section 7). Both the original ruling and the appeal ruling are preserved forever.

---

## 15. Precedent Corpus

### 15.1 Indexing

Every signed RulingBundle is content-addressed and indexed by:
- Cart Mandate template hash;
- Dispute claim_code;
- Verdict;
- Arbitrator DID;
- Signing time.

### 15.2 Citation Requirement

Future arbitrators MUST query the precedent corpus for prior rulings on similar Cart Mandate templates and either CITE or DISTINGUISH them in their RulingBundle's `rationale_hash` content.

### 15.3 v0.1 vs v0.2

v0.1 ships the data layer (signed rulings, indexed, queryable by hash). v0.2 ships the search and citation infrastructure (semantic search, similarity scoring, citation graph). A pre-v0.1 commitment per the RemixForge dissent: ship the data-collection pipeline in v0.1 even if productized search is deferred. Otherwise the corpus accumulates in a schema that requires breaking changes.

---

## 16. Verification Algorithm

### 16.1 The `verify_resolution` Function

Any third party, given (DisputeBundle, RulingBundle), MUST be able to run `verify_resolution` offline and deterministically derive the same EscrowDirective.

```python
def verify_resolution(db: DisputeBundle, rb: RulingBundle) -> Result:
    # 1. Anchor checks
    assert db.conduit_proof_ref == rb.supersedes
    assert rb.prev_hash == db.chain_tip

    # 2. Chain integrity
    h = db.conduit_proof_ref
    for ev in [db.filing, *db.evidence_chain, db.arbitrator_assignment]:
        assert ev.prev_hash == h
        assert verify_sig(ev.submitter_did, ev.sig, jcs(ev))
        h = sha256(jcs(ev))
    assert h == db.chain_tip

    # 3. Arbitrator authority at signing_time (not now)
    vc = fetch_vc(rb.arbitrator_vc_hash)
    assert vc.subject == rb.arbitrator_did
    assert vc.issuer in TRUSTED_REGISTRIES
    assert vc.valid_from <= rb.signing_time <= vc.valid_until
    assert verify_sig(vc.issuer, vc.sig, jcs(vc))

    # 4. Ruling signature
    assert verify_sig(rb.arbitrator_did, rb.sig, jcs(rb_minus_sig))

    # 5. Verdict well-formed
    assert rb.verdict in {"release", "refund", "partial"}
    if rb.verdict == "partial":
        assert sum(rb.partial_split.values()) == 1.0

    # 6. Derive directive
    return Result(valid=True, escrow_directive=EscrowDirective(
        payment_mandate_ref=db.payment_mandate_ref,
        action=rb.verdict,
        split=rb.partial_split,
        ruling_ref=sha256(jcs(rb))
    ))
```

### 16.2 Determinism Requirement

Two honest verifiers given the same inputs MUST reach the same EscrowDirective. If they do not, the implementation is non-conformant.

### 16.3 Offline Requirement

`verify_resolution` MUST NOT require network access except to fetch the arbitrator's Verifiable Credential by hash. The VC fetch MAY be cached.

---

## 17. SDK Surface

A conformant ADRP v0.1 SDK MUST expose at minimum the following six methods, implementable in ≤500 LOC total:

```python
emit_flag(
    agent_standing_token,
    claim_code,
    evidence_ref
) -> DisputeFlag

file_dispute(
    payment_mandate,
    claim_code,
    claim_detail,
    flag_ref: Optional[str] = None
) -> DisputeBundle

submit_evidence(
    dispute_id,
    artifact_bytes,
    mime_type
) -> EvidenceLink

query_status(
    dispute_id
) -> {state, chain_tip, latest_event}

render_ruling(
    dispute_id,
    verdict,
    partial_split: Optional[Dict] = None,
    rationale: str = ""
) -> RulingBundle  # arbitrator-only

verify_resolution(
    dispute_bundle,
    ruling_bundle
) -> {valid, escrow_directive}
```

### 17.1 Appeal as Re-Entry

An appeal is `file_dispute` with the `prior_ruling_ref` field set. No new SDK primitive is required.

### 17.2 LOC Budget

Approximate breakdown:

| Method | LOC |
|---|---|
| `emit_flag` | ~30 |
| `file_dispute` | ~80 |
| `submit_evidence` | ~50 |
| `query_status` | ~30 |
| `render_ruling` | ~80 |
| `verify_resolution` | ~120 |
| Common envelope, signing, hash chain | ~110 |
| **Total** | ~500 |

### 17.3 Golden Test Cases

A conformant SDK MUST pass the following test cases:

| # | Case | Expected Behavior |
|---|---|---|
| 1 | `clear_release` | Conduit-attested, no flag, no filing, window expires → release |
| 2 | `clear_refund` | DisputeFiling with valid `bundle_integrity` claim, chain-break verified → refund |
| 3 | `partial_split` | Semantic dispute, RulingBundle with `partial_split: {to_buyer: 0.30, to_seller: 0.70}` → split |
| 4 | `attestation_override` | Original ProofBundle preserved; latest valid RulingBundle wins on signing_time precedence |
| 5 | `frivolous_dispute` | Filer loses without novel evidence → 100% fee forfeited |
| 6 | `malicious_requester_reject` | Principal-A files invalid claim, Conduit upheld → fee forfeited, escrow releases to seller |
| 7 | `expired_dispute` | DisputeFiling outside tier window → rejected with `EXPIRED` terminal state |
| 8 | `flag_ratification` | DisputeFlag emitted, principal files within window referencing flag → flag promoted to filing |
| 9 | `flag_expiration` | DisputeFlag emitted, no principal action within window → flag preserved but cannot anchor filing |
| 10 | `appeal_supersedes` | Appeal RulingBundle supersedes original RulingBundle by signing-time precedence |

---

## 18. Scope Locks for v0.1

The following scope locks are normative for v0.1 conformance per the ConstraintCartographer commitment:

| Lock | Reason |
|---|---|
| US-only B2B | No consumer / cross-border / Reg E / PSD2 / GDPR overlap |
| Partnered custody | No SwarmSync-held escrow; relies on Stripe Connect, Bridge, Modern Treasury, or equivalent |
| Curated arbitrator pool | No token-staked decentralized pool (Howey) |
| Filing fee, not stake | Non-refundable, non-transferable, USD-denominated |
| Reputation logged but not modulating | Avoids bootstrap-circular trust dependency in v0.1 |
| Per-principal preference model deferred | Data collection pipeline ships in v0.1; productized search ships in v0.2 |

---

## 19. Security Considerations

### 19.1 The >90% Auto-Resolution Target Is NOT a Protocol Property

The most dangerous claim in any ADRP deployment is that >90% of disputes will auto-resolve. This is a **deployment-conditional** target, not a protocol property. It depends on whether principals author Cart Mandates with sufficient `acceptance_criteria.checks` coverage.

Empirical base rates (Kleros, eBay, Upwork, Mechanical Turk) cap auto-resolution at 60-95%, with all systems hitting walls below 95%. Kleros data shows ~40% of cases turn on spec ambiguity. Two formal dissents (Archaeologist, EpistemicAuditor) preserved verbatim in the synthesis assign probabilities of 35-45% and 25% respectively to the median Cart Mandate having sufficient acceptance criteria.

Implementations MUST:
- Ship a normative NLP-to-checks authoring helper as load-bearing infrastructure;
- Run a 90-day shadow-mode pilot before GA;
- Measure check-coverage and auto-resolution rate as primary metrics;
- Pivot to mandatory `human_review_required` flag if check coverage is <30% at day 30.

### 19.2 The Bifurcation Seam

Adversaries will probe the seam between cryptographic-class and semantic-class disputes by filing claims structured as one but actually belonging to the other. The classification validation in Section 6.3 is REQUIRED, not optional. A non-validating implementation creates an attack surface that does not exist in unified-arbitration designs.

### 19.3 Arbitrator Collusion

A 4-of-5 threshold is compromisable via collusion of 4 arbitrators. The VRF selection algorithm (Section 12.2) reduces but does not eliminate this risk. Mitigations:

- Diversify the curated pool across non-correlated risk profiles (operator, jurisdiction, model provider);
- Monitor for arbitrator-pair correlation in rulings;
- Periodically audit a random sample of rulings against an independent panel.

### 19.4 Filing Fee as DoS Vector

A wealthy adversary can pay the filing fee repeatedly to harass a counterparty via dispute spam. Mitigations:
- Per-principal rate limit (max 0.5% of rolling 30-day transaction count);
- Frivolous slash (100% fee forfeit) accumulating to reputation impact in v0.2;
- Account suspension after N frivolous filings (operator policy).

### 19.5 Standing Token Replay

A revoked Standing Token MUST NOT anchor a new DisputeFiling. Verifiers MUST check the Standing Token revocation list at filing time. Cached revocation lists with a Profile-specific TTL are acceptable.

### 19.6 Time-Skew Attacks

The `signing_time` precedence rule (Section 7.3) is vulnerable to clock-skew attacks if any individual arbitrator can backdate a signature. Mitigations:
- Threshold-signature timestamps for L2 and L3 rulings (3-of-5 from a federated timestamping authority, per ATXN Section 9.3);
- Reject rulings with `signing_time` more than 5 minutes in the future relative to the verifier's clock.

### 19.7 Jurisdictional Enforceability of the Arbitration Mandate

The Arbitration Mandate's enforceability under FAA §2 has not been tested in court. The synthesis flagged this as a HIGH-confidence claim, but it is not an HIGH-confidence claim about court behavior. Implementations SHOULD seek a written legal opinion in the deployment jurisdiction before relying on the Arbitration Mandate to defeat a court filing.

### 19.8 Sanctions Screening

Every party receiving funds via an EscrowDirective MUST be screened against OFAC and equivalent sanctions lists. ADRP defines no specific screening mechanism, but a non-screening implementation is non-conformant for US deployment.

### 19.9 Principal Capacity Mid-Dispute

If a principal's capacity attestation expires or is revoked mid-dispute (e.g., sanctions added, corporate dissolution), the dispute MUST be halted and routed to a `capacity_dispute` (Section 6.4), bypassing ADRP's normal flow and defaulting to refund-to-buyer.

### 19.10 Acknowledged Residual Risks

- **Hallucinated mandates** (a Cart Mandate the principal did not actually intend, generated by a compromised or LLM-confused operator): partially mitigated by L3 epistemic attestation per ATXN, unsolved for L1/L2.
- **Adversarial sub-agency chains** (deep delegation graphs that obscure the originating principal): bounded by `sub_delegation_depth` per ATXN; ADRP routes sub-agency disputes to the originating principal.
- **Arbitrator pool capture** (curated pool dominated by a single operator): mitigated by VRF selection but not eliminated; v0.2 decentralization is the long-term mitigation.

---

## 20. IANA Considerations

This document requests IANA registration of the following:

### 20.1 ADRP JSON-LD Context

URI: `https://swarmsync.ai/spec/adrp/v1`

### 20.2 ADRP Claim Code Registry

A new IANA registry "ADRP Claim Codes" is requested. Initial entries:

| Code | Class | Description |
|---|---|---|
| `bundle_integrity` | cryptographic | SHA-256 chain breaks; signature verification fails |
| `mandate_scope` | cryptographic | Agent acted outside Cart Mandate scope |
| `token_authority` | cryptographic | Standing Token revoked or expired pre-execution |
| `timestamp_skew` | cryptographic | Attestation timestamps violate ordering invariants |
| `oracle_contradiction` | cryptographic | Third-party oracle data contradicts attestation |
| `quality_mismatch` | semantic | Deliverable doesn't satisfy acceptance_criteria.checks |
| `spec_ambiguity` | semantic | Acceptance criteria absent or under-specified |
| `timing_breach` | semantic | SLA missed |
| `fitness_for_purpose` | semantic | Deliverable formally compliant but unfit |

Registration policy: Specification Required (per RFC 8126).

### 20.3 ADRP Verdict Enum

A new IANA registry "ADRP Verdicts" is requested. Initial entries:

| Verdict | Effect |
|---|---|
| `release` | Funds released to seller (counterparty) |
| `refund` | Funds returned to buyer (filer) |
| `partial` | Funds split per `partial_split` (must sum to 1.0) |

### 20.4 ADRP Trusted Registries List

A new IANA registry "ADRP Trusted Arbitrator Registries" is requested for the initial list of registries permitted to issue arbitrator Verifiable Credentials.

---

## 21. Acknowledgements

This specification synthesizes the output of the Ultimate Brainstorm v2.2 Agent Dispute Resolution Protocol session (2026-04-25), which spawned a 9-agent panel through four phases (Independent Divergence, Adversarial Cross-Examination, Convergence, Steelman) plus a Crux Engine analysis.

The author thanks the panel agents — EpistemicAuditor, Archaeologist, Quantifier, ConstraintCartographer, socratic-mentor, DarkMirror, IdeaMatrix, RemixForge, SoSpec, SpiderSpark — for the convergent architecture.

Specific contributions preserved as load-bearing in this specification:
- **EpistemicAuditor:** the cryptographic / semantic bifurcation;
- **Archaeologist:** the empirical base-rate floor (Kleros 40% spec ambiguity, Upwork 5-day default-resolution, MTurk rejection abuse history);
- **Quantifier:** tier parameters (windows, fees, slashing);
- **ConstraintCartographer:** the Arbitration Mandate as FAA §2 anchor; v0.1 scope locks (US-only B2B, partnered custody, filing fee not stake);
- **socratic-mentor:** principal-only standing; the cryptographic / semantic / contractual taxonomy collapsed to two-way for v0.1;
- **DarkMirror:** the flag/file distinction; UMA optimistic-oracle pattern;
- **IdeaMatrix:** the tiered hybrid architecture;
- **RemixForge:** Cart Mandate acceptance_criteria as the spec-first resolution primitive;
- **SoSpec:** the wire protocol, state machine, SDK surface, and 28-line `verify_resolution` algorithm;
- **SpiderSpark:** the three-tier bonded-escalation structure; making the spec (not the execution) the disputed object.

Three formal dissents are preserved:
- **Archaeologist** — assigns <40% probability to the >90% auto-resolution target being met in v0.1.
- **EpistemicAuditor** — assigns 25% probability to median Cart Mandates having sufficient acceptance_criteria coverage to auto-resolve >50% of semantic disputes.
- **RemixForge** — deferring the per-principal preference model to v0.2 is a 70%-probability strategic mistake.

The author also acknowledges the prior decision-oracle synthesis on the A2A Transaction Definition (codified in ATXN draft-stone-atxn-00) which provides ADRP's substrate, and Paola Di Maio's prior critical review of the SwarmSync IETF Draft Stack (AIVS, VCAP, ATEP) which informed the cryptographic floor adopted here.

---

## 22. References

### 22.1 Normative References

- [RFC2119] Bradner, S., "Key words for use in RFCs to Indicate Requirement Levels", BCP 14, RFC 2119, March 1997.
- [RFC8174] Leiba, B., "Ambiguity of Uppercase vs Lowercase in RFC 2119 Key Words", BCP 14, RFC 8174, May 2017.
- [RFC8032] Josefsson, S. and I. Liusvaara, "Edwards-Curve Digital Signature Algorithm (EdDSA)", RFC 8032, January 2017.
- [RFC8785] Rundgren, A., et al., "JSON Canonicalization Scheme (JCS)", RFC 8785, June 2020.
- [RFC3339] Klyne, G. and C. Newman, "Date and Time on the Internet: Timestamps", RFC 3339, July 2002.
- [RFC8126] Cotton, M., et al., "Guidelines for Writing an IANA Considerations Section in RFCs", BCP 26, RFC 8126, June 2017.
- [W3C-VC-2.0] W3C, "Verifiable Credentials Data Model 2.0", https://www.w3.org/TR/vc-data-model-2.0/
- [W3C-DID] W3C, "Decentralized Identifiers (DIDs) v1.0", https://www.w3.org/TR/did-core/
- [draft-stone-atxn-00] Stone, B., "ATXN: Agent-to-Agent Transaction Definition Protocol", Internet-Draft, work in progress.
- [draft-stone-aivs-01] Stone, B., "AIVS: Agentic Integrity Verification Standard", Internet-Draft, work in progress.
- [draft-stone-vcap-01] Stone, B., "VCAP: Verified Commerce for Agent Protocols", Internet-Draft, work in progress.
- [draft-stone-atep-01] Stone, B., "ATEP: Agent Trust and Execution Passport", Internet-Draft, work in progress.

### 22.2 Informative References

- [FAA] Federal Arbitration Act, 9 U.S.C. §§ 1-16
- [NY-CONVENTION] Convention on the Recognition and Enforcement of Foreign Arbitral Awards, 1958
- [UETA-14] Uniform Electronic Transactions Act §14, "Automated Transactions"
- [REST-AGENCY-3D] Restatement (Third) of Agency, American Law Institute
- [HOWEY] SEC v. W. J. Howey Co., 328 U.S. 293 (1946)
- [AP2] Google Agentic Commerce, "Agent Payments Protocol", https://github.com/google-agentic-commerce/AP2
- [KLEROS] Kleros, "Decentralized Court Protocol", https://kleros.io
- [UMA] UMA Protocol, "Optimistic Oracle"
- [STRIPE-CONNECT] Stripe, "Connect Disputes and Chargebacks"
- [UPWORK-MEDIATION] Upwork, "Dispute Process for Hourly and Fixed-Price Contracts"
- [VISA-CHARGEBACK-CODES] Visa Inc., "Visa Chargeback Reason Codes"
- [OFAC-SANCTIONS] U.S. Treasury OFAC, "Specially Designated Nationals List"

---

## Appendix A: JSON Schemas (Informative)

### A.1 DisputeFiling

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://swarmsync.ai/spec/adrp/v1/dispute-filing.schema.json",
  "title": "DisputeFiling",
  "type": "object",
  "required": ["msg_type", "msg_id", "prev_hash", "submitter_did", "submitter_signature", "timestamp", "payload"],
  "properties": {
    "msg_type": { "const": "DisputeFiling" },
    "msg_id": { "type": "string", "format": "uuid" },
    "prev_hash": { "type": "string", "pattern": "^[0-9a-f]{64}$" },
    "submitter_did": { "type": "string" },
    "submitter_signature": { "type": "string" },
    "timestamp": { "type": "string", "format": "date-time" },
    "payload": {
      "type": "object",
      "required": ["payment_mandate_ref", "claim_code", "claim_detail", "filing_fee"],
      "properties": {
        "payment_mandate_ref": { "type": "string" },
        "claim_code": { "type": "string" },
        "claim_detail": { "type": "string", "maxLength": 4096 },
        "flag_ref": { "type": "string" },
        "prior_ruling_ref": { "type": "string" },
        "filing_fee": {
          "type": "object",
          "required": ["amount", "currency", "transaction_id"],
          "properties": {
            "amount": { "type": "number", "minimum": 1, "maximum": 1000 },
            "currency": { "enum": ["USD", "USDC"] },
            "transaction_id": { "type": "string" }
          }
        }
      }
    }
  }
}
```

### A.2 RulingBundle

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://swarmsync.ai/spec/adrp/v1/ruling-bundle.schema.json",
  "title": "RulingBundle",
  "type": "object",
  "required": ["type", "supersedes", "dispute_chain_tip", "verdict", "rationale_hash", "arbitrator_did", "arbitrator_vc_hash", "signing_time", "prev_hash", "sig"],
  "properties": {
    "type": { "const": "RulingBundle" },
    "supersedes": { "type": "string", "pattern": "^[0-9a-f]{64}$" },
    "dispute_chain_tip": { "type": "string", "pattern": "^[0-9a-f]{64}$" },
    "verdict": { "enum": ["release", "refund", "partial"] },
    "partial_split": {
      "type": "object",
      "properties": {
        "to_buyer": { "type": "number", "minimum": 0, "maximum": 1 },
        "to_seller": { "type": "number", "minimum": 0, "maximum": 1 }
      }
    },
    "rationale_hash": { "type": "string", "pattern": "^[0-9a-f]{64}$" },
    "arbitrator_did": { "type": "string" },
    "arbitrator_vc_hash": { "type": "string", "pattern": "^[0-9a-f]{64}$" },
    "signing_time": { "type": "string", "format": "date-time" },
    "prev_hash": { "type": "string", "pattern": "^[0-9a-f]{64}$" },
    "sig": { "type": "string" }
  }
}
```

(Full schemas for all six message types are published at the IANA-registered URI.)

---

## Appendix B: Worked Example (Informative)

### B.1 Scenario

A small business principal (`did:web:smb.example`) dispatches an agent to research and book a $250 hotel reservation. The agent uses Conduit to record its actions. The reservation is booked, but the hotel is in the wrong city — the agent confused "Portland, OR" with "Portland, ME" due to an ambiguous Cart Mandate that did not specify state.

### B.2 Sequence

1. **Agent emits DisputeFlag** (advisory): claim_code `quality_mismatch`, evidence_ref pointing at the reservation confirmation. The agent's `Standing Token` signs the flag.
2. **Principal reviews flag** within 72h ratification window. Principal agrees the booking is wrong.
3. **Principal emits DisputeFiling**: claim_code `quality_mismatch`, claim_detail "City mismatch: requested Portland, OR; booked Portland, ME", filing_fee $25.00 USDC, flag_ref points at the agent's flag.
4. **Arbitrator assigned** via VRF from L2 curated AI pool: `did:web:arb-pool-001`, 3-of-5 panel.
5. **EvidenceSubmission** by principal: original Cart Mandate showing "Portland" without state qualifier; Conduit ProofBundle showing agent's reasoning trace.
6. **EvidenceSubmission** by counterparty (hotel-side agent operator): the Cart Mandate's `acceptance_criteria.checks` is empty.
7. **Arbitrator panel deliberates**. Finding: spec ambiguity (semantic-class). Cart Mandate did not specify state; agent's choice was reasonable but principal's intent was OR. Verdict: `partial`, split 70/30 to buyer/seller (buyer bears 30% due to ambiguous spec).
8. **RulingBundle signed** by panel arbitrator. `supersedes` = ProofBundle tip hash. `signing_time` = within 4h SLA.
9. **EscrowDirective derived**: $175 to buyer, $75 to seller, ruling_ref = SHA-256 of RulingBundle.
10. **AP2 Payment Mandate executor consumes EscrowDirective**, executes split.
11. **Filing fee disposition**: principal prevailed on the existence of dispute but lost on full refund (received 70%, not 100%). Outcome classified as "novel-but-lost" since spec ambiguity was a real issue but not fully attributable to seller. 50% of fee forfeited ($12.50); 50% refunded ($12.50).

### B.3 Verification

Any third party, given the resulting (DisputeBundle, RulingBundle), can run `verify_resolution` and reach the same EscrowDirective deterministically. Both the original ProofBundle and the RulingBundle are preserved forever in the chain.

### B.4 Precedent

The RulingBundle is indexed in the precedent corpus by Cart Mandate template hash. Future arbitrators ruling on Cart Mandates with similar `acceptance_criteria.checks` gaps for location specification MUST cite or distinguish this ruling.

---

## Appendix C: Open Issues for Shadow-Mode Validation

The following questions are explicitly UNRESOLVED in v0.1 and MUST be answered by a 90-day shadow-mode pilot before GA:

| # | Question | Falsification Threshold |
|---|---|---|
| 1 | Will principals author Cart Mandates with ≥3 acceptance_criteria.checks of ≥3/5 quality? | <30% coverage at day 30 → ship NLP→checks helper as mandatory |
| 2 | Will the auto-resolution rate hit 60%+? | <60% at day 60 → escalate to architectural review |
| 3 | Will the frivolous filing rate stay below 2%? | >2% at day 60 → tighten filing fee floor |
| 4 | Will the curated arbitrator pool meet SLA at scale? | >5% SLA breaches at day 90 → expand pool or accelerate v0.2 |
| 5 | Will the Arbitration Mandate be challenged in court? | Any court challenge → seek immediate written legal opinion |
| 6 | Will the precedent corpus accumulate fast enough to be useful? | <100 corpus queries by arbitrators at month 3 → revisit RemixForge dissent |
| 7 | Are tier thresholds calibrated correctly? | Filing rate >5% (too low) or <0.05% (too high) → recalibrate fee structure |

GA is gated on:
- (auto-resolution rate ≥ 60%) AND
- (frivolous filing rate ≤ 2%) AND
- (zero court challenges to Arbitration Mandate) AND
- (arbitrator SLA breaches ≤ 5%) AND
- (median Cart Mandate has ≥ 1 non-trivial acceptance_criteria.check)

---

**End of draft-stone-adrp-00**

*Submitted to IETF Datatracker: pending*
*Companion drafts: ATXN-00, AIVS-01, VCAP-01, ATEP-01*
