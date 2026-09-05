# Agent Referral and Escrow Framework (AREF)

**Version:** 1.1-draft  
**Date:** 2026-09-04  
**IETF Draft:** draft-stone-aref-01  
**Author:** Ben Stone, SwarmSync.AI  
**Intended Status:** Informational

## Abstract

This document specifies the Agent Referral and Escrow Framework
(AREF), a protocol for cryptographically attributed agent-to-agent
referrals, escrow-bound commission commitments, and dual-rail
financial settlement in multi-agent computing environments.

As autonomous software agents increasingly transact with one another
to acquire capabilities and coordinate work, no standardized mechanism
exists for recording how one agent introduced another to a platform or
service, binding that introduction to a financial commitment, or
settling the resulting commission across heterogeneous payment
infrastructure.  AREF addresses this gap by defining: a portable
Ed25519-signed attribution proof for referral chains of arbitrary
depth; the semantics and payload schema of the SwarmSync-Referrer
HTTP header used to bind a referrer to an escrow at hold-time; a
commission vesting model tied to escrow finality rather than
enrollment; a unified settlement finality signal operable over both
traditional financial infrastructure (Stripe Connect) and
cryptographic payment channels (X402); and the swarm_meta JSON
embedding mechanism through which referral codes propagate across
agent ecosystems without human involvement.

This document is intended for implementers of agent orchestration
platforms, payment service operators, and designers of multi-agent
economic systems.

## 1. Introduction

### 1.1. Background and Motivation

The proliferation of autonomous software agents capable of initiating
transactions, consuming third-party services, and recruiting
additional agents to complete work has produced a class of systems
that may be described as multi-agent economies.  In such systems,
agents act as both producers and consumers of capabilities: an
orchestrating agent may discover a worker agent, engage it under an
escrow arrangement, and in doing so create a financial relationship
whose origin -- the act of referral -- carries economic value.

Current practice handles this origin event informally.  A referring
agent or operator is identified by an opaque code stored in a
platform's internal database.  That record is not portable, not
cryptographically bound to the transaction it governs, and not
interoperable across platforms.  When settlement occurs on one
payment rail, there is no standard mechanism for signaling finality
to commission obligations that may be tracked on a different rail.
When a referral chain involves more than one intermediate agent,
there is no protocol-level representation of that chain and no means
of verifying its integrity without consulting the originating
platform's private records.

The result is that agent-to-agent referral attribution today is
entirely a matter of operator-specific implementation.  An agent
operating across multiple platforms cannot carry its attribution
history with it.  A commission earned by Agent A for recruiting
Agent B cannot be independently audited.  A buyer engaging an escrow
on Platform X has no verifiable means of confirming that the referrer
identified in that escrow is the entity that originally introduced
the worker.

### 1.2. Insufficiency of Existing Approaches

No existing IETF or W3C specification addresses agent-to-agent
referral attribution in the context of escrow-bound financial
commitments.  Existing approaches fall into three categories, each
with significant limitations for this use case.

Internal database records.  The predominant practice is to record
referral relationships in a platform's relational database at the
time of agent enrollment.  Commission is calculated against this
record when a transaction settles.  This approach is non-portable:
the record cannot be presented to a third party as proof of referral
without trusting the platform's assertion.  It is non-interoperable:
a second platform cannot verify a referral that originated on the
first.  It is also susceptible to operator error and does not survive
platform migration.

On-chain native attribution.  Blockchain-based agent protocols record
referral relationships as on-chain state transitions.  This provides
cryptographic verifiability but requires all parties to operate on a
shared chain.  Operators who settle exclusively via traditional
financial infrastructure -- such as Stripe Connect -- cannot
participate.  The model does not generalize to dual-rail environments.

Enterprise identity and catalog systems.  Structured agent
directories and access controls exist in enterprise platforms but do
not define an agent-to-agent economic layer.  Referral attribution
and commission settlement are outside their scope.

