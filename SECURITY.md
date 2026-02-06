# MESH Security Model

> **Secure as fuck** — The guiding principle

## Threat Model

### Adversaries

| Adversary | Capabilities | Goals |
|-----------|--------------|-------|
| **Passive Network Observer** | Can see traffic metadata | Learn who shares what with whom |
| **Active Network Attacker** | Can intercept/modify traffic | Inject false lessons, impersonate nodes |
| **Malicious Node** | Controls one or more MESH nodes | Poison knowledge base, spam, harvest data |
| **Compromised Directory** | Controls a directory server | Censor nodes, manipulate discovery |
| **Rogue Insider** | Has legitimate access to a node | Leak private lessons, forge attributions |

### Assets to Protect

1. **Private Lessons** — Must never leak without explicit publish action
2. **Node Identity** — Must not be impersonatable
3. **Lesson Integrity** — Content must not be tamperable
4. **Attribution** — Author must be verifiable
5. **Revocation** — Unpublished content must become unavailable
6. **Query Privacy** — What you search for reveals interests
7. **Social Graph** — Who trusts whom is sensitive

---

## Cryptographic Primitives

### Identity

Each MESH node has a **long-term identity keypair**:

```
Algorithm: Ed25519
Private key: 32 bytes, never leaves the node
Public key: 32 bytes, serves as node identifier
Fingerprint: SHA-256(public_key), displayed as hex
```

**Node ID** = Base58 encoding of public key (like IPFS peer IDs)

Example: `12D3KooWN7hT5qJK4vZ9kE3xP2yR8mF6wL1cD4nB2aS5gH7jU9iY`

### Lesson Signing

Every shared lesson includes a cryptographic signature:

```typescript
interface SignedLesson {
  // Original AMP lesson
  lesson: Lesson;
  
  // Signature metadata
  signature: {
    algorithm: "ed25519";
    publicKey: string;      // Author's public key
    timestamp: number;      // Unix ms when signed
    signature: string;      // Base64 Ed25519 signature
  };
  
  // What was signed
  signedData: string;       // Canonical JSON of lesson + timestamp
}
```

**Signing process:**
1. Canonicalize lesson JSON (sorted keys, no whitespace)
2. Append timestamp
3. Sign with Ed25519 private key
4. Attach signature to lesson

**Verification process:**
1. Extract signedData and signature
2. Look up public key (from signature or directory)
3. Verify Ed25519 signature
4. Check timestamp is reasonable (not future, not ancient)
5. Verify public key matches claimed author

### Transport Security

All MESH communication uses **TLS 1.3** minimum.

```
Required cipher suites:
- TLS_AES_256_GCM_SHA384
- TLS_CHACHA20_POLY1305_SHA256

Certificate validation:
- Standard PKI for initial connection
- Node identity verified via signed challenge after TLS established
```

### Optional: End-to-End Encryption

For sensitive sync between trusted nodes (not public sharing):

```
Algorithm: X25519 key exchange + XChaCha20-Poly1305
Forward secrecy: New ephemeral keypair per session
```

---

## Visibility Levels

| Level | Who can see | Indexed by directories | Appears in federated search |
|-------|-------------|------------------------|----------------------------|
| `private` | Only you | No | No |
| `unlisted` | Anyone with direct link | No | No |
| `public` | Everyone | Yes | Yes |

**Transitions:**
- `private` → `public`: Publish action (signs and announces)
- `public` → `private`: Revoke action (signs and propagates revocation)
- `unlisted`: Middle ground for sharing via direct link without broadcast

---

## Revocation

When a lesson is unpublished:

```typescript
interface RevocationRecord {
  lessonId: string;
  revokedAt: number;        // Unix ms
  reason?: string;          // Optional: "outdated", "incorrect", "private"
  signature: {
    algorithm: "ed25519";
    publicKey: string;      // Must match original author
    signature: string;
  };
}
```

