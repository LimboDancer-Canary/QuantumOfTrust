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
| Verification/acceptance | No | Customer action, not provider work |
| Publication consent | No | Visibility control, not quality signal |
| QA work | Yes, via separate contract | QA earns trust from their own contracts |

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

## Decision 2: Publication Consent Not Trust-Weighted

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

## Decision 3: QA Earns Trust via Separate Contracts

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

All three decisions follow the same principle:

> **Only stake-backed, adversarially-validated signals flow into trust.**

| Signal | Stake-backed? | Adversarially validated? | Flows into trust? |
|--------|---------------|--------------------------|-------------------|
| Contract outcome | Yes (escrow) | Yes (mutual sign-off) | Yes |
| Verification action | No | No (unilateral) | No |
| Publication consent | No | No (visibility decision) | No |
| QA contract outcome | Yes (escrow) | Yes (mutual sign-off) | Yes |

---

## Consequences

### Positive

1. **Clean termination** — No infinite regress of "who rates the rater"
2. **No coercion vectors** — Publication can't be weaponized
3. **QA as first-class profession** — Clear path to building QA reputation
4. **Consistent model** — Same contract structure applies to all work types

### Negative

1. **Two contracts for tested work** — Operational overhead for customers who want both build and QA
2. **No publication incentive** — System doesn't encourage transparency

### Neutral

1. **QA-build coordination** — Contract B's findings may influence Contract A's outcome, but this is operational, not framework concern

---

## Related Documents

- **QoT_Professional_Network_UX_Analysis.md** — Differentiators table, full contract structure
- **ADR_No_Endorsements.md** — Another trust signal boundary decision
- **ADR_Subcontract_Architecture.md** — Contract structure that enables multi-provider projects

---

## Summary

Trust signals must be stake-backed and adversarially validated. Verification is customer acceptance (not provider work). Publication consent controls visibility (not quality). QA earns trust via separate contracts (following the same structure as any other work). These boundaries keep the trust calculation clean and resistant to gaming.
