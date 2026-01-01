# Integrating Noir/Barretenberg ZK Verification into Nostr Relays

## Technical Specification v2.1

*This specification describes how to extend Nostr relays to verify zero-knowledge proofs for Quantum of Trust (q⟨T⟩), using existing ecosystem patterns and minimal external infrastructure.*

**Changelog:**
- **v2.1** — Added relay-as-index-layer architecture (contract state events with single-letter tags), trust model for non-ZK events, client query patterns
- **v2.0** — Initial specification with ZK verification pipeline

---

## Executive Summary

Zero-knowledge proof verification can be added to Nostr relays using **strfry's writePolicy plugin system** with verification result caching via **Deno KV**. Verification keys are distributed via **Blossom** (NIP-B7), DoS protection uses **NIP-13 Proof of Work** combined with existing rate limiting, and authenticated sessions use **NIP-42**. This approach adds ZK verification capability with zero external infrastructure dependencies.

**Dual Role:** The relay serves two purposes:
1. **ZK Proof Verifier** — Validates eligibility proofs before accepting events
2. **Queryable Contract Index** — Indexes contract state events by single-letter tags (NIP-01), eliminating the need for external indexer infrastructure (The Graph, Postgres)

Both capabilities are additive to standard strfry behavior. Tag indexing is automatic per NIP-01; ZK verification is a writePolicy plugin.

---

## 1. Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                        Client                                    │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────────┐  │
│  │ NoirJS      │  │ NIP-07      │  │ Blossom Client          │  │
│  │ (proving)   │  │ (signing)   │  │ (VK retrieval)          │  │
│  └─────────────┘  └─────────────┘  └─────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                     strfry Relay                                 │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │                   Ingester Thread                         │   │
│  │  1. JSON decode                                           │   │
│  │  2. Schnorr signature verify                              │   │
│  │  3. writePolicy plugin ─────────────────────────────┐     │   │
│  └─────────────────────────────────────────────────────│─────┘   │
│                                                        │         │
│  ┌─────────────────────────────────────────────────────▼─────┐   │
│  │              ZK Verification Plugin (Deno)                │   │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────────┐   │   │
│  │  │ NIP-13 PoW  │─▶│ NIP-42 Auth │─▶│ Rate Limit      │   │   │
│  │  │ Check       │  │ Check       │  │                 │   │   │
│  │  └─────────────┘  └─────────────┘  └────────┬────────┘   │   │
│  │                                              │            │   │
│  │  ┌─────────────┐  ┌─────────────┐  ┌────────▼────────┐   │   │
│  │  │ Deno KV     │◀─│ bb.js       │◀─│ Cache Check     │   │   │
│  │  │ (SQLite)    │  │ verify()    │  │                 │   │   │
│  │  └─────────────┘  └─────────────┘  └─────────────────┘   │   │
│  └───────────────────────────────────────────────────────────┘   │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │              Writer Thread + Index (LMDB)                 │   │
│  │  - Stores events                                          │   │
│  │  - Indexes single-letter tags (c, p, s, k) per NIP-01    │   │
│  │  - Enables queries: #c, #p, #s, #k                        │   │
│  └──────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                     Blossom Server                               │
│                  (Verification Key Storage)                      │
│         Files addressed by SHA-256 hash (BUD-01)                │
└─────────────────────────────────────────────────────────────────┘
```

### 1.1 Relay as Index Layer

Aztec blockchain stores contract state but only supports key→value lookups (no secondary indexes). Rather than deploying external indexer infrastructure, the relay **indexes contract state events** using NIP-01's single-letter tag requirement:

```
Aztec Blockchain (source of truth)
       ▲
       │ tx confirmation
       │
┌─────────────────────────────┐
│  Nostria Client             │
│  - Writes to Aztec          │──────┐
│  - Mirrors state to Nostr   │      │ write-through
│  - Adds indexed tags        │      │
└─────────────────────────────┘      │
                                     │
       ┌─────────────────────────────┘
       │ tagged events
       ▼
┌─────────────────────────────┐
│  strfry Relay               │
│  - Indexes single-letter    │
│    tags automatically       │
│  - Standard NIP-01 queries  │
│  - ZK verification plugin   │
└─────────────────────────────┘
       │
       │ subscription/query
       ▼
