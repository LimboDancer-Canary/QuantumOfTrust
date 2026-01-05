# Specification: Avatar‑Contract Session Keys (ACK) and Sweep‑to‑Safety

## 1. Purpose
Define deterministic per‑contract session identities (ACKs) that:
- control contract‑scoped stake/escrow/rewards, and
- sign contract messaging,
while preserving Avatar‑level trust accounting.

## 2. Scope

### 2.1 In scope
- Deterministic derivation requirements
- Aztec authoritative binding state and gating rules
- Nostr mirror event shapes using **ephemeral kinds**
- Sweep‑to‑safety / rotation flows

### 2.2 Out of scope
- The full dispute/arbitration protocol (see ADR)
- Payment rails / fiat integration

## 3. Terminology
- **Avatar:** long‑lived reputation identity.
- **Contract:** agreement between Avatars recorded on chain.
- **ACK:** Avatar‑Contract Keypair (session identity) for a given contract role.
- **Epoch:** monotonically increasing integer used to rotate ACK deterministically.
- **Mirror events:** Nostr events that reflect chain state but are not authoritative.

## 4. Goals and Non‑goals

### 4.1 Goals
- Compartmentalize value per contract.
- Preserve trust continuity per Avatar.
- Support deterministic recovery.
- Make chain authoritative; mirror is optional.

### 4.2 Non‑goals
- Guarantee that relays store mirror history.

## 5. Standards / Protocol Compliance

### 5.1 MUST
- Nostr event envelope semantics (NIP‑01)
- Nostr PoW support for abuse controls where required (NIP‑13)
- Nostr relay auth for private relays where required (NIP‑42)

### 5.2 SHOULD
- NIP‑06 compatible HD derivation support
- NIP‑46 remote signing support (client ↔ signer separation)
- NIP‑26 delegation support (optional; only when "act as root" semantics are required)

## 6. High‑Level Architecture

### 6.1 Components
- **Client/Wallet:** derives ACK keys, signs mirror events, submits Aztec tx.
- **QoTEscrow (Aztec):** authoritative bindings + escrow logic.
- **QoTAvatar (Aztec):** trust accounting and contract history.
- **Nostr Relays:** ephemeral mirror broadcast fabric.

### 6.2 Trust boundaries
- Correctness boundary is on‑chain. Nostr mirror cannot change authoritative state.

### 6.3 Client key handling and remote signing (recommended)
**Rule K1:** Browser clients MUST NOT store `master_secret`.

Recommended minimal architecture:
- A **Signer** (mobile app, hardware device, or local desktop app) holds `master_secret` and performs ACK derivation.
- The UI client requests public keys and signatures from the Signer using **NIP-46 remote signing**.
  - NIP-46 uses **ephemeral encrypted messages** (kind `24133`) between App and Signer.
  - The message `content` is encrypted and contains JSON-RPC-ish requests/responses.
- Cleanest safe default: the Signer signs Nostr mirror events and returns the signed event; the UI never handles long-lived secrets.

Optional complement:
- For "post as root" patterns (not required for ACK), use **NIP-26 delegation** to authorize a client keypair without exposing the root secret.

Aztec-side note:
- Treat Aztec as authoritative. Keep parent Avatar authority (rotation/revoke) in higher-assurance storage than routine ACK usage.

## 7. Deterministic Key Derivation

This section locks the **canonical inputs, encodings, and labels** so independent implementations produce identical ACK keys.

### 7.1 Canonical inputs
- `master_secret`: opaque bytes (from mnemonic / secure enclave / hardware). MUST NOT be transmitted or logged.
- `network_id`: ASCII string for domain separation (e.g., `"aztec:testnet"`, `"aztec:mainnet"`).
- `contract_id`: 32-byte identifier (bytes32 / Field-canonical bytes). MUST be stable.
- `role`: one-byte enum.
- `epoch`: u32, big-endian.

### 7.2 Canonical encodings
- `network_id_bytes = UTF8(network_id)`
- `contract_id_bytes = 32 bytes` (big-endian if derived from Field; hex values decode to 32 bytes)
- `role_byte`:
  - `0x01` = customer
  - `0x02` = provider
  - `0x03` = arbitrator
  - (others MAY be added; MUST be documented)
