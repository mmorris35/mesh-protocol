# MESH Trust Model

## Overview

MESH uses a **Web of Trust** model. There is no central authority deciding who to trust. Instead, trust relationships are formed node-to-node and can be transitive.

## Trust Levels

| Level | Description | Search | Sync | Re-announce |
|-------|-------------|--------|------|-------------|
| `full` | Complete trust | ✓ | ✓ | ✓ |
| `limited` | Trust for discovery only | ✓ | ✗ | ✗ |
| `none` | No trust (blocked) | ✗ | ✗ | ✗ |

## Direct Trust

You explicitly add nodes you trust:

```bash
mesh trust add 12D3KooWxyz... --level full --note "Alice's node"
mesh trust add 12D3KooWabc... --level limited --note "Public directory"
```

## Transitive Trust

If you trust Alice (full), and Alice trusts Bob (full), you have a trust path to Bob.

```
You ──full──► Alice ──full──► Bob
         └──────────────────────┘
           transitive trust (depth 2)
```

### Trust Depth

Configure how many hops to traverse:

```bash
mesh config set trust.maxDepth 2  # Default
```

### Trust Score Decay

Trust score decays with distance:

| Hops | Score |
|------|-------|
| 0 (you) | 1.0 |
| 1 (direct) | 1.0 |
| 2 | 0.5 |
| 3 | 0.25 |
| 4+ | 0.125 (minimum) |

## Trust Operations

```bash
# Add trust
mesh trust add <nodeId> --level full|limited

# Remove trust
mesh trust remove <nodeId>

# List trusted nodes
mesh trust list

# Show trust path to a node
mesh trust path <nodeId>

# Export trust graph
mesh trust export > my-trust.json

# Import trust (additive)
mesh trust import peer-trust.json
```

## First-Contact Problem

How do you trust someone initially?

### Option 1: Out-of-Band Verification
Exchange node IDs through a trusted channel (in person, Signal, etc.)

### Option 2: Directory Introduction
Directories list nodes with topics. You verify through their public lessons.

### Option 3: Referral
A trusted peer recommends another node:
```json
{
  "type": "referral",
  "referee": "12D3KooWxyz...",
  "note": "Expert on Rust security",
  "signature": "..."
}
```

## Revoking Trust

When you revoke trust:

```bash
mesh trust remove <nodeId>
```

- Transitive paths through that node are invalidated
- Lessons from that node remain (already verified at receipt)
- Future lessons from that node are rejected

## Trust and Search

When searching, results are filtered and ranked by trust:

1. **Untrusted results discarded** — No trust path = not shown
2. **Trust score affects ranking** — Direct trust ranks higher
3. **Trust depth limits scope** — Depth=2 means 2 hops max

## Trust Persistence

Trust relationships are stored locally:

```
~/.mesh/trust.db
```

This file should be backed up. Losing it means rebuilding your trust network.

## Bootstrapping a New Node

1. Install MESH, generate identity
2. Connect to at least one known-good directory
3. Find nodes publishing in your interest areas
4. Verify a few lessons manually
5. Add trust for nodes that pass verification
6. Your trust network grows through transitivity

## Trust Display

```bash
$ mesh trust list
NODE ID                     LEVEL    HOPS  NOTE
12D3KooWabc...             full      1    Alice (coworker)
12D3KooWdef...             full      1    Mike's node
12D3KooWghi...             limited   1    amp-directory.org
12D3KooWjkl...             full      2    Bob (via Alice)
12D3KooWmno...             limited   2    Public Rust lessons (via directory)
```

## Anti-Patterns

### Trusting Everyone
Don't set `maxDepth: 10` and trust every directory. You'll be flooded with noise.

### Trusting No One
A node with no trust relationships can't participate in federation. You'll only have local memories.

### Trusting Directories Fully
Directories are conveniences, not authorities. Use `limited` trust.

---

*Trust is earned, verified, and can be revoked. That's the model.*