┌─────────────────────────────┐
│  Any Nostr Client           │
│  - Query by #c, #p, #s, #k  │
│  - Real-time subscriptions  │
└─────────────────────────────┘
```

**Write-through mirroring:** The client that performs an Aztec transaction publishes the corresponding Nostr event. Aztec has no native push subscriptions—the PXE syncs by polling blocks. See `QoT_Nostria_Implementation_Plan.md` Section 10.0 for detailed rationale.

**No external indexer required.** Tag indexing is standard NIP-01 relay behavior; strfry implements this in its LMDB schema.

---

## 2. Event Structure

### 2.1 Single Addressable Kind with Namespace Tags

Following NIP-78 patterns, q⟨T⟩ uses **one addressable event kind** with `d` tag namespacing. This simplifies relay indexing and client implementation.

**Proposed kind: 30078** (application-specific data) or request allocation in 30000-39999 range.

#### Trust Score Event

```json
{
  "kind": 30078,
  "pubkey": "<avatar-pubkey>",
  "created_at": 1704067200,
  "tags": [
    ["d", "qt:score:software_engineering"],
    ["qt-type", "trust-score"],
    ["qt-domain", "software_engineering"],
    ["qt-version", "1.0.0"]
  ],
  "content": "<NIP-44 encrypted score data>",
  "sig": "<schnorr-signature>"
}
```

#### ZK Eligibility Proof Event

```json
{
  "kind": 30078,
  "pubkey": "<avatar-pubkey>",
  "created_at": 1704067200,
  "tags": [
    ["d", "qt:proof:eligibility:software_engineering"],
    ["qt-type", "zk-proof"],
    ["qt-circuit", "eligibility-v1"],
    ["qt-domain", "software_engineering"],
    ["qt-threshold", "75"],
    ["qt-vk", "<blossom-sha256-hash>"],
    ["nonce", "8192", "16"]
  ],
  "content": "<base64-encoded-noir-proof>",
  "sig": "<schnorr-signature>"
}
```

#### Contract State Event (Indexed)

Contract state events mirror on-chain Aztec contract state for queryability. They use **single-letter tags** which are automatically indexed by NIP-01 compliant relays, enabling queries like "all contracts where I'm customer."

```json
{
  "kind": 30078,
  "pubkey": "<contract-creator-pubkey>",
  "created_at": 1704067200,
  "tags": [
    ["d", "qt:contract:<contract-id>"],
    ["qt-type", "contract-state"],
    ["c", "<customer-avatar-pubkey>"],
    ["p", "<provider-avatar-pubkey>"],
    ["s", "active"],
    ["k", "software_engineering"]
  ],
  "content": "<NIP-44 encrypted contract details or state hash>",
  "sig": "<schnorr-signature>"
}
```

**Single-Letter Tags (NIP-01 Indexed):**

| Tag | Purpose | Query Example |
|-----|---------|---------------|
| `c` | Customer Avatar pubkey | `{"#c": ["<my-pubkey>"]}` |
| `p` | Provider Avatar pubkey | `{"#p": ["<my-pubkey>"]}` |
| `s` | Status (pending, active, completed, disputed, cancelled) | `{"#s": ["active"]}` |
| `k` | Skill domain | `{"#k": ["software_engineering"]}` |

Per NIP-01: "all single-letter (a-z, A-Z) key tags are expected to be indexed by relays." strfry's LMDB schema includes these tags in its PackedEvent format for efficient indexing.

### 2.2 Tag Semantics

**Multi-Letter Tags (Application-Specific):**

| Tag | Purpose | Required |
|-----|---------|----------|
| `d` | Addressable identifier with namespace | Yes |
| `qt-type` | Event type: `trust-score`, `zk-proof`, `contract-state`, `attestation` | Yes |
| `qt-circuit` | Circuit identifier for proof verification | For proofs |
| `qt-domain` | Skill domain (scopes trust independently) | Yes |
| `qt-threshold` | Public input: minimum trust threshold claimed | For proofs |
| `qt-vk` | Blossom SHA-256 hash of verification key | For proofs |
| `nonce` | NIP-13 Proof of Work (required for ZK events) | For proofs |

**Single-Letter Tags (NIP-01 Indexed):**

| Tag | Purpose | Indexed | Used In |
|-----|---------|---------|---------|
| `c` | Customer Avatar pubkey | ✅ Yes | contract-state |
| `p` | Provider Avatar pubkey | ✅ Yes | contract-state |
| `s` | Contract status | ✅ Yes | contract-state |
| `k` | Skill domain (queryable) | ✅ Yes | contract-state |

Single-letter tags enable efficient relay queries without external indexer infrastructure.

### 2.3 Namespace Conventions

```
qt:score:<domain>                    # Trust scores
qt:proof:eligibility:<domain>        # Eligibility proofs
qt:proof:membership:<dao-id>         # DAO membership proofs
qt:contract:<contract-id>            # Contract state (indexed)
qt:attestation:<attester>:<domain>   # Trust attestations
qt:circuit:<circuit-id>              # Circuit metadata
```

### 2.4 Trust Model for Contract State Events

Contract state events are self-attested (any client can publish them). Since Aztec blockchain is the source of truth, three mechanisms ensure integrity:

| Mechanism | Layer | Description |
|-----------|-------|-------------|
| **Verification on read** | Client | Before high-stakes operations (accepting contracts, signing off), clients verify against Aztec on-chain state |
| **ZK proofs in events** | Optional | Contract state events MAY include a ZK proof of on-chain state in the content field |
| **Relay filtering** | Relay | ZK-verifier relays reject contract-state events with invalid proofs (if proof is present) |

**Client-Side Verification Example:**

```typescript
async function acceptContract(contractId: string): Promise<void> {
  // 1. Get from Nostr (fast, for display)
  const nostrState = await this.getContractFromRelay(contractId);
  
  // 2. Verify against Aztec (authoritative) before committing
  const aztecState = await this.aztec.getContractState(contractId);
  if (!this.statesMatch(nostrState, aztecState)) {
    throw new Error('Contract state mismatch - possible stale data');
  }
  
  // 3. Proceed with on-chain operation
  await this.aztec.acceptListing(contractId);
}
```

**Design Principle:** The Nostr layer is a queryable cache/index of Aztec state, not the source of truth. Aztec remains authoritative; Nostr provides discoverability and real-time subscriptions.

---

## 3. Verification Key Distribution via Blossom

### 3.1 Blossom Integration

Blossom (NIP-B7) provides content-addressed storage native to Nostr:

- Files identified by SHA-256 hash
- User-controlled server lists via kind:10063 events
- Automatic failover to mirrors if primary unavailable
- Supported by major clients (Primal, Damus, Amethyst)

### 3.2 Verification Key Upload Flow

```
1. Circuit author compiles Noir circuit
2. Extract verification key (1-2KB for UltraHonk)
3. Upload to Blossom server(s):
   
   POST https://blossom.example.com/upload
   Authorization: Nostr <signed-auth-event>
   Content-Type: application/octet-stream
   
   <verification-key-bytes>
   
   Response: {
     "sha256": "7508bd9d8b0ed6e0891a3b973adf6011b1e49f6174910d6a1eb722a4a2e30539",
     "url": "https://blossom.example.com/7508bd9d..."
   }

