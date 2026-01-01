# Quantum of Trust: Mathematical Equations in C#

## Executive Summary

This document translates the mathematical equations from the Quantum of Trust framework into practical C# implementations. The Quantum of Trust (q\<T\>) is a formal system for quantifying trust as a measurable outcome of actions rather than an attribute of identityâ€”enabling decentralized networks where reputation becomes a form of tradeable, context-specific capital.

### What This Document Covers

1. **Core types** - `QuantumOfTrust`, `Agent`, and `DAO` as a recursive type hierarchy
2. **Valuation function** - Trust computation returning real numbers (positive = trusted, negative = distrusted, zero = unknown)
3. **Agent trust calculation** - Sum of weighted outcomes from contract history
4. **DAO aggregation** - Configurable Φ function (sum, average, min, etc.)
5. **Contract structure** - The full tuple with hierarchical decomposition support
6. **Outcome handling** - Continuous [-1, 1] range with discrete special cases
7. **Weighting function** - Combines stake, difficulty, counterparty trust, and recency
8. **History and trust evolution** - Incremental updates as contracts complete
9. **Eligibility checking** - Threshold-based access control for contracts
10. **Validation metrics** - Pearson correlation for convergence testing
11. **Sybil resistance analysis** - Demonstrates why identity splitting is disadvantageous
12. **Customer trust** - Bidirectional trust computation for consumers
13. **Verification weight** - Customer credibility affects rating weight
14. **Task decomposition** - Team-based implementation with sub-subcontracts

The document includes a complete working example demonstrating Jane's two separate agencies (Engineer and Designer) with independent trust quotients, illustrating the core principle that reputation can be decoupled from singular identity.

---

## Constants, Interfaces, and Utilities

Before diving into the implementations, we define shared constants, interfaces, and utility classes used throughout the framework. These provide type safety, testability, and consistent magic number management.

```csharp
using System;
using System.Collections.Concurrent;
using System.Collections.Generic;
using System.Linq;
using System.Threading;
using System.Threading.Tasks;

namespace QuantumOfTrust
{
    /// <summary>
    /// Constants used throughout the Quantum of Trust framework.
    /// </summary>
    public static class TrustConstants
    {
        // Outcome bounds
        public const double MinOutcome = -1.0;
        public const double MaxOutcome = 1.0;

        // Difficulty bounds
        public const double MinDifficulty = 0.0;
        public const double MaxDifficulty = 10.0;

        // Weight calculation
        public const double RecencyHalfLifeDays = 365.0;
        public const double MinWeight = 1e-10;
        public const double CounterpartyTrustScalingFactor = 100.0;
        public const double CounterpartyTrustMaxInfluence = 0.5;
        public const double DifficultyWeightMin = 0.5;
        public const double DifficultyWeightRange = 1.5;

        // Validation
        public const double ValidationThreshold = 0.95;

        // Threshold calculation
        public const double MinimumThresholdFactor = 0.1;

        // Verification weight bounds
        public const double IdealRatingVariance = 0.35;
        public const double MinRatingVariance = 0.10;
        public const double MaxRatingVariance = 0.80;
        public const double MinVerificationWeight = 0.5;
        public const double MaxVerificationWeight = 1.5;

        // Task decomposition
        public const int MaxTasksPerPhase = 64;

        /// <summary>
        /// Returns current UTC time for consistent timestamp handling.
        /// </summary>
        public static DateTime Now => DateTime.UtcNow;
    }

    /// <summary>
    /// Customer-specific skill types for measuring consumer behavior.
    /// Unlike provider skill types (which measure delivery capability),
    /// customer skill types measure behavior as a contract consumer.
    /// </summary>
    public static class CustomerSkillTypes
    {
        /// <summary>
        /// Measures whether specs approved by this customer lead to successful implementations.
        /// Computed from outcomes of implementation phases where customer approved spec.
        /// </summary>
        public const string SpecQuality = "customer:spec_quality";

        /// <summary>
        /// Measures completed_projects / initiated_projects ratio.
        /// High commitment = reliable partner who sees projects through.
        /// </summary>
        public const string Commitment = "customer:commitment";

        /// <summary>
        /// Measures fairness and variance of ratings given by this customer.
        /// Ideal variance ~0.35; too low = rubber-stamping, too high = erratic.
        /// </summary>
        public const string Verification = "customer:verification";

        /// <summary>
        /// Measures on-time funding and prompt release of escrow.
        /// High discipline = funds on time, releases promptly after completion.
        /// </summary>
        public const string EscrowDiscipline = "customer:escrow";

        /// <summary>
        /// Measures accuracy of deadline estimates across projects.
        /// High realism = achievable deadlines, low = chronic underestimation.
        /// </summary>
        public const string TimelineRealism = "customer:timeline";

        /// <summary>
        /// Measures adherence to agreed specifications.
        /// tasks_as_planned / total_tasks - high = stable scope, low = scope creep.
        /// </summary>
        public const string ScopeStability = "customer:scope";

        /// <summary>
        /// Returns true if the skill type is a customer skill type.
        /// </summary>
        public static bool IsCustomerSkillType(string skillType)
        {
            return skillType?.StartsWith("customer:") ?? false;
        }

        /// <summary>
        /// All customer skill types for iteration.
        /// </summary>
        public static readonly IReadOnlyList<string> All = new[]
        {
            SpecQuality, Commitment, Verification, 
            EscrowDiscipline, TimelineRealism, ScopeStability
        };
    }

    /// <summary>
    /// Contract type identifiers for hierarchical contract decomposition.
    /// Hierarchy: Project → Phases → Milestones → Tasks
    /// </summary>
    public enum ContractType
    {
        /// <summary>Standalone contract, not part of a project.</summary>
        Standalone = 0,
        /// <summary>Specification phase of a project.</summary>
        Specification = 1,
        /// <summary>Planning phase of a project.</summary>
        Planning = 2,
        /// <summary>Implementation phase of a project.</summary>
        Implementation = 3,
        /// <summary>Task within a milestone (atomic work unit).</summary>
        Task = 4,
        /// <summary>Milestone within implementation phase (payment gate).</summary>
        Milestone = 5
    }

    /// <summary>
    /// Interface for trust-bearing entities in the network.
    /// </summary>
    public interface IQuantumOfTrust
    {
        /// <summary>
        /// Computes the trust value for this entity in a specific skill context.
        /// </summary>
        double ComputeTrustValue(string skillType);
    }

    /// <summary>
    /// Interface for weight calculation, enabling dependency injection and testing.
    /// </summary>
    public interface IWeightCalculator
    {
        /// <summary>
        /// Computes the weight for a contract.
        /// </summary>
        double ComputeWeight(Contract contract, double consumerTrustValue, DateTime currentTime);
    }

    /// <summary>
    /// Interface for trust tracking, enabling dependency injection and testing.
    /// </summary>
    public interface ITrustTracker
    {
        /// <summary>
        /// Updates trust when a contract completes.
        /// </summary>
        double UpdateTrust(Agent agent, Contract completedContract, DateTime currentTime);

        /// <summary>
        /// Gets the current trust value for an agent in a skill type.
        /// </summary>
        double GetTrust(Agent agent, string skillType);

        /// <summary>
        /// Adds a contract to history and updates trust atomically.
        /// </summary>
        double AddContractAndUpdateTrust(Agent agent, Contract contract, DateTime currentTime);
    }

    /// <summary>
    /// Utility class for consistent skill type handling.
    /// </summary>
    public static class SkillTypes
    {
        /// <summary>
        /// Normalizes a skill type string for consistent comparison.
        /// </summary>
        /// <param name="skillType">The skill type to normalize.</param>
        /// <returns>Normalized lowercase skill type.</returns>
        /// <exception cref="ArgumentException">Thrown if skill type is null or whitespace.</exception>
        public static string Normalize(string skillType)
        {
            if (string.IsNullOrWhiteSpace(skillType))
                throw new ArgumentException("Skill type cannot be empty", nameof(skillType));

            return skillType.Trim().ToLowerInvariant();
        }

        /// <summary>
        /// Compares two skill types for equality (case-insensitive).
        /// </summary>
        public static bool AreEqual(string a, string b)
        {
            return string.Equals(a, b, StringComparison.OrdinalIgnoreCase);
        }

        /// <summary>
        /// Returns true if the skill type is valid (not null or whitespace).
        /// </summary>
        public static bool IsValid(string skillType)
        {
            return !string.IsNullOrWhiteSpace(skillType);
        }

        // Common skill types as constants
        public const string Engineering = "engineering";
        public const string Design = "design";
        public const string Legal = "legal";
        public const string Technical = "technical";
        public const string Accounting = "accounting";
        public const string Management = "management";
    }
}
```

---

## 1. Core Type Definition

