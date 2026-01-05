# Deterministic Per‑Contract Session Keys ("ACK") for Trust‑Escrow Systems

## Abstract
Trust‑escrow systems that aim to preserve privacy while enforcing accountability face a recurring operational risk: the same long‑lived identity key that accrues reputation also becomes the single point of compromise for every contract. This paper proposes **deterministic per‑contract session keypairs**—**Avatar‑Contract Keys (ACKs)**—that compartmentalize contract‑scoped value (stake/escrow/rewards) and contract messaging without fragmenting long‑term reputation. The on‑chain system (Aztec) is authoritative for ACK bindings; Nostr is used as a low‑latency mirror for realtime UX. We analyze threat scenarios (hot‑key compromise, device loss, coercion, relay censorship/replay, disputed rotation), specify safety requirements for deterministic derivation, and outline an implementable architecture using hardened derivation and domain separation.

## Keywords
Deterministic keys, compartmentalization, escrow, account abstraction, Nostr, Aztec, ZK proofs, key rotation, sweep‑to‑safety.

## 1. Introduction
Privacy‑preserving contract networks typically bind reputation and control to cryptographic keys. This creates a tension:

- **Long‑lived keys** preserve continuity (reputation/trust history), but increase blast radius.
- **Short‑lived keys** reduce blast radius, but risk reputation fragmentation and poor discoverability.

We explore an approach that preserves continuity at the **Avatar** level while shifting contract‑scoped value and operational control to **per‑contract session keys**.

## 2. Background

### 2.1 Two planes: on‑chain authority vs off‑chain mirror
- **Aztec (authoritative):** binding state, escrow state, and final outcomes that update trust.
- **Nostr (mirror):** realtime dissemination, indexing, and client UX hints.

### 2.2 Event semantics in Nostr
Nostr defines event envelopes (id/sig/kind/tags/content). It also classifies kinds into ranges, including **ephemeral events** (not expected to be stored by relays) and **addressable events** (latest wins per `(kind,pubkey,d)`).

### 2.3 Deterministic key derivation in Nostr
NIP‑06 standardizes derivation of multiple keypairs from a mnemonic seed (BIP39/BIP32). This enables deterministic generation of many keys. However, deterministic schemes must be chosen carefully to avoid parent‑key compromise from leaked child keys.

### 2.4 Delegated signing vs session keys
NIP‑26 introduces delegated signing so that a root key can authorize other keypairs to sign events "on behalf" of the root, avoiding routine exposure of the root secret. ACKs are conceptually similar (session isolation), but in our design **ACKs are not posting "as the root"**; rather, they are posting as contract‑scoped proxies whose linkage is provable.

## 3. Problem Statement
We want a contract network in which:

1. An Avatar may participate in many contracts over time.
2. Each contract must hold stake/escrow/rewards in a way that is **compartmentalized**.
3. Outcomes must update the **Avatar's** trust history, not fragment it across ephemeral proxies.
4. If a contract proxy key is compromised, the system can **sweep contract‑scoped value to safety** and revoke the proxy, without needing to "re‑found" the Avatar identity.

## 4. Design Requirements

### 4.1 Core invariants
- **I1 (Trust continuity):** Trust/reputation accrues to the Avatar identity, never to ACK identities.
- **I2 (Value compartmentalization):** Stake/escrow/rewards for a contract are controlled by ACK identities.
- **I3 (Authoritative binding):** The binding `Avatar ↔ Contract ↔ ACK` is authoritative on‑chain.
- **I4 (Deterministic re‑derivation):** ACK keys are deterministically derivable from Avatar secret material + contract id + role + epoch.
- **I5 (Sweep‑to‑safety):** On compromise, contract value can be migrated to a new ACK or to the Avatar treasury under on‑chain policy.

### 4.2 Threat scenarios
- Hot key compromise
- Device loss / account recovery
- Coercion / extortion
- Relay censorship / replay / partial propagation
- Counterparty disputes binding/rotation
- Implementation bugs and misuse of deterministic derivation

## 5. Proposed Architecture

### 5.1 Roles and keys
Each Avatar maintains:
- **Root/Cold key material**: highest assurance storage.
- **Warm control key material**: can authorize rotations/sweeps.
- **ACK keypairs**: per‑contract session identities.

