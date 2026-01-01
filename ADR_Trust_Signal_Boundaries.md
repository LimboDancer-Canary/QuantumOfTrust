# ADR: Trust Signal Boundaries

**Status:** Accepted  
**Date:** December 30, 2025  
**Deciders:** Dennis  
**Context:** QoT Professional Network design

---

## Decision

This ADR consolidates decisions about what affects trust calculations versus what doesn't. Three related decisions share a common principle: **only stake-backed, adversarially-validated signals flow into trust**.

| Signal | Affects Trust? | Rationale |
|--------|----------------|-----------|
| Contract outcomes | Yes | Stake-backed, mutually signed |
| Task outcomes | Yes | Stake-backed, within milestones |
| Milestone completion | No | Coordination container, not work |
| Milestone review/acceptance | No | Customer action, not provider work |
| Verification/acceptance | No | Customer action, not provider work |
| Publication consent | No | Visibility control, not quality signal |
| QA work | Yes, via separate contract | QA earns trust from their own contracts |
| Dispute resolution outcome | Yes | Affects task outcome, which flows to trust |

---

## Decision 1: Verification Is Acceptance, Not Work

### Context

Within a contract, verification determines whether the implementation meets requirements. The question: should verification be a trust-earning phase for a provider?

### Decision

Verification is the customer's acceptance mechanism. It determines the implementation outcome but doesn't itself earn trust.

```
implementation_phase: {
  provider: a_developer,
  outcome: determined_by_verification,
  verified_by: a_customer | a_qa_delegate
}
```

The verifier isn't being rated — they're doing the rating.

### Rationale

| Verification as... | Problem |
|--------------------|---------|
| Trust-earning phase | Who rates the verifier? Infinite regress. |
| Customer action | Clean termination. Customer exercises quality control. |

If the customer delegates verification to a QA specialist, that delegation is either:
- **Internal**: QA acts as customer's agent (no separate trust flow)
- **External**: QA is hired via separate contract (see Decision 3)

---

## Decision 2: Milestones Are Coordination Containers, Not Trust Sources

### Context

Within Implementation phases, tasks are grouped into milestones as payment gates. Customer reviews at milestone deadline: accept all tasks, dispute specific tasks, or timeout. Should milestones independently contribute to trust?

### Decision

Milestones are coordination containers for payment. **Trust flows through tasks, not milestones.**

```
implementation_phase: {
  provider: a_developer,
  milestones: [
    {
      milestone_id: 1,
      tasks: [task_1, task_2, task_3],
      deadline: computed_from_tasks,  // max(task deadlines)
      stake: computed_from_tasks,     // sum(task stakes)
      // No independent trust contribution
    }
  ]
}
```

### Rationale

| Milestones as... | Problem |
|------------------|---------|
| Trust-earning entities | Double-counting: tasks already earn trust |
| Coordination containers | Clean model: payment gates without trust complexity |

Milestones serve operational purposes:
- **Payment gates**: Incremental payment reduces provider risk
- **Customer review points**: Batch assessment rather than per-task
- **Deadline coordination**: Milestone deadline = max(task deadlines)

Trust attribution happens at the task level where the actual work is performed and outcomes are determined.

### Milestone Review as Acceptance

Customer review at milestone deadline is an acceptance mechanism, like verification:

| Customer Action | Result |
|-----------------|--------|
| Accept all tasks | Payment released, task outcomes recorded |
| Dispute specific tasks | Non-disputed tasks paid; disputed tasks enter arbitration |
| Timeout | All tasks automatically accepted |

The customer isn't being rated during milestone review—they're exercising acceptance rights. See **ADR_Milestone_Payment_Gates.md** and **ADR_Dispute_Resolution.md**.

---

## Decision 3: Publication Consent Not Trust-Weighted

### Context

Contract details are private by default. Both parties must consent before publication. Should willingness to publish affect trust calculations?

### Decision

Publication consent is recorded but does not affect trust weight.

```
contract_disclosure: {
  contract_hash: "...",
  disclosed_by: [a_provider, a_consumer],
  disclosure_scope: "full" | "outcome_only" | "existence_only",
  affects_trust: false  // Explicitly not a trust input
}
```

### Rationale

If publication consent affected trust:
- Providers could pressure customers to publish for trust boost
- Customers could withhold consent as leverage
- Coercion dynamics would corrupt the signal

