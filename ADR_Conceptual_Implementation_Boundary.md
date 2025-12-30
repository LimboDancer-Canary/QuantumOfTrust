# ADR: Conceptual/Implementation Boundary

**Status:** Accepted  
**Date:** December 30, 2025  
**Deciders:** Dennis  
**Context:** QoT documentation architecture

---

## Decision

The whitepaper (QuantumOfTrust_v10.md) describes the **conceptual grammar** — stable primitives and properties. Implementation documents describe how to **write sentences** in that grammar — specific mechanisms that may evolve.

---

## Context

When adding Sybil resistance mechanisms (escrow commitments, outcome variance requirements, counterparty trust weighting, temporal velocity limits), the question arose: should the whitepaper be updated?

The answer was no. These are implementation details that achieve properties the whitepaper already states abstractly.

---

## The Boundary

| Conceptual Layer (Whitepaper) | Implementation Layer (Other Docs) |
|-------------------------------|-----------------------------------|
| "Trust should be hard to fake" | Escrow commitments, variance checks |
| "Contracts involve stake" | How stake is verified cryptographically |
| "Who you work with matters" | Sigmoid functions, scale parameters |
| "Sybil attacks should be economically irrational" | Specific mechanisms that ensure this |

---

## Examples

### Whitepaper States Properties

**Sybil resistance statement** (Part Seven):
> "Sybil Resistance: Multiple fake Avatars provide no meaningful advantage over genuine trust-building through a single Avatar."

This is a **property** the system must satisfy, not a **mechanism**.

### Implementation Docs Specify Mechanisms

The four Sybil resistance mechanisms are **ways to achieve** the property:
1. Economic escrow commitments
2. Counterparty trust weighting  
3. Outcome variance requirements
4. Temporal velocity limits

If better mechanisms are discovered, they can be swapped. The whitepaper remains valid because it stated the goal, not the method.

### The Weighting Function

**Whitepaper** (Equation 7):
```
ω(c) = f(s, d, V_t(counterparty), recency)
```

The whitepaper says counterparty trust matters. It doesn't specify the exact functional form (sigmoid, linear, step function). That's deliberate generality.

**Implementation** specifies:
```csharp
double CounterpartyFactor(double counterpartyTrust)
{
    return 1.0 / (1.0 + Math.Exp(-SIGMOID_SCALE * (counterpartyTrust - SIGMOID_MIDPOINT)));
}
```

### The Stake Parameter

**Whitepaper**: `s` is "how much is at stake"

**Implementation**: Cryptographic escrow commitment proving value is actually locked

The *concept* is "value at risk." The *implementation* is "how we verify that value is actually at risk."

---

## Decision Rationale

### The Grammar/Sentences Analogy

The whitepaper describes the **grammar**:
```
q⟨T⟩ ::= Agent(t, h_t) | DAO({q⟨T⟩})
```

Implementation docs write **sentences** in that grammar:
- A customer is an Avatar with customer skill types
- Tasks are contracts with parent references
- Sybil resistance uses four specific mechanisms

The grammar doesn't change when you write new sentences.

### Stability Benefits

1. **Whitepaper remains authoritative** — No version churn
2. **Implementation can evolve** — Better mechanisms can replace old ones
3. **Clear contributor guidance** — "Does this change the grammar or write a new sentence?"
4. **Cleaner external communication** — Whitepaper is the stable reference

---

## Document Hierarchy

| Document | Stability | Contents |
|----------|-----------|----------|
| QuantumOfTrust_v10.md | Stable | Conceptual primitives, properties, grammar |
| The_Quantum_of_Trust_Math_Equations_in_Plain_English.md | Evolving | Current mathematical formalization |
| Quantum_of_Trust_Equations_in_Noir.md | Evolving | ZK circuit implementation |
| Quantum_of_Trust_Equations_in_CSharp.md | Evolving | Reference implementation |
| Sybil_Resistance_Architecture.md | Evolving | Specific defense mechanisms |

The Plain English Math document sits at the boundary — it formalizes the current state of the mathematics, which should reflect implementation reality.

---

## Consequences

### Positive

1. **Whitepaper longevity** — Remains valid as mechanisms evolve
2. **Clear separation of concerns** — Conceptual vs. implementation changes
3. **Reduced review overhead** — Implementation changes don't require whitepaper review
4. **Contributor clarity** — Clear guidance on where changes belong

### Negative

1. **Potential drift** — Implementation could diverge from conceptual intent
2. **Discovery burden** — Readers must consult multiple documents for full picture

### Mitigation

ADRs like this one document significant implementation decisions, creating a bridge between conceptual intent and implementation reality.

---

## Related Documents

- **QuantumOfTrust_v10.md** — The stable conceptual foundation
- **Sybil_Resistance_Architecture.md** — Implementation mechanisms
- **ADR_Subcontract_Architecture.md** — Example of implementation using existing grammar

---

## Summary

The whitepaper describes *what* and *why*. Implementation documents describe *how*. The recursive definitions in the whitepaper already encompass customers as avatars, tasks as contracts, and Sybil resistance as a property — we're writing sentences in an existing grammar, not changing the grammar itself.
