# The Quantum of Trust: Math Equations in Plain English

## A Complete Guide to Understanding the Mathematical Framework

---

This document catalogs every mathematical equation from the Quantum of Trust whitepaper and provides clear, plain English explanations. Each equation is presented with:

1. The original mathematical notation
2. A literal "plain English" translation
3. An expanded explanation of what it means and why it matters

---

## Table of Contents

1. [The Recursive Type Definition](#1-the-recursive-type-definition)
2. [The Valuation Function](#2-the-valuation-function)
3. [Trust Value Meanings](#3-trust-value-meanings)
4. [Agent Trust Value Calculation](#4-agent-trust-value-calculation)
5. [Contract Structure](#5-contract-structure)
6. [Outcome Range](#6-outcome-range)
7. [The Base Weighting Function](#7-the-base-weighting-function)
8. [Eligibility Condition](#8-eligibility-condition)
9. [Threshold Function](#9-threshold-function)
10. [History Evolution](#10-history-evolution)
11. [Trust Evolution](#11-trust-evolution)
12. [DAO Valuation](#12-dao-valuation)
13. [Sybil Resistance](#13-sybil-resistance)
14. [Convergence Criterion](#14-convergence-criterion)
15. [Counterparty Trust Factor](#15-counterparty-trust-factor)
16. [Velocity Weight](#16-velocity-weight)
17. [Outcome Variance Constraint](#17-outcome-variance-constraint)
18. [Escrow Verification](#18-escrow-verification)

---

## 1. The Recursive Type Definition

### The Math

$$q\langle T \rangle ::= \text{Agent}(t, h_t) \mid \text{DAO}(\{q\langle T \rangle\})$$

### Plain English Translation

> "A Quantum of Trust is defined as one of two things: either it's an **Agent** (which has a skill type and a history of contracts in that skill), or it's a **DAO** (which is a group containing other Quanta of Trust)."

### Expanded Explanation

This is the foundational definition. Think of it like defining what a "building block" is:

- **Agent**: A single Avatar with a specific skill (like "engineering" or "design") and a track record of completed contracts in that skill.

- **DAO**: A group that contains other building blocks. Those building blocks can be individual Agents OR other DAOs.

The `::=` symbol means "is defined as," and the `|` symbol means "or."

The powerful insight here is that this definition is *recursive*—a DAO contains q⟨T⟩ units, which can themselves be DAOs. It's turtles all the way down. This allows for unlimited organizational complexity: teams within teams, organizations within organizations.

**Real-world analogy**: It's like saying "A LEGO structure is either a single brick, or a collection of LEGO structures." You can build anything from this simple definition.

---

## 2. The Valuation Function

### The Math

$$V_t: q\langle T \rangle \rightarrow \mathbb{R}$$

### Plain English Translation

> "The trust value function (V) takes any Quantum of Trust and produces a real number."

### Expanded Explanation

This equation defines what the "V" function does:

- **Input**: Any q⟨T⟩ (could be an Agent or a DAO)
- **Output**: A real number (ℝ means "all real numbers"—positive, negative, or zero)

The subscript "t" indicates this is scoped to a specific skill type. So $V_{engineering}$ measures engineering trust, while $V_{design}$ measures design trust—they're completely independent.

**Real-world analogy**: It's like a credit score, but:
1. You have a separate score for each skill you practice
2. Your score can go negative (not just low, but actively in the red)
3. The score is calculated the same way for everyone

---

## 3. Trust Value Meanings

### The Math

- $V_t = 0$ → unknown, no track record
- $V_t > 0$ → net positive history, trusted
- $V_t < 0$ → net negative history, actively distrusted

### Plain English Translation

> "A trust value of zero means no track record yet. A positive value means more successes than failures—you're trusted. A negative value means more failures than successes—you're actively distrusted."

### Expanded Explanation

This is crucial: **negative trust is not the same as zero trust**.

- **Zero trust** ($V_t = 0$): "I don't know anything about this Avatar." A newcomer. Might deserve a chance on small contracts.

- **Positive trust** ($V_t > 0$): "This Avatar has a track record of delivering." The higher the number, the more demonstrated reliability.

- **Negative trust** ($V_t < 0$): "This Avatar has *earned* distrust through demonstrated failure." This is signal, not absence of signal. An Avatar with $V_t = -50$ is worse than an unknown newcomer—they've proven they can't deliver.

**Real-world analogy**: Think of a restaurant review average. No reviews (0) is different from terrible reviews (-50). A new restaurant with no reviews might get a chance; a restaurant with a history of food poisoning incidents won't.

---

## 4. Agent Trust Value Calculation

### The Math

$$V_t(\text{Agent}(t, h_t)) = \sum_{c \in h_t} \omega(c) \cdot \text{outcome}(c) \cdot \gamma(c) \cdot \nu(c)$$

### Plain English Translation

> "An Agent's trust value equals the sum of each contract's contribution, where each contribution is: (base weight) × (outcome) × (counterparty factor) × (velocity factor), added up across all contracts in their history."

### Expanded Explanation

Breaking this down piece by piece:

- $V_t(\text{Agent}(t, h_t))$ — "The trust value of an Agent who has skill type t and history h"
- $\sum_{c \in h_t}$ — "Add up the following for every contract c in the history h"
- $\omega(c)$ — "The base weight of that contract" (stake, difficulty, recency)
- $\text{outcome}(c)$ — "How well the contract went" (success or failure)
- $\gamma(c)$ — "The counterparty trust factor" (see [#15](#15-counterparty-trust-factor))
- $\nu(c)$ — "The velocity weight" (see [#16](#16-velocity-weight))
- $\cdot$ — "multiplied by"

So for each contract, four factors determine its contribution:
1. **Base weight**: High stakes + high difficulty + recent = high base weight
2. **Outcome**: Success (+1) to failure (-1)
3. **Counterparty factor**: Contracts with trusted counterparties count more
4. **Velocity factor**: Burst activity gets diminishing returns

**Example**: Imagine an Avatar with 3 contracts:
1. Big project with trusted counterparty, succeeded: ω=10, outcome=+1, γ=0.9, ν=1.0 → contributes +9.0
2. Medium project with new counterparty, partial success: ω=5, outcome=+0.5, γ=0.5, ν=1.0 → contributes +1.25
3. Small project (burst activity), failed: ω=2, outcome=-1, γ=0.8, ν=0.5 → contributes -0.8

Total trust = 9.0 + 1.25 + (-0.8) = **9.45 cutes**

---

## 5. Contract Structure

### The Math

$$c = (a_{\text{provider}}, a_{\text{consumer}}, t, s, d, \tau, \varepsilon, V_{\text{consumer}})$$

### Plain English Translation

> "A contract is defined by eight things: who's providing the service, who's consuming it, what skill type it involves, how much is at stake, how difficult it is, its deadline, the escrow commitment, and the consumer's trust level when the contract completed."

### Expanded Explanation

This tuple defines what a contract contains:

| Symbol | Meaning | Example |
|--------|---------|---------|
| $a_{\text{provider}}$ | The Avatar doing the work | "DevBot_Jane" |
| $a_{\text{consumer}}$ | The Avatar hiring/paying | "StartupCo_Avatar" |
| $t$ | The skill type | "Engineering" |
| $s$ | The stake (value at risk) | 1000 tokens |
| $d$ | Difficulty rating (0-10) | 7 |
| $\tau$ | Deadline/timestamp | "2025-03-01" |
| $\varepsilon$ | Escrow commitment | Cryptographic proof that stake is locked |
| $V_{\text{consumer}}$ | Consumer's trust at completion | 45.2 (snapshot) |

**Why these eight?** The original six determine how much a contract should "count" toward trust. The two new fields enable Sybil resistance:

- **Escrow commitment** ($\varepsilon$): Proves the stake is real, not just a claimed number. Creating fake contracts now costs real money.
- **Consumer trust snapshot** ($V_{\text{consumer}}$): Records the counterparty's trust at completion time. Used to calculate the counterparty factor. Stored as a snapshot so trust calculations remain stable and independent.

---

## 6. Outcome Range

### The Math

$$\text{outcome}(c) \in [-1, 1]$$

### Plain English Translation

> "The outcome of a contract is a number between negative one and positive one."

### Expanded Explanation

The $\in$ symbol means "is a member of" or "falls within."

This continuous range allows for nuance:

| Outcome | Meaning |
|---------|---------|
| +1.0 | Complete success—everything delivered perfectly |
| +0.5 | Good but not perfect—met expectations |
| 0 | Neutral—partial delivery, or cancelled by mutual agreement |
| -0.5 | Problematic—significant issues but some value delivered |
| -1.0 | Complete failure—nothing delivered, or actively harmful |

**Why continuous instead of just pass/fail?** Because real work has degrees of quality. A project that's 80% complete isn't the same as one that was never started.

---

## 7. The Base Weighting Function

### The Math

$$\omega(c) = f\big(s(c),\ d(c),\ \text{recency}(c)\big)$$

### Plain English Translation

> "The base weight of a contract is a function of three things: the stake involved, the difficulty, and how recent the contract is."

### Expanded Explanation

This function determines the **base weight**—how much a contract should count before adjustment factors are applied. Three factors:

1. **Stake** $s(c)$: A $100,000 contract matters more than a $100 contract. If you can deliver on high-stakes work, that's meaningful signal.

2. **Difficulty** $d(c)$: Completing a hard task proves more than completing an easy one. Anyone can succeed at trivial work.

3. **Recency**: Recent contracts matter more than old ones. Trust requires ongoing reinforcement. Coasting on old successes doesn't work forever.

**Note**: In earlier versions, counterparty trust was included in the base weight. It's now a separate adjustment factor (γ) applied after the base weight is calculated. This separation makes the Sybil resistance properties more explicit.

**Real-world analogy**: The base weight is like the "raw importance" of a recommendation letter—recent, for difficult work, with real stakes. The counterparty factor (γ) then adjusts based on who wrote the letter.

---

## 8. Eligibility Condition

### The Math

$$\text{eligible}(a, c) \iff V_t(a) \geq \theta(c) \land \text{plausible}(h_t(a))$$

### Plain English Translation

> "An Avatar is eligible for a contract if and only if their trust value meets the threshold AND their history is plausible (not suspicious)."

### Expanded Explanation

The $\iff$ symbol means "if and only if"—it's a two-way condition. The $\land$ symbol means "and."

This is the **core gatekeeping mechanism** with two requirements:

1. **Trust threshold**: Only Avatars who meet or exceed the minimum trust requirement ($\theta$) can bid
2. **History plausibility**: The Avatar's history must pass the variance check (see [#17](#17-outcome-variance-constraint))

**Why both?** The trust threshold ensures capability. The plausibility check catches gaming—an Avatar with 50 perfect contracts might have high trust but is statistically suspicious.

**The key privacy innovation**: Zero-knowledge proofs let an Avatar prove "I meet the threshold AND my history is plausible" without revealing their exact score or any of their contract history.

---

## 9. Threshold Function

### The Math

$$\theta(c) = \log(1 + s(c)) \cdot d(c)$$

### Plain English Translation

> "The minimum trust required for a contract equals the logarithm of (one plus the stake) multiplied by the difficulty rating."

### Expanded Explanation

This formula calculates how much trust you need to be eligible:

- **$\log(1 + s)$**: The logarithm of stake-plus-one. Using logarithm means high stakes raise the bar, but not linearly. A $1M contract doesn't require 1000x more trust than a $1000 contract—maybe 3x more.

- **$d$**: Difficulty multiplier. Harder contracts require proportionally more trust.

- The "$1+$" ensures the formula works even for zero-stake contracts (since $\log(0)$ is undefined).

**Example calculations**:

| Stake | Difficulty | Threshold |
|-------|------------|-----------|
| $100 | 1 | log(101) × 1 ≈ 4.6 |
| $100 | 5 | log(101) × 5 ≈ 23 |
| $10,000 | 5 | log(10,001) × 5 ≈ 46 |
| $100,000 | 8 | log(100,001) × 8 ≈ 92 |

The logarithm prevents runaway requirements—you don't need infinite trust for high-value contracts.

---

## 10. History Evolution

### The Math

$$h_t^{(n+1)}(a) = h_t^{(n)}(a) \cup \{c_n\}$$

### Plain English Translation

> "An Avatar's history after completing contract n equals their previous history plus that new contract."

### Expanded Explanation

The $\cup$ symbol means "union" or "combined with."

This equation describes how history grows:

- $h_t^{(n)}(a)$ — "Avatar a's history at time n" (all contracts so far)
- $\{c_n\}$ — "The set containing just the new contract"
- $h_t^{(n+1)}(a)$ — "Avatar a's history after the new contract"

**In simple terms**: Every time you complete a contract, it gets added to your permanent record. Your history is append-only—it only grows, never shrinks.

**Why this matters**: You can't erase bad contracts. If you fail, that failure stays in your history forever (though it fades in weight over time due to recency decay).

---

## 11. Trust Evolution

### The Math

$$V_t^{(n+1)}(a) = V_t^{(n)}(a) + \omega(c_n) \cdot \text{outcome}(c_n) \cdot \gamma(c_n) \cdot \nu(c_n)$$

### Plain English Translation

> "Your trust value after completing a contract equals your previous trust value plus the new contract's full contribution (base weight × outcome × counterparty factor × velocity factor)."

### Expanded Explanation

This is the **trust update rule**—how trust changes after each contract:

- $V_t^{(n)}(a)$ — Your trust before this contract
- $\omega(c_n) \cdot \text{outcome}(c_n) \cdot \gamma(c_n) \cdot \nu(c_n)$ — The contribution from this contract
- $V_t^{(n+1)}(a)$ — Your trust after this contract

**Key insight**: Trust changes incrementally. Each contract moves your score up or down based on:
- How important the contract was (base weight)
- How well you did (outcome)
- How trusted your counterparty was (counterparty factor)
- Whether you're accumulating too fast (velocity factor)

**Examples**:
- Succeed at a big contract with trusted counterparty: (+8 × +1 × 0.9 × 1.0) → trust increases by 7.2
- Fail at a small contract: (+2 × -1 × 0.7 × 1.0) → trust decreases by 1.4
- Burst activity (10th contract this week): (+5 × +1 × 0.8 × 0.3) → trust increases by only 1.2

**There is no coasting**: If you stop working, your trust doesn't stay the same—it gradually decays as recency weighting makes old contracts count for less.

---

## 12. DAO Valuation

### The Math

$$V_t(\text{DAO}(S)) = \Phi\left(\{V_t(q) : q \in S\}\right)$$

### Plain English Translation

> "A DAO's trust value equals the aggregation function applied to the trust values of all its members."

### Expanded Explanation

Breaking this down:

- $\text{DAO}(S)$ — A DAO with member set S
- $\{V_t(q) : q \in S\}$ — "The set of trust values for each member q in S"
- $\Phi$ — An aggregation function (chosen by the DAO's governance)

**What aggregation function?** The DAO decides. Common choices:

| Function | Meaning | Use Case |
|----------|---------|----------|
| Sum | Total combined capability | Capacity-focused: "We can handle this much work" |
| Average | Mean reliability | Balanced: "This is our typical performance" |
| Minimum | Weakest link | Security-focused: "We're only as good as our worst member" |
| Maximum | Best member | Star-focused: "Our best person can handle this" |

**Example**: A security DAO might use minimum—if one member is compromised, the whole DAO's security rating drops to that level.

---

## 13. Sybil Resistance

### The Math

Sybil resistance is achieved through four complementary mechanisms:

$$\text{cost}(\text{attack}) \propto \sum_{c} s(c) \quad \text{(Economic Escrow)}$$

$$\gamma(c) = \sigma\left(\frac{V_t(a_{\text{consumer}})}{\lambda}\right) \quad \text{(Counterparty Weighting)}$$

$$\text{plausible}(h) \iff |h| < N_{\min} \lor \text{var}(\text{outcomes}) \geq \varepsilon_{\min} \quad \text{(Variance)}$$

$$\frac{\Delta V_t}{\Delta \text{time}} \leq v_{\max} \quad \text{(Velocity Limits)}$$

### Plain English Translation

> "Sybil attacks are defeated through four mechanisms: (1) creating contracts costs real money, (2) contracts with low-trust counterparties count less, (3) suspiciously perfect histories are flagged, and (4) trust can't be accumulated too quickly."

### Expanded Explanation

**What's a Sybil attack?** Creating multiple fake identities to game the system.

**Why the original defense was insufficient**: The original equation only captured one aspect:

$$|h_t(a_{\text{honest}})| > |h_t(a_{\text{sybil}_i})| \quad \forall i$$

This says "honest Avatars have bigger histories than individual Sybils." But a sophisticated attacker could create 20 Sybils six months ago, have them trade with each other, and each would have deep history, large size, and diverse counterparties—all fake.

**The four mechanisms work together**:

| Mechanism | What It Prevents | How |
|-----------|------------------|-----|
| Economic Escrow | Free fake contracts | Must lock real funds for every contract |
| Counterparty Weighting | Trust laundering | Contracts with low-trust Sybils contribute little |
| Outcome Variance | Perfect-score gaming | All +1.0 outcomes are statistically implausible |
| Velocity Limits | Burst grinding | Can't gain unlimited trust quickly |

**Example: The 20-Sybil Ring Attack**

Without the mechanisms: Attacker creates 20 Sybils, each trades with all 19 others, giving perfect scores. Each Sybil ends up with high trust.

With the mechanisms:
1. **Escrow**: 380 contracts require locking real capital (20 × 19)
2. **Counterparty factor**: All counterparties start at zero trust, so γ ≈ 0.27, meaning contracts contribute only 27% of face value
3. **Variance**: All +1.0 outcomes fail the plausibility check
4. **Velocity**: Creating 19 contracts quickly means most have ν << 1

Result: Each Sybil ends up with trust far below what an honest single-identity actor would achieve. The attack is economically irrational.

---

## 14. Convergence Criterion

### The Math

$$\lim_{n \to \infty} \text{Corr}\big(V_t^{(n)}(a), R_t(a)\big) = 1$$

### Plain English Translation

> "As the number of contracts approaches infinity, the correlation between measured trust and actual reliability approaches one (perfect correlation)."

### Expanded Explanation

This is the **validation criterion**—how we know the math works:

- $\lim_{n \to \infty}$ — "As n goes to infinity" (as we get more and more data)
- $\text{Corr}(...)$ — The correlation coefficient (how closely two things track each other)
- $V_t^{(n)}(a)$ — The trust score the network calculates
- $R_t(a)$ — The Avatar's *actual* reliability (known to simulators, not to the network)
- $= 1$ — Perfect correlation

**What this means**: Over time, the trust scores should accurately reflect who's actually reliable. If the math works:

- Reliable Avatars end up with high scores
- Unreliable Avatars end up with low (or negative) scores
- The correlation between "what the network thinks" and "what's actually true" approaches 1

**How we test this**: In simulation, we know each Avatar's true reliability (we control it). We run the network and check: do the computed trust values match the true reliability? When they do, the mathematics are validated.

---

## 15. Counterparty Trust Factor

### The Math

$$\gamma(c) = \sigma\left(\frac{V_t(a_{\text{consumer}})}{\lambda}\right)$$

Where $\sigma(x) = \frac{1}{1 + e^{-x}}$ is the sigmoid function.

### Plain English Translation

> "The counterparty trust factor is the sigmoid of the consumer's trust divided by a scale parameter. This produces a value between 0 and 1 that's low when the counterparty has low or negative trust, and high when the counterparty has high trust."

### Expanded Explanation

This factor adjusts how much a contract contributes based on **who you worked with**:

- $V_t(a_{\text{consumer}})$ — The trust level of the Avatar who hired you
- $\lambda$ — Scale parameter (suggested: 50)
- $\sigma$ — Sigmoid function, which smoothly maps any number to the range (0, 1)

**How the sigmoid works**:

| Counterparty Trust | γ (with λ=50) | Effect |
|-------------------|---------------|--------|
| -100 | ≈ 0.12 | Contract counts for only 12% |
| -50 | ≈ 0.27 | Contract counts for only 27% |
| 0 | = 0.50 | Contract counts for 50% |
| +50 | ≈ 0.73 | Contract counts for 73% |
| +100 | ≈ 0.88 | Contract counts for 88% |

**Why this creates a Sybil trap**: New Sybils have zero trust. When they contract with each other, γ ≈ 0.5. As they accumulate negative trust (from variance/velocity penalties), γ drops further. To escape, they need contracts with legitimate high-trust parties—who have no incentive to contract with unproven Sybils.

**The bootstrapping problem**: This also affects legitimate newcomers. The difference: legitimate newcomers can work with established parties by starting small and proving themselves. Sybil networks can only work with each other.

---

## 16. Velocity Weight

### The Math

$$\nu(c) = \frac{1}{1 + k \cdot \max(0, \text{rank}(c, T) - N)}$$

### Plain English Translation

> "The velocity weight starts at 1 for your first N contracts in a time period, then decreases for each additional contract. The more contracts you try to cram into a short period, the less each one counts."

### Expanded Explanation

This factor prevents **burst attacks** where someone tries to grind many contracts quickly:

- $\text{rank}(c, T)$ — This contract's position among all contracts completed in time period T
- $N$ — Number of full-weight contracts allowed per period (suggested: 10)
- $k$ — Decay rate (suggested: 0.5)

**How it works**:

| Contract # in period | ν (with N=10, k=0.5) | Effect |
|---------------------|----------------------|--------|
| 1-10 | 1.0 | Full weight |
| 11 | 0.67 | 67% weight |
| 12 | 0.50 | 50% weight |
| 15 | 0.29 | 29% weight |
| 20 | 0.17 | 17% weight |

**Why progressive decay instead of hard cutoff?** A hard limit ("only 10 contracts per week count") creates cliff effects and punishes legitimate burst activity. Progressive decay is softer: extra contracts still count, just less. Legitimate users doing a big project push are mildly affected; Sybils grinding 100 contracts get severely diminished returns.

**Parameters**:
- $T$ — Time period (suggested: 7 days)
- $N$ — Full-weight allowance (suggested: 10 contracts)
- $k$ — Decay rate (suggested: 0.5)

---

## 17. Outcome Variance Constraint

### The Math

$$\text{plausible}(h) \iff |h| < N_{\min} \lor \text{var}(\text{outcomes}(h)) \geq \varepsilon_{\min}$$

### Plain English Translation

> "A history is plausible if either: it's small (fewer than the minimum for statistical analysis), OR its outcome variance meets the minimum threshold. Histories with many contracts but suspiciously uniform outcomes are implausible."

### Expanded Explanation

The $\lor$ symbol means "or."

This constraint catches **outcome gaming**—when someone gives themselves (or their Sybil friends) perfect scores:

- $|h|$ — Size of the history
- $N_{\min}$ — Minimum size for variance check (suggested: 10)
- $\text{var}(\text{outcomes})$ — Statistical variance of outcome values
- $\varepsilon_{\min}$ — Minimum required variance (suggested: 0.1)

**Why small histories are exempt**: With only 5 contracts, you might legitimately have all successes. With 50 contracts, all +1.0 outcomes are statistically implausible—real work has variation.

**What counts as suspicious?**

| History | Outcome Distribution | Variance | Plausible? |
|---------|---------------------|----------|------------|
| 5 contracts | All +1.0 | 0 | ✓ (too small to judge) |
| 50 contracts | All +1.0 | 0 | ✗ (impossible) |
| 50 contracts | Mix of +0.7 to +1.0 | 0.08 | ✗ (barely, but suspicious) |
| 50 contracts | Mix of +0.3 to +1.0 | 0.15 | ✓ (realistic) |
| 50 contracts | Mix of -0.5 to +1.0 | 0.35 | ✓ (realistic) |

**ZK-friendly alternative**: Computing exact variance is expensive in zero-knowledge circuits. An equivalent constraint: "At least M contracts have outcome < 0.8" where M scales with history size. This achieves the same goal with simpler arithmetic.

---

## 18. Escrow Verification

### The Math

$$\text{valid\_escrow}(c) \iff \text{verify}(\varepsilon, s) \land \text{locked}(\varepsilon) \land \text{owner}(\varepsilon) \in \{a_{\text{provider}}, a_{\text{consumer}}\}$$

### Plain English Translation

> "An escrow is valid if and only if: the cryptographic commitment verifies against the stated stake amount, the funds are currently locked, and the escrow is owned by one of the contract parties."

### Expanded Explanation

The $\land$ symbol means "and."

This equation defines **what makes an escrow commitment valid**:

1. **verify(ε, s)**: The cryptographic proof ε must verify that exactly s tokens are committed. You can't claim a 1000-token stake while only locking 10 tokens.

2. **locked(ε)**: The escrow must be currently locked, not already released. You can't reuse the same escrow for multiple contracts.

3. **owner(ε) ∈ {provider, consumer}**: The locked funds must belong to one of the contract parties. You can't use someone else's escrow.

**Why this matters for Sybil resistance**: Creating fake contracts now has real cost. If you want to give your Sybil friend a contract with 1000-token stake, you must actually lock 1000 tokens. For a 20-Sybil ring attack with 380 contracts at 1000 tokens each, you'd need to lock 380,000 tokens in escrow at various times. Even if you get them back after each contract completes, the capital requirement and opportunity cost make attacks expensive.

**How escrow works in practice** (on Aztec):
1. Before creating a contract, one party locks tokens in a private note
2. The note's nullifier prevents double-spending
3. Upon contract completion, the escrow is released based on outcome
4. The note system ensures privacy while proving the commitment exists

---

## Summary: The Complete Picture

Here's how all the equations fit together:

```
1. DEFINITION: What is a Quantum of Trust?
   q⟨T⟩ = Agent(skill, history) OR DAO({members})

2. MEASUREMENT: How do we measure trust?
   V_t: q⟨T⟩ → ℝ (produces a real number)
   
3. CALCULATION: How is an Agent's trust computed?
   V_t = Σ(base_weight × outcome × counterparty_factor × velocity_factor)

4. BASE WEIGHTING: What makes a contract's base weight high?
   ω = f(stake, difficulty, recency)

5. COUNTERPARTY FACTOR: How does counterparty trust affect contribution?
   γ = sigmoid(counterparty_trust / scale)

6. VELOCITY FACTOR: How does burst activity affect contribution?
   ν = 1 / (1 + k × excess_contracts_in_period)

7. GATEKEEPING: Who can access what contracts?
   eligible if V_t ≥ threshold AND history is plausible

8. THRESHOLD: How is the minimum trust calculated?
   threshold = log(1 + stake) × difficulty

9. PLAUSIBILITY: What makes a history plausible?
   plausible if small OR outcome_variance ≥ minimum

10. ESCROW: How is stake verified?
    valid if commitment_verifies AND locked AND owned_by_party

11. EVOLUTION: How does trust change over time?
    new_trust = old_trust + (ω × outcome × γ × ν)

12. AGGREGATION: How do DAOs measure trust?
    DAO_trust = Φ(member trust values)

13. SECURITY: Why don't Sybil attacks work?
    Four mechanisms: escrow cost, counterparty weighting, 
    variance requirements, velocity limits

14. VALIDATION: How do we know it works?
    Correlation(computed trust, actual reliability) → 1
```

---

## Key Takeaways

1. **Trust is a number**: Not a binary yes/no, but a continuous value that can be positive, zero, or negative.

2. **Trust is earned through action**: Every contract adds to or subtracts from your score based on how well you performed.

3. **Context matters**: The contribution of a contract depends on stakes, difficulty, recency, who you worked with, and how fast you're accumulating.

4. **Trust unlocks access**: Higher trust = eligibility for better contracts = opportunity to build more trust (virtuous cycle).

5. **DAOs aggregate trust**: Groups can combine individual trust values into collective capability measures.

6. **Gaming is expensive**: Four mechanisms make Sybil attacks economically irrational—you're better off being honest.

7. **The system is verifiable**: We can mathematically prove that trust scores converge to actual reliability.

8. **Defense in depth**: No single mechanism stops all attacks; the combination creates robust resistance.

---

## Parameters Reference

| Parameter | Symbol | Suggested Default | Purpose |
|-----------|--------|-------------------|---------|
| Counterparty scale | λ | 50 | Centers sigmoid around trust=50 |
| Velocity period | T | 7 days | Time window for velocity checks |
| Velocity allowance | N | 10 | Full-weight contracts per period |
| Velocity decay | k | 0.5 | Decay rate for excess contracts |
| Variance threshold | ε_min | 0.1 | Minimum required outcome variance |
| Variance exemption | N_min | 10 | History size below which variance not checked |

---

*This document accompanies the Quantum of Trust whitepaper and Noir implementation.*
