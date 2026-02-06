# MESH Protocol Specification

> **Memory Exchange & Sharing Hub** — v0.1 Draft

## Abstract

MESH defines a secure, decentralized protocol for federating AMP (Agent Memory Protocol) nodes. It enables selective sharing of knowledge across a network of trusted peers while maintaining strong cryptographic guarantees for authenticity, integrity, and revocability.

---

## 1. Node Identity

### 1.1 Keypair Generation

Every MESH node MUST have a long-term identity keypair:

```
Algorithm: Ed25519
Key size: 256 bits (32 bytes)
```

Generation (pseudocode):
```python
private_key = random_bytes(32)
public_key = ed25519_derive_public(private_key)
node_id = base58_encode(public_key)
fingerprint = sha256(public_key).hex()[:16]
```

### 1.2 Node Identity Document

Nodes publish an identity document at a well-known endpoint:

```
GET /.well-known/mesh/identity
```

Response:
```json
{
  "version": "mesh/1.0",
  "nodeId": "12D3KooW...",
  "publicKey": "base64_encoded_ed25519_public_key",
  "endpoints": {
    "mesh": "https://amp.example.com/mesh",
    "amp": "https://amp.example.com/amp"
  },
  "capabilities": ["search", "sync", "directory"],
  "signature": "base64_encoded_self_signature"
}
```

The identity document MUST be self-signed to prove key possession.

---

## 2. Signed Lessons

### 2.1 Signature Format

All shared lessons MUST be signed:

```typescript
interface SignedLesson {
  // The AMP lesson content
  lesson: {
    id: string;
    type: string;
    title?: string;
    content: string;
    tags?: string[];
    severity?: string;
    created_at: number;
    // ... other AMP fields
  };
  
  // Publication metadata
  publication: {
    visibility: "unlisted" | "public";
    publishedAt: number;
    topics?: string[];      // Optional categorization
  };
  
  // Cryptographic signature
  signature: {
    algorithm: "ed25519";
    nodeId: string;         // Publisher's node ID
    publicKey: string;      // Base64 Ed25519 public key
    timestamp: number;      // Signing timestamp (Unix ms)
    sig: string;            // Base64 signature
  };
}
```

### 2.2 Signing Process

1. Construct the signable payload:
```typescript
const payload = {
  lesson: lesson,
  publication: publication,
  timestamp: Date.now()
};
const canonical = canonicalize(payload); // RFC 8785 JSON Canonicalization
```

2. Sign with Ed25519:
```typescript
const signature = ed25519_sign(private_key, canonical);
```

3. Attach signature to produce SignedLesson.

### 2.3 Verification Process

1. Extract `lesson`, `publication`, and `signature.timestamp`
2. Reconstruct canonical payload
3. Verify signature against payload using `signature.publicKey`
4. Verify `signature.nodeId` matches `base58(signature.publicKey)`
5. Verify timestamp is within acceptable window (e.g., ±1 hour of current time for new, any past time for sync)
6. Optionally verify trust path to signer

---

## 3. Visibility & Publication

### 3.1 Visibility Levels

| Visibility | Description |
|------------|-------------|
| `private` | Never leaves the node. Default for all AMP records. |
| `unlisted` | Shared if requested by ID, not indexed or broadcast. |
| `public` | Indexed by directories, included in federated search. |

### 3.2 Publication Flow

```
amp store lesson (private)
         │
         ▼
mesh publish <id> --visibility public
         │
         ├─► Sign lesson with node key
         │
         ├─► Announce to connected peers
         │
         └─► Submit to known directories
```

### 3.3 Announcement Message

When a lesson is published, nodes announce to peers:

```typescript
interface PublicationAnnouncement {
  type: "publication";
  signedLesson: SignedLesson;
}
```

Peers receiving an announcement:
1. Verify signature
2. Check trust path to publisher
3. If trusted: store locally, optionally re-announce to their peers
4. If untrusted: ignore

---

## 4. Revocation

### 4.1 Revocation Record

```typescript
interface Revocation {
  type: "revocation";
  lessonId: string;
  nodeId: string;           // Must match original publisher
  revokedAt: number;
  reason?: "outdated" | "incorrect" | "private" | "other";
  signature: {
    algorithm: "ed25519";
    publicKey: string;
    sig: string;
  };
}
```

### 4.2 Revocation Rules

1. ONLY the original publisher can revoke (signature must verify with same key)
2. Revocation MUST be propagated like publications
3. Nodes SHOULD delete revoked lessons from local cache
4. Directories MUST remove revoked lessons from index
5. Revocation records SHOULD be retained to reject re-announcements

### 4.3 Revocation Propagation

```
mesh revoke <lesson_id>
         │
         ├─► Sign revocation with node key
         │
         ├─► Announce to connected peers
         │
         └─► Submit to known directories
```

---

## 5. Federated Search

### 5.1 Search Request

```typescript
interface SearchRequest {
  type: "search";
  query: string;
  filters?: {
    types?: string[];       // AMP record types
    tags?: string[];
    topics?: string[];
    since?: number;         // Unix ms
    nodes?: string[];       // Limit to specific nodes
  };
  limit?: number;           // Default: 20, max: 100
  hops?: number;            // How far to propagate (default: 2)
  requestId: string;        // For deduplication
  origin: string;           // Originating node ID
  ttl: number;              // Remaining hops
}
```

### 5.2 Search Flow

```
You ──search──► Peer A ──search──► Peer A's peers
                  │                      │
                  ▼                      ▼
              local search          local search
                  │                      │
                  ▼                      ▼
              results ◄─────────── results
                  │
                  ▼
         aggregate & return
```