4. Publish circuit metadata event:
   {
     "kind": 30078,
     "tags": [
       ["d", "qt:circuit:eligibility-v1"],
       ["qt-type", "circuit-metadata"],
       ["qt-vk", "7508bd9d8b0ed6e0891a3b973adf6011b1e49f6174910d6a1eb722a4a2e30539"],
       ["qt-vk-url", "https://blossom.example.com/7508bd9d..."],
       ["qt-backend", "ultrahonk"],
       ["qt-noir-version", "1.0.0-beta.3"]
     ],
     "content": "<circuit description and public input schema>"
   }
```

### 3.3 Verification Key Retrieval

```typescript
async function getVerificationKey(vkHash: string): Promise<Uint8Array> {
  // 1. Check local cache
  const cached = await kv.get(["vk", vkHash]);
  if (cached.value) return cached.value;
  
  // 2. Fetch from Blossom
  const url = `https://blossom.primal.net/${vkHash}`;
  const response = await fetch(url);
  const vkBytes = new Uint8Array(await response.arrayBuffer());
  
  // 3. Verify hash matches
  const computedHash = await crypto.subtle.digest("SHA-256", vkBytes);
  const computedHex = bytesToHex(new Uint8Array(computedHash));
  if (computedHex !== vkHash) {
    throw new Error("Verification key hash mismatch");
  }
  
  // 4. Cache indefinitely (VKs are immutable)
  await kv.set(["vk", vkHash], vkBytes);
  
  return vkBytes;
}
```

---

## 4. Verification Caching with Deno KV

### 4.1 Deno KV for State Management

Deno KV is used by strfry-policies for `antiDuplicationPolicy` and `rateLimitPolicy`:

- Ships with Deno runtime (zero additional dependencies)
- SQLite-backed locally, FoundationDB on Deno Deploy
- Native TTL support via `expireIn` option
- Can mount on tmpfs for memory-speed access

### 4.2 Cache Implementation

```typescript
// Initialize Deno KV (uses SQLite locally)
const kv = await Deno.openKv("./zk-verify-cache.sqlite");

interface VerificationResult {
  isValid: boolean;
  verifiedAt: number;
  circuitId: string;
}

