# AIVS: AI Visibility Verification Standard

**The proof format that makes AI agent sessions tamper-evident and self-verifiable.**

AIVS defines a portable, self-verifiable archive format for cryptographic proof of AI agent sessions. An AIVS bundle is a gzip-compressed tar archive containing a SHA-256 hash-chained audit log, an Ed25519 digital signature, a machine-readable manifest, and an embedded verification script requiring only Python 3 standard library.

## Specification

| Document | Description |
|----------|-------------|
| **[AIVS v1.0 (Draft)](./AIVS-v1.0-draft.md)** | Full specification: proof bundle format, hash chain construction, Ed25519 signing, AIVS-Micro attestations |

## The Problem

When an AI agent performs a browser session, API call, or any autonomous task, there is no standard way to:

1. Prove every action was recorded and unmodified (integrity)
2. Prove no actions were inserted, deleted, or reordered (sequencing)
3. Prove the session was produced by a specific cryptographic identity (attribution)
4. Verify all of the above offline, without any external dependencies (self-verification)

AIVS solves all four.

## What AIVS Defines

### Proof Bundle Contents

```
conduit_proof_<session>_<timestamp>.tar.gz
  ├── audit_log.jsonl       # SHA-256 hash-chained action log (one entry per line)
  ├── manifest.json         # Session metadata, timestamps, summary statistics
  ├── public_key.pem        # Agent's Ed25519 public key (SPKI format)
  ├── session_sig.txt       # Ed25519 signature over the final chain hash
  └── verify.py             # Zero-dependency verification script (Python 3 stdlib only)
```

### Hash Chain Construction

```
hash_0 = SHA-256(json(action_0))
hash_1 = SHA-256(json(action_1) || hash_0)
hash_2 = SHA-256(json(action_2) || hash_1)
...
hash_n = SHA-256(json(action_n) || hash_(n-1))
```

Modifying any past action immediately breaks all subsequent hashes.

### AIVS-Micro

A minimal 6-field attestation (~200 bytes) for embedded widgets, API responses, and continuous monitoring where a full bundle is not practical.

## Companion Specifications

| Spec | Repository | Purpose |
|------|------------|---------|
| **VCAP** | [swarmsync-ai/vcap-spec](https://github.com/swarmsync-ai/vcap-spec) | Verified Commerce for Agent Protocols -- uses AIVS proof bundles for escrow settlement |
| **ATEP** | [swarmsync-ai/atep-spec](https://github.com/swarmsync-ai/atep-spec) | Agent Trust & Execution Passport -- AIVS-verified sessions contribute to trust scores |

Together: **AIVS** defines the proof format, **VCAP** defines the settlement protocol, **ATEP** defines the trust layer.

## Reference Implementation

**[Conduit](https://github.com/bkauto3/Conduit)** -- Headless browser with SHA-256 hash chain + Ed25519 audit trails. MCP server for AI agents.

## Status

- **AIVS v1.0**: Draft
- **W3C Submission**: In progress
- **IETF Internet-Draft**: Planned

## License

Dual-licensed under [MIT](./LICENSE) and Apache 2.0.
