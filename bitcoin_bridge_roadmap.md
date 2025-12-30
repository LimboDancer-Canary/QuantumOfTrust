# Bitcoin Bridge Integration Roadmap

**Document Type**: Roadmap  
**Last Updated**: December 28, 2025  
**Status**: Planning Phase

---

## Overview

This roadmap outlines the path to enabling Bitcoin-backed stakes in the Quantum of Trust framework on Aztec Network. Bitcoin integration is a **Phase 2+ feature**, dependent on external infrastructure that is currently under development.

### Current Blockers

| Dependency | Owner | Status | Expected |
|------------|-------|--------|----------|
| Aztec execution layer mainnet | Aztec Labs | Testnet | Q1 2026 |
| Wormhole → Aztec bridge | Wormhole | Building | Q1-Q2 2026 |
| Arbitrum → Aztec bridge | Wormhole/Others | Building | Q1-Q2 2026 |

**Bottom line**: Bitcoin integration requires bridge infrastructure that doesn't exist yet. Plan for Q2 2026 at earliest.

---

## Target Architecture

### Primary Path: Trustless Bitcoin via Grail

```
BTC → Grail (ZK-verified) → grBTC on Arbitrum → [Bridge] → Aztec → Private grBTC → q⟨T⟩ stake
```

**Why this path**:
- Grail is **fully trustless** (ZK proofs verified on Bitcoin mainnet)
- Aligns with q⟨T⟩ philosophy: no custodians, no threshold signatures
- Grail Pro mainnet is live with $10M+ institutional BTC flows

### Secondary Path: tBTC via Wormhole

```
BTC → Threshold Network → tBTC on Ethereum → Wormhole → Aztec → Private tBTC → q⟨T⟩ stake
```

**Why as secondary**:
- Deeper liquidity ($400M+ established)
- Threshold trust model (51-of-100 signers) — secure but not trustless
- Good option for users who prefer established infrastructure

---

## Roadmap Phases

### Phase 1: Foundation (Current → Q1 2026)

**Goal**: Build q⟨T⟩ core on Aztec with multi-asset architecture ready for Bitcoin

| Task | Priority | Dependency | Notes |
|------|----------|------------|-------|
| Design multi-asset stake abstraction in Noir | High | None | Asset-agnostic from day one |
| Implement price oracle pattern for private circuits | High | None | Required for cross-asset normalization |
| Deploy q⟨T⟩ core contracts on Aztec testnet | High | None | ETH-only initially |
| Monitor Aztec execution layer progress | Medium | External | Track mainnet timeline |
| Monitor Wormhole-Aztec bridge development | Medium | External | Join Aztec Discord for updates |

**Deliverables**:
- Multi-asset stake struct supporting ETH, tBTC, grBTC asset types
- Price oracle integration pattern
- Working q⟨T⟩ contracts on Aztec testnet

### Phase 2: Mainnet Launch with ETH (Q1 2026)

**Goal**: Launch q⟨T⟩ on Aztec mainnet with ETH-only stakes

| Task | Priority | Dependency | Notes |
|------|----------|------------|-------|
| Deploy q⟨T⟩ on Aztec mainnet | High | Aztec execution mainnet | When ready |
| ETH bridging via L1 portal | High | Aztec portal contracts | Straightforward path |
| Build initial user base | High | Mainnet deployment | Prove the system |
| Continue monitoring bridge progress | Medium | External | Track for Phase 3 |

**Why ETH first**:
- ETH → Aztec path is straightforward (L1 portal)
- No external bridge dependencies
- Allows proving the system while waiting for Bitcoin infrastructure

### Phase 3: Bitcoin Integration (Q2 2026+)

**Goal**: Add trustless Bitcoin stakes via Grail

| Task | Priority | Dependency | Notes |
|------|----------|------------|-------|
| Validate Arbitrum → Aztec bridge is operational | Critical | Wormhole/bridge teams | Gate for all BTC integration |
| Implement grBTC portal contracts | High | Bridge operational | ~200-300 lines Noir |
| Test grBTC flow end-to-end on testnet | High | Portal contracts | Full path validation |
| Deploy grBTC support to mainnet | High | Testnet validation | Primary Bitcoin asset |
| Optionally add tBTC support | Medium | Wormhole-Aztec bridge | For deeper liquidity |