// Cache key: SHA-256(proof || vkHash || publicInputs)
function computeCacheKey(
  proof: Uint8Array,
  vkHash: string,
  publicInputs: string[]
): string {
  const data = new TextEncoder().encode(
    bytesToHex(proof) + vkHash + publicInputs.join(",")
  );
  const hash = await crypto.subtle.digest("SHA-256", data);
  return bytesToHex(new Uint8Array(hash));
}

async function getCachedVerification(
  cacheKey: string
): Promise<VerificationResult | null> {
  const result = await kv.get<VerificationResult>(["zk-verify", cacheKey]);
  return result.value;
}

async function setCachedVerification(
  cacheKey: string,
  result: VerificationResult
): Promise<void> {
  // Cache valid proofs indefinitely (proofs are deterministic)
  // Cache invalid proofs for 1 hour (prevent repeated attacks)
  const ttl = result.isValid ? undefined : 60 * 60 * 1000;
  
  await kv.set(["zk-verify", cacheKey], result, { expireIn: ttl });
}
```

### 4.3 Performance Optimization: tmpfs Mount

For maximum cache speed:

```bash
# Create tmpfs mount for cache database
mkdir -p /mnt/zk-cache
mount -t tmpfs -o size=512M tmpfs /mnt/zk-cache

# Plugin uses this path
const kv = await Deno.openKv("/mnt/zk-cache/zk-verify.sqlite");
```

This provides sub-millisecond cache lookups while maintaining the SQLite interface.

---

## 5. DoS Protection Pipeline

### 5.1 Defense-in-Depth with Existing NIPs

```
Incoming Event
     │
     ▼
┌─────────────────────────────────────────┐
│ 1. NIP-13 Proof of Work Check           │
│    Require difficulty ≥ 16 for ZK events│
│    Cost: ~50μs (bit counting)           │
│    Rejects: cheap spam                  │
└─────────────────────────────────────────┘
     │ pass
     ▼
┌─────────────────────────────────────────┐
│ 2. Schnorr Signature Verification       │
│    (strfry core, always runs)           │
│    Cost: ~50μs                          │
│    Rejects: unsigned/forged events      │
└─────────────────────────────────────────┘
     │ pass
     ▼
┌─────────────────────────────────────────┐
│ 3. NIP-42 Authentication Check          │
│    Require auth for ZK event submission │
│    Cost: ~10μs (session lookup)         │
│    Rejects: unauthenticated clients     │
└─────────────────────────────────────────┘
     │ pass
     ▼
┌─────────────────────────────────────────┐
│ 4. Rate Limit (rateLimitPolicy)         │
│    Per-pubkey limits for ZK events      │
│    Cost: ~100μs (SQLite lookup)         │
│    Rejects: flood attempts              │
└─────────────────────────────────────────┘
     │ pass
     ▼
┌─────────────────────────────────────────┐
│ 5. Structural Validation                │
│    Check proof size, tag format         │
│    Cost: ~10μs                          │
│    Rejects: malformed events            │
└─────────────────────────────────────────┘
     │ pass
     ▼
┌─────────────────────────────────────────┐
│ 6. Cache Lookup                         │
│    Check if proof already verified      │
│    Cost: ~100μs (SQLite/tmpfs)          │
│    Short-circuits: repeated proofs      │
└─────────────────────────────────────────┘
     │ cache miss
     ▼
┌─────────────────────────────────────────┐
│ 7. ZK Verification (expensive)          │
│    Barretenberg UltraHonk verify        │
│    Cost: 3-10ms                         │
│    Final validation step                │
└─────────────────────────────────────────┘
     │
     ▼
   Accept/Reject
```

### 5.2 NIP-13 PoW Requirement

Requiring Proof of Work for ZK events creates asymmetric cost:

- **Attacker cost**: ~100ms per event (PoW generation at difficulty 16)
- **Relay cost**: ~50μs per event (PoW verification)
- **Ratio**: 2000:1 attacker disadvantage

```typescript
import { powPolicy } from 'jsr:@nostrify/policies';

// Require PoW only for q<T> events
const qtPowPolicy = powPolicy({
  difficulty: 16,
  filter: (event) => event.tags.some(t => t[0] === "qt-type")
});
```

### 5.3 NIP-42 Authentication Gating

Require authentication before accepting ZK proof events:

```typescript
function requireAuthForZK(input: StrfryInput): StrfryOutput {
  const isQTEvent = input.event.tags.some(t => t[0] === "qt-type");
  
  if (isQTEvent && !input.authPubkey) {
    return {
      id: input.event.id,
      action: "reject",
      msg: "auth-required: ZK proof events require NIP-42 authentication"
    };
  }
  
  return { id: input.event.id, action: "accept" };
}
```

---

## 6. Complete Plugin Implementation

### 6.1 File Structure

```
/opt/strfry-qt/
├── policy.ts           # Main plugin entry point
├── zk-verify.ts        # Barretenberg verification wrapper
├── cache.ts            # Deno KV cache operations
├── blossom.ts          # Verification key fetcher
└── deno.json           # Dependencies
```

### 6.2 Main Policy Plugin

```typescript
#!/bin/sh
//bin/true; exec deno run -A "$0" "$@"