### 5.3 Search Response

```typescript
interface SearchResponse {
  type: "searchResponse";
  requestId: string;
  results: Array<{
    signedLesson: SignedLesson;
    score: number;          // Relevance score (0-1)
    trustScore: number;     // Based on trust path
    via?: string;           // Node that provided this result
  }>;
  truncated: boolean;       // More results available
}
```

### 5.4 Result Ranking

Final score = `relevance * trustScore * freshness`

Where:
- `relevance`: Semantic similarity from AMP search
- `trustScore`: 1.0 for direct trust, 0.5 per hop
- `freshness`: Decay function based on age (optional)

---

## 6. Peer Discovery & Directory

### 6.1 Peer Connection

Nodes maintain a set of connected peers:

```typescript
interface PeerConnection {
  nodeId: string;
  endpoint: string;
  trustLevel: "full" | "limited" | "none";
  lastSeen: number;
  connectedSince: number;
}
```

### 6.2 Directory Protocol

Directories index public lessons for discovery.

**Registration:**
```
POST /directory/register
{
  "identity": { /* Node identity document */ },
  "topics": ["rust", "devops", "agents"]
}
```

**Search:**
```
POST /directory/search
{
  "query": "cargo parallel builds",
  "topics": ["rust"],
  "limit": 20
}
```

**Response:**
```json
{
  "results": [ /* SignedLesson[] */ ],
  "sources": ["nodeA", "nodeB"],
  "signature": { /* Directory signature */ }
}
```

### 6.3 Directory Federation

Nodes SHOULD connect to multiple directories. Directories MAY sync with each other.

No single directory is authoritative. Results from multiple directories are merged client-side.

---

## 7. Wire Protocol

### 7.1 Transport

MESH uses HTTP/2 or HTTP/3 over TLS 1.3.

Base endpoint: `https://{host}/mesh/v1/`

### 7.2 Endpoints

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/identity` | GET | Node identity document |
| `/announce` | POST | Receive publication/revocation |
| `/search` | POST | Federated search |
| `/sync` | POST | Bulk sync (optional) |
| `/peers` | GET | List connected peers (optional) |

### 7.3 Message Envelope

All messages wrapped in envelope:

```typescript
interface MeshEnvelope {
  version: "mesh/1.0";
  type: string;
  timestamp: number;
  sender: string;           // Node ID
  payload: any;
  signature?: string;       // Optional, required for mutations
}
```

---

## 8. Sync Protocol (Optional)

For nodes that want full or filtered replication:

### 8.1 Sync Request

```typescript
interface SyncRequest {
  type: "syncRequest";
  since?: number;           // Lessons published after this time
  topics?: string[];        // Filter by topics
  checkpoint?: string;      // Resume token
}
```

### 8.2 Sync Response

```typescript
interface SyncResponse {
  type: "syncResponse";
  lessons: SignedLesson[];
  revocations: Revocation[];
  checkpoint: string;       // For resumption
  hasMore: boolean;
}
```

---

## 9. Error Handling

### 9.1 Error Response

```typescript
interface MeshError {
  error: true;
  code: string;
  message: string;
  details?: any;
}
```

### 9.2 Error Codes

| Code | Meaning |
|------|---------|
| `INVALID_SIGNATURE` | Signature verification failed |
| `UNTRUSTED_NODE` | No trust path to sender |
| `UNKNOWN_LESSON` | Requested lesson not found |
| `ALREADY_REVOKED` | Lesson was already revoked |
| `RATE_LIMITED` | Too many requests |
| `INVALID_REQUEST` | Malformed request |

---

## 10. Conformance

### 10.1 MUST Requirements

A conformant MESH implementation MUST:

1. Generate and protect Ed25519 identity keypair
2. Sign all outgoing publications and revocations
3. Verify signatures on all incoming lessons
4. Honor visibility levels
5. Process revocations from original publishers
6. Use TLS 1.3+ for all connections
7. Implement `/identity` and `/announce` endpoints

### 10.2 SHOULD Requirements

A conformant MESH implementation SHOULD:

1. Support federated search
2. Connect to multiple directories
3. Implement trust graph with configurable depth
4. Support sync protocol
5. Rate limit incoming requests
6. Log security-relevant events

---

## Appendix A: Canonical JSON

MESH uses RFC 8785 (JSON Canonicalization Scheme) for deterministic serialization before signing.

Key rules:
- Object keys sorted lexicographically
- No whitespace
- Numbers in shortest form
- Strings escaped minimally

---

## Appendix B: Example Messages

### Publish Announcement

```json
{
  "version": "mesh/1.0",
  "type": "publication",
  "timestamp": 1707235200000,
  "sender": "12D3KooWabc...",
  "payload": {
    "signedLesson": {
      "lesson": {
        "id": "lesson_xyz",
        "type": "lesson",
        "title": "Cargo parallel builds crash servers",
        "content": "Never run cargo commands in parallel...",
        "tags": ["rust", "cargo", "build"]
      },
      "publication": {
        "visibility": "public",
        "publishedAt": 1707235200000,
        "topics": ["rust", "devops"]
      },
      "signature": {
        "algorithm": "ed25519",
        "nodeId": "12D3KooWabc...",
        "publicKey": "MCowBQYDK2VwAyEA...",
        "timestamp": 1707235200000,
        "sig": "base64..."
      }
    }
  }
}
```

---

*Specification draft. Subject to change based on security review.*