- `epoch_bytes = U32BE(epoch)`

### 7.3 Derivation families

#### Family A (DEFAULT): HKDF-SHA256 seed expansion → direct key mapping
This is the cleanest safe default and avoids HD-wallet foot-guns.

Define:
- `salt = network_id_bytes`
- `info_root = ASCII("QoT/ACK/v1") || 0x00 || contract_id_bytes || 0x00 || role_byte || 0x00 || epoch_bytes`
- `prk = HKDF-Extract(SHA256, salt=salt, IKM=master_secret)`
- `ack_seed   = HKDF-Expand(SHA256, PRK=prk, info=info_root || ASCII("/ack"),       L=32)`
- `nostr_seed = HKDF-Expand(SHA256, PRK=prk, info=info_root || ASCII("/nostr/sign"), L=32)`
- `aztec_seed = HKDF-Expand(SHA256, PRK=prk, info=info_root || ASCII("/aztec/auth"), L=32)`

**Mapping to keys**
- Nostr (secp256k1): map `nostr_seed` → valid scalar using rejection sampling (or equivalent) and compute pubkey.
- Aztec ACK: derive key material from `aztec_seed` using the chosen account-auth scheme.

Rotation is deterministic via `epoch++`.

#### Family B (OPTIONAL): NIP-06 hardened HD derivation (Nostr only)
To align with common Nostr libraries:
- Use NIP-06 path `m/44'/1237'/<account>'/0/0`.
- Compute `<account>` deterministically from the canonical inputs.
- MUST use hardened derivation for `<account>'`.

**Note:** Even in this mode, Aztec key material SHOULD still be derived using Family A (HKDF) to keep Aztec auth independent from Nostr wallet tooling.

### 7.4 Safety rules
- Derivation MUST be domain-separated (network id + purpose strings + contract context).
- Derivation MUST NOT use non-hardened HD steps for any key that may be exposed.
- Master-secret compromise MUST be treated as total compromise.

## 8. Aztec Authoritative State

### 8.1 QoTEscrow state
For each `contract_id`, QoTEscrow MUST store:
- `customer_avatar` (AztecAddress)
- `provider_avatar` (AztecAddress)
- `customer_ack_account` (AztecAddress)
- `provider_ack_account` (AztecAddress)
- `customer_ack_status` (unbound|active|revoked)
- `provider_ack_status` (unbound|active|revoked)
- `customer_ack_epoch` (u32)
- `provider_ack_epoch` (u32)

(Optionally, store `prev_*_ack_account` and `*_revoked_at` for lightweight auditability.)

### 8.2 Gating rules
Aztec function context exposes the sender as an `Option<AztecAddress>` in newer versions. Therefore, any gated function MUST explicitly handle the `None` case.

**Rule G1 (simple + safe default):** For any role-restricted **public** function, the contract MUST require a non-null sender:
- `let sender = context.msg_sender().expect("sender required");`

Then enforce:
- customer-role actions MUST require `sender == customer_ack_account` and `customer_ack_status == active`.
- provider-role actions MUST require `sender == provider_ack_account` and `provider_ack_status == active`.

**Rule G2 (rotation/revoke authority):** Rotation/revoke MUST be authorized by the parent Avatar address, not the ACK:
- `sender == customer_avatar` for customer-side rotation/revoke
- `sender == provider_avatar` for provider-side rotation/revoke

**Rule G3 (minimal complexity):** Do NOT add delegation machinery unless needed.
- If third-party execution is later required (e.g., a relayer paying fees), add an OPTIONAL `authwit` path for those specific functions.

### 8.3 Binding update rules
QoTEscrow MUST expose binding operations that are unambiguous per role.

- `bind_ack(contract_id, role, ack_account, ack_nostr_pub_hash, epoch)`
- `revoke_ack(contract_id, role, expected_epoch, reason)`

Authorization (MUST):
- `bind_ack` and `revoke_ack` MUST require a non-null sender.
- For `role=customer`, sender MUST equal `customer_avatar`.
- For `role=provider`, sender MUST equal `provider_avatar`.

