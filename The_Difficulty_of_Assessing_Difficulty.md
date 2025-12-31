# The Difficulty of Assessing Difficulty

## Problem Statement

The Quantum of Trust threshold function includes a difficulty multiplier:

$$\theta(c) = \log(1 + s) \cdot d$$

Where `d` is the difficulty rating (0-10). Harder contracts require proportionally more trust to bid on.

**The fundamental problem**: Accurately assessing task difficulty requires expertise in the task domain. The customer (who sets contract parameters) typically lacks this expertise—that's why they're hiring a provider. This creates an information asymmetry that undermines the threshold function's purpose.

| Party | Has Domain Expertise? | Incentive |
|-------|----------------------|-----------|
| Customer | Usually no | Wants qualified providers, but can't assess what "qualified" means |
| Provider | Yes | May inflate difficulty to boost trust contribution |

If difficulty is set incorrectly:
- **Too low**: Unqualified providers can bid; customer gets poor results
- **Too high**: Qualified providers excluded; contract may go unfilled

**Empirical observation**: Failed tasks or tasks requiring more time and effort than planned are frequently due to incorrect difficulty ratings. Getting difficulty right is essential for project success.

---

## The Solution: Task-Level Difficulty

### Core Insight

Difficulty should be assessed **at the task level**, not the contract level. The contract's difficulty is simply an **aggregation** of the difficulties of its constituent tasks.

Since tasks are already defined as sub-subcontracts in the QoT framework, the math already works. We don't need new primitives—just proper application of existing ones.

### Why Task-Level Works

| Level | Who Assesses | Accuracy | Granularity |
|-------|--------------|----------|-------------|
| Contract | Customer (non-expert) | Poor | Coarse—misses variation |
| Phase | Mixed | Medium | Better, but still abstract |
| **Task** | Provider (expert) | **High** | **Concrete, specific, verifiable** |

Tasks are concrete:
- "Implement OAuth2 flow" is assessable
- "Build authentication system" is vague
- "Build the app" is impossible to estimate accurately

### The Aggregation Pattern

Contract difficulty emerges from task difficulties using weighted average by stake:

$$d_{phase} = \frac{\sum_{i} d_{task_i} \cdot s_{task_i}}{\sum_{i} s_{task_i}}$$

This means high-stake tasks contribute more to the overall difficulty rating than low-stake tasks. The pattern matches the existing stake allocation formula, maintaining mathematical consistency across the framework.

---

## When Difficulty Is Negotiated

### The Subcontract Architecture

QoT uses a recursive contract structure (see ADR: Subcontract Architecture):

```
Project
└── Phase (Subcontract) ← Full contract, linked to project
    └── Task (Sub-Subcontract) ← Full contract, linked to phase
```

Each level is a **full contract in its own right**. The trust equation applies identically at every level:

$$V_t = \sum_{c \in h_t} \omega(c) \cdot \text{outcome}(c) \cdot \gamma(c) \cdot \nu(c)$$

### The Key Moment: Provider Acceptance

When a provider **accepts** a contract, they commit to the work. At this point—before work begins—there is an opportunity to discuss task-level difficulty with the customer.

```
┌─────────────────────────────────────────────────────────────────┐
│  PROVIDER ACCEPTS CONTRACT                                      │
│                                                                 │
│  At acceptance, provider and customer negotiate:                │
│  • Review task breakdown (from Planning phase)                  │
│  • Difficulty rating for each task (provider assesses)          │
│  • Request task refinement if needed (provider's right)         │
│  • The phase/contract difficulty emerges from task aggregation  │
└─────────────────────────────────────────────────────────────────┘
```

### Bidirectional Consequences

The difficulty ratings that are mutually agreed upon influence the **future trust values of both parties**. A failed task or contract affects everyone:

| Party | If Difficulty Is Wrong | Trust Consequence |
|-------|------------------------|-------------------|
| **Provider** | Struggles with underestimated work | Outcome suffers → provider trust decreases |
| **Customer** | Project fails or drags | Commitment trust decreases; scope stability may suffer |

This creates **mutual incentive for accuracy**:
- The provider has expertise and proposes realistic difficulty
- The customer validates and accepts only what makes sense
- Both sign off knowing their future trust depends on getting it right

**Best to get it right now.** Neither party benefits from optimistic or inflated difficulty ratings—the outcome will reveal the truth, and trust consequences follow.

### Why Acceptance Is the Right Moment

