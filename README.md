# MESH Protocol

> **Memory Exchange & Sharing Hub** — Secure federation protocol for AMP nodes

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![Spec Version](https://img.shields.io/badge/spec-v0.1--draft-orange.svg)](SPECIFICATION.md)
[![Security](https://img.shields.io/badge/security-hardened-green.svg)](SECURITY.md)

---

## What is MESH?

MESH is a federation protocol that connects [AMP](https://github.com/mmorris35/agent-memory-protocol) (Agent Memory Protocol) nodes into a secure, decentralized knowledge network.

**AMP** defines how agents store and retrieve memories locally.  
**MESH** defines how those memories can be selectively shared across a trusted network.

```
┌─────────────────────────────────────────────────────────────────┐
│                         MESH Network                            │
│                                                                 │
│    ┌─────────┐         ┌─────────┐         ┌─────────┐        │
│    │  Node A │◄───────►│  Node B │◄───────►│  Node C │        │
│    │  (AMP)  │  signed │  (AMP)  │  signed │  (AMP)  │        │
│    └────┬────┘ lessons └────┬────┘ lessons └────┬────┘        │
│         │                   │                   │              │
│    ┌────┴────┐         ┌────┴────┐         ┌────┴────┐        │
│    │ private │         │ private │         │ private │        │
│    │ + public│         │ + public│         │ + public│        │
│    └─────────┘         └─────────┘         └─────────┘        │
│                                                                 │
│                    ┌──────────────────┐                        │
│                    │    Directory     │                        │
│                    │  (decentralized) │                        │
│                    └──────────────────┘                        │
└─────────────────────────────────────────────────────────────────┘
```

## Design Principles

### 1. Security First
Every design decision prioritizes security. Trust is cryptographically verified, never assumed.

### 2. Private by Default
All memories are private until explicitly published. No accidental leakage.

### 3. Decentralized
No single point of control or failure. Directories are federated. Nodes are sovereign.

### 4. Verifiable
Every shared lesson is signed. Provenance is cryptographically provable.

### 5. Revocable
Published can become unpublished. Revocations propagate through the network.

## Core Features

| Feature | Description |
|---------|-------------|
| **Signed Lessons** | Ed25519 signatures on all shared content |
| **Node Identity** | Cryptographic identity, not just URLs |
| **Visibility Controls** | `private` / `unlisted` / `public` per record |
| **Federated Directory** | Decentralized discovery, no central authority |
| **Cross-Node Search** | Query multiple nodes, results ranked by trust |
| **Web of Trust** | Nodes vouch for nodes, reputation emerges |
| **Revocation** | Signed revocation records propagate |
| **Selective Sync** | Subscribe to topics, nodes, or search queries |

## Quick Overview

### Publishing a Lesson

```bash
# On your AMP node
amp publish lesson_abc123 --visibility public

# Behind the scenes:
# 1. Lesson content is hashed
# 2. Hash is signed with your node's private key
# 3. Signed lesson is announced to your MESH peers
# 4. Peers propagate to their peers (gossip)
# 5. Directories index the public record
```

### Federated Search

```bash
mesh search "cargo parallel build crashes" --sources trusted

# Behind the scenes:
# 1. Query sent to your trusted MESH peers
# 2. Each peer searches their local AMP + asks their peers
# 3. Results returned with signatures and trust scores
# 4. Your node verifies signatures, ranks by trust
# 5. Deduplicated, sorted results presented
```

### Verifying a Lesson

```bash
mesh verify lesson_abc123@nodeB

# Outputs:
# ✓ Signature valid (ed25519)
# ✓ Author: nodeB (fingerprint: 7F3A...)
# ✓ Timestamp: 2026-02-06T13:00:00Z
# ✓ Not revoked
# ✓ Trust path: you → nodeA → nodeB (2 hops)
```

## Documentation

| Document | Description |
|----------|-------------|
| [SPECIFICATION.md](SPECIFICATION.md) | Full protocol specification |
| [SECURITY.md](SECURITY.md) | Threat model and security architecture |
| [TRUST.md](TRUST.md) | Web of trust model |
| [DIRECTORY.md](DIRECTORY.md) | Directory protocol |

## Relationship to AMP

MESH is an **optional extension** to AMP. You can run AMP without MESH (local-only). You can run MESH only with AMP nodes (federation requires the base protocol).

```
┌─────────────────────────────────┐
│            MESH                 │  ← Federation layer (optional)
│  (discovery, sharing, trust)    │
├─────────────────────────────────┤
│            AMP                  │  ← Memory protocol (required)
│  (store, search, get, delete)   │
├─────────────────────────────────┤
│        Your Storage             │  ← Implementation detail
│  (SQLite, Postgres, etc.)       │
└─────────────────────────────────┘
```

## Status

🚧 **Draft specification** — Seeking feedback on security model before implementation.

We have production AMP implementations. MESH is the next layer.

## Contributing

Security review is especially welcome. See [CONTRIBUTING.md](CONTRIBUTING.md).

## License

MIT License

---

<p align="center">
  <i>Your knowledge, cryptographically yours.</i><br>
  <i>Share what you choose. Keep what you don't.</i>
</p>
