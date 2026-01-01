# Implementation Plan: Extending Nostria with QoT Professional Network Pages

*A technical roadmap for integrating Quantum of Trust into the Nostria Nostr client*

**Version:** 2.0  
**Date:** January 1, 2026

---

## Executive Summary

This document outlines the implementation timeline and phases for adding QoT Professional Network functionality to Nostria, an Angular-based Nostr client.

**Key Architectural Decisions:**
- **Aztec blockchain is the source of truth** for trust state, contracts, and escrow
- **Nostr is the discovery/communication layer** for listings, proofs, and negotiation
- QoT pages are **architecturally separate** from existing Nostria features
- New route namespace: `/qot/*`
- Client uses `qot_circuits.nr` (compiled to WASM) for local computation and proof generation

For technical specifications including service interfaces, component architecture, event schemas, and data flows, see **QoT_Nostria_Client_Specification.md**.

---

## Implementation Phases

### Phase 0: Aztec Infrastructure (2-3 weeks)

**Deliverables:**
1. Deploy QoTRegistry, QoTEscrow, QoTAvatar contracts to Aztec testnet
2. Compile `qot_circuits.nr` to WASM via nargo
3. Create `@qot/circuits-wasm` npm package
4. Implement `AztecService` with read operations
5. Implement `QotCircuitsService` wrapper

**Testing:**
- Contract deployment verification
- RPC connectivity tests
- Circuit compilation validation

### Phase 1: Core Infrastructure (4-6 weeks)

**Deliverables:**
1. `QotService` with basic event handling
2. `TrustCacheService` with IndexedDB integration
3. Trust Dashboard page with read-only score display
4. Avatar Public Profile page
5. Event schema implementation (kind 30078)

**Testing:**
- Unit tests for services
- Integration tests with mock relays
- IndexedDB persistence tests

### Phase 2: Contract System (4-6 weeks)

**Deliverables:**
1. `ContractStateService` with lifecycle management
2. Contract Marketplace page
3. Contract Detail page with phase timeline
4. Provider Acceptance workflow
5. Mutual sign-off dialog
6. Contract amendment flow
7. Milestone review page
8. Task-level outcome display

**Testing:**
- Contract state machine tests
- Multi-party signing flow tests
- Edge case handling (disputes, timeouts)
- Milestone deadline tests

### Phase 3: ZK Proving (4-6 weeks)

**Deliverables:**
1. `ZkProvingService` with NoirJS/bb.js integration
2. Eligibility Proof page
3. Client-side verification
4. Verification key management via Blossom
5. Proof caching and sharing

**Testing:**
- Proof generation/verification tests
- Browser compatibility (Chrome, Firefox, Safari)
- Performance benchmarking (~5 seconds target)

### Phase 4: Customer Trust & Teams (3-4 weeks)

**Deliverables:**
1. Customer Dashboard page
2. Bidirectional trust display
3. Provider Calibration view
4. Task-level difficulty assessment UI
5. Team-based contract support

**Testing:**
- Customer metric calculations
- Team attribution logic
- Verification weight calculations

### Phase 5: Organizational Trust (3-4 weeks)

**Deliverables:**
1. DAO Profile page
2. Aggregate trust calculation
3. Member roster with role weights
4. Multi-party contract coordination

**Testing:**
- DAO aggregate trust tests
- Role-based outcome weighting
- Member permission validation

### Phase 6: Dispute Resolution (2-3 weeks)

**Deliverables:**
1. Dispute initiation dialog
2. Evidence upload component
3. Arbitrator selection UI
4. Appeal workflow
5. Dispute status tracking

**Testing:**
- Deadline enforcement tests
- Multi-arbitrator panel tests
- Appeal fee calculation

---

## NIP Proposal Roadmap

### Step 1: Reference Implementation

Implement in Nostria client as described in QoT_Nostria_Client_Specification.md.

### Step 2: Verifier Relay

Implement strfry plugin per ZK_Relay_Integration_Spec.md.

### Step 3: Draft NIP-XX

> **Note:** The implementation uses kind 30078 with d-tag namespacing for rapid deployment without requiring NIP allocation. The NIP proposal below requests dedicated kinds (30090-30094) for ecosystem standardization. Migration path: clients would support both patterns during transition.

