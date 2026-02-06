# MESH Directory Protocol

## Overview

Directories are **optional convenience services** that index public lessons for discovery. They are not authorities—trust is still node-to-node.

## Directory Principles

1. **Untrusted by design** — Directories can be malicious; cryptographic verification happens at the node level
2. **Federated** — Multiple directories exist, nodes query several
3. **Anyone can run one** — No permission needed
4. **Signed responses** — Directories sign their responses for accountability

## Directory API

### Register Node

```
POST /directory/v1/register
```

Request:
```json
{
  "identity": {
    "nodeId": "12D3KooW...",
    "publicKey": "base64...",
    "endpoints": { "mesh": "https://..." },
    "signature": "..."
  },
  "topics": ["rust", "agents", "devops"],
  "description": "Mike's code lessons"
}
```

Response:
```json
{
  "registered": true,
  "expiresAt": 1710000000000,
  "renewBefore": 1709900000000
}
```

Registrations expire. Nodes must re-register periodically (heartbeat).

### Search

```
POST /directory/v1/search
```

Request:
```json
{
  "query": "cargo build optimization",
  "filters": {
    "topics": ["rust"],
    "since": 1700000000000,
    "nodes": []
  },
  "limit": 20,
  "offset": 0
}
```

Response:
```json
{
  "results": [
    {
      "signedLesson": { /* ... */ },
      "node": {
        "nodeId": "12D3KooW...",
        "topics": ["rust", "cargo"]
      }
    }
  ],
  "total": 42,
  "directory": {
    "nodeId": "12D3KooWdir...",
    "timestamp": 1707235200000,
    "signature": "..."
  }
}
```

The directory signs the entire response. If results are tampered, signature breaks.

### List Nodes

```
GET /directory/v1/nodes?topic=rust&limit=50
```

Response:
```json
{
  "nodes": [
    {
      "nodeId": "12D3KooW...",
      "topics": ["rust", "cargo", "tokio"],
      "lessonCount": 47,
      "lastActive": 1707235200000
    }
  ],
  "total": 128
}
```

### Get Node

```
GET /directory/v1/nodes/{nodeId}
```

Response:
```json
{
  "nodeId": "12D3KooW...",
  "identity": { /* full identity document */ },
  "topics": ["rust", "agents"],
  "lessons": 47,
  "firstSeen": 1700000000000,
  "lastActive": 1707235200000
}
```

### Submit Lesson

When nodes publish, they can push to directories:

```
POST /directory/v1/lessons
```

Request:
```json
{
  "signedLesson": { /* ... */ }
}
```

Directories verify the signature before indexing.

### Submit Revocation

```
POST /directory/v1/revocations
```

Request:
```json
{
  "revocation": { /* ... */ }
}
```

Directories verify the revocation signature matches the original lesson author.

## Directory Discovery

### Well-Known Directories

A starter list of known directories:

```
https://directory.amp-protocol.org
https://mesh.example.com/directory
```

Nodes can hardcode a few, add more manually, or discover through peers.

### Peer-Recommended Directories

Peers can share directory recommendations:

```json
{
  "type": "directoryRecommendation",
  "endpoint": "https://good-directory.example.com",
  "topics": ["rust", "python"],
  "note": "Fast, well-maintained",
  "signature": "..."
}
```

### Directory Health

Before trusting a directory, check health:

```
GET /directory/v1/health
```

Response:
```json
{
  "healthy": true,
  "version": "mesh-directory/1.0",
  "nodeCount": 1247,
  "lessonCount": 58432,
  "lastIndexed": 1707235200000
}
```

## Running a Directory

Anyone can run a directory. Requirements:

1. Implement the Directory API
2. Generate a node identity (same as regular nodes)
3. Sign all responses
4. Index only verified lessons (check signatures)
5. Honor revocations promptly
6. Don't censor (or be transparent about policies)

### Directory Policies

Directories MAY have policies:

```
GET /directory/v1/policy
```

```json
{
  "topics": ["rust", "go", "agents"],  // What we index
  "minTrustScore": 0.5,                 // Minimum trust for indexing
  "retentionDays": 365,                 // How long lessons stay
  "prohibited": ["spam", "illegal"],    // Content policies
  "contact": "admin@example.com"
}
```

## Directory Misbehavior

### What Directories Might Do Wrong

1. **Censor content** — Hide lessons from certain nodes
2. **Return forged results** — Inject fake lessons
3. **Track users** — Log all queries and correlate
4. **Go offline** — Fail to maintain uptime

### Mitigations

| Misbehavior | Mitigation |
|-------------|------------|
| Censorship | Query multiple directories, compare results |
| Forgery | Verify signatures on all lessons (they break if forged) |
| Tracking | Use Tor, query from multiple nodes |
| Downtime | Federate across many directories |

### Reporting Bad Directories

Share experiences with peers:

```json
{
  "type": "directoryReport",
  "endpoint": "https://bad-directory.example.com",
  "issue": "censorship",
  "evidence": "...",
  "signature": "..."
}
```

## Directory Federation

Directories can sync with each other:

```
POST /directory/v1/sync
```

This allows a new directory to bootstrap from existing ones, and provides redundancy.

## Privacy Considerations

Directories see:
- What nodes exist
- What lessons are public
- What searches are performed (if not using Tor)

Directories do NOT see:
- Private lessons (never leave nodes)
- Who trusts whom (trust is node-to-node)
- Non-public node activity

---

*Directories are helpful servants, not trusted masters.*

---

## Who Runs Directories?

**You do, if you want one to exist.**

MESH is decentralized. There is no "MESH Inc" running infrastructure. No venture-funded startup maintaining servers. No cloud service to subscribe to.

### The Bootstrap Problem

A federated network needs someone to go first:

1. **Early adopters run the first directories** — Probably on a cheap VPS or spare server
2. **Spec includes a bootstrap list** — Known-good directories to start with
3. **Network grows, more directories appear** — Communities, companies, enthusiasts
4. **Federation prevents single points of failure** — No one directory is critical

### Why Would Anyone Run One?

| Operator | Motivation |
|----------|------------|
| **Protocol creators** | Bootstrap the network, prove it works |
| **Topic communities** | "The Rust MESH directory" as community resource |
| **Companies** | Value-add services (curation, analytics, premium features) |
| **Enthusiasts** | Same reason people run Tor relays, Mastodon instances, BitTorrent trackers |
| **Self-interest** | Your directory, your rules, your availability guarantee |

### What It Costs

Running a directory is cheap:

- **Compute**: Small VPS ($5-20/month) handles significant traffic
- **Storage**: Lessons are text, not video. Modest disk requirements.
- **Bandwidth**: Scales with popularity, but text is tiny
- **Maintenance**: Keep it updated, keep it online

A Raspberry Pi could run a small community directory.

### What If Nobody Runs One?

**MESH still works.** 

Directories are a convenience for discovery, not a requirement for operation. Nodes can:

- Connect directly to known peers
- Exchange peer lists with friends
- Operate in "friends only" mode indefinitely

A group of 10 friends could run MESH forever without any public directory. They'd just share node IDs out-of-band.

### The Bootstrap List

The spec will include a starter list of directories:

```
https://directory.amp-protocol.org    # Reference implementation
https://mesh.example.com/directory    # Community operated
```

**These don't exist yet.** They will when someone (probably us) runs them.

### Your Directory, Your Rules

If you don't like how existing directories operate:

1. Run your own
2. Set your own policies
3. Announce it to the network
4. Let nodes choose

That's the point of decentralization. No permission needed.

---

*Directories are community infrastructure. The community has to build them.*
