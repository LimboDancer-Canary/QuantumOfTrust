# Quantum of Trust

**Trust Primitives for the Agent Economy**

q\<T\> decouples trust from identity. Build reputation through verified action, not disclosure. Same primitives for humans and AI agents.

## Documents

### Overview

- [**Introduction**](./introduction.md) — Start here for a quick overview

### Conceptual Framework

- [**Whitepaper**](./QuantumOfTrust_v10.md) — Complete framework specification
- [**Math Equations in Plain English**](./The_Quantum_of_Trust_Math_Equations_in_Plain_English.md) — Every equation explained with examples
- [**Sybil Resistance Architecture**](./Sybil_Resistance_Architecture.md) — Defense-in-depth design rationale

### Platform

- [**Blockchain Selection**](./Blockchain_Selection_for_Quantum_of_Trust_Implementation.md) — Why Aztec/Noir

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