import { 
  powPolicy, 
  rateLimitPolicy, 
  pipeline, 
  readStdin, 
  writeStdout,
  type InputMessage,
  type OutputMessage
} from 'jsr:@nostrify/strfry';

import { verifyZKProof } from './zk-verify.ts';
import { getCachedVerification, setCachedVerification, computeCacheKey } from './cache.ts';
import { getVerificationKey } from './blossom.ts';

// Initialize Deno KV for caching
const kv = await Deno.openKv("/mnt/zk-cache/zk-verify.sqlite");

// Check if event is a q<T> ZK proof event
function isZKProofEvent(event: NostrEvent): boolean {
  return event.tags.some(t => t[0] === "qt-type" && t[1] === "zk-proof");
}

// Check if event is a contract state event (indexed for queries)
function isContractStateEvent(event: NostrEvent): boolean {
  return event.tags.some(t => t[0] === "qt-type" && t[1] === "contract-state");
}

// Check if event is any q<T> event
function isQTEvent(event: NostrEvent): boolean {
  return event.tags.some(t => t[0] === "qt-type");
}

// Extract ZK-related data from event
function extractZKData(event: NostrEvent) {
  const getTag = (name: string) => event.tags.find(t => t[0] === name)?.[1];
  
  return {
    proof: base64ToBytes(event.content),
    circuitId: getTag("qt-circuit"),
    vkHash: getTag("qt-vk"),
    threshold: getTag("qt-threshold"),
    domain: getTag("qt-domain")
  };
}

// Validate contract state event structure
function validateContractStateEvent(event: NostrEvent): string | null {
  const getTag = (name: string) => event.tags.find(t => t[0] === name)?.[1];
  
  // Required single-letter indexed tags
  const customer = getTag("c");
  const provider = getTag("p");
  const status = getTag("s");
  const skillDomain = getTag("k");
  
  if (!customer || !provider) {
    return "invalid: contract-state must have 'c' (customer) and 'p' (provider) tags";
  }
  
  if (!status) {
    return "invalid: contract-state must have 's' (status) tag";
  }
  
  const validStatuses = ["pending", "active", "completed", "disputed", "cancelled"];
  if (!validStatuses.includes(status)) {
    return `invalid: status must be one of: ${validStatuses.join(", ")}`;
  }
  
  return null; // Valid
}

// Main ZK verification policy
async function zkVerifyPolicy(msg: InputMessage): Promise<OutputMessage> {
  const { event } = msg;
  
  // Pass through non-q<T> events (standard Nostr events)
  if (!isQTEvent(event)) {
    return { id: event.id, action: "accept" };
  }
  
  // Handle contract-state events (indexed for queries)
  if (isContractStateEvent(event)) {
    // Validate structure
    const validationError = validateContractStateEvent(event);
    if (validationError) {
      return { id: event.id, action: "reject", msg: validationError };
    }
    
    // Contract-state events with embedded ZK proof get verified
    // (content may contain optional proof of on-chain state)
    const hasEmbeddedProof = event.tags.some(t => t[0] === "qt-vk");
    if (hasEmbeddedProof) {
      // Fall through to ZK verification below
    } else {
      // No proof - accept for indexing (client verifies against Aztec on read)
      return { id: event.id, action: "accept" };
    }
  }
  
  // Handle ZK proof events (require verification)
  if (!isZKProofEvent(event) && !isContractStateEvent(event)) {
    // Other q<T> event types (trust-score, attestation) - accept without ZK check
    return { id: event.id, action: "accept" };
  }
  
  // Require NIP-42 authentication for ZK events
  if (!msg.authPubkey) {
    return {
      id: event.id,
      action: "reject",
      msg: "auth-required: ZK proof events require authentication"
    };
  }
  
  try {
    const zkData = extractZKData(event);
    
    // Validate required fields
    if (!zkData.circuitId || !zkData.vkHash || !zkData.proof) {
      return {
        id: event.id,
        action: "reject",
        msg: "invalid: missing required ZK proof fields"
      };
    }
    
    // Check cache
    const cacheKey = await computeCacheKey(
      zkData.proof,
      zkData.vkHash,
      [zkData.threshold, zkData.domain]
    );
    
    const cached = await getCachedVerification(kv, cacheKey);
    if (cached !== null) {
      return cached.isValid
        ? { id: event.id, action: "accept" }
        : { id: event.id, action: "reject", msg: "invalid: ZK proof verification failed (cached)" };
    }
    
    // Fetch verification key from Blossom
    const vk = await getVerificationKey(zkData.vkHash);
    
    // Verify proof
    const isValid = await verifyZKProof(
      zkData.proof,
      vk,
      [zkData.threshold, zkData.domain]
    );
    
    // Cache result
    await setCachedVerification(kv, cacheKey, {
      isValid,
      verifiedAt: Date.now(),
      circuitId: zkData.circuitId
    });
    
    return isValid
      ? { id: event.id, action: "accept" }
      : { id: event.id, action: "reject", msg: "invalid: ZK proof verification failed" };
      
  } catch (error) {
    console.error(`ZK verification error: ${error.message}`);
    return {
      id: event.id,
      action: "reject",
      msg: `error: ZK verification failed - ${error.message}`
    };
  }
}