**Deliverables**:
- grBTC portal contracts (Aztec side)
- Working BTC → Grail → Arbitrum → Aztec → q⟨T⟩ flow
- Optional: tBTC support for liquidity diversity

### Phase 4: Optimization (Q3 2026+)

**Goal**: Improve Bitcoin integration based on usage patterns

| Task | Priority | Dependency | Notes |
|------|----------|------------|-------|
| Analyze stake asset distribution | Medium | Phase 3 live | ETH vs BTC usage |
| Optimize gas costs for BTC paths | Medium | Usage data | May require contract updates |
| Evaluate additional BTC representations | Low | Market evolution | Other bridges may emerge |
| Consider cross-asset contract support | Low | User demand | Alice stakes BTC, Bob stakes ETH |

---

## External Dependencies

### Aztec Network

| Component | Current Status | Required For | Notes |
|-----------|----------------|--------------|-------|
| Ignition Chain (consensus) | ✅ Mainnet (Nov 2025) | — | Already live |
| Execution layer | ⏳ Testnet | Phase 2+ | Smart contracts, bridges |
| AZTEC token | ✅ Live | — | Sale completed Dec 2025 |
| TGE | ⏳ Feb 2026 | — | Token transferability |

**Action**: Monitor Aztec announcements for execution layer mainnet date.

### BitcoinOS Grail

| Component | Current Status | Required For | Notes |
|-----------|----------------|--------------|-------|
| Grail Pro mainnet | ✅ Live (Q4 2025) | Phase 3 | $10M+ BTC flows |
| Arbitrum integration | ✅ Live | Phase 3 | Operational |
| BitSNARK verification | ✅ Live | Phase 3 | Proven on Bitcoin mainnet |

**Action**: No waiting required on Grail side. Ready when Aztec bridges are.

### Bridge Infrastructure

| Component | Current Status | Required For | Notes |
|-----------|----------------|--------------|-------|
| Wormhole → Aztec | 🔨 Building | Phase 3 | Official Aztec partner |
| TRAIN → Aztec | 🔨 Building | Phase 3 (alt) | Alternative option |
| Substance Labs → Aztec | 🔨 Building | Phase 3 (alt) | Alternative option |

**Action**: Monitor bridge team progress. First operational bridge unblocks Phase 3.

### tBTC (Secondary Path)

| Component | Current Status | Required For | Notes |
|-----------|----------------|--------------|-------|
| tBTC on Ethereum | ✅ Production | Phase 3 | $400M+ TVL |
| Wormhole tBTC gateway | ✅ Production | Phase 3 | 20+ chains supported |
| Wormhole → Aztec | 🔨 Building | Phase 3 | Same dependency as Grail path |

**Action**: tBTC path shares same Aztec bridge dependency. No separate tracking needed.

---

## Technical Specifications

### Multi-Asset Stake Structure (Noir)

```noir
// Asset type constants - defined now, used when assets available
global ASSET_ETH: Field = 0;
global ASSET_TBTC: Field = 1;
global ASSET_GRBTC: Field = 2;

struct ContractStake {
    amount: Field,
    asset_type: Field,
}

// Asset-agnostic validation - extend as assets become available
fn validate_stake(stake: ContractStake) -> bool {
    match stake.asset_type {
        ASSET_ETH => validate_eth_stake(stake),
        ASSET_TBTC => validate_tbtc_stake(stake),
        ASSET_GRBTC => validate_grbtc_stake(stake),
        _ => false,
    }
}

// Normalize stakes to common unit for trust computation
fn compute_stake_weight(stake: ContractStake, prices: OraclePrices) -> Field {
    let base_price = match stake.asset_type {
        ASSET_ETH => prices.eth_usd,
        ASSET_TBTC => prices.btc_usd,
        ASSET_GRBTC => prices.btc_usd,
        _ => 0,
    };
    stake.amount * base_price / PRECISION
}
```

### Portal Contract Requirements (Phase 3)

**grBTC Portal on Aztec** (~200-300 lines Noir):
- Claim L1→L2 messages from Arbitrum bridge
- Mint private grBTC notes
- Burn private notes for L2→L1 withdrawals
- Integration with q⟨T⟩ stake validation