Rules:
- `bind_ack` MUST set `{role}_ack_account = ack_account`, `{role}_ack_status = active`, `{role}_ack_epoch = epoch`.
- `revoke_ack` MUST set `{role}_ack_status = revoked` and MUST NOT change the epoch.
- Rotation is implemented as **revoke old (optional) + bind new at epoch+1** or as a single helper that performs both.

Binding updates MUST be recorded on‑chain.

### 8.4 Value ownership
All stake/escrow/reward notes/accounts for the contract MUST be owned by the appropriate ACK account(s) or by the escrow under a policy that is only releasable via ACK authority.

## 9. QoTAvatar Trust Accounting
QoTAvatar MUST:
- store contract history per Avatar
- apply trust outcomes ONLY to parent Avatars
- prevent double‑application per contract_id

`record_contract_completion(contract_id, customer_avatar, provider_avatar, outcome)` MUST be callable only by QoTEscrow and MUST verify that the avatars match the authoritative binding for contract_id.

## 10. Nostr Mirror Events (Ephemeral)

### 10.1 Kind assignments (QoT provisional)
- `kind 24210` — ACK‑BIND (ephemeral)
- `kind 24211` — ACK‑REVOKE (ephemeral)
- `kind 24212` — CONTRACT‑STATE (ephemeral)

These kinds are in the NIP‑01 ephemeral range and MUST NOT be relied on for persistence.

### 10.2 Tags (minimal)
We use a small, consistent tag vocabulary. Do not depend on relay indexing correctness.

**MUST include**
- `cid` — contract id (32-byte hex)

**SHOULD include**
- `tx` — Aztec transaction hash that produced the authoritative change (if known)
- `epoch` — epoch as a stringified u32
- `role` — `customer` | `provider` (for bind/revoke)
- `ack` — ACK identifier (see 10.3; may be pubkey OR hash depending on privacy posture)
- `az` — ACK Aztec address (optional; same privacy caveat)
- `state` — contract status for CONTRACT-STATE
- `t` — topic tag, e.g. `qot` (optional grouping)

### 10.3 Event schemas

#### ACK‑BIND (kind 24210)
- `pubkey`: Avatar pubkey (publisher)
- `tags` MUST include:
  - `["cid", "<contract_id_hex>"]`
  - `["role", "customer"|"provider"]`
  - `["epoch", "<u32>"]`
  - `["state", "active"]`
- `tags` SHOULD include:
  - `["ack", "<ack_pubkey_or_hash>"]`
  - `["az", "<ack_aztec_address_or_hash>"]`
  - `["tx", "<aztec_tx_hash>"]`
  - `["t", "qot"]`

**Client rule:** Treat as a hint and confirm binding on Aztec.

#### ACK‑REVOKE (kind 24211)
- `pubkey`: Avatar pubkey (publisher)
- `tags` MUST include:
  - `["cid", "<contract_id_hex>"]`
  - `["role", "customer"|"provider"]`
  - `["epoch", "<expected_epoch_u32>"]`
  - `["state", "revoked"]`
  - `["reason", "<string>"]`
- `tags` SHOULD include:
  - `["ack", "<ack_pubkey_or_hash>"]`
  - `["tx", "<aztec_tx_hash>"]`
  - `["t", "qot"]`

**Client rule:** Treat as a hint and confirm revocation on Aztec.

#### CONTRACT‑STATE (kind 24212)
- `pubkey`: mirror publisher (typically the client submitting the Aztec tx)
- `tags` MUST include:
  - `["cid", "<contract_id_hex>"]`
  - `["state", "Active"|"Disputed"|"Settled"|"Cancelled"]`
- `tags` SHOULD include:
  - `["tx", "<aztec_tx_hash>"]`
  - `["t", "qot"]`

**Client rule:** Treat as a hint and confirm state on Aztec.

### 10.4 Relay policy
Relays MAY require:
- **NIP-42 authentication** (AUTH challenge/response), where clients authenticate by sending an event of kind `22242`.
- **NIP-13 PoW** for anti-spam.
Relays SHOULD rate-limit these kinds and MAY drop them at will.