// Compose pipeline with existing policies
for await (const msg of readStdin()) {
  const result = await pipeline(msg, [
    // 1. Require PoW for ZK proof events only (expensive to verify)
    [powPolicy, { 
      difficulty: 16,
      filter: (e) => e.tags.some(t => t[0] === "qt-type" && t[1] === "zk-proof")
    }],
    
    // 2. Rate limit by pubkey (applies to all q<T> events)
    [rateLimitPolicy, { 
      whitelist: ['127.0.0.1'],
      limit: 100,  // Higher limit for contract-state updates
      window: 60000
    }],
    
    // 3. Stricter rate limit for ZK proof events
    [rateLimitPolicy, { 
      whitelist: ['127.0.0.1'],
      limit: 10,
      window: 60000,
      filter: (e) => e.tags.some(t => t[0] === "qt-type" && t[1] === "zk-proof")
    }],
    
    // 4. ZK verification and contract-state validation
    zkVerifyPolicy
  ]);
  
  writeStdout(result);
}
```

### 6.3 Barretenberg Verification Wrapper

```typescript
// zk-verify.ts
import { UltraHonkBackend } from 'npm:@aztec/bb.js';

// Cache compiled backends by circuit
const backends = new Map<string, UltraHonkBackend>();

export async function verifyZKProof(
  proof: Uint8Array,
  verificationKey: Uint8Array,
  publicInputs: string[]
): Promise<boolean> {
  try {
    // Get or create backend for this VK
    const vkHash = await hashBytes(verificationKey);
    
    let backend = backends.get(vkHash);
    if (!backend) {
      backend = new UltraHonkBackend(verificationKey);
      backends.set(vkHash, backend);
    }
    
    // Verify proof
    const isValid = await backend.verifyProof({
      proof,
      publicInputs: publicInputs.map(BigInt)
    });
    
    return isValid;
  } catch (error) {
    console.error("Proof verification error:", error);
    return false;
  }
}

async function hashBytes(data: Uint8Array): Promise<string> {
  const hash = await crypto.subtle.digest("SHA-256", data);
  return bytesToHex(new Uint8Array(hash));
}

function bytesToHex(bytes: Uint8Array): string {
  return Array.from(bytes)
    .map(b => b.toString(16).padStart(2, '0'))
    .join('');
}
```

### 6.4 strfry Configuration

```toml
# strfry.conf

[relay]
bind = "127.0.0.1"
port = 7777

[relay.info]
name = "q<T> Enabled Relay"
description = "Nostr relay with Quantum of Trust ZK verification"
supported_nips = [1, 2, 9, 11, 13, 42, 78]

[writePolicy]
plugin = "/opt/strfry-qt/policy.ts"

[events]
maxEventSize = 131072  # 128KB to accommodate ZK proofs
```

---

## 7. Horizontal Scaling

### 7.1 strfry Multi-Instance

strfry supports multiple instances sharing the same LMDB database via `SO_REUSEPORT`:

```bash
# Start 4 instances on same port (kernel load-balances)
for i in {1..4}; do
  strfry relay --config /etc/strfry.conf &
done
```

Each instance runs its own writePolicy plugin, distributing ZK verification load across CPU cores.

### 7.2 Shared Cache for Multi-Server Deployments

For multi-server deployments, the SQLite cache can be placed on shared storage:

```typescript
// Point all instances to shared NFS/EFS mount
const kv = await Deno.openKv("/mnt/shared/zk-verify.sqlite");
```

Alternatively, use Deno Deploy's distributed KV (FoundationDB-backed) for global deployments.

---

## 8. Client Implementation Guide

### 8.1 Generating and Submitting ZK Proofs

```typescript
import { NoirJS } from '@noir-lang/noir_js';
import { UltraHonkBackend } from '@aztec/bb.js';
import { finalizeEvent, generateSecretKey, getPublicKey } from 'nostr-tools';

