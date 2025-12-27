# Quantum of Trust - Unit Tests

This document presents the comprehensive unit test suite for the Quantum of Trust framework.
Each test validates the C# implementation against the mathematical equations defined in the whitepaper.

At the time of writing the C# implementation doesn't exist! So this is a nice example of 'Test First' engineering.

## Test Philosophy

All tests follow the **Arrange-Act-Assert (AAA)** pattern:

| Phase | Description |
|-------|-------------|
| **ARRANGE** | Define all inputs to the equation (the domain) |
| **ACT** | Execute the function (apply f to inputs) |
| **ASSERT** | Verify output matches expected (validate codomain) |

---

## Table of Contents

1. [Agent Trust Value Calculation Tests](#1-agent-trust-value-calculation-tests)
2. [DAO Trust Aggregation Tests](#2-dao-trust-aggregation-tests)
3. [Aggregation Functions Tests](#3-aggregation-functions-tests)
4. [Outcome Function Tests](#4-outcome-function-tests)
5. [Weighting Function Tests](#5-weighting-function-tests)
6. [History Evolution Tests](#6-history-evolution-tests)
7. [Trust Evolution Tests](#7-trust-evolution-tests)
8. [Eligibility Function Tests](#8-eligibility-function-tests)
9. [Sybil Resistance Tests](#9-sybil-resistance-tests)
10. [Convergence Criterion Tests](#10-convergence-criterion-tests)
11. [Contract Validation Tests](#11-contract-validation-tests)
12. [SkillTypes Utility Tests](#12-skilltypes-utility-tests)
13. [TrustValuation Static Helper Tests](#13-trustvaluation-static-helper-tests)

---

## 1. Agent Trust Value Calculation Tests

Tests for the Agent trust valuation function.

**Mathematical Notation:**

```
┌─────────────────────────────────────────────────────────────────────
│  V⟨t⟩(Agent(t, h⟨t⟩)) = Σ ω(c) · outcome(c)   for all c ∈ h⟨t⟩       │
└─────────────────────────────────────────────────────────────────────┘
```

```csharp

/// <summary>
/// Tests for the Agent trust valuation function.
/// 
/// Mathematical notation:
/// ┌─────────────────────────────────────────────────────────────────────
/// │  V⟨t⟩(Agent(t, h⟨t⟩)) = Σ ω(c) · outcome(c)   for all c ∈ h⟨t⟩       │
/// └─────────────────────────────────────────────────────────────────────┘
/// 
/// Where:
///   V⟨t⟩     = Trust value for skill type t
///   h⟨t⟩     = History of contracts for skill type t
///   ω(c)    = Weight of contract c
///   outcome(c) = Result of contract c ∈ [-1, 1]
/// 
/// Properties:
///   V⟨t⟩ = 0   → Unknown, no track record
///   V⟨t⟩ > 0   → Net positive history, trusted
///   V⟨t⟩ < 0   → Net negative history, actively distrusted
/// </summary>
public class AgentTrustValueTests
{
    [Fact]
    public void ComputeTrustValue_EmptyHistory_ReturnsZero()
    {
        // ═══════════════════════════════════════════════════════════════
        // EQUATION: V⟨t⟩(Agent(t, ∅)) = Σ_{c ∈ ∅} ω(c)·outcome(c) = 0
        // 
        // Mathematical identity: Sum over empty set equals zero
        // ═══════════════════════════════════════════════════════════════
        
        // ARRANGE: h⟨t⟩ = ∅ (empty history)
        var agent = new Agent { SkillType = SkillTypes.Engineering };
        var expected = 0.0;

        // ACT: V⟨t⟩(agent)
        var actual = agent.ComputeTrustValue(SkillTypes.Engineering);

        // ASSERT: V⟨t⟩ = 0 (unknown/no track record)
        Assert.Equal(expected, actual);
    }

    [Fact]
    public void ComputeTrustValue_SingleSuccessfulContract_ReturnsPositive()
    {
        // ═══════════════════════════════════════════════════════════════
        // EQUATION: V⟨t⟩ = ω(c) · outcome(c)
        // 
        // When |h⟨t⟩| = 1:
        //   V⟨t⟩ = 2.0 × 1.0 = 2.0
        // ═══════════════════════════════════════════════════════════════
        
        // ARRANGE: h⟨t⟩ = {c} where ω(c)=2.0, outcome(c)=1.0
        var agent = new Agent { SkillType = SkillTypes.Engineering };
        var contract = ContractFactory.CreateSimple(
            skillType: SkillTypes.Engineering,
            outcome: 1.0,    // Success
            weight: 2.0
        );
        agent.AddToHistory(contract);
        
        var expected = 2.0;  // ω × outcome = 2.0 × 1.0

        // ACT
        var actual = agent.ComputeTrustValue(SkillTypes.Engineering);

        // ASSERT: V⟨t⟩ > 0 (trusted)
        Assert.Equal(expected, actual);
        Assert.True(actual > 0, "Positive outcome should yield positive trust");
    }

    [Fact]
    public void ComputeTrustValue_SingleFailedContract_ReturnsNegative()
    {
        // ═══════════════════════════════════════════════════════════════
        // EQUATION: V⟨t⟩ = ω(c) · outcome(c)
        // 
        // When outcome = -1 (failure):
        //   V⟨t⟩ = 3.0 × (-1.0) = -3.0
        // ═══════════════════════════════════════════════════════════════
        
        // ARRANGE: h⟨t⟩ = {c} where ω(c)=3.0, outcome(c)=-1.0
        var agent = new Agent { SkillType = SkillTypes.Engineering };
        var contract = ContractFactory.CreateSimple(
            skillType: SkillTypes.Engineering,
            outcome: -1.0,   // Failure
            weight: 3.0
        );
        agent.AddToHistory(contract);
        
        var expected = -3.0;  // ω × outcome = 3.0 × (-1.0)

        // ACT
        var actual = agent.ComputeTrustValue(SkillTypes.Engineering);

        // ASSERT: V⟨t⟩ < 0 (actively distrusted)
        Assert.Equal(expected, actual);
        Assert.True(actual < 0, "Negative outcome should yield negative trust");
    }

    [Fact]
    public void ComputeTrustValue_MixedOutcomes_ReturnsSumOfWeightedOutcomes()
    {
        // ═══════════════════════════════════════════════════════════════
        // EQUATION: V⟨t⟩ = Σ_{c ∈ h⟨t⟩} ω(c) · outcome(c)
        // 
        // h⟨t⟩ = {c, c₂, c₃, c₄}
        // 
        // Manual calculation:
        //   c: ω=1.0, outcome=+1.0  →  +1.0
        //   c₂: ω=0.8, outcome=+1.0  →  +0.8
        //   c₃: ω=1.0, outcome=-1.0  →  -1.0
        //   c₄: ω=0.5, outcome= 0.0  →   0.0
        //                            ────────
        //   V⟨t⟩ = 1.0 + 0.8 - 1.0 + 0.0 = 0.8
        // ═══════════════════════════════════════════════════════════════
        
        // ARRANGE
        var agent = new Agent { SkillType = SkillTypes.Engineering };
        var contracts = new[]
        {
            ContractFactory.CreateSimple(SkillTypes.Engineering, outcome: 1.0,  weight: 1.0),
            ContractFactory.CreateSimple(SkillTypes.Engineering, outcome: 1.0,  weight: 0.8),
            ContractFactory.CreateSimple(SkillTypes.Engineering, outcome: -1.0, weight: 1.0),
            ContractFactory.CreateSimple(SkillTypes.Engineering, outcome: 0.0,  weight: 0.5),
        };
        
        foreach (var c in contracts)
            agent.AddToHistory(c);

        var expected = 0.8;

        // ACT
        var actual = agent.ComputeTrustValue(SkillTypes.Engineering);

        // ASSERT
        Assert.Equal(expected, actual, precision: 10);
    }

    [Fact]
    public void ComputeTrustValue_PartialOutcomes_ReturnsCorrectSum()
    {
        // ═══════════════════════════════════════════════════════════════
        // EQUATION: outcome(c) ∈ [-1, 1] (continuous range)
        // 
        // Testing partial success/failure values:
        //   c: ω=2.0, outcome=+0.5  →  +1.0
        //   c₂: ω=2.0, outcome=-0.3  →  -0.6
        //                            ────────
        //   V⟨t⟩ = 1.0 - 0.6 = 0.4
        // ═══════════════════════════════════════════════════════════════
        
        // ARRANGE
        var agent = new Agent { SkillType = SkillTypes.Design };
        agent.AddToHistory(ContractFactory.CreateSimple(SkillTypes.Design, outcome: 0.5, weight: 2.0));
        agent.AddToHistory(ContractFactory.CreateSimple(SkillTypes.Design, outcome: -0.3, weight: 2.0));

        var expected = 0.4;  // (2.0 × 0.5) + (2.0 × -0.3) = 1.0 - 0.6

        // ACT
        var actual = agent.ComputeTrustValue(SkillTypes.Design);

        // ASSERT
        Assert.Equal(expected, actual, precision: 10);
    }

    [Fact]
    public void ComputeTrustValue_DifferentSkillTypes_AreIndependent()
    {
        // ═══════════════════════════════════════════════════════════════
        // PROPERTY: Skill types are independently scoped
        // 
        // V⟨e⟩ngineering(agent) is computed from h⟨e⟩ngineering only
        // V⟨d⟩esign(agent) is computed from h⟨d⟩esign only
        // 
        // Jane's example from whitepaper:
        //   V⟨e⟩ngineering = 85 cutes (thriving)
        //   V⟨d⟩esign = -12 cutes (struggling)
        // ═══════════════════════════════════════════════════════════════
        
        // ARRANGE: Two separate histories
        var agent = new Agent { SkillType = SkillTypes.Engineering };
        
        // Engineering contracts (10 successes × weight 8.5)
        for (int i = 0; i < 10; i++)
            agent.AddToHistory(ContractFactory.CreateSimple(SkillTypes.Engineering, outcome: 1.0, weight: 8.5));
        
        // Design contracts (5 failures × weight 2.4)
        for (int i = 0; i < 5; i++)
            agent.AddToHistory(ContractFactory.CreateSimple(SkillTypes.Design, outcome: -1.0, weight: 2.4));

        var expectedEngineering = 85.0;  // 10 × 8.5 × 1.0
        var expectedDesign = -12.0;       // 5 × 2.4 × (-1.0)

        // ACT
        var actualEngineering = agent.ComputeTrustValue(SkillTypes.Engineering);
        var actualDesign = agent.ComputeTrustValue(SkillTypes.Design);

        // ASSERT: Independent values
        Assert.Equal(expectedEngineering, actualEngineering, precision: 10);
        Assert.Equal(expectedDesign, actualDesign, precision: 10);
        Assert.True(actualEngineering > 0, "Engineering should be positive");
        Assert.True(actualDesign < 0, "Design should be negative");
    }

    [Fact]
    public void ComputeTrustValue_InvalidSkillType_ReturnsZero()
    {
        // ═══════════════════════════════════════════════════════════════
        // EDGE CASE: Invalid skill type input
        // 
        // V⟨t⟩(agent, null) = 0
        // V⟨t⟩(agent, "") = 0
        // ═══════════════════════════════════════════════════════════════
        
        // ARRANGE
        var agent = new Agent { SkillType = SkillTypes.Engineering };
        agent.AddToHistory(ContractFactory.CreateSimple(SkillTypes.Engineering, outcome: 1.0, weight: 5.0));

        // ACT & ASSERT
        Assert.Equal(0.0, agent.ComputeTrustValue(null!));
        Assert.Equal(0.0, agent.ComputeTrustValue(""));
        Assert.Equal(0.0, agent.ComputeTrustValue("   "));
    }

    [Theory]
    [InlineData(1.0, 1.0, 1.0)]       // Full success
    [InlineData(-1.0, 1.0, -1.0)]     // Full failure
    [InlineData(0.0, 1.0, 0.0)]       // Neutral
    [InlineData(0.5, 2.0, 1.0)]       // Partial success, weighted
    [InlineData(-0.5, 4.0, -2.0)]     // Partial failure, weighted
    [InlineData(1.0, 0.0, 0.0)]       // Zero weight
    public void ComputeTrustValue_SingleContract_ReturnsWeightTimesOutcome(
        double outcome, 
        double weight, 
        double expected)
    {
        // ═══════════════════════════════════════════════════════════════
        // EQUATION: V⟨t⟩ = ω(c) · outcome(c)  when |h⟨t⟩| = 1
        // 
        // This is the fundamental multiplication at the heart of trust.
        // ═══════════════════════════════════════════════════════════════
        
        // ARRANGE
        var agent = new Agent { SkillType = SkillTypes.Engineering };
        agent.AddToHistory(ContractFactory.CreateSimple(
            SkillTypes.Engineering, outcome: outcome, weight: weight));

        // ACT
        var actual = agent.ComputeTrustValue(SkillTypes.Engineering);

        // ASSERT
        Assert.Equal(expected, actual, precision: 10);
    }
}

```

---

## 2. DAO Trust Aggregation Tests

Tests for DAO trust aggregation.

**Mathematical Notation:**

```
┌─────────────────────────────────────────────────────────────────────
│  V⟨t⟩(DAO(S)) = Φ({V⟨t⟩(q) : q ∈ S})                                 │
└─────────────────────────────────────────────────────────────────────┘
```

```csharp

/// <summary>
/// Tests for DAO trust aggregation.
/// 
/// Mathematical notation:
/// ┌─────────────────────────────────────────────────────────────────────
/// │  V⟨t⟩(DAO(S)) = Φ({V⟨t⟩(q) : q ∈ S})                                 │
/// └─────────────────────────────────────────────────────────────────────┘
/// 
/// Where:
///   S   = Set of member entities (Agents or DAOs)
///   Φ   = Aggregation function (sum, avg, min, max, etc.)
///   q   = A member entity (q ∈ S)
/// 
/// Recursive property: A DAO is itself a q⟨T⟩, enabling:
///   q⟨T⟩ ::= Agent(t, h⟨t⟩) | DAO({q⟨T⟩})
/// </summary>
public class DAOTrustAggregationTests
{
    [Fact]
    public void ComputeTrustValue_EmptyDAO_ReturnsZero()
    {
        // ═══════════════════════════════════════════════════════════════
        // EQUATION: V⟨t⟩(DAO(∅)) = Φ(∅) = 0
        // 
        // All aggregation functions return 0 for empty sets.
        // ═══════════════════════════════════════════════════════════════
        
        // ARRANGE
        var dao = new DAO
        {
            Members = new HashSet<QuantumOfTrust>(),
            Phi = AggregationFunctions.Sum
        };

        // ACT
        var actual = dao.ComputeTrustValue(SkillTypes.Engineering);

        // ASSERT
        Assert.Equal(0.0, actual);
    }

    [Fact]
    public void ComputeTrustValue_WithSumAggregation_ReturnsTotalCapability()
    {
        // ═══════════════════════════════════════════════════════════════
        // EQUATION: Φ_sum({V, V₂, V₃}) = V + V₂ + V₃
        // 
        // Sum represents total combined capability.
        // 
        // Members:
        //   Agent: V⟨t⟩ = 10.0
        //   Agent₂: V⟨t⟩ = 20.0
        //   Agent₃: V⟨t⟩ = 30.0
        // 
        // V⟨t⟩(DAO) = 10 + 20 + 30 = 60
        // ═══════════════════════════════════════════════════════════════
        
        // ARRANGE
        var agents = CreateAgentsWithTrust(new[] { 10.0, 20.0, 30.0 });
        var dao = new DAO
        {
            Members = agents.Cast<QuantumOfTrust>().ToHashSet(),
            Phi = AggregationFunctions.Sum
        };

        var expected = 60.0;

        // ACT
        var actual = dao.ComputeTrustValue(SkillTypes.Engineering);

        // ASSERT
        Assert.Equal(expected, actual, precision: 10);
    }

    [Fact]
    public void ComputeTrustValue_WithAverageAggregation_ReturnsMeanReliability()
    {
        // ═══════════════════════════════════════════════════════════════
        // EQUATION: Φ_avg({V, V₂, V₃}) = (V + V₂ + V₃) / 3
        // 
        // Average represents mean reliability of members.
        // 
        // Members: {10.0, 20.0, 30.0}
        // V⟨t⟩(DAO) = (10 + 20 + 30) / 3 = 20
        // ═══════════════════════════════════════════════════════════════
        
        // ARRANGE
        var agents = CreateAgentsWithTrust(new[] { 10.0, 20.0, 30.0 });
        var dao = new DAO
        {
            Members = agents.Cast<QuantumOfTrust>().ToHashSet(),
            Phi = AggregationFunctions.Average
        };

        var expected = 20.0;

        // ACT
        var actual = dao.ComputeTrustValue(SkillTypes.Engineering);

        // ASSERT
        Assert.Equal(expected, actual, precision: 10);
    }

    [Fact]
    public void ComputeTrustValue_WithMinimumAggregation_ReturnsWeakestLink()
    {
        // ═══════════════════════════════════════════════════════════════
        // EQUATION: Φ_min({V, V₂, V₃}) = min(V, V₂, V₃)
        // 
        // Minimum represents "weakest link" analysis.
        // Use case: Security-focused DAO (only as strong as weakest member)
        // 
        // Members: {10.0, 5.0, 30.0}
        // V⟨t⟩(DAO) = min(10, 5, 30) = 5
        // ═══════════════════════════════════════════════════════════════
        
        // ARRANGE
        var agents = CreateAgentsWithTrust(new[] { 10.0, 5.0, 30.0 });
        var dao = new DAO
        {
            Members = agents.Cast<QuantumOfTrust>().ToHashSet(),
            Phi = AggregationFunctions.Minimum
        };

        var expected = 5.0;

        // ACT
        var actual = dao.ComputeTrustValue(SkillTypes.Engineering);

        // ASSERT
        Assert.Equal(expected, actual, precision: 10);
    }

    [Fact]
    public void ComputeTrustValue_WithMaximumAggregation_ReturnsStrongestMember()
    {
        // ═══════════════════════════════════════════════════════════════
        // EQUATION: Φ_max({V, V₂, V₃}) = max(V, V₂, V₃)
        // 
        // Maximum represents "strongest member" capability.
        // 
        // Members: {10.0, 5.0, 30.0}
        // V⟨t⟩(DAO) = max(10, 5, 30) = 30
        // ═══════════════════════════════════════════════════════════════
        
        // ARRANGE
        var agents = CreateAgentsWithTrust(new[] { 10.0, 5.0, 30.0 });
        var dao = new DAO
        {
            Members = agents.Cast<QuantumOfTrust>().ToHashSet(),
            Phi = AggregationFunctions.Maximum
        };

        var expected = 30.0;

        // ACT
        var actual = dao.ComputeTrustValue(SkillTypes.Engineering);

        // ASSERT
        Assert.Equal(expected, actual, precision: 10);
    }

    [Fact]
    public void ComputeTrustValue_WithMedianAggregation_ReturnsMiddleValue()
    {
        // ═══════════════════════════════════════════════════════════════
        // EQUATION: Φ_median({V, V₂, ..., Vₙ}) = middle value when sorted
        // 
        // Median is robust to outliers.
        // 
        // Members: {5.0, 10.0, 100.0} (sorted)
        // V⟨t⟩(DAO) = median = 10.0
        // ═══════════════════════════════════════════════════════════════
        
        // ARRANGE
        var agents = CreateAgentsWithTrust(new[] { 100.0, 5.0, 10.0 });  // Unsorted
        var dao = new DAO
        {
            Members = agents.Cast<QuantumOfTrust>().ToHashSet(),
            Phi = AggregationFunctions.Median
        };

        var expected = 10.0;  // Middle of {5, 10, 100}

        // ACT
        var actual = dao.ComputeTrustValue(SkillTypes.Engineering);

        // ASSERT
        Assert.Equal(expected, actual, precision: 10);
    }

    [Fact]
    public void ComputeTrustValue_NestedDAOs_ComputesRecursively()
    {
        // ═══════════════════════════════════════════════════════════════
        // RECURSIVE PROPERTY: q⟨T⟩ ::= Agent | DAO({q⟨T⟩})
        // 
        // "Turtles all the way down" - DAOs can contain DAOs.
        // 
        // Structure:
        //   OuterDAO (Φ = sum)
        //   ├── Agent: V⟨t⟩ = 10.0
        //   └── InnerDAO (Φ = average)
        //       ├── Agent₂: V⟨t⟩ = 20.0
        //       └── Agent₃: V⟨t⟩ = 40.0
        // 
        // Calculation:
        //   V⟨t⟩(InnerDAO) = avg(20, 40) = 30
        //   V⟨t⟩(OuterDAO) = sum(10, 30) = 40
        // ═══════════════════════════════════════════════════════════════
        
        // ARRANGE
        var agent1 = CreateAgentWithTrust(10.0);
        var agent2 = CreateAgentWithTrust(20.0);
        var agent3 = CreateAgentWithTrust(40.0);

        var innerDao = new DAO
        {
            Members = new HashSet<QuantumOfTrust> { agent2, agent3 },
            Phi = AggregationFunctions.Average
        };

        var outerDao = new DAO
        {
            Members = new HashSet<QuantumOfTrust> { agent1, innerDao },
            Phi = AggregationFunctions.Sum
        };

        var expected = 40.0;  // 10 + avg(20, 40) = 10 + 30 = 40

        // ACT
        var actual = outerDao.ComputeTrustValue(SkillTypes.Engineering);

        // ASSERT
        Assert.Equal(expected, actual, precision: 10);
    }

    [Fact]
    public void ComputeTrustValue_WithNegativeTrust_HandlesCorrectly()
    {
        // ═══════════════════════════════════════════════════════════════
        // PROPERTY: V⟨t⟩ ∈  (can be negative)
        // 
        // DAOs can have members with negative trust (distrusted).
        // 
        // Members: {10.0, -5.0, 20.0}
        // Φ_sum = 10 + (-5) + 20 = 25
        // ═══════════════════════════════════════════════════════════════
        
        // ARRANGE
        var agents = CreateAgentsWithTrust(new[] { 10.0, -5.0, 20.0 });
        var dao = new DAO
        {
            Members = agents.Cast<QuantumOfTrust>().ToHashSet(),
            Phi = AggregationFunctions.Sum
        };

        var expected = 25.0;

        // ACT
        var actual = dao.ComputeTrustValue(SkillTypes.Engineering);

        // ASSERT
        Assert.Equal(expected, actual, precision: 10);
    }

    [Fact]
    public void ComputeTrustValue_WithoutPhi_ThrowsInvalidOperation()
    {
        // ═══════════════════════════════════════════════════════════════
        // CONSTRAINT: Φ must be defined before computing trust
        // ═══════════════════════════════════════════════════════════════
        
        // ARRANGE
        var dao = new DAO
        {
            Members = new HashSet<QuantumOfTrust> { new Agent { SkillType = SkillTypes.Engineering } },
            Phi = null  // Not set
        };

        // ACT & ASSERT
        Assert.Throws<InvalidOperationException>(() => 
            dao.ComputeTrustValue(SkillTypes.Engineering));
    }

    // Helper methods
    private static Agent CreateAgentWithTrust(double trustValue)
    {
        var agent = new Agent { SkillType = SkillTypes.Engineering };
        if (trustValue != 0)
        {
            agent.AddToHistory(ContractFactory.CreateSimple(
                SkillTypes.Engineering, 
                outcome: trustValue > 0 ? 1.0 : -1.0, 
                weight: Math.Abs(trustValue)));
        }
        return agent;
    }

    private static List<Agent> CreateAgentsWithTrust(double[] trustValues)
    {
        return trustValues.Select(CreateAgentWithTrust).ToList();
    }
}

```

---

## 3. Aggregation Functions Tests

Tests for aggregation functions Φ.

**Mathematical Notation:**

```
┌─────────────────────────────────────────────────────────────────────
│  Φ: {} →                                                         │
│                                                                     │
│  Φ_sum({x, ..., xₙ}) = Σᵢ xᵢ                                      │
│  Φ_avg({x, ..., xₙ}) = (Σᵢ xᵢ) / n                                │
│  Φ_min({x, ..., xₙ}) = min(x, ..., xₙ)                           │
│  Φ_max({x, ..., xₙ}) = max(x, ..., xₙ)                           │
│  Φ_median = middle value when sorted                                │
│  Φ_weighted({x,...,xₙ}, {w,...,wₙ}) = (Σᵢ xᵢwᵢ) / (Σᵢ wᵢ)       │
└─────────────────────────────────────────────────────────────────────┘
```

```csharp

/// <summary>
/// Tests for aggregation functions Φ.
/// 
/// Mathematical notation:
/// ┌─────────────────────────────────────────────────────────────────────
/// │  Φ: {} →                                                         │
/// │                                                                     │
/// │  Φ_sum({x, ..., xₙ}) = Σᵢ xᵢ                                      │
/// │  Φ_avg({x, ..., xₙ}) = (Σᵢ xᵢ) / n                                │
/// │  Φ_min({x, ..., xₙ}) = min(x, ..., xₙ)                           │
/// │  Φ_max({x, ..., xₙ}) = max(x, ..., xₙ)                           │
/// │  Φ_median = middle value when sorted                                │
/// │  Φ_weighted({x,...,xₙ}, {w,...,wₙ}) = (Σᵢ xᵢwᵢ) / (Σᵢ wᵢ)       │
/// └─────────────────────────────────────────────────────────────────────┘
/// </summary>
public class AggregationFunctionsTests
{
    [Theory]
    [InlineData(new double[] { }, 0.0)]
    [InlineData(new double[] { 5.0 }, 5.0)]
    [InlineData(new double[] { 1.0, 2.0, 3.0 }, 6.0)]
    [InlineData(new double[] { -1.0, 2.0, -3.0 }, -2.0)]
    public void Sum_ReturnsCorrectTotal(double[] values, double expected)
    {
        // ═══════════════════════════════════════════════════════════════
        // EQUATION: Φ_sum = Σᵢ xᵢ
        // ═══════════════════════════════════════════════════════════════
        
        // ACT
        var actual = AggregationFunctions.Sum(values);

        // ASSERT
        Assert.Equal(expected, actual, precision: 10);
    }

    [Theory]
    [InlineData(new double[] { }, 0.0)]
    [InlineData(new double[] { 6.0 }, 6.0)]
    [InlineData(new double[] { 2.0, 4.0, 6.0 }, 4.0)]
    [InlineData(new double[] { -2.0, 4.0 }, 1.0)]
    public void Average_ReturnsMean(double[] values, double expected)
    {
        // ═══════════════════════════════════════════════════════════════
        // EQUATION: Φ_avg = (Σᵢ xᵢ) / n
        // ═══════════════════════════════════════════════════════════════
        
        // ACT
        var actual = AggregationFunctions.Average(values);

        // ASSERT
        Assert.Equal(expected, actual, precision: 10);
    }

    [Theory]
    [InlineData(new double[] { }, 0.0)]
    [InlineData(new double[] { 5.0 }, 5.0)]
    [InlineData(new double[] { 3.0, 1.0, 4.0 }, 1.0)]
    [InlineData(new double[] { -3.0, -1.0, -4.0 }, -4.0)]
    public void Minimum_ReturnsSmallestValue(double[] values, double expected)
    {
        // ═══════════════════════════════════════════════════════════════
        // EQUATION: Φ_min = min(x, ..., xₙ)
        // ═══════════════════════════════════════════════════════════════
        
        // ACT
        var actual = AggregationFunctions.Minimum(values);

        // ASSERT
        Assert.Equal(expected, actual, precision: 10);
    }

    [Theory]
    [InlineData(new double[] { }, 0.0)]
    [InlineData(new double[] { 5.0 }, 5.0)]
    [InlineData(new double[] { 3.0, 1.0, 4.0 }, 4.0)]
    [InlineData(new double[] { -3.0, -1.0, -4.0 }, -1.0)]
    public void Maximum_ReturnsLargestValue(double[] values, double expected)
    {
        // ═══════════════════════════════════════════════════════════════
        // EQUATION: Φ_max = max(x, ..., xₙ)
        // ═══════════════════════════════════════════════════════════════
        
        // ACT
        var actual = AggregationFunctions.Maximum(values);

        // ASSERT
        Assert.Equal(expected, actual, precision: 10);
    }

    [Theory]
    [InlineData(new double[] { }, 0.0)]
    [InlineData(new double[] { 5.0 }, 5.0)]
    [InlineData(new double[] { 1.0, 2.0, 3.0 }, 2.0)]           // Odd count: middle
    [InlineData(new double[] { 1.0, 2.0, 3.0, 4.0 }, 2.5)]      // Even count: avg of middle two
    [InlineData(new double[] { 100.0, 1.0, 2.0 }, 2.0)]          // Outlier handling
    public void Median_ReturnsMiddleValue(double[] values, double expected)
    {
        // ═══════════════════════════════════════════════════════════════
        // EQUATION: Φ_median = middle value when sorted
        // 
        // For even n: average of two middle values
        // For odd n: the middle value
        // ═══════════════════════════════════════════════════════════════
        
        // ACT
        var actual = AggregationFunctions.Median(values);

        // ASSERT
        Assert.Equal(expected, actual, precision: 10);
    }

    [Fact]
    public void WeightedAverage_ReturnsCorrectValue()
    {
        // ═══════════════════════════════════════════════════════════════
        // EQUATION: Φ_weighted = (Σᵢ xᵢwᵢ) / (Σᵢ wᵢ)
        // 
        // values:  {10, 20, 30}
        // weights: {1,  2,  3}
        // 
        // = (10×1 + 20×2 + 30×3) / (1 + 2 + 3)
        // = (10 + 40 + 90) / 6
        // = 140 / 6
        // ≈ 23.333...
        // ═══════════════════════════════════════════════════════════════
        
        // ARRANGE
        var values = new[] { 10.0, 20.0, 30.0 };
        var weights = new[] { 1.0, 2.0, 3.0 };
        var expected = 140.0 / 6.0;

        // ACT
        var actual = AggregationFunctions.WeightedAverage(values, weights);

        // ASSERT
        Assert.Equal(expected, actual, precision: 10);
    }

    [Fact]
    public void WeightedAverage_WithZeroTotalWeight_ReturnsZero()
    {
        // ═══════════════════════════════════════════════════════════════
        // EDGE CASE: Σᵢ wᵢ = 0 → undefined, return 0
        // ═══════════════════════════════════════════════════════════════
        
        // ARRANGE
        var values = new[] { 10.0, 20.0 };
        var weights = new[] { 0.0, 0.0 };

        // ACT
        var actual = AggregationFunctions.WeightedAverage(values, weights);

        // ASSERT
        Assert.Equal(0.0, actual);
    }

    [Fact]
    public void AllFunctions_HandleNullGracefully()
    {
        // ═══════════════════════════════════════════════════════════════
        // EDGE CASE: null input → return 0
        // ═══════════════════════════════════════════════════════════════
        
        Assert.Equal(0.0, AggregationFunctions.Sum(null!));
        Assert.Equal(0.0, AggregationFunctions.Average(null!));
        Assert.Equal(0.0, AggregationFunctions.Minimum(null!));
        Assert.Equal(0.0, AggregationFunctions.Maximum(null!));
        Assert.Equal(0.0, AggregationFunctions.Median(null!));
        Assert.Equal(0.0, AggregationFunctions.WeightedAverage(null!, null!));
    }
}

```

---

## 4. Outcome Function Tests

Tests for the outcome function.

**Mathematical Notation:**

```
┌─────────────────────────────────────────────────────────────────────
│  outcome(c) ∈ [-1, 1]                                              │
│                                                                     │
│  Discrete special case: {-1, 0, 1} = {failure, partial, success}   │
└─────────────────────────────────────────────────────────────────────┘
```

```csharp

/// <summary>
/// Tests for the outcome function.
/// 
/// Mathematical notation:
/// ┌─────────────────────────────────────────────────────────────────────
/// │  outcome(c) ∈ [-1, 1]                                              │
/// │                                                                     │
/// │  Discrete special case: {-1, 0, 1} = {failure, partial, success}   │
/// └─────────────────────────────────────────────────────────────────────┘
/// </summary>
public class OutcomeFunctionTests
{
    [Theory]
    [InlineData(-1.0)]
    [InlineData(-0.5)]
    [InlineData(0.0)]
    [InlineData(0.5)]
    [InlineData(1.0)]
    public void ValidateOutcome_AcceptsValidRange(double outcome)
    {
        // ═══════════════════════════════════════════════════════════════
        // CONSTRAINT: outcome(c) ∈ [-1, 1]
        // ═══════════════════════════════════════════════════════════════
        
        // ACT & ASSERT: Should not throw
        var result = OutcomeCalculator.ValidateOutcome(outcome);
        Assert.Equal(outcome, result);
    }

    [Theory]
    [InlineData(-1.1)]
    [InlineData(1.1)]
    [InlineData(-100)]
    [InlineData(100)]
    public void ValidateOutcome_RejectsInvalidRange(double outcome)
    {
        // ═══════════════════════════════════════════════════════════════
        // CONSTRAINT: outcome(c) ∉ [-1, 1] → throw
        // ═══════════════════════════════════════════════════════════════
        
        // ACT & ASSERT
        Assert.Throws<ArgumentOutOfRangeException>(() => 
            OutcomeCalculator.ValidateOutcome(outcome));
    }

    [Theory]
    [InlineData(DiscreteOutcome.Failure, -1.0)]
    [InlineData(DiscreteOutcome.Partial, 0.0)]
    [InlineData(DiscreteOutcome.Success, 1.0)]
    public void FromDiscrete_ConvertsCorrectly(DiscreteOutcome discrete, double expected)
    {
        // ═══════════════════════════════════════════════════════════════
        // DISCRETE CASE: {-1, 0, 1} = {failure, partial, success}
        // ═══════════════════════════════════════════════════════════════
        
        // ACT
        var actual = OutcomeCalculator.FromDiscrete(discrete);

        // ASSERT
        Assert.Equal(expected, actual);
    }

    [Theory]
    [InlineData(1.0, 1.0, 1.0)]      // 100% complete, 100% quality → +1
    [InlineData(0.0, 0.0, -1.0)]     // 0% complete, 0% quality → -1
    [InlineData(0.5, 0.5, 0.0)]      // 50% each → 0 (neutral)
    [InlineData(1.0, 0.0, 0.0)]      // Complete but bad quality → 0
    [InlineData(0.75, 0.75, 0.5)]    // 75% each → +0.5
    public void CalculatePartialOutcome_MapsCorrectly(
        double completion, 
        double quality, 
        double expected)
    {
        // ═══════════════════════════════════════════════════════════════
        // MAPPING: [0,1] × [0,1] → [-1,1]
        // 
        // Formula: outcome = ((completion + quality) / 2) × 2 - 1
        //        = (completion + quality) - 1
        // ═══════════════════════════════════════════════════════════════
        
        // ACT
        var actual = OutcomeCalculator.CalculatePartialOutcome(completion, quality);

        // ASSERT
        Assert.Equal(expected, actual, precision: 10);
    }

    [Theory]
    [InlineData(-0.1, 0.5)]
    [InlineData(1.1, 0.5)]
    [InlineData(0.5, -0.1)]
    [InlineData(0.5, 1.1)]
    public void CalculatePartialOutcome_RejectsInvalidInputs(double completion, double quality)
    {
        // ═══════════════════════════════════════════════════════════════
        // CONSTRAINT: completion, quality ∈ [0, 1]
        // ═══════════════════════════════════════════════════════════════
        
        // ACT & ASSERT
        Assert.Throws<ArgumentOutOfRangeException>(() => 
            OutcomeCalculator.CalculatePartialOutcome(completion, quality));
    }

    [Theory]
    [InlineData(-5.0, -1.0)]
    [InlineData(5.0, 1.0)]
    [InlineData(0.5, 0.5)]
    public void Clamp_ConstrainsToValidRange(double input, double expected)
    {
        // ═══════════════════════════════════════════════════════════════
        // UTILITY: Clamp any value to [-1, 1]
        // ═══════════════════════════════════════════════════════════════
        
        // ACT
        var actual = OutcomeCalculator.Clamp(input);

        // ASSERT
        Assert.Equal(expected, actual);
    }
}

```

---

## 5. Weighting Function Tests

Tests for the weighting function ω.

**Mathematical Notation:**

```
┌─────────────────────────────────────────────────────────────────────
│  ω(c) = f(s(c), d(c), V⟨t⟩(a_consumer), recency(c))                 │
│                                                                     │
│  Components:                                                        │
│    stake_weight       = ln(1 + s)           [logarithmic]          │
│    difficulty_weight  = 0.5 + (d/10) × 1.5  [linear, 0.5 to 2.0]   │
│    counterparty_weight= 1 + tanh(V⟨t⟩/100) × 0.5  [sigmoid, 0.5-1.5]│
│    recency_weight     = 0.5^(days/365)      [exponential decay]    │
│                                                                     │
│  ω = stake × difficulty × counterparty × recency                   │
└─────────────────────────────────────────────────────────────────────┘
```

```csharp

/// <summary>
/// Tests for the weighting function ω.
/// 
/// Mathematical notation:
/// ┌─────────────────────────────────────────────────────────────────────
/// │  ω(c) = f(s(c), d(c), V⟨t⟩(a_consumer), recency(c))                 │
/// │                                                                     │
/// │  Components:                                                        │
/// │    stake_weight       = ln(1 + s)           [logarithmic]          │
/// │    difficulty_weight  = 0.5 + (d/10) × 1.5  [linear, 0.5 to 2.0]   │
/// │    counterparty_weight= 1 + tanh(V⟨t⟩/100) × 0.5  [sigmoid, 0.5-1.5]│
/// │    recency_weight     = 0.5^(days/365)      [exponential decay]    │
/// │                                                                     │
/// │  ω = stake × difficulty × counterparty × recency                   │
/// └─────────────────────────────────────────────────────────────────────┘
/// </summary>
public class WeightingFunctionTests
{
    private readonly WeightCalculator _calculator = new();
    private readonly DateTime _now = new(2025, 1, 1, 12, 0, 0, DateTimeKind.Utc);

    [Theory]
    [InlineData(0, 0.0)]                    // ln(1 + 0) = ln(1) = 0
    [InlineData(1, 0.693147)]               // ln(1 + 1) = ln(2) ≈ 0.693
    [InlineData(100, 4.615120)]             // ln(101) ≈ 4.615
    [InlineData(10000, 9.210440)]           // ln(10001) ≈ 9.210
    public void StakeWeight_IsLogarithmic(double stake, double expectedLogComponent)
    {
        // ═══════════════════════════════════════════════════════════════
        // EQUATION: stake_weight = ln(1 + s)
        // 
        // Logarithmic scaling prevents high stakes from dominating.
        // ═══════════════════════════════════════════════════════════════
        
        // ARRANGE
        var contract = ContractFactory.CreateSimple(
            skillType: SkillTypes.Engineering,
            outcome: 0,
            weight: 0,
            stake: stake,
            difficulty: 5  // Middle difficulty for isolation
        );

        // ACT
        var weight = _calculator.ComputeWeight(contract, 0, _now);

        // ASSERT: Weight should be proportional to ln(1+stake)
        // With d=5 → difficulty_weight ≈ 1.25, counterparty=1.0, recency=1.0
        // So weight ≈ ln(1+s) × 1.25 × 1.0 × 1.0
        var expectedWeight = expectedLogComponent * 1.25 * 1.0 * 1.0;
        Assert.Equal(Math.Max(expectedWeight, TrustConstants.MinWeight), weight, precision: 3);
    }

    [Theory]
    [InlineData(0, 0.5)]     // d=0  → 0.5 + 0.0 × 1.5 = 0.5
    [InlineData(5, 1.25)]    // d=5  → 0.5 + 0.5 × 1.5 = 1.25
    [InlineData(10, 2.0)]    // d=10 → 0.5 + 1.0 × 1.5 = 2.0
    public void DifficultyWeight_IsLinear(double difficulty, double expectedDiffWeight)
    {
        // ═══════════════════════════════════════════════════════════════
        // EQUATION: difficulty_weight = 0.5 + (d/10) × 1.5
        // 
        // Maps difficulty [0,10] to weight [0.5, 2.0]
        // Higher difficulty = more signal
        // ═══════════════════════════════════════════════════════════════
        
        // ARRANGE: Fixed stake=100 → ln(101) ≈ 4.615
        var contract = ContractFactory.CreateSimple(
            skillType: SkillTypes.Engineering,
            outcome: 0,
            weight: 0,
            stake: 100,
            difficulty: difficulty
        );

        // ACT
        var weight = _calculator.ComputeWeight(contract, 0, _now);

        // ASSERT: With counterparty=1.0, recency=1.0
        var expectedWeight = Math.Log(101) * expectedDiffWeight * 1.0 * 1.0;
        Assert.Equal(expectedWeight, weight, precision: 3);
    }

    [Theory]
    [InlineData(0, 1.0)]           // V⟨t⟩=0 → tanh(0)=0 → 1 + 0×0.5 = 1.0
    [InlineData(100, 1.38)]        // V⟨t⟩=100 → tanh(1)≈0.76 → 1 + 0.76×0.5 ≈ 1.38
    [InlineData(-100, 0.62)]       // V⟨t⟩=-100 → tanh(-1)≈-0.76 → 1 - 0.76×0.5 ≈ 0.62
    [InlineData(1000, 1.5)]        // V⟨t⟩=1000 → tanh(10)≈1 → 1 + 1×0.5 = 1.5 (max)
    public void CounterpartyWeight_IsSigmoidBounded(double counterpartyTrust, double expectedCpWeight)
    {
        // ═══════════════════════════════════════════════════════════════
        // EQUATION: counterparty_weight = 1 + tanh(V⟨t⟩/100) × 0.5
        // 
        // Maps any trust value to [0.5, 1.5]
        // High trust counterparty → more signal (up to 1.5×)
        // Low trust counterparty → less signal (down to 0.5×)
        // ═══════════════════════════════════════════════════════════════
        
        // ARRANGE: Fixed stake=100, difficulty=5
        var contract = ContractFactory.CreateSimple(
            skillType: SkillTypes.Engineering,
            outcome: 0,
            weight: 0,
            stake: 100,
            difficulty: 5
        );

        // ACT
        var weight = _calculator.ComputeWeight(contract, counterpartyTrust, _now);

        // ASSERT: ln(101) × 1.25 × cpWeight × 1.0
        var expectedWeight = Math.Log(101) * 1.25 * expectedCpWeight * 1.0;
        Assert.Equal(expectedWeight, weight, precision: 1);
    }

    [Theory]
    [InlineData(0, 1.0)]           // Today → 0.5^0 = 1.0
    [InlineData(365, 0.5)]         // 1 year ago → 0.5^1 = 0.5
    [InlineData(730, 0.25)]        // 2 years ago → 0.5^2 = 0.25
    [InlineData(1095, 0.125)]      // 3 years ago → 0.5^3 = 0.125
    public void RecencyWeight_DecaysExponentially(int daysAgo, double expectedRecencyWeight)
    {
        // ═══════════════════════════════════════════════════════════════
        // EQUATION: recency_weight = 0.5^(days/365)
        // 
        // Half-life of 365 days.
        // Contract from 1 year ago contributes half as much as today's.
        // ═══════════════════════════════════════════════════════════════
        
        // ARRANGE
        var completedAt = _now.AddDays(-daysAgo);
        var contract = new Contract
        {
            SkillType = SkillTypes.Engineering,
            Stake = 100,
            Difficulty = 5,
            Outcome = 0,
            CompletedAt = completedAt,
            Deadline = _now
        };

        // ACT
        var weight = _calculator.ComputeWeight(contract, 0, _now);

        // ASSERT: ln(101) × 1.25 × 1.0 × recency
        var expectedWeight = Math.Log(101) * 1.25 * 1.0 * expectedRecencyWeight;
        Assert.Equal(expectedWeight, weight, precision: 2);
    }

    [Fact]
    public void ComputeWeight_NullContract_Throws()
    {
        // ═══════════════════════════════════════════════════════════════
        // CONSTRAINT: Contract cannot be null
        // ═══════════════════════════════════════════════════════════════
        
        Assert.Throws<ArgumentNullException>(() => 
            _calculator.ComputeWeight(null!, 0, _now));
    }

    [Fact]
    public void ComputeWeight_ReturnsAtLeastMinWeight()
    {
        // ═══════════════════════════════════════════════════════════════
        // CONSTRAINT: ω(c) ≥ MinWeight (for numerical stability)
        // ═══════════════════════════════════════════════════════════════
        
        // ARRANGE: Zero stake, zero difficulty, very old
        var contract = new Contract
        {
            SkillType = SkillTypes.Engineering,
            Stake = 0,
            Difficulty = 0,
            Outcome = 0,
            CompletedAt = _now.AddYears(-100),
            Deadline = _now
        };

        // ACT
        var weight = _calculator.ComputeWeight(contract, -1000, _now);

        // ASSERT
        Assert.True(weight >= TrustConstants.MinWeight);
    }
}

```

---

## 6. History Evolution Tests

Tests for history evolution.

**Mathematical Notation:**

```
┌─────────────────────────────────────────────────────────────────────
│  h⟨t⟩^(n+1)(a) = h⟨t⟩^(n)(a) ∪ {c⟨n⟩}                                 │
└─────────────────────────────────────────────────────────────────────┘
```

```csharp

/// <summary>
/// Tests for history evolution.
/// 
/// Mathematical notation:
/// ┌─────────────────────────────────────────────────────────────────────
/// │  h⟨t⟩^(n+1)(a) = h⟨t⟩^(n)(a) ∪ {c⟨n⟩}                                 │
/// └─────────────────────────────────────────────────────────────────────┘
/// 
/// The history grows by adding completed contracts.
/// History is append-only and partitioned by skill type.
/// </summary>
public class HistoryEvolutionTests
{
    [Fact]
    public void AddToHistory_AppendsContract()
    {
        // ═══════════════════════════════════════════════════════════════
        // EQUATION: h⟨t⟩^(n+1) = h⟨t⟩^(n) ∪ {c⟨n⟩}
        // 
        // Union operation: Add new contract to existing history.
        // ═══════════════════════════════════════════════════════════════
        
        // ARRANGE: h⟨t⟩^(0) = ∅
        var agent = new Agent { SkillType = SkillTypes.Engineering };
        Assert.Empty(agent.ContractHistory);

        // ACT: h⟨t⟩^(1) = h⟨t⟩^(0) ∪ {c}
        var c1 = ContractFactory.CreateSimple(SkillTypes.Engineering, 1.0, 1.0);
        agent.AddToHistory(c1);

        // ASSERT
        Assert.Single(agent.ContractHistory);
        Assert.Contains(c1, agent.ContractHistory);
    }

    [Fact]
    public void AddToHistory_PreservesOrder()
    {
        // ═══════════════════════════════════════════════════════════════
        // PROPERTY: History maintains insertion order
        // 
        // h⟨t⟩ = {c, c₂, c₃} in order of completion
        // ═══════════════════════════════════════════════════════════════
        
        // ARRANGE
        var agent = new Agent { SkillType = SkillTypes.Engineering };
        var contracts = Enumerable.Range(1, 5)
            .Select(i => ContractFactory.CreateSimple(SkillTypes.Engineering, i * 0.1, 1.0))
            .ToList();

        // ACT
        foreach (var c in contracts)
            agent.AddToHistory(c);

        // ASSERT
        Assert.Equal(5, agent.ContractHistory.Count);
        for (int i = 0; i < 5; i++)
            Assert.Same(contracts[i], agent.ContractHistory[i]);
    }

    [Fact]
    public void AddToHistory_NullContract_Throws()
    {
        // ═══════════════════════════════════════════════════════════════
        // CONSTRAINT: Cannot add null contract
        // ═══════════════════════════════════════════════════════════════
        
        var agent = new Agent { SkillType = SkillTypes.Engineering };
        Assert.Throws<ArgumentNullException>(() => agent.AddToHistory(null!));
    }

    [Fact]
    public void GetHistoryForSkill_FiltersCorrectly()
    {
        // ═══════════════════════════════════════════════════════════════
        // PROPERTY: History is partitioned by skill type
        // 
        // h⟨e⟩ngineering ∩ h⟨d⟩esign = ∅ (disjoint sets)
        // ═══════════════════════════════════════════════════════════════
        
        // ARRANGE
        var agent = new Agent { SkillType = SkillTypes.Engineering };
        agent.AddToHistory(ContractFactory.CreateSimple(SkillTypes.Engineering, 1.0, 1.0));
        agent.AddToHistory(ContractFactory.CreateSimple(SkillTypes.Engineering, 1.0, 1.0));
        agent.AddToHistory(ContractFactory.CreateSimple(SkillTypes.Design, 1.0, 1.0));

        // ACT
        var engineeringHistory = agent.GetHistoryForSkill(SkillTypes.Engineering).ToList();
        var designHistory = agent.GetHistoryForSkill(SkillTypes.Design).ToList();

        // ASSERT
        Assert.Equal(2, engineeringHistory.Count);
        Assert.Single(designHistory);
    }

    [Fact]
    public void ContractCountForSkill_ReturnsCorrectCount()
    {
        // ═══════════════════════════════════════════════════════════════
        // EQUATION: |h⟨t⟩| = number of contracts for skill type t
        // ═══════════════════════════════════════════════════════════════
        
        // ARRANGE
        var agent = new Agent { SkillType = SkillTypes.Engineering };
        for (int i = 0; i < 7; i++)
            agent.AddToHistory(ContractFactory.CreateSimple(SkillTypes.Engineering, 1.0, 1.0));
        for (int i = 0; i < 3; i++)
            agent.AddToHistory(ContractFactory.CreateSimple(SkillTypes.Design, 1.0, 1.0));

        // ACT & ASSERT
        Assert.Equal(7, agent.ContractCountForSkill(SkillTypes.Engineering));
        Assert.Equal(3, agent.ContractCountForSkill(SkillTypes.Design));
        Assert.Equal(0, agent.ContractCountForSkill(SkillTypes.Legal));
    }
}

```

---

## 7. Trust Evolution Tests

Tests for incremental trust evolution.

**Mathematical Notation:**

```
┌─────────────────────────────────────────────────────────────────────
│  V⟨t⟩^(n+1)(a) = V⟨t⟩^(n)(a) + ω(c⟨n⟩) · outcome(c⟨n⟩)                 │
└─────────────────────────────────────────────────────────────────────┘
```

```csharp

/// <summary>
/// Tests for incremental trust evolution.
/// 
/// Mathematical notation:
/// ┌─────────────────────────────────────────────────────────────────────
/// │  V⟨t⟩^(n+1)(a) = V⟨t⟩^(n)(a) + ω(c⟨n⟩) · outcome(c⟨n⟩)                 │
/// └─────────────────────────────────────────────────────────────────────┘
/// 
/// Trust updates incrementally with each contract completion.
/// </summary>
public class TrustEvolutionTests
{
    [Fact]
    public void UpdateTrust_AddsWeightedOutcome()
    {
        // ═══════════════════════════════════════════════════════════════
        // EQUATION: V⟨t⟩^(n+1) = V⟨t⟩^(n) + ω(c⟨n⟩) · outcome(c⟨n⟩)
        // 
        // Starting from V⟨t⟩^(0) = 0:
        //   V⟨t⟩^(1) = 0 + (2.0 × 1.0) = 2.0
        // ═══════════════════════════════════════════════════════════════
        
        // ARRANGE
        var tracker = new TrustTracker();
        var agent = new Agent { SkillType = SkillTypes.Engineering };
        var contract = ContractFactory.CreateSimple(
            SkillTypes.Engineering, 
            outcome: 1.0, 
            weight: 2.0
        );
        var now = DateTime.UtcNow;

        // ACT
        var newTrust = tracker.UpdateTrust(agent, contract, now);

        // ASSERT
        Assert.Equal(2.0, newTrust);
        Assert.Equal(2.0, tracker.GetTrust(agent, SkillTypes.Engineering));
    }

    [Fact]
    public void UpdateTrust_AccumulatesOverMultipleContracts()
    {
        // ═══════════════════════════════════════════════════════════════
        // EQUATION: V⟨t⟩^(n) = Σ_{i=1}^{n} ω(cᵢ) · outcome(cᵢ)
        // 
        // V⟨t⟩^(0) = 0
        // V⟨t⟩^(1) = 0 + (1.0 × 1.0) = 1.0
        // V⟨t⟩^(2) = 1.0 + (2.0 × 0.5) = 2.0
        // V⟨t⟩^(3) = 2.0 + (1.5 × -1.0) = 0.5
        // ═══════════════════════════════════════════════════════════════
        
        // ARRANGE
        var tracker = new TrustTracker();
        var agent = new Agent { SkillType = SkillTypes.Engineering };
        var now = DateTime.UtcNow;

        var contracts = new[]
        {
            ContractFactory.CreateSimple(SkillTypes.Engineering, outcome: 1.0, weight: 1.0),
            ContractFactory.CreateSimple(SkillTypes.Engineering, outcome: 0.5, weight: 2.0),
            ContractFactory.CreateSimple(SkillTypes.Engineering, outcome: -1.0, weight: 1.5),
        };

        // ACT
        double trust = 0;
        foreach (var c in contracts)
            trust = tracker.UpdateTrust(agent, c, now);

        // ASSERT
        Assert.Equal(0.5, trust, precision: 10);
    }

    [Fact]
    public void UpdateTrust_CreatesAuditLog()
    {
        // ═══════════════════════════════════════════════════════════════
        // PROPERTY: All trust changes are auditable
        // ═══════════════════════════════════════════════════════════════
        
        // ARRANGE
        var tracker = new TrustTracker();
        var agent = new Agent { SkillType = SkillTypes.Engineering };
        var contract = ContractFactory.CreateSimple(SkillTypes.Engineering, 1.0, 5.0);
        var now = DateTime.UtcNow;

        // ACT
        tracker.UpdateTrust(agent, contract, now);

        // ASSERT
        Assert.Single(tracker.AuditLog);
        var entry = tracker.AuditLog[0];
        Assert.Equal(agent.Id, entry.AgentId);
        Assert.Equal(0.0, entry.PreviousTrust);
        Assert.Equal(5.0, entry.NewTrust);
        Assert.Equal(5.0, entry.Delta);
    }

    [Fact]
    public void AddContractAndUpdateTrust_UpdatesBothHistoryAndTrust()
    {
        // ═══════════════════════════════════════════════════════════════
        // COMBINED OPERATION:
        //   h⟨t⟩^(n+1) = h⟨t⟩^(n) ∪ {c⟨n⟩}
        //   V⟨t⟩^(n+1) = V⟨t⟩^(n) + ω(c⟨n⟩) · outcome(c⟨n⟩)
        // ═══════════════════════════════════════════════════════════════
        
        // ARRANGE
        var tracker = new TrustTracker();
        var agent = new Agent { SkillType = SkillTypes.Engineering };
        var contract = ContractFactory.CreateSimple(SkillTypes.Engineering, 1.0, 3.0);
        var now = DateTime.UtcNow;

        // ACT
        var newTrust = tracker.AddContractAndUpdateTrust(agent, contract, now);

        // ASSERT: Both history and cached trust updated
        Assert.Single(agent.ContractHistory);
        Assert.Equal(3.0, newTrust);
        Assert.Equal(3.0, tracker.GetTrust(agent, SkillTypes.Engineering));
    }
}

```

---

## 8. Eligibility Function Tests

Tests for the eligibility function.

**Mathematical Notation:**

```
┌─────────────────────────────────────────────────────────────────────
│  eligible(a, c) ⟺ V⟨t⟩(a) ≥ θ(c)                                   │
│                                                                     │
│  θ(c) = max(ln(1+s) × 0.1, ln(1+s) × d)                            │
└─────────────────────────────────────────────────────────────────────┘
```

```csharp

/// <summary>
/// Tests for the eligibility function.
/// 
/// Mathematical notation:
/// ┌─────────────────────────────────────────────────────────────────────
/// │  eligible(a, c) ⟺ V⟨t⟩(a) ≥ θ(c)                                   │
/// │                                                                     │
/// │  θ(c) = max(ln(1+s) × 0.1, ln(1+s) × d)                            │
/// └─────────────────────────────────────────────────────────────────────┘
/// 
/// An agent is eligible for a contract iff their trust meets the threshold.
/// </summary>
public class EligibilityFunctionTests
{
    [Fact]
    public void IsEligible_TrustMeetsThreshold_ReturnsTrue()
    {
        // ═══════════════════════════════════════════════════════════════
        // EQUATION: eligible(a, c) ⟺ V⟨t⟩(a) ≥ θ(c)
        // 
        // V⟨t⟩(agent) = 50.0
        // θ(contract) = ln(1+1000) × 5 ≈ 6.9 × 5 = 34.5
        // 50.0 ≥ 34.5 → eligible = true
        // ═══════════════════════════════════════════════════════════════
        
        // ARRANGE
        var agent = new Agent { SkillType = SkillTypes.Engineering };
        agent.AddToHistory(ContractFactory.CreateSimple(SkillTypes.Engineering, 1.0, 50.0));

        var contract = ContractFactory.CreateSimple(
            SkillTypes.Engineering,
            outcome: 0,
            weight: 0,
            stake: 1000,
            difficulty: 5
        );

        // ACT
        var isEligible = EligibilityChecker.IsEligible(agent, contract);

        // ASSERT
        Assert.True(isEligible);
    }

    [Fact]
    public void IsEligible_TrustBelowThreshold_ReturnsFalse()
    {
        // ═══════════════════════════════════════════════════════════════
        // EQUATION: ¬eligible(a, c) ⟺ V⟨t⟩(a) < θ(c)
        // 
        // V⟨t⟩(agent) = 10.0
        // θ(contract) = ln(1+10000) × 8 ≈ 9.2 × 8 = 73.6
        // 10.0 < 73.6 → eligible = false
        // ═══════════════════════════════════════════════════════════════
        
        // ARRANGE
        var agent = new Agent { SkillType = SkillTypes.Engineering };
        agent.AddToHistory(ContractFactory.CreateSimple(SkillTypes.Engineering, 1.0, 10.0));

        var contract = ContractFactory.CreateSimple(
            SkillTypes.Engineering,
            outcome: 0,
            weight: 0,
            stake: 10000,
            difficulty: 8
        );

        // ACT
        var isEligible = EligibilityChecker.IsEligible(agent, contract);

        // ASSERT
        Assert.False(isEligible);
    }

    [Theory]
    [InlineData(0, 0, 0.0)]                    // θ = max(0, 0) = 0
    [InlineData(100, 0, 0.461)]                // θ = max(ln(101)×0.1, 0) ≈ 0.461
    [InlineData(100, 5, 23.08)]                // θ = max(0.461, ln(101)×5) ≈ 23.08
    [InlineData(1000, 3, 20.73)]               // θ = ln(1001)×3 ≈ 20.73
    public void CalculateThreshold_ReturnsCorrectValue(
        double stake, 
        double difficulty, 
        double expected)
    {
        // ═══════════════════════════════════════════════════════════════
        // EQUATION: θ(c) = max(ln(1+s) × 0.1, ln(1+s) × d)
        // 
        // The minimum threshold factor (0.1) ensures even zero-difficulty
        // contracts have some barrier to entry.
        // ═══════════════════════════════════════════════════════════════
        
        // ARRANGE
        var contract = ContractFactory.CreateSimple(
            SkillTypes.Engineering,
            outcome: 0,
            weight: 0,
            stake: stake,
            difficulty: difficulty
        );

        // ACT
        var actual = EligibilityChecker.CalculateThreshold(contract);

        // ASSERT
        Assert.Equal(expected, actual, precision: 2);
    }

    [Fact]
    public void IsEligible_ZeroTrustZeroThreshold_ReturnsTrue()
    {
        // ═══════════════════════════════════════════════════════════════
        // EDGE CASE: V⟨t⟩ = 0, θ = 0
        // 
        // 0 ≥ 0 → true (newcomer can take zero-stake contracts)
        // ═══════════════════════════════════════════════════════════════
        
        // ARRANGE
        var agent = new Agent { SkillType = SkillTypes.Engineering };  // No history
        var contract = ContractFactory.CreateSimple(
            SkillTypes.Engineering, 0, 0, stake: 0, difficulty: 0);

        // ACT
        var isEligible = EligibilityChecker.IsEligible(agent, contract);

        // ASSERT
        Assert.True(isEligible);
    }

    [Fact]
    public void IsEligible_NegativeTrust_ReturnsFalse()
    {
        // ═══════════════════════════════════════════════════════════════
        // EDGE CASE: V⟨t⟩ < 0 (distrusted agent)
        // 
        // -10 < any positive threshold → never eligible
        // ═══════════════════════════════════════════════════════════════
        
        // ARRANGE
        var agent = new Agent { SkillType = SkillTypes.Engineering };
        agent.AddToHistory(ContractFactory.CreateSimple(SkillTypes.Engineering, -1.0, 10.0));

        var contract = ContractFactory.CreateSimple(
            SkillTypes.Engineering, 0, 0, stake: 100, difficulty: 1);

        // ACT
        var isEligible = EligibilityChecker.IsEligible(agent, contract);

        // ASSERT
        Assert.False(isEligible);
    }

    [Fact]
    public void GetEligibleContracts_FiltersCorrectly()
    {
        // ═══════════════════════════════════════════════════════════════
        // FILTER: {c : eligible(a, c)}
        // ═══════════════════════════════════════════════════════════════
        
        // ARRANGE: Agent with trust = 30
        var agent = new Agent { SkillType = SkillTypes.Engineering };
        agent.AddToHistory(ContractFactory.CreateSimple(SkillTypes.Engineering, 1.0, 30.0));

        var contracts = new[]
        {
            ContractFactory.CreateSimple(SkillTypes.Engineering, 0, 0, stake: 10, difficulty: 1),    // θ≈2.4, eligible
            ContractFactory.CreateSimple(SkillTypes.Engineering, 0, 0, stake: 100, difficulty: 5),  // θ≈23, eligible
            ContractFactory.CreateSimple(SkillTypes.Engineering, 0, 0, stake: 1000, difficulty: 8), // θ≈55, not eligible
        };

        // ACT
        var eligible = agent.GetEligibleContracts(contracts).ToList();

        // ASSERT
        Assert.Equal(2, eligible.Count);
    }
}

```

---

## 9. Sybil Resistance Tests

Tests for Sybil resistance properties.

**Mathematical Notation:**

```
┌─────────────────────────────────────────────────────────────────────
│  |h⟨t⟩(a_honest)| > |h⟨t⟩(a_sybil_i)|   ∀i                           │
└─────────────────────────────────────────────────────────────────────┘
```

```csharp

/// <summary>
/// Tests for Sybil resistance properties.
/// 
/// Mathematical notation:
/// ┌─────────────────────────────────────────────────────────────────────
/// │  |h⟨t⟩(a_honest)| > |h⟨t⟩(a_sybil_i)|   ∀i                           │
/// └─────────────────────────────────────────────────────────────────────┘
/// 
/// Splitting activity across k fake identities results in each having
/// ~1/k the history of an honest single-identity agent.
/// </summary>
public class SybilResistanceTests
{
    [Fact]
    public void AnalyzeSybilResistance_HonestHasMoreHistory()
    {
        // ═══════════════════════════════════════════════════════════════
        // EQUATION: |h⟨t⟩(a_honest)| > |h⟨t⟩(a_sybil_i)|  ∀i
        // 
        // Scenario:
        //   Honest agent: 100 contracts
        //   Attacker splits across 5 sybils: 20 contracts each
        // 
        // 100 > 20 for all sybils → Sybil resistant
        // ═══════════════════════════════════════════════════════════════
        
        // ARRANGE
        var honest = new Agent { SkillType = SkillTypes.Engineering };
        for (int i = 0; i < 100; i++)
            honest.AddToHistory(ContractFactory.CreateSimple(SkillTypes.Engineering, 1.0, 1.0));

        var sybils = Enumerable.Range(0, 5)
            .Select(_ =>
            {
                var s = new Agent { SkillType = SkillTypes.Engineering };
                for (int i = 0; i < 20; i++)
                    s.AddToHistory(ContractFactory.CreateSimple(SkillTypes.Engineering, 1.0, 1.0));
                return s;
            })
            .ToList();

        // ACT
        var result = SybilResistanceAnalyzer.AnalyzeSybilResistance(
            honest, sybils, SkillTypes.Engineering);

        // ASSERT
        Assert.True(result.IsSybilResistant);
        Assert.Equal(100, result.HonestHistorySize);
        Assert.All(result.SybilHistorySizes, size => Assert.Equal(20, size));
    }

    [Fact]
    public void AnalyzeSybilResistance_HonestHasHigherTrust()
    {
        // ═══════════════════════════════════════════════════════════════
        // CONSEQUENCE: V⟨t⟩(honest) > V⟨t⟩(sybil_i)  ∀i
        // 
        // More history with same success rate → more trust
        // 
        // V⟨t⟩(honest) = 100 × 1.0 × 1.0 = 100
        // V⟨t⟩(sybil) = 20 × 1.0 × 1.0 = 20
        // Advantage ratio = 100/20 = 5.0
        // ═══════════════════════════════════════════════════════════════
        
        // ARRANGE
        var honest = new Agent { SkillType = SkillTypes.Engineering };
        for (int i = 0; i < 100; i++)
            honest.AddToHistory(ContractFactory.CreateSimple(SkillTypes.Engineering, 1.0, 1.0));

        var sybils = Enumerable.Range(0, 5)
            .Select(_ =>
            {
                var s = new Agent { SkillType = SkillTypes.Engineering };
                for (int i = 0; i < 20; i++)
                    s.AddToHistory(ContractFactory.CreateSimple(SkillTypes.Engineering, 1.0, 1.0));
                return s;
            })
            .ToList();

        // ACT
        var result = SybilResistanceAnalyzer.AnalyzeSybilResistance(
            honest, sybils, SkillTypes.Engineering);

        // ASSERT
        Assert.Equal(100.0, result.HonestTrust);
        Assert.All(result.SybilTrusts, trust => Assert.Equal(20.0, trust));
        Assert.Equal(5.0, result.HonestAdvantageRatio, precision: 2);
    }

    [Theory]
    [InlineData(100, 0, double.PositiveInfinity)]   // Honest positive, sybils zero
    [InlineData(0, 0, 1.0)]                          // Both zero
    [InlineData(100, 50, 2.0)]                       // Both positive
    [InlineData(50, -50, double.PositiveInfinity)]   // Honest positive, sybils negative
    [InlineData(-10, -20, 2.0)]                      // Both negative (less negative = better)
    [InlineData(-50, 50, 0.0)]                       // Honest negative, sybils positive
    public void CalculateAdvantageRatio_HandlesEdgeCases(
        double honestTrust, 
        double maxSybilTrust, 
        double expected)
    {
        // ═══════════════════════════════════════════════════════════════
        // EQUATION: advantage_ratio = V⟨t⟩(honest) / V⟨t⟩(best_sybil)
        // 
        // With special handling for:
        //   - Zero denominators
        //   - Negative trust values
        //   - Mixed signs
        // ═══════════════════════════════════════════════════════════════
        
        // ACT
        var actual = SybilResistanceAnalyzer.CalculateAdvantageRatio(honestTrust, maxSybilTrust);

        // ASSERT
        Assert.Equal(expected, actual, precision: 2);
    }

    [Fact]
    public void AnalyzeSybilResistance_EmptySybilList_NotResistant()
    {
        // ═══════════════════════════════════════════════════════════════
        // EDGE CASE: No sybils to compare against
        // 
        // Cannot claim "resistant" if there's no attack to resist.
        // ═══════════════════════════════════════════════════════════════
        
        // ARRANGE
        var honest = new Agent { SkillType = SkillTypes.Engineering };
        honest.AddToHistory(ContractFactory.CreateSimple(SkillTypes.Engineering, 1.0, 10.0));

        // ACT
        var result = SybilResistanceAnalyzer.AnalyzeSybilResistance(
            honest, new List<Agent>(), SkillTypes.Engineering);

        // ASSERT
        Assert.False(result.IsSybilResistant);
    }
}

```

---

## 10. Convergence Criterion Tests

Tests for network validation through convergence.

**Mathematical Notation:**

```
┌─────────────────────────────────────────────────────────────────────
│  lim(n→∞) Corr(V⟨t⟩^(n)(a), R⟨t⟩(a)) = 1                             │
└─────────────────────────────────────────────────────────────────────┘
```

```csharp

/// <summary>
/// Tests for network validation through convergence.
/// 
/// Mathematical notation:
/// ┌─────────────────────────────────────────────────────────────────────
/// │  lim(n→∞) Corr(V⟨t⟩^(n)(a), R⟨t⟩(a)) = 1                             │
/// └─────────────────────────────────────────────────────────────────────┘
/// 
/// As history accumulates, the correlation between computed trust
/// and actual reliability should approach 1.0.
/// </summary>
public class ConvergenceCriterionTests
{
    [Fact]
    public void CalculateConvergence_PerfectCorrelation_ReturnsOne()
    {
        // ═══════════════════════════════════════════════════════════════
        // EQUATION: Corr(V⟨t⟩, R⟨t⟩) = 1 when V⟨t⟩  R⟨t⟩
        // 
        // If trust perfectly reflects reliability:
        //   Agent with R=0.9 gets V~90
        //   Agent with R=0.5 gets V~50
        //   → Correlation = 1.0
        // ═══════════════════════════════════════════════════════════════
        
        // ARRANGE: Agents where trust = reliability × 100
        var agents = new List<SimulatedAgent>();
        foreach (var reliability in new[] { 0.1, 0.3, 0.5, 0.7, 0.9 })
        {
            var agent = new SimulatedAgent
            {
                SkillType = SkillTypes.Engineering,
                ActualReliability = reliability
            };
            // Trust = reliability × 100 (perfect linear relationship)
            agent.AddToHistory(ContractFactory.CreateSimple(
                SkillTypes.Engineering, 
                outcome: 1.0, 
                weight: reliability * 100));
            agents.Add(agent);
        }

        // ACT
        var correlation = NetworkValidator.CalculateConvergence(
            agents,
            (a, s) => (a as SimulatedAgent)?.ActualReliability ?? 0,
            SkillTypes.Engineering);

        // ASSERT
        Assert.Equal(1.0, correlation, precision: 5);
    }

    [Fact]
    public void CalculateConvergence_NoCorrelation_ReturnsNearZero()
    {
        // ═══════════════════════════════════════════════════════════════
        // EQUATION: Corr(V⟨t⟩, R⟨t⟩) ≈ 0 when V⟨t⟩ and R⟨t⟩ are independent
        // ═══════════════════════════════════════════════════════════════
        
        // ARRANGE: Random trust values unrelated to reliability
        var random = new Random(42);
        var agents = new List<SimulatedAgent>();
        for (int i = 0; i < 100; i++)
        {
            var agent = new SimulatedAgent
            {
                SkillType = SkillTypes.Engineering,
                ActualReliability = random.NextDouble()
            };
            // Random trust unrelated to reliability
            agent.AddToHistory(ContractFactory.CreateSimple(
                SkillTypes.Engineering,
                outcome: random.NextDouble() * 2 - 1,
                weight: random.NextDouble() * 10));
            agents.Add(agent);
        }

        // ACT
        var correlation = NetworkValidator.CalculateConvergence(
            agents,
            (a, s) => (a as SimulatedAgent)?.ActualReliability ?? 0,
            SkillTypes.Engineering);

        // ASSERT: Should be near zero (allowing some random correlation)
        Assert.True(Math.Abs(correlation) < 0.3, 
            $"Expected near-zero correlation, got {correlation}");
    }

    [Fact]
    public void CalculateConvergence_TooFewAgents_ReturnsNaN()
    {
        // ═══════════════════════════════════════════════════════════════
        // EDGE CASE: Pearson correlation requires n ≥ 2
        // ═══════════════════════════════════════════════════════════════
        
        // ARRANGE
        var singleAgent = new List<Agent> { new Agent { SkillType = SkillTypes.Engineering } };

        // ACT
        var correlation = NetworkValidator.CalculateConvergence(
            singleAgent,
            (a, s) => 0.5,
            SkillTypes.Engineering);

        // ASSERT
        Assert.True(double.IsNaN(correlation));
    }

    [Fact]
    public void IsValidated_HighCorrelation_ReturnsTrue()
    {
        // ═══════════════════════════════════════════════════════════════
        // VALIDATION: Corr ≥ ValidationThreshold (0.95)
        // ═══════════════════════════════════════════════════════════════
        
        Assert.True(NetworkValidator.IsValidated(0.95));
        Assert.True(NetworkValidator.IsValidated(0.99));
        Assert.True(NetworkValidator.IsValidated(1.0));
    }

    [Fact]
    public void IsValidated_LowCorrelation_ReturnsFalse()
    {
        Assert.False(NetworkValidator.IsValidated(0.94));
        Assert.False(NetworkValidator.IsValidated(0.5));
        Assert.False(NetworkValidator.IsValidated(0.0));
        Assert.False(NetworkValidator.IsValidated(-0.5));
        Assert.False(NetworkValidator.IsValidated(double.NaN));
    }

    [Fact]
    public void SimulateAndValidate_ConvergesOverTime()
    {
        // ═══════════════════════════════════════════════════════════════
        // SIMULATION: As n→∞, Corr(V⟨t⟩, R⟨t⟩)→1
        // 
        // With enough contracts, trust scores should reflect actual
        // reliability due to the law of large numbers.
        // ═══════════════════════════════════════════════════════════════
        
        // ARRANGE
        var random = new Random(123);
        var agents = TrustTestHelpers.CreateSimulatedAgents(20, SkillTypes.Engineering, random);

        // ACT: Simulate many contracts
        TrustTestHelpers.SimulateContracts(agents, 200, SkillTypes.Engineering, 1.0, random);

        var correlation = NetworkValidator.CalculateConvergence(
            agents,
            (a, s) => (a as SimulatedAgent)?.ActualReliability ?? 0,
            SkillTypes.Engineering);

        // ASSERT: Should have high correlation after many contracts
        Assert.True(correlation > 0.7, 
            $"Expected high correlation after simulation, got {correlation}");
    }
}

```

---

## 11. Contract Validation Tests

Tests for contract structure and validation.

**Mathematical Notation:**

```
┌─────────────────────────────────────────────────────────────────────
│  c = (a_provider, a_consumer, t, s, d, τ)                          │
│                                                                     │
│  Constraints:                                                       │
│    s ≥ 0              (stake is non-negative)                      │
│    d ∈ [0, 10]        (difficulty in valid range)                  │
│    outcome ∈ [-1, 1]  (outcome in valid range)                     │
└─────────────────────────────────────────────────────────────────────┘
```

```csharp

/// <summary>
/// Tests for contract structure and validation.
/// 
/// Mathematical notation:
/// ┌─────────────────────────────────────────────────────────────────────
/// │  c = (a_provider, a_consumer, t, s, d, τ)                          │
/// │                                                                     │
/// │  Constraints:                                                       │
/// │    s ≥ 0              (stake is non-negative)                      │
/// │    d ∈ [0, 10]        (difficulty in valid range)                  │
/// │    outcome ∈ [-1, 1]  (outcome in valid range)                     │
/// └─────────────────────────────────────────────────────────────────────┘
/// </summary>
public class ContractValidationTests
{
    [Fact]
    public void Contract_ValidValues_IsValid()
    {
        // ═══════════════════════════════════════════════════════════════
        // VALID CONTRACT: All constraints satisfied
        // ═══════════════════════════════════════════════════════════════
        
        // ARRANGE & ACT
        var contract = ContractFactory.CreateSimple(
            skillType: SkillTypes.Engineering,
            outcome: 0.5,
            weight: 1.0,
            stake: 1000,
            difficulty: 5
        );

        // ASSERT
        Assert.True(contract.IsValid);
        Assert.Empty(contract.Validate());
    }

    [Fact]
    public void Contract_NegativeStake_Throws()
    {
        // ═══════════════════════════════════════════════════════════════
        // CONSTRAINT: s ≥ 0
        // ═══════════════════════════════════════════════════════════════
        
        Assert.Throws<ArgumentOutOfRangeException>(() => new Contract
        {
            SkillType = SkillTypes.Engineering,
            Stake = -100,  // Invalid
            Difficulty = 5,
            Outcome = 0,
            Deadline = DateTime.UtcNow.AddDays(30)
        });
    }

    [Theory]
    [InlineData(-0.1)]
    [InlineData(10.1)]
    [InlineData(-5)]
    [InlineData(15)]
    public void Contract_InvalidDifficulty_Throws(double difficulty)
    {
        // ═══════════════════════════════════════════════════════════════
        // CONSTRAINT: d ∈ [0, 10]
        // ═══════════════════════════════════════════════════════════════
        
        Assert.Throws<ArgumentOutOfRangeException>(() => new Contract
        {
            SkillType = SkillTypes.Engineering,
            Stake = 100,
            Difficulty = difficulty,  // Invalid
            Outcome = 0,
            Deadline = DateTime.UtcNow.AddDays(30)
        });
    }

    [Theory]
    [InlineData(-1.1)]
    [InlineData(1.1)]
    [InlineData(-5)]
    [InlineData(5)]
    public void Contract_InvalidOutcome_Throws(double outcome)
    {
        // ═══════════════════════════════════════════════════════════════
        // CONSTRAINT: outcome ∈ [-1, 1]
        // ═══════════════════════════════════════════════════════════════
        
        Assert.Throws<ArgumentOutOfRangeException>(() => new Contract
        {
            SkillType = SkillTypes.Engineering,
            Stake = 100,
            Difficulty = 5,
            Outcome = outcome,  // Invalid
            Deadline = DateTime.UtcNow.AddDays(30)
        });
    }

    [Fact]
    public void Contract_EmptySkillType_Throws()
    {
        // ═══════════════════════════════════════════════════════════════
        // CONSTRAINT: t ≠ ∅ (skill type required)
        // ═══════════════════════════════════════════════════════════════
        
        Assert.Throws<ArgumentException>(() => new Contract
        {
            SkillType = "",  // Invalid
            Stake = 100,
            Difficulty = 5,
            Outcome = 0,
            Deadline = DateTime.UtcNow.AddDays(30)
        });
    }

    [Fact]
    public void Contract_Create_ValidatesAllFields()
    {
        // ═══════════════════════════════════════════════════════════════
        // FACTORY: Contract.Create validates inputs
        // ═══════════════════════════════════════════════════════════════
        
        // ARRANGE
        var provider = new Agent { SkillType = SkillTypes.Engineering };
        var consumer = new Agent { SkillType = SkillTypes.Engineering };

        // ACT
        var contract = Contract.Create(
            provider: provider,
            consumer: consumer,
            skillType: SkillTypes.Engineering,
            stake: 5000,
            difficulty: 7,
            deadline: DateTime.UtcNow.AddDays(30),
            outcome: 0.8,
            weight: 2.5
        );

        // ASSERT
        Assert.Same(provider, contract.Provider);
        Assert.Same(consumer, contract.Consumer);
        Assert.Equal(SkillTypes.Engineering, contract.SkillType);
        Assert.Equal(5000, contract.Stake);
        Assert.Equal(7, contract.Difficulty);
        Assert.Equal(0.8, contract.Outcome);
        Assert.Equal(2.5, contract.Weight);
    }
}

```

---

## 12. SkillTypes Utility Tests

Tests for skill type normalization and comparison. Ensures case-insensitive, whitespace-normalized skill type handling.

```csharp

/// <summary>
/// Tests for skill type normalization and comparison.
/// 
/// Ensures case-insensitive, whitespace-normalized skill type handling.
/// </summary>
public class SkillTypesTests
{
    [Theory]
    [InlineData("Engineering", "engineering")]
    [InlineData("ENGINEERING", "engineering")]
    [InlineData("  engineering  ", "engineering")]
    [InlineData("Design", "design")]
    public void Normalize_ReturnsLowercaseTrimmed(string input, string expected)
    {
        // ACT
        var actual = SkillTypes.Normalize(input);

        // ASSERT
        Assert.Equal(expected, actual);
    }

    [Theory]
    [InlineData(null)]
    [InlineData("")]
    [InlineData("   ")]
    public void Normalize_InvalidInput_Throws(string? input)
    {
        Assert.Throws<ArgumentException>(() => SkillTypes.Normalize(input!));
    }

    [Theory]
    [InlineData("engineering", "ENGINEERING", true)]
    [InlineData("Design", "design", true)]
    [InlineData("engineering", "design", false)]
    public void AreEqual_CaseInsensitive(string a, string b, bool expected)
    {
        Assert.Equal(expected, SkillTypes.AreEqual(a, b));
    }

    [Theory]
    [InlineData("engineering", true)]
    [InlineData("", false)]
    [InlineData(null, false)]
    [InlineData("   ", false)]
    public void IsValid_ChecksForContent(string? input, bool expected)
    {
        Assert.Equal(expected, SkillTypes.IsValid(input!));
    }
}

```

---

## 13. TrustValuation Static Helper Tests

Tests for the TrustValuation.V() static helper function.

**Mathematical Notation:**

```
┌─────────────────────────────────────────────────────────────────────
│  V⟨t⟩: q⟨T⟩ →                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

```csharp

/// <summary>
/// Tests for the TrustValuation.V() static helper function.
/// 
/// Mathematical notation:
/// ┌─────────────────────────────────────────────────────────────────────
/// │  V⟨t⟩: q⟨T⟩ →                                                      │
/// └─────────────────────────────────────────────────────────────────────┘
/// </summary>
public class TrustValuationTests
{
    [Fact]
    public void V_Agent_ReturnsTrustValue()
    {
        // ═══════════════════════════════════════════════════════════════
        // EQUATION: V⟨t⟩(Agent) = computed trust value
        // ═══════════════════════════════════════════════════════════════
        
        // ARRANGE
        var agent = new Agent { SkillType = SkillTypes.Engineering };
        agent.AddToHistory(ContractFactory.CreateSimple(SkillTypes.Engineering, 1.0, 25.0));

        // ACT
        var trust = TrustValuation.V(agent, SkillTypes.Engineering);

        // ASSERT
        Assert.Equal(25.0, trust);
    }

    [Fact]
    public void V_DAO_ReturnsAggregatedTrust()
    {
        // ═══════════════════════════════════════════════════════════════
        // EQUATION: V⟨t⟩(DAO) = Φ({member trusts})
        // ═══════════════════════════════════════════════════════════════
        
        // ARRANGE
        var agent1 = new Agent { SkillType = SkillTypes.Engineering };
        agent1.AddToHistory(ContractFactory.CreateSimple(SkillTypes.Engineering, 1.0, 10.0));
        
        var agent2 = new Agent { SkillType = SkillTypes.Engineering };
        agent2.AddToHistory(ContractFactory.CreateSimple(SkillTypes.Engineering, 1.0, 20.0));

        var dao = new DAO
        {
            Members = new HashSet<QuantumOfTrust> { agent1, agent2 },
            Phi = AggregationFunctions.Sum
        };

        // ACT
        var trust = TrustValuation.V(dao, SkillTypes.Engineering);

        // ASSERT
        Assert.Equal(30.0, trust);
    }

    [Fact]
    public void V_NullEntity_Throws()
    {
        Assert.Throws<ArgumentNullException>(() => 
            TrustValuation.V(null!, SkillTypes.Engineering));
    }

    [Fact]
    public void V_InvalidSkillType_ReturnsZero()
    {
        var agent = new Agent { SkillType = SkillTypes.Engineering };
        agent.AddToHistory(ContractFactory.CreateSimple(SkillTypes.Engineering, 1.0, 10.0));

        Assert.Equal(0.0, TrustValuation.V(agent, ""));
        Assert.Equal(0.0, TrustValuation.V(agent, null!));
    }
}

```

---

## Dependencies

```csharp
using System;
using System.Collections.Generic;
using System.Linq;
using Xunit;

namespace QuantumOfTrust.Tests;
```

---

*Generated from QuantumOfTrustTests.cs - Quantum of Trust Framework*