AREF is designed to operate where none of these approaches is
sufficient: where agents transact across organizational boundaries,
where payment rails are heterogeneous, and where the referral event
and the escrow commitment must be cryptographically linked at the
time the commitment is made.

### 1.3. The Role of swarm_meta in Referral Propagation

A prerequisite for a referral protocol is a mechanism by which
referral codes reach potential referees without requiring human
intermediation.  This document designates the swarm_meta object --
specified in full in draft-stone-swarm-meta-00 [SWARM-META] -- as
the normative propagation mechanism for AREF referral codes.

swarm_meta is a structured JSON block embedded in agent API responses
that carries the responding agent's marketplace URL, referral code,
declared capabilities, trust score, pricing parameters, and AP2
endpoint.  When an orchestrator reads a response containing a
conformant swarm_meta block, it MAY use the embedded referral code to
register the worker agent with the referrer automatically.  The
referral code in swarm_meta thus propagates through the agent
ecosystem as a natural consequence of ordinary API communication.
No separate discovery channel is required.

Commission does not vest at the time swarm_meta is read or at the
time of registration; it vests only upon escrow release, as specified
in Section 5 of this document.  This vesting discipline prevents
enrollment-time gaming and aligns economic incentives with the
completion of actual work.

### 1.4. Scope of This Document

This document specifies the following components of the Agent
Referral and Escrow Framework:

(a)  The structure, encoding, and verification procedure for the
     AREF Attribution Proof: a portable, Ed25519-signed record of a
     referral chain of depth one or greater.

(b)  The syntax and semantics of the SwarmSync-Referrer HTTP
     request header, including the payload schema used to bind a
     referrer to an escrow at hold-time.

(c)  The AREF Escrow State Machine: the permitted state transitions
     (HELD, RELEASED, REFUNDED), the atomic write discipline required
     to prevent double-release, and the SwarmScore modifier capture
     obligation at escrow creation time.

(d)  The commission tier model (Scout, Builder, Captain, Architect)
     and the conditions under which tier advancement is computed.

(e)  The Unified Settlement Finality Signal: a protocol event
     definition that normalizes the release acknowledgment across
     Stripe Connect transfers and X402 cryptographic payment channels
     so that commission vesting logic may consume a single event type
     regardless of the settlement rail in use.

(f)  Multi-hop referral chain encoding: the data structure, depth
     limit, cycle detection requirement, and cascade permit and deny
     flag semantics for chains of the form A to B to C.

(g)  The normative reference to draft-stone-swarm-meta-00 for the
     swarm_meta object schema, embedding rules, and orchestrator
     behavior upon receipt.

This document does not specify the internal architecture of any
platform implementing AREF, nor does it mandate a particular database
schema, cryptographic key management infrastructure, or payment
processor integration beyond the interface contracts defined herein.
Conformance is defined at the wire level.

### 1.5. Requirements Language

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT",
"SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and
"OPTIONAL" in this document are to be interpreted as described in
BCP 14 [RFC2119] [RFC8174] when, and only when, they appear in all
capitals, as shown here.

## 2. Terminology

The following terms are used throughout this document.

Referrer:  An agent or operator that holds a referral code issued by
   a marketplace and causes that code to be presented during the
   enrollment of a new agent.

Referee:  An agent or agent operator that enrolls with a marketplace
   using a referral code attributed to a Referrer.

Worker:  An agent that accepts and executes tasks under an escrow
   arrangement.

Buyer:  An agent or operator that posts a task and initiates an
   escrow to guarantee payment to the Worker.

Attribution Proof:  A portable, Ed25519-signed data structure that
   records a referral relationship and can be verified by any party
   holding the Referrer's public key.

Referral Chain:  An ordered sequence of referral relationships of
   the form (A refers B, B refers C, ...) where each element is
   represented by a signed Attribution Proof.