async function submitEligibilityProof(
  circuit: CompiledCircuit,
  privateInputs: { trustScore: number; history: ContractOutcome[] },
  threshold: number,
  domain: string,
  relay: string
) {
  // 1. Generate proof
  const backend = new UltraHonkBackend(circuit.bytecode);
  const noir = new NoirJS(circuit);
  
  const { witness } = await noir.execute({
    trust_score: privateInputs.trustScore,
    threshold: threshold,
    // ... other private inputs
  });
  
  const proof = await backend.generateProof(witness);
  
  // 2. Get verification key hash (should match published circuit metadata)
  const vk = await backend.getVerificationKey();
  const vkHash = await sha256Hex(vk);
  
  // 3. Mine PoW (NIP-13, difficulty 16)
  const eventTemplate = {
    kind: 30078,
    created_at: Math.floor(Date.now() / 1000),
    tags: [
      ["d", `qt:proof:eligibility:${domain}`],
      ["qt-type", "zk-proof"],
      ["qt-circuit", "eligibility-v1"],
      ["qt-domain", domain],
      ["qt-threshold", threshold.toString()],
      ["qt-vk", vkHash],
      ["nonce", "0", "16"]  // Will be updated by PoW miner
    ],
    content: bytesToBase64(proof)
  };
  
  const minedEvent = await minePoW(eventTemplate, 16);
  
  // 4. Sign with NIP-07 or local key
  const signedEvent = await window.nostr.signEvent(minedEvent);
  
  // 5. Submit to relay (requires NIP-42 auth)
  const ws = new WebSocket(relay);
  
  ws.onopen = async () => {
    // Authenticate first
    const authEvent = await createAuthEvent(relay, challenge);
    ws.send(JSON.stringify(["AUTH", authEvent]));
    
    // Then submit proof
    ws.send(JSON.stringify(["EVENT", signedEvent]));
  };
}

async function minePoW(event: UnsignedEvent, difficulty: number): Promise<UnsignedEvent> {
  let nonce = 0;
  while (true) {
    event.tags = event.tags.map(t => 
      t[0] === "nonce" ? ["nonce", nonce.toString(), difficulty.toString()] : t
    );
    
    const id = getEventHash(event);
    if (countLeadingZeroBits(id) >= difficulty) {
      return event;
    }
    nonce++;
  }
}
```

### 8.2 Verifying Proofs Client-Side

Clients can optionally verify proofs locally for trust-minimized operation:

```typescript
import { UltraHonkBackend } from '@aztec/bb.js';

async function verifyProofEvent(event: NostrEvent): Promise<boolean> {
  // 1. Extract proof data
  const vkHash = event.tags.find(t => t[0] === "qt-vk")?.[1];
  const proof = base64ToBytes(event.content);
  
  // 2. Fetch VK from Blossom
  const vk = await fetchFromBlossom(vkHash);
  
  // 3. Verify hash
  if (await sha256Hex(vk) !== vkHash) {
    throw new Error("VK hash mismatch");
  }
  
  // 4. Verify proof
  const backend = new UltraHonkBackend(vk);
  const threshold = event.tags.find(t => t[0] === "qt-threshold")?.[1];
  
  return backend.verifyProof({
    proof,
    publicInputs: [BigInt(threshold)]
  });
}
```

### 8.3 Querying Contract State (Indexed Events)

The relay automatically indexes single-letter tags per NIP-01. Clients query contracts using standard Nostr filters:

```typescript
import { SimplePool } from 'nostr-tools';

async function queryMyContracts(
  myPubkey: string,
  relays: string[]
): Promise<NostrEvent[]> {
  const pool = new SimplePool();
  
  // Query contracts where I'm customer OR provider
  const events = await pool.querySync(relays, [
    { kinds: [30078], "#c": [myPubkey] },  // I'm customer
    { kinds: [30078], "#p": [myPubkey] }   // I'm provider
  ]);
  
  // Dedupe (same contract may match both filters if I'm both parties)
  const unique = new Map(events.map(e => [e.id, e]));
  return [...unique.values()];
}

// Subscribe to real-time contract updates
function subscribeToMyContracts(
  myPubkey: string,
  relays: string[],
  onEvent: (event: NostrEvent) => void
): () => void {
  const pool = new SimplePool();
  
  const sub = pool.subscribeMany(relays, [
    { kinds: [30078], "#c": [myPubkey] },
    { kinds: [30078], "#p": [myPubkey] }
  ], {
    onevent: onEvent
  });
  
  // Return unsubscribe function
  return () => sub.close();
}

