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
7. [The Weighting Function](#7-the-weighting-function)
8. [Eligibility Condition](#8-eligibility-condition)
9. [Threshold Function](#9-threshold-function)
10. [History Evolution](#10-history-evolution)
11. [Trust Evolution](#11-trust-evolution)
12. [DAO Valuation](#12-dao-valuation)
13. [Sybil Resistance](#13-sybil-resistance)
14. [Convergence Criterion](#14-convergence-criterion)

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

$$V_t(\text{Agent}(t, h_t)) = \sum_{c \in h_t} \omega(c) \cdot \text{outcome}(c)$$

### Plain English Translation

> "An Agent's trust value equals the sum of: (each contract's weight) times (that contract's outcome), added up across all contracts in their history."

### Expanded Explanation

Breaking this down piece by piece:

- $V_t(\text{Agent}(t, h_t))$ — "The trust value of an Agent who has skill type t and history h"
- $\sum_{c \in h_t}$ — "Add up the following for every contract c in the history h"
- $\omega(c)$ — "The weight of that contract" (how much it should count)
- $\text{outcome}(c)$ — "How well the contract went" (success or failure)
- $\cdot$ — "multiplied by"

So for each contract:
- A successful contract with high weight adds a lot to your trust
- A failed contract with high weight subtracts a lot from your trust
- Low-weight contracts (small stakes, easy tasks) matter less either way

**Example**: Imagine an Avatar with 3 contracts:
1. Big project, succeeded: weight=10, outcome=+1 → contributes +10
2. Medium project, partial success: weight=5, outcome=+0.5 → contributes +2.5
3. Small project, failed: weight=2, outcome=-1 → contributes -2

Total trust = 10 + 2.5 + (-2) = **10.5 cutes**

---

## 5. Contract Structure

### The Math

$$c = (a_{\text{provider}}, a_{\text{consumer}}, t, s, d, \tau)$$

### Plain English Translation

> "A contract is defined by six things: who's providing the service, who's consuming it, what skill type it involves, how much is at stake, how difficult it is, and its deadline."

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

**Why these six?** Because they determine how much a contract should "count" toward trust:
- Higher stakes → more signal
- Higher difficulty → more signal
- Who you worked with matters (counterparty trust)
- When it happened matters (recency)

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

## 7. The Weighting Function

### The Math

$$\omega(c) = f\big(s(c),\ d(c),\ V_t(a_{\text{consumer}}),\ \text{recency}(c)\big)$$

### Plain English Translation

> "The weight of a contract is a function of four things: the stake involved, the difficulty, the trust level of the counterparty who hired you, and how recent the contract is."

### Expanded Explanation

This function determines **how much a contract should count**. Four factors:

1. **Stake** $s(c)$: A $100,000 contract matters more than a $100 contract. If you can deliver on high-stakes work, that's meaningful signal.

2. **Difficulty** $d(c)$: Completing a hard task proves more than completing an easy one. Anyone can succeed at trivial work.

3. **Counterparty trust** $V_t(a_{\text{consumer}})$: A positive review from a trusted Avatar counts more than one from an unknown Avatar. This prevents trust laundering—you can't boost your reputation by having your friends (with no reputation) give you fake positive reviews.

4. **Recency**: Recent contracts matter more than old ones. Trust requires ongoing reinforcement. Coasting on old successes doesn't work forever.

**Real-world analogy**: It's like how a recommendation letter matters more if it's from someone respected, recent, for difficult work, and with real stakes.

---

## 8. Eligibility Condition

### The Math

$$\text{eligible}(a, c) \iff V_t(a) \geq \theta(c)$$

### Plain English Translation

> "An Avatar is eligible for a contract if and only if their trust value is greater than or equal to the contract's threshold requirement."

### Expanded Explanation

The $\iff$ symbol means "if and only if"—it's a two-way condition.

This is the **core gatekeeping mechanism**:

- Every contract has a minimum trust requirement ($\theta$)
- Only Avatars who meet or exceed that requirement can bid
- Higher-stakes, harder contracts have higher thresholds

**Why this matters**: It creates a natural progression. New Avatars can only access small, easy contracts. As they build trust, they unlock access to bigger opportunities. This is the "trust ladder."

**The key privacy innovation**: Zero-knowledge proofs let an Avatar prove "I meet the threshold" without revealing their exact score or any of their contract history.

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

$$V_t^{(n+1)}(a) = V_t^{(n)}(a) + \omega(c_n) \cdot \text{outcome}(c_n)$$

### Plain English Translation

> "Your trust value after completing a contract equals your previous trust value plus (the contract's weight times its outcome)."

### Expanded Explanation

This is the **trust update rule**—how trust changes after each contract:

- $V_t^{(n)}(a)$ — Your trust before this contract
- $\omega(c_n) \cdot \text{outcome}(c_n)$ — The contribution (positive or negative) from this contract
- $V_t^{(n+1)}(a)$ — Your trust after this contract

**Key insight**: Trust changes incrementally. Each contract moves your score up or down based on:
- How important the contract was (weight)
- How well you did (outcome)

**Examples**:
- Succeed at a big contract (+8 weight × +1 outcome) → trust increases by 8
- Fail at a small contract (+2 weight × -1 outcome) → trust decreases by 2
- Partial success on medium contract (+5 weight × +0.6 outcome) → trust increases by 3

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

$$|h_t(a_{\text{honest}})| > |h_t(a_{\text{sybil}_i})| \quad \forall i$$

### Plain English Translation

> "The size of an honest Avatar's history is greater than the size of any individual sybil Avatar's history, for all sybils."

### Expanded Explanation

The $|...|$ notation means "the size of" or "number of items in."
The $\forall$ symbol means "for all."

**What's a Sybil attack?** Creating multiple fake identities to game the system.

**Why Sybils fail in this framework**: If you split your activity across $k$ fake Avatars:

- Your time/resources are divided $k$ ways
- Each sybil gets roughly $1/k$ as many contracts as an honest Avatar would
- Smaller history = less trust = access to fewer contracts
- The economics favor consolidation, not fragmentation

**Example**: 
- Honest Alice completes 100 contracts → $V_t = 85$
- Sybil Bob splits across 5 fake Avatars, each completes 20 contracts → each has $V_t ≈ 17$

Alice qualifies for high-value contracts; none of Bob's sybils do. Bob would have been better off using one identity.

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

## Summary: The Complete Picture

Here's how all the equations fit together:

```
1. DEFINITION: What is a Quantum of Trust?
   q⟨T⟩ = Agent(skill, history) OR DAO({members})

2. MEASUREMENT: How do we measure trust?
   V_t: q⟨T⟩ → ℝ (produces a real number)
   
3. CALCULATION: How is an Agent's trust computed?
   V_t = Σ(weight × outcome) for each contract

4. WEIGHTING: What makes a contract count more?
   weight = f(stake, difficulty, counterparty trust, recency)

5. GATEKEEPING: Who can access what contracts?
   eligible if V_t ≥ threshold
   threshold = log(1 + stake) × difficulty

6. EVOLUTION: How does trust change over time?
   new_trust = old_trust + (weight × outcome)

7. AGGREGATION: How do DAOs measure trust?
   DAO_trust = Φ(member trust values)

8. SECURITY: Why don't Sybil attacks work?
   Honest history size > any individual sybil's history size

9. VALIDATION: How do we know it works?
   Correlation(computed trust, actual reliability) → 1
```

---

## Key Takeaways

1. **Trust is a number**: Not a binary yes/no, but a continuous value that can be positive, zero, or negative.

2. **Trust is earned through action**: Every contract adds to or subtracts from your score based on how well you performed.

3. **Context matters**: The weight of a contract depends on stakes, difficulty, who you worked with, and recency.

4. **Trust unlocks access**: Higher trust = eligibility for better contracts = opportunity to build more trust (virtuous cycle).

5. **DAOs aggregate trust**: Groups can combine individual trust values into collective capability measures.

6. **Gaming is expensive**: The math makes Sybil attacks economically irrational—you're better off being honest.

7. **The system is verifiable**: We can mathematically prove that trust scores converge to actual reliability.

---

*This document accompanies the Quantum of Trust whitepaper and Noir implementation.*