**Revocation propagation:**
1. Author signs revocation record
2. Sends to all known peers
3. Peers verify signature matches original lesson author
4. Peers delete cached copies, propagate revocation
5. Directories remove from index

**Revocation is best-effort.** Once something is public, copies may exist. Revocation signals intent and compliant nodes will honor it.

---

## Trust Model

### Web of Trust

Nodes don't trust a central authority. Instead:

```
Direct trust:     You explicitly mark a node as trusted
Transitive trust: If you trust A, and A trusts B, you have a path to B
Trust depth:      How many hops you'll traverse (default: 2)
Trust score:      Decays with distance: 1.0 → 0.5 → 0.25
```

### Trust Operations

```bash
mesh trust add nodeB --level full        # Direct trust
mesh trust add nodeC --level limited     # Trust for search, not sync
mesh trust remove nodeD                  # Revoke trust
mesh trust list                          # Show trust graph
```

### Trust Verification

Before accepting a lesson from the network:

1. **Signature valid?** — Ed25519 verification passes
2. **Author trusted?** — Direct or transitive trust path exists
3. **Not revoked?** — No valid revocation record seen
4. **Timestamp sane?** — Within acceptable window

Lessons failing verification are **rejected**, not stored.

---

## Directory Security

Directories are **untrusted conveniences**, not authorities.

### What Directories Can Do
- Index public lessons
- Return search results
- Provide node discovery

### What Directories Cannot Do
- Forge lessons (no private keys)
- Modify content (signatures break)
- Hide revocations (nodes gossip directly too)
- Compel trust (trust is node-to-node)

### Directory Federation

Multiple directories exist. Nodes can:
- Query multiple directories
- Run their own directory
- Operate without any directory (peer-to-peer only)

```bash
mesh directory add https://dir.example.com
mesh directory add https://amp-directory.org
mesh directory list
mesh directory remove https://evil.example.com
```

### Directory Accountability

Directories sign their responses. Misbehavior is provable:

```typescript
interface DirectoryResponse {
  results: SignedLesson[];
  directory: {
    nodeId: string;
    timestamp: number;
    signature: string;    // Directory signs the response
  };
}
```

If a directory returns forged results, the forgery is cryptographically evident.

---

## Query Privacy

### The Problem
Your searches reveal your interests. "How to fix vulnerability X" tells observers you might have vulnerability X.

### Mitigations

**Level 1: TLS**
- Passive observers see you talking to nodes, not what you're asking
- Baseline, always on

**Level 2: Query Fanout**
- Query multiple nodes with different query fragments
- Combine results locally
- Observers see multiple partial queries

**Level 3: Private Information Retrieval (Future)**
- Cryptographic PIR allows queries without revealing the query
- Computationally expensive, optional for sensitive searches

### Metadata Minimization

MESH nodes should:
- Not log queries by default
- Not require authentication for public searches
- Support Tor/I2P for network-level anonymity

---

## Attack Scenarios & Mitigations

### 1. Impersonation Attack
**Attack:** Mallory claims to be Alice's node.
**Mitigation:** All messages signed with Ed25519. Mallory can't forge Alice's signature.

### 2. Replay Attack
**Attack:** Mallory captures a valid signed lesson, replays it later.
**Mitigation:** Timestamps in signatures. Nodes reject stale timestamps.

### 3. Sybil Attack
**Attack:** Mallory creates 1000 fake nodes to dominate the network.
**Mitigation:** Web of trust. Fake nodes have no trust paths to real users.

### 4. Poisoning Attack
**Attack:** Mallory publishes plausible but wrong lessons.
**Mitigation:** Attribution is permanent. Reputation tracks accuracy. Block bad actors.

### 5. Eclipse Attack
**Attack:** Mallory isolates a node from the real network.
**Mitigation:** Multiple directory sources. Direct peer connections. Out-of-band verification.