Clients MUST treat mirror events as hints and MUST confirm state from Aztec.

### 10.5 Privacy posture (what the mirror reveals)
We define two simple operating modes. Default to the safer one.

**Mode P (Public relays) — DEFAULT**
- Mirror events MUST NOT publish raw ACK Nostr pubkeys or raw ACK Aztec addresses.
- If included, `ack` and `az` tags MUST contain **hashes**:
  - `ack = SHA256(ack_nostr_pubkey_compressed)` as 32-byte hex
  - `az  = SHA256(ack_aztec_address_bytes)` as 32-byte hex
- `cid`, `role`, `epoch`, and `tx` are sufficient for UX hints without disclosing linkable identifiers.

**Mode R (Restricted/private relays)**
- On access-controlled relays (e.g., via NIP‑42), implementations MAY include raw `ack` pubkeys and raw `az` addresses to improve client convenience.
- Even in Mode R, correctness MUST still be confirmed from Aztec.

## 11. Sweep‑to‑Safety

This section defines the sweep/rotation state machine. Aztec is authoritative; Nostr is only a mirror.

### 11.1 Trigger
Sweep may be triggered by:
- user panic action
- anomaly detection
- counterparty dispute initiation

### 11.2 Contract lifecycle interaction
Define a minimal `contract_status` enum (names illustrative):
- `Active`
- `Disputed`
- `Settled`
- `Cancelled`

**Rule S1:** Sweeps (revoke/rotate) MUST be allowed in `Active` and `Disputed`.
- Rationale: compromise recovery must still work during disputes.

**Rule S2:** Sweeps SHOULD be disallowed after `Settled` (no authority needed), and MAY be allowed after `Cancelled` only to recover residual funds if any.

### 11.3 Procedure (per role)
1) **On‑chain (optional):** revoke compromised ACK for the role.
2) **Epoch bump:** set `new_epoch = expected_epoch + 1`.
3) **Derive replacement ACK** deterministically using `new_epoch`.
4) **On‑chain:** bind replacement ACK at `new_epoch`.
5) **On‑chain:** migrate remaining contract‑scoped value to replacement ACK or to Avatar treasury per policy.
6) **Mirror (optional):** emit ACK‑REVOKE and ACK‑BIND ephemeral events referencing the Aztec tx.

### 11.4 Freezing semantics
When the contract enters `Disputed`:
- **Value-moving operations** (e.g., withdrawals, milestone payouts) MUST be frozen unless executed by the escrow's dispute-resolution pathway.
- **Rotation/revoke** MUST remain available to the *parent Avatar* (Rule G2), because it is a security recovery operation.

### 11.5 Post‑conditions
- A revoked ACK can no longer pass gating checks.
- Contract‑scoped value is no longer spendable by revoked ACK.
- Trust outcomes still apply ONLY to parent Avatars.

## 12. Security Requirements
- Root secrets MUST be protected; ACK compromise MUST NOT reveal root.
- Derivation MUST be domain‑separated and collision‑resistant.
- All authoritative decisions MUST be verifiable on chain.

## 13. Observability & Operations
- Emit Aztec events/logs for binding/revocation/settlement.
- Provide metrics for: binds, revokes, sweeps, failed gating checks.

## 14. Test Requirements

### 14.1 Derivation conformance
- Implement Appendix A vectors and assert exact matches for `ack_seed`, `nostr_priv/pub`, and `aztec_seed`.
- Add negative tests for canonical encoding (wrong endianness, wrong role byte, wrong contract-id length).

### 14.2 On-chain gating (QoTEscrow)
Properties:
- **P1:** If `{role}_ack_status != active`, role actions MUST fail.
- **P2:** If `msg_sender != {role}_ack_account`, role actions MUST fail.
- **P3:** Rotation/revoke MUST only succeed when `msg_sender == {role}_avatar`.
- **P4:** After revoke, old ACK can never regain authority without an explicit bind.

### 14.3 Sweep-to-safety
- **S1:** In `Active`, revoke + bind at `epoch+1` restores role authority only to the new ACK.
- **S2:** In `Disputed`, value-moving ops are frozen, but rotation/revoke still works.
- **S3:** After sweep, remaining contract-scoped value is unspendable by the revoked ACK.