The system produces two proxy identities per contract:
- `ACK_customer(contractId)`
- `ACK_provider(contractId)`

### 5.2 Deterministic derivation
The recommended approach for deterministic ACK keys is:

1) Start from a **single master secret** stored at highest assurance (seed phrase / secure enclave / hardware).

2) Derive a **contract‑scoped seed** using a KDF (HKDF‑SHA256) with strong domain separation:

- `ack_seed = HKDF(master_seed, salt=network_id, info="QoT/ACK" || contract_id || role || epoch)`

3) Map `ack_seed` into curve‑specific private keys for:
- the Nostr signing key (secp256k1)
- the Aztec account key material (as required by Aztec account model)

4) When using BIP32 derivation (NIP‑06), use **hardened derivation only** for ACK children.

This avoids common foot‑guns of non‑hardened HD derivation while preserving stateless re‑derivation.

### 5.3 On‑chain authoritative binding
On contract entry, the on‑chain escrow records:
- Parent avatars (customer/provider)
- ACK accounts (customer/provider)
- Binding status and epoch

All contract actions requiring authority are gated by `msg_sender == ACK_account` for the appropriate role.

### 5.4 Nostr mirror: ephemeral realtime signals
Because Aztec is authoritative, Nostr is used as a low‑latency mirror:
- emit ephemeral "ACK‑BIND" signals when bindings are created
- emit ephemeral "ACK‑REVOKE" signals when bindings are revoked
- emit ephemeral "CONTRACT‑STATE" signals on lifecycle changes

Clients can subscribe to these for realtime UX, while still verifying authoritative state from Aztec.

## 6. Security Analysis

### 6.1 Hot key compromise
Blast radius is limited to the compromised contract proxy:
- attacker can attempt contract actions and move contract‑scoped value
- cannot affect other contracts or the Avatar treasury if policy is correct

Mitigation:
- revoke ACK on‑chain
- sweep remaining value to replacement ACK or treasury
- update mirror signals

### 6.2 Device loss
Deterministic derivation enables recovery by re‑deriving ACK keys. The master secret must remain recoverable (e.g., mnemonic backup).

### 6.3 Coercion
A coercer might demand signing; mitigate with:
- time‑delays on high‑value withdrawals
- "panic revoke" paths (immediate revoke + sweep)
- optional multi‑party recovery

### 6.4 Relay censorship / replay
Ephemeral events may be dropped or replayed. Because the chain is authoritative:
- clients treat Nostr mirror as hinting only
- all state transitions must be confirmed by reading on‑chain state

### 6.5 Disputed rotation
Counterparty disputes are handled via:
- on‑chain binding history
- escrow rules that define which party may rotate which role's ACK
- an arbitration/dispute mechanism (separate ADR)

## 7. Implementation Outline
- Add ACK bindings to `QoTEscrow` state.
- Add contract‑binding registry + replay protection to `QoTAvatar`.
- Emit Aztec events/logs for binding changes and settlement.
- Publish Nostr ephemeral mirror events from the client/wallet that submits the on‑chain transaction.

## 8. Evaluation and Testing
- Property tests: "trust accrues only to Avatar", "value moves only via ACK authority", "revocation blocks further ACK actions".
- Adversarial tests: compromised ACK cannot impact unrelated contracts.
- Mirror tests: Nostr events absent/reordered do not break correctness.

## 9. Future Work
- Formal verification of binding invariants.
- Multi‑recipient private event delivery strategies.
- Cross‑chain escrow portability.

## References