```markdown
# NIP-XX: Skill-Domain Trust and ZK Eligibility Proofs

## Abstract
This NIP defines event kinds and structures for peer-to-peer trust 
attestations, contract outcome records, computed trust scores, and 
zero-knowledge eligibility proofs within skill domains.

## Event Kinds
- 30090: Trust attestation (peer-to-peer)
- 30091: Contract outcome (encrypted)
- 30092: Computed trust score
- 30093: ZK eligibility proof
- 30094: Circuit metadata

## Tag Semantics
[Per QoT_Nostria_Client_Specification.md Section 5]

## Relay Behavior
- Relays MAY verify ZK proofs before accepting kind 30093 events
- Relays SHOULD require NIP-13 PoW for kind 30093 events
- Relays MAY require NIP-42 authentication for write access

## Client Behavior
- Clients SHOULD verify ZK proofs locally if relay doesn't
- Clients SHOULD aggregate events from multiple relays using CRDT merge
- Clients MAY weight attestations by attestor trust
```

### Step 4: Community Engagement

- Present at Nostr developer forums
- Participate in WoT-a-thon (April 2026)
- Iterate based on feedback

---

## Success Metrics

### Technical Metrics

| Metric | Target | Measurement |
|--------|--------|-------------|
| Proof generation time | < 5 seconds | Browser performance API |
| Trust score load time | < 500ms (cached) | Time to first paint |
| Contract state transition | < 2 seconds | Relay round-trip |
| Bundle size increase | < 500KB (non-ZK) | Build output |
| ZK WASM load | < 3 seconds (first load), < 500ms (cached) | Service worker metrics |

### User Experience Metrics

| Metric | Target | Measurement |
|--------|--------|-------------|
| Time to first trust score view | < 3 seconds | User testing |
| Contract creation completion rate | > 80% | Funnel analytics |
| Proof sharing success rate | > 95% | Error tracking |
| Mobile usability score | > 90/100 | Lighthouse audit |

### Adoption Metrics

| Metric | Target (6 months) | Target (12 months) |
|--------|-------------------|-------------------|
| QoT-enabled profiles | 500 | 5,000 |
| Contracts created | 100 | 2,000 |
| Eligibility proofs generated | 200 | 5,000 |
| Verifier relay deployments | 3 | 10 |

---

## Timeline Summary

| Phase | Duration | Cumulative |
|-------|----------|------------|
| Phase 0: Aztec Infrastructure | 2-3 weeks | 2-3 weeks |
| Phase 1: Core Infrastructure | 4-6 weeks | 6-9 weeks |
| Phase 2: Contract System | 4-6 weeks | 10-15 weeks |
| Phase 3: ZK Proving | 4-6 weeks | 14-21 weeks |
| Phase 4: Customer Trust & Teams | 3-4 weeks | 17-25 weeks |
| Phase 5: Organizational Trust | 3-4 weeks | 20-29 weeks |
| Phase 6: Dispute Resolution | 2-3 weeks | 22-32 weeks |

**Total estimated duration: 22-32 weeks (5-8 months)**

---

## Conclusion

Extending Nostria with QoT Professional Network pages is architecturally feasible and well-aligned with Nostria's existing patterns. The key success factors are:

1. **Maintain separation**: QoT pages are additive, not modifications to existing features
2. **Follow existing patterns**: Signals, lazy loading, Material Design
3. **Aztec as source of truth**: Trust state, contracts, and escrow live on-chain; Nostr provides discovery
4. **Progressive enhancement**: Core features work with Aztec reads; ZK proofs add efficient verification
5. **Performance-first**: Lazy load WASM, cache aggressively, use Web Workers

The implementation can proceed in parallel with Nostria's existing development, with integration points limited to navigation and optional profile badges.

---

## Related Documents

- **QoT_Nostria_Client_Specification.md** — Technical architecture specification
- **QoT_Professional_Network_UX_Analysis.md** — Full UX specification
- **ZK_Relay_Integration_Spec.md** — Relay-side verification
- **Nostr_QoT_Gap_Analysis.md** — Technical gap resolutions
- **Nostr_Analysis.md** — Protocol compatibility analysis
- **ADR_Subcontract_Architecture.md** — Contract decomposition patterns
- **ADR_Milestone_Payment_Gates.md** — Milestone-based payment model
- **ADR_Dispute_Resolution.md** — Deadline-based dispute resolution
- **QoT_Aztec_Contract_Layer_Specification.md** — Smart contract architecture
