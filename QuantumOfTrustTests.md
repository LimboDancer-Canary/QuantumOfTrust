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
14. [Customer Trust Calculation Tests](#14-customer-trust-calculation-tests)
15. [Verification Weight Tests](#15-verification-weight-tests)
16. [Task Decomposition Tests](#16-task-decomposition-tests)
17. [Customer Profile Tests](#17-customer-profile-tests)
18. [Hierarchical Contract Tests](#18-hierarchical-contract-tests)
19. [Milestone Tests](#19-milestone-tests)

---

## 1. Agent Trust Value Calculation Tests

Tests for the Agent trust valuation function.

**Mathematical Notation:**

```
â”Œâ”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€
â”‚  VâŸ¨tâŸ©(Agent(t, hâŸ¨tâŸ©)) = Î£ Ï‰(c) Â· outcome(c)   for all c âˆˆ hâŸ¨tâŸ©       â”‚
â””â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”˜
```

```csharp

/// <summary>
/// Tests for the Agent trust valuation function.
/// 
/// Mathematical notation:
/// â”Œâ”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€
/// â”‚  VâŸ¨tâŸ©(Agent(t, hâŸ¨tâŸ©)) = Î£ Ï‰(c) Â· outcome(c)   for all c âˆˆ hâŸ¨tâŸ©       â”‚
/// â””â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”˜
/// 
/// Where:
///   VâŸ¨tâŸ©     = Trust value for skill type t
///   hâŸ¨tâŸ©     = History of contracts for skill type t
///   Ï‰(c)    = Weight of contract c
///   outcome(c) = Result of contract c âˆˆ [-1, 1]
/// 
/// Properties:
///   VâŸ¨tâŸ© = 0   â†’ Unknown, no track record
///   VâŸ¨tâŸ© > 0   â†’ Net positive history, trusted
///   VâŸ¨tâŸ© < 0   â†’ Net negative history, actively distrusted
/// </summary>
public class AgentTrustValueTests
{
    [Fact]
    public void ComputeTrustValue_EmptyHistory_ReturnsZero()
    {
        // â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•
        // EQUATION: VâŸ¨tâŸ©(Agent(t, âˆ…)) = Î£_{c âˆˆ âˆ…} Ï‰(c)Â·outcome(c) = 0
        // 
        // Mathematical identity: Sum over empty set equals zero
        // â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•
        
        // ARRANGE: hâŸ¨tâŸ© = âˆ… (empty history)
        var agent = new Agent { SkillType = SkillTypes.Engineering };
        var expected = 0.0;

        // ACT: VâŸ¨tâŸ©(agent)
        var actual = agent.ComputeTrustValue(SkillTypes.Engineering);

        // ASSERT: VâŸ¨tâŸ© = 0 (unknown/no track record)
        Assert.Equal(expected, actual);
    }

    [Fact]
    public void ComputeTrustValue_SingleSuccessfulContract_ReturnsPositive()
    {
        // â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•
        // EQUATION: VâŸ¨tâŸ© = Ï‰(c) Â· outcome(c)
        // 
        // When |hâŸ¨tâŸ©| = 1:
        //   VâŸ¨tâŸ© = 2.0 Ã— 1.0 = 2.0
        // â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•
        
        // ARRANGE: hâŸ¨tâŸ© = {c} where Ï‰(c)=2.0, outcome(c)=1.0
        var agent = new Agent { SkillType = SkillTypes.Engineering };
        var contract = ContractFactory.CreateSimple(
            skillType: SkillTypes.Engineering,
            outcome: 1.0,    // Success
            weight: 2.0
        );
        agent.AddToHistory(contract);
        
        var expected = 2.0;  // Ï‰ Ã— outcome = 2.0 Ã— 1.0

        // ACT
        var actual = agent.ComputeTrustValue(SkillTypes.Engineering);

        // ASSERT: VâŸ¨tâŸ© > 0 (trusted)
        Assert.Equal(expected, actual);
        Assert.True(actual > 0, "Positive outcome should yield positive trust");
    }

    [Fact]
    public void ComputeTrustValue_SingleFailedContract_ReturnsNegative()
    {
        // â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•
        // EQUATION: VâŸ¨tâŸ© = Ï‰(c) Â· outcome(c)
        // 
        // When outcome = -1 (failure):
        //   VâŸ¨tâŸ© = 3.0 Ã— (-1.0) = -3.0
        // â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•
        
        // ARRANGE: hâŸ¨tâŸ© = {c} where Ï‰(c)=3.0, outcome(c)=-1.0
        var agent = new Agent { SkillType = SkillTypes.Engineering };
        var contract = ContractFactory.CreateSimple(
            skillType: SkillTypes.Engineering,
            outcome: -1.0,   // Failure
            weight: 3.0
        );
        agent.AddToHistory(contract);
        
        var expected = -3.0;  // Ï‰ Ã— outcome = 3.0 Ã— (-1.0)

        // ACT
        var actual = agent.ComputeTrustValue(SkillTypes.Engineering);

        // ASSERT: VâŸ¨tâŸ© < 0 (actively distrusted)
        Assert.Equal(expected, actual);
        Assert.True(actual < 0, "Negative outcome should yield negative trust");
    }

    [Fact]
    public void ComputeTrustValue_MixedOutcomes_ReturnsSumOfWeightedOutcomes()
    {
        // â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•
        // EQUATION: VâŸ¨tâŸ© = Î£_{c âˆˆ hâŸ¨tâŸ©} Ï‰(c) Â· outcome(c)
        // 
        // hâŸ¨tâŸ© = {c, câ‚‚, câ‚ƒ, câ‚„}
        // 
        // Manual calculation:
        //   c: Ï‰=1.0, outcome=+1.0  â†’  +1.0
        //   câ‚‚: Ï‰=0.8, outcome=+1.0  â†’  +0.8
        //   câ‚ƒ: Ï‰=1.0, outcome=-1.0  â†’  -1.0
        //   câ‚„: Ï‰=0.5, outcome= 0.0  â†’   0.0
        //                            â”€â”€â”€â”€â”€â”€â”€â”€
        //   VâŸ¨tâŸ© = 1.0 + 0.8 - 1.0 + 0.0 = 0.8
        // â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•
        
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
        // â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•
        // EQUATION: outcome(c) âˆˆ [-1, 1] (continuous range)
        // 
        // Testing partial success/failure values:
        //   c: Ï‰=2.0, outcome=+0.5  â†’  +1.0
        //   câ‚‚: Ï‰=2.0, outcome=-0.3  â†’  -0.6
        //                            â”€â”€â”€â”€â”€â”€â”€â”€
        //   VâŸ¨tâŸ© = 1.0 - 0.6 = 0.4
        // â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•
        
        // ARRANGE
        var agent = new Agent { SkillType = SkillTypes.Design };
        agent.AddToHistory(ContractFactory.CreateSimple(SkillTypes.Design, outcome: 0.5, weight: 2.0));
        agent.AddToHistory(ContractFactory.CreateSimple(SkillTypes.Design, outcome: -0.3, weight: 2.0));

        var expected = 0.4;  // (2.0 Ã— 0.5) + (2.0 Ã— -0.3) = 1.0 - 0.6

        // ACT
        var actual = agent.ComputeTrustValue(SkillTypes.Design);

        // ASSERT
        Assert.Equal(expected, actual, precision: 10);
    }

    [Fact]
    public void ComputeTrustValue_DifferentSkillTypes_AreIndependent()
    {
        // â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•
        // PROPERTY: Skill types are independently scoped
        // 
        // VâŸ¨eâŸ©ngineering(agent) is computed from hâŸ¨eâŸ©ngineering only
        // VâŸ¨dâŸ©esign(agent) is computed from hâŸ¨dâŸ©esign only
        // 
        // Jane's example from whitepaper:
        //   VâŸ¨eâŸ©ngineering = 85 cutes (thriving)
        //   VâŸ¨dâŸ©esign = -12 cutes (struggling)
        // â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•
        
        // ARRANGE: Two separate histories
        var agent = new Agent { SkillType = SkillTypes.Engineering };
        
        // Engineering contracts (10 successes Ã— weight 8.5)
        for (int i = 0; i < 10; i++)
            agent.AddToHistory(ContractFactory.CreateSimple(SkillTypes.Engineering, outcome: 1.0, weight: 8.5));
        
        // Design contracts (5 failures Ã— weight 2.4)
        for (int i = 0; i < 5; i++)
            agent.AddToHistory(ContractFactory.CreateSimple(SkillTypes.Design, outcome: -1.0, weight: 2.4));

        var expectedEngineering = 85.0;  // 10 Ã— 8.5 Ã— 1.0
        var expectedDesign = -12.0;       // 5 Ã— 2.4 Ã— (-1.0)

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
        // â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•
        // EDGE CASE: Invalid skill type input
        // 
        // VâŸ¨tâŸ©(agent, null) = 0
        // VâŸ¨tâŸ©(agent, "") = 0
        // â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•
        
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
        // â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•
        // EQUATION: VâŸ¨tâŸ© = Ï‰(c) Â· outcome(c)  when |hâŸ¨tâŸ©| = 1
        // 
        // This is the fundamental multiplication at the heart of trust.
        // â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•
        
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
â”Œâ”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€
â”‚  VâŸ¨tâŸ©(DAO(S)) = Î¦({VâŸ¨tâŸ©(q) : q âˆˆ S})                                 â”‚
â””â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”˜
```

```csharp

/// <summary>
/// Tests for DAO trust aggregation.
/// 
/// Mathematical notation:
/// â”Œâ”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€
/// â”‚  VâŸ¨tâŸ©(DAO(S)) = Î¦({VâŸ¨tâŸ©(q) : q âˆˆ S})                                 â”‚
/// â””â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”˜
/// 
/// Where:
///   S   = Set of member entities (Agents or DAOs)
///   Î¦   = Aggregation function (sum, avg, min, max, etc.)
///   q   = A member entity (q âˆˆ S)
/// 
/// Recursive property: A DAO is itself a qâŸ¨TâŸ©, enabling:
///   qâŸ¨TâŸ© ::= Agent(t, hâŸ¨tâŸ©) | DAO({qâŸ¨TâŸ©})
/// </summary>
public class DAOTrustAggregationTests
{
    [Fact]
    public void ComputeTrustValue_EmptyDAO_ReturnsZero()
    {
        // â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•
        // EQUATION: VâŸ¨tâŸ©(DAO(âˆ…)) = Î¦(âˆ…) = 0
        // 
        // All aggregation functions return 0 for empty sets.
        // â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•
        
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
        // â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•
        // EQUATION: Î¦_sum({V, Vâ‚‚, Vâ‚ƒ}) = V + Vâ‚‚ + Vâ‚ƒ
        // 
        // Sum represents total combined capability.
        // 
        // Members:
        //   Agent: VâŸ¨tâŸ© = 10.0
        //   Agentâ‚‚: VâŸ¨tâŸ© = 20.0
        //   Agentâ‚ƒ: VâŸ¨tâŸ© = 30.0
        // 
        // VâŸ¨tâŸ©(DAO) = 10 + 20 + 30 = 60
        // â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•
        
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
        // â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•
        // EQUATION: Î¦_avg({V, Vâ‚‚, Vâ‚ƒ}) = (V + Vâ‚‚ + Vâ‚ƒ) / 3
        // 
        // Average represents mean reliability of members.
        // 
        // Members: {10.0, 20.0, 30.0}
        // VâŸ¨tâŸ©(DAO) = (10 + 20 + 30) / 3 = 20
        // â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•
        
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
        // â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•
        // EQUATION: Î¦_min({V, Vâ‚‚, Vâ‚ƒ}) = min(V, Vâ‚‚, Vâ‚ƒ)
        // 
        // Minimum represents "weakest link" analysis.
        // Use case: Security-focused DAO (only as strong as weakest member)
        // 
        // Members: {10.0, 5.0, 30.0}
        // VâŸ¨tâŸ©(DAO) = min(10, 5, 30) = 5
        // â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•
        
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
        // â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•
        // EQUATION: Î¦_max({V, Vâ‚‚, Vâ‚ƒ}) = max(V, Vâ‚‚, Vâ‚ƒ)
        // 
        // Maximum represents "strongest member" capability.
        // 
        // Members: {10.0, 5.0, 30.0}
        // VâŸ¨tâŸ©(DAO) = max(10, 5, 30) = 30
        // â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•
        
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
        // â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•
        // EQUATION: Î¦_median({V, Vâ‚‚, ..., Vâ‚™}) = middle value when sorted
        // 
        // Median is robust to outliers.
        // 
        // Members: {5.0, 10.0, 100.0} (sorted)
        // VâŸ¨tâŸ©(DAO) = median = 10.0
        // â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•
        
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
        // â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•
        // RECURSIVE PROPERTY: qâŸ¨TâŸ© ::= Agent | DAO({qâŸ¨TâŸ©})
        // 
        // "Turtles all the way down" - DAOs can contain DAOs.
        // 
        // Structure:
        //   OuterDAO (Î¦ = sum)
        //   â”œâ”€â”€ Agent: VâŸ¨tâŸ© = 10.0
        //   â””â”€â”€ InnerDAO (Î¦ = average)
        //       â”œâ”€â”€ Agentâ‚‚: VâŸ¨tâŸ© = 20.0
        //       â””â”€â”€ Agentâ‚ƒ: VâŸ¨tâŸ© = 40.0
        // 
        // Calculation:
        //   VâŸ¨tâŸ©(InnerDAO) = avg(20, 40) = 30
        //   VâŸ¨tâŸ©(OuterDAO) = sum(10, 30) = 40
        // â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•
        
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
        // â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•
        // PROPERTY: VâŸ¨tâŸ© âˆˆ  (can be negative)
        // 
        // DAOs can have members with negative trust (distrusted).
        // 
        // Members: {10.0, -5.0, 20.0}
        // Î¦_sum = 10 + (-5) + 20 = 25
        // â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•
        
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
        // â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•
        // CONSTRAINT: Î¦ must be defined before computing trust
        // â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•
        
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

Tests for aggregation functions Î¦.

**Mathematical Notation:**

```
â”Œâ”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€
â”‚  Î¦: {} â†’                                                         â”‚
â”‚                                                                     â”‚
â”‚  Î¦_sum({x, ..., xâ‚™}) = Î£áµ¢ xáµ¢                                      â”‚
â”‚  Î¦_avg({x, ..., xâ‚™}) = (Î£áµ¢ xáµ¢) / n                                â”‚
â”‚  Î¦_min({x, ..., xâ‚™}) = min(x, ..., xâ‚™)                           â”‚
â”‚  Î¦_max({x, ..., xâ‚™}) = max(x, ..., xâ‚™)                           â”‚
â”‚  Î¦_median = middle value when sorted                                â”‚
â”‚  Î¦_weighted({x,...,xâ‚™}, {w,...,wâ‚™}) = (Î£áµ¢ xáµ¢wáµ¢) / (Î£áµ¢ wáµ¢)       â”‚
â””â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”˜
```

```csharp

/// <summary>
/// Tests for aggregation functions Î¦.
/// 
/// Mathematical notation:
/// â”Œâ”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€
/// â”‚  Î¦: {} â†’                                                         â”‚
/// â”‚                                                                     â”‚
/// â”‚  Î¦_sum({x, ..., xâ‚™}) = Î£áµ¢ xáµ¢                                      â”‚
/// â”‚  Î¦_avg({x, ..., xâ‚™}) = (Î£áµ¢ xáµ¢) / n                                â”‚
/// â”‚  Î¦_min({x, ..., xâ‚™}) = min(x, ..., xâ‚™)                           â”‚
/// â”‚  Î¦_max({x, ..., xâ‚™}) = max(x, ..., xâ‚™)                           â”‚
/// â”‚  Î¦_median = middle value when sorted                                â”‚
/// â”‚  Î¦_weighted({x,...,xâ‚™}, {w,...,wâ‚™}) = (Î£áµ¢ xáµ¢wáµ¢) / (Î£áµ¢ wáµ¢)       â”‚
/// â””â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”˜
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
        // â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•
        // EQUATION: Î¦_sum = Î£áµ¢ xáµ¢
        // â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•
        
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
        // â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•
        // EQUATION: Î¦_avg = (Î£áµ¢ xáµ¢) / n
        // â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•
        
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
        // â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•
        // EQUATION: Î¦_min = min(x, ..., xâ‚™)
        // â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•
        
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
        // â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•
        // EQUATION: Î¦_max = max(x, ..., xâ‚™)
        // â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•
        
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
        // â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•
        // EQUATION: Î¦_median = middle value when sorted
        // 
        // For even n: average of two middle values
        // For odd n: the middle value
        // â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•
        
        // ACT
        var actual = AggregationFunctions.Median(values);

        // ASSERT
        Assert.Equal(expected, actual, precision: 10);
    }

    [Fact]
    public void WeightedAverage_ReturnsCorrectValue()
    {
        // â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•
        // EQUATION: Î¦_weighted = (Î£áµ¢ xáµ¢wáµ¢) / (Î£áµ¢ wáµ¢)
        // 
        // values:  {10, 20, 30}
        // weights: {1,  2,  3}
        // 
        // = (10Ã—1 + 20Ã—2 + 30Ã—3) / (1 + 2 + 3)
        // = (10 + 40 + 90) / 6
        // = 140 / 6
        // â‰ˆ 23.333...
        // â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•
        
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
        // â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•
        // EDGE CASE: Î£áµ¢ wáµ¢ = 0 â†’ undefined, return 0
        // â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•
        
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
        // â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•
        // EDGE CASE: null input â†’ return 0
        // â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•
        
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
â”Œâ”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€
â”‚  outcome(c) âˆˆ [-1, 1]                                              â”‚
â”‚                                                                     â”‚
â”‚  Discrete special case: {-1, 0, 1} = {failure, partial, success}   â”‚
â””â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”˜
```

```csharp

/// <summary>
/// Tests for the outcome function.
/// 
/// Mathematical notation:
/// â”Œâ”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€
/// â”‚  outcome(c) âˆˆ [-1, 1]                                              â”‚
/// â”‚                                                                     â”‚
/// â”‚  Discrete special case: {-1, 0, 1} = {failure, partial, success}   â”‚
/// â””â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”˜
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
        // â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•
        // CONSTRAINT: outcome(c) âˆˆ [-1, 1]
        // â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•
        
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
        // â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•
        // CONSTRAINT: outcome(c) âˆ‰ [-1, 1] â†’ throw
        // â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•
        
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
        // â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•
        // DISCRETE CASE: {-1, 0, 1} = {failure, partial, success}
        // â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•
        
        // ACT
        var actual = OutcomeCalculator.FromDiscrete(discrete);

        // ASSERT
        Assert.Equal(expected, actual);
    }

    [Theory]
    [InlineData(1.0, 1.0, 1.0)]      // 100% complete, 100% quality â†’ +1
    [InlineData(0.0, 0.0, -1.0)]     // 0% complete, 0% quality â†’ -1
    [InlineData(0.5, 0.5, 0.0)]      // 50% each â†’ 0 (neutral)
    [InlineData(1.0, 0.0, 0.0)]      // Complete but bad quality â†’ 0
    [InlineData(0.75, 0.75, 0.5)]    // 75% each â†’ +0.5
    public void CalculatePartialOutcome_MapsCorrectly(
        double completion, 
        double quality, 
        double expected)
    {
        // â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•
        // MAPPING: [0,1] Ã— [0,1] â†’ [-1,1]
        // 
        // Formula: outcome = ((completion + quality) / 2) Ã— 2 - 1
        //        = (completion + quality) - 1
        // â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•
        
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
        // â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•
        // CONSTRAINT: completion, quality âˆˆ [0, 1]
        // â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•
        
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
        // â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•
        // UTILITY: Clamp any value to [-1, 1]
        // â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•
        
        // ACT
        var actual = OutcomeCalculator.Clamp(input);

        // ASSERT
        Assert.Equal(expected, actual);
    }
}

```

---

## 5. Weighting Function Tests

Tests for the weighting function Ï‰.

**Mathematical Notation:**

```
â”Œâ”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€
â”‚  Ï‰(c) = f(s(c), d(c), VâŸ¨tâŸ©(a_consumer), recency(c))                 â”‚
â”‚                                                                     â”‚
â”‚  Components:                                                        â”‚
â”‚    stake_weight       = ln(1 + s)           [logarithmic]          â”‚
â”‚    difficulty_weight  = 0.5 + (d/10) Ã— 1.5  [linear, 0.5 to 2.0]   â”‚
â”‚    counterparty_weight= 1 + tanh(VâŸ¨tâŸ©/100) Ã— 0.5  [sigmoid, 0.5-1.5]â”‚
â”‚    recency_weight     = 0.5^(days/365)      [exponential decay]    â”‚
â”‚                                                                     â”‚
â”‚  Ï‰ = stake Ã— difficulty Ã— counterparty Ã— recency                   â”‚
â””â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”˜
```

```csharp

/// <summary>
/// Tests for the weighting function Ï‰.
/// 
/// Mathematical notation:
/// â”Œâ”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€
/// â”‚  Ï‰(c) = f(s(c), d(c), VâŸ¨tâŸ©(a_consumer), recency(c))                 â”‚
/// â”‚                                                                     â”‚
/// â”‚  Components:                                                        â”‚
/// â”‚    stake_weight       = ln(1 + s)           [logarithmic]          â”‚
/// â”‚    difficulty_weight  = 0.5 + (d/10) Ã— 1.5  [linear, 0.5 to 2.0]   â”‚
/// â”‚    counterparty_weight= 1 + tanh(VâŸ¨tâŸ©/100) Ã— 0.5  [sigmoid, 0.5-1.5]â”‚
/// â”‚    recency_weight     = 0.5^(days/365)      [exponential decay]    â”‚
/// â”‚                                                                     â”‚
/// â”‚  Ï‰ = stake Ã— difficulty Ã— counterparty Ã— recency                   â”‚
/// â””â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”˜
/// </summary>
public class WeightingFunctionTests
{
    private readonly WeightCalculator _calculator = new();
    private readonly DateTime _now = new(2025, 1, 1, 12, 0, 0, DateTimeKind.Utc);

    [Theory]
    [InlineData(0, 0.0)]                    // ln(1 + 0) = ln(1) = 0
    [InlineData(1, 0.693147)]               // ln(1 + 1) = ln(2) â‰ˆ 0.693
    [InlineData(100, 4.615120)]             // ln(101) â‰ˆ 4.615
    [InlineData(10000, 9.210440)]           // ln(10001) â‰ˆ 9.210
    public void StakeWeight_IsLogarithmic(double stake, double expectedLogComponent)
    {
        // â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•
        // EQUATION: stake_weight = ln(1 + s)
        // 
        // Logarithmic scaling prevents high stakes from dominating.
        // â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•
        
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
        // With d=5 â†’ difficulty_weight â‰ˆ 1.25, counterparty=1.0, recency=1.0
        // So weight â‰ˆ ln(1+s) Ã— 1.25 Ã— 1.0 Ã— 1.0
        var expectedWeight = expectedLogComponent * 1.25 * 1.0 * 1.0;
        Assert.Equal(Math.Max(expectedWeight, TrustConstants.MinWeight), weight, precision: 3);
    }

    [Theory]
    [InlineData(0, 0.5)]     // d=0  â†’ 0.5 + 0.0 Ã— 1.5 = 0.5
    [InlineData(5, 1.25)]    // d=5  â†’ 0.5 + 0.5 Ã— 1.5 = 1.25
    [InlineData(10, 2.0)]    // d=10 â†’ 0.5 + 1.0 Ã— 1.5 = 2.0
    public void DifficultyWeight_IsLinear(double difficulty, double expectedDiffWeight)
    {
        // â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•
        // EQUATION: difficulty_weight = 0.5 + (d/10) Ã— 1.5
        // 
        // Maps difficulty [0,10] to weight [0.5, 2.0]
        // Higher difficulty = more signal
        // â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•
        
        // ARRANGE: Fixed stake=100 â†’ ln(101) â‰ˆ 4.615
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
    [InlineData(0, 1.0)]           // VâŸ¨tâŸ©=0 â†’ tanh(0)=0 â†’ 1 + 0Ã—0.5 = 1.0
    [InlineData(100, 1.38)]        // VâŸ¨tâŸ©=100 â†’ tanh(1)â‰ˆ0.76 â†’ 1 + 0.76Ã—0.5 â‰ˆ 1.38
    [InlineData(-100, 0.62)]       // VâŸ¨tâŸ©=-100 â†’ tanh(-1)â‰ˆ-0.76 â†’ 1 - 0.76Ã—0.5 â‰ˆ 0.62
    [InlineData(1000, 1.5)]        // VâŸ¨tâŸ©=1000 â†’ tanh(10)â‰ˆ1 â†’ 1 + 1Ã—0.5 = 1.5 (max)
    public void CounterpartyWeight_IsSigmoidBounded(double counterpartyTrust, double expectedCpWeight)
    {
        // â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•
        // EQUATION: counterparty_weight = 1 + tanh(VâŸ¨tâŸ©/100) Ã— 0.5
        // 
        // Maps any trust value to [0.5, 1.5]
        // High trust counterparty â†’ more signal (up to 1.5Ã—)
        // Low trust counterparty â†’ less signal (down to 0.5Ã—)
        // â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•
        
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

        // ASSERT: ln(101) Ã— 1.25 Ã— cpWeight Ã— 1.0
        var expectedWeight = Math.Log(101) * 1.25 * expectedCpWeight * 1.0;
        Assert.Equal(expectedWeight, weight, precision: 1);
    }

    [Theory]
    [InlineData(0, 1.0)]           // Today â†’ 0.5^0 = 1.0
    [InlineData(365, 0.5)]         // 1 year ago â†’ 0.5^1 = 0.5
    [InlineData(730, 0.25)]        // 2 years ago â†’ 0.5^2 = 0.25
    [InlineData(1095, 0.125)]      // 3 years ago â†’ 0.5^3 = 0.125
    public void RecencyWeight_DecaysExponentially(int daysAgo, double expectedRecencyWeight)
    {
        // â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•
        // EQUATION: recency_weight = 0.5^(days/365)
        // 
        // Half-life of 365 days.
        // Contract from 1 year ago contributes half as much as today's.
        // â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•
        
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

        // ASSERT: ln(101) Ã— 1.25 Ã— 1.0 Ã— recency
        var expectedWeight = Math.Log(101) * 1.25 * 1.0 * expectedRecencyWeight;
        Assert.Equal(expectedWeight, weight, precision: 2);
    }

    [Fact]
    public void ComputeWeight_NullContract_Throws()
    {
        // â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•
        // CONSTRAINT: Contract cannot be null
        // â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•
        
        Assert.Throws<ArgumentNullException>(() => 
            _calculator.ComputeWeight(null!, 0, _now));
    }

    [Fact]
    public void ComputeWeight_ReturnsAtLeastMinWeight()
    {
        // â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•
        // CONSTRAINT: Ï‰(c) â‰¥ MinWeight (for numerical stability)
        // â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•
        
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
â”Œâ”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€
â”‚  hâŸ¨tâŸ©^(n+1)(a) = hâŸ¨tâŸ©^(n)(a) âˆª {câŸ¨nâŸ©}                                 â”‚
â””â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”˜
```

```csharp

/// <summary>
/// Tests for history evolution.
/// 
/// Mathematical notation:
/// â”Œâ”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€
/// â”‚  hâŸ¨tâŸ©^(n+1)(a) = hâŸ¨tâŸ©^(n)(a) âˆª {câŸ¨nâŸ©}                                 â”‚
/// â””â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”˜
/// 
/// The history grows by adding completed contracts.
/// History is append-only and partitioned by skill type.
/// </summary>
public class HistoryEvolutionTests
{
    [Fact]
    public void AddToHistory_AppendsContract()
    {
        // â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•
        // EQUATION: hâŸ¨tâŸ©^(n+1) = hâŸ¨tâŸ©^(n) âˆª {câŸ¨nâŸ©}
        // 
        // Union operation: Add new contract to existing history.
        // â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•
        
        // ARRANGE: hâŸ¨tâŸ©^(0) = âˆ…
        var agent = new Agent { SkillType = SkillTypes.Engineering };
        Assert.Empty(agent.ContractHistory);

        // ACT: hâŸ¨tâŸ©^(1) = hâŸ¨tâŸ©^(0) âˆª {c}
        var c1 = ContractFactory.CreateSimple(SkillTypes.Engineering, 1.0, 1.0);
        agent.AddToHistory(c1);

        // ASSERT
        Assert.Single(agent.ContractHistory);
        Assert.Contains(c1, agent.ContractHistory);
    }

    [Fact]
    public void AddToHistory_PreservesOrder()
    {
        // â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•
        // PROPERTY: History maintains insertion order
        // 
        // hâŸ¨tâŸ© = {c, câ‚‚, câ‚ƒ} in order of completion
        // â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•
        
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
        // â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•
        // CONSTRAINT: Cannot add null contract
        // â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•
        
        var agent = new Agent { SkillType = SkillTypes.Engineering };
        Assert.Throws<ArgumentNullException>(() => agent.AddToHistory(null!));
    }

    [Fact]
    public void GetHistoryForSkill_FiltersCorrectly()
    {
        // â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•
        // PROPERTY: History is partitioned by skill type
        // 
        // hâŸ¨eâŸ©ngineering âˆ© hâŸ¨dâŸ©esign = âˆ… (disjoint sets)
        // â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•
        
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
        // â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•
        // EQUATION: |hâŸ¨tâŸ©| = number of contracts for skill type t
        // â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•
        
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
â”Œâ”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€
â”‚  VâŸ¨tâŸ©^(n+1)(a) = VâŸ¨tâŸ©^(n)(a) + Ï‰(câŸ¨nâŸ©) Â· outcome(câŸ¨nâŸ©)                 â”‚
â””â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”˜
```

```csharp

/// <summary>
/// Tests for incremental trust evolution.
/// 
/// Mathematical notation:
/// â”Œâ”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€
/// â”‚  VâŸ¨tâŸ©^(n+1)(a) = VâŸ¨tâŸ©^(n)(a) + Ï‰(câŸ¨nâŸ©) Â· outcome(câŸ¨nâŸ©)                 â”‚
/// â””â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”˜
/// 
/// Trust updates incrementally with each contract completion.
/// </summary>
public class TrustEvolutionTests
{
    [Fact]
    public void UpdateTrust_AddsWeightedOutcome()
    {
        // â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•
        // EQUATION: VâŸ¨tâŸ©^(n+1) = VâŸ¨tâŸ©^(n) + Ï‰(câŸ¨nâŸ©) Â· outcome(câŸ¨nâŸ©)
        // 
        // Starting from VâŸ¨tâŸ©^(0) = 0:
        //   VâŸ¨tâŸ©^(1) = 0 + (2.0 Ã— 1.0) = 2.0
        // â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•
        
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
        // â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•
        // EQUATION: VâŸ¨tâŸ©^(n) = Î£_{i=1}^{n} Ï‰(cáµ¢) Â· outcome(cáµ¢)
        // 
        // VâŸ¨tâŸ©^(0) = 0
        // VâŸ¨tâŸ©^(1) = 0 + (1.0 Ã— 1.0) = 1.0
        // VâŸ¨tâŸ©^(2) = 1.0 + (2.0 Ã— 0.5) = 2.0
        // VâŸ¨tâŸ©^(3) = 2.0 + (1.5 Ã— -1.0) = 0.5
        // â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•
        
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
        // â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•
        // PROPERTY: All trust changes are auditable
        // â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•
        
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
        // â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•
        // COMBINED OPERATION:
        //   hâŸ¨tâŸ©^(n+1) = hâŸ¨tâŸ©^(n) âˆª {câŸ¨nâŸ©}
        //   VâŸ¨tâŸ©^(n+1) = VâŸ¨tâŸ©^(n) + Ï‰(câŸ¨nâŸ©) Â· outcome(câŸ¨nâŸ©)
        // â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•
        
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
â”Œâ”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€
â”‚  eligible(a, c) âŸº VâŸ¨tâŸ©(a) â‰¥ Î¸(c)                                   â”‚
â”‚                                                                     â”‚
â”‚  Î¸(c) = max(ln(1+s) Ã— 0.1, ln(1+s) Ã— d)                            â”‚
â””â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”˜
```

```csharp

/// <summary>
/// Tests for the eligibility function.
/// 
/// Mathematical notation:
/// â”Œâ”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€
/// â”‚  eligible(a, c) âŸº VâŸ¨tâŸ©(a) â‰¥ Î¸(c)                                   â”‚
/// â”‚                                                                     â”‚
/// â”‚  Î¸(c) = max(ln(1+s) Ã— 0.1, ln(1+s) Ã— d)                            â”‚
/// â””â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”˜
/// 
/// An agent is eligible for a contract iff their trust meets the threshold.
/// </summary>
public class EligibilityFunctionTests
{
    [Fact]
    public void IsEligible_TrustMeetsThreshold_ReturnsTrue()
    {
        // â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•
        // EQUATION: eligible(a, c) âŸº VâŸ¨tâŸ©(a) â‰¥ Î¸(c)
        // 
        // VâŸ¨tâŸ©(agent) = 50.0
        // Î¸(contract) = ln(1+1000) Ã— 5 â‰ˆ 6.9 Ã— 5 = 34.5
        // 50.0 â‰¥ 34.5 â†’ eligible = true
        // â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•
        
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
        // â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•
        // EQUATION: Â¬eligible(a, c) âŸº VâŸ¨tâŸ©(a) < Î¸(c)
        // 
        // VâŸ¨tâŸ©(agent) = 10.0
        // Î¸(contract) = ln(1+10000) Ã— 8 â‰ˆ 9.2 Ã— 8 = 73.6
        // 10.0 < 73.6 â†’ eligible = false
        // â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•
        
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
    [InlineData(0, 0, 0.0)]                    // Î¸ = max(0, 0) = 0
    [InlineData(100, 0, 0.461)]                // Î¸ = max(ln(101)Ã—0.1, 0) â‰ˆ 0.461
    [InlineData(100, 5, 23.08)]                // Î¸ = max(0.461, ln(101)Ã—5) â‰ˆ 23.08
    [InlineData(1000, 3, 20.73)]               // Î¸ = ln(1001)Ã—3 â‰ˆ 20.73
    public void CalculateThreshold_ReturnsCorrectValue(
        double stake, 
        double difficulty, 
        double expected)
    {
        // â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•
        // EQUATION: Î¸(c) = max(ln(1+s) Ã— 0.1, ln(1+s) Ã— d)
        // 
        // The minimum threshold factor (0.1) ensures even zero-difficulty
        // contracts have some barrier to entry.
        // â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•
        
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
        // â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•
        // EDGE CASE: VâŸ¨tâŸ© = 0, Î¸ = 0
        // 
        // 0 â‰¥ 0 â†’ true (newcomer can take zero-stake contracts)
        // â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•
        
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
        // â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•
        // EDGE CASE: VâŸ¨tâŸ© < 0 (distrusted agent)
        // 
        // -10 < any positive threshold â†’ never eligible
        // â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•
        
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
        // â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•
        // FILTER: {c : eligible(a, c)}
        // â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•
        
        // ARRANGE: Agent with trust = 30
        var agent = new Agent { SkillType = SkillTypes.Engineering };
        agent.AddToHistory(ContractFactory.CreateSimple(SkillTypes.Engineering, 1.0, 30.0));

        var contracts = new[]
        {
            ContractFactory.CreateSimple(SkillTypes.Engineering, 0, 0, stake: 10, difficulty: 1),    // Î¸â‰ˆ2.4, eligible
            ContractFactory.CreateSimple(SkillTypes.Engineering, 0, 0, stake: 100, difficulty: 5),  // Î¸â‰ˆ23, eligible
            ContractFactory.CreateSimple(SkillTypes.Engineering, 0, 0, stake: 1000, difficulty: 8), // Î¸â‰ˆ55, not eligible
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
â”Œâ”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€
â”‚  |hâŸ¨tâŸ©(a_honest)| > |hâŸ¨tâŸ©(a_sybil_i)|   âˆ€i                           â”‚
â””â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”˜
```

```csharp

/// <summary>
/// Tests for Sybil resistance properties.
/// 
/// Mathematical notation:
/// â”Œâ”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€
/// â”‚  |hâŸ¨tâŸ©(a_honest)| > |hâŸ¨tâŸ©(a_sybil_i)|   âˆ€i                           â”‚
/// â””â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”˜
/// 
/// Splitting activity across k fake identities results in each having
/// ~1/k the history of an honest single-identity agent.
/// </summary>
public class SybilResistanceTests
{
    [Fact]
    public void AnalyzeSybilResistance_HonestHasMoreHistory()
    {
        // â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•
        // EQUATION: |hâŸ¨tâŸ©(a_honest)| > |hâŸ¨tâŸ©(a_sybil_i)|  âˆ€i
        // 
        // Scenario:
        //   Honest agent: 100 contracts
        //   Attacker splits across 5 sybils: 20 contracts each
        // 
        // 100 > 20 for all sybils â†’ Sybil resistant
        // â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•
        
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
        // â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•
        // CONSEQUENCE: VâŸ¨tâŸ©(honest) > VâŸ¨tâŸ©(sybil_i)  âˆ€i
        // 
        // More history with same success rate â†’ more trust
        // 
        // VâŸ¨tâŸ©(honest) = 100 Ã— 1.0 Ã— 1.0 = 100
        // VâŸ¨tâŸ©(sybil) = 20 Ã— 1.0 Ã— 1.0 = 20
        // Advantage ratio = 100/20 = 5.0
        // â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•
        
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
        // â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•
        // EQUATION: advantage_ratio = VâŸ¨tâŸ©(honest) / VâŸ¨tâŸ©(best_sybil)
        // 
        // With special handling for:
        //   - Zero denominators
        //   - Negative trust values
        //   - Mixed signs
        // â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•
        
        // ACT
        var actual = SybilResistanceAnalyzer.CalculateAdvantageRatio(honestTrust, maxSybilTrust);

        // ASSERT
        Assert.Equal(expected, actual, precision: 2);
    }

    [Fact]
    public void AnalyzeSybilResistance_EmptySybilList_NotResistant()
    {
        // â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•
        // EDGE CASE: No sybils to compare against
        // 
        // Cannot claim "resistant" if there's no attack to resist.
        // â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•
        
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
â”Œâ”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€
â”‚  lim(nâ†’âˆž) Corr(VâŸ¨tâŸ©^(n)(a), RâŸ¨tâŸ©(a)) = 1                             â”‚
â””â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”˜
```

```csharp

/// <summary>
/// Tests for network validation through convergence.
/// 
/// Mathematical notation:
/// â”Œâ”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€
/// â”‚  lim(nâ†’âˆž) Corr(VâŸ¨tâŸ©^(n)(a), RâŸ¨tâŸ©(a)) = 1                             â”‚
/// â””â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”˜
/// 
/// As history accumulates, the correlation between computed trust
/// and actual reliability should approach 1.0.
/// </summary>
public class ConvergenceCriterionTests
{
    [Fact]
    public void CalculateConvergence_PerfectCorrelation_ReturnsOne()
    {
        // â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•
        // EQUATION: Corr(VâŸ¨tâŸ©, RâŸ¨tâŸ©) = 1 when VâŸ¨tâŸ©  RâŸ¨tâŸ©
        // 
        // If trust perfectly reflects reliability:
        //   Agent with R=0.9 gets V~90
        //   Agent with R=0.5 gets V~50
        //   â†’ Correlation = 1.0
        // â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•
        
        // ARRANGE: Agents where trust = reliability Ã— 100
        var agents = new List<SimulatedAgent>();
        foreach (var reliability in new[] { 0.1, 0.3, 0.5, 0.7, 0.9 })
        {
            var agent = new SimulatedAgent
            {
                SkillType = SkillTypes.Engineering,
                ActualReliability = reliability
            };
            // Trust = reliability Ã— 100 (perfect linear relationship)
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
        // â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•
        // EQUATION: Corr(VâŸ¨tâŸ©, RâŸ¨tâŸ©) â‰ˆ 0 when VâŸ¨tâŸ© and RâŸ¨tâŸ© are independent
        // â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•
        
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
        // â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•
        // EDGE CASE: Pearson correlation requires n â‰¥ 2
        // â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•
        
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
        // â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•
        // VALIDATION: Corr â‰¥ ValidationThreshold (0.95)
        // â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•
        
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
        // â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•
        // SIMULATION: As nâ†’âˆž, Corr(VâŸ¨tâŸ©, RâŸ¨tâŸ©)â†’1
        // 
        // With enough contracts, trust scores should reflect actual
        // reliability due to the law of large numbers.
        // â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•
        
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
â”Œâ”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€
â”‚  c = (a_provider, a_consumer, t, s, d, Ï„)                          â”‚
â”‚                                                                     â”‚
â”‚  Constraints:                                                       â”‚
â”‚    s â‰¥ 0              (stake is non-negative)                      â”‚
â”‚    d âˆˆ [0, 10]        (difficulty in valid range)                  â”‚
â”‚    outcome âˆˆ [-1, 1]  (outcome in valid range)                     â”‚
â””â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”˜
```

```csharp

/// <summary>
/// Tests for contract structure and validation.
/// 
/// Mathematical notation:
/// â”Œâ”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€
/// â”‚  c = (a_provider, a_consumer, t, s, d, Ï„)                          â”‚
/// â”‚                                                                     â”‚
/// â”‚  Constraints:                                                       â”‚
/// â”‚    s â‰¥ 0              (stake is non-negative)                      â”‚
/// â”‚    d âˆˆ [0, 10]        (difficulty in valid range)                  â”‚
/// â”‚    outcome âˆˆ [-1, 1]  (outcome in valid range)                     â”‚
/// â””â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”˜
/// </summary>
public class ContractValidationTests
{
    [Fact]
    public void Contract_ValidValues_IsValid()
    {
        // â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•
        // VALID CONTRACT: All constraints satisfied
        // â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•
        
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
        // â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•
        // CONSTRAINT: s â‰¥ 0
        // â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•
        
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
        // â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•
        // CONSTRAINT: d âˆˆ [0, 10]
        // â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•
        
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
        // â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•
        // CONSTRAINT: outcome âˆˆ [-1, 1]
        // â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•
        
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
        // â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•
        // CONSTRAINT: t â‰  âˆ… (skill type required)
        // â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•
        
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
        // â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•
        // FACTORY: Contract.Create validates inputs
        // â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•
        
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
â”Œâ”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€
â”‚  VâŸ¨tâŸ©: qâŸ¨TâŸ© â†’                                                      â”‚
â””â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”˜
```

```csharp

/// <summary>
/// Tests for the TrustValuation.V() static helper function.
/// 
/// Mathematical notation:
/// â”Œâ”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€
/// â”‚  VâŸ¨tâŸ©: qâŸ¨TâŸ© â†’                                                      â”‚
/// â””â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”˜
/// </summary>
public class TrustValuationTests
{
    [Fact]
    public void V_Agent_ReturnsTrustValue()
    {
        // â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•
        // EQUATION: VâŸ¨tâŸ©(Agent) = computed trust value
        // â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•
        
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
        // â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•
        // EQUATION: VâŸ¨tâŸ©(DAO) = Î¦({member trusts})
        // â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•
        
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

## 14. Customer Trust Calculation Tests

Tests for customer trust computation based on behavioral metrics.

**Mathematical Notation:**

```
┌───────────────────────────────────────────────────────────────────────────────
│  V⟨t⟩^customer(Consumer) = Σ ω(c) · behavior(c) · γ(c) · ν(c)               │
└───────────────────────────────────────────────────────────────────────────────
```

```csharp

/// <summary>
/// Tests for customer trust computation.
/// 
/// Mathematical notation:
/// ┌───────────────────────────────────────────────────────────────────────────────
/// │  V⟨t⟩^customer(Consumer) = Σ ω(c) · behavior(c) · γ(c) · ν(c)               │
/// └───────────────────────────────────────────────────────────────────────────────
/// 
/// Customer trust is computed from observable behaviors, not assigned outcomes.
/// Six customer skill types:
///   - Commitment: completed / initiated projects
///   - Escrow Discipline: on-time funding + prompt release
///   - Verification Integrity: rating variance near ideal
///   - Scope Stability: tasks as planned / total tasks
///   - Timeline Realism: deadline accuracy
///   - Spec Quality: impl outcomes for approved specs
/// </summary>
public class CustomerTrustCalculationTests
{
    [Fact]
    public void ComputeCommitmentTrust_AllCompleted_ReturnsOne()
    {
        // ═══════════════════════════════════════════════════════════════════
        // EQUATION: V_commitment = completed / initiated
        // When completed = initiated, commitment = 1.0 (perfect)
        // ═══════════════════════════════════════════════════════════════════
        
        // ARRANGE
        var profile = new CustomerProfile
        {
            Customer = new Agent { SkillType = SkillTypes.Engineering },
            ProjectsInitiated = 10,
            ProjectsCompleted = 10
        };
        var expected = 1.0;

        // ACT
        var actual = CustomerTrustCalculator.ComputeCommitmentTrust(profile);

        // ASSERT
        Assert.Equal(expected, actual);
    }

    [Fact]
    public void ComputeCommitmentTrust_HalfCompleted_ReturnsHalf()
    {
        // ═══════════════════════════════════════════════════════════════════
        // EQUATION: V_commitment = completed / initiated = 5 / 10 = 0.5
        // ═══════════════════════════════════════════════════════════════════
        
        // ARRANGE
        var profile = new CustomerProfile
        {
            Customer = new Agent { SkillType = SkillTypes.Engineering },
            ProjectsInitiated = 10,
            ProjectsCompleted = 5
        };
        var expected = 0.5;

        // ACT
        var actual = CustomerTrustCalculator.ComputeCommitmentTrust(profile);

        // ASSERT
        Assert.Equal(expected, actual);
    }

    [Fact]
    public void ComputeCommitmentTrust_NoneInitiated_ReturnsZero()
    {
        // ═══════════════════════════════════════════════════════════════════
        // Edge case: Division by zero protection
        // ═══════════════════════════════════════════════════════════════════
        
        // ARRANGE
        var profile = new CustomerProfile
        {
            Customer = new Agent { SkillType = SkillTypes.Engineering },
            ProjectsInitiated = 0,
            ProjectsCompleted = 0
        };

        // ACT
        var actual = CustomerTrustCalculator.ComputeCommitmentTrust(profile);

        // ASSERT
        Assert.Equal(0.0, actual);
    }

    [Fact]
    public void ComputeEscrowTrust_PerfectRecord_ReturnsHigh()
    {
        // ═══════════════════════════════════════════════════════════════════
        // EQUATION: V_escrow = 0.7 × funding_rate + 0.3 × delay_factor
        // Perfect: funding_rate = 1.0, delay = 0 → delay_factor = 1.0
        // Result: 0.7 × 1.0 + 0.3 × 1.0 = 1.0
        // ═══════════════════════════════════════════════════════════════════
        
        // ARRANGE
        var profile = new CustomerProfile
        {
            Customer = new Agent { SkillType = SkillTypes.Engineering },
            OnTimeFundings = 10,
            TotalFundings = 10,
            AvgReleaseDelayDays = 0.0
        };
        var expected = 1.0;

        // ACT
        var actual = CustomerTrustCalculator.ComputeEscrowTrust(profile);

        // ASSERT
        Assert.Equal(expected, actual, precision: 2);
    }

    [Fact]
    public void ComputeEscrowTrust_LateReleases_ReducesTrust()
    {
        // ═══════════════════════════════════════════════════════════════════
        // EQUATION: delay_factor decreases as avg delay increases
        // 7+ days delay → delay_factor = 0.5
        // ═══════════════════════════════════════════════════════════════════
        
        // ARRANGE
        var profile = new CustomerProfile
        {
            Customer = new Agent { SkillType = SkillTypes.Engineering },
            OnTimeFundings = 10,
            TotalFundings = 10,
            AvgReleaseDelayDays = 10.0  // Very late
        };

        // ACT
        var actual = CustomerTrustCalculator.ComputeEscrowTrust(profile);

        // ASSERT: Should be 0.7 × 1.0 + 0.3 × 0.5 = 0.85
        Assert.Equal(0.85, actual, precision: 2);
    }

    [Fact]
    public void ComputeVerificationIntegrity_IdealVariance_ReturnsHigh()
    {
        // ═══════════════════════════════════════════════════════════════════
        // EQUATION: Ideal variance ≈ 0.35
        // Distance from ideal = 0 → integrity = 1.0
        // ═══════════════════════════════════════════════════════════════════
        
        // ARRANGE
        var profile = new CustomerProfile
        {
            Customer = new Agent { SkillType = SkillTypes.Engineering },
            TotalRatingsGiven = 20,
            RatingsVariance = TrustConstants.IdealRatingVariance  // 0.35
        };

        // ACT
        var actual = CustomerTrustCalculator.ComputeVerificationIntegrity(profile);

        // ASSERT
        Assert.True(actual > 0.9, "Ideal variance should yield high integrity");
    }

    [Fact]
    public void ComputeVerificationIntegrity_ZeroVariance_RubberStamping_ReturnsLow()
    {
        // ═══════════════════════════════════════════════════════════════════
        // EQUATION: Very low variance = rubber-stamping → severe penalty
        // variance < MIN_RATING_VARIANCE → 0.5x multiplier
        // ═══════════════════════════════════════════════════════════════════
        
        // ARRANGE
        var profile = new CustomerProfile
        {
            Customer = new Agent { SkillType = SkillTypes.Engineering },
            TotalRatingsGiven = 20,
            RatingsVariance = 0.01  // Almost no variance (rubber-stamping)
        };

        // ACT
        var actual = CustomerTrustCalculator.ComputeVerificationIntegrity(profile);

        // ASSERT
        Assert.True(actual < 0.5, "Rubber-stamping should yield low integrity");
    }

    [Fact]
    public void ComputeVerificationIntegrity_HighVariance_Erratic_ReturnsReduced()
    {
        // ═══════════════════════════════════════════════════════════════════
        // EQUATION: variance > MAX_RATING_VARIANCE → 0.75x multiplier
        // ═══════════════════════════════════════════════════════════════════
        
        // ARRANGE
        var profile = new CustomerProfile
        {
            Customer = new Agent { SkillType = SkillTypes.Engineering },
            TotalRatingsGiven = 20,
            RatingsVariance = 0.9  // Very high variance (erratic)
        };

        // ACT
        var actual = CustomerTrustCalculator.ComputeVerificationIntegrity(profile);

        // ASSERT
        Assert.True(actual < 0.7, "Erratic rating should reduce integrity");
    }

    [Fact]
    public void ComputeScopeStability_AllAsPlanned_ReturnsOne()
    {
        // ═══════════════════════════════════════════════════════════════════
        // EQUATION: V_scope = tasks_as_planned / total_tasks = 1.0
        // ═══════════════════════════════════════════════════════════════════
        
        // ARRANGE
        var profile = new CustomerProfile
        {
            Customer = new Agent { SkillType = SkillTypes.Engineering },
            ScopeTasksAsPlanned = 50,
            ScopeTotalTasks = 50
        };

        // ACT
        var actual = CustomerTrustCalculator.ComputeScopeStability(profile);

        // ASSERT
        Assert.Equal(1.0, actual);
    }

    [Fact]
    public void ComputeScopeStability_HalfDeviated_ReturnsHalf()
    {
        // ═══════════════════════════════════════════════════════════════════
        // EQUATION: V_scope = 25 / 50 = 0.5
        // ═══════════════════════════════════════════════════════════════════
        
        // ARRANGE
        var profile = new CustomerProfile
        {
            Customer = new Agent { SkillType = SkillTypes.Engineering },
            ScopeTasksAsPlanned = 25,
            ScopeTotalTasks = 50
        };

        // ACT
        var actual = CustomerTrustCalculator.ComputeScopeStability(profile);

        // ASSERT
        Assert.Equal(0.5, actual);
    }

    [Fact]
    public void ComputeTimelineRealism_OnTime_ReturnsHigh()
    {
        // ═══════════════════════════════════════════════════════════════════
        // EQUATION: accuracy >= 0 → maps to [0.5, 1.0]
        // On-time delivery (accuracy = 0) → 0.5
        // Early delivery (accuracy > 0) → > 0.5
        // ═══════════════════════════════════════════════════════════════════
        
        // ARRANGE
        var profile = new CustomerProfile
        {
            Customer = new Agent { SkillType = SkillTypes.Engineering },
            TimelineAccuracySum = 0.0,  // On time
            TimelineProjectsCount = 10
        };

        // ACT
        var actual = CustomerTrustCalculator.ComputeTimelineRealism(profile);

        // ASSERT
        Assert.Equal(0.5, actual, precision: 2);
    }

    [Fact]
    public void ComputeTimelineRealism_ChronicOverruns_ReturnsLow()
    {
        // ═══════════════════════════════════════════════════════════════════
        // EQUATION: accuracy < 0 → maps to [0, 0.5]
        // Large overruns → low realism score
        // ═══════════════════════════════════════════════════════════════════
        
        // ARRANGE
        var profile = new CustomerProfile
        {
            Customer = new Agent { SkillType = SkillTypes.Engineering },
            TimelineAccuracySum = -10.0,  // 1.0 average overrun per project
            TimelineProjectsCount = 10
        };

        // ACT
        var actual = CustomerTrustCalculator.ComputeTimelineRealism(profile);

        // ASSERT
        Assert.True(actual < 0.5, "Chronic overruns should yield low realism");
    }

    [Fact]
    public void ComputeSpecQuality_GoodImplementations_ReturnsPositive()
    {
        // ═══════════════════════════════════════════════════════════════════
        // EQUATION: V_spec = Σ(weight × outcome) / Σ(weight)
        // Weighted average of implementation outcomes
        // ═══════════════════════════════════════════════════════════════════
        
        // ARRANGE
        var profile = new CustomerProfile
        {
            Customer = new Agent { SkillType = SkillTypes.Engineering },
            ImplementationOutcomes = new[]
            {
                (Outcome: 1.0, Weight: 2.0),  // +2.0
                (Outcome: 0.8, Weight: 1.0),  // +0.8
                (Outcome: 0.6, Weight: 1.0),  // +0.6
            }
        };
        // Expected: (2×1.0 + 1×0.8 + 1×0.6) / (2+1+1) = 3.4 / 4 = 0.85

        // ACT
        var actual = CustomerTrustCalculator.ComputeSpecQuality(profile);

        // ASSERT
        Assert.Equal(0.85, actual, precision: 2);
    }

    [Fact]
    public void ComputeCustomerTrust_RoutesToCorrectMethod()
    {
        // ═══════════════════════════════════════════════════════════════════
        // EQUATION: ComputeCustomerTrust dispatches to skill-specific method
        // ═══════════════════════════════════════════════════════════════════
        
        // ARRANGE
        var profile = new CustomerProfile
        {
            Customer = new Agent { SkillType = SkillTypes.Engineering },
            ProjectsInitiated = 10,
            ProjectsCompleted = 8
        };

        // ACT
        var commitment = CustomerTrustCalculator.ComputeCustomerTrust(
            profile, CustomerSkillTypes.Commitment);

        // ASSERT
        Assert.Equal(0.8, commitment);
    }

    [Fact]
    public void ComputeCompositeTrust_AveragesAllDimensions()
    {
        // ═══════════════════════════════════════════════════════════════════
        // EQUATION: Composite = weighted average of all customer skills
        // ═══════════════════════════════════════════════════════════════════
        
        // ARRANGE: Perfect customer across all dimensions
        var profile = new CustomerProfile
        {
            Customer = new Agent { SkillType = SkillTypes.Engineering },
            ProjectsInitiated = 10,
            ProjectsCompleted = 10,
            OnTimeFundings = 10,
            TotalFundings = 10,
            AvgReleaseDelayDays = 0,
            TotalRatingsGiven = 20,
            RatingsVariance = TrustConstants.IdealRatingVariance,
            ScopeTasksAsPlanned = 50,
            ScopeTotalTasks = 50,
            TimelineAccuracySum = 5.0,  // Early delivery
            TimelineProjectsCount = 10,
            ImplementationOutcomes = new[] { (1.0, 1.0) }
        };

        // ACT
        var composite = CustomerTrustCalculator.ComputeCompositeTrust(profile);

        // ASSERT: Should be high (all dimensions near 1.0)
        Assert.True(composite > 0.7, $"Perfect customer should have high composite trust, got {composite}");
    }
}

```

---

## 15. Verification Weight Tests

Tests for verification weight calculation based on customer credibility.

**Mathematical Notation:**

```
┌───────────────────────────────────────────────────────────────────────────────
│  verification_weight(c) = f(V_verify(consumer), σ²(ratings))                 │
└───────────────────────────────────────────────────────────────────────────────
```

```csharp

/// <summary>
/// Tests for verification weight calculation.
/// 
/// Mathematical notation:
/// ┌───────────────────────────────────────────────────────────────────────────────
/// │  verification_weight(c) = f(V_verify(consumer), σ²(ratings))                 │
/// └───────────────────────────────────────────────────────────────────────────────
/// 
/// Verification weight adjusts how much a customer's rating contributes to trust.
/// High-credibility customers' ratings count more.
/// </summary>
public class VerificationWeightTests
{
    [Fact]
    public void ComputeVerificationWeight_HighTrust_NormalVariance_ReturnsHigh()
    {
        // ═══════════════════════════════════════════════════════════════════
        // EQUATION: weight = trust_factor × variance_factor
        // High trust + normal variance → high weight
        // ═══════════════════════════════════════════════════════════════════
        
        // ARRANGE
        double customerVerificationTrust = 0.9;  // High integrity
        double customerRatingVariance = 0.35;    // Ideal variance

        // ACT
        var weight = VerificationWeightCalculator.ComputeVerificationWeight(
            customerVerificationTrust,
            customerRatingVariance);

        // ASSERT: Should be close to 1.0 or above
        Assert.True(weight >= 0.9, $"High trust + ideal variance should yield high weight, got {weight}");
    }

    [Fact]
    public void ComputeVerificationWeight_LowTrust_ReturnsLow()
    {
        // ═══════════════════════════════════════════════════════════════════
        // EQUATION: trust_factor = 0.5 + (trust × 0.5)
        // Low trust (0.1) → trust_factor = 0.5 + 0.05 = 0.55
        // ═══════════════════════════════════════════════════════════════════
        
        // ARRANGE
        double customerVerificationTrust = 0.1;  // Low integrity
        double customerRatingVariance = 0.35;    // Normal variance

        // ACT
        var weight = VerificationWeightCalculator.ComputeVerificationWeight(
            customerVerificationTrust,
            customerRatingVariance);

        // ASSERT
        Assert.True(weight < 0.7, $"Low trust should yield low weight, got {weight}");
    }

    [Fact]
    public void ComputeVerificationWeight_RubberStamping_PenalizesWeight()
    {
        // ═══════════════════════════════════════════════════════════════════
        // EQUATION: variance < MIN → variance_factor = 0.5
        // ═══════════════════════════════════════════════════════════════════
        
        // ARRANGE
        double customerVerificationTrust = 0.9;  // High trust
        double customerRatingVariance = 0.05;    // Too low (rubber-stamping)

        // ACT
        var weight = VerificationWeightCalculator.ComputeVerificationWeight(
            customerVerificationTrust,
            customerRatingVariance);

        // ASSERT: Despite high trust, variance penalty reduces weight
        Assert.True(weight < 0.75, $"Rubber-stamping should reduce weight, got {weight}");
    }

    [Fact]
    public void ComputeVerificationWeight_ErraticRating_PenalizesWeight()
    {
        // ═══════════════════════════════════════════════════════════════════
        // EQUATION: variance > MAX → variance_factor = 0.7
        // ═══════════════════════════════════════════════════════════════════
        
        // ARRANGE
        double customerVerificationTrust = 0.9;  // High trust
        double customerRatingVariance = 0.9;     // Too high (erratic)

        // ACT
        var weight = VerificationWeightCalculator.ComputeVerificationWeight(
            customerVerificationTrust,
            customerRatingVariance);

        // ASSERT: Erratic rating penalty
        Assert.True(weight < 0.9, $"Erratic rating should reduce weight, got {weight}");
    }

    [Fact]
    public void ComputeVerificationWeight_ClampedToRange()
    {
        // ═══════════════════════════════════════════════════════════════════
        // EQUATION: weight ∈ [0.5, 1.5]
        // ═══════════════════════════════════════════════════════════════════
        
        // ARRANGE - worst case
        double lowTrust = 0.0;
        double lowVariance = 0.0;

        // ACT
        var weight = VerificationWeightCalculator.ComputeVerificationWeight(
            lowTrust, lowVariance);

        // ASSERT
        Assert.True(weight >= TrustConstants.MinVerificationWeight,
            $"Weight should be at least {TrustConstants.MinVerificationWeight}, got {weight}");
        Assert.True(weight <= TrustConstants.MaxVerificationWeight,
            $"Weight should be at most {TrustConstants.MaxVerificationWeight}, got {weight}");
    }

    [Fact]
    public void ComputeAdjustedContribution_AppliesWeight()
    {
        // ═══════════════════════════════════════════════════════════════════
        // EQUATION: adjusted = base_contribution × verification_weight
        // ═══════════════════════════════════════════════════════════════════
        
        // ARRANGE
        var contract = ContractFactory.CreateSimple(
            skillType: SkillTypes.Engineering,
            outcome: 1.0,
            weight: 2.0
        );
        // Base contribution = 2.0 × 1.0 = 2.0

        var profile = new CustomerProfile
        {
            Customer = new Agent { SkillType = SkillTypes.Engineering },
            TotalRatingsGiven = 20,
            RatingsVariance = 0.35  // Ideal
        };

        // ACT
        var adjusted = VerificationWeightCalculator.ComputeAdjustedContribution(
            contract, profile);

        // ASSERT: Adjusted should differ from base if weight != 1.0
        Assert.NotEqual(0.0, adjusted);
    }

    [Fact]
    public void ComputeAdjustedContribution_NullProfile_ReturnsBase()
    {
        // ═══════════════════════════════════════════════════════════════════
        // Edge case: No profile → use base contribution unchanged
        // ═══════════════════════════════════════════════════════════════════
        
        // ARRANGE
        var contract = ContractFactory.CreateSimple(
            skillType: SkillTypes.Engineering,
            outcome: 1.0,
            weight: 2.0
        );
        var expected = 2.0;  // weight × outcome

        // ACT
        var adjusted = VerificationWeightCalculator.ComputeAdjustedContribution(
            contract, null);

        // ASSERT
        Assert.Equal(expected, adjusted);
    }
}

```

---

## 16. Task Decomposition Tests

Tests for task-level decomposition within milestones and team-based implementation.

**Mathematical Notation:**

```
┌───────────────────────────────────────────────────────────────────────────────
│  s_task = s_milestone × (w_task / Σ w_tasks)                                  │
└───────────────────────────────────────────────────────────────────────────────
```

```csharp

/// <summary>
/// Tests for task decomposition and team-based implementation.
/// 
/// Mathematical notation:
/// ┌───────────────────────────────────────────────────────────────────────────────
/// │  s_task = s_milestone × (w_task / Σ w_tasks)                                  │
/// └───────────────────────────────────────────────────────────────────────────────
/// 
/// Tasks are atomic work units within milestones.
/// Each task can have its own provider (enabling team-based implementation).
/// Trust flows through tasks. Milestones are coordination containers.
/// </summary>
public class TaskDecompositionTests
{
    [Fact]
    public void ComputeTaskStake_EqualWeights_SplitsEvenly()
    {
        // ═══════════════════════════════════════════════════════════════════
        // EQUATION: s_task = s_milestone × (w_task / Σ w_tasks)
        // 
        // milestone_stake = 1000, 4 tasks with weight 1.0 each
        // task_stake = 1000 × (1 / 4) = 250
        // ═══════════════════════════════════════════════════════════════════
        
        // ARRANGE
        double milestoneStake = 1000;
        double taskWeight = 1.0;
        double totalTaskWeight = 4.0;  // 4 tasks × 1.0 each

        // ACT
        var taskStake = TaskDecomposition.ComputeTaskStake(
            milestoneStake, taskWeight, totalTaskWeight);

        // ASSERT
        Assert.Equal(250.0, taskStake);
    }

    [Fact]
    public void ComputeTaskStake_UnequalWeights_ProportionalSplit()
    {
        // ═══════════════════════════════════════════════════════════════════
        // EQUATION: s_task = s_milestone × (w_task / Σ w_tasks)
        // 
        // milestone_stake = 1000
        // Task weights: 2.0, 1.0, 1.0 (total = 4.0)
        // First task: 1000 × (2 / 4) = 500
        // ═══════════════════════════════════════════════════════════════════
        
        // ARRANGE
        double milestoneStake = 1000;
        double taskWeight = 2.0;
        double totalTaskWeight = 4.0;

        // ACT
        var taskStake = TaskDecomposition.ComputeTaskStake(
            milestoneStake, taskWeight, totalTaskWeight);

        // ASSERT
        Assert.Equal(500.0, taskStake);
    }

    [Fact]
    public void ComputeTaskStake_ZeroTotalWeight_ReturnsZero()
    {
        // ═══════════════════════════════════════════════════════════════════
        // Edge case: Division by zero protection
        // ═══════════════════════════════════════════════════════════════════
        
        // ARRANGE
        double milestoneStake = 1000;
        double taskWeight = 1.0;
        double totalTaskWeight = 0.0;

        // ACT
        var taskStake = TaskDecomposition.ComputeTaskStake(
            milestoneStake, taskWeight, totalTaskWeight);

        // ASSERT
        Assert.Equal(0.0, taskStake);
    }

    [Fact]
    public void ComputePlanningAccuracy_AllAsPlanned_ReturnsOne()
    {
        // ═══════════════════════════════════════════════════════════════════
        // EQUATION: planning_accuracy = as_planned / total
        // ═══════════════════════════════════════════════════════════════════
        
        // ARRANGE
        int tasksAsPlanned = 10;
        int tasksDeviated = 0;

        // ACT
        var accuracy = TaskDecomposition.ComputePlanningAccuracy(
            tasksAsPlanned, tasksDeviated);

        // ASSERT
        Assert.Equal(1.0, accuracy);
    }

    [Fact]
    public void ComputePlanningAccuracy_HalfDeviated_ReturnsHalf()
    {
        // ═══════════════════════════════════════════════════════════════════
        // EQUATION: planning_accuracy = 5 / 10 = 0.5
        // ═══════════════════════════════════════════════════════════════════
        
        // ARRANGE
        int tasksAsPlanned = 5;
        int tasksDeviated = 5;

        // ACT
        var accuracy = TaskDecomposition.ComputePlanningAccuracy(
            tasksAsPlanned, tasksDeviated);

        // ASSERT
        Assert.Equal(0.5, accuracy);
    }

    [Fact]
    public void ComputePlanningAccuracy_NoTasks_ReturnsOne()
    {
        // ═══════════════════════════════════════════════════════════════════
        // Edge case: No tasks = vacuously true (100% accuracy)
        // ═══════════════════════════════════════════════════════════════════
        
        // ARRANGE
        int tasksAsPlanned = 0;
        int tasksDeviated = 0;

        // ACT
        var accuracy = TaskDecomposition.ComputePlanningAccuracy(
            tasksAsPlanned, tasksDeviated);

        // ASSERT
        Assert.Equal(1.0, accuracy);
    }

    [Fact]
    public void PhaseWithTasks_AggregateProviderContribution_SingleProvider()
    {
        // ═══════════════════════════════════════════════════════════════════
        // EQUATION: contribution = Σ(phase_weight × stake_ratio × task_outcome)
        // ═══════════════════════════════════════════════════════════════════
        
        // ARRANGE
        var provider = new Agent { SkillType = SkillTypes.Engineering };
        var consumer = new Agent { SkillType = SkillTypes.Engineering };
        
        var phaseContract = Contract.Create(
            provider: provider,
            consumer: consumer,
            skillType: SkillTypes.Engineering,
            stake: 1000,
            difficulty: 5,
            deadline: DateTime.UtcNow.AddDays(30),
            weight: 2.0
        );

        var phase = new PhaseWithTasks
        {
            PhaseContract = phaseContract,
            Tasks = new[]
            {
                new Task { Provider = provider, Weight = 1.0, Outcome = 1.0 },
                new Task { Provider = provider, Weight = 1.0, Outcome = 0.5 },
            }
        };

        // ACT
        var contribution = phase.AggregateProviderContribution(provider);

        // ASSERT: Both tasks belong to this provider
        Assert.True(contribution > 0, "Provider with positive outcomes should have positive contribution");
    }

    [Fact]
    public void PhaseWithTasks_AggregateProviderContribution_TeamBased()
    {
        // ═══════════════════════════════════════════════════════════════════
        // Team-based implementation: Different providers for different tasks
        // Each provider only gets credit for their own tasks
        // ═══════════════════════════════════════════════════════════════════
        
        // ARRANGE
        var alice = new Agent { SkillType = SkillTypes.Engineering };
        var bob = new Agent { SkillType = SkillTypes.Engineering };
        var consumer = new Agent { SkillType = SkillTypes.Engineering };
        
        var phaseContract = Contract.Create(
            provider: alice,  // Phase "owner" but tasks split
            consumer: consumer,
            skillType: SkillTypes.Engineering,
            stake: 1000,
            difficulty: 5,
            deadline: DateTime.UtcNow.AddDays(30),
            weight: 2.0
        );

        var phase = new PhaseWithTasks
        {
            PhaseContract = phaseContract,
            Tasks = new[]
            {
                new Task { Provider = alice, Weight = 1.0, Outcome = 1.0 },  // Alice's task
                new Task { Provider = bob, Weight = 1.0, Outcome = -0.5 },   // Bob's task
            }
        };

        // ACT
        var aliceContribution = phase.AggregateProviderContribution(alice);
        var bobContribution = phase.AggregateProviderContribution(bob);

        // ASSERT
        Assert.True(aliceContribution > 0, "Alice (success) should have positive contribution");
        Assert.True(bobContribution < 0, "Bob (failure) should have negative contribution");
        Assert.NotEqual(aliceContribution, bobContribution);
    }

    [Fact]
    public void PhaseWithTasks_ComputeAggregateOutcome_WeightedAverage()
    {
        // ═══════════════════════════════════════════════════════════════════
        // EQUATION: phase_outcome = Σ(w_i × outcome_i) / Σ w_i
        // ═══════════════════════════════════════════════════════════════════
        
        // ARRANGE
        var provider = new Agent { SkillType = SkillTypes.Engineering };
        var consumer = new Agent { SkillType = SkillTypes.Engineering };
        
        var phaseContract = Contract.Create(
            provider: provider,
            consumer: consumer,
            skillType: SkillTypes.Engineering,
            stake: 1000,
            difficulty: 5,
            deadline: DateTime.UtcNow.AddDays(30)
        );

        var phase = new PhaseWithTasks
        {
            PhaseContract = phaseContract,
            Tasks = new[]
            {
                new Task { Provider = provider, Weight = 2.0, Outcome = 1.0 },  // +2.0
                new Task { Provider = provider, Weight = 1.0, Outcome = -1.0 }, // -1.0
            }
            // Total weight = 3.0, weighted sum = 2.0 - 1.0 = 1.0
            // Average = 1.0 / 3.0 ≈ 0.333
        };

        // ACT
        var outcome = phase.ComputeAggregateOutcome();

        // ASSERT
        Assert.Equal(1.0 / 3.0, outcome, precision: 3);
    }

    [Fact]
    public void AggregateAllProviderContributions_ReturnsDictionary()
    {
        // ═══════════════════════════════════════════════════════════════════
        // Team aggregation: Dictionary mapping provider → contribution
        // ═══════════════════════════════════════════════════════════════════
        
        // ARRANGE
        var alice = new Agent { SkillType = SkillTypes.Engineering };
        var bob = new Agent { SkillType = SkillTypes.Engineering };
        var consumer = new Agent { SkillType = SkillTypes.Engineering };
        
        var phaseContract = Contract.Create(
            provider: alice,
            consumer: consumer,
            skillType: SkillTypes.Engineering,
            stake: 1000,
            difficulty: 5,
            deadline: DateTime.UtcNow.AddDays(30),
            weight: 2.0
        );

        var phase = new PhaseWithTasks
        {
            PhaseContract = phaseContract,
            Tasks = new[]
            {
                new Task { Provider = alice, Weight = 1.0, Outcome = 1.0 },
                new Task { Provider = alice, Weight = 1.0, Outcome = 0.8 },
                new Task { Provider = bob, Weight = 2.0, Outcome = 0.5 },
            }
        };

        // ACT
        var contributions = TaskDecomposition.AggregateAllProviderContributions(phase);

        // ASSERT
        Assert.Equal(2, contributions.Count);
        Assert.True(contributions.ContainsKey(alice.Id));
        Assert.True(contributions.ContainsKey(bob.Id));
    }
}

```

---

## 17. Customer Profile Tests

Tests for CustomerProfile computed properties and behavior aggregation.

```csharp

/// <summary>
/// Tests for CustomerProfile computed properties.
/// </summary>
public class CustomerProfileTests
{
    [Fact]
    public void CommitmentRate_ComputedCorrectly()
    {
        // ARRANGE
        var profile = new CustomerProfile
        {
            Customer = new Agent { SkillType = SkillTypes.Engineering },
            ProjectsInitiated = 20,
            ProjectsCompleted = 15
        };

        // ACT & ASSERT
        Assert.Equal(0.75, profile.CommitmentRate);
    }

    [Fact]
    public void FundingReliability_ComputedCorrectly()
    {
        // ARRANGE
        var profile = new CustomerProfile
        {
            Customer = new Agent { SkillType = SkillTypes.Engineering },
            OnTimeFundings = 8,
            TotalFundings = 10
        };

        // ACT & ASSERT
        Assert.Equal(0.8, profile.FundingReliability);
    }

    [Fact]
    public void ScopeStability_ComputedCorrectly()
    {
        // ARRANGE
        var profile = new CustomerProfile
        {
            Customer = new Agent { SkillType = SkillTypes.Engineering },
            ScopeTasksAsPlanned = 45,
            ScopeTotalTasks = 50
        };

        // ACT & ASSERT
        Assert.Equal(0.9, profile.ScopeStability);
    }

    [Fact]
    public void AvgTimelineAccuracy_ComputedCorrectly()
    {
        // ARRANGE
        var profile = new CustomerProfile
        {
            Customer = new Agent { SkillType = SkillTypes.Engineering },
            TimelineAccuracySum = 5.0,
            TimelineProjectsCount = 10
        };

        // ACT & ASSERT
        Assert.Equal(0.5, profile.AvgTimelineAccuracy);
    }

    [Fact]
    public void AllRates_ZeroDenominator_ReturnZero()
    {
        // Edge case: Empty profile should return zeros, not throw
        var profile = new CustomerProfile
        {
            Customer = new Agent { SkillType = SkillTypes.Engineering }
        };

        Assert.Equal(0.0, profile.CommitmentRate);
        Assert.Equal(0.0, profile.FundingReliability);
        Assert.Equal(0.0, profile.ScopeStability);
        Assert.Equal(0.0, profile.AvgTimelineAccuracy);
    }
}

```

---

## 18. Hierarchical Contract Tests

Tests for hierarchical contract structure (Project → Phase → Milestone → Task).

```csharp

/// <summary>
/// Tests for hierarchical contract structure.
/// 
/// Hierarchy: Project → Phase → Milestone → Task
/// Trust flows through tasks. Milestones are coordination containers (payment gates).
/// </summary>
public class HierarchicalContractTests
{
    [Fact]
    public void Contract_TypeDefaults_ToStandalone()
    {
        // ARRANGE & ACT
        var contract = ContractFactory.CreateSimple(
            skillType: SkillTypes.Engineering,
            outcome: 1.0,
            weight: 1.0
        );

        // ASSERT
        Assert.Equal(ContractType.Standalone, contract.Type);
    }

    [Fact]
    public void Contract_TaskType_RequiresParentRef()
    {
        // ARRANGE: Task contracts must reference their parent milestone
        var provider = new Agent { SkillType = SkillTypes.Engineering };
        var consumer = new Agent { SkillType = SkillTypes.Engineering };
        
        var taskContract = new Contract
        {
            Provider = provider,
            Consumer = consumer,
            SkillType = SkillTypes.Engineering,
            Stake = 100,
            Difficulty = 5,
            Deadline = DateTime.UtcNow.AddDays(7),
            Type = ContractType.Task,
            ParentRef = null  // Missing parent milestone!
        };

        // ACT
        var errors = taskContract.Validate();

        // ASSERT: Task must have ParentRef pointing to parent milestone
        Assert.Contains(errors, e => e.Contains("ParentRef"));
    }

    [Fact]
    public void Contract_PlanningAccuracy_ComputedFromFields()
    {
        // ARRANGE
        var provider = new Agent { SkillType = SkillTypes.Engineering };
        var consumer = new Agent { SkillType = SkillTypes.Engineering };
        
        var implContract = new Contract
        {
            Provider = provider,
            Consumer = consumer,
            SkillType = SkillTypes.Engineering,
            Stake = 1000,
            Difficulty = 5,
            Deadline = DateTime.UtcNow.AddDays(30),
            Type = ContractType.Implementation,
            TasksCompletedAsPlanned = 8,
            TasksDeviated = 2
        };

        // ACT
        var accuracy = implContract.PlanningAccuracy;

        // ASSERT: 8 / (8 + 2) = 0.8
        Assert.Equal(0.8, accuracy);
    }

    [Fact]
    public void Contract_PlanningAccuracy_NoTasks_ReturnsOne()
    {
        // ARRANGE
        var provider = new Agent { SkillType = SkillTypes.Engineering };
        var consumer = new Agent { SkillType = SkillTypes.Engineering };
        
        var implContract = new Contract
        {
            Provider = provider,
            Consumer = consumer,
            SkillType = SkillTypes.Engineering,
            Stake = 1000,
            Difficulty = 5,
            Deadline = DateTime.UtcNow.AddDays(30),
            Type = ContractType.Implementation,
            TasksCompletedAsPlanned = 0,
            TasksDeviated = 0
        };

        // ACT
        var accuracy = implContract.PlanningAccuracy;

        // ASSERT: 0 / 0 = 1.0 (vacuously true)
        Assert.Equal(1.0, accuracy);
    }

    [Fact]
    public void Task_ComputeStake_ProportionalToWeight()
    {
        // ARRANGE
        var provider = new Agent { SkillType = SkillTypes.Engineering };
        var task = new Task
        {
            Provider = provider,
            Weight = 2.0,
            Outcome = 1.0
        };

        // ACT: Task stake relative to parent milestone
        var taskStake = task.ComputeStake(milestoneStake: 1000, totalTaskWeight: 5.0);

        // ASSERT: 1000 × (2 / 5) = 400
        Assert.Equal(400.0, taskStake);
    }

    [Theory]
    [InlineData(ContractType.Standalone)]
    [InlineData(ContractType.Specification)]
    [InlineData(ContractType.Planning)]
    [InlineData(ContractType.Implementation)]
    [InlineData(ContractType.Task)]
    [InlineData(ContractType.Milestone)]
    public void ContractType_AllValuesValid(ContractType type)
    {
        // ARRANGE
        var provider = new Agent { SkillType = SkillTypes.Engineering };
        var consumer = new Agent { SkillType = SkillTypes.Engineering };
        
        // Task and Milestone types require ParentRef
        Guid? parentRef = (type == ContractType.Task || type == ContractType.Milestone) 
            ? Guid.NewGuid() 
            : null;
        
        var contract = new Contract
        {
            Provider = provider,
            Consumer = consumer,
            SkillType = SkillTypes.Engineering,
            Stake = 100,
            Difficulty = 5,
            Deadline = DateTime.UtcNow.AddDays(7),
            Type = type,
            ParentRef = parentRef
        };

        // ACT
        var errors = contract.Validate();

        // ASSERT
        Assert.Empty(errors);
    }
}

```

---

## 19. Milestone Tests

Tests for milestone payment gates within Implementation phases.

**Mathematical Notation:**

```
┌───────────────────────────────────────────────────────────────────────────────
│  s_milestone = Σ s_task                                                      │
│  d_milestone = max(d_task)  // deadline, not difficulty                      │
│  d_diff_milestone = Σ(d_task × s_task) / Σ s_task  // difficulty aggregation │
└───────────────────────────────────────────────────────────────────────────────
```

```csharp

/// <summary>
/// Tests for milestone payment gates.
/// 
/// Milestones are coordination containers within Implementation phases.
/// Trust flows through tasks, not milestones.
/// Customer reviews at milestone deadline: accept all, dispute specific, or timeout.
/// 
/// See: ADR_Milestone_Payment_Gates.md, ADR_Dispute_Resolution.md
/// </summary>
public class MilestoneTests
{
    [Fact]
    public void MilestoneWithTasks_ComputeStake_SumOfTaskStakes()
    {
        // ═══════════════════════════════════════════════════════════════════
        // EQUATION: s_milestone = Σ s_task
        // Milestone stake is not stored—it's computed from child tasks
        // ═══════════════════════════════════════════════════════════════════
        
        // ARRANGE
        var provider = new Agent { SkillType = SkillTypes.Engineering };
        
        var milestone = new MilestoneWithTasks
        {
            ParentPhaseId = Guid.NewGuid(),
            Tasks = new[]
            {
                new Task { Provider = provider, Weight = 2.0, Outcome = 1.0 },  // 400 sats
                new Task { Provider = provider, Weight = 1.0, Outcome = 0.9 },  // 200 sats
                new Task { Provider = provider, Weight = 2.0, Outcome = 0.8 },  // 400 sats
            }
        };
        
        double phaseStake = 1000;
        double totalPhaseTaskWeight = 5.0;

        // ACT
        var milestoneStake = milestone.ComputeStake(phaseStake, totalPhaseTaskWeight);

        // ASSERT: All tasks are in this milestone, so stake = 1000
        Assert.Equal(1000.0, milestoneStake);
    }

    [Fact]
    public void MilestoneWithTasks_ComputeDeadline_MaxOfTaskDeadlines()
    {
        // ═══════════════════════════════════════════════════════════════════
        // EQUATION: d_milestone = max(d_task)
        // Milestone deadline computed from latest task deadline
        // ═══════════════════════════════════════════════════════════════════
        
        // ARRANGE
        var provider = new Agent { SkillType = SkillTypes.Engineering };
        var baseDate = DateTime.UtcNow;
        
        var milestone = new MilestoneWithTasks
        {
            ParentPhaseId = Guid.NewGuid(),
            Tasks = new[]
            {
                new Task { Provider = provider, Weight = 1.0, Deadline = baseDate.AddDays(5) },
                new Task { Provider = provider, Weight = 1.0, Deadline = baseDate.AddDays(10) },  // Latest
                new Task { Provider = provider, Weight = 1.0, Deadline = baseDate.AddDays(7) },
            }
        };

        // ACT
        var milestoneDeadline = milestone.ComputeDeadline();

        // ASSERT: Deadline is the latest task deadline
        Assert.NotNull(milestoneDeadline);
        Assert.Equal(baseDate.AddDays(10), milestoneDeadline.Value);
    }

    [Fact]
    public void MilestoneWithTasks_ComputeDeadline_NoDeadlines_ReturnsNull()
    {
        // ═══════════════════════════════════════════════════════════════════
        // Edge case: Tasks without deadlines
        // ═══════════════════════════════════════════════════════════════════
        
        // ARRANGE
        var provider = new Agent { SkillType = SkillTypes.Engineering };
        
        var milestone = new MilestoneWithTasks
        {
            ParentPhaseId = Guid.NewGuid(),
            Tasks = new[]
            {
                new Task { Provider = provider, Weight = 1.0, Deadline = null },
                new Task { Provider = provider, Weight = 1.0, Deadline = null },
            }
        };

        // ACT
        var milestoneDeadline = milestone.ComputeDeadline();

        // ASSERT
        Assert.Null(milestoneDeadline);
    }

    [Fact]
    public void MilestoneWithTasks_ComputeAggregateDifficulty_StakeWeightedAverage()
    {
        // ═══════════════════════════════════════════════════════════════════
        // EQUATION: d_milestone = Σ(d_task × w_task) / Σ w_task
        // ═══════════════════════════════════════════════════════════════════
        
        // ARRANGE
        var provider = new Agent { SkillType = SkillTypes.Engineering };
        
        var milestone = new MilestoneWithTasks
        {
            ParentPhaseId = Guid.NewGuid(),
            Tasks = new[]
            {
                new Task { Provider = provider, Weight = 2.0, Difficulty = 8.0, Outcome = 1.0 },
                new Task { Provider = provider, Weight = 1.0, Difficulty = 5.0, Outcome = 1.0 },
            }
            // Weighted difficulty = (2×8 + 1×5) / (2+1) = 21/3 = 7.0
        };

        // ACT
        var difficulty = milestone.ComputeAggregateDifficulty();

        // ASSERT
        Assert.Equal(7.0, difficulty);
    }

    [Fact]
    public void MilestoneWithTasks_AggregateProviderContribution_SingleProvider()
    {
        // ═══════════════════════════════════════════════════════════════════
        // Provider contribution aggregated from tasks within milestone
        // ═══════════════════════════════════════════════════════════════════
        
        // ARRANGE
        var provider = new Agent { SkillType = SkillTypes.Engineering };
        
        var milestone = new MilestoneWithTasks
        {
            ParentPhaseId = Guid.NewGuid(),
            Tasks = new[]
            {
                new Task { Provider = provider, Weight = 1.0, Outcome = 1.0 },
                new Task { Provider = provider, Weight = 1.0, Outcome = 0.8 },
            }
        };

        // ACT
        var contribution = milestone.AggregateProviderContribution(provider, milestoneWeight: 2.0);

        // ASSERT
        Assert.True(contribution > 0, "Provider with positive outcomes should have positive contribution");
    }

    [Fact]
    public void MilestoneWithTasks_AggregateProviderContribution_TeamBased()
    {
        // ═══════════════════════════════════════════════════════════════════
        // Team-based: Multiple providers in same milestone
        // Each provider only gets credit for their own tasks
        // ═══════════════════════════════════════════════════════════════════
        
        // ARRANGE
        var alice = new Agent { SkillType = SkillTypes.Engineering };
        var bob = new Agent { SkillType = SkillTypes.Engineering };
        
        var milestone = new MilestoneWithTasks
        {
            ParentPhaseId = Guid.NewGuid(),
            Tasks = new[]
            {
                new Task { Provider = alice, Weight = 1.0, Outcome = 1.0 },   // Alice's task
                new Task { Provider = bob, Weight = 1.0, Outcome = -0.5 },    // Bob's task
            }
        };

        // ACT
        var aliceContribution = milestone.AggregateProviderContribution(alice, milestoneWeight: 2.0);
        var bobContribution = milestone.AggregateProviderContribution(bob, milestoneWeight: 2.0);

        // ASSERT
        Assert.True(aliceContribution > 0, "Alice (success) should have positive contribution");
        Assert.True(bobContribution < 0, "Bob (failure) should have negative contribution");
    }

    [Fact]
    public void Contract_MilestoneType_RequiresParentRef()
    {
        // ═══════════════════════════════════════════════════════════════════
        // Milestone contracts must reference their parent implementation phase
        // ═══════════════════════════════════════════════════════════════════
        
        // ARRANGE
        var provider = new Agent { SkillType = SkillTypes.Engineering };
        var consumer = new Agent { SkillType = SkillTypes.Engineering };
        
        var milestoneContract = new Contract
        {
            Provider = provider,
            Consumer = consumer,
            SkillType = SkillTypes.Engineering,
            Stake = 500,
            Difficulty = 5,
            Deadline = DateTime.UtcNow.AddDays(14),
            Type = ContractType.Milestone,
            ParentRef = null  // Missing parent implementation phase!
        };

        // ACT
        var errors = milestoneContract.Validate();

        // ASSERT
        Assert.Contains(errors, e => e.Contains("ParentRef"));
    }

    [Fact]
    public void PhaseWithTasks_WithMilestones_AggregatesAcrossMilestones()
    {
        // ═══════════════════════════════════════════════════════════════════
        // PhaseWithTasks can contain milestones, each with its own tasks
        // Provider contribution aggregates across all milestones
        // ═══════════════════════════════════════════════════════════════════
        
        // ARRANGE
        var alice = new Agent { SkillType = SkillTypes.Engineering };
        var consumer = new Agent { SkillType = SkillTypes.Engineering };
        
        var phaseContract = Contract.Create(
            provider: alice,
            consumer: consumer,
            skillType: SkillTypes.Engineering,
            stake: 1000,
            difficulty: 5,
            deadline: DateTime.UtcNow.AddDays(30),
            weight: 2.0
        );

        var phase = new PhaseWithTasks
        {
            PhaseContract = phaseContract,
            Milestones = new[]
            {
                new MilestoneWithTasks
                {
                    ParentPhaseId = phaseContract.Id,
                    Tasks = new[]
                    {
                        new Task { Provider = alice, Weight = 1.0, Outcome = 1.0 },
                    }
                },
                new MilestoneWithTasks
                {
                    ParentPhaseId = phaseContract.Id,
                    Tasks = new[]
                    {
                        new Task { Provider = alice, Weight = 1.0, Outcome = 0.8 },
                    }
                }
            }
        };

        // ACT
        var contribution = phase.AggregateProviderContribution(alice);

        // ASSERT: Contribution spans both milestones
        Assert.True(contribution > 0, "Provider with positive outcomes should have positive contribution");
    }

    [Fact]
    public void PhaseWithTasks_TotalTaskWeight_IncludesMilestones()
    {
        // ═══════════════════════════════════════════════════════════════════
        // TotalTaskWeight includes tasks from all milestones
        // ═══════════════════════════════════════════════════════════════════
        
        // ARRANGE
        var provider = new Agent { SkillType = SkillTypes.Engineering };
        var consumer = new Agent { SkillType = SkillTypes.Engineering };
        
        var phaseContract = Contract.Create(
            provider: provider,
            consumer: consumer,
            skillType: SkillTypes.Engineering,
            stake: 1000,
            difficulty: 5,
            deadline: DateTime.UtcNow.AddDays(30)
        );

        var phase = new PhaseWithTasks
        {
            PhaseContract = phaseContract,
            Milestones = new[]
            {
                new MilestoneWithTasks
                {
                    Tasks = new[]
                    {
                        new Task { Provider = provider, Weight = 2.0, Outcome = 1.0 },
                        new Task { Provider = provider, Weight = 1.0, Outcome = 1.0 },
                    }
                },
                new MilestoneWithTasks
                {
                    Tasks = new[]
                    {
                        new Task { Provider = provider, Weight = 2.0, Outcome = 1.0 },
                    }
                }
            }
        };

        // ACT
        var totalWeight = phase.TotalTaskWeight;

        // ASSERT: 2 + 1 + 2 = 5
        Assert.Equal(5.0, totalWeight);
    }

    [Fact]
    public void Task_ParentMilestoneId_TracksHierarchy()
    {
        // ═══════════════════════════════════════════════════════════════════
        // Tasks reference their parent milestone via ParentMilestoneId
        // ═══════════════════════════════════════════════════════════════════
        
        // ARRANGE
        var provider = new Agent { SkillType = SkillTypes.Engineering };
        var milestoneId = Guid.NewGuid();
        
        var task = new Task
        {
            Provider = provider,
            ParentMilestoneId = milestoneId,
            Weight = 1.0,
            Outcome = 1.0,
            Deadline = DateTime.UtcNow.AddDays(7)
        };

        // ACT & ASSERT
        Assert.Equal(milestoneId, task.ParentMilestoneId);
    }

    [Fact]
    public void Task_Deadline_IsWorkDuration_NotPaymentTrigger()
    {
        // ═══════════════════════════════════════════════════════════════════
        // Task deadline is work duration allocation (planning data)
        // NOT a payment trigger—payment happens at milestone deadline
        // See: ADR_Milestone_Payment_Gates.md
        // ═══════════════════════════════════════════════════════════════════
        
        // ARRANGE
        var provider = new Agent { SkillType = SkillTypes.Engineering };
        var taskDeadline = DateTime.UtcNow.AddDays(7);
        
        var task = new Task
        {
            Provider = provider,
            Weight = 1.0,
            Outcome = 1.0,
            Deadline = taskDeadline  // Work duration deadline
        };

        // ACT & ASSERT
        Assert.Equal(taskDeadline, task.Deadline);
        // Note: This is planning data. Customer reviews at milestone deadline, not per-task.
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