Publication is about **visibility**, not **quality**. A private contract with excellent outcomes should earn the same trust as a published one.

### Disclosure Scopes

| Scope | What's Revealed |
|-------|-----------------|
| `existence_only` | "We worked together" |
| `outcome_only` | Skill type + outcome (no counterparty identity) |
| `full` | Complete contract details |

---

## Decision 4: QA Earns Trust via Separate Contracts

### Context

Quality Assurance is a valid professional skill. QA professionals should be able to build trust. But QA work reviews other providers' deliverables — how does this fit?

### Decision

QA professionals earn trust through their own contracts with customers, not as a phase within someone else's build contract.

```
Contract A: Build the thing
  consumer: a_customer
  provider: a_developer
  phases: [spec, planning, implementation]
  verification: performed by a_customer
  
Contract B: Test the thing (separate contract)
  consumer: a_customer
  provider: a_qa_professional
  phases: [
    { spec: "test plan scope" },
    { planning: "test strategy" },
    { implementation: "execute tests, report findings" }
  ]
  verification: performed by a_customer
```

### Rationale

| QA as... | Problem |
|----------|---------|
| Phase in build contract | Conflates verification with QA work; who rates the QA phase? |
| Separate contract | Clean model. QA earns "Quality Assurance" skill trust from their work. |

The QA professional earns trust based on how well they performed their testing work — as judged by the customer. Contract B's outcome may inform the customer's verification decision on Contract A, but that's operational coordination, not part of the trust framework.

### Same Structure, Different Skill

QA contracts follow the same three-phase structure:
- **Specification**: Test plan scope
- **Planning**: Test strategy
- **Implementation**: Execute tests, report findings

The customer judges: "Did the QA professional do good work?"

---

## Common Principle

All four decisions follow the same principle:

> **Only stake-backed, adversarially-validated signals flow into trust.**

| Signal | Stake-backed? | Adversarially validated? | Flows into trust? |
|--------|---------------|--------------------------|-------------------|
| Contract outcome | Yes (escrow) | Yes (mutual sign-off) | Yes |
| Task outcome | Yes (escrow) | Yes (mutual sign-off) | Yes |
| Milestone completion | No (coordination) | No (container) | No |
| Milestone review | No | No (unilateral) | No |
| Verification action | No | No (unilateral) | No |
| Publication consent | No | No (visibility decision) | No |
| QA contract outcome | Yes (escrow) | Yes (mutual sign-off) | Yes |
| Dispute resolution | Yes (affects task) | Yes (arbitrator decision) | Yes (via task) |

---

## Consequences

### Positive

1. **Clean termination** — No infinite regress of "who rates the rater"
2. **No coercion vectors** — Publication can't be weaponized
3. **QA as first-class profession** — Clear path to building QA reputation
4. **Consistent model** — Same contract structure applies to all work types
5. **Simple trust math** — Trust flows through tasks; milestones don't add complexity
6. **Clear payment gates** — Milestones serve operational purpose without trust overhead

### Negative

1. **Two contracts for tested work** — Operational overhead for customers who want both build and QA
2. **No publication incentive** — System doesn't encourage transparency
3. **Milestone complexity** — Additional coordination layer for multi-task implementations

### Neutral

1. **QA-build coordination** — Contract B's findings may influence Contract A's outcome, but this is operational, not framework concern
2. **Milestone granularity** — How to group tasks into milestones is a planning decision, not a trust decision

---

## Related Documents

- **ADR_Milestone_Payment_Gates.md** — Milestone-based payment model and customer review workflow
- **ADR_Dispute_Resolution.md** — Deadline-based dispute resolution with tiered arbitration
- **QoT_Professional_Network_UX_Analysis.md** — Differentiators table, full contract structure
- **ADR_No_Endorsements.md** — Another trust signal boundary decision
- **ADR_Subcontract_Architecture.md** — Contract structure that enables multi-provider projects

---

## Summary

Trust signals must be stake-backed and adversarially validated. Verification is customer acceptance (not provider work). Milestones are coordination containers for payment (trust flows through tasks, not milestones). Publication consent controls visibility (not quality). QA earns trust via separate contracts (following the same structure as any other work). These boundaries keep the trust calculation clean and resistant to gaming.