| Timing | Who Has Information | Result |
|--------|---------------------|--------|
| Before acceptance | Customer defines scope; provider evaluates | Provider can decline if scope is unreasonable |
| **At acceptance** | **Both parties engaged; provider commits** | **Negotiation on task details and difficulty** |
| After acceptance | Work begins | Too late to negotiate—difficulty is locked |

---

## The Provider's Role

The provider is the **domain expert** for implementation. But the task breakdown comes from the **Planning phase**, which is the customer's responsibility (possibly via a hired planner through a subcontract).

### The Phase Sequence

```
Specification Phase (Customer's responsibility)
    ↓
Planning Phase (Customer's responsibility)
    → Produces task breakdown
    ↓
Implementation Phase (Provider executes)
    → Provider reviews existing tasks at acceptance
```

By the time an Implementation provider considers acceptance, **tasks should already exist** from the Planning phase.

### Provider Reviews and Validates Task Breakdown

At acceptance, the provider reviews the planned tasks and assesses difficulty:

```
Customer's Task Breakdown (from Planning phase):
┌────────────────────────────────────────────────────────┐
│ Task (Sub-Subcontract)        │ Stake %  │ Difficulty │
├────────────────────────────────────────────────────────┤
│ T1: Database schema design    │   10%    │     ?      │
│ T2: Password hashing impl     │   15%    │     ?      │
│ T3: Session management        │   20%    │     ?      │
│ T4: OAuth2 integration        │   25%    │     ?      │
│ T5: Rate limiting             │   15%    │     ?      │
│ T6: Security audit            │   15%    │     ?      │
└────────────────────────────────────────────────────────┘

Provider assesses difficulty for each task:
┌────────────────────────────────────────────────────────┐
│ Task (Sub-Subcontract)        │ Stake %  │ Difficulty │
├────────────────────────────────────────────────────────┤
│ T1: Database schema design    │   10%    │     3      │
│ T2: Password hashing impl     │   15%    │     4      │
│ T3: Session management        │   20%    │     5      │
│ T4: OAuth2 integration        │   25%    │     7      │
│ T5: Rate limiting             │   15%    │     4      │
│ T6: Security audit            │   15%    │     6      │
├────────────────────────────────────────────────────────┤
│ PHASE DIFFICULTY (aggregate)  │  100%    │    5.05    │
└────────────────────────────────────────────────────────┘
```

### Provider Can Request Task Refinement

The provider should be allowed to identify tasks that need **further breakdown** before they'll commit. If a task is too vague or too large, the provider cannot confidently assess difficulty or ensure successful completion.

```
Provider: "T4: OAuth2 integration is too broad. I need this broken 
           into subtasks before I can accept:
           - T4a: OAuth2 flow for Google
           - T4b: OAuth2 flow for GitHub  
           - T4c: Token refresh handling
           - T4d: Error handling and edge cases"

Customer: Either refines the planning, or negotiates as-is.
```

This is the provider exercising expertise to **validate the plan's executability**—not rewriting the plan.

### Customer's Role: Owns Specification and Planning

The customer is responsible for the **Specification** and **Planning** phases. They may hire specialists via subcontracts to help, but the accountability sits with them.

| Phase | Customer's Responsibility | May Hire Help? |
|-------|---------------------------|----------------|
| Specification | Define what is needed | Yes (Requirements specialist) |
| Planning | Break work into tasks, allocate stakes | Yes (Architect, PM) |
| Implementation | Select provider, validate difficulty | Provider executes |