// Query by status and skill domain
async function queryActiveContractsInDomain(
  domain: string,
  relays: string[]
): Promise<NostrEvent[]> {
  const pool = new SimplePool();
  
  return pool.querySync(relays, [{
    kinds: [30078],
    "#s": ["active"],
    "#k": [domain]
  }]);
}
```

**Trust Verification:** For high-stakes operations, verify against Aztec before proceeding:

```typescript
async function acceptContractWithVerification(
  contractId: string,
  nostrEvent: NostrEvent,
  aztecService: AztecService
): Promise<void> {
  // 1. Parse Nostr event
  const nostrState = parseContractEvent(nostrEvent);
  
  // 2. Verify against Aztec (source of truth)
  const aztecState = await aztecService.getContractState(contractId);
  
  if (nostrState.status !== aztecState.status ||
      nostrState.customer !== aztecState.customer.toString() ||
      nostrState.provider !== aztecState.provider.toString()) {
    throw new Error('State mismatch: Nostr cache is stale');
  }
  
  // 3. Proceed with on-chain operation
  await aztecService.acceptListing(contractId);
}
```

---

## 9. Future NIP Proposal Outline

To standardize this integration, a NIP should be drafted covering:

### NIP-XX: Zero-Knowledge Proof Verification

**Abstract**: This NIP defines event structures and relay behaviors for publishing and verifying zero-knowledge proofs on Nostr.

**Sections**:

1. **Motivation**: Privacy-preserving attestations, trust scores, credentials
2. **Event Structure**: Tags, content format, addressable kind usage
3. **Proof Systems**: Supported backends (UltraHonk, Groth16)
4. **Verification Key Distribution**: Blossom integration, hash verification
5. **Relay Behavior**: 
   - SHOULD verify proofs before accepting
   - SHOULD cache verification results
   - MAY require NIP-13 PoW for ZK events
   - MAY require NIP-42 authentication
6. **Client Behavior**:
   - SHOULD verify proofs locally when possible
   - MUST include PoW if relay requires
   - MUST authenticate if relay requires
7. **Security Considerations**: DoS, malformed proofs, VK trust

---

## 10. Performance Characteristics

### 10.1 Expected Latencies

| Operation | Time | Notes |
|-----------|------|-------|
| NIP-13 PoW check | ~50μs | Bit counting |
| Schnorr signature verify | ~50μs | libsecp256k1 |
| NIP-42 session lookup | ~10μs | In-memory |
| Rate limit check | ~100μs | SQLite lookup |
| Cache hit | ~100μs | SQLite/tmpfs |
| VK fetch (cached) | ~100μs | SQLite |
| VK fetch (Blossom) | ~50-200ms | Network, one-time |
| ZK verification | 3-10ms | UltraHonk |

### 10.2 Throughput Estimates

With the defense-in-depth pipeline:

- **Spam rejection rate**: >99% at PoW/rate-limit layers
- **Cache hit rate**: >90% for repeated proofs
- **ZK verifications/second**: 100-300 (per strfry instance)
- **Total throughput with 4 instances**: 400-1200 ZK verifications/second

---

## 11. Deployment Checklist

### Phase 1: Single Instance MVP

- [ ] Install Deno on relay server
- [ ] Clone q⟨T⟩ strfry plugin repository
- [ ] Configure tmpfs mount for cache (optional, recommended)
- [ ] Update strfry.conf with writePolicy plugin path
- [ ] Set up Blossom server or use public server for VKs
- [ ] Upload verification keys to Blossom
- [ ] Publish circuit metadata events
- [ ] Test with sample proof events
- [ ] Monitor logs for verification errors

### Phase 2: Production Hardening

- [ ] Enable NIP-42 authentication requirement
- [ ] Tune NIP-13 PoW difficulty based on load
- [ ] Configure rate limits based on traffic patterns
- [ ] Set up monitoring for cache hit rates
- [ ] Add alerting for verification failures
- [ ] Document backup/recovery procedures

### Phase 3: Horizontal Scaling

- [ ] Deploy multiple strfry instances with REUSE_PORT
- [ ] Configure shared storage for cache (NFS/EFS) or use Deno Deploy KV
- [ ] Set up load balancer health checks
- [ ] Test failover scenarios
- [ ] Establish capacity planning metrics

---

*This specification integrates q⟨T⟩ ZK verification with established Nostr patterns, minimizing operational complexity while maintaining security and performance.*