### 14.4 Trust accounting (QoTAvatar)
- **T1:** Contract outcomes update parent Avatars' trust histories only.
- **T2:** Outcomes are idempotent per `contract_id` (no double-apply).
- **T3:** Only QoTEscrow can call `record_contract_completion`, and it MUST validate the authoritative avatars for the contract.

### 14.5 Nostr mirror "chaos" tests
- Drop/reorder/replay mirror events: correctness MUST be unaffected.
- Ensure clients treat mirror as hints and confirm from Aztec.

### 14.6 Verification Suite (binding/rotation/recovery)

This suite is the **minimum** set of local tests that prove the design's core invariants: deterministic ACK derivation, on-chain authoritative gating, safe rotation/sweep, and mirror non-authority.

#### VS1 — Derivation conformance (cross-implementation)
- Use Appendix A vectors (with real filled outputs) and assert byte-for-byte matches for: `ack_seed`, `nostr_priv`, `nostr_pub`, `aztec_seed`.
- Negative cases MUST fail: wrong contract_id length, wrong epoch endianness, wrong role byte, wrong `network_id`.

#### VS2 — On-chain gating (QoTEscrow)
- `msg_sender == None` MUST fail for any role-gated public function.
- If `{role}_ack_status != active` then role actions MUST fail (P1).
- If `msg_sender != {role}_ack_account` then role actions MUST fail (P2).
- Rotation/revoke MUST only succeed when `msg_sender == {role}_avatar` (P3), and MUST fail if invoked by an ACK.

#### VS3 — Binding/rotation state machine
- Active → revoke(role, e) → bind(role, e+1): authority transfers to the new ACK only.
- Attempt to act with a revoked ACK MUST fail permanently (unless explicitly re-bound).
- Revoke with wrong `expected_epoch` MUST fail (prevents stale-client races).

#### VS4 — Sweep-to-safety (value + authority)
- After sweep, remaining contract-scoped value MUST be unspendable by the revoked ACK.
- In `Disputed`: value-moving ops MUST remain frozen, but rotation/revoke MUST work (S2).

#### VS5 — Trust accounting invariants (QoTAvatar)
- Outcomes update **parent Avatars only** (T1), are idempotent per `contract_id` (T2), and are callable only by QoTEscrow with avatar validation (T3).

#### VS6 — Nostr mirror chaos + privacy
- Drop/reorder/replay mirror events: correctness MUST be unaffected; clients MUST confirm from Aztec.
- Mirror is **ephemeral-only**: tests MUST NOT assume relay persistence.
- Mode P: if `ack`/`az` tags are present, they MUST be hashes (never raw identifiers).

#### VS7 — Remote signing boundary (recommended, if enabled)
- Browser/UI must never receive `master_secret`.
- UI can request: (a) derive ACK pubkeys for (cid, role, epoch), (b) sign mirror events; signer returns signed events.

## Appendix A — Derivation Test Vectors (format)
Publish test vectors so every client can confirm deterministic compatibility.

### A.1 Vector fields
- `network_id` (string)
- `contract_id` (32-byte hex)
- `role` (string + byte value)
- `epoch` (u32)
- `master_secret` (hex, ONLY in test vectors)
- `prk` (hex, optional — useful for debugging)
- `ack_seed` (hex)
- `nostr_priv` (hex)
- `nostr_pub` (hex)
- `aztec_seed` (hex)

### A.2 Example vector (illustrative)
```json
{
  "network_id": "aztec:testnet",
  "contract_id": "0000000000000000000000000000000000000000000000000000000000000042",
  "role": { "name": "customer", "byte": "01" },
  "epoch": 0,
  "master_secret": "00112233445566778899aabbccddeeff00112233445566778899aabbccddeeff",
  "ack_seed": "<fill>",
  "nostr_priv": "<fill>",
  "nostr_pub": "<fill>",
  "aztec_seed": "<fill>"
}
```

## Appendix B — Minimal safety warnings
- Never publish or log real `master_secret`.
- Do not use non-hardened HD steps for exposed child keys.