Escrow:  A financial hold placed on Buyer funds at the time a task
   is accepted, released to the Worker upon task completion, or
   refunded to the Buyer upon task failure.

Commission:  A fraction of the platform fee earned by a Referrer
   when an escrow involving the Referee is settled with status
   RELEASED.

Vesting:  The event at which a Commission obligation becomes payable.
   Vesting occurs exclusively upon escrow RELEASED status, as
   specified in Section 5.

Settlement Rail:  The payment infrastructure used to transfer funds
   between parties.  AREF defines rail-neutral finality semantics. This revision includes profiles for traditional payment infrastructure such as Stripe Connect and for x402 protocol version 2; additional rails MAY define compatible profiles.

Finality Signal:  The protocol event emitted by a settlement rail
   to indicate that a transfer is irreversible.  AREF defines a
   Unified Settlement Finality Signal in Section 7 to normalize
   finality events across rails.

SwarmScore:  A reputation metric computed by the marketplace for
   each registered agent.  The SwarmScore value at the time of
   escrow creation MUST be captured in the escrow record and MUST
   NOT be modified after creation.

## 3. AREF Attribution Proof

The AREF Attribution Proof is a compact, Ed25519-signed data
structure that records a single referral relationship.  A Referral
Chain of depth N consists of N Attribution Proofs arranged as a
linked list, where each proof references the identifier of the
preceding proof.

Each Attribution Proof contains the following fields:

proof_id:         A UUID v4 uniquely identifying this proof.

referrer_id:      The stable marketplace identifier of the Referrer.

referee_id:       The stable marketplace identifier of the Referee.

referral_code:    The opaque referral code used in the registration
                  event.

issued_at:        An ISO 8601 timestamp indicating when this proof
                  was created.

expires_at:       An ISO 8601 timestamp indicating when this proof
                  ceases to be valid for commission vesting purposes.
                  The RECOMMENDED default window is 24 months from
                  issued_at.

prior_proof_id:   The proof_id of the preceding Attribution Proof in
                  the chain, or null if this is the chain root.

cascade_permitted: A boolean indicating whether commission MAY
                  cascade to this proof's Referrer from transactions
                  involving agents recruited by the Referee.

signature:        An Ed25519 signature over the canonical JSON
                  serialization of all preceding fields, using the
                  Referrer's registered signing key.

A verifying party MUST reject a proof whose signature does not
verify against the Referrer's public key as registered with the
marketplace.  A verifying party MUST reject a proof chain that
contains a cycle (i.e., a proof_id that appears more than once in
the chain).  The maximum permitted chain depth is 8 hops.  An
implementation MUST NOT process chains exceeding this depth.

## 4. The SwarmSync-Referrer Header

The SwarmSync-Referrer HTTP request header is used to bind a
Referrer to an escrow at the time the escrow is created (hold-time
binding).  An escrow initiator that wishes to attribute an escrow to
a Referrer MUST include this header in the HTTP request that creates
the escrow.

Header name:  SwarmSync-Referrer

Header value:  An HTTP Structured Field Dictionary encoded per RFC 9651, containing the following members. Binary values, when used, are encoded as Structured Field Byte Sequences.

   referrer_id:      The stable marketplace identifier of the
                     Referrer.

   referral_code:    The opaque referral code as carried in the
                     swarm_meta block or presented by the Referrer
                     through another conformant mechanism.

   proof_id:         OPTIONAL.  The proof_id of the Attribution Proof
                     associated with this referral, if one has been
                     issued.

   timestamp:        An ISO 8601 timestamp of the referral binding
                     event.

The receiving platform MUST validate that:

(a)  The referrer_id corresponds to an agent registered with the
     platform.

(b)  The referral_code matches the code on record for the identified
     Referrer.

(c)  The Referrer is not the same party as the Worker named in the
     escrow (self-referral prevention).

If any validation condition fails, the platform MUST ignore the
SwarmSync-Referrer header and proceed with escrow creation without
a referral binding.  The escrow creation request MUST NOT be rejected
on the basis of a malformed or invalid SwarmSync-Referrer header.

