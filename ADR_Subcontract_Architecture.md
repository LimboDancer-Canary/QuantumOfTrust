# ADR: Subcontract Architecture for Multi-Phase Contracts

**Status:** Accepted  
**Date:** January 1, 2026 (Updated)  
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
Implementation Phase contains Milestones (payment gates),
Milestones contain Tasks (which are sub-Contracts).

Trust flows from contracts at every level.
Projects coordinate contracts.
Milestones coordinate payment.
The math doesn't care about the coordination layer.
```

### Four-Level Hierarchy

```
Project
├── Specification Phase (contract)
├── Planning Phase (contract)
│   └── Outputs: Task definitions, Milestone groupings
│
└── Implementation Phase (contract)
    ├── Milestone 1 (contract, payment gate)
    │   ├── Task A (contract)
    │   ├── Task B (contract)
    │   └── Task C (contract)
    │
    ├── Milestone 2 (contract, payment gate)
    │   └── Tasks...
    │
    └── Milestone N...
```

Milestones batch tasks into reviewable units with a single customer review point. Payment releases at milestone completion, not per-task. See **ADR_Milestone_Payment_Gates.md** for details.

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
3. **Extensible** -- Milestones and Tasks extend the pattern to incremental payment
4. **Clean separation** -- Trust math vs coordination concerns decoupled
5. **Incremental proofs** -- Each phase proven independently as completed
6. **Incremental payment** -- Milestones release payment as work progresses

### Negative

1. **Cross-contract references** -- Planning accuracy requires linking Implementation to Planning contract
2. **Coordination overhead** -- Project metadata layer needed for sequencing
3. **Additional contract level** -- Milestone layer adds hierarchy depth

### Neutral

1. **Publication consent** -- Moves to project level (all phases share consent)
2. **Escrow distribution** -- Coordinated by project, allocated through milestones to tasks

---

## Implementation

### Contract Extension

```
contract_type: "standalone" | "specification" | "planning" | "implementation" | "milestone" | "task"
project_id: optional reference to parent project
parent_ref: optional reference to parent contract (for milestones and tasks)
```

### Contract Type Constants

```noir
global CONTRACT_TYPE_STANDALONE: Field = 0;
global CONTRACT_TYPE_SPECIFICATION: Field = 1;
global CONTRACT_TYPE_PLANNING: Field = 2;
global CONTRACT_TYPE_IMPLEMENTATION: Field = 3;
global CONTRACT_TYPE_TASK: Field = 4;
global CONTRACT_TYPE_MILESTONE: Field = 5;
```

### Task Decomposition

The pattern extends to four levels:
```
Project -> Phases (Subcontracts) -> Milestones (Payment Gates) -> Tasks (Sub-Subcontracts)
```

**Milestones** are payment gates within the Implementation phase. They:
- Batch related tasks into reviewable units
- Have a single customer review point (milestone deadline)
- Release payment incrementally as work progresses
- Serve as health check points for project continuation

**Tasks** are atomic work units within a milestone. Each task:
- Has its own provider (enabling team-based implementation)
- Has a work duration deadline (planning data, not payment trigger)
- Contributes to provider's skill-specific trust on acceptance
- Can be disputed independently at milestone review

**Trust flows through tasks, not milestones.** The milestone is a coordination container. Each task is a full contract in the provider's history.

### Difficulty Assessment

Difficulty flows through the subcontract hierarchy:

1. **Tasks**: Provider assesses difficulty (0-10) at contract acceptance
2. **Milestones**: Difficulty aggregates from tasks via stake-weighted average
3. **Phases**: Difficulty aggregates from milestones via stake-weighted average
4. **Projects**: Difficulty aggregates from phases via stake-weighted average

```
d_milestone = Σ(d_task × s_task) / Σ(s_task)
d_phase = Σ(d_milestone × s_milestone) / Σ(s_milestone)
```

Tasks come from the Planning phase (customer's responsibility). The Implementation provider reviews tasks at acceptance and can request refinement before committing. Both parties have incentive for accurate ratings—incorrect difficulty leads to failed tasks, affecting trust for both provider and customer.

### Milestone Definition

Milestones are defined in the **Planning phase** as part of the Plan:

```
Plan Output
├── Task definitions (what, skill type, stake allocation)
├── Task deadlines (work duration)
├── Milestone groupings (which tasks batch together)
└── Milestone sequence (order of delivery)
```

At **contract acceptance**, the Implementation provider can request refinements:
- Task breakdown changes
- Deadline adjustments
- Milestone regrouping
- Stake reallocation within milestones

Both parties must agree on the complete Plan before work begins.

### Customer Trust

Customers are trust-bearing entities. Bidirectional trust emerges naturally:
- Providers earn trust from contract outcomes
- Customers earn trust from behaviors (commitment, escrow discipline, verification integrity)

---

## Related Documents

- **ADR_Milestone_Payment_Gates.md** -- Milestone lifecycle and payment flow
- **ADR_Dispute_Resolution.md** -- Arbitration contracts and task-level disputes
- **The_Difficulty_of_Assessing_Difficulty.md** -- How difficulty ratings are determined at the task level
- **Quantum_of_Trust_Equations_in_CSharp.md** -- Full implementation with Task, CustomerProfile, CustomerTrustCalculator
- **Quantum_of_Trust_Equations_in_Noir.md** -- ZK circuit implementation
- **QuantumOfTrustTests.md** -- Test specifications including hierarchical contracts
- **QoT_Professional_Network_UX_Analysis.md** -- UX flows and design decisions
- **QoT_Aztec_Contract_Layer_Specification.md** -- Smart contract implementation

---

## Migration

No migration required. The subcontract model is backward compatible:

| Contract Type | Handling |
|---------------|----------|
| Legacy standalone | project_id = null, contract_type = "standalone" |
| New standalone | Same as legacy |
| New phase | project_id = id, contract_type = phase type |
| New milestone | project_id = id, parent_ref = phase_id, contract_type = "milestone" |
| New task | project_id = id, parent_ref = milestone_id, contract_type = "task" |

Existing contracts and proofs continue to work without modification.
