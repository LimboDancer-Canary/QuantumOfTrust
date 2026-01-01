# ADR: Milestone Payment Gates

**Status:** Accepted  
**Date:** January 1, 2026  
**Deciders:** Dennis  
**Context:** QoT Professional Network — Implementation phase payment structure

---

## Decision

Introduce **Milestones** as payment gates within the Implementation phase. Milestones batch tasks into reviewable units with a single customer review point. Payment releases at milestone completion, not per-task.

---

## Context

The Implementation phase can span months with many tasks across multiple providers. The original all-or-nothing model created significant risk asymmetry:

| Party | Risk |
|-------|------|
| Provider | Complete 90% of work, dispute on task 10 → no payment for months of labor |
| Customer | Pay at end → limited visibility into project health until too late |

Milestones address this by creating intermediate payment gates that also serve as health checks.

---

## Design

### Hierarchy

```
Project
└── Implementation Phase (contract)
    └── Milestone (contract)
        ├── completion_deadline: computed from max(child task deadlines)
        ├── stake: computed from sum(child task stakes)
        │
        └── Tasks (contracts)
            ├── Task A: deadline, stake, provider
            ├── Task B: deadline, stake, provider
            └── Task C: deadline, stake, provider
```

### Milestone as Contract

Milestones follow the existing subcontract pattern:

```noir
contract_type: Field,      // = CONTRACT_TYPE_MILESTONE
parent_ref: Field,         // → Implementation phase
project_id: Field,         // → Project
```

**Computed fields (not stored, derived from children):**

| Field | Derivation |
|-------|------------|
| `completion_deadline` | `max(task.deadline for task in milestone.tasks)` |
| `stake` | `sum(task.stake for task in milestone.tasks)` |

### Task Deadlines

Task deadlines represent **work duration allocations**, not customer review points:

```
Milestone 1
├── Task A: deadline Day 8   (provider has 8 days)
├── Task B: deadline Day 15  (provider has 15 days)
└── Task C: deadline Day 25  (provider has 25 days)

Milestone deadline = Day 25 (derived)
```

Task deadlines serve as:
- Planning constraints for providers
- Evidence in arbitration ("Task B was allocated 15 days")
- Derivation basis for milestone deadline

Task deadlines do NOT serve as:
- Individual payment triggers
- Individual customer review points

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

### Milestone Review Flow

```
┌─────────────────────────────────────────────────────────────────────┐
│  BEFORE MILESTONE DEADLINE                                           │
│                                                                      │
│  - Providers work on tasks                                           │
│  - Providers deliver off-chain (files, repos, DMs)                   │
│  - Parties communicate about progress                                │
│  - No on-chain state changes                                         │
└─────────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────────┐
│  AT MILESTONE DEADLINE: Customer must act                            │
│                                                                      │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐      │
│  │ ACCEPT          │  │ DISPUTE         │  │ DO NOTHING      │      │
│  │ MILESTONE       │  │ TASK(S)         │  │                 │      │
│  └────────┬────────┘  └────────┬────────┘  └────────┬────────┘      │
│           │                    │                    │               │
│           ▼                    ▼                    ▼               │
│    All task stakes       Disputed tasks       Timeout:              │
│    released to           enter arbitration    All task stakes       │
│    respective            Non-disputed tasks   released              │
│    providers             paid normally                              │
└─────────────────────────────────────────────────────────────────────┘
```

### Payment Distribution

Milestone stake is the sum of child task stakes. On milestone acceptance:

```noir
fn accept_milestone(milestone_id: Field) {
    let milestone = get_milestone(milestone_id);
    assert(msg_sender() == milestone.customer);
    assert(current_block() <= milestone.completion_deadline);
    
    // Pay each task's stake to its provider
    for task in milestone.tasks {
        transfer(task.provider, task.stake);
        
        // Record outcome for each task
        task.outcome = PRECISION;  // +1.0
        update_trust(task.provider, task.skill_type, PRECISION);
    }
    
    // Customer earns verification trust
    update_trust(milestone.customer, SKILL_CUSTOMER_VERIFICATION, PRECISION);
    
    milestone.status = MilestoneStatus::Accepted;
    emit MilestoneAccepted { milestone_id };
}
```

### Partial Dispute

Customer can dispute specific tasks while accepting others:

```noir
fn dispute_tasks(milestone_id: Field, task_ids: [Field], evidence_hash: Field) {
    let milestone = get_milestone(milestone_id);
    assert(msg_sender() == milestone.customer);
    assert(current_block() <= milestone.completion_deadline);
    
    for task in milestone.tasks {
        if task_ids.contains(task.task_id) {
            // This task enters arbitration
            task.status = TaskStatus::Disputed;
            task.dispute_state = Some(DisputeState {
                status: DisputeStatus::Initiated,
                customer_evidence_hash: evidence_hash,
                ..Default::default()
            });
        } else {
            // Non-disputed tasks paid immediately
            transfer(task.provider, task.stake);
            task.outcome = PRECISION;
            task.status = TaskStatus::Accepted;
            update_trust(task.provider, task.skill_type, PRECISION);
        }
    }
    
    milestone.status = MilestoneStatus::PartialDispute;
    emit TasksDisputed { milestone_id, task_ids };
}
```

