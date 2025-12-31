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
4. [Provider Trust Value Calculation](#4-provider-trust-value-calculation)
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
15. [Customer Trust Value Calculation](#15-customer-trust-value-calculation)
16. [Customer Skill Types](#16-customer-skill-types)
17. [Verification Weight](#17-verification-weight)
18. [Task Decomposition](#18-task-decomposition)

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

**Note on Roles**: An Avatar can participate in contracts as either a provider (doing work) or a consumer (commissioning work). The same recursive structure applies to both roles—each accumulates trust in their respective skill types.

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

## 4. Provider Trust Value Calculation

### The Math

$$V_t^{\text{provider}}(\text{Agent}(t, h_t)) = \sum_{c \in h_t} \omega(c) \cdot \text{outcome}(c) \cdot \gamma(c) \cdot \nu(c)$$

### Plain English Translation

> "A Provider's trust value equals the sum of: (each contract's weight) times (that contract's outcome) times (counterparty trust factor) times (outcome variance factor), added up across all contracts in their history where they were the provider."

### Expanded Explanation

Breaking this down piece by piece:

- $V_t^{\text{provider}}(\text{Agent}(t, h_t))$ — "The provider trust value of an Agent who has skill type t and history h"
- $\sum_{c \in h_t}$ — "Add up the following for every contract c in the history h"
- $\omega(c)$ — "The base weight of that contract" (how much it should count)
- $\text{outcome}(c)$ — "How well the contract went" (success or failure, as rated by the customer)
- $\gamma(c)$ — "Counterparty trust factor" (contracts with high-trust customers count more)
- $\nu(c)$ — "Outcome variance factor" (ratings from discriminating customers count more)
- $\cdot$ — "multiplied by"

So for each contract:
- A successful contract with high weight adds a lot to your trust
- A failed contract with high weight subtracts a lot from your trust
- Low-weight contracts (small stakes, easy tasks) matter less either way
- Contracts with credible customers contribute more signal

**Example**: Imagine an Avatar with 3 contracts:
1. Big project, succeeded, trusted customer: weight=10, outcome=+1, γ=1.2, ν=1.0 → contributes +12
2. Medium project, partial success, new customer: weight=5, outcome=+0.5, γ=0.8, ν=1.0 → contributes +2
3. Small project, failed, trusted customer: weight=2, outcome=-1, γ=1.2, ν=1.0 → contributes -2.4

Total trust = 12 + 2 + (-2.4) = **11.6 cutes**

**Key Insight**: Provider trust comes from *outcomes assigned by customers*. The customer rates the work; that rating flows into the provider's trust score.

---

## 5. Contract Structure

### The Math

$$c = (a_{\text{provider}}, a_{\text{consumer}}, t, s, d, \tau, \varepsilon, V_{\text{consumer}})$$

### Plain English Translation

> "A contract is defined by eight things: who's providing the service, who's consuming it, what skill type it involves, how much is at stake, how difficult it is, its deadline, the escrow commitment, and the consumer's trust at contract time."

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
| $\varepsilon$ | Escrow commitment | 1000 tokens locked |
| $V_{\text{consumer}}$ | Consumer's trust at contract time | 45.2 cutes |

**Why these eight?** Because they determine how much a contract should "count" toward trust:
- Higher stakes → more signal
- Higher difficulty → more signal
- Who you worked with matters (counterparty trust captured in $V_{\text{consumer}}$)
- When it happened matters (recency)
- Escrow ensures skin in the game

**Bidirectional Trust**: Note that both provider and consumer are Avatars with their own trust values. The consumer's trust ($V_{\text{consumer}}$) is captured at contract creation and influences the weight of the provider's outcome.

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

**Who assigns the outcome?** For provider trust, the *customer* assigns the outcome based on their assessment of the delivered work. This is why customer credibility matters (see Verification Weight, equation #17).

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

**Note**: The weighting function applies symmetrically. When computing customer trust, the provider's trust becomes the counterparty trust factor.

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

**How is difficulty determined?** The provider assesses difficulty at the task level when accepting a contract. Tasks come from the Planning phase (customer's responsibility). The provider reviews each task and assigns a difficulty rating (0-10) based on their domain expertise. Phase and contract difficulty aggregate from task difficulties via stake-weighted average:

$$d_{phase} = \frac{\sum_{i} d_{task_i} \cdot s_{task_i}}{\sum_{i} s_{task_i}}$$

The provider can request task refinement before accepting if tasks are too vague to assess confidently. Both parties have incentive for accurate ratings—incorrect difficulty leads to failed tasks, which affects the trust of both provider and customer. See **Section 19: Difficulty Aggregation** for the full equation treatment, and [The Difficulty of Assessing Difficulty](./The_Difficulty_of_Assessing_Difficulty.md) for the complete analysis.

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

**Note**: History accumulates for both roles. An Avatar who acts as both provider and consumer has separate histories for each role in each skill type.

---

## 11. Trust Evolution

### The Math

$$V_t^{(n+1)}(a) = V_t^{(n)}(a) + \omega(c_n) \cdot \text{outcome}(c_n) \cdot \gamma(c_n) \cdot \nu(c_n)$$

### Plain English Translation

> "Your trust value after completing a contract equals your previous trust value plus (the contract's weight times its outcome times the counterparty factor times the variance factor)."

### Expanded Explanation

This is the **trust update rule**—how trust changes after each contract:

- $V_t^{(n)}(a)$ — Your trust before this contract
- $\omega(c_n) \cdot \text{outcome}(c_n) \cdot \gamma(c_n) \cdot \nu(c_n)$ — The contribution (positive or negative) from this contract
- $V_t^{(n+1)}(a)$ — Your trust after this contract

**Key insight**: Trust changes incrementally. Each contract moves your score up or down based on:
- How important the contract was (weight)
- How well you did (outcome)
- How credible the counterparty is (γ)
- How discriminating their ratings are (ν)

**Examples**:
- Succeed at a big contract (+8 weight × +1 outcome × 1.2 γ × 1.0 ν) → trust increases by 9.6
- Fail at a small contract (+2 weight × -1 outcome × 1.0 γ × 1.0 ν) → trust decreases by 2
- Partial success on medium contract (+5 weight × +0.6 outcome × 0.9 γ × 1.0 ν) → trust increases by 2.7

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

**Additional Sybil defenses**:
- Economic escrow requirements
- Counterparty diversity requirements
- Temporal depth requirements (history must span time, not just volume)
- Customer trust as a filter (bad customers attract bad providers, creating a self-isolating subnetwork)

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

## 15. Customer Trust Value Calculation

### The Math

$$V_t^{\text{customer}}(\text{Consumer}) = \sum_{c \in h_c} \omega(c) \cdot \text{behavior}(c) \cdot \gamma(c) \cdot \nu(c)$$

### Plain English Translation

> "A Customer's trust value equals the sum of: (each contract's weight) times (their behavior score on that contract) times (counterparty trust factor) times (variance factor), added up across all contracts where they were the consumer."

### Expanded Explanation

This is the **bidirectional trust** equation—customers earn trust too, just from different signals.

**Key difference from Provider Trust**:
- **Provider trust** comes from *outcomes assigned by customers* (did the work succeed?)
- **Customer trust** comes from *behaviors observed in contracts* (did they act professionally?)

The behavior score $\text{behavior}(c)$ is computed from observable contract metadata:
- Did they complete the project or abandon it?
- Did they fund escrow on time?
- Did they rate fairly (with reasonable variance)?
- Did their specifications lead to successful implementations?

**Why this matters**: A provider evaluating a contract offer needs to know:
- Will this customer see the project through?
- Will they pay on time?
- Will they rate fairly?

Customer trust answers these questions with data, not guesswork.

**Example**: A customer who:
- Completes 18 of 19 projects (94.7% commitment)
- Always funds escrow on time (100% reliability)
- Has rating variance of 0.34 (discriminating, not rubber-stamping)

...would have high customer trust, signaling to providers that this is a good contract to accept.

**The Symmetry**: Just as customers use provider trust to filter who can bid, providers can use customer trust to decide which contracts to pursue.

---

## 16. Customer Skill Types

### The Math

$$t_{\text{customer}} \in \{t_{\text{spec}}, t_{\text{commit}}, t_{\text{verify}}, t_{\text{escrow}}, t_{\text{timeline}}, t_{\text{scope}}\}$$

### Plain English Translation

> "Customer skill types include: specification quality, project commitment, verification integrity, escrow discipline, timeline realism, and scope stability."

### Expanded Explanation

Customers have measurable skills too—behaviors that can be tracked across their contract history:

| Skill Type | Symbol | What It Measures | Computed From |
|------------|--------|------------------|---------------|
| **Specification Quality** | $t_{\text{spec}}$ | Do their accepted specs lead to successful implementations? | Outcomes of impl contracts based on specs they approved |
| **Project Commitment** | $t_{\text{commit}}$ | Do they see projects through? | completed_projects / initiated_projects |
| **Verification Integrity** | $t_{\text{verify}}$ | Are their ratings fair and discriminating? | Rating variance + dispute rate |
| **Escrow Discipline** | $t_{\text{escrow}}$ | Do they fund and release escrow promptly? | On-time funding rate + release latency |
| **Timeline Realism** | $t_{\text{timeline}}$ | Do they set achievable deadlines? | Deadline accuracy across their projects |
| **Scope Stability** | $t_{\text{scope}}$ | Do they stick to agreed specifications? | tasks_as_planned / total_tasks (from provider view) |

**Computation Examples**:

**Commitment Trust**:
$$V_{\text{commit}}(\text{customer}) = \frac{\text{completed\_projects}}{\text{initiated\_projects}}$$

A customer who starts 20 projects and completes 18 has 90% commitment trust.

**Verification Integrity**:
$$V_{\text{verify}}(\text{customer}) = f(\sigma^2_{\text{ratings}}, \text{dispute\_rate})$$

A customer who always gives +1.0 ratings has zero variance—they're rubber-stamping, not evaluating. Ideal variance is around 0.3-0.5, indicating discriminating judgment.

**Why this matters**: These signals help providers make informed decisions:
- Low commitment trust → Provider might get abandoned mid-project
- Low timeline realism → Provider should pad estimates by 20-30%
- Low scope stability → Provider should price in scope creep

---

## 17. Verification Weight

### The Math

$$\text{verification\_weight}(c) = f\big(V_{\text{verify}}(\text{consumer}), \sigma^2(\text{ratings})\big)$$

### Plain English Translation

> "The weight given to a customer's rating depends on their verification integrity trust and the variance of their historical ratings."

### Expanded Explanation

Not all ratings are created equal. A rating from a credible, discriminating customer should count more than one from an unknown or rubber-stamping customer.

**The Problem**: 
- A customer who always rates +1.0 isn't providing signal—they approve everything
- A customer who always rates -1.0 might be gaming or adversarial
- A new customer with no history has unknown credibility

**The Solution**: Weight ratings by the rater's credibility.

**Verification Weight Factors**:

1. **Verification Integrity Trust** ($V_{\text{verify}}$): Does this customer have a track record of fair, reasonable ratings?

2. **Rating Variance** ($\sigma^2$): Does this customer discriminate between good and bad work?

| Variance | Interpretation | Weight Adjustment |
|----------|----------------|-------------------|
| $\sigma^2 < 0.1$ | Rubber-stamping (always same rating) | Reduce weight |
| $0.1 < \sigma^2 < 0.6$ | Discriminating judgment | Full weight |
| $\sigma^2 > 0.8$ | Erratic or inconsistent | Reduce weight |

**Example**:
- Trusted customer with good variance rates you +0.8 → counts as +0.8 × 1.2 = +0.96
- Unknown customer rates you +0.8 → counts as +0.8 × 0.7 = +0.56
- Rubber-stamper rates you +1.0 → counts as +1.0 × 0.5 = +0.50

**Why this matters**: It prevents trust inflation. You can't boost your reputation by finding customers who will give you perfect scores for mediocre work—those perfect scores will be discounted.

---

## 18. Task Decomposition

### The Math

$$s_{\text{task}} = s_{\text{phase}} \cdot \frac{w_{\text{task}}}{\sum_{i} w_{\text{task}_i}}$$

### Plain English Translation

> "A task's stake equals the phase stake multiplied by the task's weight divided by the total weight of all tasks in that phase."

### Expanded Explanation

This equation formalizes how value flows down through the contract hierarchy:

```
Project (total stake)
└── Phase (phase stake = portion of project)
    └── Task (task stake = portion of phase)
```

**The Recursive Pattern**: Tasks are just contracts. They follow the same structure:

$$\text{Task} = (a_{\text{provider}}, a_{\text{consumer}}, t, s_{\text{task}}, d, \tau, \text{parent\_ref})$$

The only addition is `parent_ref`—a link to the parent phase contract.

**Stake Allocation Example**:

```
Implementation Phase: 3000 sats total
├── M4.P1 (6 tasks, 4:25 duration) → 750 sats allocated
│   ├── M4.P1.T1 (00:11, weight 11) →  31 sats
│   ├── M4.P1.T2 (00:12, weight 12) →  34 sats
│   └── ... (remaining tasks proportionally)
├── M4.P2 (8 tasks, 5:29 duration) → 935 sats allocated
│   ├── M4.P2.T1 (04:59, weight 299) → 523 sats
│   └── ... (remaining tasks proportionally)
└── ...
```

**Weight Assignment Options**:
- **Time-based**: Task weight = duration (as shown above)
- **Complexity-based**: Task weight = estimated difficulty
- **Equal**: All tasks have weight = 1
- **Explicit**: Specification assigns weights directly

**Trust Contribution from Tasks**:

Each completed task contributes to:

1. **Provider skill trust** (micro-contributions from task outcomes)
2. **Planning accuracy** (tasks_as_planned / total_tasks)
3. **Phase outcome** (aggregate of task outcomes determines phase success)

**The Math Doesn't Change**: A task is a contract. The trust equation applies:

$$V_t^{(n+1)}(a) = V_t^{(n)}(a) + \omega(c_{\text{task}}) \cdot \text{outcome}(c_{\text{task}}) \cdot \gamma(c) \cdot \nu(c)$$

The weight will be small (small stake), so task-level contributions are proportionally small. But they accumulate, and they enable granular tracking of where projects succeed or fail.

**Why this matters**:
- **Granular accountability**: If task M4.P2.T3 fails, the specific failure point is identified
- **Partial credit**: 38 of 41 tasks completed = 92.7% completion, not binary success/failure
- **Planning accuracy**: Precise measurement of estimation quality

---

## 19. Difficulty Aggregation

### The Math

$$d_{\text{phase}} = \frac{\sum_{i} d_{\text{task}_i} \cdot s_{\text{task}_i}}{\sum_{i} s_{\text{task}_i}}$$

### Plain English Translation

> "A phase's difficulty rating equals the stake-weighted average of its tasks' difficulty ratings."

### Expanded Explanation

This equation formalizes how difficulty flows up through the contract hierarchy:

```
Task difficulties (assessed by provider)
    ↓ aggregate via stake-weighted average
Phase difficulty
    ↓ aggregate via stake-weighted average  
Project difficulty
```

**Why stake-weighted?** High-stake tasks contribute more to the aggregate because they represent more of the work's value. A difficult but tiny task shouldn't dominate the phase difficulty.

**Example Calculation**:

| Task | Stake | Difficulty | Contribution |
|------|-------|------------|--------------|
| T1: Database schema | 500 sats | 3 | 500 × 3 = 1,500 |
| T2: Password hashing | 750 sats | 4 | 750 × 4 = 3,000 |
| T3: Session mgmt | 1,000 sats | 5 | 1,000 × 5 = 5,000 |
| T4: OAuth2 integration | 1,250 sats | 7 | 1,250 × 7 = 8,750 |
| T5: Rate limiting | 750 sats | 4 | 750 × 4 = 3,000 |
| T6: Security audit | 750 sats | 6 | 750 × 6 = 4,500 |
| **Total** | **5,000 sats** | | **25,750** |

$$d_{\text{phase}} = \frac{25,750}{5,000} = 5.15$$

**Who Assesses Difficulty?** The provider (domain expert) assesses difficulty at the task level when accepting a contract. Tasks come from the Planning phase (customer's responsibility). The provider reviews each task and assigns a difficulty rating (0-10) based on their expertise.

**Why Not Customer-Assigned?** Assessing difficulty requires domain expertise—that's why the customer is hiring a provider. A customer who knew how hard the work was wouldn't need to hire someone.

**Bidirectional Incentive**: Both parties want accurate difficulty ratings:
- **Provider**: Underestimating difficulty → struggles → poor outcome → trust decreases
- **Customer**: Project fails or drags → commitment trust decreases

The outcome reveals whether difficulty was assessed correctly. Trust consequences follow.

**The Recursive Pattern**: Just as tasks aggregate to phases, phases aggregate to projects:

$$d_{\text{project}} = \frac{\sum_{j} d_{\text{phase}_j} \cdot s_{\text{phase}_j}}{\sum_{j} s_{\text{phase}_j}}$$

**Why This Matters for Eligibility**: The threshold function uses difficulty:

$$\theta(c) = \log(1 + s) \cdot d$$

Accurate difficulty assessment ensures the right providers are eligible. Too low = unqualified providers bid. Too high = qualified providers excluded.

See [The Difficulty of Assessing Difficulty](./The_Difficulty_of_Assessing_Difficulty.md) for the complete analysis of the information asymmetry problem and why task-level assessment by the provider is the solution.

---

## Summary: The Complete Picture

Here's how all the equations fit together:

```
FOUNDATION (Equations 1-3)
├── q⟨T⟩ = Agent(skill, history) OR DAO({members})
├── V_t: q⟨T⟩ → ℝ (produces a real number)
└── V_t meanings: 0 = unknown, >0 = trusted, <0 = distrusted

PROVIDER TRUST (Equations 4, 6-7, 11)
├── V_provider = Σ(weight × outcome × γ × ν) for each contract
├── outcome ∈ [-1, 1] (assigned by customer)
├── weight = f(stake, difficulty, counterparty trust, recency)
└── Trust evolves: new = old + (weight × outcome × γ × ν)

CUSTOMER TRUST (Equations 15-17) ← NEW
├── V_customer = Σ(weight × behavior × γ × ν) for each contract
├── Customer skills: spec quality, commitment, verification, escrow, timeline, scope
└── Verification weight: credible customers' ratings count more

CONTRACT STRUCTURE (Equations 5, 18-19)
├── c = (provider, consumer, skill, stake, difficulty, deadline, escrow, V_consumer)
├── Tasks inherit structure: task stake = phase stake × (task weight / total weight)
└── Difficulty aggregates: phase difficulty = stake-weighted average of task difficulties

GATEKEEPING (Equations 8-9)
├── eligible if V_t ≥ threshold
└── threshold = log(1 + stake) × difficulty

AGGREGATION (Equation 12)
└── DAO_trust = Φ(member trust values)

SECURITY (Equation 13)
└── Honest history size > any individual sybil's history size

VALIDATION (Equation 14)
└── Correlation(computed trust, actual reliability) → 1
```

---

## Key Takeaways

1. **Trust is bidirectional**: Both providers and customers earn trust. Providers from outcomes; customers from behaviors.

2. **Trust is a number**: Not a binary yes/no, but a continuous value that can be positive, zero, or negative.

3. **Trust is earned through action**: Every contract adds to or subtracts from your score based on performance (providers) or behavior (customers).

4. **Context matters**: The weight of a contract depends on stakes, difficulty, who you worked with, and recency.

5. **Credibility matters**: Ratings from discriminating, trustworthy customers count more than ratings from unknown or rubber-stamping customers.

6. **Trust unlocks access**: Higher trust = eligibility for better contracts = opportunity to build more trust (virtuous cycle).

7. **Tasks decompose recursively**: Projects contain phases contain tasks. Each is a contract. The math doesn't change—it just applies at different granularities.

8. **DAOs aggregate trust**: Groups can combine individual trust values into collective capability measures.

9. **Gaming is expensive**: The math makes Sybil attacks economically irrational—you're better off being honest.

10. **The system is verifiable**: We can mathematically prove that trust scores converge to actual reliability.

---

*This document accompanies the Quantum of Trust whitepaper and Noir implementation.*
