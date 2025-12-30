# ADR: Subcontract Architecture for Multi-Phase Contracts

**Status:** Accepted  
**Date:** December 30, 2025  
**Deciders:** Dennis  
**Context:** QoT Professional Network design

---

## Decision

Treat contract phases (Specification, Planning, Implementation) as **subcontracts**--full contracts in their own right, linked by a parent project--rather than embedding phases within a single contract structure.

---

## Context

The QoT Professional Network requires multi-phase, multi-provider contracts where:
- Different providers may handle different phases (specialist collaboration)
- Each phase has distinct skill types (Requirements, Architecture, Development)
- Planning accuracy must be tracked across phase boundaries
- Team-based implementation requires task-level decomposition

Two architectural approaches were considered.

---

## Options Considered

### Option A: Embedded Phases (Rejected)

```
Contract = {
    phases: [Phase; 3],      // Fixed array of phases
    current_phase: Field,
    ...
}
```

**Problems:**
- Circuit size 3x per contract (all phases in one proof)
- Fixed phase count limits flexibility
- All phases proven together, no incremental verification
- Complex state machine for phase transitions

### Option B: Subcontracts (Accepted)

```
Project = { project_id, consumer, total_stake, ... }
Contract = { ..., project_id?, contract_type?, ... }
```

**Benefits:**
- Trust calculation unchanged--subcontracts ARE contracts
- Circuit size 1x per contract (proven independently)
- Incremental proving as work completes
- Any number of phases per project
- Recursive pattern matches existing q<T> / q<DAO> nesting

---

## Decision Rationale

### The Recursive Insight

```
Just as DAO contains Agents (which are q<T>),
Project contains Phases (which are Contracts),
Phases contain Tasks (which are sub-Contracts).

Trust flows from contracts at every level.
Projects coordinate contracts.
The math doesn't care about the coordination layer.
```

### Circuit Size Comparison

| Approach | Circuit Size | Proving Time | Flexibility |
|----------|--------------|--------------|-------------|
| Embedded phases | 3x per contract | All at once | Fixed 3 phases |
| **Subcontracts** | 1x per contract | Incremental | Any number |

### Trust Equation Unchanged

The core equation applies without modification to subcontracts:

```
V_t(Agent(t, h_t)) = Sum over c in h_t of: omega(c) . outcome(c) . gamma(c) . nu(c)
```

Each subcontract is simply another contract in the agent's history.

---

## Consequences

### Positive

1. **No equation changes** -- Subcontracts use existing trust math
2. **Backward compatible** -- Legacy standalone contracts work unchanged
3. **Extensible** -- Tasks extend the pattern to team-based implementation
4. **Clean separation** -- Trust math vs coordination concerns decoupled
5. **Incremental proofs** -- Each phase proven independently as completed

### Negative

1. **Cross-contract references** -- Planning accuracy requires linking Implementation to Planning contract
2. **Coordination overhead** -- Project metadata layer needed for sequencing

### Neutral

1. **Publication consent** -- Moves to project level (all phases share consent)
2. **Escrow distribution** -- Coordinated by project, allocated to subcontracts

---

## Implementation

### Contract Extension

```
contract_type: "standalone" | "specification" | "planning" | "implementation" | "task"
project_id: optional reference to parent project
parent_ref: optional reference to parent contract (for tasks)
```

### Task Decomposition

Tasks extend the pattern one level deeper:
```
Project -> Phases (Subcontracts) -> Tasks (Sub-Subcontracts)
```

Each task is a contract with its own provider, enabling team-based implementation where different providers handle different tasks within a single phase.

### Customer Trust

Customers are trust-bearing entities. Bidirectional trust emerges naturally:
- Providers earn trust from contract outcomes
- Customers earn trust from behaviors (commitment, escrow discipline, verification integrity)

---

## Related Documents

- **Quantum_of_Trust_Equations_in_CSharp.md** -- Full implementation with Task, CustomerProfile, CustomerTrustCalculator
- **Quantum_of_Trust_Equations_in_Noir.md** -- ZK circuit implementation
- **QuantumOfTrustTests.md** -- Test specifications including hierarchical contracts
- **QoT_Professional_Network_UX_Analysis.md** -- UX flows and design decisions

---

## Migration

No migration required. The subcontract model is backward compatible:

| Contract Type | Handling |
|---------------|----------|
| Legacy standalone | project_id = null, contract_type = "standalone" |
| New standalone | Same as legacy |
| New subcontract | project_id = id, contract_type = phase type |

Existing contracts and proofs continue to work without modification.
