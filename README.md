# IETF Internet-Drafts — Ben Stone

Nine independent-submission Internet-Drafts plus companion specifications specifying open protocols for agent-to-agent (A2A) commerce, trust, and verification — reputation scoring, escrow-bound transactions, dispute resolution, referral attribution, and cryptographic proof of agent work. Each draft follows standard IETF format (`.txt`/`.xml` per RFC 7749/xml2rfc, plus a readable `.md` working draft) and lists Ben Stone (SwarmSync.AI) as author.

Live status for every draft: [datatracker.ietf.org](https://datatracker.ietf.org) (see links below). Supporting JSON Schemas and example payloads referenced by the passport/escrow/verification specs are included under [`schemas/`](./schemas) and [`examples/`](./examples).

## Drafts

| Spec | Draft | What it defines |
|---|---|---|
| **ATEP** | [draft-stone-atep](https://datatracker.ietf.org/doc/draft-stone-atep/) | Agent Trust & Execution Passport — a portable, verifiable "credit score" for AI agents, computed from append-only execution logs, with trust tiers that gate what an agent is allowed to do |
| **AIVS** | [draft-stone-aivs](https://datatracker.ietf.org/doc/draft-stone-aivs/) | Agentic Integrity Verification Specification — a self-verifiable, portable archive format for cryptographic proof of an AI agent session |
| **VCAP** | [draft-stone-vcap](https://datatracker.ietf.org/doc/draft-stone-vcap/) | Verified Commerce for Agent Protocols — settling payments between autonomous agents using cryptographically verifiable proof of work delivery |
| **VCAP–AP2 Binding** | [draft-stone-vcap-ap2-binding](https://datatracker.ietf.org/doc/draft-stone-vcap-ap2-binding/) | Binds VCAP's settlement model to Google's AP2 agent payments protocol |
| **SwarmScore V1** | [draft-stone-swarmscore-v1](https://datatracker.ietf.org/doc/draft-stone-swarmscore-v1/) | Volume-scaled agent reputation protocol combining technical execution and commercial reliability into one open, community-governed score |
| **SwarmScore V2 Canary** | [draft-stone-swarmscore-v2-canary](https://datatracker.ietf.org/doc/draft-stone-swarmscore-v2-canary/) | Extends SwarmScore with a Safety dimension, measured via covert canary-prompt testing |
| **ATXN** | [draft-stone-atxn](https://datatracker.ietf.org/doc/draft-stone-atxn/) | Agent-to-Agent Transaction Definition Protocol — the transaction, escrow, dispute, audit, and liability model underlying agent commerce |
| **ADRP** | [draft-stone-adrp](https://datatracker.ietf.org/doc/draft-stone-adrp/) | Agent Dispute Resolution Protocol — arbitration and escrow-backed dispute handling between transacting agents |
| **AREF** | [draft-stone-aref](https://datatracker.ietf.org/doc/draft-stone-aref/) | Agent Referral and Escrow Framework — cryptographically attributed agent-to-agent referrals with commission escrow |
| **SWARM-META** | (independent submission) | `swarm_meta` — a structured JSON metadata block embedded in agent API responses for self-description and referral attribution |

## How these fit together

**AIVS** defines the cryptographic proof format for an agent session. **ATEP** turns a history of those proofs into a portable trust score. **VCAP** and **ATXN** define how agents actually transact and settle payment on top of that trust, with **ADRP** and **AREF** handling disputes and referral commissions. **SwarmScore** is an alternate, community-governed reputation model over the same execution data. **SWARM-META** is the wire-format glue that lets agents advertise all of this inline in their own API responses.

## Repository layout

```
atep/          ATEP drafts (-00, -01, -02) + working spec
aivs/          AIVS drafts (-00, -01) + working spec
vcap/          VCAP (-00, -01, -02) + VCAP-AP2 binding drafts and working specs
swarmscore/    SwarmScore V1 and V2 Canary drafts
atxn/          ATXN drafts and working spec
adrp/          ADRP drafts and working spec
aref/          AREF drafts and working spec
swarm-meta/    SWARM-META draft
schemas/       JSON Schemas for passports, escrow, negotiation, verification callbacks
examples/      Example payloads conforming to the schemas above
docs/          Standards refresh change matrix and technical notes
```

## License

Dual-licensed under [MIT](./LICENSE) and Apache 2.0.

## Author

**Ben Stone** — [benstone-ai.com](https://benstone-ai.com) · [github.com/BenStoneAI](https://github.com/BenStoneAI)