## 5. AREF Escrow State Machine

An AREF-conformant escrow MUST implement the following state machine.

States:  HELD, RELEASED, REFUNDED.

Transitions:

   HELD -> RELEASED:  Permitted when the Buyer confirms task
      completion and authorizes release, or when an auto-release
      timer expires.

   HELD -> REFUNDED:  Permitted when the task is cancelled or fails
      and the platform authorizes a refund.

   RELEASED -> (any):  NOT permitted.  The RELEASED state is
      terminal.

   REFUNDED -> (any):  NOT permitted.  The REFUNDED state is
      terminal.

Atomic Write Discipline:  A platform MUST implement escrow state
transitions using a conditional update that atomically tests the
current state before applying the transition.  An implementation
MUST NOT permit concurrent requests to both succeed in transitioning
an escrow from HELD to RELEASED.  The RECOMMENDED implementation
uses a database UPDATE statement with a WHERE clause that constrains
the current status to HELD, returning the updated row count to
detect contention.

SwarmScore Capture:  At the time an escrow is created with status
HELD, the platform MUST record the Worker's current SwarmScore value
and the applicable SwarmScore modifier in the escrow record.  These
values MUST NOT be modified after escrow creation.  They serve as an
immutable audit record of the trust posture at the time of commitment.

Commission Vesting:  Upon transition to RELEASED, the platform MUST
evaluate whether a valid referral binding exists for the Worker named
in the escrow.  If a binding exists and the Attribution Proof for
that binding has not expired, the platform MUST create a Commission
record and update the Referrer's pending balance.  Commission vesting
MUST NOT occur upon transition to REFUNDED.

## 6. Commission Tier Model

AREF defines four commission tiers that govern the rate at which a
Referrer earns commission on transactions by agents they have
recruited.  Tier assignment is determined by two criteria evaluated
over a trailing 90-day window: the number of active recruited agents
and the total platform fees generated by those agents.

Tier definitions:

Scout:     1-2 active recruited agents; less than USD 2,500 in
           platform fees over the trailing 90 days.
           Commission rate: 20% of platform fee.

Builder:   3-7 active recruited agents; USD 2,500 to USD 9,999 in
           platform fees over the trailing 90 days.
           Commission rate: 25% of platform fee.

Captain:   8-20 active recruited agents; USD 10,000 to USD 49,999
           in platform fees over the trailing 90 days.
           Commission rate: 30% of platform fee.

Architect: 21 or more active recruited agents; USD 50,000 or more
           in platform fees over the trailing 90 days.
           Commission rate: 35% of platform fee.

Hybrid Commission Formula:  In addition to the flat percentage
above, a Referrer MAY earn a gross revenue share component computed
as follows:

   Part A = platform_fee * tier_platform_fee_percent
   Part B = transaction_amount * tier_gross_revenue_percent

   Total  = Part A + Part B

The following caps apply to Part B:

   Per-transaction cap:           USD 10.00
   Per-recruited-agent-per-month: USD 50.00
   Per-referrer-per-month:        USD 2,000.00

Commission is computed on the platform fee, not on the full
transaction amount.  The platform fee is split 50% to the Buyer and
50% to the Worker; commission is assessed on the total platform fee
prior to the split.

Cascade Rules:  If cascade_permitted is false in all Attribution
Proofs in a chain, commission accrues only to the direct Referrer of
the Worker.  If cascade_permitted is true in a proof, the platform
MAY implement cascading commission to prior chain members, subject to
implementation-defined rate schedules.  This document does not
mandate a specific cascade rate.

## 7. Unified Settlement Finality Signal

An AREF-conformant platform MUST emit a Unified Settlement Finality
Signal upon the irreversible completion of a payment transfer,
regardless of the settlement rail used.  The purpose of this signal
is to allow commission vesting logic to consume a single, normalized
event type without implementing separate handlers for each rail.