|  # | Reference                                                             | What it was used for in the research paper                                                                                                 | Publisher / Canonical source | Link                       |
| -: | --------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------ | ---------------------------- | -------------------------- |
|  1 | **NIP-01 — Basic Protocol Flow**                                      | Canonical Nostr event envelope semantics (id/sig/kind/tags/content) and baseline relay/client messaging assumptions                        | nostr-protocol/nips (GitHub) | ([GitHub][1])              |
|  2 | **NIP-06 — Basic key derivation from mnemonic seed phrase**           | Deterministic derivation framing for Nostr keys; motivation for deterministic multi-key schemes; ties into BIP39/BIP32                     | nostr-protocol/nips (GitHub) | ([GitHub][2])              |
|  3 | **NIP-13 — Proof of Work**                                            | Background on optional anti-spam / abuse controls for relay usage (PoW as a universal bearer proof)                                        | NIPs (mirrored)              | ([NIPs][3])                |
|  4 | **NIP-26 — Delegated Event Signing**                                  | Comparison point for "delegation vs session keys"; how delegation avoids exposing root keys while enabling proxy signing                   | nostr-protocol/nips (GitHub) | ([GitHub][4])              |
|  5 | **NIP-42 — Authentication of clients to relays (AUTH)**               | Relay authentication model for private/restricted relays; how clients authenticate and how relays challenge                                | nostr-protocol/nips (GitHub) | ([GitHub][5])              |
|  6 | **BIP-39 — Mnemonic code for generating deterministic keys**          | Underlying mnemonic → seed assumptions referenced via NIP-06's BIP39 dependency                                                            | bitcoin/bips (GitHub)        | ([GitHub][6])              |
|  7 | **BIP-32 — Hierarchical Deterministic Wallets**                       | HD wallet / child-key derivation background referenced via NIP-06's BIP32 dependency; "hardened vs non-hardened" safety considerations     | bitcoin/bips (GitHub)        | ([GitHub][7])              |
|  8 | **RFC 5869 — HKDF (HMAC-based Extract-and-Expand KDF)**               | Basis for HKDF-based "contract-scoped seed expansion" pattern (domain separation + extract/expand) used to avoid HD foot-guns              | IETF Datatracker             | ([IETF Datatracker][8])    |
|  9 | **SEC 2 v2.0 — Recommended Elliptic Curve Domain Parameters**         | Canonical parameter source for secp256k1 (relevant because Nostr uses secp256k1 for signatures/keys)                                       | SECG (PDF)                   | ([secg.org][9])            |
| 10 | **Aztec Docs — Emitting Events**                                      | Authoritative reference for emitting logs/events from Aztec contracts (used in the "authoritative chain + mirror" architecture discussion) | docs.aztec.network           | ([docs.aztec.network][10]) |
| 11 | **Aztec Docs — Understanding Function Context**                       | Authoritative reference for how `msg_sender` behaves across kernel iterations (including "empty" sender in first call)                     | docs.aztec.network           | ([docs.aztec.network][11]) |
| 12 | **Aztec Docs — Migration Notes (msg_sender is Option<AztecAddress>)** | Concrete confirmation that `msg_sender` became `Option<AztecAddress>` and why (native account abstraction / first private call)            | docs.aztec.network           | ([docs.aztec.network][12]) |

[1]: https://github.com/nostr-protocol/nips/blob/master/01.md "nips/01.md at master · nostr-protocol/nips"
[2]: https://github.com/nostr-protocol/nips/blob/master/06.md "nips/06.md at master · nostr-protocol/nips - GitHub"
[3]: https://nips.nostr.com/13 "NIP-13 - Proof of Work - NIPs (Nostr Improvement Proposals)"
[4]: https://github.com/nostr-protocol/nips/blob/master/26.md "nips/26.md at master · nostr-protocol/nips"
[5]: https://github.com/nostr-protocol/nips/blob/master/42.md "nips/42.md at master · nostr-protocol/nips"
[6]: https://github.com/bitcoin/bips/blob/master/bip-0039.mediawiki "bips/bip-0039.mediawiki at master · bitcoin/bips"
[7]: https://github.com/bitcoin/bips/blob/master/bip-0032.mediawiki "bips/bip-0032.mediawiki at master · bitcoin/bips"
[8]: https://datatracker.ietf.org/doc/html/rfc5869 "RFC 5869 - HMAC-based Extract-and-Expand Key ..."
[9]: https://www.secg.org/sec2-v2.pdf "SEC 2: Recommended Elliptic Curve Domain Parameters"
[10]: https://docs.aztec.network/devnet/developers/docs/guides/smart_contracts/how_to_emit_event "Emitting Events | Privacy-first zkRollup"
[11]: https://docs.aztec.network/developers/docs/concepts/smart_contracts/functions/context "Understanding Function Context | Privacy-first zkRollup"
[12]: https://docs.aztec.network/developers/docs/resources/migration_notes "Migration notes | Privacy-first zkRollup | Aztec Documentation"
