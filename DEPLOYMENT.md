# MESH Deployment Guide

## The Zero Infrastructure Model

**You don't need a VPS. You don't need a public IP. You don't need to open ports.**

Every participant in MESH—nodes, relays, directories—can run from home behind NAT using outbound tunnels.

```
┌─────────────────────────────────────────────────────────────────┐
│                    THE MESH NETWORK                             │
│                                                                 │
│  Your Home         Their Home         Another Home              │
│  (NAT)             (NAT)              (NAT)                     │
│                                                                 │
│  ┌────────┐        ┌────────┐         ┌────────┐               │
│  │ Nellie │        │ Relay  │         │Directory│               │
│  └───┬────┘        └───┬────┘         └───┬─────┘               │
│      │ outbound        │ outbound         │ outbound            │
│      ▼                 ▼                  ▼                     │
│  ┌────────┐        ┌────────┐         ┌────────┐               │
│  │Cloudflare│       │Cloudflare│        │Cloudflare│              │
│  │ Tunnel │        │ Tunnel │         │ Tunnel │               │
│  └───┬────┘        └───┬────┘         └───┬────┘               │
│      │                 │                  │                     │
│      └─────────────────┼──────────────────┘                     │
│                        ▼                                        │
│              ┌──────────────────┐                               │
│              │  Cloudflare Edge │  (free, global)               │
│              │  routes traffic  │                               │
│              └────────┬─────────┘                               │
│                       │                                         │
│                       ▼                                         │
│              ┌──────────────────┐                               │
│              │    The Internet  │                               │
│              │   (anyone can    │                               │
│              │    reach MESH)   │                               │
│              └──────────────────┘                               │
└─────────────────────────────────────────────────────────────────┘
```

## Why This Works

| Traditional Self-Hosting | MESH Model |
|-------------------------|------------|
| Rent a VPS ($5-20/mo) | Run at home (free) |
| Configure firewall/ports | No ports to open |
| Get a static IP or DDNS | Cloudflare handles routing |
| Worry about DDoS | Cloudflare absorbs it |
| Pay for bandwidth | Cloudflare caches |

**All connections are outbound.** Your firewall doesn't need to allow anything in.

## Recommended Stack

### For Node Operators (Your Nellie)

```bash
# 1. Install cloudflared
brew install cloudflare/cloudflare/cloudflared  # macOS
# or: apt install cloudflared                    # Debian/Ubuntu

# 2. Login to Cloudflare (one-time)
cloudflared tunnel login

# 3. Create a tunnel
cloudflared tunnel create nellie

# 4. Configure DNS (your domain)
cloudflared tunnel route dns nellie nellie.yourdomain.com

# 5. Run Nellie with tunnel
cloudflared tunnel run nellie --url http://localhost:8765
```

Your node is now reachable at `https://nellie.yourdomain.com` from anywhere.

**No domain?** Use Cloudflare's free `trycloudflare.com`:
```bash
cloudflared tunnel --url http://localhost:8765
# Outputs: https://random-words.trycloudflare.com
```

### For Relay Operators

Same process. A relay is just a MESH node that forwards traffic for others.

```bash
cloudflared tunnel create mesh-relay
cloudflared tunnel route dns mesh-relay relay.yourdomain.com
cloudflared tunnel run mesh-relay --url http://localhost:8766
```

Register your relay in the MESH directory. NAT'd nodes can now use it.

### For Directory Operators

Same process. Directories are just nodes that index public lessons.

```bash
cloudflared tunnel create mesh-directory
cloudflared tunnel route dns mesh-directory directory.yourdomain.com
cloudflared tunnel run mesh-directory --url http://localhost:8767
```

## Node Registration

When a NAT'd node comes online, it registers its reachability:

```json
{
  "nodeId": "12D3KooWabc...",
  "reachability": {
    "tunnel": "https://nellie.yourdomain.com",
    "via": null
  },
  "signature": "..."
}
```

If using a relay instead of direct tunnel:

```json
{
  "nodeId": "12D3KooWabc...",
  "reachability": {
    "tunnel": null,
    "via": "https://relay.mesh.community"
  },
  "signature": "..."
}
```

## Traffic Flow

### Direct (Node has Cloudflare Tunnel)

```
Requester
    │
    ▼
Cloudflare Edge
    │
    ▼ (tunnel)
Your Nellie
```

**Latency:** ~10-50ms added by tunnel

### Via Relay (Node behind NAT without tunnel)

```
Requester
    │
    ▼
Cloudflare Edge
    │
    ▼ (tunnel)
Relay Node
    │
    ▼ (persistent connection)
Your Nellie
```

**Latency:** ~20-100ms added

### End-to-End Encryption

**Cloudflare and relays cannot read your data.**

```
┌─────────────────────────────────────────────────────┐
│                                                     │
│  Requester ◄───── E2E Encrypted (keys) ─────► Node │
│                                                     │
│      │                                        │     │
│      └──► Cloudflare ──► Relay ──► Cloudflare ─┘    │
│           (sees encrypted blobs only)               │
│                                                     │
└─────────────────────────────────────────────────────┘
```

Tunnels and relays are **transport**, not **trust**. They move bytes they can't decrypt.

## Cost Analysis

| Component | Cost |
|-----------|------|
| Nellie node (Pi 4) | $55 one-time |
| Cloudflare Tunnel | Free |
| Domain (optional) | $10/year |
| Electricity | ~$5/year |
| **Total Year 1** | **~$70** |
| **Total Year 2+** | **~$15/year** |

Compare to cloud hosting: $60-240/year for a basic VPS.

**Self-hosting is cheaper AND more private.**

## Running Without Cloudflare

Don't want to use Cloudflare? Alternatives:

| Service | Notes |
|---------|-------|
| **Tailscale Funnel** | Share a node publicly via Tailscale |
| **ngrok** | Quick tunnels, free tier limited |
| **Pagekite** | Open source alternative |
| **bore** | Simple Rust-based tunnel |
| **rathole** | Self-hosted relay |

Or run your own relay on a cheap VPS and let others use it.

## Bootstrap List

The MESH network will maintain a bootstrap list of community relays and directories:

```yaml
relays:
  - url: https://relay.mesh.community
    operator: "MESH Community"
    regions: [us-west, us-east, eu-west]
    
  - url: https://relay.example.org
    operator: "Example Org"
    regions: [eu-central]

directories:
  - url: https://directory.mesh.community
    operator: "MESH Community"
    topics: [all]
```

These don't exist yet. They will when the community runs them.

## Quick Start

**Fastest path to a reachable MESH node:**

```bash
# 1. Start your AMP node (e.g., amp-mini)
./amp-mini --port 8765

# 2. In another terminal, expose via Cloudflare
cloudflared tunnel --url http://localhost:8765

# 3. Note the URL it gives you
# Example: https://example-words.trycloudflare.com

# 4. Register in a MESH directory
curl -X POST https://directory.mesh.community/v1/register \
  -H "Content-Type: application/json" \
  -d '{
    "nodeId": "your-node-id",
    "endpoint": "https://example-words.trycloudflare.com",
    "topics": ["rust", "iot"]
  }'
```

That's it. You're on MESH. From your home. Behind NAT. No ports opened.

---

## Summary

**MESH is designed for the real world:**
- Most people are behind NAT
- Most people can't open ports
- Most people won't pay for VPS
- Everyone can make outbound connections

**The solution: outbound tunnels everywhere.**

Cloudflare (or alternatives) becomes the connective tissue. The actual nodes—your knowledge, your sensors, your data—stay on hardware you control, in your home, on your terms.

**Zero infrastructure. Zero barriers. Maximum sovereignty.**
