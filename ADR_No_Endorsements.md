# ADR: No Endorsements — Contract Outcomes Are Sufficient

**Status:** Accepted  
**Date:** December 30, 2025  
**Deciders:** Dennis  
**Context:** QoT Professional Network design

---

## Decision

QoT does not include an endorsement mechanism. Trust flows exclusively from contract outcomes.

---

## Context

LinkedIn has endorsements ("Alice endorsed Bob for JavaScript"). The question arose whether QoT should include a similar attestation system where users could vouch for others' capabilities.

---

## Options Considered

### Option A: Trust Attestations (Rejected)

Allow users to endorse others, with endorsements weighted by the endorser's trust score:

```
attestation: {
  from: alice,
  subject: bob,
  skill: "typescript_development",
  value: "positive",
  weight: alice.trust_score  // γ(c) weighting
}
```

**Problems identified:**

| Constraint | Result |
|------------|--------|
| Only counterparties can attest | Redundant — contract outcome already captured the signal |
| Anyone can attest | Gameable — no stakes, opens collusion vectors |

If Alice was Bob's counterparty, she already rated Bob via the contract outcome. Adding an endorsement double-counts the signal.

If Alice wasn't Bob's counterparty, how does she know Bob's capability? She's claiming things without stakes. Alice and Bob can trade endorsements.

### Option B: No Endorsements (Accepted)

Trust comes only from contract outcomes. No endorsement mechanism exists.

The contract outcome IS the attestation. Both parties signing off on phase completion is the verification.

---

## Decision Rationale

### LinkedIn's Problem

LinkedIn endorsements are notoriously worthless precisely because:
- No stakes involved
- No verification of actual collaboration  
- Reciprocal endorsement gaming is trivial

### The Dilemma

Attestations as trust inputs are either:

| Design | Outcome |
|--------|---------|
| Counterparty-only attestations | Redundant with contract outcome |
| Open attestations | Gameable, no stakes |

There is no middle ground that adds signal without adding gaming vectors.

### What Mutual Sign-Off Already Provides

Each contract phase requires sign-off from provider AND consumer:
- Provider can't claim unearned success (consumer must agree)
- Consumer can't deny legitimate delivery (provider must agree)
- The adversarial dynamic provides natural validation

This is already an attestation — just one with stakes attached.

---

## Consequences

### Positive

1. **Ungameable** — No mechanism for trading fake endorsements
2. **Cleaner math** — Trust equation has fewer inputs to validate
3. **Clear signal** — Every trust point traces to a real contract
4. **Differentiation** — Distinct from LinkedIn's low-signal endorsements

### Negative

1. **No testimonials** — Users cannot add qualitative descriptions to contracts
2. **Cold start harder** — New users cannot bootstrap with endorsements from known contacts

### Considered and Rejected

**Testimonials as non-trust metadata**: Allow testimonials that don't affect trust calculation. Rejected because it adds complexity without adding trust signal — if it doesn't flow into the math, why is it in the system?

---

## Related Documents

- **QoT_Professional_Network_UX_Analysis.md** — Differentiators table
- **ADR_Trust_Signal_Boundaries.md** — What affects trust vs. what doesn't

---

## Summary

The contract outcome IS the attestation. Separate endorsement layers are either redundant or gameable. QoT achieves signal integrity by refusing to add a mechanism that cannot provide trustworthy signal.