The Quantum of Trust framework is built on a recursive type system where trust-bearing entities come in two forms: individual **Agents** (avatars representing a person's capabilities in a specific skill domain) and **DAOs** (Decentralized Autonomous Organizations that contain other trust entities). This recursive structureâ€”where a DAO can contain other DAOsâ€”enables "turtles all the way down" composability, allowing complex organizational hierarchies to emerge from simple primitives. The mathematical notation uses a grammar-like definition showing that a q\<T\> is either an Agent with a skill type and history, or a DAO containing a set of other q\<T\> entities.

**Mathematical notation:**
$$q\langle T \rangle ::= \text{Agent}(t, h_t) \mid \text{DAO}(\{q\langle T \rangle\})$$

**C# Implementation:**

```csharp
/// <summary>
/// Represents a trust-bearing entity in the Quantum of Trust network.
/// This is the base type for both Agents and DAOs.
/// </summary>
public abstract class QuantumOfTrust : IQuantumOfTrust
{
    /// <summary>
    /// Computes the trust value (q&lt;T&gt;) for this entity in a specific skill context.
    /// </summary>
    /// <param name="skillType">The skill domain to compute trust for.</param>
    /// <returns>
    /// A real number where:
    /// <list type="bullet">
    /// <item><description>0 = unknown/no track record</description></item>
    /// <item><description>positive = trusted</description></item>
    /// <item><description>negative = actively distrusted</description></item>
    /// </list>
    /// </returns>
    public abstract double ComputeTrustValue(string skillType);
}

/// <summary>
/// Agent: a single avatar with a skill type and contract history.
/// Implements IEquatable for proper dictionary key behavior.
/// </summary>
public class Agent : QuantumOfTrust, IEquatable<Agent>
{
    private readonly List<Contract> _contractHistory = new();

    /// <summary>
    /// Unique identifier for this agent, used for equality comparisons.
    /// </summary>
    public Guid Id { get; } = Guid.NewGuid();

    /// <summary>
    /// The primary skill type this agent operates in.
    /// Note: Agents can have contracts in multiple skill types.
    /// </summary>
    public string SkillType { get; init; } = string.Empty;

    /// <summary>
    /// The complete history of contracts this agent has participated in (read-only view).
    /// </summary>
    public IReadOnlyList<Contract> ContractHistory => _contractHistory;

    /// <summary>
    /// Computes trust value by summing weighted outcomes for contracts
    /// matching the specified skill type (case-insensitive).
    /// V_t(Agent(t, h_t)) = Î£ Ï‰(c) Â· outcome(c) for all c in h_t
    /// </summary>
    public override double ComputeTrustValue(string skillType)
    {
        if (!SkillTypes.IsValid(skillType))
            return 0.0;

        return _contractHistory
            .Where(c => SkillTypes.AreEqual(c.SkillType, skillType))
            .Sum(c => c.Weight * c.Outcome);
    }

    /// <summary>
    /// Adds a completed contract to this agent's history.
    /// h_t^(n+1)(a) = h_t^(n)(a) âˆª {c_n}
    /// </summary>
    /// <exception cref="ArgumentNullException">Thrown if contract is null.</exception>
    public void AddToHistory(Contract contract)
    {
        if (contract == null)
            throw new ArgumentNullException(nameof(contract));
        _contractHistory.Add(contract);
    }

    /// <summary>
    /// Gets all contracts for a specific skill type (case-insensitive).
    /// </summary>
    public IEnumerable<Contract> GetHistoryForSkill(string skillType)
    {
        if (!SkillTypes.IsValid(skillType))
            return Enumerable.Empty<Contract>();
        return _contractHistory.Where(c => SkillTypes.AreEqual(c.SkillType, skillType));
    }

    /// <summary>
    /// Clears all contract history. Use with caution.
    /// </summary>
    public void ClearHistory()
    {
        _contractHistory.Clear();
    }

    // IEquatable implementation for proper dictionary key behavior
    public bool Equals(Agent? other) => other != null && Id == other.Id;
    public override bool Equals(object? obj) => Equals(obj as Agent);
    public override int GetHashCode() => Id.GetHashCode();

    public override string ToString() =>
        $"Agent[{Id.ToString().Substring(0, 8)}] {SkillType} (Contracts: {_contractHistory.Count})";
}

/// <summary>
/// DAO: a composite entity containing other q&lt;T&gt; units.
/// Trust is computed by aggregating member trust values.
/// Implements IEquatable for proper collection behavior.
/// </summary>
public class DAO : QuantumOfTrust, IEquatable<DAO>
{
    private HashSet<QuantumOfTrust> _members = new();

    /// <summary>
    /// Unique identifier for this DAO.
    /// </summary>
    public Guid Id { get; } = Guid.NewGuid();

    /// <summary>
    /// The set of member entities (Agents or other DAOs).
    /// Cannot be set to null.
    /// </summary>
    /// <exception cref="ArgumentNullException">Thrown if value is null.</exception>
    public HashSet<QuantumOfTrust> Members
    {
        get => _members;
        set => _members = value ?? throw new ArgumentNullException(nameof(value), "Members cannot be null");
    }

    /// <summary>
    /// The aggregation function Î¦ used to combine member trust values.
    /// Must be set before calling ComputeTrustValue.
    /// </summary>
    public Func<IEnumerable<double>, double>? Phi { get; set; }

    /// <summary>
    /// Computes trust by aggregating member trust values using Î¦.
    /// V_t(DAO(S)) = Î¦({V_t(q) : q âˆˆ S})
    /// Includes cycle detection to prevent stack overflow from circular references.
    /// </summary>
    /// <exception cref="InvalidOperationException">Thrown if Phi is null or circular reference detected.</exception>
    public override double ComputeTrustValue(string skillType)
    {
        return ComputeTrustValueWithCycleDetection(skillType, new HashSet<DAO>());
    }

    private double ComputeTrustValueWithCycleDetection(string skillType, HashSet<DAO> visited)
    {
        if (Phi == null)
            throw new InvalidOperationException(
                "Aggregation function Î¦ must be set before computing trust.");

        if (_members == null || !_members.Any())
            return 0.0;

        // Cycle detection
        if (!visited.Add(this))
            throw new InvalidOperationException(
                "Circular reference detected in DAO membership.");

        // Materialize to prevent multiple enumerations by Phi
        var memberTrustValues = _members.Select(q =>
        {
            if (q is DAO nestedDao)
                return nestedDao.ComputeTrustValueWithCycleDetection(skillType, visited);
            return q.ComputeTrustValue(skillType);
        }).ToList();

        return Phi(memberTrustValues);
    }

    // IEquatable implementation
    public bool Equals(DAO? other) => other != null && Id == other.Id;
    public override bool Equals(object? obj) => Equals(obj as DAO);
    public override int GetHashCode() => Id.GetHashCode();

    public override string ToString() =>
        $"DAO[{Id.ToString().Substring(0, 8)}] (Members: {_members?.Count ?? 0})";
}
```

---

## 2. Valuation Function

The valuation function is the core mechanism that computes a trust score for any entity in the network. Unlike traditional reputation systems that only allow positive scores, this function maps to all real numbers (â„), enabling three distinct states: zero indicates an unknown entity with no track record, positive values indicate earned trust, and negative values indicate an entity that has actively earned distrust through failed contracts. This tri-state model is crucialâ€”a newcomer (zero trust) deserves a chance, while an agent with negative trust has demonstrated unreliability and should be treated with appropriate caution.

**Mathematical notation:**
$$V_t: q\langle T \rangle \rightarrow \mathbb{R}$$

Where:
- $V_t = 0$ â†’ unknown, no track record
- $V_t > 0$ â†’ net positive history, trusted
- $V_t < 0$ â†’ net negative history, actively distrusted

**C# Implementation:**

```csharp
/// <summary>
/// Provides the valuation function V_t for computing trust values.
/// </summary>
public static class TrustValuation
{
    /// <summary>
    /// Computes the trust value for any q&lt;T&gt; entity in a given skill context.
    /// V_t: q&lt;T&gt; â†’ â„
    /// </summary>
    /// <param name="entity">The trust-bearing entity to evaluate.</param>
    /// <param name="skillType">The skill context for evaluation.</param>
    /// <returns>
    /// A real number where:
    ///   0 = unknown/no track record
    ///   positive = trusted
    ///   negative = actively distrusted
    /// </returns>
    /// <exception cref="ArgumentNullException">Thrown if entity is null.</exception>
    public static double V(IQuantumOfTrust entity, string skillType)
    {
        if (entity == null)
            throw new ArgumentNullException(nameof(entity));
        if (!SkillTypes.IsValid(skillType))
            return 0.0;

        return entity.ComputeTrustValue(skillType);
    }
}
```

---

## 3. Agent Trust Value Calculation

An individual agent's trust value is computed by summing up the weighted outcomes of all contracts in their history for a specific skill type. This is the heart of "trust as action, not attribution"â€”your trust score is literally the accumulation of what you've done, not who you are. The skill type scoping is critical: Jane's engineering trust and design trust are computed independently from separate histories. A failure in one domain doesn't contaminate success in another. The weighting function (Ï‰) ensures that not all contracts contribute equallyâ€”high-stakes, difficult contracts with reputable counterparties carry more signal than easy, low-value work.

**Mathematical notation:**
$$V_t(\text{Agent}(t, h_t)) = \sum_{c \in h_t} \omega(c) \cdot \text{outcome}(c)$$

Where:
- $h_t$ is the set of contracts in the agent's history for skill type $t$
- $\omega(c)$ is the weighting function
- $\text{outcome}(c)$ represents contract success or failure

**C# Implementation:**

```csharp
// The Agent.ComputeTrustValue method (defined in Section 1) implements this equation:
//
// public override double ComputeTrustValue(string skillType)
// {
//     if (!SkillTypes.IsValid(skillType))
//         return 0.0;
//
//     return _contractHistory
//         .Where(c => SkillTypes.AreEqual(c.SkillType, skillType))
//         .Sum(c => c.Weight * c.Outcome);
// }

/// <summary>
/// Example demonstrating Jane's separate, independent trust quotients.
/// </summary>
public class JaneExample
{
    public void DemonstrateIndependentTrust()
    {
        var janeEngineer = new Agent { SkillType = SkillTypes.Engineering };
        var janeDesigner = new Agent { SkillType = SkillTypes.Design };

        // Add some successful engineering contracts
        for (int i = 0; i < 10; i++)
        {
            janeEngineer.AddToHistory(ContractFactory.CreateSimple(
                skillType: SkillTypes.Engineering,
                outcome: 1.0,
                weight: 8.5));
        }

        // Add some struggling design contracts
        for (int i = 0; i < 5; i++)
        {
            janeDesigner.AddToHistory(ContractFactory.CreateSimple(
                skillType: SkillTypes.Design,
                outcome: -0.3,
                weight: 4.0));
        }

        // V_Engineer(Jane) = 85 cutes (thriving)
        double engineerTrust = janeEngineer.ComputeTrustValue(SkillTypes.Engineering);

        // V_Designer(Jane) = -6 cutes (struggling, but independent)
        double designerTrust = janeDesigner.ComputeTrustValue(SkillTypes.Design);

        Console.WriteLine($"Engineering trust: {engineerTrust}");
        Console.WriteLine($"Design trust: {designerTrust}");
    }
}
```

---

## 4. DAO Trust Value Calculation

A DAO's trust value is computed by aggregating the trust values of all its member entities using a governance-chosen aggregation function (Î¦). This function could be a simple sum (total capability), an average (mean reliability), a minimum (weakest-link analysis), or any custom function appropriate to the organization's purpose. The flexibility here is intentionalâ€”a security-focused DAO might use minimum (only as strong as weakest member), while a capacity-focused DAO might use sum (total available capability). Since DAOs can contain other DAOs, this creates a recursive trust computation that flows up through organizational hierarchies.

**Mathematical notation:**
$$V_t(\text{DAO}(S)) = \Phi\left(\{V_t(q) : q \in S\}\right)$$

Where $\Phi$ is an aggregation function chosen by the DAO's governance.

**C# Implementation:**

```csharp
// The DAO.ComputeTrustValue method (defined in Section 1) implements this equation.

/// <summary>
/// Common aggregation functions for DAO trust computation.
/// All functions handle null and empty sequences gracefully.
/// </summary>
public static class AggregationFunctions
{
    /// <summary>
    /// Sum of all member trust values (total capability).
    /// Returns 0 for null or empty sequences.
    /// </summary>
    public static double Sum(IEnumerable<double> values)
    {
        if (values == null)
            return 0.0;
        return values.Sum();
    }

    /// <summary>
    /// Average of all member trust values (mean reliability).
    /// Returns 0 for null or empty sequences.
    /// </summary>
    public static double Average(IEnumerable<double> values)
    {
        if (values == null)
            return 0.0;
        var list = values.ToList();
        return list.Any() ? list.Average() : 0.0;
    }

    /// <summary>
    /// Minimum of all member trust values (weakest-link analysis).
    /// Returns 0 for null or empty sequences.
    /// </summary>
    public static double Minimum(IEnumerable<double> values)
    {
        if (values == null)
            return 0.0;
        var list = values.ToList();
        return list.Any() ? list.Min() : 0.0;
    }

    /// <summary>
    /// Maximum of all member trust values (strongest member).
    /// Returns 0 for null or empty sequences.
    /// </summary>
    public static double Maximum(IEnumerable<double> values)
    {
        if (values == null)
            return 0.0;
        var list = values.ToList();
        return list.Any() ? list.Max() : 0.0;
    }

    /// <summary>
    /// Median of all member trust values (robust to outliers).
    /// Returns 0 for null or empty sequences.
    /// </summary>
    public static double Median(IEnumerable<double> values)
    {
        if (values == null)
            return 0.0;
        var sorted = values.OrderBy(v => v).ToList();
        if (!sorted.Any())
            return 0.0;

        int mid = sorted.Count / 2;
        if (sorted.Count % 2 == 0)
            return (sorted[mid - 1] + sorted[mid]) / 2.0;
        return sorted[mid];
    }

    /// <summary>
    /// Weighted average of member trust values.
    /// </summary>
    /// <param name="values">The trust values to aggregate.</param>
    /// <param name="weights">The weights for each value.</param>
    /// <returns>Weighted average, or 0 if inputs are empty or weights sum to zero.</returns>
    public static double WeightedAverage(IEnumerable<double> values, IEnumerable<double> weights)
    {
        if (values == null || weights == null)
            return 0.0;

        var pairs = values.Zip(weights, (v, w) => (Value: v, Weight: w)).ToList();

        if (!pairs.Any())
            return 0.0;

        double totalWeight = pairs.Sum(p => p.Weight);

        if (totalWeight == 0.0)
            return 0.0;

        return pairs.Sum(p => p.Value * p.Weight) / totalWeight;
    }

    /// <summary>
    /// Creates a weighted average Phi function with fixed weights.
    /// Use this to create a Phi-compatible function from WeightedAverage.
    /// </summary>
    /// <param name="weights">The weights to apply to each member in order.</param>
    /// <returns>A function compatible with DAO.Phi.</returns>
    /// <exception cref="ArgumentNullException">Thrown if weights is null.</exception>
    public static Func<IEnumerable<double>, double> CreateWeightedAveragePhi(IReadOnlyList<double> weights)
    {
        if (weights == null)
            throw new ArgumentNullException(nameof(weights));

        return values =>
        {
            var valueList = values?.ToList() ?? new List<double>();
            if (valueList.Count != weights.Count)
                throw new ArgumentException(
                    $"Expected {weights.Count} values but got {valueList.Count}");
            return WeightedAverage(valueList, weights);
        };
    }
}

/// <summary>
/// Example demonstrating DAO creation with nested structure.
/// </summary>
public class DAOExample
{
    public void CreateDAO()
    {
        var legalAgent = new Agent { SkillType = SkillTypes.Legal };
        var technicalAgent = new Agent { SkillType = SkillTypes.Technical };

        var nestedDao = new DAO
        {
            Members = new HashSet<QuantumOfTrust> { technicalAgent },
            Phi = AggregationFunctions.Sum
        };

        var dao = new DAO
        {
            Members = new HashSet<QuantumOfTrust>
            {
                legalAgent,
                nestedDao  // DAOs can contain other DAOs
            },
            Phi = AggregationFunctions.Average
        };

        double daoTrust = dao.ComputeTrustValue(SkillTypes.Technical);
        Console.WriteLine($"DAO trust: {daoTrust}");
    }

    public void CreateDAOWithWeightedAverage()
    {
        var agent1 = new Agent { SkillType = SkillTypes.Engineering };
        var agent2 = new Agent { SkillType = SkillTypes.Engineering };
        var agent3 = new Agent { SkillType = SkillTypes.Engineering };

        var dao = new DAO
        {
            Members = new HashSet<QuantumOfTrust> { agent1, agent2, agent3 },
            Phi = AggregationFunctions.CreateWeightedAveragePhi(new[] { 0.5, 0.3, 0.2 })
        };

        double daoTrust = dao.ComputeTrustValue(SkillTypes.Engineering);
        Console.WriteLine($"Weighted DAO trust: {daoTrust}");
    }
}
```

---

## 5. Contract Definition

Contracts are the atomic unit of interaction in the trust networkâ€”they represent formal agreements between a provider (offering services) and a consumer (requesting services). Each contract is defined as a tuple containing all the information needed to execute and evaluate the agreement: who's involved, what skill domain it covers, how much value is at stake, how difficult the work is, and when it must be completed. Smart contracts on the blockchain enforce these terms automatically, ensuring that both parties are held accountable and that outcomes are recorded immutably. The contract structure directly feeds into the weighting function to determine how much signal each completed contract provides.

**Mathematical notation:**
$$c = (a_{\text{provider}}, a_{\text{consumer}}, t, s, d, \tau)$$

Where:
- $a_{\text{provider}}$ is the agent offering services
- $a_{\text{consumer}}$ is the agent requesting services
- $t$ is the skill type
- $s$ is the stake (value at risk)
- $d$ is the difficulty rating
- $\tau$ is the deadline or timestamp

**C# Implementation:**

```csharp
/// <summary>
/// Represents a contract between a provider and consumer in the trust network.
/// Contracts are immutable once created to ensure blockchain integrity.
/// </summary>
public class Contract
{
    private double _outcome;
    private double _stake;
    private double _difficulty;
    private string _skillType = string.Empty;
    private DateTime? _completedAt;

    /// <summary>
    /// Unique identifier for this contract.
    /// </summary>
    public Guid Id { get; init; } = Guid.NewGuid();

    /// <summary>a_provider - the agent offering services</summary>
    public Agent? Provider { get; init; }

    /// <summary>a_consumer - the agent requesting services</summary>
    public Agent? Consumer { get; init; }

    /// <summary>t - the skill type. Cannot be empty or whitespace.</summary>
    /// <exception cref="ArgumentException">Thrown if value is empty or whitespace.</exception>
    public string SkillType
    {
        get => _skillType;
        init
        {
            if (string.IsNullOrWhiteSpace(value))
                throw new ArgumentException("Skill type cannot be empty", nameof(value));
            _skillType = SkillTypes.Normalize(value);
        }
    }

    /// <summary>s - stake (value at risk). Must be non-negative.</summary>
    /// <exception cref="ArgumentOutOfRangeException">Thrown if value is negative.</exception>
    public double Stake
    {
        get => _stake;
        init
        {
            if (value < 0)
                throw new ArgumentOutOfRangeException(nameof(value),
                    "Stake cannot be negative");
            _stake = value;
        }
    }

    /// <summary>d - difficulty rating in range [0, 10].</summary>
    /// <exception cref="ArgumentOutOfRangeException">Thrown if value is outside valid range.</exception>
    public double Difficulty
    {
        get => _difficulty;
        init
        {
            if (value < TrustConstants.MinDifficulty || value > TrustConstants.MaxDifficulty)
                throw new ArgumentOutOfRangeException(nameof(value),
                    $"Difficulty must be in range [{TrustConstants.MinDifficulty}, {TrustConstants.MaxDifficulty}]");
            _difficulty = value;
        }
    }

    /// <summary>Ï„ - deadline or timestamp</summary>
    public DateTime Deadline { get; init; }

    /// <summary>outcome(c) âˆˆ [-1, 1]</summary>
    /// <exception cref="ArgumentOutOfRangeException">Thrown if value is outside valid range.</exception>
    public double Outcome
    {
        get => _outcome;
        init
        {
            if (value < TrustConstants.MinOutcome || value > TrustConstants.MaxOutcome)
                throw new ArgumentOutOfRangeException(nameof(value),
                    $"Outcome must be in range [{TrustConstants.MinOutcome}, {TrustConstants.MaxOutcome}]");
            _outcome = value;
        }
    }

    /// <summary>Computed weight Ï‰(c). Must be non-negative.</summary>
    public double Weight { get; init; }

    /// <summary>When the contract was completed. Cannot be in the future.</summary>
    /// <exception cref="ArgumentOutOfRangeException">Thrown if value is more than 5 minutes in the future.</exception>
    public DateTime? CompletedAt
    {
        get => _completedAt;
        init
        {
            if (value.HasValue && value.Value > DateTime.UtcNow.AddMinutes(5))
                throw new ArgumentOutOfRangeException(nameof(value),
                    "CompletedAt cannot be in the future");
            _completedAt = value;
        }
    }

    // === Hierarchical Contract Fields (Task Decomposition) ===

    /// <summary>
    /// Contract type for hierarchical decomposition.
    /// Standalone (default), Specification, Planning, Implementation, or Task.
    /// </summary>
    public ContractType Type { get; init; } = ContractType.Standalone;

    /// <summary>
    /// Reference to parent contract (project ID for phases, phase ID for tasks).
    /// Null for standalone contracts.
    /// </summary>
    public Guid? ParentRef { get; init; }

    /// <summary>
    /// Task weight for proportional stake allocation within a phase.
    /// Only meaningful when Type = Task.
    /// task_stake = phase_stake × (task_weight / Σ task_weights)
    /// </summary>
    public double TaskWeight { get; init; } = 1.0;

    /// <summary>
    /// For planning contracts: number of planned tasks.
    /// Used to measure planning accuracy.
    /// </summary>
    public int PlannedTaskCount { get; init; }

    /// <summary>
    /// For implementation contracts: tasks completed as originally planned.
    /// </summary>
    public int TasksCompletedAsPlanned { get; init; }

    /// <summary>
    /// For implementation contracts: tasks that deviated from plan.
    /// </summary>
    public int TasksDeviated { get; init; }

    /// <summary>
    /// Computes planning accuracy: tasks_as_planned / total_tasks.
    /// Returns 1.0 if no tasks (vacuously true).
    /// </summary>
    public double PlanningAccuracy
    {
        get
        {
            int total = TasksCompletedAsPlanned + TasksDeviated;
            return total == 0 ? 1.0 : (double)TasksCompletedAsPlanned / total;
        }
    }

    /// <summary>
    /// Validates the contract's state.
    /// </summary>
    /// <returns>List of validation errors, empty if valid.</returns>
    public IReadOnlyList<string> Validate()
    {
        var errors = new List<string>();

        if (string.IsNullOrWhiteSpace(_skillType))
            errors.Add("SkillType is required");

        if (_stake < 0)
            errors.Add("Stake cannot be negative");

        if (_difficulty < TrustConstants.MinDifficulty || _difficulty > TrustConstants.MaxDifficulty)
            errors.Add($"Difficulty must be between {TrustConstants.MinDifficulty} and {TrustConstants.MaxDifficulty}");

        if (_outcome < TrustConstants.MinOutcome || _outcome > TrustConstants.MaxOutcome)
            errors.Add($"Outcome must be between {TrustConstants.MinOutcome} and {TrustConstants.MaxOutcome}");

        if (_completedAt.HasValue && _completedAt.Value > DateTime.UtcNow.AddMinutes(5))
            errors.Add("CompletedAt cannot be in the future");

        if (Weight < 0)
            errors.Add("Weight cannot be negative");

        if (TaskWeight < 0)
            errors.Add("TaskWeight cannot be negative");

        if (Type == ContractType.Task && !ParentRef.HasValue)
            errors.Add("Task contracts must have a ParentRef pointing to parent milestone");

        if (Type == ContractType.Milestone && !ParentRef.HasValue)
            errors.Add("Milestone contracts must have a ParentRef pointing to parent implementation phase");

        return errors;
    }

    /// <summary>
    /// Returns true if the contract is valid.
    /// </summary>
    public bool IsValid => !Validate().Any();

    /// <summary>
    /// Factory method for creating validated contracts with all required fields.
    /// </summary>
    /// <exception cref="ArgumentNullException">Thrown if provider or consumer is null.</exception>
    /// <exception cref="ArgumentException">Thrown if skillType is empty.</exception>
    public static Contract Create(
        Agent provider,
        Agent consumer,
        string skillType,
        double stake,
        double difficulty,
        DateTime deadline,
        double outcome = 0.0,
        double weight = 0.0,
        DateTime? completedAt = null)
    {
        if (provider == null) throw new ArgumentNullException(nameof(provider));
        if (consumer == null) throw new ArgumentNullException(nameof(consumer));
        if (string.IsNullOrWhiteSpace(skillType))
            throw new ArgumentException("Skill type is required", nameof(skillType));

        return new Contract
        {
            Provider = provider,
            Consumer = consumer,
            SkillType = skillType,
            Stake = stake,
            Difficulty = difficulty,
            Deadline = deadline,
            Outcome = outcome,
            Weight = weight,
            CompletedAt = completedAt
        };
    }

    public override string ToString() =>
        $"Contract[{Id.ToString().Substring(0, 8)}] {SkillType} Stake:{Stake:C} Diff:{Difficulty} Out:{Outcome:F2}";
}

/// <summary>
/// Factory for creating contracts in various scenarios.
/// </summary>
public static class ContractFactory
{
    /// <summary>
    /// Creates a minimal contract for testing/examples where provider/consumer aren't needed.
    /// </summary>
    public static Contract CreateSimple(
        string skillType,
        double outcome,
        double weight,
        double stake = 0,
        double difficulty = 5)
    {
        return new Contract
        {
            SkillType = skillType,
            Outcome = outcome,
            Weight = weight,
            Stake = stake,
            Difficulty = difficulty,
            Deadline = TrustConstants.Now.AddDays(30)
        };
    }
}

/// <summary>
/// Builder pattern for creating contracts with computed weights.
/// </summary>
public class ContractBuilder
{
    private Agent? _provider;
    private Agent? _consumer;
    private string _skillType = string.Empty;
    private double _stake;
    private double _difficulty = 5;
    private DateTime _deadline = TrustConstants.Now.AddDays(30);
    private double _outcome;
    private DateTime? _completedAt;
    private IWeightCalculator? _weightCalculator;

    public ContractBuilder WithProvider(Agent provider)
    {
        _provider = provider;
        return this;
    }

    public ContractBuilder WithConsumer(Agent consumer)
    {
        _consumer = consumer;
        return this;
    }

    public ContractBuilder WithSkillType(string skillType)
    {
        _skillType = skillType;
        return this;
    }

    public ContractBuilder WithStake(double stake)
    {
        _stake = stake;
        return this;
    }

    public ContractBuilder WithDifficulty(double difficulty)
    {
        _difficulty = difficulty;
        return this;
    }

    public ContractBuilder WithDeadline(DateTime deadline)
    {
        _deadline = deadline;
        return this;
    }

    public ContractBuilder WithOutcome(double outcome)
    {
        _outcome = outcome;
        return this;
    }

    public ContractBuilder CompletedAt(DateTime completedAt)
    {
        _completedAt = completedAt;
        return this;
    }

    public ContractBuilder WithWeightCalculator(IWeightCalculator calculator)
    {
        _weightCalculator = calculator;
        return this;
    }

    /// <summary>
    /// Builds the contract, optionally computing weight if a calculator was provided.
    /// </summary>
    /// <param name="currentTime">Current time for weight calculation. Defaults to UTC now.</param>
    /// <exception cref="InvalidOperationException">Thrown if required fields are missing.</exception>
    public Contract Build(DateTime? currentTime = null)
    {
        if (_provider == null) throw new InvalidOperationException("Provider is required");
        if (_consumer == null) throw new InvalidOperationException("Consumer is required");
        if (string.IsNullOrWhiteSpace(_skillType)) throw new InvalidOperationException("SkillType is required");

        double weight = 0;
        if (_weightCalculator != null)
        {
            var time = currentTime ?? TrustConstants.Now;
            double consumerTrust = _consumer.ComputeTrustValue(_skillType);

            // Create temporary contract to calculate weight
            var tempContract = new Contract
            {
                SkillType = _skillType,
                Stake = _stake,
                Difficulty = _difficulty,
                CompletedAt = _completedAt,
                Outcome = 0,
                Deadline = _deadline
            };
            weight = _weightCalculator.ComputeWeight(tempContract, consumerTrust, time);
        }

        return Contract.Create(
            _provider, _consumer, _skillType, _stake, _difficulty,
            _deadline, _outcome, weight, _completedAt);
    }
}

/// <summary>
/// Represents a task within a milestone.
/// Tasks are atomic work units that enable granular tracking and
/// team-based implementation where different providers handle different tasks.
/// 
/// Hierarchy: Project → Phase → Milestone → Task
/// Trust flows through tasks. Milestones are coordination containers (payment gates).
/// </summary>
public class Task
{
    /// <summary>Unique identifier for this task.</summary>
    public Guid Id { get; init; } = Guid.NewGuid();

    /// <summary>
    /// The provider assigned to this task.
    /// Enables team-based implementation where different avatars
    /// handle different tasks within the same milestone.
    /// </summary>
    public Agent? Provider { get; init; }

    /// <summary>Reference to the parent milestone (payment gate).</summary>
    public Guid ParentMilestoneId { get; init; }

    /// <summary>Task weight for proportional stake allocation.</summary>
    public double Weight { get; init; } = 1.0;

    /// <summary>
    /// Difficulty rating for this task (0-10).
    /// Assessed by the provider at contract acceptance.
    /// Milestone difficulty aggregates from task difficulties via stake-weighted average.
    /// </summary>
    public double Difficulty { get; init; } = 5.0;

    /// <summary>Outcome of this specific task [-1, 1].</summary>
    public double Outcome { get; init; }

    /// <summary>Whether this task was completed as planned.</summary>
    public bool CompletedAsPlanned { get; init; } = true;

    /// <summary>Timestamp when task was completed.</summary>
    public DateTime? CompletedAt { get; init; }

    /// <summary>
    /// Work duration deadline (planning data, not payment trigger).
    /// Customer review happens at parent milestone deadline.
    /// </summary>
    public DateTime? Deadline { get; init; }

    /// <summary>
    /// Computes this task's stake allocation within the milestone.
    /// task_stake = milestone_stake × (task_weight / total_task_weight)
    /// </summary>
    public double ComputeStake(double milestoneStake, double totalTaskWeight)
    {
        if (totalTaskWeight <= 0) return 0;
        return milestoneStake * (Weight / totalTaskWeight);
    }
}

/// <summary>
/// Represents a milestone with its associated tasks for team-based implementation.
/// Milestones are payment gates within the Implementation phase.
/// 
/// Hierarchy: Project → Phase → Milestone → Task
/// Trust flows through tasks. Milestones coordinate payment.
/// 
/// Customer reviews at milestone deadline:
/// - Accept all tasks → payment released
/// - Dispute specific tasks → disputed tasks enter arbitration, non-disputed paid
/// - Timeout → all tasks paid (customer forfeited review)
/// 
/// See: ADR_Milestone_Payment_Gates.md, ADR_Dispute_Resolution.md
/// </summary>
public class MilestoneWithTasks
{
    /// <summary>Unique identifier for this milestone.</summary>
    public Guid Id { get; init; } = Guid.NewGuid();

    /// <summary>Reference to the parent implementation phase.</summary>
    public Guid ParentPhaseId { get; init; }

    /// <summary>Tasks within this milestone.</summary>
    public IReadOnlyList<Task> Tasks { get; init; } = Array.Empty<Task>();

    /// <summary>Total weight of all tasks in this milestone.</summary>
    public double TotalTaskWeight => Tasks.Sum(t => t.Weight);

    /// <summary>
    /// Computes milestone stake from sum of task stakes.
    /// Milestone stake is not stored—it's computed from child tasks.
    /// </summary>
    public double ComputeStake(double phaseStake, double totalPhaseTaskWeight)
    {
        if (totalPhaseTaskWeight <= 0) return 0;
        return Tasks.Sum(t => t.ComputeStake(phaseStake, totalPhaseTaskWeight));
    }

    /// <summary>
    /// Computes milestone deadline from max of task deadlines.
    /// Milestone deadline is not stored—it's computed from child tasks.
    /// </summary>
    public DateTime? ComputeDeadline()
    {
        var deadlines = Tasks.Where(t => t.Deadline.HasValue).Select(t => t.Deadline!.Value);
        return deadlines.Any() ? deadlines.Max() : null;
    }

    /// <summary>
    /// Aggregates task contributions for a specific provider within this milestone.
    /// Enables team members to prove their individual contribution.
    /// </summary>
    public double AggregateProviderContribution(Agent provider, double milestoneWeight)
    {
        if (provider == null || TotalTaskWeight <= 0) return 0;

        double milestoneStake = Tasks.Sum(t => t.Weight); // Use weight as proxy for stake proportion

        return Tasks
            .Where(t => t.Provider?.Id == provider.Id)
            .Sum(t =>
            {
                double stakeRatio = milestoneStake > 0 
                    ? t.Weight / milestoneStake 
                    : 0;
                return milestoneWeight * stakeRatio * t.Outcome;
            });
    }

    /// <summary>
    /// Computes the aggregate milestone difficulty from task difficulties.
    /// Milestone difficulty = stake-weighted average of task difficulties.
    /// </summary>
    public double ComputeAggregateDifficulty()
    {
        if (TotalTaskWeight <= 0) return 0;

        double weightedSum = Tasks.Sum(t => t.Difficulty * t.Weight);
        return weightedSum / TotalTaskWeight;
    }
}

/// <summary>
/// Represents a phase with its associated milestones and tasks for team-based implementation.
/// 
/// Hierarchy: Project → Phase → Milestone → Task
/// 
/// For Implementation phases, tasks are grouped into milestones (payment gates).
/// For backward compatibility, this class can also work with tasks directly
/// when milestones are not used (legacy mode).
/// </summary>
public class PhaseWithTasks
{
    /// <summary>The phase contract.</summary>
    public Contract PhaseContract { get; init; } = null!;

    /// <summary>
    /// Milestones within this phase (for Implementation phases).
    /// Each milestone contains tasks and serves as a payment gate.
    /// </summary>
    public IReadOnlyList<MilestoneWithTasks> Milestones { get; init; } = Array.Empty<MilestoneWithTasks>();

    /// <summary>
    /// Tasks within this phase (legacy mode, or for non-Implementation phases).
    /// For Implementation phases, use Milestones instead.
    /// </summary>
    public IReadOnlyList<Task> Tasks { get; init; } = Array.Empty<Task>();

    /// <summary>Total weight of all tasks (across all milestones or direct tasks).</summary>
    public double TotalTaskWeight => 
        Milestones.Any() 
            ? Milestones.Sum(m => m.TotalTaskWeight)
            : Tasks.Sum(t => t.Weight);

    /// <summary>
    /// Aggregates task contributions for a specific provider.
    /// Enables team members to prove their individual contribution.
    /// </summary>
    public double AggregateProviderContribution(Agent provider)
    {
        if (provider == null || TotalTaskWeight <= 0) return 0;

        // If using milestones, aggregate across all milestones
        if (Milestones.Any())
        {
            return Milestones.Sum(m => m.AggregateProviderContribution(provider, PhaseContract.Weight));
        }

        // Legacy mode: direct tasks
        return Tasks
            .Where(t => t.Provider?.Id == provider.Id)
            .Sum(t =>
            {
                double taskStake = t.ComputeStake(PhaseContract.Stake, TotalTaskWeight);
                double stakeRatio = PhaseContract.Stake > 0 
                    ? taskStake / PhaseContract.Stake 
                    : 0;
                return PhaseContract.Weight * stakeRatio * t.Outcome;
            });
    }

    /// <summary>
    /// Computes the aggregate phase outcome from task outcomes.
    /// Phase outcome = weighted average of task outcomes.
    /// </summary>
    public double ComputeAggregateOutcome()
    {
        if (TotalTaskWeight <= 0) return 0;

        var allTasks = Milestones.Any() 
            ? Milestones.SelectMany(m => m.Tasks)
            : Tasks;

        double weightedSum = allTasks.Sum(t => t.Weight * t.Outcome);
        return weightedSum / TotalTaskWeight;
    }

    /// <summary>
    /// Computes the aggregate phase difficulty from task difficulties.
    /// Phase difficulty = stake-weighted average of task difficulties.
    /// d_phase = Σ(d_task × s_task) / Σ(s_task)
    /// </summary>
    /// <remarks>
    /// Task difficulty is assessed by the provider at contract acceptance.
    /// Tasks come from the Planning phase (customer's responsibility).
    /// The provider can request task refinement before accepting if tasks
    /// are too vague to assess difficulty confidently.
    /// See: The_Difficulty_of_Assessing_Difficulty.md
    /// </remarks>
    public double ComputeAggregateDifficulty()
    {
        if (TotalTaskWeight <= 0) return 0;

        var allTasks = Milestones.Any() 
            ? Milestones.SelectMany(m => m.Tasks)
            : Tasks;

        double totalStake = allTasks.Sum(t => t.ComputeStake(PhaseContract.Stake, TotalTaskWeight));
        if (totalStake <= 0) return 0;

        double weightedSum = allTasks.Sum(t => 
            t.Difficulty * t.ComputeStake(PhaseContract.Stake, TotalTaskWeight));
        return weightedSum / totalStake;
    }
}

/// <summary>
/// Customer behavior profile for computing customer trust.
/// Customers earn trust from their behavior in contracts,
/// not from outcomes assigned by others.
/// </summary>
public class CustomerProfile
{
    /// <summary>The customer's agent.</summary>
    public Agent Customer { get; init; } = null!;

    /// <summary>Total number of projects initiated by this customer.</summary>
    public int ProjectsInitiated { get; set; }

    /// <summary>Number of projects completed (all phases reached terminal state).</summary>
    public int ProjectsCompleted { get; set; }

    /// <summary>Total number of ratings (outcomes) given by this customer.</summary>
    public int TotalRatingsGiven { get; set; }

    /// <summary>
    /// Variance of ratings given.
    /// Low variance indicates rubber-stamping; high variance indicates erratic rating.
    /// </summary>
    public double RatingsVariance { get; set; }

    /// <summary>Number of disputed outcomes.</summary>
    public int DisputesCount { get; set; }

    /// <summary>Number of on-time escrow fundings.</summary>
    public int OnTimeFundings { get; set; }

    /// <summary>Total number of required fundings.</summary>
    public int TotalFundings { get; set; }

    /// <summary>
    /// Average delay in releasing escrow after completion (days).
    /// Negative values indicate early release (good).
    /// </summary>
    public double AvgReleaseDelayDays { get; set; }

    /// <summary>
    /// Sum of (planned_duration - actual_duration) across projects.
    /// Negative indicates overruns (bad), positive indicates early delivery.
    /// </summary>
    public double TimelineAccuracySum { get; set; }

    /// <summary>Total projects for timeline calculation.</summary>
    public int TimelineProjectsCount { get; set; }

    /// <summary>Sum of tasks completed as planned across all projects.</summary>
    public int ScopeTasksAsPlanned { get; set; }

    /// <summary>Total tasks across all projects.</summary>
    public int ScopeTotalTasks { get; set; }

    /// <summary>
    /// Implementation outcomes for specs approved by this customer.
    /// Used for spec quality calculation.
    /// </summary>
    public IReadOnlyList<(double Outcome, double Weight)> ImplementationOutcomes { get; set; } 
        = Array.Empty<(double, double)>();

    // === Computed Properties ===

    /// <summary>
    /// Commitment rate: completed / initiated.
    /// Range [0, 1]. Higher is better.
    /// </summary>
    public double CommitmentRate => ProjectsInitiated == 0 
        ? 0.0 
        : (double)ProjectsCompleted / ProjectsInitiated;

    /// <summary>
    /// Funding reliability: on_time / total.
    /// Range [0, 1]. Higher is better.
    /// </summary>
    public double FundingReliability => TotalFundings == 0 
        ? 0.0 
        : (double)OnTimeFundings / TotalFundings;

    /// <summary>
    /// Scope stability: tasks_as_planned / total_tasks.
    /// Range [0, 1]. Higher is better.
    /// </summary>
    public double ScopeStability => ScopeTotalTasks == 0 
        ? 0.0 
        : (double)ScopeTasksAsPlanned / ScopeTotalTasks;

    /// <summary>
    /// Average timeline accuracy per project.
    /// Positive = early delivery (good), negative = overruns (bad).
    /// </summary>
    public double AvgTimelineAccuracy => TimelineProjectsCount == 0 
        ? 0.0 
        : TimelineAccuracySum / TimelineProjectsCount;
}
```

---

## 5b. Customer Trust Calculation

Customer trust is computed from observable behaviors rather than outcomes assigned by others.
This creates bidirectional accountability: providers are judged by their work quality,
customers are judged by their behavior as contract consumers.

**Mathematical notation:**
$$V_t^{\text{customer}}(\text{Consumer}) = \sum_{c \in h_c} \omega(c) \cdot \text{behavior}(c) \cdot \gamma(c) \cdot \nu(c)$$

**C# Implementation:**

```csharp
/// <summary>
/// Calculates customer trust values from behavior profiles.
/// </summary>
public static class CustomerTrustCalculator
{
    /// <summary>
    /// Computes commitment trust: completed_projects / initiated_projects.
    /// High commitment = reliable partner who sees projects through.
    /// </summary>
    public static double ComputeCommitmentTrust(CustomerProfile profile)
    {
        if (profile == null) return 0.0;
        return profile.CommitmentRate;
    }

    /// <summary>
    /// Computes escrow discipline trust from funding reliability and release promptness.
    /// </summary>
    public static double ComputeEscrowTrust(CustomerProfile profile)
    {
        if (profile == null) return 0.0;

        // Primary factor: on-time funding rate (70%)
        double fundingFactor = profile.FundingReliability;

        // Secondary factor: release delay (30%)
        // Map delay to [0.5, 1.0]: 0 delay = 1.0, 7+ days = 0.5
        double delayFactor;
        if (profile.AvgReleaseDelayDays <= 0)
        {
            delayFactor = 1.0; // Early or on-time release
        }
        else if (profile.AvgReleaseDelayDays >= 7)
        {
            delayFactor = 0.5; // 7+ days late = minimum
        }
        else
        {
            delayFactor = 1.0 - (profile.AvgReleaseDelayDays / 14.0);
        }

        return (fundingFactor * 0.7) + (delayFactor * 0.3);
    }

    /// <summary>
    /// Computes verification integrity trust based on rating variance.
    /// Ideal variance ~0.35. Too low = rubber-stamping, too high = erratic.
    /// </summary>
    public static double ComputeVerificationIntegrity(CustomerProfile profile)
    {
        if (profile == null || profile.TotalRatingsGiven == 0) return 0.0;

        double variance = profile.RatingsVariance;

        // Distance from ideal variance
        double distance = Math.Abs(variance - TrustConstants.IdealRatingVariance);
        double maxDistance = 0.65; // Max possible distance from ideal

        double normalized = Math.Min(distance / maxDistance, 1.0);
        double baseIntegrity = 1.0 - normalized;

        // Additional penalties for extreme variance
        if (variance < TrustConstants.MinRatingVariance)
        {
            // Severe penalty for rubber-stamping
            return baseIntegrity * 0.5;
        }
        else if (variance > TrustConstants.MaxRatingVariance)
        {
            // Penalty for erratic rating
            return baseIntegrity * 0.75;
        }

        return baseIntegrity;
    }

    /// <summary>
    /// Computes scope stability trust: tasks_as_planned / total_tasks.
    /// </summary>
    public static double ComputeScopeStability(CustomerProfile profile)
    {
        if (profile == null) return 0.0;
        return profile.ScopeStability;
    }

    /// <summary>
    /// Computes timeline realism trust from deadline accuracy.
    /// </summary>
    public static double ComputeTimelineRealism(CustomerProfile profile)
    {
        if (profile == null || profile.TimelineProjectsCount == 0) return 0.0;

        double avgAccuracy = profile.AvgTimelineAccuracy;

        if (avgAccuracy < 0)
        {
            // Chronic overruns (bad) - map to [0, 0.5]
            double overrunPct = Math.Min(Math.Abs(avgAccuracy), 1.0);
            return (1.0 - overrunPct) * 0.5;
        }
        else
        {
            // Early/on-time (good) - map to [0.5, 1.0]
            double bufferPct = Math.Min(avgAccuracy, 1.0);
            return 0.5 + (bufferPct * 0.5);
        }
    }

    /// <summary>
    /// Computes spec quality trust from implementation outcomes.
    /// Measures whether specs approved by this customer lead to successful implementations.
    /// </summary>
    public static double ComputeSpecQuality(CustomerProfile profile)
    {
        if (profile?.ImplementationOutcomes == null || !profile.ImplementationOutcomes.Any())
            return 0.0;

        double totalWeight = profile.ImplementationOutcomes.Sum(x => x.Weight);
        if (totalWeight <= 0) return 0.0;

        double weightedSum = profile.ImplementationOutcomes.Sum(x => x.Weight * x.Outcome);
        return weightedSum / totalWeight;
    }

    /// <summary>
    /// Computes customer trust for a specific customer skill type.
    /// </summary>
    /// <param name="profile">The customer's behavior profile.</param>
    /// <param name="skillType">One of CustomerSkillTypes constants.</param>
    /// <returns>Trust value in range appropriate to the skill type.</returns>
    public static double ComputeCustomerTrust(CustomerProfile profile, string skillType)
    {
        if (profile == null || string.IsNullOrWhiteSpace(skillType))
            return 0.0;

        return skillType switch
        {
            CustomerSkillTypes.Commitment => ComputeCommitmentTrust(profile),
            CustomerSkillTypes.EscrowDiscipline => ComputeEscrowTrust(profile),
            CustomerSkillTypes.Verification => ComputeVerificationIntegrity(profile),
            CustomerSkillTypes.ScopeStability => ComputeScopeStability(profile),
            CustomerSkillTypes.TimelineRealism => ComputeTimelineRealism(profile),
            CustomerSkillTypes.SpecQuality => ComputeSpecQuality(profile),
            _ => 0.0
        };
    }

    /// <summary>
    /// Computes a composite customer trust score across all customer skill types.
    /// </summary>
    /// <param name="profile">The customer's behavior profile.</param>
    /// <param name="weights">Optional weights for each skill type. Defaults to equal weights.</param>
    /// <returns>Weighted average of all customer trust dimensions.</returns>
    public static double ComputeCompositeTrust(
        CustomerProfile profile,
        Dictionary<string, double>? weights = null)
    {
        if (profile == null) return 0.0;

        var defaultWeights = new Dictionary<string, double>
        {
            [CustomerSkillTypes.Commitment] = 1.0,
            [CustomerSkillTypes.EscrowDiscipline] = 1.0,
            [CustomerSkillTypes.Verification] = 1.0,
            [CustomerSkillTypes.ScopeStability] = 1.0,
            [CustomerSkillTypes.TimelineRealism] = 1.0,
            [CustomerSkillTypes.SpecQuality] = 1.0
        };

        weights ??= defaultWeights;

        double totalWeight = weights.Values.Sum();
        if (totalWeight <= 0) return 0.0;

        double weightedSum = weights.Sum(kv =>
            kv.Value * ComputeCustomerTrust(profile, kv.Key));

        return weightedSum / totalWeight;
    }
}
```

---

## 5c. Verification Weight

The verification weight adjusts how much a customer's rating contributes to a provider's trust.
Credible, discriminating customers' ratings count more than those from unknown or rubber-stamping customers.

**Mathematical notation:**
$$\text{verification\_weight}(c) = f\big(V_{\text{verify}}(\text{consumer}), \sigma^2(\text{ratings})\big)$$

**C# Implementation:**

```csharp
/// <summary>
/// Calculates verification weights that adjust rating credibility.
/// </summary>
public static class VerificationWeightCalculator
{
    /// <summary>
    /// Computes the verification weight for a customer's rating.
    /// Range approximately [0.5, 1.5].
    /// </summary>
    /// <param name="customerVerificationTrust">Customer's verification integrity score [0, 1].</param>
    /// <param name="customerRatingVariance">Variance of customer's historical ratings.</param>
    /// <returns>Weight multiplier for the customer's ratings.</returns>
    public static double ComputeVerificationWeight(
        double customerVerificationTrust,
        double customerRatingVariance)
    {
        // Base weight from verification trust: maps [0, 1] to [0.5, 1.0]
        double trustFactor = 0.5 + (customerVerificationTrust * 0.5);

        // Variance factor: penalize extreme variance
        double varianceFactor;
        if (customerRatingVariance < TrustConstants.MinRatingVariance)
        {
            // Very low variance = rubber-stamping = 0.5x
            varianceFactor = 0.5;
        }
        else if (customerRatingVariance > TrustConstants.MaxRatingVariance)
        {
            // Very high variance = erratic = 0.7x
            varianceFactor = 0.7;
        }
        else
        {
            // Reasonable variance = full weight
            varianceFactor = 1.0;
        }

        return Math.Clamp(
            trustFactor * varianceFactor,
            TrustConstants.MinVerificationWeight,
            TrustConstants.MaxVerificationWeight);
    }

    /// <summary>
    /// Computes the adjusted trust contribution for a contract,
    /// applying verification weight from customer credibility.
    /// </summary>
    public static double ComputeAdjustedContribution(
        Contract contract,
        CustomerProfile customerProfile)
    {
        if (contract == null) return 0.0;

        double baseContribution = contract.Weight * contract.Outcome;

        if (customerProfile == null) return baseContribution;

        double verificationTrust = CustomerTrustCalculator
            .ComputeVerificationIntegrity(customerProfile);
        double verificationWeight = ComputeVerificationWeight(
            verificationTrust,
            customerProfile.RatingsVariance);

        return baseContribution * verificationWeight;
    }
}
```

---

## 5d. Task Decomposition

Tasks enable granular tracking within phases and team-based implementation.
Multiple providers can work on different tasks within a single Implementation phase.

**Mathematical notation:**
$$s_{\text{task}} = s_{\text{phase}} \cdot \frac{w_{\text{task}}}{\sum_{i} w_{\text{task}_i}}$$

**C# Implementation:**

```csharp
/// <summary>
/// Utilities for task decomposition calculations.
/// </summary>
public static class TaskDecomposition
{
    /// <summary>
    /// Computes the stake allocation for a task within a phase.
    /// </summary>
    public static double ComputeTaskStake(
        double phaseStake,
        double taskWeight,
        double totalTaskWeight)
    {
        if (totalTaskWeight <= 0) return 0;
        return phaseStake * (taskWeight / totalTaskWeight);
    }

    /// <summary>
    /// Computes planning accuracy from task completion data.
    /// </summary>
    public static double ComputePlanningAccuracy(
        int tasksAsPlanned,
        int tasksDeviated)
    {
        int total = tasksAsPlanned + tasksDeviated;
        return total == 0 ? 1.0 : (double)tasksAsPlanned / total;
    }

    /// <summary>
    /// Aggregates task contributions for all providers in a phase.
    /// Returns dictionary mapping provider ID to their total contribution.
    /// </summary>
    public static Dictionary<Guid, double> AggregateAllProviderContributions(
        PhaseWithTasks phase)
    {
        var result = new Dictionary<Guid, double>();
        if (phase?.Tasks == null || phase.TotalTaskWeight <= 0)
            return result;

        foreach (var task in phase.Tasks)
        {
            if (task.Provider == null) continue;

            double taskStake = task.ComputeStake(phase.PhaseContract.Stake, phase.TotalTaskWeight);
            double stakeRatio = phase.PhaseContract.Stake > 0
                ? taskStake / phase.PhaseContract.Stake
                : 0;
            double contribution = phase.PhaseContract.Weight * stakeRatio * task.Outcome;

            if (result.ContainsKey(task.Provider.Id))
                result[task.Provider.Id] += contribution;
            else
                result[task.Provider.Id] = contribution;
        }

        return result;
    }

    /// <summary>
    /// Creates task contracts from a phase contract.
    /// Each task becomes a sub-subcontract with proportional stake.
    /// </summary>
    public static IReadOnlyList<Contract> CreateTaskContracts(
        PhaseWithTasks phase,
        IWeightCalculator weightCalculator,
        DateTime currentTime)
    {
        if (phase?.Tasks == null || phase.TotalTaskWeight <= 0)
            return Array.Empty<Contract>();

        var contracts = new List<Contract>();

        foreach (var task in phase.Tasks)
        {
            double taskStake = task.ComputeStake(
                phase.PhaseContract.Stake,
                phase.TotalTaskWeight);

            var contract = new Contract
            {
                Id = task.Id,
                Provider = task.Provider,
                Consumer = phase.PhaseContract.Consumer,
                SkillType = phase.PhaseContract.SkillType,
                Stake = taskStake,
                Difficulty = task.Difficulty,  // Use task-level difficulty
                Deadline = phase.PhaseContract.Deadline,
                Outcome = task.Outcome,
                CompletedAt = task.CompletedAt,
                Type = ContractType.Task,
                ParentRef = phase.PhaseContract.Id,
                TaskWeight = task.Weight,
                Weight = weightCalculator?.ComputeWeight(
                    new Contract { Stake = taskStake, Difficulty = task.Difficulty },
                    0,
                    currentTime) ?? 1.0
            };

            contracts.Add(contract);
        }

        return contracts;
    }
}

Contract outcomes are represented as continuous values in the range [-1, 1], allowing for nuanced evaluation beyond simple pass/fail. A value of 1 represents complete success, -1 represents complete failure, and values in between represent partial outcomes. This continuous range acknowledges real-world complexityâ€”a project might be delivered late but with excellent quality, or on time but with minor defects. The discrete special case {-1, 0, 1} (failure, partial, success) can be used for simpler evaluation scenarios. This outcome value is multiplied by the contract's weight to determine its contribution to the agent's trust quotient.

**Mathematical notation:**
$$\text{outcome}(c) \in [-1, 1]$$

This continuous range allows for partial success or failure. Discrete outcomes $\{-1, 0, 1\}$ represent {failure, partial, success}.

**C# Implementation:**

```csharp
/// <summary>
/// Discrete outcome values for simple pass/partial/fail scenarios.
/// </summary>
public enum DiscreteOutcome
{
    Failure = -1,
    Partial = 0,
    Success = 1
}

/// <summary>
/// Utility class for calculating and validating contract outcomes.
/// </summary>
public static class OutcomeCalculator
{
    /// <summary>
    /// Validates that an outcome is within the valid [-1, 1] range.
    /// </summary>
    /// <param name="outcome">The outcome value to validate.</param>
    /// <returns>The validated outcome.</returns>
    /// <exception cref="ArgumentOutOfRangeException">Thrown if outcome is outside valid range.</exception>
    public static double ValidateOutcome(double outcome)
    {
        if (outcome < TrustConstants.MinOutcome || outcome > TrustConstants.MaxOutcome)
            throw new ArgumentOutOfRangeException(
                nameof(outcome),
                $"Outcome must be in range [{TrustConstants.MinOutcome}, {TrustConstants.MaxOutcome}]");
        return outcome;
    }

    /// <summary>
    /// Converts a discrete outcome to its double representation.
    /// </summary>
    public static double FromDiscrete(DiscreteOutcome outcome)
    {
        return (double)outcome;
    }

    /// <summary>
    /// Calculates a partial outcome based on completion percentage and quality score.
    /// Both inputs should be in range [0, 1].
    /// </summary>
    /// <param name="completionPercentage">How much of the work was completed (0-1).</param>
    /// <param name="qualityScore">Quality of the completed work (0-1).</param>
    /// <returns>An outcome value in range [-1, 1].</returns>
    /// <exception cref="ArgumentOutOfRangeException">Thrown if inputs are outside [0, 1].</exception>
    public static double CalculatePartialOutcome(
        double completionPercentage,
        double qualityScore)
    {
        if (completionPercentage < 0.0 || completionPercentage > 1.0)
            throw new ArgumentOutOfRangeException(nameof(completionPercentage),
                "Completion percentage must be in range [0, 1]");
        if (qualityScore < 0.0 || qualityScore > 1.0)
            throw new ArgumentOutOfRangeException(nameof(qualityScore),
                "Quality score must be in range [0, 1]");

        // Average of completion and quality, in [0, 1]
        double rawScore = (completionPercentage + qualityScore) / 2.0;

        // Map [0, 1] to [-1, 1]
        return (rawScore * 2.0) - 1.0;
    }

    /// <summary>
    /// Clamps an outcome to the valid range [-1, 1].
    /// </summary>
    public static double Clamp(double outcome)
    {
        return Math.Clamp(outcome, TrustConstants.MinOutcome, TrustConstants.MaxOutcome);
    }
}
```

---

## 7. Weighting Function

The weighting function determines how much signal each contract contributes to an agent's trust quotient. Not all contracts are created equalâ€”completing a difficult, high-stakes contract for a reputable counterparty should contribute more to your reputation than completing an easy, low-value contract for an unknown entity. The function combines four factors: stake (higher value = more signal), difficulty (harder work = more signal), counterparty trust (endorsement from trusted agents matters more), and recency (recent performance is more relevant than ancient history). The implementation uses logarithmic scaling for stakes, linear scaling for difficulty, sigmoid bounding for counterparty influence, and exponential decay for recency.

**Mathematical notation:**
$$\omega(c) = f\big(s(c),\ d(c),\ V_t(a_{\text{consumer}}),\ \text{recency}(c)\big)$$

The weight depends on:
- Stake: Higher-value contracts carry more signal
- Difficulty: Harder contracts carry more signal
- Counterparty trust: Contracts with high-trust counterparties carry more signal
- Recency: Recent contracts weighted more heavily than old ones

**C# Implementation:**

```csharp
/// <summary>
/// Calculates contract weights based on stake, difficulty, counterparty trust, and recency.
/// </summary>
public class WeightCalculator : IWeightCalculator
{
    /// <summary>
    /// Computes the weight Ï‰(c) for a contract.
    /// Ï‰(c) = f(s(c), d(c), V_t(a_consumer), recency(c))
    /// </summary>
    /// <param name="contract">The contract to compute weight for.</param>
    /// <param name="consumerTrustValue">The trust value of the consumer agent.</param>
    /// <param name="currentTime">The current time for recency calculation.</param>
    /// <returns>The computed weight, guaranteed to be at least MinWeight.</returns>
    /// <exception cref="ArgumentNullException">Thrown if contract is null.</exception>
    public double ComputeWeight(
        Contract contract,
        double consumerTrustValue,
        DateTime currentTime)
    {
        if (contract == null)
            throw new ArgumentNullException(nameof(contract));

        double stakeWeight = NormalizeStake(contract.Stake);
        double difficultyWeight = NormalizeDifficulty(contract.Difficulty);
        double counterpartyWeight = NormalizeCounterpartyTrust(consumerTrustValue);
        double recencyWeight = CalculateRecency(contract.CompletedAt, currentTime);

        // Combine factors multiplicatively
        double weight = stakeWeight * difficultyWeight * counterpartyWeight * recencyWeight;

        // Ensure minimum weight to prevent numerical precision issues
        return Math.Max(weight, TrustConstants.MinWeight);
    }

    /// <summary>
    /// Logarithmic scaling for stake values.
    /// </summary>
    private static double NormalizeStake(double stake)
    {
        // Stake is already validated as non-negative in Contract
        return Math.Log(1 + stake);
    }

    /// <summary>
    /// Linear scaling for difficulty, mapping [0, 10] to [0.5, 2.0].
    /// </summary>
    private static double NormalizeDifficulty(double difficulty)
    {
        // Clamp to expected range for safety
        difficulty = Math.Clamp(difficulty, TrustConstants.MinDifficulty, TrustConstants.MaxDifficulty);
        return TrustConstants.DifficultyWeightMin +
               (difficulty / TrustConstants.MaxDifficulty) * TrustConstants.DifficultyWeightRange;
    }

    /// <summary>
    /// Sigmoid-like function to bound counterparty trust influence to [0.5, 1.5].
    /// </summary>
    private static double NormalizeCounterpartyTrust(double trustValue)
    {
        return 1.0 + (Math.Tanh(trustValue / TrustConstants.CounterpartyTrustScalingFactor)
                      * TrustConstants.CounterpartyTrustMaxInfluence);
    }

    /// <summary>
    /// Exponential decay based on time since completion.
    /// </summary>
    private static double CalculateRecency(DateTime? completedAt, DateTime currentTime)
    {
        if (!completedAt.HasValue)
            return 1.0;

        double daysSinceCompletion = (currentTime - completedAt.Value).TotalDays;

        // Clamp to prevent future dates from inflating weight
        if (daysSinceCompletion < 0)
            daysSinceCompletion = 0;

        return Math.Pow(0.5, daysSinceCompletion / TrustConstants.RecencyHalfLifeDays);
    }
}
```

---

## 8. History Evolution

An agent's contract history grows over time as new contracts are completed. This equation simply states that the history at step n+1 equals the history at step n plus the newly completed contractâ€”a union operation that appends to the immutable record. On a blockchain, this history is tamper-proof and publicly verifiable (though the agent's identity behind the avatar may remain anonymous). The history is partitioned by skill type, so an agent maintains separate, independent histories for each domain they operate in. This separation is what enables the decoupling of reputation from singular identity.

**Mathematical notation:**
$$h_t^{(n+1)}(a) = h_t^{(n)}(a) \cup \{c_n\}$$

**C# Implementation:**

```csharp
// The history evolution is implemented in the Agent class (Section 1):
//
// public void AddToHistory(Contract contract)
// {
//     if (contract == null)
//         throw new ArgumentNullException(nameof(contract));
//     _contractHistory.Add(contract);
// }
//
// public IEnumerable<Contract> GetHistoryForSkill(string skillType)
// {
//     if (!SkillTypes.IsValid(skillType))
//         return Enumerable.Empty<Contract>();
//     return _contractHistory.Where(c => SkillTypes.AreEqual(c.SkillType, skillType));
// }

/// <summary>
/// Extension methods for working with agent histories.
/// </summary>
public static class HistoryExtensions
{
    /// <summary>
    /// Gets the total number of contracts in an agent's history.
    /// </summary>
    public static int TotalContractCount(this Agent agent)
    {
        return agent?.ContractHistory?.Count ?? 0;
    }

    /// <summary>
    /// Gets the number of contracts for a specific skill type.
    /// </summary>
    public static int ContractCountForSkill(this Agent agent, string skillType)
    {
        if (agent == null || !SkillTypes.IsValid(skillType))
            return 0;
        return agent.GetHistoryForSkill(skillType).Count();
    }

    /// <summary>
    /// Gets the success rate (outcomes > 0) for a specific skill type.
    /// </summary>
    public static double SuccessRateForSkill(this Agent agent, string skillType)
    {
        if (agent == null || !SkillTypes.IsValid(skillType))
            return 0.0;

        var contracts = agent.GetHistoryForSkill(skillType).ToList();
        if (!contracts.Any())
            return 0.0;

        return contracts.Count(c => c.Outcome > 0) / (double)contracts.Count;
    }

    /// <summary>
    /// Gets the average outcome for a specific skill type.
    /// </summary>
    public static double AverageOutcomeForSkill(this Agent agent, string skillType)
    {
        if (agent == null || !SkillTypes.IsValid(skillType))
            return 0.0;

        var contracts = agent.GetHistoryForSkill(skillType).ToList();
        if (!contracts.Any())
            return 0.0;

        return contracts.Average(c => c.Outcome);
    }

    /// <summary>
    /// Gets the total stake across all contracts for a specific skill type.
    /// </summary>
    public static double TotalStakeForSkill(this Agent agent, string skillType)
    {
        if (agent == null || !SkillTypes.IsValid(skillType))
            return 0.0;

        return agent.GetHistoryForSkill(skillType).Sum(c => c.Stake);
    }
}
```

---

## 9. Trust Evolution

This equation describes how an agent's trust value updates incrementally as each contract completes. Rather than recomputing the entire sum from scratch, trust evolves by adding the weighted outcome of each new contract to the existing value. This incremental update is more efficient for real-time systems and clearly shows the accumulative nature of trustâ€”each action either adds to or subtracts from your reputation. The `TrustTracker` class maintains cached trust values and updates them as contracts complete, providing an efficient implementation for networks with high transaction volumes.

**Mathematical notation:**
$$V_t^{(n+1)}(a) = V_t^{(n)}(a) + \omega(c_n) \cdot \text{outcome}(c_n)$$

**C# Implementation:**

```csharp
/// <summary>
/// Represents a single trust change event for audit purposes.
/// </summary>
public class TrustChangeEvent
{
    public Guid AgentId { get; init; }
    public string SkillType { get; init; } = string.Empty;
    public double PreviousTrust { get; init; }
    public double NewTrust { get; init; }
    public double Delta { get; init; }
    public Guid ContractId { get; init; }
    public DateTime Timestamp { get; init; }

    public override string ToString() =>
        $"[{Timestamp:u}] Agent {AgentId.ToString().Substring(0, 8)} {SkillType}: " +
        $"{PreviousTrust:F2} -> {NewTrust:F2} (Î”{Delta:+0.00;-0.00})";
}

/// <summary>
/// Thread-safe tracker for incrementally updating trust values.
/// Uses caching for efficient updates in high-volume networks.
/// </summary>
public class TrustTracker : ITrustTracker
{
    private readonly ConcurrentDictionary<(Guid agentId, string skillType), double> _cachedTrust = new();
    private readonly List<TrustChangeEvent> _auditLog = new();
    private readonly object _auditLock = new();

    /// <summary>
    /// Gets the audit log of all trust changes.
    /// </summary>
    public IReadOnlyList<TrustChangeEvent> AuditLog
    {
        get
        {
            lock (_auditLock)
            {
                return _auditLog.ToList();
            }
        }
    }

    /// <summary>
    /// Updates trust incrementally when a contract completes.
    /// V_t^(n+1)(a) = V_t^(n)(a) + Ï‰(c_n) Â· outcome(c_n)
    /// Uses the pre-computed weight stored on the contract for consistency.
    /// </summary>
    /// <param name="agent">The agent whose trust to update.</param>
    /// <param name="completedContract">The completed contract.</param>
    /// <param name="currentTime">Current time for the audit log.</param>
    /// <returns>The new trust value.</returns>
    /// <exception cref="ArgumentNullException">Thrown if agent or contract is null.</exception>
    /// <exception cref="ArgumentException">Thrown if contract has no skill type.</exception>
    public double UpdateTrust(
        Agent agent,
        Contract completedContract,
        DateTime currentTime)
    {
        if (agent == null)
            throw new ArgumentNullException(nameof(agent));
        if (completedContract == null)
            throw new ArgumentNullException(nameof(completedContract));
        if (!SkillTypes.IsValid(completedContract.SkillType))
            throw new ArgumentException("Contract must have a skill type", nameof(completedContract));

        var normalizedSkillType = SkillTypes.Normalize(completedContract.SkillType);
        var key = (agent.Id, normalizedSkillType);

        // Use the pre-computed weight stored on the contract for consistency
        // This ensures Agent.ComputeTrustValue and TrustTracker produce same results
        double delta = completedContract.Weight * completedContract.Outcome;

        double previousTrust = _cachedTrust.GetValueOrDefault(key, 0.0);

        // Atomic update using ConcurrentDictionary
        double newTrust = _cachedTrust.AddOrUpdate(
            key,
            _ => delta,  // If new, start with delta
            (_, existing) => existing + delta);  // If exists, add delta

        // Record audit event
        lock (_auditLock)
        {
            _auditLog.Add(new TrustChangeEvent
            {
                AgentId = agent.Id,
                SkillType = normalizedSkillType,
                PreviousTrust = previousTrust,
                NewTrust = newTrust,
                Delta = delta,
                ContractId = completedContract.Id,
                Timestamp = currentTime
            });
        }

        return newTrust;
    }

    /// <summary>
    /// Gets the current cached trust value for an agent in a skill type.
    /// </summary>
    public double GetTrust(Agent agent, string skillType)
    {
        if (agent == null)
            throw new ArgumentNullException(nameof(agent));
        if (!SkillTypes.IsValid(skillType))
            return 0.0;

        var normalizedSkillType = SkillTypes.Normalize(skillType);
        var key = (agent.Id, normalizedSkillType);
        return _cachedTrust.GetValueOrDefault(key, 0.0);
    }

    /// <summary>
    /// Adds contract to agent's history and updates trust atomically.
    /// This is the recommended way to record completed contracts.
    /// </summary>
    public double AddContractAndUpdateTrust(
        Agent agent,
        Contract contract,
        DateTime currentTime)
    {
        if (agent == null) throw new ArgumentNullException(nameof(agent));
        if (contract == null) throw new ArgumentNullException(nameof(contract));

        agent.AddToHistory(contract);
        return UpdateTrust(agent, contract, currentTime);
    }

    /// <summary>
    /// Clears the audit log. Use with caution.
    /// </summary>
    public void ClearAuditLog()
    {
        lock (_auditLock)
        {
            _auditLog.Clear();
        }
    }

    /// <summary>
    /// Gets the trust history for a specific agent and skill type.
    /// </summary>
    public IReadOnlyList<TrustChangeEvent> GetHistoryForAgent(Agent agent, string skillType)
    {
        if (agent == null)
            return Array.Empty<TrustChangeEvent>();
        if (!SkillTypes.IsValid(skillType))
            return Array.Empty<TrustChangeEvent>();

        var normalizedSkillType = SkillTypes.Normalize(skillType);

        lock (_auditLock)
        {
            return _auditLog
                .Where(e => e.AgentId == agent.Id &&
                           SkillTypes.AreEqual(e.SkillType, normalizedSkillType))
                .ToList();
        }
    }
}
```

---

## 10. Eligibility Function

The eligibility function creates a virtuous cycle in the trust network: higher trust unlocks access to better opportunities. An agent can only bid on a contract if their trust value meets or exceeds the contract's threshold (Î¸). High-stakes, difficult contracts require higher trust to even participate. This mechanism serves multiple purposes: it protects consumers from unproven agents on critical work, it rewards agents who build genuine trust with access to premium contracts, and it reinforces Sybil resistanceâ€”splitting your activity across multiple fake identities means each identity has lower trust, making each eligible for fewer and lower-quality contracts.

**Mathematical notation:**
$$\text{eligible}(a, c) \iff V_t(a) \geq \theta(c)$$

Where $\theta(c)$ is the minimum trust required to bid on contract $c$.

**C# Implementation:**

```csharp
/// <summary>
/// Determines agent eligibility for contracts based on trust thresholds.
/// </summary>
public static class EligibilityChecker
{
    /// <summary>
    /// Determines if an agent is eligible to bid on a contract.
    /// eligible(a, c) âŸº V_t(a) â‰¥ Î¸(c)
    /// </summary>
    /// <param name="agent">The agent to check eligibility for.</param>
    /// <param name="contract">The contract to check eligibility against.</param>
    /// <returns>True if the agent meets the trust threshold for this contract.</returns>
    /// <exception cref="ArgumentNullException">Thrown if agent or contract is null.</exception>
    /// <exception cref="ArgumentException">Thrown if contract has no skill type.</exception>
    public static bool IsEligible(Agent agent, Contract contract)
    {
        if (agent == null)
            throw new ArgumentNullException(nameof(agent));
        if (contract == null)
            throw new ArgumentNullException(nameof(contract));
        if (!SkillTypes.IsValid(contract.SkillType))
            throw new ArgumentException("Contract must have a skill type", nameof(contract));

        double agentTrust = agent.ComputeTrustValue(contract.SkillType);
        double threshold = CalculateThreshold(contract);
        return agentTrust >= threshold;
    }

    /// <summary>
    /// Calculates the minimum trust threshold for a contract.
    /// Î¸(c) - Higher stakes and difficulty require higher trust.
    /// Includes a minimum threshold based on stake alone to prevent
    /// zero-difficulty contracts from having zero threshold.
    /// </summary>
    /// <param name="contract">The contract to calculate threshold for.</param>
    /// <returns>The minimum trust value required to bid on this contract.</returns>
    /// <exception cref="ArgumentNullException">Thrown if contract is null.</exception>
    public static double CalculateThreshold(Contract contract)
    {
        if (contract == null)
            throw new ArgumentNullException(nameof(contract));

        double stakeComponent = Math.Log(1 + contract.Stake);

        // Minimum threshold based on stake alone (10% of stake factor)
        double minimumThreshold = stakeComponent * TrustConstants.MinimumThresholdFactor;

        // Standard threshold scales with stake and difficulty
        double difficultyThreshold = stakeComponent * contract.Difficulty;

        // Return the higher of the two
        return Math.Max(minimumThreshold, difficultyThreshold);
    }
}

/// <summary>
/// Extension methods for filtering eligible contracts.
/// </summary>
public static class ContractMarketplace
{
    /// <summary>
    /// Filters a list of contracts to only those the agent is eligible for.
    /// Silently skips invalid contracts (null or missing skill type).
    /// </summary>
    public static IEnumerable<Contract> GetEligibleContracts(
        this Agent agent,
        IEnumerable<Contract> availableContracts)
    {
        if (agent == null)
            throw new ArgumentNullException(nameof(agent));
        if (availableContracts == null)
            return Enumerable.Empty<Contract>();

        return availableContracts
            .Where(c => c != null && SkillTypes.IsValid(c.SkillType))
            .Where(c => EligibilityChecker.IsEligible(agent, c));
    }

    /// <summary>
    /// Gets the highest-value contract the agent is eligible for.
    /// </summary>
    public static Contract? GetBestEligibleContract(
        this Agent agent,
        IEnumerable<Contract> availableContracts)
    {
        return agent
            .GetEligibleContracts(availableContracts)
            .OrderByDescending(c => c.Stake)
            .FirstOrDefault();
    }

    /// <summary>
    /// Gets all contracts the agent is eligible for, sorted by stake descending.
    /// </summary>
    public static IEnumerable<Contract> GetEligibleContractsByValue(
        this Agent agent,
        IEnumerable<Contract> availableContracts)
    {
        return agent
            .GetEligibleContracts(availableContracts)
            .OrderByDescending(c => c.Stake);
    }
}
```

---

## 11. Convergence Criterion (Validation)

Before deploying the trust network to human participants, we can validate it using AI agents in simulation. The convergence criterion measures whether the network's computed trust values actually reflect agents' true reliability. In simulation, we know each agent's actual reliability (R)â€”their programmed probability of successful delivery. As the network accumulates history, the correlation between computed trust (V) and actual reliability (R) should approach 1.0. When this correlation is high (e.g., >0.95), the trust mathematics are working correctlyâ€”the network successfully discovers who's genuinely capable based purely on observed actions, without any identity information.

**Mathematical notation:**
$$\lim_{n \to \infty} \text{Corr}\big(V_t^{(n)}(a), R_t(a)\big) = 1$$

Where $R_t(a)$ is the agent's actual reliability (known in simulation).

**C# Implementation:**

```csharp
/// <summary>
/// Validates trust network convergence through statistical analysis.
/// </summary>
public static class NetworkValidator
{
    /// <summary>
    /// Measures how well trust values correlate with actual reliability.
    /// lim(nâ†’âˆž) Corr(V_t^(n)(a), R_t(a)) = 1 indicates validation.
    /// </summary>
    /// <param name="agents">The agents to analyze.</param>
    /// <param name="getActualReliability">Function to get actual reliability (known in simulation).</param>
    /// <param name="skillType">The skill type to evaluate.</param>
    /// <returns>Pearson correlation coefficient, or NaN if insufficient data.</returns>
    /// <exception cref="ArgumentNullException">Thrown if agents or function is null.</exception>
    /// <exception cref="ArgumentException">Thrown if skill type is empty.</exception>
    public static double CalculateConvergence(
        IEnumerable<Agent> agents,
        Func<Agent, string, double> getActualReliability,
        string skillType)
    {
        if (agents == null)
            throw new ArgumentNullException(nameof(agents));
        if (getActualReliability == null)
            throw new ArgumentNullException(nameof(getActualReliability));
        if (!SkillTypes.IsValid(skillType))
            throw new ArgumentException("Skill type is required", nameof(skillType));

        var dataPoints = agents
            .Select(a => (
                TrustValue: a.ComputeTrustValue(skillType),
                ActualReliability: getActualReliability(a, skillType)))
            .ToList();

        // Need at least 2 points for correlation
        if (dataPoints.Count < 2)
            return double.NaN;

        return PearsonCorrelation(
            dataPoints.Select(p => p.TrustValue),
            dataPoints.Select(p => p.ActualReliability));
    }

    /// <summary>
    /// Async version for large agent populations.
    /// </summary>
    public static async Task<double> CalculateConvergenceAsync(
        IEnumerable<Agent> agents,
        Func<Agent, string, double> getActualReliability,
        string skillType,
        CancellationToken cancellationToken = default)
    {
        if (agents == null)
            throw new ArgumentNullException(nameof(agents));
        if (getActualReliability == null)
            throw new ArgumentNullException(nameof(getActualReliability));
        if (!SkillTypes.IsValid(skillType))
            throw new ArgumentException("Skill type is required", nameof(skillType));

        var dataPoints = await Task.Run(() =>
            agents
                .AsParallel()
                .WithCancellation(cancellationToken)
                .Select(a => (
                    TrustValue: a.ComputeTrustValue(skillType),
                    ActualReliability: getActualReliability(a, skillType)))
                .ToList(),
            cancellationToken);

        if (dataPoints.Count < 2)
            return double.NaN;

        return PearsonCorrelation(
            dataPoints.Select(p => p.TrustValue),
            dataPoints.Select(p => p.ActualReliability));
    }

    /// <summary>
    /// Determines if the network has validated (correlation exceeds threshold).
    /// </summary>
    public static bool IsValidated(double correlation)
    {
        return !double.IsNaN(correlation) && correlation >= TrustConstants.ValidationThreshold;
    }

    /// <summary>
    /// Calculates Pearson correlation coefficient between two sequences.
    /// </summary>
    /// <returns>
    /// Correlation coefficient in range [-1, 1], or NaN if:
    /// - Either sequence has fewer than 2 elements
    /// - Sequences have different lengths
    /// - Either sequence has zero variance (all values identical)
    /// </returns>
    /// <remarks>
    /// Zero variance case: If all trust values are identical (e.g., all agents have 0 trust),
    /// the denominator becomes 0 and NaN is returned. This is mathematically correctâ€”
    /// correlation is undefined when there's no variation to correlate.
    /// </remarks>
    private static double PearsonCorrelation(
        IEnumerable<double> x,
        IEnumerable<double> y)
    {
        var xList = x.ToList();
        var yList = y.ToList();

        if (xList.Count != yList.Count || xList.Count < 2)
            return double.NaN;

        double n = xList.Count;
        double sumX = xList.Sum();
        double sumY = yList.Sum();
        double sumXY = xList.Zip(yList, (a, b) => a * b).Sum();
        double sumX2 = xList.Sum(v => v * v);
        double sumY2 = yList.Sum(v => v * v);

        double numerator = (n * sumXY) - (sumX * sumY);
        double denominator = Math.Sqrt(
            ((n * sumX2) - (sumX * sumX)) *
            ((n * sumY2) - (sumY * sumY)));

        if (denominator == 0)
            return double.NaN;

        return numerator / denominator;
    }
}

/// <summary>
/// An agent with known actual reliability for simulation testing.
/// </summary>
public class SimulatedAgent : Agent
{
    /// <summary>R_t(a) - the agent's actual reliability, known only in simulation.</summary>
    public double ActualReliability { get; set; }
}

/// <summary>
/// Example demonstrating network validation in simulation.
/// </summary>
public class SimulationValidator
{
    public void ValidateNetwork(List<SimulatedAgent> agents)
    {
        double correlation = NetworkValidator.CalculateConvergence(
            agents,
            (agent, skill) => agent is SimulatedAgent sim ? sim.ActualReliability : 0.0,
            SkillTypes.Engineering);

        bool isValidated = NetworkValidator.IsValidated(correlation);

        Console.WriteLine($"Convergence: {correlation:F4}");
        Console.WriteLine($"Validated: {isValidated}");

        if (!isValidated && !double.IsNaN(correlation))
        {
            Console.WriteLine($"Need {TrustConstants.ValidationThreshold:P0} correlation, " +
                            $"currently at {correlation:P0}");
        }
    }
}
```

---

## 12. Sybil Resistance

A Sybil attack occurs when a malicious actor creates multiple fake identities to game a reputation systemâ€”for example, creating sock-puppet accounts to inflate ratings. The Quantum of Trust framework has built-in Sybil resistance through its economic structure. If an attacker splits their activity across k fake identities, each identity accumulates only 1/k of the history that an honest agent would build. Less history means lower trust. Lower trust means eligibility for fewer and lower-quality contracts. The economics favor consolidation of reputation over fragmentationâ€”you're better off building genuine trust through a single agent than spreading thin across multiple sybils. This inequality formalizes that insight: an honest agent's history size will always exceed any individual sybil's history.

**Mathematical notation:**
$$|h_t(a_{\text{honest}})| > |h_t(a_{\text{sybil}_i})| \quad \forall i$$

Creating $k$ fake identities splits activity, resulting in less history per identity.

**C# Implementation:**

```csharp
/// <summary>
/// Analyzes Sybil resistance properties of the trust network.
/// </summary>
public static class SybilResistanceAnalyzer
{
    /// <summary>
    /// Analyzes whether honest agents have advantages over sybil identities.
    /// |h_t(a_honest)| > |h_t(a_sybil_i)| for all i
    /// </summary>
    /// <param name="honestAgent">An agent operating honestly with single identity.</param>
    /// <param name="sybilAgents">Multiple identities operated by an attacker.</param>
    /// <param name="skillType">The skill type to analyze.</param>
    /// <returns>Analysis results showing honest vs sybil comparison.</returns>
    /// <exception cref="ArgumentNullException">Thrown if honestAgent or sybilAgents is null.</exception>
    /// <exception cref="ArgumentException">Thrown if skill type is empty.</exception>
    public static SybilAnalysisResult AnalyzeSybilResistance(
        Agent honestAgent,
        List<Agent> sybilAgents,
        string skillType)
    {
        if (honestAgent == null)
            throw new ArgumentNullException(nameof(honestAgent));
        if (sybilAgents == null)
            throw new ArgumentNullException(nameof(sybilAgents));
        if (!SkillTypes.IsValid(skillType))
            throw new ArgumentException("Skill type is required", nameof(skillType));

        int honestHistorySize = honestAgent.ContractCountForSkill(skillType);

        var sybilHistorySizes = sybilAgents
            .Select(s => s.ContractCountForSkill(skillType))
            .ToList();

        double honestTrust = honestAgent.ComputeTrustValue(skillType);

        var sybilTrusts = sybilAgents
            .Select(s => s.ComputeTrustValue(skillType))
            .ToList();

        // Sybil resistant if there are sybils AND honest has more history than every sybil
        // Empty sybil list means we can't claim resistance (no comparison to make)
        bool isSybilResistant = sybilHistorySizes.Any() &&
            sybilHistorySizes.All(sybilSize => honestHistorySize > sybilSize);

        double maxSybilTrust = sybilTrusts.Any() ? sybilTrusts.Max() : 0.0;
        double advantageRatio = CalculateAdvantageRatio(honestTrust, maxSybilTrust);

        return new SybilAnalysisResult
        {
            HonestHistorySize = honestHistorySize,
            SybilHistorySizes = sybilHistorySizes,
            HonestTrust = honestTrust,
            SybilTrusts = sybilTrusts,
            IsSybilResistant = isSybilResistant,
            HonestAdvantageRatio = advantageRatio
        };
    }

    /// <summary>
    /// Calculates the advantage ratio between honest and sybil trust values.
    /// Handles edge cases with negative trust values correctly.
    /// </summary>
    /// <param name="honestTrust">The honest agent's trust value.</param>
    /// <param name="maxSybilTrust">The maximum trust among sybil agents.</param>
    /// <returns>
    /// Ratio where > 1 means honest is advantaged.
    /// Returns PositiveInfinity if honest is trusted but sybils are not.
    /// Returns 0 if sybils are trusted but honest is not.
    /// Returns 1 if both are equal or both are zero.
    /// </returns>
    public static double CalculateAdvantageRatio(double honestTrust, double maxSybilTrust)
    {
        // Both zero: no advantage either way
        if (Math.Abs(maxSybilTrust) < 1e-10 && Math.Abs(honestTrust) < 1e-10)
            return 1.0;

        // Honest is positive/zero, sybils are negative: honest infinitely better
        if (honestTrust >= 0 && maxSybilTrust < 0)
            return double.PositiveInfinity;

        // Sybils are positive, honest is negative: sybils are better (honest disadvantaged)
        if (maxSybilTrust > 0 && honestTrust < 0)
            return 0.0;

        // Both negative: less negative is better
        // honestTrust = -10, maxSybilTrust = -20 â†’ honest is "better"
        // Ratio = -20 / -10 = 2.0 (honest is 2x better)
        if (maxSybilTrust < 0 && honestTrust < 0)
        {
            if (Math.Abs(honestTrust) < 1e-10)
                return double.PositiveInfinity; // Honest nearly zero, sybils negative
            return maxSybilTrust / honestTrust;
        }

        // Both positive: higher is better
        if (Math.Abs(maxSybilTrust) < 1e-10)
            return double.PositiveInfinity; // Avoid division by zero
        return honestTrust / maxSybilTrust;
    }
}

/// <summary>
/// Results of Sybil resistance analysis.
/// </summary>
public class SybilAnalysisResult
{
    public int HonestHistorySize { get; init; }
    public IReadOnlyList<int> SybilHistorySizes { get; init; } = Array.Empty<int>();
    public double HonestTrust { get; init; }
    public IReadOnlyList<double> SybilTrusts { get; init; } = Array.Empty<double>();

    /// <summary>
    /// True if the honest agent has more history than all sybils AND there is at least one sybil.
    /// </summary>
    public bool IsSybilResistant { get; init; }

    /// <summary>
    /// Ratio of honest trust to maximum sybil trust.
    /// Values > 1 indicate honest agents are advantaged.
    /// </summary>
    public double HonestAdvantageRatio { get; init; }

    public override string ToString() =>
        $"Honest: {HonestHistorySize} contracts, {HonestTrust:F2} trust | " +
        $"Best Sybil: {(SybilHistorySizes.Any() ? SybilHistorySizes.Max() : 0)} contracts, " +
        $"{(SybilTrusts.Any() ? SybilTrusts.Max() : 0):F2} trust | " +
        $"Advantage: {HonestAdvantageRatio:F1}x | Resistant: {IsSybilResistant}";
}

/// <summary>
/// Example demonstrating why sybils are economically disadvantaged.
/// </summary>
public class SybilExample
{
    public void DemonstrateSybilDisadvantage()
    {
        // Honest agent completes 100 contracts
        var honest = new Agent { SkillType = SkillTypes.Engineering };
        for (int i = 0; i < 100; i++)
        {
            honest.AddToHistory(ContractFactory.CreateSimple(
                skillType: SkillTypes.Engineering,
                outcome: 1.0,
                weight: 1.0));
        }

        // Attacker splits same effort across 5 sybil identities
        var sybils = Enumerable.Range(0, 5)
            .Select(_ => new Agent { SkillType = SkillTypes.Engineering })
            .ToList();

        for (int i = 0; i < 100; i++)
        {
            // Each contract goes to one sybil (round-robin)
            sybils[i % 5].AddToHistory(ContractFactory.CreateSimple(
                skillType: SkillTypes.Engineering,
                outcome: 1.0,
                weight: 1.0));
        }

        // Analyze the results
        var result = SybilResistanceAnalyzer.AnalyzeSybilResistance(
            honest, sybils, SkillTypes.Engineering);

        Console.WriteLine(result);
        // Output:
        // Honest: 100 contracts, 100.00 trust | Best Sybil: 20 contracts, 20.00 trust | 
        // Advantage: 5.0x | Resistant: True
    }
}
```

---

## 13. Test Helpers

Utility classes for unit testing the trust framework.

```csharp
/// <summary>
/// Helpers for unit testing the trust framework.
/// </summary>
public static class TrustTestHelpers
{
    /// <summary>
    /// Creates an agent with pre-populated history for testing.
    /// </summary>
    /// <param name="skillType">The skill type for the agent.</param>
    /// <param name="successCount">Number of successful contracts to add.</param>
    /// <param name="failureCount">Number of failed contracts to add.</param>
    /// <param name="weight">Weight for each contract.</param>
    /// <returns>An agent with the specified history.</returns>
    public static Agent CreateAgentWithHistory(
        string skillType,
        int successCount,
        int failureCount,
        double weight = 1.0)
    {
        var agent = new Agent { SkillType = skillType };

        for (int i = 0; i < successCount; i++)
            agent.AddToHistory(ContractFactory.CreateSimple(skillType, 1.0, weight));

        for (int i = 0; i < failureCount; i++)
            agent.AddToHistory(ContractFactory.CreateSimple(skillType, -1.0, weight));

        return agent;
    }

    /// <summary>
    /// Creates a DAO with the specified number of agents.
    /// </summary>
    public static DAO CreateDAOWithAgents(
        int agentCount,
        string skillType,
        Func<IEnumerable<double>, double> phi)
    {
        var agents = Enumerable.Range(0, agentCount)
            .Select(_ => new Agent { SkillType = skillType })
            .Cast<QuantumOfTrust>()
            .ToHashSet();

        return new DAO { Members = agents, Phi = phi };
    }

    /// <summary>
    /// Creates a list of simulated agents with random reliabilities.
    /// </summary>
    public static List<SimulatedAgent> CreateSimulatedAgents(
        int count,
        string skillType,
        Random? random = null)
    {
        random ??= new Random();
        return Enumerable.Range(0, count)
            .Select(_ => new SimulatedAgent
            {
                SkillType = skillType,
                ActualReliability = random.NextDouble()
            })
            .ToList();
    }

    /// <summary>
    /// Simulates contract execution for simulated agents based on their reliability.
    /// </summary>
    public static void SimulateContracts(
        List<SimulatedAgent> agents,
        int contractsPerAgent,
        string skillType,
        double weight = 1.0,
        Random? random = null)
    {
        random ??= new Random();

        foreach (var agent in agents)
        {
            for (int i = 0; i < contractsPerAgent; i++)
            {
                // Outcome based on reliability: higher reliability = higher chance of success
                double roll = random.NextDouble();
                double outcome = roll < agent.ActualReliability ? 1.0 : -1.0;

                agent.AddToHistory(ContractFactory.CreateSimple(skillType, outcome, weight));
            }
        }
    }
}
```

---

## Complete Working Example

Here's a complete example bringing all the pieces together:

```csharp
using System;
using System.Collections.Generic;
using System.Linq;

namespace QuantumOfTrust
{
    public class TrustNetworkDemo
    {
        public static void Main()
        {
            var weightCalculator = new WeightCalculator();
            var trustTracker = new TrustTracker();

            // Create Jane's two agencies (separate avatars, same human)
            var janeEngineer = new Agent { SkillType = SkillTypes.Engineering };
            var janeDesigner = new Agent { SkillType = SkillTypes.Design };

            // Create counterparty agents
            var saasCompany = new Agent { SkillType = "consumer" };
            var marketingFirm = new Agent { SkillType = "consumer" };

            // Jane completes engineering contracts successfully
            Console.WriteLine("=== Engineering Contracts ===");
            for (int i = 0; i < 10; i++)
            {
                var contract = new ContractBuilder()
                    .WithProvider(janeEngineer)
                    .WithConsumer(saasCompany)
                    .WithSkillType(SkillTypes.Engineering)
                    .WithStake(5000)
                    .WithDifficulty(7)
                    .WithDeadline(TrustConstants.Now.AddDays(14))
                    .WithOutcome(1.0)
                    .CompletedAt(TrustConstants.Now.AddDays(-30 * i))
                    .WithWeightCalculator(weightCalculator)
                    .Build();

                trustTracker.AddContractAndUpdateTrust(janeEngineer, contract, TrustConstants.Now);
            }

            // Jane's design work has mixed results
            Console.WriteLine("\n=== Design Contracts ===");
            for (int i = 0; i < 5; i++)
            {
                double outcome = i < 2 ? 0.5 : -0.5;  // First 2 partial success, rest partial failure

                var contract = new ContractBuilder()
                    .WithProvider(janeDesigner)
                    .WithConsumer(marketingFirm)
                    .WithSkillType(SkillTypes.Design)
                    .WithStake(2000)
                    .WithDifficulty(5)
                    .WithDeadline(TrustConstants.Now.AddMonths(1))
                    .WithOutcome(outcome)
                    .CompletedAt(TrustConstants.Now.AddDays(-60 * i))
                    .WithWeightCalculator(weightCalculator)
                    .Build();

                trustTracker.AddContractAndUpdateTrust(janeDesigner, contract, TrustConstants.Now);
            }

            // Compute and display trust values
            double engineerTrust = janeEngineer.ComputeTrustValue(SkillTypes.Engineering);
            double designerTrust = janeDesigner.ComputeTrustValue(SkillTypes.Design);

            Console.WriteLine($"\n=== Trust Values ===");
            Console.WriteLine($"Jane's Engineering q<T>: {engineerTrust:F2} cutes");
            Console.WriteLine($"Jane's Design q<T>: {designerTrust:F2} cutes");

            // Check eligibility for contracts
            Console.WriteLine($"\n=== Eligibility ===");

            var smallContract = Contract.Create(
                provider: janeEngineer,
                consumer: saasCompany,
                skillType: SkillTypes.Engineering,
                stake: 1000,
                difficulty: 3,
                deadline: TrustConstants.Now.AddDays(7));

            var bigContract = Contract.Create(
                provider: janeEngineer,
                consumer: saasCompany,
                skillType: SkillTypes.Engineering,
                stake: 50000,
                difficulty: 9,
                deadline: TrustConstants.Now.AddMonths(3));

            Console.WriteLine($"Small contract threshold: {EligibilityChecker.CalculateThreshold(smallContract):F2}");
            Console.WriteLine($"Eligible for small contract: {EligibilityChecker.IsEligible(janeEngineer, smallContract)}");
            Console.WriteLine($"Big contract threshold: {EligibilityChecker.CalculateThreshold(bigContract):F2}");
            Console.WriteLine($"Eligible for big contract: {EligibilityChecker.IsEligible(janeEngineer, bigContract)}");

            // Create a DAO
            Console.WriteLine($"\n=== DAO ===");
            var engineeringDAO = new DAO
            {
                Members = new HashSet<QuantumOfTrust> { janeEngineer, saasCompany },
                Phi = AggregationFunctions.Sum
            };

            double daoTrust = engineeringDAO.ComputeTrustValue(SkillTypes.Engineering);
            Console.WriteLine($"Engineering DAO q<T> (sum): {daoTrust:F2} cutes");

            engineeringDAO.Phi = AggregationFunctions.Average;
            daoTrust = engineeringDAO.ComputeTrustValue(SkillTypes.Engineering);
            Console.WriteLine($"Engineering DAO q<T> (avg): {daoTrust:F2} cutes");

            // Demonstrate Sybil resistance
            Console.WriteLine($"\n=== Sybil Resistance ===");
            var sybilExample = new SybilExample();
            sybilExample.DemonstrateSybilDisadvantage();

            // Show audit log
            Console.WriteLine($"\n=== Audit Log (last 5 entries) ===");
            foreach (var entry in trustTracker.AuditLog.TakeLast(5))
            {
                Console.WriteLine(entry);
            }

            // Run a simulation validation
            Console.WriteLine($"\n=== Simulation Validation ===");
            var random = new Random(42);
            var simulatedAgents = TrustTestHelpers.CreateSimulatedAgents(50, SkillTypes.Engineering, random);
            TrustTestHelpers.SimulateContracts(simulatedAgents, 100, SkillTypes.Engineering, 1.0, random);

            double correlation = NetworkValidator.CalculateConvergence(
                simulatedAgents,
                (agent, skill) => agent is SimulatedAgent sim ? sim.ActualReliability : 0.0,
                SkillTypes.Engineering);

            Console.WriteLine($"Correlation between trust and actual reliability: {correlation:F4}");
            Console.WriteLine($"Network validated: {NetworkValidator.IsValidated(correlation)}");
        }
    }
}
```

---

## Summary of Equation Mappings

| Math Notation | C# Implementation |
|--------------|-------------------|
| $q\langle T \rangle$ | `QuantumOfTrust` (abstract base class implementing `IQuantumOfTrust`) |
| $\text{Agent}(t, h_t)$ | `Agent` class with `SkillType`, `IReadOnlyList<Contract> ContractHistory`, and `IEquatable<Agent>` |
| $\text{DAO}(\{q\langle T \rangle\})$ | `DAO` class with `HashSet<QuantumOfTrust>`, nullable `Phi`, cycle detection, and `IEquatable<DAO>` |
| $V_t^{\text{provider}}(q)$ | `ComputeTrustValue(string skillType)` with case-insensitive skill matching |
| $V_t^{\text{customer}}(q)$ | `CustomerTrustCalculator.ComputeCustomerTrust()` family of methods |
| $\sum_{c \in h_t} \omega(c) \cdot \text{outcome}(c)$ | LINQ `.Sum(c => c.Weight * c.Outcome)` |
| $\Phi(\{V_t(q) : q \in S\})$ | `Func<IEnumerable<double>, double>? Phi` with null checks and materialization |
| $h_t^{(n+1)} = h_t^{(n)} \cup \{c_n\}$ | `AddToHistory(Contract)` with null validation |
| $V_t^{(n+1)} = V_t^{(n)} + \omega(c_n) \cdot \text{outcome}(c_n)$ | `TrustTracker.UpdateTrust()` with `ConcurrentDictionary` and audit log |
| $\omega(c) = f(s, d, V_t, \text{recency})$ | `WeightCalculator.ComputeWeight()` with all constants defined |
| $\text{eligible}(a, c) \iff V_t(a) \geq \theta(c)$ | `EligibilityChecker.IsEligible()` with minimum threshold |
| $\text{Corr}(V_t^{(n)}, R_t)$ | `NetworkValidator.CalculateConvergence()` with documented edge cases |
| $\|h_t(a_{\text{honest}})\| > \|h_t(a_{\text{sybil}_i})\|$ | `SybilResistanceAnalyzer` with correct advantage ratio for negative values |
| $\text{verification\_weight}(c)$ | `VerificationWeightCalculator.ComputeVerificationWeight()` |
| **Hierarchical Contracts** | |
| $\text{ContractType} \in \{0..5\}$ | `ContractType` enum (Standalone=0 through Milestone=5) |
| $\text{Milestone}(\{T_i\})$ | `MilestoneWithTasks` class with `Tasks`, `ComputeStake()`, `ComputeDeadline()` |
| $d_{\text{milestone}} = \max_i(d_{\text{task}_i})$ | `MilestoneWithTasks.ComputeDeadline()` |
| $s_{\text{milestone}} = \sum_i s_{\text{task}_i}$ | `MilestoneWithTasks.ComputeStake()` |
| **Task Decomposition** | |
| $s_{\text{task}} = s_{\text{milestone}} \cdot \frac{w}{\sum w}$ | `Task.ComputeStake()` |
| $d_{\text{phase}} = \frac{\sum d_{\text{task}} \cdot s_{\text{task}}}{\sum s_{\text{task}}}$ | `PhaseWithTasks.ComputeAggregateDifficulty()` |
| $\text{planning\_accuracy}$ | `TaskDecomposition.ComputePlanningAccuracy()` or `Contract.PlanningAccuracy` |
| Customer skill types | `CustomerSkillTypes` constants + `CustomerProfile` class |

---

## Key Architectural Additions (Pass 4)

### Bidirectional Trust
- ✅ `CustomerProfile` class captures customer behavior metrics
- ✅ `CustomerTrustCalculator` computes trust from 6 customer skill types
- ✅ `CustomerSkillTypes` constants define customer skill dimensions
- ✅ Customers judged by behavior, not assigned outcomes

### Verification Weight
- ✅ `VerificationWeightCalculator` adjusts rating credibility
- ✅ Customer verification integrity affects how much their ratings count
- ✅ Rating variance used to detect rubber-stamping or erratic behavior

### Task Decomposition
- ✅ `Task` class for atomic work units within milestones
- ✅ `Task.Difficulty` property for provider-assessed task difficulty
- ✅ `Task.Deadline` property for work duration (planning data, not payment trigger)
- ✅ `Task.ParentMilestoneId` reference to parent milestone
- ✅ `MilestoneWithTasks` class for payment gates within Implementation phases
- ✅ `MilestoneWithTasks.ComputeDeadline()` from max of task deadlines
- ✅ `MilestoneWithTasks.ComputeStake()` from sum of task stakes
- ✅ `PhaseWithTasks` enables team-based implementation with milestones
- ✅ `PhaseWithTasks.ComputeAggregateDifficulty()` for stake-weighted difficulty aggregation
- ✅ `TaskDecomposition` utilities for stake allocation
- ✅ `ContractType` enum for hierarchical classification (0-5, including Milestone=5)
- ✅ `Contract` extended with hierarchical fields (Type, ParentRef, TaskWeight, etc.)
- ✅ Trust flows through tasks, milestones are coordination containers

---

## All Fixes Applied

This document incorporates all fixes from four code review passes:

### Pass 1 Fixes (8 errors, 13 improvements)
- ✅ Consolidated `Agent` class with `IReadOnlyList<Contract>` for history
- ✅ Added null check for `DAO.Phi` before use
- ✅ `AggregationFunctions` handle null and empty sequences
- ✅ `WeightedAverage` handles zero total weight
- ✅ `TrustTracker` uses stored `Contract.Weight` for consistency
- ✅ `Agent` and `DAO` implement `IEquatable` with GUID-based identity
- ✅ Stake validation prevents negative values
- ✅ All constants extracted to `TrustConstants`
- ✅ Interfaces for dependency injection
- ✅ Comprehensive XML documentation
- ✅ `init` properties for immutability

### Pass 2 Fixes (5 errors, 12 improvements)
- ✅ `ContractFactory.CreateSimple` for examples
- ✅ `SimulationValidator` uses pattern matching for safe cast
- ✅ `DAO.Members` setter validates against null
- ✅ Cycle detection in `DAO.ComputeTrustValue`
- ✅ `ContractMarketplace.GetEligibleContracts` filters invalid contracts
- ✅ `ContractBuilder` for cleaner construction
- ✅ `DAO` implements `IEquatable`
- ✅ `TrustTracker.AddContractAndUpdateTrust` for atomic operations
- ✅ `CompletedAt` validates not in future
- ✅ `Median` aggregation function added
- ✅ `ToString` overrides for debugging
- ✅ Test helpers class

### Pass 3 Fixes (6 errors, 10 improvements)
- ✅ `HonestAdvantageRatio` correctly handles negative trust values
- ✅ `IsSybilResistant` returns false for empty sybil list
- ✅ `CreateWeightedAveragePhi` factory for Phi-compatible weighted average
- ✅ `DAO.ComputeTrustValue` materializes to prevent multiple enumerations
- ✅ Case-insensitive skill type comparison via `SkillTypes` utility
- ✅ `TrustTracker` uses stored weight instead of recalculating
- ✅ `AggregationFunctions` null checks on all methods
- ✅ Minimum threshold for zero-difficulty contracts
- ✅ `DateTime.UtcNow` via `TrustConstants.Now`
- ✅ `Contract.SkillType` validates non-empty
- ✅ `Contract.Validate()` method
- ✅ `Contract.Id` for tracking
- ✅ `TrustChangeEvent` audit log
- ✅ Pearson correlation edge cases documented

### Pass 4 Additions (Bidirectional Trust & Task Decomposition)
- ✅ `CustomerSkillTypes` constants for customer behavior dimensions
- ✅ `ContractType` enum for hierarchical contract classification (Standalone=0 through Milestone=5)
- ✅ `Contract` extended with Type, ParentRef, TaskWeight, planning fields
- ✅ `Task` class for atomic work units with provider assignment and deadline
- ✅ `MilestoneWithTasks` class for payment gates within Implementation phases
- ✅ `PhaseWithTasks` for team-based implementation with milestone support
- ✅ `CustomerProfile` class for customer behavior metrics
- ✅ `CustomerTrustCalculator` with 6 skill-specific computation methods
- ✅ `VerificationWeightCalculator` for rating credibility adjustment
- ✅ `TaskDecomposition` utilities for stake allocation and aggregation

### Pass 5 Additions (Milestone Payment Gates)
- ✅ `ContractType.Milestone = 5` for payment gate classification
- ✅ `MilestoneWithTasks` class with computed deadline and stake
- ✅ `Task.ParentMilestoneId` reference to parent milestone
- ✅ `Task.Deadline` for work duration (planning data)
- ✅ 4-level hierarchy: Project → Phase → Milestone → Task
- ✅ Trust flows through tasks, milestones coordinate payment
- ✅ See: ADR_Milestone_Payment_Gates.md, ADR_Dispute_Resolution.md
