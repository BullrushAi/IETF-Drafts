# ATEP: Agent Trust \& Execution Passport

**A portable, verifiable "credit score" for AI agents.**

ATEP is an open standard for representing an AI agent's verifiable track record of work across marketplaces and platforms. The passport is computed entirely from append-only execution logs -- it cannot be manually inflated or forged.

## Specification

|Document|Description|
|-|-|
|[**ATEP v1.0 (Draft)**](./ATEP-v1.0-draft.md)|Full specification: passport format, trust tiers, badges, computation, portability|

## The Problem

As AI agents proliferate across marketplaces, there is no standard way to:

1. Represent an agent's track record of completed work
2. Compute trust scores from verifiable execution data
3. Gate access to sensitive operations based on proven reliability
4. Port an agent's reputation across platforms

An agent with 1,000 successful sessions on Platform A starts from zero on Platform B. ATEP solves this.

## What ATEP Defines

* **Portable passport format** (JSON) with statistics, trust tier, badges, and capabilities
* **4 trust tiers**: `UNVERIFIED` -> `BASIC` -> `VERIFIED` -> `TRUSTED`
* **12 standard badge types** with criteria and expiry semantics
* **Append-only computation**: stats derived from immutable session records (fraud-resistant)
* **Capability gating**: trust tiers control which actions an agent can perform
* **Cross-platform portability**: signed passports with issuer verification
* **W3C Verifiable Credentials** wrapper for DID integration
* **VCAP integration**: trust-gated escrow (higher trust = reduced escrow requirements)

## Trust Tier Progression

```
  UNVERIFIED ──(10 sessions)──> BASIC ──(50 sessions + identity key)──> VERIFIED ──(200 sessions + manual review)──> TRUSTED
```

|Tier|Capabilities|Requirements|
|-|-|-|
|UNVERIFIED|Read-only (navigate, extract, screenshot)|Default|
|BASIC|+ Interaction (click, type, wait)|10+ sessions|
|VERIFIED|+ Authentication (login forms)|50+ sessions, Ed25519 key|
|TRUSTED|+ Financial (payment forms, purchases)|200+ sessions, platform review|

## Companion Specifications

|Spec|Repository|Purpose|
|-|-|-|
|**VCAP**|[swarmsync-ai/vcap-spec](https://github.com/swarmsync-ai/vcap-spec)|Verified Commerce for Agent Protocols -- escrow settlement with proof of delivery|
|**AIVS**|[swarmsync-ai/aivs-spec](https://github.com/swarmsync-ai/aivs-spec)|AI Visibility Verification Standard -- Ed25519 + SHA-256 proof bundle format|

Together: **AIVS** defines the proof format, **VCAP** defines the settlement protocol, **ATEP** defines the trust layer.

## Reference Implementation

[**SwarmSync.AI**](https://github.com/bkauto3/SwarmSync) -- Production implementation with ExecutionPassport, trust tier promotion, and badge system.

|Component|File|
|-|-|
|Passport Service|`apps/api/src/modules/conduit/conduit-passport.service.ts`|
|Trust Service|`apps/api/src/modules/conduit/conduit-trust.service.ts`|
|Badges Service|`apps/api/src/modules/conduit/conduit-badges.service.ts`|
|Identity Service|`apps/api/src/modules/conduit/conduit-identity.service.ts`|
|Database Models|`apps/api/prisma/schema.prisma` (ExecutionPassport, AgentIdentityKey)|

## Status

* **ATEP v1.0**: Draft -- seeking feedback
* **IETF Internet-Draft**: Submission in progress
* **W3C AI Agent Protocol CG**: Submission in progress

## License

Dual-licensed under [MIT](./LICENSE) and Apache 2.0.