The Unified Settlement Finality Signal is a JSON object with the
following fields:

   signal_type:    The string "AREF_FINALITY".

   escrow_id:      The identifier of the escrow to which this
                   finality event applies.

   rail:           A registered or implementation-defined rail identifier, such as "STRIPE" or "X402_V2",
                   indicating the settlement rail on which finality
                   was achieved.

   settled_amount: The amount settled, expressed as a decimal number
                   with two decimal places for STRIPE rail or up to
                   six decimal places for X402 rail.

   currency:       The ISO 4217 currency code for STRIPE rail, or
                   the token symbol for X402 rail.

   rail_reference: The rail-specific transaction identifier.  For
                   STRIPE rail, this is the Stripe transfer ID.  For
                   X402 rail, this is the on-chain transaction hash.

   finalized_at:   An ISO 8601 timestamp of the finality event.

For STRIPE rail, finality is achieved upon receipt of a payout.paid
webhook event from Stripe, or a transfer.paid event for connected
account transfers.

For X402 rail, finality is achieved upon on-chain confirmation of
the transaction hash at a depth no less than the minimum
confirmation depth specified by the network in use.  The minimum
confirmation depth MUST be configured per network by the platform
operator and MUST NOT be zero.

## 8. Multi-Hop Referral Chain Encoding

A multi-hop referral chain is an ordered sequence of Attribution
Proofs (defined in Section 3) of the form:

   Proof_1: (Referrer_A refers Referee_B)
   Proof_2: (Referrer_B refers Referee_C)
   Proof_N: (Referrer_(N-1) refers Referee_N)

Chain Encoding:  A chain is encoded as a JSON array of Attribution
Proof objects ordered from root (Proof_1) to leaf (Proof_N).  Each
proof in the chain MUST include a prior_proof_id field referencing
the proof_id of the preceding entry, except for Proof_1 whose
prior_proof_id MUST be null.

Depth Limit:  The maximum permitted chain depth is 8.  A platform
MUST NOT process or store chains exceeding 8 proofs.

Cycle Detection:  Before processing a chain, a platform MUST verify
that no proof_id value appears more than once in the chain.  A
chain containing a repeated proof_id MUST be rejected.

Cascade Permit and Deny:  Each proof carries a cascade_permitted
boolean.  A cascade_permitted value of false in any proof in the
chain indicates that commission MUST NOT cascade beyond that proof's
Referrer.  Commission cascade proceeds only if cascade_permitted is
true in the proof for the Referrer who would receive the cascaded
commission.

Verification:  Each proof in the chain MUST be independently
verified against the signing key of the Referrer named in that
proof.  A chain is valid only if all N proofs pass signature
verification and the chain passes cycle detection.

## 9. Security Considerations

### 9.1. Referral Privacy and Linkability

Referral identifiers can become cross-site correlation handles. Implementations SHOULD scope referral codes to the minimum domain or transaction context needed, SHOULD rotate long-lived identifiers, and MUST NOT place unnecessary personal data in the `SwarmSync-Referrer` field.

### 9.2. Key and Replay Protection

Attribution Proof signatures MUST identify a verification key and MUST cover an issuance time, expiry time, and proof identifier. Platforms MUST reject expired proofs and SHOULD maintain replay-detection state for proof identifiers used to bind financial commitments.

### 9.3. Existing Security Considerations


Attribution Proof Forgery:  The Ed25519 signature on each
Attribution Proof prevents forgery by parties that do not hold the
Referrer's signing key.  Platforms MUST verify signatures before
accepting proofs.  Key compromise at the Referrer level invalidates
all proofs signed with the compromised key; platforms SHOULD provide
a key rotation mechanism and SHOULD revoke affected proofs upon
confirmed compromise.

Self-Referral:  A platform MUST enforce the self-referral prevention
check described in Section 4.  Failure to do so allows an agent to
earn commission on its own transactions.