### Timeout

If customer takes no action by milestone deadline:

```noir
fn timeout_milestone(milestone_id: Field) {
    let milestone = get_milestone(milestone_id);
    assert(current_block() > milestone.completion_deadline);
    assert(milestone.status == MilestoneStatus::Active);
    
    // All tasks paid — customer forfeited review
    for task in milestone.tasks {
        transfer(task.provider, task.stake);
        task.outcome = PRECISION;
        task.status = TaskStatus::Accepted;
        update_trust(task.provider, task.skill_type, PRECISION);
    }
    
    update_trust(milestone.customer, SKILL_CUSTOMER_VERIFICATION, PRECISION);
    
    milestone.status = MilestoneStatus::TimedOut;
    emit MilestoneTimedOut { milestone_id };
}
```

---

## Trust Accounting

### Providers

Each task is a contract in the provider's history:

| Task | Provider | Outcome | Trust Contribution |
|------|----------|---------|-------------------|
| Task A | Alice | +1.0 | Added to Alice's skill history |
| Task B | Bob | Arbitrated: 0.5 | Added to Bob's skill history |
| Task C | Alice | +1.0 | Added to Alice's skill history |

Different providers earn trust independently based on their task outcomes.

### Customer

Customer earns `SKILL_CUSTOMER_VERIFICATION` trust from milestone outcomes:
- Accept milestone → +1.0 per task (fair reviewer)
- Dispute → outcome from arbitration
- Timeout → +1.0 (treated as acceptance)

### Milestone Contract Itself

The milestone is a coordination container. It does not independently contribute to trust — trust flows through the child tasks.

---

## Contract Type Constants

```noir
global CONTRACT_TYPE_STANDALONE: Field = 0;
global CONTRACT_TYPE_SPECIFICATION: Field = 1;
global CONTRACT_TYPE_PLANNING: Field = 2;
global CONTRACT_TYPE_IMPLEMENTATION: Field = 3;
global CONTRACT_TYPE_TASK: Field = 4;
global CONTRACT_TYPE_MILESTONE: Field = 5;  // NEW
```

---

## Data Structures

### Milestone

```noir
struct Milestone {
    milestone_id: Field,
    project_id: Field,
    parent_phase_id: Field,           // → Implementation phase
    customer: AztecAddress,
    
    // Computed from children — not stored
    // completion_deadline: derived
    // stake: derived
    
    // Status
    status: MilestoneStatus,
    
    // Arbitration parameters (inherited from project)
    arbitrator_min_trust: Field,
}

enum MilestoneStatus {
    Active,           // Work in progress
    Accepted,         // Customer accepted all tasks
    PartialDispute,   // Some tasks disputed, others accepted
    Resolved,         // All disputes resolved
    TimedOut,         // Deadline passed, payment released
}
```

### Task (unchanged structure, clarified semantics)

```noir
struct Task {
    task_id: Field,
    project_id: Field,
    parent_milestone_id: Field,       // → Milestone (changed from parent_phase_id)
    
    customer: AztecAddress,
    provider: AztecAddress,
    skill_type: Field,
    
    stake: Field,
    difficulty: Field,
    deadline: Field,                  // Work duration deadline
    
    status: TaskStatus,
    outcome: Option<Field>,
    
    // Dispute state (if disputed)
    dispute_state: Option<DisputeState>,
}
```

---

## Validation Rules

### Milestone Creation

```noir
fn create_milestone(milestone_id: Field, phase_id: Field, task_ids: [Field]) {
    let phase = get_phase(phase_id);
    assert(phase.contract_type == CONTRACT_TYPE_IMPLEMENTATION);
    assert(msg_sender() == phase.customer);
    
    // All tasks must belong to this phase's project
    for task_id in task_ids {
        let task = get_task(task_id);
        assert(task.project_id == phase.project_id);
    }
    
    // Milestone deadline derived from tasks
    // Milestone stake derived from tasks
    
    emit MilestoneCreated { milestone_id, phase_id, task_ids };
}
```

### Deadline Constraints

```noir
// Task deadlines must be <= parent milestone deadline (which is derived from them)
// Milestone deadlines must be <= phase deadline
// Phase deadlines must be <= project deadline

assert(task.deadline <= milestone.completion_deadline);  // Tautological: milestone derived from max
assert(milestone.completion_deadline <= phase.completion_deadline);
assert(phase.completion_deadline <= project.completion_deadline);
```

---

## Health Check Pattern

Milestones serve as natural health check points:

```
Milestone 1 Complete
    │
    ├── All tasks accepted?
    │   └── Yes → Payment released, proceed to Milestone 2
    │
    ├── Some tasks disputed?
    │   └── Resolve arbitration, then assess project health
    │
    └── Fundamental problems discovered?
        └── Parties negotiate: continue, restructure, or terminate
```

**Project Termination:** If milestone review reveals irreconcilable issues, parties can:
1. Complete current milestone (pay for work done)
2. Cancel remaining milestones (remaining stake returned to customer)
3. Use existing mutual cancellation mechanism for the phase

---

## Example

### Project Structure

```
Project: E-commerce Platform
├── Specification Phase (complete)
├── Planning Phase (complete)
│   └── Plan Output:
│       ├── 12 tasks defined
│       ├── 3 milestones
│       └── Total stake: 10,000 tokens
│
└── Implementation Phase
    │
    ├── Milestone 1: Core Infrastructure (3,000 tokens)
    │   ├── Task 1: Database schema     (deadline: Day 5,  stake: 800,  provider: Alice)
    │   ├── Task 2: API scaffolding     (deadline: Day 8,  stake: 1,200, provider: Alice)
    │   └── Task 3: Auth system         (deadline: Day 12, stake: 1,000, provider: Bob)
    │   └── Milestone deadline: Day 12 (derived)
    │
    ├── Milestone 2: Features (4,500 tokens)
    │   ├── Task 4: Product catalog     (deadline: Day 20, stake: 1,500, provider: Carol)
    │   ├── Task 5: Shopping cart       (deadline: Day 25, stake: 1,500, provider: Carol)
    │   └── Task 6: Checkout flow       (deadline: Day 30, stake: 1,500, provider: Bob)
    │   └── Milestone deadline: Day 30 (derived)
    │
    └── Milestone 3: Polish (2,500 tokens)
        ├── Task 7: UI refinement       (deadline: Day 38, stake: 1,000, provider: Diana)
        ├── Task 8: Performance tuning  (deadline: Day 42, stake: 1,000, provider: Alice)
        └── Task 9: Integration tests   (deadline: Day 45, stake: 500,   provider: Bob)
        └── Milestone deadline: Day 45 (derived)
```

### Review Flow

**Day 12 — Milestone 1 deadline:**
- Customer reviews Tasks 1, 2, 3
- Tasks 1, 2: Satisfactory → Customer will accept
- Task 3: Auth system has bugs → Customer will dispute

**Customer calls:** `dispute_tasks(milestone_1, [task_3], evidence_hash)`

**Result:**
- Task 1: 800 tokens → Alice, outcome +1.0
- Task 2: 1,200 tokens → Alice, outcome +1.0
- Task 3: Enters arbitration (Bob vs Customer)

**Day 20 — Arbitration resolves:**
- Arbitrator rules: 60% to Bob (bugs were minor, mostly complete)
- Task 3: 540 tokens → Bob (900 × 60%), 360 → Customer
- Task 3 outcome: +0.2 (derived from 60% payout)

**Day 30 — Milestone 2 deadline:**
- Customer accepts all tasks
- 4,500 tokens distributed to Carol and Bob
- Project proceeds to Milestone 3

---

## Consequences

### Positive

1. **Incremental payment** — Providers see money as work completes
2. **Health checks** — Customer reviews at defined intervals
3. **Risk distribution** — Neither party carries all-or-nothing risk
4. **Existing patterns** — Milestones are contracts, tasks are contracts
5. **Flexible grouping** — Milestone structure negotiated in Planning
6. **Partial disputes** — Dispute specific tasks, pay the rest

### Negative

1. **Additional contract level** — Milestone between Phase and Task
2. **Derived fields** — Deadline and stake computed, not stored
3. **Batch review** — Customer reviews multiple tasks at once (could be overwhelming)

### Neutral

1. **Off-chain coordination unchanged** — Parties still communicate progress informally
2. **Trust math unchanged** — Tasks contribute to provider trust as before
3. **Dispute mechanism unchanged** — Uses ADR_Dispute_Resolution for task disputes

---

## Related Documents

- **ADR_Subcontract_Architecture.md** — Recursive contract pattern
- **ADR_Dispute_Resolution.md** — Task-level dispute handling
- **The_Difficulty_of_Assessing_Difficulty.md** — Task decomposition rationale
- **QoT_Aztec_Contract_Layer_Specification.md** — Contract structures

---

## Summary

Milestones are payment gates that batch tasks into reviewable units. They are contracts in the subcontract hierarchy, with deadline and stake derived from child tasks. Customer reviews at milestone deadline: accept, dispute specific tasks, or timeout. Payment releases per-task to respective providers. Trust flows through tasks, not the milestone itself.

The mechanism adds incremental payment without new primitives — milestones are just another contract level following the existing recursive pattern.
