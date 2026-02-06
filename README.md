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

---

## 🌡️ Killer Use Case: Sovereign IoT Sensor Network

MESH + AMP turn dumb sensors into a **private, federated data network**.

```
┌──────────────────────────────────────────────────────────────┐
│                      Your Home                               │
│                                                              │
│   🌡️ Garage        💧 Basement       ⚡ Breaker Panel        │
│   Temp Sensor      Humidity          Power Meter             │
│   (amp-storage)    (amp-storage)     (amp-storage)           │
│   ESP32 · $5       Pi Zero · $10     ESP32 · $5              │
│        │                │                  │                 │
│        └────────────────┼──────────────────┘                 │
│                         │                                    │
│                  ┌──────▼──────┐                            │
│                  │    MESH     │                            │
│                  │  discovers  │                            │
│                  │  all nodes  │                            │
│                  └──────┬──────┘                            │
│                         │                                    │
│    ┌────────────────────┼────────────────────┐              │
│    │                    │                    │              │
│    ▼                    ▼                    ▼              │
│  📱 Phone          🏠 Home Assistant    🤖 AI Agent         │
│  "garage temp?"    automation rules    "basement humid?"    │
└──────────────────────────────────────────────────────────────┘
```

**No cloud. No subscription. No "service discontinued."**

```bash
# From anywhere on your network
mesh search "temperature last 24h"
mesh search "humidity > 60%" --sources basement
mesh search "power anomaly"
```

Each sensor:
- Stores readings locally with TTL (old data auto-expires)
- Serves data on demand
- Cryptographically signed (tamper-evident)
- **Encrypted with keys you control**
- Runs on $5-10 hardware

**And here's the key insight: sensors don't need to be on your local network.**

Because data is encrypted, sensors can live **anywhere on the public internet**:

```
🏔️ Remote Cabin     🏠 Rental Property    🚗 Vehicle        🌍 Anywhere
   (satellite)         (4G LTE)            (cellular)         (internet)
        │                   │                   │                  │
        └───────────────────┴───────────────────┴──────────────────┘
                                    │
                             PUBLIC INTERNET
                           (all data encrypted)
                                    │
                             ┌──────▼──────┐
                             │    MESH     │
                             └──────┬──────┘
                                    │
                         Only YOUR keys decrypt
                                    │
                    ┌───────────────┼───────────────┐
                    ▼               ▼               ▼
               📱 Phone      💻 Laptop       🤖 AI Agent
               anywhere       anywhere        anywhere
```

**Access control via cryptography:**
- **Share access?** Issue a key to family, tenants, contractors
- **Revoke access?** Delete their key. Instant. Cryptographic. Done.
- **Sensor compromised?** They get encrypted blobs. Useless without keys.

**This is zero-trust IoT.** Your sensors, anywhere in the world. Your data, encrypted in transit and at rest. Your keys, your rules.

### 📻 Off-Grid: LoRa / Meshtastic Integration

No internet at all? MESH works with [Meshtastic](https://meshtastic.org/) LoRa radio networks.

```
┌────────────────────────────────────────────────────────────┐
│              WILDERNESS / DISASTER ZONE                    │
│              (zero infrastructure)                         │
│                                                            │
│  🌡️ Weather      🦌 Trail Cam      🌊 Flood Gauge          │
│  (solar+LoRa)    (solar+LoRa)      (solar+LoRa)            │
│       │               │                 │                  │
│       └───── LoRa Mesh (10+ km range) ─────┘               │
│                       │                                    │
│          tiny packets over radio mesh                      │
│             "I exist, I have data"                         │
│                       │                                    │
│                ┌──────▼──────┐                            │
│                │   Gateway   │  ← LoRa + satellite/cell   │
│                │   Node      │     when available          │
│                └──────┬──────┘                            │
│                       │                                    │
│          Full MESH sync when connectivity exists           │
│                       │                                    │
│                ┌──────▼──────┐                            │
│                │    MESH     │                            │
│                │   Network   │                            │
│                └─────────────┘                            │
└────────────────────────────────────────────────────────────┘
```

**Two-layer architecture:**
1. **LoRa layer:** Immediate, low-bandwidth announcements (sensor readings, alerts)
2. **MESH layer:** Full records sync when gateway reaches internet

**Why this matters:**
- Agricultural sensors across 1000 acres with no WiFi
- Backcountry weather stations miles from roads
- Disaster recovery when cell towers are down
- Wildlife/environmental monitoring in wilderness
- Post-infrastructure resilience

**$35 Meshtastic node + solar = years of autonomous operation.**

All data still encrypted. Keys still yours. Just works without the internet.

**Post-infrastructure IoT.** No cell towers. No WiFi. No cloud. Just radios, sun, and cryptography.

**This is the IoT we were promised.**

---

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
| [DEPLOYMENT.md](DEPLOYMENT.md) | **Zero-infrastructure deployment guide** |

---

## 🏠 Run From Home (Zero Infrastructure)

**You don't need a VPS. You don't need a public IP. You don't need to open ports.**

```bash
# Start your node
./amp-mini --port 8765

# Expose via Cloudflare Tunnel (free, outbound-only)
cloudflared tunnel --url http://localhost:8765
# → https://random-words.trycloudflare.com

# You're on MESH. From home. Behind NAT.
```

Everyone—nodes, relays, directories—runs from home using outbound tunnels. Cloudflare's free tier handles global routing. See [DEPLOYMENT.md](DEPLOYMENT.md) for details.

**Cost:** ~$15/year (electricity + optional domain). **Privacy:** Your hardware, your home, your rules.

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