By the time an Implementation provider reviews the contract:
- Tasks are already defined (from Planning)
- Stakes are already allocated (customer's value assessment)
- Difficulty is **not yet set**—this is what the provider brings

### Customer Validates, Doesn't Estimate

The customer doesn't need domain expertise to participate in difficulty negotiation:

| Customer Can | Customer Cannot |
|--------------|-----------------|
| Ask "why is T4 rated 7?" | Accurately assess T4 themselves |
| Compare to similar past contracts | Know OAuth2 implementation details |
| Challenge outliers that seem extreme | Determine correct difficulty |
| Accept or reject the provider's assessment | Force provider to accept lower ratings |
| Walk away if estimates seem inflated | Substitute their judgment for provider's |

The customer's leverage is **accepting or declining**—not counter-proposing difficulty ratings they aren't qualified to assess.

---

## Handling Disagreement

### Scenario: Provider Rates Task Too High

```
Provider: "OAuth2 integration is difficulty 9"
Customer: "My last provider did similar work at difficulty 5"
```

**Resolution options**:

1. **Discussion**: Provider explains hidden complexity (custom claims, multiple IdPs, etc.)
2. **Scope clarification**: Maybe customer's "similar work" was actually simpler
3. **Compromise**: Agree on 7 with documented assumptions
4. **Walk away**: Provider declines the contract; customer finds different provider

### Scenario: Provider Rates Task Too Low

```
Provider: "Full authentication system, difficulty 2"
Customer: "That seems too easy for this scope..."
```

This is rarer—providers usually don't want to undervalue their work. But if it happens:

1. **Clarification**: Provider may be very experienced; difficulty is relative to skill
2. **Scope check**: Maybe provider is planning to cut corners
3. **Accept with monitoring**: Proceed but watch for quality issues

### The Outcome Corrects

Over time, providers who systematically mis-rate difficulty will show patterns in their **outcomes**:

| Pattern | Outcome Signal | Trust Consequence |
|---------|----------------|-------------------|
| Underestimates, struggles | Lower outcomes, delays | Trust decreases |
| Overestimates, delivers easily | High outcomes | Trust increases, but customer learns to push back |
| Accurate estimates | Outcomes match expectations | Sustainable trust growth |

Failed tasks or tasks requiring more effort than planned are due to incorrect difficulty ratings. The outcome reflects this—and flows into trust.

---

## Stake vs. Difficulty: Two Distinct Signals

### Stake: Customer's Value Assessment

The customer has already assigned value to each task through **stake allocation**. This is a potent signal—it reflects what the customer believes each task is worth to the project.

$$s_{task} = s_{phase} \cdot \frac{w_{task}}{\sum_{i} w_{task_i}}$$

But stake does not speak to difficulty. A task can be:
- High value, easy to complete
- High value, hard to complete
- Low value, easy to complete
- Low value, hard to complete

### Difficulty: Provider's Effort Assessment

Difficulty is the provider's assessment of **how hard the task is to complete**. This requires domain expertise the customer typically lacks.

| Task | Stake (Customer) | Difficulty (Provider) | Interpretation |
|------|------------------|----------------------|-----------------|
| T1 | 500 sats | d=3 | High value, easy → good rate for provider |
| T2 | 500 sats | d=8 | High value, hard → fair rate |
| T3 | 100 sats | d=8 | Low value, hard → provider may decline |
| T4 | 100 sats | d=2 | Low value, easy → fair rate |

### The Threshold Captures Both

The threshold function combines both signals:

$$\theta(c) = \log(1 + s) \cdot d$$

- **Stake** (from customer): How much value is at risk
- **Difficulty** (from provider): How hard the work is

A task with high stake and high difficulty requires significant trust to bid on. A task with low stake and low difficulty is accessible to newcomers.

### Misalignment Reveals Problems

When stake and difficulty don't align, it surfaces issues:

| Misalignment | What It Reveals | Resolution |
|--------------|-----------------|------------|
| High stake, low difficulty | Customer overvalues the task | Provider accepts (good deal) |
| Low stake, high difficulty | Customer undervalues the task | Provider declines or negotiates |
| Stakes don't match task complexity | Planning phase was inadequate | Refine task breakdown |

This negotiation is healthy—it aligns customer expectations with implementation reality before work begins.

---

## Integration with Existing Math

The existing formulas already accommodate difficulty—no changes required.

### Existing: Contract Structure

$$c = (a_{provider}, a_{consumer}, t, s, d, \tau, \varepsilon, V_{consumer})$$

The `d` parameter is already defined. This document specifies **how it gets its value**.

### Existing: Threshold Function

$$\theta(c) = \log(1 + s) \cdot d$$

Unchanged. Uses whatever `d` is assigned to the contract.

### Existing: Weight Function

$$\omega(c) = f(s, d, V_t, recency)$$

Unchanged. Uses whatever `d` is assigned to the contract.

### Aggregation Pattern (Follows Existing Stake Allocation)

Difficulty aggregates from tasks to phases using the same pattern as stake allocation:

$$d_{phase} = \frac{\sum_{i} d_{task_i} \cdot s_{task_i}}{\sum_{i} s_{task_i}}$$

This is not a new formula—it applies the existing weighted-average pattern to difficulty.

---

## Why This Mirrors Reality

### The Empirical Observation

> **Failed tasks or tasks that require more time and effort than planned are due to incorrect difficulty ratings.**

This is not theoretical—it's observed from direct experience. When difficulty is wrong:
- Providers struggle with work rated too easy
- Outcomes suffer when complexity is underestimated  
- Trust decreases when estimates don't match reality

Getting difficulty right is essential for project success. Task-level assessment by the expert (provider) maximizes accuracy.

### Real-World Project Estimation

In practice, accurate project estimates come from:

1. **Decomposition**: Break work into small pieces
2. **Expert assessment**: People who do the work estimate the pieces
3. **Aggregation**: Sum or average the pieces
4. **Buffer**: Add contingency for unknowns

QoT's task-level difficulty follows exactly this pattern.

### The Planning Fallacy

Humans systematically underestimate task duration and difficulty at high levels of abstraction. The cure is granular decomposition. QoT enforces this structurally.

### QoT's Approach

QoT enforces accurate estimation through structure:
- **Decomposition**: Tasks are sub-subcontracts with concrete scope
- **Expert assessment**: Providers estimate at the task level
- **Aggregation**: Phase/project difficulty emerges from task difficulties
- **Outcome feedback**: Trust consequences correct poor estimation over time

---

## Tracking Estimation Accuracy

### Post-Hoc Difficulty Signal

After task completion, we can infer whether difficulty was accurate:

| Signal | Indicates |
|--------|-----------|
| Completed on time, good outcome | Difficulty was accurate or overestimated |
| Completed late, good outcome | Difficulty was underestimated |
| Failed despite adequate trust | Difficulty was significantly underestimated |
| Completed early, trivial effort | Difficulty was overestimated |

### Calibration as a Skill Type

Provider difficulty calibration could be tracked:

$$V_{calibration}(provider) = f(\text{predicted difficulty}, \text{actual signals})$$

This creates incentive for accurate estimation—providers known for good calibration become more attractive to customers.

### Customer Signal: Scope Stability

The customer's contribution to estimation accuracy is **scope stability**. Already tracked as `customer:scope`:

$$\text{scope stability} = \frac{\text{tasks as planned}}{\text{total tasks}}$$

A customer who constantly changes scope makes difficulty estimation impossible. Low scope stability is the customer's "fault" in estimation failures.

---

## Edge Cases

### Standalone Contracts (No Task Breakdown)

Some contracts may be too simple for task decomposition—a single deliverable.

**Solution**: Standalone contracts use direct difficulty assessment at the contract level. The provider proposes; customer validates at acceptance. This is backward compatible with the existing contract structure.

### Inadequate Task Breakdown from Planning

The Planning phase produced tasks that are too vague or too large.

**Solution**: Provider identifies problematic tasks at acceptance and requests refinement before committing. If customer refuses to refine, provider declines the contract. This is healthy—it surfaces planning inadequacy before work begins.

### Difficulty Changes Mid-Contract

Discovered complexity makes original estimates wrong.

**Solution**: Contract amendment process. New tasks added as new sub-subcontracts with their own difficulty ratings. Original tasks retain original ratings. Phase aggregate recalculates to include new tasks.

### Team-Based Implementation

Multiple providers work on different tasks within a phase.

**Solution**: Already handled by the subcontract architecture. Each task is a sub-subcontract with its own provider. Each provider proposes difficulty for their assigned tasks. Phase difficulty = weighted average across all tasks regardless of provider.

### AI Agents as Providers

AI-operated Avatars need to estimate difficulty.

**Solution**: AI agents can estimate based on:
- Historical performance on similar tasks
- Complexity metrics (lines of code, integration points, etc.)
- Explicit difficulty models trained on past contracts

AI may actually be *more* consistent than human estimators—and their calibration can be tracked just like human providers.

---

## Summary

### The Design

1. **Difficulty is assessed at the task level** by the provider (domain expert)
2. **Tasks are sub-subcontracts** defined in the Planning phase (customer's responsibility)
3. **Phase/contract difficulty aggregates** from task difficulties (stake-weighted average)
4. **Negotiation occurs at acceptance**—provider reviews tasks and assesses difficulty
5. **Provider can request task refinement** before committing
6. **Customer validates** but doesn't estimate—they lack the expertise
7. **Outcomes correct** systematic over/under-estimation through trust consequences

### Why This Works

| Requirement | How Task-Level Difficulty Satisfies It |
|-------------|----------------------------------------|
| Expert assessment | Provider estimates their own work at task level |
| Gaming resistance | Customer validation + outcome tracking |
| Privacy preservation | No identity disclosure required |
| Efficiency | Part of contract acceptance, not separate process |
| AI compatibility | Agents can estimate same as humans |
| Mathematical consistency | Aggregation follows existing stake allocation patterns |
| Backward compatibility | Standalone contracts work unchanged (ADR) |

### The Key Insight

**Failed tasks or tasks requiring more effort than planned are due to incorrect difficulty ratings.** By assessing difficulty at the task level—where the work is concrete and the provider is the expert—we maximize estimation accuracy. The outcome history proves whether estimates were correct.

---


*This document accompanies the Quantum of Trust framework development.*
