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
