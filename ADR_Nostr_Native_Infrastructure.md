# ADR: Nostr-Native Infrastructure

**Status:** Accepted  
**Date:** December 30, 2025  
**Deciders:** Dennis  
**Context:** ZK Relay Integration for QoT on Nostr

---

## Decision

The ZK relay integration uses existing Nostr ecosystem patterns rather than external infrastructure. Every component maps to a Nostr-native solution.

| Component | Decision |
|-----------|----------|
| Caching | Deno KV (SQLite-backed), not Redis |
| Verification key storage | Blossom (NIP-B7), not IPFS |
| Horizontal scaling | strfry REUSE_PORT, not custom worker pools |
| Rate limiting | Existing strfry-policies, not custom implementation |
| DoS protection | NIP-13 Proof of Work, not custom queuing |
| Authentication | NIP-42, not custom session management |
| Event structure | Single addressable kind + d-tag, not multiple custom kinds |

---

## Context

The initial ZK relay integration specification proposed several external infrastructure components (Redis, IPFS, custom worker pools). An audit revealed that Nostr-native patterns already exist for each use case.

---

## Decisions by Component

### 1. Caching: Deno KV over Redis

**Original proposal**: Redis cluster for verification result caching

**Nostr-native alternative**: Deno KV (SQLite-backed)

**Rationale**: The strfry-policies library already uses Deno KV for stateful policies like `antiDuplicationPolicy` and `rateLimitPolicy`. This is the established pattern.

```typescript
const kv = await Deno.openKv();

async function getCachedVerification(proofHash: string): Promise<boolean | null> {
  const result = await kv.get(["zk-verify", proofHash]);
  return result.value as boolean | null;
}
```

**Performance option**: Mount SQLite on tmpfs for memory-speed access without Redis operational overhead.

**When to upgrade**: Multi-server deployments needing shared cache can use Deno Deploy's distributed KV (FoundationDB-backed) or NFS-mounted SQLite.

---

### 2. Verification Key Storage: Blossom over IPFS

**Original proposal**: IPFS CIDs for verification key references

**Nostr-native alternative**: Blossom (NIP-B7)

**Rationale**: Blossom is Nostr-native content-addressed storage. Files are identified by SHA-256 hash. Clients already understand this pattern for media.

```typescript
// Verification key reference in event tags
["vk", "blossom:<sha256-hash>", "https://blossom.example.com"]
```

**Benefits**:
- Native to ecosystem — no IPFS daemon to run
- Same hash-based retrieval model
- Multiple server redundancy built into protocol

---

### 3. Horizontal Scaling: strfry REUSE_PORT over Custom Workers

**Original proposal**: Custom verification worker pool

**Nostr-native alternative**: strfry's native multi-instance pattern

```bash
# Start 4 instances on same port (kernel load-balances)
for i in {1..4}; do
  strfry relay --config /etc/strfry.conf &
done
```

**Rationale**: strfry supports multiple instances sharing the same LMDB database via `SO_REUSEPORT`. The kernel handles load balancing. Each instance runs its own writePolicy plugin.

**Benefits**:
- Kernel-level load balancing (proven, efficient)
- No custom infrastructure to maintain
- ZK verification load distributed across CPU cores automatically

---

### 4. Rate Limiting: Existing Policies over Custom Implementation

**Original proposal**: Custom rate limiting for ZK events

**Nostr-native alternative**: Existing `rateLimitPolicy` from strfry-policies

```typescript
import { rateLimitPolicy } from 'strfry-policies';

[rateLimitPolicy, { 
  whitelist: ['127.0.0.1'],
  limit: 10,
  window: 60000
}]
```

**Rationale**: Battle-tested policy already handles the pattern. ZK events can use stricter limits via configuration.

---

### 5. DoS Protection: NIP-13 Proof of Work over Custom Queuing

**Original proposal**: Custom queue management for expensive ZK verification

**Nostr-native alternative**: NIP-13 Proof of Work requirement

```typescript
import { powPolicy } from 'strfry-policies';

[powPolicy, { 
  difficulty: 16,
  filter: (e) => e.tags.some(t => t[0] === "qt-type")
}]
```

**Rationale**: NIP-13 requires clients to do computational work before submission. This is protocol-level DoS protection that shifts cost to clients.

**Defense in depth**:
1. NIP-13 PoW (protocol level)
2. Rate limiting (policy level)
3. NIP-42 auth requirement (session level)

---

### 6. Authentication: NIP-42 over Custom Sessions

**Original proposal**: Custom authenticated session management

**Nostr-native alternative**: NIP-42 Client Authentication

**Rationale**: NIP-42 defines challenge-response authentication for relays. Authenticated users can get priority in verification pipeline.

```typescript
// Require auth before accepting ZK proof events
if (!event.authenticated && isZKEvent(event)) {
  return { action: 'reject', msg: 'AUTH required for ZK events' };
}
```

---

### 7. Event Structure: Addressable Events over Multiple Kinds

**Original proposal**: Custom event kinds 39001-39010

**Nostr-native alternative**: Single kind (30078) with `d` tag namespacing

```json
{
  "kind": 30078,
  "tags": [
    ["d", "qt:eligibility-proof:typescript_development"],
    ["qt-type", "eligibility_proof"],
    ["qt-circuit", "eligibility_v1"],
    ["qt-proof", "<base64-proof>"]
  ]
}
```

**Rationale**: NIP-78 defines kind 30078 for application-specific data. Using `d` tag namespacing follows the addressable event pattern (kinds 30000-39999) where only the latest event per kind/pubkey/d-tag is stored.

---

## Summary Table

| Component | Before | After | Benefit |
|-----------|--------|-------|---------|
| Caching | Redis | Deno KV | Zero new infrastructure |
| VK storage | IPFS | Blossom | Native to ecosystem |
| Scaling | Custom workers | REUSE_PORT | Kernel-level, proven |
| Rate limiting | Custom | Existing policies | Battle-tested |
| DoS protection | Custom queue | NIP-13 PoW | Protocol-level |
| Auth gating | Custom | NIP-42 | Standard flow |
| Event kinds | Multiple custom | Single + d-tag | Simpler indexing |

**Total external dependencies eliminated**: Redis, IPFS daemon, custom worker pool infrastructure

---

## Consequences

### Positive

1. **Zero external dependencies** — No Redis, IPFS, or custom infrastructure to deploy
2. **Ecosystem alignment** — Uses patterns other Nostr developers already understand
3. **Reduced operational complexity** — Fewer moving parts to monitor and maintain
4. **Proven patterns** — Each component is battle-tested in existing deployments

### Negative

1. **Deno KV limits** — Single-process only without shared storage; may need upgrade for multi-server
2. **Blossom maturity** — Newer than IPFS; fewer public servers

### Mitigation

- Deno Deploy provides distributed KV for production scale
- Multiple Blossom servers provide redundancy; can self-host

---

## Related Documents

- **ZK_Relay_Integration_Spec.md** — Full technical specification using these patterns
- **Nostr_Analysis.md** — Gap analysis for QoT on Nostr
- **Nostr_QoT_Gap_Analysis.md** — Detailed protocol gap analysis

---

## Summary

Every component of ZK relay integration maps to an existing Nostr pattern. This reduces operational complexity, aligns with ecosystem conventions, and eliminates external infrastructure dependencies while maintaining full ZK verification capability.