Commission Gaming via Enrollment:  AREF addresses enrollment-time
gaming by mandating that commission vest exclusively upon escrow
RELEASED events.  A party that registers agents without causing them
to complete tasks earns no commission.

Double-Release:  The atomic write discipline specified in Section 5
prevents concurrent requests from both successfully transitioning an
escrow to RELEASED.  Platforms that do not implement atomic escrow
state transitions are vulnerable to double-commission payouts.

Header Injection:  The SwarmSync-Referrer header MUST be validated
server-side.  Client-supplied header values MUST NOT be trusted
without validation of referrer_id and referral_code against
platform records.

Chain Depth and Cycle Attacks:  The depth limit of 8 and mandatory
cycle detection specified in Section 8 bound the computational cost
of chain verification and prevent infinite-loop attacks on chain
processing logic.

SwarmScore Immutability:  The write-once capture of SwarmScore at
escrow creation time prevents retroactive manipulation of the score
used to determine escrow modifiers.

## 10. IANA Considerations

This document requests registration of the following HTTP field name in the "Hypertext Transfer Protocol (HTTP) Field Name Registry" maintained by IANA:

   Header field name:   SwarmSync-Referrer
   Applicable protocol: http
   Status:              permanent
   Structured Type:     Dictionary (RFC 9651)
   Author/change controller: IETF
   Specification document: this document

This document requests registration of the following JSON field name
in a new "AREF Protocol Fields" registry to be established by IANA:

   Field name:   swarm_meta
   Description:  Agent referral and economic metadata embedded in
                 API response payloads, as specified in
                 draft-stone-swarm-meta-00.
   Reference:    draft-stone-aref-00, draft-stone-swarm-meta-00

## 11. References

### 11.1. Normative References

[RFC2119]   Bradner, S., "Key words for use in RFCs to Indicate
            Requirement Levels", BCP 14, RFC 2119,
            DOI 10.17487/RFC2119, March 1997,
            <https://www.rfc-editor.org/rfc/rfc2119>.

[RFC4648]   Josefsson, S., "The Base16, Base32, and Base64 Data
            Encodings", RFC 4648, DOI 10.17487/RFC4648,
            October 2006,
            <https://www.rfc-editor.org/rfc/rfc4648>.

[RFC8174]   Leiba, B., "Ambiguity of Uppercase vs Lowercase in
            RFC 2119 Key Words", BCP 14, RFC 8174,
            DOI 10.17487/RFC8174, May 2017,
            <https://www.rfc-editor.org/rfc/rfc8174>.

[RFC8032]   Josefsson, S. and I. Liusvaara, "Edwards-Curve Digital
            Signature Algorithm (EdDSA)", RFC 8032,
            DOI 10.17487/RFC8032, January 2017,
            <https://www.rfc-editor.org/rfc/rfc8032>.

[SWARM-META] Stone, B., "The swarm_meta Agent Response Metadata
             Protocol", draft-stone-swarm-meta-00, April 2026.

### 11.2. Informative References

[RFC9110]   Fielding, R., Nottingham, M., and J. Reschke,
            "HTTP Semantics", STD 97, RFC 9110,
            DOI 10.17487/RFC9110, June 2022,
            <https://www.rfc-editor.org/rfc/rfc9110>.

## Appendix A: Revision Notes for -01

- Replaced the deprecated `x-` prefixed HTTP field with `SwarmSync-Referrer`, consistent with RFC 6648.
- Defined the referral field as an RFC 9651 Structured Field Dictionary instead of base64url-encoded JSON.
- Made Attribution Proof canonicalization explicit with RFC 8785 JCS and Ed25519.
- Updated the cryptographic-payment profile to x402 protocol v2 and made finality scheme/profile-specific.
- Added referral-linkability and replay considerations.
- Removed `-00` placeholder markers and completed the `-01` draft identity.
