# Quantum of Trust

**Trust Primitives for the Agent Economy**

q⟨T⟩ decouples trust from identity. Build reputation through verified action, not disclosure. Same primitives for humans and AI agents.

## Documents

### Overview

- [**Introduction**](./introduction.md) — Start here for a quick overview

### Conceptual Framework

- [**Whitepaper**](./QuantumOfTrust_v10.md) — Complete framework specification
- [**Math Equations in Plain English**](./The_Quantum_of_Trust_Math_Equations_in_Plain_English.md) — Every equation explained with examples
- [**The Difficulty of Assessing Difficulty**](./The_Difficulty_of_Assessing_Difficulty.md) — How difficulty ratings are determined at the task level
- [**Sybil Resistance Architecture**](./Sybil_Resistance_Architecture.md) — Defense-in-depth design rationale

### Architecture Decisions

- [**Subcontract Architecture**](./ADR_Subcontract_Architecture.md) — Multi-phase contract decomposition
- [**No Endorsements**](./ADR_No_Endorsements.md) — Why contract outcomes replace attestations
- [**Conceptual/Implementation Boundary**](./ADR_Conceptual_Implementation_Boundary.md) — Separation of concerns
- [**Trust Signal Boundaries**](./ADR_Trust_Signal_Boundaries.md) — What q⟨T⟩ captures vs excludes
- [**Nostr Native Infrastructure**](./ADR_Nostr_Native_Infrastructure.md) — Native ecosystem patterns

### Platform

- [**Blockchain Selection**](./Blockchain_Selection_for_Quantum_of_Trust_Implementation.md) — Why Aztec/Noir

### Specifications

- [**Aztec Contract Layer**](./QoT_Aztec_Contract_Layer_Specification.md) — Smart contract architecture for QoTRegistry, QoTEscrow, QoTAvatar

### Nostr Integration

- [**Nostr Analysis**](./Nostr_Analysis.md) — Protocol evaluation for q⟨T⟩
- [**Nostr QoT Gap Analysis**](./Nostr_QoT_Gap_Analysis.md) — Feature gaps and solutions
- [**ZK Relay Integration Spec**](./ZK_Relay_Integration_Spec.md) — strfry extension architecture

### Bitcoin Integration

- [**Bitcoin Bridge Roadmap**](./bitcoin_bridge_roadmap.md) — Cross-chain integration path

### Application Design

- [**Professional Network UX Analysis**](./QoT_Professional_Network_UX_Analysis.md) — LinkedIn-style application design

### Implementation

- [**Noir Implementation**](./Quantum_of_Trust_Equations_in_Noir.md) — Core ZK circuits
- [**Sybil Resistance Circuits (Noir)**](./Sybil_Resistance_Circuits_Noir.md) — Enhanced Sybil defense circuits
- [**C# Implementation**](./Quantum_of_Trust_Equations_in_CSharp.md) — Traditional systems implementation
- [**Sybil Resistance (C#)**](./Sybil_Resistance_Implementation_CSharp.md) — Enhanced Sybil defense in C#

### Tests

- [**C# Unit Tests**](./QuantumOfTrustTests.md)
- **Noir Test Suites:**
  - [Contract Validation](./contract_validation_tests_noir.md)
  - [Skill Type Independence](./skill_type_independence_tests_noir.md)
  - [Eligibility Edge Cases](./eligibility_edge_cases_tests_noir.md)
  - [History & Trust Evolution](./history_trust_evolution_tests_noir.md)
  - [Sybil Resistance](./sybil_resistance_tests_noir.md)