### 6. Censorship Attack
**Attack:** Compromised directory hides certain lessons.
**Mitigation:** Multiple directories. Direct peer gossip. Anyone can run a directory.

### 7. Harvest Now, Decrypt Later
**Attack:** Record encrypted traffic, break crypto in future.
**Mitigation:** Forward secrecy (ephemeral keys). Lessons are signed, not encrypted—public content isn't secret.

---

## Implementation Checklist

For a MESH implementation to be considered secure:

- [ ] Ed25519 for all signatures
- [ ] TLS 1.3+ for all connections
- [ ] Private keys never logged, never transmitted
- [ ] Signature verification before storing any remote lesson
- [ ] Trust graph enforced on all queries
- [ ] Revocation records honored within reasonable time
- [ ] No query logging by default
- [ ] Timestamp validation on all signed content
- [ ] Multiple directory support
- [ ] Peer-to-peer fallback if directories unavailable

---

## Security Reporting

Found a vulnerability? Please report responsibly:

1. **Do not** open a public GitHub issue
2. Email: security@[TBD]
3. Include: Description, reproduction steps, potential impact
4. We will respond within 48 hours
5. Coordinated disclosure after fix is available

---

*Security is not a feature. It's the foundation.*

---

## Cryptographic Revocation (Hard Delete)

Standard revocation is best-effort: compliant nodes honor it, but copies may persist. **Cryptographic revocation** makes content mathematically inaccessible.

### Principle

> The content never leaves your control. Only encrypted forms are shared. You hold the key. Revoke the key, and all copies become permanent noise.

### Architecture

```
Publishing:
  Lesson ─► AES-256-GCM encrypt ─► Encrypted blob (shared)
                    │
                    └─► Symmetric key (kept on your node)

Reading:
  Encrypted blob + Key fetch from author ─► Decrypted lesson

Revoking:
  Delete key ─► All encrypted blobs become unreadable
```

### Encrypted Lesson Format

```typescript
interface EncryptedLesson {
  // Encryption metadata
  encryption: {
    algorithm: "aes-256-gcm";
    keyId: string;              // Unique key identifier
    keyEndpoint: string;        // URL to fetch key
    nonce: string;              // Base64, 12 bytes for GCM
    tag: string;                // Base64, GCM auth tag
  };
  
  // The encrypted content
  ciphertext: string;           // Base64 encrypted lesson JSON
  
  // Signature covers the encrypted form
  signature: {
    algorithm: "ed25519";
    nodeId: string;
    publicKey: string;
    timestamp: number;
    sig: string;
  };
}
```

### Key Management

Keys are served by the author's node:

```
GET /mesh/v1/keys/{keyId}
```

Response (if authorized and not revoked):
```json
{
  "keyId": "key_abc123",
  "key": "base64_encoded_aes_key",
  "expiresAt": 1710000000000
}
```

Response (if revoked):
```json
{
  "error": "KEY_REVOKED",
  "revokedAt": 1707235200000
}
```

### Authorization for Key Access

Keys can be restricted:

```typescript
interface KeyPolicy {
  keyId: string;
  access: "public" | "trusted" | "explicit";
  allowedNodes?: string[];      // For explicit access
  minTrustScore?: number;       // For trusted access
}
```

- `public`: Any node can fetch the key
- `trusted`: Only nodes in your trust graph
- `explicit`: Only specifically listed nodes

### Revocation Process

```bash
mesh revoke lesson_abc123 --hard
```

This:
1. Deletes the key from local storage
2. Marks the keyId as revoked (returns KEY_REVOKED forever)
3. Propagates standard revocation record to network
4. Content is now cryptographically dead

### Key Escrow (Optional)

For availability when your node is offline, keys can be escrowed:

```typescript
interface KeyEscrow {
  keyId: string;
  escrowNodes: string[];        // Trusted nodes holding backup
  threshold: number;            // k-of-n required to reconstruct
  encryptedShares: string[];    // Shamir secret sharing
}
```

