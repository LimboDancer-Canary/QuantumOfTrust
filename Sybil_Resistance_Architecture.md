# Sybil Resistance Architecture

## Design Rationale for Defense-in-Depth Against Identity Attacks

---

## Executive Summary

This document describes the Quantum of Trust framework's approach to Sybil resistance. Rather than relying on a single mechanism, we employ four complementary defenses that compose into a robust barrier against identity-based attacks.

The core insight: **no single mechanism stops all attacks, but the combination makes attacks economically irrational**.

---

## Table of Contents

1. [The Threat Model](#the-threat-model)
2. [Why Simple Defenses Fail](#why-simple-defenses-fail)
3. [The Four Mechanisms](#the-four-mechanisms)
4. [How the Mechanisms Compose](#how-the-mechanisms-compose)
5. [Attack Scenario Analysis](#attack-scenario-analysis)
6. [Parameter Tuning](#parameter-tuning)
7. [Edge Cases and Mitigations](#edge-cases-and-mitigations)
8. [Comparison with Alternatives](#comparison-with-alternatives)
9. [Open Questions](#open-questions)

---

## The Threat Model

### What is a Sybil Attack?

A Sybil attack occurs when an adversary creates multiple fake identities to gain disproportionate influence in a system. Named after a case study of dissociative identity disorder, Sybil attacks exploit systems that assume one person = one identity.

### Sybil Attacks in Quantum of Trust

In our context, a Sybil attacker might:

1. **Inflate trust artificially**: Create multiple Avatars that give each other perfect ratings
2. **Bypass eligibility thresholds**: Use fake trust scores to access high-value contracts
3. **Manipulate DAO governance**: Pack a DAO with Sybil identities to control votes
4. **Harvest newcomer contracts**: Create fresh Avatars to repeatedly access "starter" opportunities
5. **Launder negative trust**: Abandon Avatars with bad history and start fresh

### Attacker Capabilities

We assume attackers can:

- Create unlimited new Avatar identities (addresses are cheap)
- Coordinate multiple identities (they control all keys)
- Plan attacks months in advance (patient adversaries)
- Invest moderate capital (not unlimited, but substantial)
- Optimize their strategy based on public mechanism details

We assume attackers cannot:

- Forge cryptographic proofs
- Compromise the underlying blockchain consensus
- Control legitimate high-trust Avatars (social engineering attacks are out of scope)
- Have unlimited capital

---

## Why Simple Defenses Fail

### The Original Three Checks

The initial Sybil resistance relied on:

| Check | What It Measures | Detection Logic |
|-------|------------------|-----------------|
| History Size | Number of contracts | Sybils have fewer contracts each |
| History Depth | Age of oldest contract | New Sybils have shallow history |
| Counterparty Diversity | Unique counterparties | Sybil rings trade among themselves |

### Why They're Insufficient

**History Size**: A patient attacker who plans months ahead can grind out many contracts. If Alice-Sybil and Bob-Sybil execute 100 contracts between themselves over 6 months, each has "large" history.

**History Depth**: Time defeats this check. Create Sybils early, let them age. A 6-month-old Sybil passes any reasonable depth check.

**Counterparty Diversity**: Sybil rings solve this. With 20 Sybils trading in a complete graph (each trades with all 19 others), each Sybil has 19 unique counterparties. The diversity is fake but undetectable by counting.

### The Fundamental Problem

These checks examine the *structure* of history but not its *authenticity*. They ask "do you have history?" not "is your history genuine?"

Genuine history has properties that fake history lacks:

- **Real economic commitment**: Actual value at risk during contracts
- **External validation**: Counterparties with independent reputation
- **Statistical realism**: Natural variation in outcomes
- **Temporal distribution**: Work spread over time, not burst-accumulated

Our four mechanisms target these properties.

---

## The Four Mechanisms

### Mechanism 1: Economic Escrow

**Principle**: Creating contracts costs real money.

**Implementation**:
```
valid_escrow(c) ≡ verify(ε, s) ∧ locked(ε) ∧ owner(ε) ∈ {provider, consumer}
```

Every contract requires locked funds. The escrow commitment ε cryptographically proves that stake s is locked. Funds are released upon contract completion.

**Why it works**:
- Each fake contract requires real capital lockup
- Attack cost scales linearly with contract volume
- Capital has opportunity cost (can't be used elsewhere while locked)
- Gas fees add per-contract overhead

**Cost model**:
```
attack_cost = Σ(stake × lockup_duration × opportunity_rate) + Σ(gas_fees)
```

For a 20-Sybil ring attack with 380 contracts at 1000 tokens each, assuming 7-day average lockup and 5% annual opportunity cost:

```
capital_cost = 380 × 1000 × (7/365) × 0.05 ≈ 365 tokens in opportunity cost
gas_cost = 380 × 2 × gas_per_tx ≈ 760 transactions worth of gas
```

This isn't prohibitive for a determined attacker, but it's no longer free.

### Mechanism 2: Counterparty Trust Weighting

**Principle**: Contracts with low-trust counterparties contribute less.

**Implementation**:
```
γ(c) = σ(V_t(a_consumer) / λ)

where σ(x) = 1 / (1 + e^(-x)) is the sigmoid function
```

The counterparty factor γ scales contract contribution based on counterparty trust:

| Counterparty Trust | γ (λ=50) |
|-------------------|----------|
| -100 | 0.12 |
| -50 | 0.27 |
| 0 | 0.50 |
| +50 | 0.73 |
| +100 | 0.88 |

**Why it works**:
- Sybils start at zero trust: γ = 0.50
- Sybil-to-Sybil contracts contribute only ~50% of face value
- As Sybils accumulate penalties from other mechanisms, γ drops further
- To escape, Sybils need contracts with legitimate high-trust parties
- High-trust parties have no incentive to contract with unproven Sybils

**The bootstrapping trap**: Sybil networks are trapped in a low-trust equilibrium. They can only trade with each other, and those contracts contribute little. Legitimate newcomers escape this trap by working with established parties on small contracts.

### Mechanism 3: Outcome Variance Requirements

**Principle**: Suspiciously perfect histories are implausible.

**Implementation**:
```
plausible(h) ≡ (|h| < N_min) ∨ (var(outcomes(h)) ≥ ε_min)
```

Histories with many contracts must show realistic outcome variance. All +1.0 outcomes are statistically implausible.

**Why it works**:
- Real work has variation: some projects go perfectly, some okay, some poorly
- A history of 50 contracts all rated +1.0 is effectively impossible in reality
- Sybils giving each other perfect scores are caught

**The gaming dilemma**: To pass variance checks, Sybils must give each other some negative outcomes. But negative outcomes reduce trust. They can't maximize trust AND pass variance checks simultaneously.

**ZK-friendly alternative**: Instead of computing exact variance, prove "at least M contracts have outcome < 0.8" where M = max(0, |h| - N_min) × ratio. This achieves the same goal with simpler arithmetic.

### Mechanism 4: Temporal Velocity Limits

**Principle**: Trust can't be accumulated too quickly.

**Implementation**:
```
ν(c) = 1 / (1 + k × max(0, rank(c, T) - N))

where:
  rank(c, T) = this contract's position among contracts in period T
  N = full-weight allowance per period
  k = decay rate
```

Contracts beyond the velocity allowance have diminishing contribution:

| Contract # in period | ν (N=10, k=0.5) |
|---------------------|-----------------|
| 1-10 | 1.00 |
| 11 | 0.67 |
| 15 | 0.29 |
| 20 | 0.17 |
| 50 | 0.05 |

**Why it works**:
- Legitimate trust-building is gradual (few contracts per week)
- Sybil grinding requires volume to overcome other penalties
- High volume triggers severe velocity penalties
- The 50th contract in a week contributes only 5% of its face value

**Progressive vs. hard cutoff**: We use progressive decay rather than hard limits to avoid cliff effects. Legitimate burst activity (a big project push) is mildly penalized; grinding attacks are severely penalized.

---

## How the Mechanisms Compose

The four mechanisms create multiplicative penalties. A contract's contribution is:

```
contribution = ω × outcome × γ × ν
```

Consider a Sybil contract:
- Base weight ω = 10 (moderate stake/difficulty)
- Outcome = +1.0 (perfect score from Sybil friend)
- Counterparty factor γ = 0.5 (Sybil has zero trust)
- Velocity factor ν = 0.2 (this is the 15th contract this week)

```
contribution = 10 × 1.0 × 0.5 × 0.2 = 1.0
```

Compare to a legitimate contract:
- Base weight ω = 10 (same)
- Outcome = +0.8 (realistic good outcome)
- Counterparty factor γ = 0.85 (trusted counterparty at +75 trust)
- Velocity factor ν = 1.0 (first contract this week)

```
contribution = 10 × 0.8 × 0.85 × 1.0 = 6.8
```

The legitimate contract contributes **6.8x more** than the Sybil contract, even though the Sybil contract had a "perfect" outcome.

### The Compounding Effect

Over many contracts, these ratios compound dramatically:

| After N contracts | Sybil Trust | Legitimate Trust | Ratio |
|-------------------|-------------|------------------|-------|
| 10 | ~10 | ~68 | 6.8x |
| 50 | ~35 | ~340 | 9.7x |
| 100 | ~55 | ~680 | 12.4x |

The gap widens because:
- Sybil velocity penalties compound (later contracts count even less)
- Sybil counterparty factor stays low (can't escape the trap)
- Legitimate counterparty factor improves (reputation begets reputation)

---

## Attack Scenario Analysis

### Scenario 1: The Patient Solo Attacker

**Setup**: Single attacker creates 5 Sybil identities 6 months ago. Plans to build them up slowly and use them to access high-value contracts.

**Attack execution**:
- Creates 5 Sybils, waits 6 months (passes depth check)
- Executes 20 contracts between pairs over 2 months (passes size check)
- Uses diverse pairings (passes diversity check)

**Defense analysis**:

| Mechanism | Effect |
|-----------|--------|
| Escrow | 20 contracts × 1000 stake = 20,000 tokens locked over time |
| Counterparty factor | All Sybils at ~0 trust → γ ≈ 0.5 for all contracts |
| Variance | Must include some negative outcomes or fail plausibility |
| Velocity | 20 contracts over 8 weeks = ~2.5/week, ν ≈ 1.0 |

**Outcome**: Each Sybil ends up with trust ≈ 20 × 0.5 × 0.9 = 9 (assuming average 0.9 outcome to pass variance). This is far below what 20 legitimate contracts would yield (~136). Attack is ineffective.

### Scenario 2: The 20-Sybil Ring

**Setup**: Attacker creates 20 Sybil identities, executes contracts in complete graph pattern (each trades with all 19 others).

**Attack execution**:
- 20 × 19 = 380 contracts total
- Each Sybil has 19 contracts with 19 unique counterparties
- Spread over 6 months for depth

**Defense analysis**:

| Mechanism | Effect |
|-----------|--------|
| Escrow | 380 contracts × 1000 stake = 380,000 tokens capital requirement |
| Counterparty factor | All counterparties have low trust → γ ≈ 0.3 average |
| Variance | Must include negative outcomes in each Sybil's history |
| Velocity | 19 contracts per Sybil over 6 months ≈ 3/month, ν ≈ 0.95 |

**Outcome**: Each Sybil ends up with trust ≈ 19 × 0.3 × 0.85 × 0.95 = 4.6. Compare to a legitimate actor with 19 contracts: ~100 trust. The 20-Sybil ring collectively achieves 92 trust; one honest actor achieves 100.

**Key insight**: Creating 20 Sybils is worse than creating 1 honest identity. The attack is not just ineffective—it's actively counterproductive.

### Scenario 3: The Wealthy Attacker

**Setup**: Attacker has unlimited capital and is willing to spend heavily on escrow. Can they buy their way to high trust?

**Attack execution**:
- Massive escrow deposits to signal "seriousness"
- High-stake contracts between Sybils

**Defense analysis**:

| Mechanism | Effect |
|-----------|--------|
| Escrow | Passes—attacker has capital |
| Counterparty factor | Still γ ≈ 0.5 (counterparties are Sybils with no trust) |
| Variance | Still must include negative outcomes |
| Velocity | Still limited to ~10 full-weight contracts per week |

**Outcome**: Wealth bypasses the escrow mechanism but not the others. Counterparty factor and velocity limits still constrain trust accumulation. The attacker can build trust faster than a poor attacker, but not faster than a poor legitimate actor working with established parties.

**Note**: If the attacker uses their wealth to contract with legitimate high-trust parties, they're no longer attacking—they're participating legitimately. That's the desired outcome.

### Scenario 4: The Variance Gamer

**Setup**: Attacker understands the variance requirement and carefully constructs histories that pass the check while maximizing trust.

**Attack execution**:
- Plans outcome distribution: 80% at +1.0, 20% at +0.5
- This yields variance ≈ 0.04 × 0.25 + 0.8 × 0 = 0.01... still too low
- Adjusts to: 60% at +1.0, 30% at +0.6, 10% at -0.2
- Variance now ≈ 0.18, passing threshold

**Defense analysis**:

| Mechanism | Effect |
|-----------|--------|
| Variance | Passes (barely) with engineered distribution |
| Trust impact | Average outcome = 0.6×1 + 0.3×0.6 + 0.1×(-0.2) = 0.76 |

**Outcome**: To pass variance, the attacker must accept lower average outcomes. Their trust accumulation is reduced by ~24% compared to all-perfect scores (which they can't achieve anyway). Combined with counterparty and velocity penalties, Sybil trust remains far below legitimate.

---

## Parameter Tuning

### Counterparty Scale (λ)

**What it controls**: How much trust a counterparty needs before contracts with them count significantly.

| λ value | Trust for γ=0.5 | Trust for γ=0.73 | Trust for γ=0.88 |
|---------|-----------------|------------------|------------------|
| 25 | 0 | 25 | 50 |
| 50 | 0 | 50 | 100 |
| 100 | 0 | 100 | 200 |

**Tradeoffs**:
- Lower λ: Stricter on newcomers, faster Sybil trap convergence
- Higher λ: More forgiving to newcomers, slower Sybil trap

**Recommendation**: λ = 50 balances accessibility and security.

### Velocity Parameters (T, N, k)

**What they control**: How many full-weight contracts per time period.

| Configuration | Effect |
|---------------|--------|
| T=7d, N=10, k=0.5 | ~10 contracts/week at full weight |
| T=7d, N=5, k=0.5 | ~5 contracts/week at full weight |
| T=30d, N=20, k=0.3 | ~20 contracts/month at full weight |

**Tradeoffs**:
- Shorter T, lower N: Stricter velocity, better burst protection, may frustrate legitimate users
- Longer T, higher N: More permissive, worse burst protection, better UX

**Recommendation**: T=7 days, N=10, k=0.5 allows reasonable weekly activity while preventing grinding.

### Variance Parameters (N_min, ε_min)

**What they control**: When variance is checked and how much is required.

| Configuration | Effect |
|---------------|--------|
| N_min=5, ε_min=0.05 | Early, loose checking |
| N_min=10, ε_min=0.1 | Moderate checking |
| N_min=20, ε_min=0.15 | Late, strict checking |

**Tradeoffs**:
- Lower N_min: Catches gaming earlier, more false positives for lucky newcomers
- Higher ε_min: Requires more variation, harder to game, may flag legitimate specialists

**Recommendation**: N_min=10, ε_min=0.1 catches gaming without flagging normal variation.

---

## Edge Cases and Mitigations

### Edge Case 1: Legitimate Newcomers

**Problem**: New users have zero trust, so their counterparty factor is 0.5.

**Mitigation**: 
- Low-stake contracts are still accessible (threshold is low)
- Established parties can choose to contract with newcomers (mentorship)
- 50% contribution is not zero—newcomers can still build trust, just slower
- This mirrors reality: unproven workers start with lower-stakes opportunities

### Edge Case 2: Niche Specialists

**Problem**: A legitimate specialist might have 50 contracts all rated highly in a narrow field.

**Mitigation**:
- Variance check uses ε_min=0.1, which allows 90% of contracts at +0.95 and 10% at +0.7
- Real specialists still have some variation (hard problems occasionally don't go perfectly)
- If truly no variation, the specialist can include a few deliberately modest ratings
- This is a feature: even experts should demonstrate range

### Edge Case 3: Project Crunch

**Problem**: A legitimate team might complete 20 contracts in one week during a major project.

**Mitigation**:
- Velocity penalty is progressive, not cliff-based
- Contracts 11-20 still contribute (~40-70% weight on average)
- The total contribution is reduced but not eliminated
- If this is common in a domain, parameters can be adjusted

### Edge Case 4: Recovering from Bad History

**Problem**: Someone with legitimate negative trust from past failures wants to rebuild.

**Mitigation**:
- The system allows recovery through consistent positive performance
- Velocity limits prevent quick reputation laundering
- Counterparty factor actually helps here: as trust improves, γ improves
- Recovery is slow but possible—exactly the right behavior

### Edge Case 5: DAO Formation with Newcomers

**Problem**: A new DAO with all-newcomer members has low aggregate trust.

**Mitigation**:
- DAOs choose their aggregation function (sum, average, etc.)
- A "sum" DAO can accumulate collective trust even from low-trust members
- Individual members build trust in parallel, benefiting the DAO over time
- Mixed DAOs (some established, some new) can leverage established members' trust

---

## Comparison with Alternatives

### Proof of Humanity

**Approach**: Verify that each identity corresponds to a unique human (biometrics, video calls, social vouching).

**Tradeoffs**:
- ✓ Strong Sybil resistance if verification is robust
- ✗ Couples trust to identity (violates core q⟨T⟩ principle)
- ✗ Privacy-destroying (must reveal identity to verify)
- ✗ Excludes legitimate multi-Avatar use cases

**Our approach**: Maintains privacy while achieving similar resistance through economics.

### Proof of Work / Stake at Identity Level

**Approach**: Require computational work or stake deposit to create identities.

**Tradeoffs**:
- ✓ Creates cost for identity creation
- ✗ One-time cost; once created, Sybils persist forever
- ✗ Doesn't address ongoing behavior
- ✗ Wealthy attackers can pay the cost

**Our approach**: Ongoing cost through escrow, counterparty factor, and velocity limits.

### Social Graph Analysis

**Approach**: Detect Sybils through graph topology analysis (clustering, PageRank anomalies).

**Tradeoffs**:
- ✓ Can detect ring structures
- ✗ Computationally expensive
- ✗ Hard to do in ZK
- ✗ False positives for legitimate tight-knit communities

**Our approach**: Achieves similar effects through counterparty weighting without explicit graph analysis.

### Reputation Import

**Approach**: Import reputation from existing systems (GitHub, LinkedIn, etc.).

**Tradeoffs**:
- ✓ Bootstraps from existing Sybil resistance
- ✗ Re-couples to external identity
- ✗ Imports biases and limitations of source systems
- ✗ Not all legitimate actors have external reputation

**Our approach**: Allows fresh-start building of reputation within the system.

---

## Open Questions

### 1. Escrow Release Timing

**Question**: When exactly should escrow be released?

**Options**:
- Immediately upon outcome recording
- After a dispute window (7 days?)
- Partial release with holdback

**Current stance**: Immediate release upon outcome. Dispute mechanisms are separate.

### 2. Negative Trust Counterparty Factor

**Question**: Should γ(c) become negative when counterparty has deeply negative trust?

**Current stance**: No. γ is bounded to (0, 1). Negative trust counterparty just means the contract counts for very little, not that it actively hurts you.

### 3. Cross-Skill Velocity

**Question**: Should velocity limits apply per-skill or globally?

**Options**:
- Per-skill: Can build 10 engineering + 10 design contracts per week
- Global: Total 10 contracts per week across all skills

**Current stance**: Per-skill, to avoid penalizing legitimate generalists.

### 4. DAO Implications

**Question**: How do these mechanisms interact with DAO aggregation?

**Current stance**: Mechanisms apply at the Agent level. DAOs aggregate already-computed Agent trust values. DAO-specific Sybil resistance (packing DAOs with Sybil members) is a separate concern addressed through DAO governance.

### 5. Parameter Governance

**Question**: Who sets and adjusts parameters over time?

**Options**:
- Fixed at protocol level (immutable)
- DAO governance (community votes)
- Per-contract customization (contract posters choose)

**Current stance**: Protocol-level defaults with per-contract overrides for stricter (not looser) requirements.

---

## Summary

Sybil resistance in Quantum of Trust is achieved through four complementary mechanisms:

| Mechanism | What It Prevents | Key Property |
|-----------|------------------|--------------|
| Economic Escrow | Free fake contracts | Cost scales with attack volume |
| Counterparty Weighting | Trust laundering | Low-trust networks are trapped |
| Outcome Variance | Perfect-score gaming | Must accept lower average outcomes |
| Velocity Limits | Burst grinding | Trust accumulation is rate-limited |

These mechanisms compose multiplicatively, creating a robust defense where:

1. **Simple attacks fail dramatically** (10x+ disadvantage vs. honest behavior)
2. **Sophisticated attacks fail economically** (cost exceeds benefit)
3. **Legitimate newcomers can still participate** (slower but not blocked)
4. **Privacy is preserved** (no identity verification required)

The system makes one strong claim: **honest behavior is the optimal strategy**. Any attempt to game the system produces worse results than simply participating legitimately.

---

*This document accompanies the Quantum of Trust whitepaper and implementation.*
