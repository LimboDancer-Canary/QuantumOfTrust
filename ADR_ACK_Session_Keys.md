# ADR: Deterministic Avatar‑Contract Session Keys (ACK) with On‑Chain Authoritative Binding

## Decision
Adopt **deterministic per‑contract session keypairs (ACKs)** for each Avatar role in a contract. **Aztec is authoritative** for ACK bindings and escrow/value ownership. Nostr is a **mirror** using **ephemeral events** for realtime UX.

## Context
QoT requires accountability via keys without binding trust to real‑world identity. Using a single long‑lived key for all contracts creates unacceptable blast radius. Using per‑contract identities without continuity fragments reputation. We need compartmentalization without trust fragmentation.

## Drivers
- Reduce blast radius of key compromise.
- Preserve Avatar‑level trust continuity.
- Stateless recovery (deterministic) for durability and operability.
- Minimize reliance on third‑party indexers: chain is source of truth.
- Support realtime UX through a relay fabric without correctness dependence.

## Considered Options

### Option A — Single Avatar key for everything
- ✅ Simple
- ❌ Catastrophic blast radius; no compartmentalization

### Option B — Random per‑contract keys (non‑deterministic)
- ✅ Strong compartmentalization
- ❌ Recovery complexity; wallet state becomes critical

### Option C — Deterministic per‑contract keys (ACK) (chosen)
- ✅ Compartmentalization + recovery
- ✅ Enables consistent sweep/rotation policies
- ⚠ Requires careful derivation scheme (avoid HD foot‑guns)

### Option D — Delegated event signing (NIP‑26) as primary isolation
- ✅ Avoids exposing root key in clients
- ✅ Good complement
- ❌ Does not by itself define contract‑scoped value ownership semantics

## Decision Details
- For each contract and role, derive `ACK(contractId, role, epoch)` deterministically from the Avatar master secret using domain separation and hardened derivation.
- Store binding state in `QoTEscrow` and a minimal binding registry in `QoTAvatar`.
- Gate role actions by `msg_sender == role.ACK_account`.
- Accrue trust outcomes to the parent Avatar only.
- Emit Nostr ephemeral mirror events for ACK‑BIND / ACK‑REVOKE / CONTRACT‑STATE.

## Consequences

### Positive
- Hot‑key compromise is contained to one contract.
- Stateless recovery of ACK keys.
- Clean separation between value (ACK) and trust (Avatar).

### Negative
- Deterministic derivation mistakes can weaken security.
- Requires additional on‑chain state fields and enforcement logic.
- Ephemeral mirror events reduce historical discoverability (mitigated by on‑chain queries; mirror is ephemeral-only by design).

### Mitigations
- Mandate hardened derivation or HKDF‑based seed expansion.
- Provide a conformance test suite with known derivation vectors.

## Related Documents
- QuantumOfTrust_v10
- QoT_Aztec_Contract_Layer_Specification
- ZK_Relay_Integration_Spec
- ADR_Trust_Signal_Boundaries

## Summary
ACK session keys provide deterministic, compartmentalized control of contract‑scoped value, while Aztec authoritative bindings preserve correctness and Avatar‑level trust continuity.
