# Sybil Resistance Implementation in C#

## Supplementary Implementation for Defense-in-Depth

---

This document extends the base Quantum of Trust C# implementation with four Sybil resistance mechanisms. It should be read alongside `Quantum_of_Trust_Equations_in_CSharp.md`.

---

## Table of Contents

1. [Overview](#overview)
2. [Configuration](#configuration)
3. [Enhanced Contract Model](#enhanced-contract-model)
4. [Counterparty Trust Factor](#counterparty-trust-factor)
5. [Velocity Weight Calculator](#velocity-weight-calculator)
6. [Outcome Variance Analyzer](#outcome-variance-analyzer)
7. [Escrow Verification](#escrow-verification)
8. [Enhanced Trust Calculator](#enhanced-trust-calculator)
9. [Combined Eligibility Service](#combined-eligibility-service)
10. [Unit Tests](#unit-tests)

---

## Overview

The base implementation provides Sybil resistance through history size, depth, and counterparty diversity checks. This extension adds four complementary mechanisms that compose into robust defense.

```csharp
/// <summary>
/// Summary of Sybil resistance mechanisms.
/// </summary>
/// <remarks>
/// <list type="table">
///   <listheader>
///     <term>Mechanism</term>
///     <description>Purpose</description>
///   </listheader>
///   <item>
///     <term>Economic Escrow</term>
///     <description>Proves real funds are locked for each contract</description>
///   </item>
///   <item>
///     <term>Counterparty Weighting</term>
///     <description>Scales contribution by counterparty trust</description>
///   </item>
///   <item>
///     <term>Outcome Variance</term>
///     <description>Catches suspiciously uniform outcomes</description>
///   </item>
///   <item>
///     <term>Velocity Limits</term>
///     <description>Prevents burst trust accumulation</description>
///   </item>
/// </list>
/// </remarks>
```

---

## Configuration

```csharp
/// <summary>
/// Configuration parameters for Sybil resistance mechanisms.
/// </summary>
public static class SybilResistanceConfig
{
    // ═══════════════════════════════════════════════════════════════
    // COUNTERPARTY FACTOR PARAMETERS
    // ═══════════════════════════════════════════════════════════════
    
    /// <summary>
    /// Scale parameter λ for counterparty trust sigmoid.
    /// Controls how quickly γ rises with counterparty trust.
    /// At λ=50: γ(0)=0.5, γ(50)≈0.73, γ(100)≈0.88
    /// </summary>
    public const double CounterpartySigmoidScale = 50.0;
    
    // ═══════════════════════════════════════════════════════════════
    // VELOCITY PARAMETERS
    // ═══════════════════════════════════════════════════════════════
    
    /// <summary>
    /// Time period for velocity checks.
    /// </summary>
    public static readonly TimeSpan VelocityPeriod = TimeSpan.FromDays(7);
    
    /// <summary>
    /// Number of full-weight contracts allowed per velocity period.
    /// </summary>
    public const int VelocityAllowance = 10;
    
    /// <summary>
    /// Decay rate k for contracts beyond velocity allowance.
    /// k=0.5 means contract 11 has weight 0.67, contract 15 has weight 0.29
    /// </summary>
    public const double VelocityDecayRate = 0.5;
    
    // ═══════════════════════════════════════════════════════════════
    // VARIANCE PARAMETERS
    // ═══════════════════════════════════════════════════════════════
    
    /// <summary>
    /// Minimum history size before variance is checked.
    /// Smaller histories are exempt from the plausibility check.
    /// </summary>
    public const int VarianceExemptionSize = 10;
    
    /// <summary>
    /// Minimum required outcome variance.
    /// 0.1 variance requires some contracts to deviate from the mean.
    /// </summary>
    public const double MinimumVariance = 0.1;
    
    /// <summary>
    /// Threshold for "imperfect" outcomes in simplified variance check.
    /// Outcomes below this value (e.g., 0.8) count as imperfect.
    /// </summary>
    public const double ImperfectOutcomeThreshold = 0.8;
    
    /// <summary>
    /// Fraction of contracts (above exemption) that must be imperfect.
    /// 0.2 means 20% must have outcome below ImperfectOutcomeThreshold.
    /// </summary>
    public const double RequiredImperfectFraction = 0.2;
}
```

---

## Enhanced Contract Model

```csharp
/// <summary>
/// A completed contract with Sybil resistance fields.
/// </summary>
/// <remarks>
/// Extends the base Contract with:
/// - EscrowCommitment: Cryptographic proof of locked stake
/// - CounterpartyTrustSnapshot: Consumer's trust at completion time
/// </remarks>
public class EnhancedContract : Contract
{
    // ═══════════════════════════════════════════════════════════════
    // MATH: c = (provider, consumer, t, s, d, τ, ε, V_consumer)
    // ═══════════════════════════════════════════════════════════════
    
    /// <summary>
    /// Cryptographic commitment proving stake is locked.
    /// In production: hash of the escrow note or transaction ID.
    /// </summary>
    public string EscrowCommitment { get; init; } = string.Empty;
    
    /// <summary>
    /// The counterparty's trust value when the contract completed.
    /// Stored as a snapshot to avoid circular dependencies.
    /// </summary>
    public double CounterpartyTrustSnapshot { get; init; }
    
    /// <summary>
    /// Creates an enhanced contract from a base contract.
    /// </summary>
    public static EnhancedContract FromBase(
        Contract baseContract,
        string escrowCommitment,
        double counterpartyTrust)
    {
        return new EnhancedContract
        {
            Provider = baseContract.Provider,
            Consumer = baseContract.Consumer,
            SkillType = baseContract.SkillType,
            Stake = baseContract.Stake,
            Difficulty = baseContract.Difficulty,
            Outcome = baseContract.Outcome,
            CompletedAt = baseContract.CompletedAt,
            Weight = baseContract.Weight,
            EscrowCommitment = escrowCommitment,
            CounterpartyTrustSnapshot = counterpartyTrust
        };
    }
}

/// <summary>
/// Escrow data backing a contract stake.
/// </summary>
public class EscrowNote
{
    /// <summary>
    /// The amount of tokens locked.
    /// </summary>
    public decimal Amount { get; init; }
    
    /// <summary>
    /// Owner of the escrow (provider or consumer ID).
    /// </summary>
    public string OwnerId { get; init; } = string.Empty;
    
    /// <summary>
    /// Unique identifier/hash of the escrow note.
    /// </summary>
    public string NoteHash { get; init; } = string.Empty;
    
    /// <summary>
    /// Whether the escrow is currently locked.
    /// </summary>
    public bool IsLocked { get; init; }
}
```

---

## Counterparty Trust Factor

```csharp
/// <summary>
/// Calculates the counterparty trust factor γ(c).
/// </summary>
/// <remarks>
/// MATH: γ(c) = σ(V_t(counterparty) / λ)
/// 
/// PLAIN ENGLISH: "Contracts with high-trust counterparties count more.
/// If your counterparty has zero or negative trust, your contract with
/// them contributes very little—even if you performed perfectly."
/// </remarks>
public static class CounterpartyFactorCalculator
{
    /// <summary>
    /// Computes the sigmoid function σ(x) = 1 / (1 + e^(-x)).
    /// </summary>
    public static double Sigmoid(double x)
    {
        return 1.0 / (1.0 + Math.Exp(-x));
    }
    
    /// <summary>
    /// Computes the counterparty trust factor γ(c).
    /// </summary>
    /// <param name="counterpartyTrust">The counterparty's trust value.</param>
    /// <returns>Factor in range (0, 1).</returns>
    public static double ComputeCounterpartyFactor(double counterpartyTrust)
    {
        // ═══════════════════════════════════════════════════════════════
        // MATH: γ(c) = σ(V_t(counterparty) / λ)
        // ═══════════════════════════════════════════════════════════════
        
        double scaledInput = counterpartyTrust / SybilResistanceConfig.CounterpartySigmoidScale;
        return Sigmoid(scaledInput);
    }
    
    /// <summary>
    /// Gets the counterparty factor for an enhanced contract.
    /// </summary>
    public static double GetFactor(EnhancedContract contract)
    {
        return ComputeCounterpartyFactor(contract.CounterpartyTrustSnapshot);
    }
}
```

---

## Velocity Weight Calculator

```csharp
/// <summary>
/// Calculates velocity weights to prevent burst accumulation.
/// </summary>
/// <remarks>
/// MATH: ν(c) = 1 / (1 + k × max(0, rank - N))
/// 
/// PLAIN ENGLISH: "Your first N contracts in any time period count fully.
/// After that, additional contracts have diminishing impact."
/// </remarks>
public static class VelocityWeightCalculator
{
    /// <summary>
    /// Computes the velocity weight for a given rank in a period.
    /// </summary>
    /// <param name="rankInPeriod">The contract's position (1-indexed).</param>
    /// <returns>Weight in range (0, 1].</returns>
    public static double ComputeVelocityWeight(int rankInPeriod)
    {
        // ═══════════════════════════════════════════════════════════════
        // MATH: ν(c) = 1 / (1 + k × max(0, rank - N))
        // ═══════════════════════════════════════════════════════════════
        
        if (rankInPeriod <= SybilResistanceConfig.VelocityAllowance)
        {
            return 1.0;
        }
        
        int excess = rankInPeriod - SybilResistanceConfig.VelocityAllowance;
        double penaltyTerm = SybilResistanceConfig.VelocityDecayRate * excess;
        return 1.0 / (1.0 + penaltyTerm);
    }
    
    /// <summary>
    /// Computes the rank of a contract within its velocity period.
    /// </summary>
    /// <param name="contracts">All contracts in the agent's history.</param>
    /// <param name="targetContract">The contract to rank.</param>
    /// <param name="skillType">Skill type to filter by.</param>
    /// <returns>Rank within the period (1 = first).</returns>
    public static int ComputeContractRank(
        IEnumerable<EnhancedContract> contracts,
        EnhancedContract targetContract,
        SkillTypes skillType)
    {
        var periodStart = targetContract.CompletedAt - SybilResistanceConfig.VelocityPeriod;
        
        return contracts
            .Where(c => c.SkillType == skillType)
            .Where(c => c.CompletedAt >= periodStart)
            .Where(c => c.CompletedAt <= targetContract.CompletedAt)
            .Count();
    }
    
    /// <summary>
    /// Gets the velocity weight for a contract given its history context.
    /// </summary>
    public static double GetWeight(
        IEnumerable<EnhancedContract> history,
        EnhancedContract contract,
        SkillTypes skillType)
    {
        int rank = ComputeContractRank(history, contract, skillType);
        return ComputeVelocityWeight(rank);
    }
}
```

---

## Outcome Variance Analyzer

```csharp
/// <summary>
/// Analyzes outcome variance to detect suspiciously uniform histories.
/// </summary>
/// <remarks>
/// MATH: plausible(h) ≡ (|h| &lt; N_min) ∨ (var(outcomes) ≥ ε_min)
/// 
/// PLAIN ENGLISH: "Small histories get a pass. But if you have many
/// contracts, they can't all be perfect. A history of 50 contracts
/// all rated +1.0 is statistically implausible."
/// </remarks>
public static class OutcomeVarianceAnalyzer
{
    /// <summary>
    /// Computes the statistical variance of outcomes.
    /// </summary>
    public static double ComputeVariance(IEnumerable<double> outcomes)
    {
        var outcomeList = outcomes.ToList();
        if (outcomeList.Count == 0) return 0;
        
        double mean = outcomeList.Average();
        double sumSquaredDeviations = outcomeList
            .Sum(o => Math.Pow(o - mean, 2));
        
        return sumSquaredDeviations / outcomeList.Count;
    }
    
    /// <summary>
    /// Checks if a history is plausible (not suspiciously uniform).
    /// </summary>
    /// <param name="contracts">The agent's contracts.</param>
    /// <param name="skillType">Skill type to filter by.</param>
    /// <returns>True if history is plausible.</returns>
    public static bool IsHistoryPlausible(
        IEnumerable<EnhancedContract> contracts,
        SkillTypes skillType)
    {
        // ═══════════════════════════════════════════════════════════════
        // MATH: plausible(h) ≡ (|h| < N_min) ∨ (var(outcomes) ≥ ε_min)
        // ═══════════════════════════════════════════════════════════════
        
        var matchingContracts = contracts
            .Where(c => c.SkillType == skillType)
            .ToList();
        
        // Small histories are exempt
        if (matchingContracts.Count < SybilResistanceConfig.VarianceExemptionSize)
        {
            return true;
        }
        
        // Check variance
        var outcomes = matchingContracts.Select(c => c.Outcome);
        double variance = ComputeVariance(outcomes);
        
        return variance >= SybilResistanceConfig.MinimumVariance;
    }
    
    /// <summary>
    /// Simplified plausibility check using outcome diversity.
    /// </summary>
    /// <remarks>
    /// Instead of computing exact variance, checks that enough contracts
    /// have "imperfect" outcomes (below a threshold).
    /// </remarks>
    public static bool IsHistoryPlausibleSimple(
        IEnumerable<EnhancedContract> contracts,
        SkillTypes skillType)
    {
        var matchingContracts = contracts
            .Where(c => c.SkillType == skillType)
            .ToList();
        
        // Small histories are exempt
        if (matchingContracts.Count < SybilResistanceConfig.VarianceExemptionSize)
        {
            return true;
        }
        
        // Count imperfect outcomes
        int imperfectCount = matchingContracts
            .Count(c => c.Outcome < SybilResistanceConfig.ImperfectOutcomeThreshold);
        
        // Require minimum fraction of imperfect
        int excess = matchingContracts.Count - SybilResistanceConfig.VarianceExemptionSize;
        int required = (int)Math.Ceiling(excess * SybilResistanceConfig.RequiredImperfectFraction);
        
        return imperfectCount >= required;
    }
}
```

---

## Escrow Verification

```csharp
/// <summary>
/// Verifies escrow commitments for contracts.
/// </summary>
/// <remarks>
/// MATH: valid_escrow(c) ≡ verify(ε, s) ∧ locked(ε) ∧ owner(ε) ∈ {provider, consumer}
/// 
/// PLAIN ENGLISH: "The stake isn't just a number—it's a cryptographic
/// proof that real value is locked."
/// </remarks>
public static class EscrowVerifier
{
    /// <summary>
    /// Verifies that an escrow is valid for a contract.
    /// </summary>
    /// <param name="contract">The contract to verify.</param>
    /// <param name="escrow">The escrow note.</param>
    /// <returns>True if escrow is valid.</returns>
    public static bool VerifyEscrow(EnhancedContract contract, EscrowNote escrow)
    {
        // ═══════════════════════════════════════════════════════════════
        // MATH: valid_escrow(c) ≡ verify(ε, s) ∧ locked(ε) 
        //                        ∧ owner(ε) ∈ {provider, consumer}
        // ═══════════════════════════════════════════════════════════════
        
        // 1. Commitment matches note hash
        bool commitmentValid = contract.EscrowCommitment == escrow.NoteHash;
        
        // 2. Amount matches stake
        bool amountValid = escrow.Amount >= contract.Stake;
        
        // 3. Escrow is locked
        bool isLocked = escrow.IsLocked;
        
        // 4. Owner is provider or consumer
        bool ownerValid = escrow.OwnerId == contract.Provider.Id 
                       || escrow.OwnerId == contract.Consumer.Id;
        
        return commitmentValid && amountValid && isLocked && ownerValid;
    }
    
    /// <summary>
    /// Verifies escrows for all contracts in a history.
    /// </summary>
    public static bool VerifyAllEscrows(
        IEnumerable<EnhancedContract> contracts,
        IDictionary<string, EscrowNote> escrowsByCommitment,
        SkillTypes skillType)
    {
        foreach (var contract in contracts.Where(c => c.SkillType == skillType))
        {
            if (!escrowsByCommitment.TryGetValue(contract.EscrowCommitment, out var escrow))
            {
                return false; // Missing escrow
            }
            
            if (!VerifyEscrow(contract, escrow))
            {
                return false;
            }
        }
        
        return true;
    }
}
```

---

## Enhanced Trust Calculator

```csharp
/// <summary>
/// Calculates trust values with all Sybil resistance factors.
/// </summary>
/// <remarks>
/// MATH: V_t(Agent) = Σ ω(c) × outcome(c) × γ(c) × ν(c)
/// 
/// PLAIN ENGLISH: "A contract contributes most when: it had high stakes
/// and difficulty, you performed well, your counterparty was reputable,
/// and you weren't rushing to accumulate contracts."
/// </remarks>
public class EnhancedTrustCalculator
{
    /// <summary>
    /// Calculates the full contribution of a single contract.
    /// </summary>
    public static double CalculateContractContribution(
        EnhancedContract contract,
        IEnumerable<EnhancedContract> fullHistory,
        SkillTypes skillType)
    {
        // ═══════════════════════════════════════════════════════════════
        // MATH: contribution = ω(c) × outcome(c) × γ(c) × ν(c)
        // ═══════════════════════════════════════════════════════════════
        
        // Base contribution: ω × outcome
        double baseContribution = contract.Weight * contract.Outcome;
        
        // Counterparty factor: γ(c)
        double gamma = CounterpartyFactorCalculator.GetFactor(contract);
        
        // Velocity factor: ν(c)
        double nu = VelocityWeightCalculator.GetWeight(fullHistory, contract, skillType);
        
        // Final contribution
        return baseContribution * gamma * nu;
    }
    
    /// <summary>
    /// Calculates the total trust value with all factors.
    /// </summary>
    /// <param name="contracts">The agent's contract history.</param>
    /// <param name="skillType">Skill type to calculate trust for.</param>
    /// <returns>Trust value (can be positive, zero, or negative).</returns>
    public static double CalculateTrustValue(
        IEnumerable<EnhancedContract> contracts,
        SkillTypes skillType)
    {
        // ═══════════════════════════════════════════════════════════════
        // MATH: V_t = Σ ω(c) × outcome(c) × γ(c) × ν(c)
        // ═══════════════════════════════════════════════════════════════
        
        var contractList = contracts.ToList();
        
        return contractList
            .Where(c => c.SkillType == skillType)
            .Sum(c => CalculateContractContribution(c, contractList, skillType));
    }
    
    /// <summary>
    /// Calculates trust with detailed breakdown for analysis.
    /// </summary>
    public static TrustBreakdown CalculateTrustDetailed(
        IEnumerable<EnhancedContract> contracts,
        SkillTypes skillType)
    {
        var contractList = contracts.ToList();
        var matchingContracts = contractList.Where(c => c.SkillType == skillType).ToList();
        
        var contributions = new List<ContractContribution>();
        double totalTrust = 0;
        
        foreach (var contract in matchingContracts)
        {
            double gamma = CounterpartyFactorCalculator.GetFactor(contract);
            double nu = VelocityWeightCalculator.GetWeight(contractList, contract, skillType);
            double contribution = contract.Weight * contract.Outcome * gamma * nu;
            
            contributions.Add(new ContractContribution
            {
                Contract = contract,
                BaseContribution = contract.Weight * contract.Outcome,
                CounterpartyFactor = gamma,
                VelocityFactor = nu,
                FinalContribution = contribution
            });
            
            totalTrust += contribution;
        }
        
        return new TrustBreakdown
        {
            TotalTrust = totalTrust,
            ContractCount = matchingContracts.Count,
            Contributions = contributions
        };
    }
}

/// <summary>
/// Detailed breakdown of a single contract's trust contribution.
/// </summary>
public class ContractContribution
{
    public EnhancedContract Contract { get; init; } = null!;
    public double BaseContribution { get; init; }
    public double CounterpartyFactor { get; init; }
    public double VelocityFactor { get; init; }
    public double FinalContribution { get; init; }
}

/// <summary>
/// Detailed breakdown of trust calculation.
/// </summary>
public class TrustBreakdown
{
    public double TotalTrust { get; init; }
    public int ContractCount { get; init; }
    public IReadOnlyList<ContractContribution> Contributions { get; init; } = Array.Empty<ContractContribution>();
}
```

---

## Combined Eligibility Service

```csharp
/// <summary>
/// Service for checking contract eligibility with all Sybil resistance mechanisms.
/// </summary>
/// <remarks>
/// MATH: eligible(a, c) ⟺ V_t(a) ≥ θ(c) ∧ plausible(h_t(a))
/// 
/// PLAIN ENGLISH: "To be eligible, you need enough trust AND a plausible history."
/// </remarks>
public class EnhancedEligibilityService
{
    /// <summary>
    /// Checks if an agent is eligible for a contract.
    /// </summary>
    /// <param name="agent">The agent to check.</param>
    /// <param name="contract">The contract to check eligibility for.</param>
    /// <param name="requirements">Optional custom requirements.</param>
    /// <returns>Eligibility result with details.</returns>
    public EligibilityResult CheckEligibility(
        IAgent agent,
        Contract contract,
        EligibilityRequirements? requirements = null)
    {
        requirements ??= EligibilityRequirements.Default;
        
        var history = agent.GetHistory()
            .OfType<EnhancedContract>()
            .ToList();
        
        var skillType = contract.SkillType;
        
        // ═══════════════════════════════════════════════════════════════
        // CHECK 1: Trust meets threshold
        // ═══════════════════════════════════════════════════════════════
        double trust = EnhancedTrustCalculator.CalculateTrustValue(history, skillType);
        double threshold = TrustValuation.CalculateThreshold(contract.Stake, contract.Difficulty);
        bool meetsThreshold = trust >= threshold;
        
        // ═══════════════════════════════════════════════════════════════
        // CHECK 2: History is plausible (variance)
        // ═══════════════════════════════════════════════════════════════
        bool isPlausible = OutcomeVarianceAnalyzer.IsHistoryPlausibleSimple(history, skillType);
        
        // ═══════════════════════════════════════════════════════════════
        // CHECK 3: History size
        // ═══════════════════════════════════════════════════════════════
        int historySize = history.Count(c => c.SkillType == skillType);
        bool meetsSize = historySize >= requirements.MinHistorySize;
        
        // ═══════════════════════════════════════════════════════════════
        // CHECK 4: History depth
        // ═══════════════════════════════════════════════════════════════
        var oldestContract = history
            .Where(c => c.SkillType == skillType)
            .OrderBy(c => c.CompletedAt)
            .FirstOrDefault();
        
        bool meetsDepth = oldestContract != null &&
            (DateTime.UtcNow - oldestContract.CompletedAt) >= requirements.MinHistoryDepth;
        
        // ═══════════════════════════════════════════════════════════════
        // CHECK 5: Counterparty diversity
        // ═══════════════════════════════════════════════════════════════
        int uniqueCounterparties = history
            .Where(c => c.SkillType == skillType)
            .Select(c => c.Consumer.Id)
            .Distinct()
            .Count();
        bool meetsDiversity = uniqueCounterparties >= requirements.MinCounterpartyDiversity;
        
        // ═══════════════════════════════════════════════════════════════
        // RESULT
        // ═══════════════════════════════════════════════════════════════
        bool isEligible = meetsThreshold && isPlausible && meetsSize && meetsDepth && meetsDiversity;
        
        return new EligibilityResult
        {
            IsEligible = isEligible,
            Trust = trust,
            Threshold = threshold,
            MeetsThreshold = meetsThreshold,
            IsPlausible = isPlausible,
            HistorySize = historySize,
            MeetsSize = meetsSize,
            MeetsDepth = meetsDepth,
            UniqueCounterparties = uniqueCounterparties,
            MeetsDiversity = meetsDiversity
        };
    }
}

/// <summary>
/// Requirements for eligibility checks.
/// </summary>
public class EligibilityRequirements
{
    public int MinHistorySize { get; init; } = 5;
    public TimeSpan MinHistoryDepth { get; init; } = TimeSpan.FromDays(30);
    public int MinCounterpartyDiversity { get; init; } = 3;
    
    public static EligibilityRequirements Default => new();
    
    public static EligibilityRequirements Strict => new()
    {
        MinHistorySize = 10,
        MinHistoryDepth = TimeSpan.FromDays(90),
        MinCounterpartyDiversity = 5
    };
}

/// <summary>
/// Result of an eligibility check.
/// </summary>
public class EligibilityResult
{
    public bool IsEligible { get; init; }
    
    // Trust check
    public double Trust { get; init; }
    public double Threshold { get; init; }
    public bool MeetsThreshold { get; init; }
    
    // Plausibility check
    public bool IsPlausible { get; init; }
    
    // Size check
    public int HistorySize { get; init; }
    public bool MeetsSize { get; init; }
    
    // Depth check
    public bool MeetsDepth { get; init; }
    
    // Diversity check
    public int UniqueCounterparties { get; init; }
    public bool MeetsDiversity { get; init; }
    
    /// <summary>
    /// Gets reasons why eligibility failed (if any).
    /// </summary>
    public IEnumerable<string> GetFailureReasons()
    {
        if (!MeetsThreshold)
            yield return $"Trust {Trust:F2} below threshold {Threshold:F2}";
        if (!IsPlausible)
            yield return "History failed plausibility check (suspicious outcome uniformity)";
        if (!MeetsSize)
            yield return $"History size {HistorySize} below minimum";
        if (!MeetsDepth)
            yield return "History too shallow (not enough time since first contract)";
        if (!MeetsDiversity)
            yield return $"Only {UniqueCounterparties} unique counterparties (need more diversity)";
    }
}
```

---

## Unit Tests

```csharp
/// <summary>
/// Tests for Sybil resistance mechanisms.
/// </summary>
public class SybilResistanceTests
{
    // ════════════════════════════════════════════════════════════════════
    // COUNTERPARTY FACTOR TESTS
    // ════════════════════════════════════════════════════════════════════
    
    [Theory]
    [InlineData(0, 0.5)]      // Zero trust → γ = 0.5
    [InlineData(50, 0.73)]    // Moderate trust → γ ≈ 0.73
    [InlineData(100, 0.88)]   // High trust → γ ≈ 0.88
    [InlineData(-50, 0.27)]   // Negative trust → γ ≈ 0.27
    [InlineData(-100, 0.12)]  // Very negative → γ ≈ 0.12
    public void CounterpartyFactor_ReturnsExpectedValues(
        double counterpartyTrust, 
        double expectedGamma)
    {
        // ═══════════════════════════════════════════════════════════════
        // MATH: γ(c) = σ(V_t(counterparty) / λ)
        // ═══════════════════════════════════════════════════════════════
        
        double gamma = CounterpartyFactorCalculator.ComputeCounterpartyFactor(counterpartyTrust);
        
        Assert.Equal(expectedGamma, gamma, precision: 2);
    }
    
    // ════════════════════════════════════════════════════════════════════
    // VELOCITY WEIGHT TESTS
    // ════════════════════════════════════════════════════════════════════
    
    [Theory]
    [InlineData(1, 1.0)]      // First contract → full weight
    [InlineData(10, 1.0)]     // At allowance → full weight
    [InlineData(11, 0.67)]    // First excess → reduced
    [InlineData(15, 0.29)]    // More excess → further reduced
    [InlineData(20, 0.17)]    // Heavy burst → heavily reduced
    public void VelocityWeight_DiminishesForExcessContracts(
        int rank, 
        double expectedWeight)
    {
        // ═══════════════════════════════════════════════════════════════
        // MATH: ν(c) = 1 / (1 + k × max(0, rank - N))
        // ═══════════════════════════════════════════════════════════════
        
        double weight = VelocityWeightCalculator.ComputeVelocityWeight(rank);
        
        Assert.Equal(expectedWeight, weight, precision: 2);
    }
    
    // ════════════════════════════════════════════════════════════════════
    // VARIANCE TESTS
    // ════════════════════════════════════════════════════════════════════
    
    [Fact]
    public void OutcomeVariance_AllPerfect_ReturnsZero()
    {
        // ═══════════════════════════════════════════════════════════════
        // PLAIN ENGLISH: "50 contracts all rated +1.0 is suspicious"
        // ═══════════════════════════════════════════════════════════════
        
        var outcomes = Enumerable.Repeat(1.0, 50);
        
        double variance = OutcomeVarianceAnalyzer.ComputeVariance(outcomes);
        
        Assert.Equal(0, variance, precision: 5);
    }
    
    [Fact]
    public void OutcomeVariance_RealisticDistribution_PassesPlausibility()
    {
        var contracts = CreateContractsWithOutcomes(
            Enumerable.Range(0, 50)
                .Select(i => 0.5 + (i % 10) * 0.05) // Range from 0.5 to 0.95
        );
        
        bool isPlausible = OutcomeVarianceAnalyzer.IsHistoryPlausible(
            contracts, 
            SkillTypes.Engineering);
        
        Assert.True(isPlausible);
    }
    
    [Fact]
    public void OutcomeVariance_AllPerfectScores_FailsPlausibility()
    {
        var contracts = CreateContractsWithOutcomes(
            Enumerable.Repeat(1.0, 50)
        );
        
        bool isPlausible = OutcomeVarianceAnalyzer.IsHistoryPlausible(
            contracts, 
            SkillTypes.Engineering);
        
        Assert.False(isPlausible);
    }
    
    [Fact]
    public void OutcomeVariance_SmallHistory_ExemptFromCheck()
    {
        // ═══════════════════════════════════════════════════════════════
        // MATH: |h| < N_min → automatically plausible
        // ═══════════════════════════════════════════════════════════════
        
        var contracts = CreateContractsWithOutcomes(
            Enumerable.Repeat(1.0, 5) // Below exemption threshold
        );
        
        bool isPlausible = OutcomeVarianceAnalyzer.IsHistoryPlausible(
            contracts, 
            SkillTypes.Engineering);
        
        Assert.True(isPlausible);
    }
    
    // ════════════════════════════════════════════════════════════════════
    // COMBINED SYBIL SCENARIO TESTS
    // ════════════════════════════════════════════════════════════════════
    
    [Fact]
    public void SybilRingAttack_ProducesLowerTrustThanHonest()
    {
        // ═══════════════════════════════════════════════════════════════
        // SCENARIO: 20 Sybils each trade with all 19 others
        // Each Sybil has 19 contracts, all perfect outcomes
        // Compare to one honest agent with 19 contracts with varied counterparties
        // ═══════════════════════════════════════════════════════════════
        
        // Sybil: all counterparties have zero trust
        var sybilContracts = CreateSybilContracts(19, counterpartyTrust: 0);
        double sybilTrust = EnhancedTrustCalculator.CalculateTrustValue(
            sybilContracts, 
            SkillTypes.Engineering);
        
        // Honest: counterparties have established trust (avg 50)
        var honestContracts = CreateHonestContracts(19, avgCounterpartyTrust: 50);
        double honestTrust = EnhancedTrustCalculator.CalculateTrustValue(
            honestContracts, 
            SkillTypes.Engineering);
        
        // Honest should have significantly more trust
        Assert.True(honestTrust > sybilTrust * 1.5, 
            $"Honest trust {honestTrust:F2} should be >1.5x Sybil trust {sybilTrust:F2}");
    }
    
    [Fact]
    public void BurstAttack_DiminishedByVelocity()
    {
        // ═══════════════════════════════════════════════════════════════
        // SCENARIO: Attacker creates 50 contracts in one week
        // Compare to 50 contracts spread over months
        // ═══════════════════════════════════════════════════════════════
        
        var burstContracts = CreateBurstContracts(50, TimeSpan.FromDays(7));
        double burstTrust = EnhancedTrustCalculator.CalculateTrustValue(
            burstContracts, 
            SkillTypes.Engineering);
        
        var spreadContracts = CreateSpreadContracts(50, TimeSpan.FromDays(180));
        double spreadTrust = EnhancedTrustCalculator.CalculateTrustValue(
            spreadContracts, 
            SkillTypes.Engineering);
        
        // Spread should have significantly more trust
        Assert.True(spreadTrust > burstTrust * 2, 
            $"Spread trust {spreadTrust:F2} should be >2x burst trust {burstTrust:F2}");
    }
    
    // ════════════════════════════════════════════════════════════════════
    // HELPER METHODS
    // ════════════════════════════════════════════════════════════════════
    
    private static List<EnhancedContract> CreateContractsWithOutcomes(
        IEnumerable<double> outcomes)
    {
        return outcomes.Select((o, i) => new EnhancedContract
        {
            SkillType = SkillTypes.Engineering,
            Outcome = o,
            Weight = 1.0,
            CounterpartyTrustSnapshot = 50,
            CompletedAt = DateTime.UtcNow.AddDays(-i)
        }).ToList();
    }
    
    private static List<EnhancedContract> CreateSybilContracts(
        int count, 
        double counterpartyTrust)
    {
        return Enumerable.Range(0, count).Select(i => new EnhancedContract
        {
            SkillType = SkillTypes.Engineering,
            Outcome = 1.0, // Perfect scores
            Weight = 1.0,
            CounterpartyTrustSnapshot = counterpartyTrust,
            CompletedAt = DateTime.UtcNow.AddDays(-i)
        }).ToList();
    }
    
    private static List<EnhancedContract> CreateHonestContracts(
        int count, 
        double avgCounterpartyTrust)
    {
        var random = new Random(42);
        return Enumerable.Range(0, count).Select(i => new EnhancedContract
        {
            SkillType = SkillTypes.Engineering,
            Outcome = 0.7 + random.NextDouble() * 0.3, // Realistic outcomes
            Weight = 1.0,
            CounterpartyTrustSnapshot = avgCounterpartyTrust + random.NextDouble() * 20 - 10,
            CompletedAt = DateTime.UtcNow.AddDays(-i * 7) // Spread over weeks
        }).ToList();
    }
    
    private static List<EnhancedContract> CreateBurstContracts(
        int count, 
        TimeSpan period)
    {
        return Enumerable.Range(0, count).Select(i => new EnhancedContract
        {
            SkillType = SkillTypes.Engineering,
            Outcome = 1.0,
            Weight = 1.0,
            CounterpartyTrustSnapshot = 50,
            CompletedAt = DateTime.UtcNow - period + TimeSpan.FromHours(i)
        }).ToList();
    }
    
    private static List<EnhancedContract> CreateSpreadContracts(
        int count, 
        TimeSpan period)
    {
        var interval = period / count;
        return Enumerable.Range(0, count).Select(i => new EnhancedContract
        {
            SkillType = SkillTypes.Engineering,
            Outcome = 1.0,
            Weight = 1.0,
            CounterpartyTrustSnapshot = 50,
            CompletedAt = DateTime.UtcNow - period + interval * i
        }).ToList();
    }
}
```

---

## Summary of New Classes

| Class | Purpose | Key Method |
|-------|---------|------------|
| `EnhancedContract` | Contract with Sybil resistance fields | `FromBase()` |
| `EscrowNote` | Escrow data model | — |
| `CounterpartyFactorCalculator` | Computes γ(c) | `ComputeCounterpartyFactor()` |
| `VelocityWeightCalculator` | Computes ν(c) | `ComputeVelocityWeight()` |
| `OutcomeVarianceAnalyzer` | Checks plausibility | `IsHistoryPlausible()` |
| `EscrowVerifier` | Validates escrow | `VerifyEscrow()` |
| `EnhancedTrustCalculator` | Trust with all factors | `CalculateTrustValue()` |
| `EnhancedEligibilityService` | Combined eligibility | `CheckEligibility()` |

---

*This document extends the base Quantum of Trust C# implementation with Sybil resistance mechanisms.*
