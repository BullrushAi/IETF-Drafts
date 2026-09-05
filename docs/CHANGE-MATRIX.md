# Change Matrix

## Why this is more than an expiration refresh

Several upstream standards and internal SwarmSync specifications evolved after the March/April submissions. Renewing the old text unchanged would preserve known inconsistencies for another six months.

| Standard | Refresh type | Main changes |
|---|---|---|
| AIVS | Substantive | Canonical JSON v1.1 retained; Data Integrity/EdDSA alignment added. |
| VCAP-AP2 Binding | Rewrite | Rebased on current AP2 mandate/authorization model; removed obsolete PaymentIntent escrow-state assumptions. |
| SwarmScore V1 | Substantive | Formula/tier thresholds unchanged; HMAC passport signing replaced by Ed25519 + JCS; governance additions retained. |
| SwarmScore V2 Canary | Substantive | Five-pillar formula retained; Ed25519/JCS, canary-result provenance, governance/legal claim hardening. |
| VCAP | Substantive | Current AP2 positioning, x402 v2, verifier key identifier/version pinning. |
| ATEP | Substantive | DataIntegrityProof/eddsa-jcs-2022, key-separation and credential-status/privacy corrections. |
| AREF | Substantive | HTTP field modernized from X-prefix to SwarmSync-Referrer Structured Field; Ed25519/JCS; x402 v2. |
| ATXN | Substantive | AP2/UCP and x402 v2 alignment; crypto canonicalization clarified; ADRP arbitration-mandate anchoring aligned. |
| ADRP | Substantive | AP2 relationship corrected; safer timeout/default behavior; legal/security claims hardened; ATXN-01 alignment. |

## Cross-stack rules applied

### 1. Publicly verifiable signatures
Where a portable proof/passport is intended to be verified by unrelated parties, this refresh prefers **Ed25519** with deterministic **RFC 8785 JCS** serialization instead of a shared HMAC secret.

### 2. Current AP2 semantics
AP2 is treated as an **authorization/mandate and security layer** used with a commerce protocol, not as an invented escrow `PaymentIntent` state machine. The VCAP–AP2 binding therefore binds VCAP settlement evidence to AP2 mandates by stable reference/hash instead of pretending AP2 owns VCAP settlement states.

### 3. Current x402 generation
References are updated to the current x402 v2 model where x402 is used as a payment profile.

### 4. W3C VC proof modernization
ATEP uses `DataIntegrityProof` with the EdDSA/JCS cryptosuite model rather than the legacy `Ed25519Signature2020` example.

### 5. Safer normative/legal language
Categorical legal conclusions, fixed insurance requirements, unsupported empirical claims, and protocol-triggered financial defaults that could themselves create attack surfaces were narrowed or made deployment-policy choices.

## SwarmScore V1: intentionally unchanged economics

Unchanged:
- 90-day measurement window
- Technical Execution maximum: 400
- Commercial Reliability maximum: 600
- Total score: 0–1000
- Escrow modifier curve and 0.25 floor
- STANDARD: score >= 700 plus volume floors
- ELITE: score >= 850 plus volume floors

Changed:
- HMAC-SHA256 portable passport signature -> Ed25519 + JCS
- `key_id`/verification-key resolution added
- key separation/rotation guidance hardened
- V2 Canary reference advanced to `-01`
- existing community-governed canary roadmap retained

## SwarmScore V2 Canary: intentionally unchanged five-pillar allocation

Unchanged:
- Technical Execution 300
- Commercial Reliability 300
- Operational Depth 150
- Safety 100
- Identity Verification 150
- Total 1000
- existing safety-score and tier framework

Changed:
- portable signing aligned with V1 Ed25519/JCS
- cryptographic provenance fields for canary results
- canary library governance moved toward multi-stakeholder control
- fixed cost/insurance assumptions removed from protocol conformance
- categorical legal claims softened
- unsupported “most agents score above 75” onboarding assertion removed
- references advanced to refreshed companion drafts