Revocation must notify escrow nodes to delete their shares.

### Caching Considerations

Nodes MAY cache decrypted content in memory for performance. To limit exposure:

1. **TTL on decrypted cache**: Re-fetch key periodically
2. **Revocation push**: Author can push revocation to known readers
3. **Key rotation**: Periodically rotate keys for long-lived content

### Guarantees

| Scenario | Outcome |
|----------|---------|
| Compliant node, revoked | Deletes cached content, key fetch fails |
| Non-compliant node, revoked | Has encrypted blob, key fetch fails, content unreadable |
| Offline/archived copy | Encrypted blob is permanent noise |
| Backup restored | Key still revoked, content still dead |

### Limitations

- **RAM snapshots**: If content was decrypted in RAM and node was compromised, that instance is exposed
- **Screenshots/copies**: If a human copied the text manually, crypto can't help
- **Key availability**: Your node must be reachable for others to read (mitigate with escrow)

### Summary

Cryptographic revocation provides **mathematical certainty** that revoked content is inaccessible. It doesn't require trust in other nodes' compliance—the laws of cryptography enforce it.

```
You don't ask "please delete this."
You delete the key.
The content dies everywhere, forever.
```

---

## Honest Limitations

### What Cryptography Cannot Do

**Cryptographic revocation guarantees that *unaccessed* copies become unreadable. It cannot retroactively un-read something a human or node already decrypted.**

No technology can solve this. If someone decrypted your lesson while they had key access and saved the plaintext, they have it forever. This is physics, not a bug.

```
The Analog Hole:
┌─────────────────────────────────────────────┐
│  Encrypted lesson ──► Decrypted in RAM      │
│                            │                │
│                            ▼                │
│                    ┌───────────────┐        │
│                    │ Copy/paste    │        │
│                    │ Screenshot    │        │
│                    │ Save to file  │        │
│                    │ Write it down │        │
│                    └───────────────┘        │
│                            │                │
│                            ▼                │
│                     Plaintext copy          │
│                  (crypto can't help)        │
└─────────────────────────────────────────────┘
```

### What We CAN Guarantee

| Scenario | Protection |
|----------|------------|
| Content never accessed before revoke | ✅ Permanently unreadable |
| Backups, logs, archives (encrypted) | ✅ Permanently unreadable |
| Compliant nodes that cached | ✅ Will delete, can't re-fetch key |
| Node that decrypted but didn't save plaintext | ✅ Can't re-decrypt |
| Node that saved decrypted plaintext | ❌ They have it |
| Human who copy/pasted | ❌ They have it |

### Mitigation Strategies

Since we can't prevent determined bad actors, we minimize exposure:

1. **Trust graph limits access**
   - Only trusted nodes can fetch keys
   - Fewer decryptions = fewer potential leaks

2. **Audit trail**
   - Log who fetched which keys
   - You know who *could* have the plaintext

3. **Reputation system**
   - Leak content? Get blacklisted network-wide
   - Social/economic consequences

4. **Time-limited access**
   - Keys can have short TTLs
   - Reduces window of exposure

5. **Legal/social layer**
   - Terms of service
   - Community norms
   - "Don't be a jerk" still matters

### The Right Mental Model

Think of cryptographic revocation like a **self-destructing message**:

- It works perfectly against passive threats (backups, forensics, archives)
- It works well against lazy threats (compliant nodes, casual users)
- It raises the bar significantly against active threats
- It cannot defeat a determined adversary with a camera

**This is still valuable.** Most data leaks aren't from determined adversaries—they're from forgotten backups, old logs, and systems that never got the memo to delete. Crypto handles all of those perfectly.

### Summary

> We are honest about our limits because security through obscurity is no security at all.
>
> Cryptographic revocation is powerful. It is not magic.
>
> Use it to protect against the 99% of threats it defeats, not the 1% it can't.