**Pattern**: Follow Aztec token bridge tutorial architecture at:
https://docs.aztec.network/tutorials/codealong/contract_tutorials/advanced/token_bridge

---

## Trust Model Comparison

| Path | Bitcoin Custody | Bridge Trust | Overall |
|------|-----------------|--------------|---------|
| **Grail (Primary)** | Trustless (ZK on BTC) | Wormhole guardians | Mostly trustless |
| **tBTC (Secondary)** | 51-of-100 threshold | Wormhole guardians | Distributed trust |

**Recommendation**: Grail as primary because Bitcoin custody leg is fully trustless. tBTC as option for users preferring established liquidity.

---

## Open Design Decisions

To be resolved before Phase 3 implementation:

### 1. Stake Denomination

**Options**:
- A) Asset-native (stakes recorded in ETH, BTC, etc.)
- B) USD-normalized (all stakes converted to USD equivalent)
- C) Hybrid (asset-native storage, USD-normalized for trust calculation)

**Current lean**: Option C — preserves asset identity while enabling cross-asset comparison.

### 2. Price Oracle in Private Circuits

**Challenge**: How to get price data into private computation without revealing stake amounts?

**Options**:
- A) Public oracle with private stake amounts
- B) Commit-reveal pattern for price attestation
- C) Time-weighted average prices committed on-chain

**Action**: Research Aztec patterns for oracle integration in private functions.

### 3. Cross-Asset Contracts

**Question**: Should Alice (BTC stake) and Bob (ETH stake) be able to enter a contract together?

**Options**:
- A) Yes, with USD-normalized stake comparison
- B) No, require matching asset types
- C) Yes, but weight by asset volatility

**Current lean**: Option A — more flexible, enables broader participation.

---

## Risk Assessment

| Risk | Likelihood | Impact | Mitigation |
|------|------------|--------|------------|
| Aztec execution layer delayed | Medium | High | Build on testnet, ETH-first strategy |
| Bridge security incident | Low | High | Start with small stake limits |
| Grail liquidity insufficient | Medium | Medium | tBTC as alternative path |
| Price oracle manipulation | Medium | High | TWAP, multiple sources |
| Gas costs prohibitive | Medium | Medium | Batch operations, L2 optimization |

---

## Success Metrics (Phase 3)

| Metric | Target | Notes |
|--------|--------|-------|
| BTC stakes as % of total | >20% | Indicates BTC holder adoption |
| Average BTC stake size | >0.01 BTC | Meaningful participation |
| BTC bridge transaction success rate | >99% | Reliable infrastructure |
| Time from BTC lock to stake active | <1 hour | Acceptable UX |

---

## Timeline Summary

```
2025 Q4     ████████░░░░░░░░░░░░  Phase 1: Foundation (testnet)
2026 Q1     ░░░░░░░░████████░░░░  Phase 2: ETH mainnet launch
2026 Q2     ░░░░░░░░░░░░░░░░████  Phase 3: Bitcoin integration
2026 Q3+    ░░░░░░░░░░░░░░░░░░░░  Phase 4: Optimization
            ↑                   ↑
            Now                 BTC Live
```

**Key milestones**:
- **Now**: Multi-asset architecture design
- **Q1 2026**: Aztec execution mainnet, q⟨T⟩ launch with ETH
- **Q2 2026**: Aztec bridges operational, Bitcoin integration
- **Q3 2026+**: Optimization based on real usage

---

## References

**Aztec Network**
- Main site: https://aztec.network/
- Token bridge tutorial: https://docs.aztec.network/tutorials/codealong/contract_tutorials/advanced/token_bridge
- Status: Consensus mainnet (Nov 2025), execution testnet

**BitcoinOS**
- Main site: https://bitcoinos.build/
- Grail Pro: Mainnet Q4 2025
- Roadmap: EVM → UTXO → SolanaVM → MoveVM (no Aztec planned)

**Bridge Partners**
- Wormhole, TRAIN, Substance Labs building Aztec bridges
- Expected operational: Q1-Q2 2026

**Other**
- Threshold Network (tBTC): https://threshold.network/
- Wormhole: https://wormhole.com/
