# Sybil Resistance Circuits in Noir

## Supplementary Implementation for Defense-in-Depth

---

This document extends the base Quantum of Trust Noir implementation with four Sybil resistance mechanisms. It should be read alongside `Quantum_of_Trust_Equations_in_Noir.md`.

---

## Table of Contents

1. [Overview](#overview)
2. [Enhanced Contract Structure](#enhanced-contract-structure)
3. [Counterparty Trust Factor](#counterparty-trust-factor)
4. [Velocity Weight](#velocity-weight)
5. [Outcome Variance Constraint](#outcome-variance-constraint)
6. [Escrow Verification](#escrow-verification)
7. [Enhanced Trust Calculation](#enhanced-trust-calculation)
8. [Combined Eligibility Proof](#combined-eligibility-proof)
9. [Integration Notes](#integration-notes)

---

## Overview

The base implementation provides Sybil resistance through history size, depth, and counterparty diversity checks. These mechanisms are insufficient against sophisticated attacks where Sybils create fake contracts between themselves over extended periods.

This extension adds four complementary mechanisms:

| Mechanism | Circuit | Purpose |
|-----------|---------|---------|
| Economic Escrow | `verify_escrow()` | Proves real funds are locked |
| Counterparty Weighting | `compute_counterparty_factor()` | Scales contribution by counterparty trust |
| Outcome Variance | `prove_plausible_history()` | Catches suspiciously uniform outcomes |
| Velocity Limits | `compute_velocity_weight()` | Prevents burst accumulation |

---

## Enhanced Contract Structure

### Mathematical Notation

$$c = (a_{\text{provider}}, a_{\text{consumer}}, t, s, d, \tau, \varepsilon, V_{\text{consumer}})$$

### Noir Implementation

```noir
// ============================================
// SYBIL RESISTANCE PARAMETERS
// ============================================

/// Scale parameter for counterparty trust sigmoid (λ).
/// Controls how quickly γ rises with counterparty trust.
/// At λ=50: γ(0)=0.5, γ(50)≈0.73, γ(100)≈0.88
global COUNTERPARTY_SIGMOID_SCALE: Field = 50;

/// Time period for velocity checks in days.
global VELOCITY_PERIOD_DAYS: Field = 7;

/// Number of full-weight contracts allowed per velocity period.
global VELOCITY_ALLOWANCE: Field = 10;

/// Decay rate for contracts beyond velocity allowance.
/// k=0.5 means contract 11 has weight 0.67, contract 15 has weight 0.29
global VELOCITY_DECAY_RATE: Field = 500000;  // 0.5 * PRECISION

/// Minimum history size before variance is checked.
global VARIANCE_EXEMPTION_SIZE: Field = 10;

/// Minimum required outcome variance (scaled by PRECISION).
/// 0.1 variance requires some contracts to deviate from the mean.
global MINIMUM_VARIANCE: Field = 100000;  // 0.1 * PRECISION

/// Maximum valid escrow amount (prevents overflow).
global MAX_ESCROW_AMOUNT: Field = 1000000000000;  // 1 trillion tokens

// ============================================
// ENHANCED CONTRACT STRUCTURE
// ============================================

/// A completed contract with Sybil resistance fields.
/// 
/// Extends the base Contract with:
/// - `escrow_commitment`: Cryptographic proof of locked stake
/// - `counterparty_trust_snapshot`: Consumer's trust at completion time
/// 
/// The escrow commitment is verified separately; the snapshot enables
/// counterparty weighting without circular dependencies.
struct EnhancedContract {
    /// The other party in this contract.
    counterparty: AgentId,
    
    /// The skill type this contract was executed under.
    skill_type: SkillType,
    
    /// The stake value in tokens (unscaled).
    stake: Field,
    
    /// Difficulty rating from 0 to 10.
    difficulty: Field,
    
    /// Outcome stored as offset: 0=-100, 100=0, 200=+100.
    outcome_offset: Field,
    
    /// Timestamp when the contract was completed.
    completed_at: Field,
    
    /// Pre-computed base weight (scaled by PRECISION).
    weight: Field,
    
    /// Cryptographic commitment proving stake is locked.
    /// In Aztec: hash of the escrow note nullifier.
    escrow_commitment: Field,
    
    /// The counterparty's trust value when contract completed.
    /// Stored as scaled positive value (snapshot, not current).
    counterparty_trust_magnitude: Field,
    
    /// Whether the counterparty trust snapshot was negative.
    counterparty_trust_negative: bool,
}

impl EnhancedContract {
    /// Creates an empty/inactive contract for unused array slots.
    fn empty() -> Self {
        EnhancedContract {
            counterparty: AgentId::new(0),
            skill_type: SkillType::new(0),
            stake: 0,
            difficulty: 0,
            outcome_offset: OUTCOME_OFFSET,
            completed_at: 0,
            weight: 0,
            escrow_commitment: 0,
            counterparty_trust_magnitude: 0,
            counterparty_trust_negative: false,
        }
    }
    
    /// Gets the actual outcome as a Signed value in range [-100, +100].
    fn outcome(self) -> Signed {
        if self.outcome_offset >= OUTCOME_OFFSET {
            Signed::from_positive(self.outcome_offset - OUTCOME_OFFSET)
        } else {
            Signed::from_negative(OUTCOME_OFFSET - self.outcome_offset)
        }
    }
    
    /// Gets the counterparty trust snapshot as a Signed value.
    fn counterparty_trust(self) -> Signed {
        Signed::new(self.counterparty_trust_magnitude, self.counterparty_trust_negative)
    }
    
    /// Checks if this contract is for a given skill type.
    fn is_skill(self, skill: SkillType) -> bool {
        self.skill_type.eq(skill)
    }
    
    /// Checks if this contract slot is active.
    fn is_active(self) -> bool {
        self.weight != 0
    }
    
    /// Validates that all contract fields are within expected bounds.
    fn is_valid(self) -> bool {
        let valid_difficulty = self.difficulty <= MAX_DIFFICULTY;
        let valid_outcome = self.outcome_offset <= OUTCOME_MAX;
        let valid_stake = self.stake <= MAX_ESCROW_AMOUNT;
        valid_difficulty & valid_outcome & valid_stake
    }
}

/// Enhanced agent history with Sybil-resistant contracts.
struct EnhancedAgentHistory {
    /// Fixed-size array of contracts.
    contracts: [EnhancedContract; MAX_HISTORY],
    
    /// Number of active contracts in the array.
    count: Field,
    
    /// The agent this history belongs to.
    agent_id: AgentId,
}

impl EnhancedAgentHistory {
    /// Creates an empty history for a given agent.
    fn empty(agent: AgentId) -> Self {
        EnhancedAgentHistory {
            contracts: [EnhancedContract::empty(); MAX_HISTORY],
            count: 0,
            agent_id: agent,
        }
    }
}
```

---

## Counterparty Trust Factor

### Mathematical Notation

$$\gamma(c) = \sigma\left(\frac{V_t(a_{\text{consumer}})}{\lambda}\right)$$

Where σ(x) = 1 / (1 + e^(-x)) is the sigmoid function.

### Noir Implementation

```noir
// ============================================
// SIGMOID APPROXIMATION
// ============================================

/// Lookup table for sigmoid function.
/// x values (scaled by PRECISION).
global SIGMOID_TABLE_X: [Field; 11] = [
    0,           // σ(0) = 0.5
    500000,      // 0.5 * PRECISION
    1000000,     // 1.0 * PRECISION
    1500000,     // 1.5 * PRECISION
    2000000,     // 2.0 * PRECISION
    2500000,     // 2.5 * PRECISION
    3000000,     // 3.0 * PRECISION
    3500000,     // 3.5 * PRECISION
    4000000,     // 4.0 * PRECISION
    4500000,     // 4.5 * PRECISION
    5000000,     // 5.0 * PRECISION (saturation)
];

/// y values for sigmoid (scaled by PRECISION).
/// σ(x) for positive x; for negative x, use 1 - σ(|x|).
global SIGMOID_TABLE_Y: [Field; 11] = [
    500000,      // σ(0.0) = 0.500
    622459,      // σ(0.5) ≈ 0.622
    731059,      // σ(1.0) ≈ 0.731
    817574,      // σ(1.5) ≈ 0.818
    880797,      // σ(2.0) ≈ 0.881
    924142,      // σ(2.5) ≈ 0.924
    952574,      // σ(3.0) ≈ 0.953
    970688,      // σ(3.5) ≈ 0.971
    982014,      // σ(4.0) ≈ 0.982
    989013,      // σ(4.5) ≈ 0.989
    993307,      // σ(5.0) ≈ 0.993 (effective saturation)
];

/// Approximates the sigmoid function σ(x) = 1/(1+e^(-x)).
/// 
/// Uses linear interpolation between lookup table values.
/// For negative inputs, uses the identity σ(-x) = 1 - σ(x).
/// 
/// # Arguments
/// * `x` - Input value (scaled by PRECISION)
/// 
/// # Returns
/// σ(x) scaled by PRECISION, in range (0, PRECISION).
fn approx_sigmoid(x: Signed) -> Field {
    let abs_x = x.magnitude;
    
    // Find lookup table bounds
    let mut lower_idx: u32 = 0;
    let mut upper_idx: u32 = 1;
    
    for i in 0..10 {
        if SIGMOID_TABLE_X[i] <= abs_x {
            if SIGMOID_TABLE_X[i + 1] > abs_x {
                lower_idx = i;
                upper_idx = i + 1;
            }
        }
    }
    
    // Handle saturation
    let positive_result = if abs_x >= SIGMOID_TABLE_X[10] {
        SIGMOID_TABLE_Y[10]
    } else {
        let x_lower = SIGMOID_TABLE_X[lower_idx];
        let x_upper = SIGMOID_TABLE_X[upper_idx];
        let y_lower = SIGMOID_TABLE_Y[lower_idx];
        let y_upper = SIGMOID_TABLE_Y[upper_idx];
        
        if x_upper == x_lower {
            y_lower
        } else {
            assert(abs_x >= x_lower);
            let t = ratio(abs_x - x_lower, x_upper - x_lower);
            y_lower + fp_mul(t, y_upper - y_lower)
        }
    };
    
    // Apply sign: σ(-x) = 1 - σ(x)
    if x.is_negative {
        PRECISION - positive_result
    } else {
        positive_result
    }
}

/// Computes the counterparty trust factor γ(c).
/// 
/// γ(c) = σ(V_t(counterparty) / λ)
/// 
/// # Arguments
/// * `counterparty_trust` - The counterparty's trust value (from snapshot)
/// 
/// # Returns
/// Factor in range (0, PRECISION), representing (0, 1).
fn compute_counterparty_factor(counterparty_trust: Signed) -> Field {
    // Scale the trust value by λ (COUNTERPARTY_SIGMOID_SCALE)
    // scaled_x = trust / λ
    // Since trust is in "cutes" and λ=50, we need to scale properly
    
    // trust is scaled by PRECISION internally
    // We want sigmoid input to be trust/50
    // So we compute: magnitude * PRECISION / (50 * PRECISION) = magnitude / 50
    
    let scaled_magnitude = counterparty_trust.magnitude / COUNTERPARTY_SIGMOID_SCALE;
    let scaled_input = Signed::new(scaled_magnitude, counterparty_trust.is_negative);
    
    approx_sigmoid(scaled_input)
}
```

---

## Velocity Weight

### Mathematical Notation

$$\nu(c) = \frac{1}{1 + k \cdot \max(0, \text{rank}(c, T) - N)}$$

### Noir Implementation

```noir
// ============================================
// VELOCITY WEIGHT CALCULATION
// ============================================

/// Computes the velocity weight for a contract.
/// 
/// ν(c) = 1 / (1 + k × max(0, rank - N))
/// 
/// Contracts beyond the velocity allowance (N) get diminishing weight.
/// 
/// # Arguments
/// * `rank_in_period` - This contract's position in the current period (1-indexed)
/// 
/// # Returns
/// Weight in range (0, PRECISION], representing (0, 1].
fn compute_velocity_weight(rank_in_period: Field) -> Field {
    if rank_in_period <= VELOCITY_ALLOWANCE {
        // Within allowance: full weight
        PRECISION
    } else {
        // Beyond allowance: diminishing weight
        // ν = 1 / (1 + k × (rank - N))
        // With k = 0.5: ν = 1 / (1 + 0.5 × excess)
        
        let excess = rank_in_period - VELOCITY_ALLOWANCE;
        
        // k × excess, where k is scaled by PRECISION
        let penalty_term = fp_mul(VELOCITY_DECAY_RATE, excess * PRECISION);
        
        // 1 + penalty_term (in fixed point: PRECISION + penalty_term)
        let denominator = PRECISION + penalty_term;
        
        // 1 / denominator (in fixed point)
        fp_div(PRECISION, denominator)
    }
}

/// Computes the rank of a contract within its velocity period.
/// 
/// Counts how many contracts in the same history were completed
/// within the velocity period before (and including) this one.
/// 
/// # Arguments
/// * `history` - The agent's contract history
/// * `contract_idx` - Index of the contract to rank
/// * `skill_type` - Skill type to filter by
/// * `current_time` - Current timestamp for period calculation
/// 
/// # Returns
/// Rank within the period (1 = first contract, 2 = second, etc.)
fn compute_contract_rank(
    history: EnhancedAgentHistory,
    contract_idx: u32,
    skill_type: SkillType,
    current_time: Field,
) -> Field {
    let target_contract = history.contracts[contract_idx];
    let period_start = if current_time > VELOCITY_PERIOD_DAYS * 86400 {
        current_time - (VELOCITY_PERIOD_DAYS * 86400)
    } else {
        0
    };
    
    let mut rank: Field = 0;
    
    for i in 0..MAX_HISTORY {
        let contract = history.contracts[i];
        
        if contract.is_active() & contract.is_skill(skill_type) {
            // Count if in period and completed at or before target
            if contract.completed_at >= period_start {
                if contract.completed_at <= target_contract.completed_at {
                    rank = rank + 1;
                }
            }
        }
    }
    
    rank
}

/// Computes velocity weight for a contract given its history context.
/// 
/// Combines rank computation with weight calculation.
/// 
/// # Arguments
/// * `history` - The agent's contract history
/// * `contract_idx` - Index of the contract
/// * `skill_type` - Skill type
/// * `current_time` - Current timestamp
/// 
/// # Returns
/// Velocity weight in range (0, PRECISION].
fn compute_velocity_weight_for_contract(
    history: EnhancedAgentHistory,
    contract_idx: u32,
    skill_type: SkillType,
    current_time: Field,
) -> Field {
    let rank = compute_contract_rank(history, contract_idx, skill_type, current_time);
    compute_velocity_weight(rank)
}
```

---

## Outcome Variance Constraint

### Mathematical Notation

$$\text{plausible}(h) \iff |h| < N_{\min} \lor \text{var}(\text{outcomes}(h)) \geq \varepsilon_{\min}$$

### Noir Implementation

```noir
// ============================================
// OUTCOME VARIANCE CHECK
// ============================================

/// Computes the variance of outcomes in a history.
/// 
/// var = Σ(outcome - mean)² / n
/// 
/// Note: This is computationally expensive in ZK. For large histories,
/// consider using the simplified constraint `prove_outcome_diversity()`.
/// 
/// # Arguments
/// * `history` - The agent's contract history
/// * `skill_type` - Skill type to filter by
/// 
/// # Returns
/// Variance scaled by PRECISION, or 0 if history is empty.
fn compute_outcome_variance(
    history: EnhancedAgentHistory,
    skill_type: SkillType,
) -> Field {
    // First pass: compute count and sum for mean
    let mut count: Field = 0;
    let mut sum: Field = 0;
    
    for i in 0..MAX_HISTORY {
        let contract = history.contracts[i];
        if contract.is_active() & contract.is_skill(skill_type) {
            count = count + 1;
            // outcome_offset is in [0, 200], center at 100
            sum = sum + contract.outcome_offset;
        }
    }
    
    if count == 0 {
        return 0;
    }
    
    // Mean in same scale as outcome_offset (centered at 100)
    let mean = sum / count;
    
    // Second pass: compute sum of squared deviations
    let mut sum_squared_dev: Field = 0;
    
    for i in 0..MAX_HISTORY {
        let contract = history.contracts[i];
        if contract.is_active() & contract.is_skill(skill_type) {
            // Compute |outcome - mean|
            let deviation = if contract.outcome_offset >= mean {
                contract.outcome_offset - mean
            } else {
                mean - contract.outcome_offset
            };
            
            // Square the deviation
            sum_squared_dev = sum_squared_dev + (deviation * deviation);
        }
    }
    
    // Variance = sum_squared_dev / count
    // Scale to PRECISION for consistency
    // Max deviation is 100, max squared is 10000, so scale by PRECISION/10000
    (sum_squared_dev * PRECISION) / (count * 10000)
}

/// ZK-friendly plausibility check: proves outcome diversity.
/// 
/// Instead of computing exact variance, proves that at least M contracts
/// have outcome below a threshold (not perfect +1.0 scores).
/// 
/// # Arguments
/// * `history` - The agent's contract history
/// * `skill_type` - Skill type to filter by
/// * `required_imperfect` - Minimum contracts with outcome < 180 (i.e., < +0.8)
/// 
/// # Returns
/// True if enough contracts are "imperfect" to demonstrate realism.
fn prove_outcome_diversity(
    history: EnhancedAgentHistory,
    skill_type: SkillType,
    required_imperfect: Field,
) -> bool {
    let mut imperfect_count: Field = 0;
    
    for i in 0..MAX_HISTORY {
        let contract = history.contracts[i];
        if contract.is_active() & contract.is_skill(skill_type) {
            // Outcome < +80 (offset < 180) counts as "imperfect"
            if contract.outcome_offset < 180 {
                imperfect_count = imperfect_count + 1;
            }
        }
    }
    
    imperfect_count >= required_imperfect
}

/// Proves that a history is plausible (not suspiciously uniform).
/// 
/// plausible(h) ≡ (|h| < N_min) ∨ (var(outcomes) ≥ ε_min)
/// 
/// # Arguments
/// * `history` - The agent's contract history
/// * `skill_type` - Skill type to filter by
/// 
/// # Returns
/// True if history is plausible.
fn prove_plausible_history(
    history: EnhancedAgentHistory,
    skill_type: SkillType,
) -> bool {
    // Count matching contracts
    let mut count: Field = 0;
    for i in 0..MAX_HISTORY {
        let contract = history.contracts[i];
        if contract.is_active() & contract.is_skill(skill_type) {
            count = count + 1;
        }
    }
    
    // Small histories are exempt
    if count < VARIANCE_EXEMPTION_SIZE {
        return true;
    }
    
    // For larger histories, check variance
    let variance = compute_outcome_variance(history, skill_type);
    variance >= MINIMUM_VARIANCE
}

/// Alternative plausibility check using diversity instead of variance.
/// 
/// Requires that at least 20% of contracts (above exemption threshold)
/// have non-perfect outcomes.
/// 
/// # Arguments
/// * `history` - The agent's contract history
/// * `skill_type` - Skill type to filter by
/// 
/// # Returns
/// True if history demonstrates realistic diversity.
fn prove_plausible_history_simple(
    history: EnhancedAgentHistory,
    skill_type: SkillType,
) -> bool {
    // Count matching contracts
    let mut count: Field = 0;
    for i in 0..MAX_HISTORY {
        let contract = history.contracts[i];
        if contract.is_active() & contract.is_skill(skill_type) {
            count = count + 1;
        }
    }
    
    // Small histories are exempt
    if count < VARIANCE_EXEMPTION_SIZE {
        return true;
    }
    
    // Require at least 20% imperfect
    // required = (count - VARIANCE_EXEMPTION_SIZE) / 5
    let excess = count - VARIANCE_EXEMPTION_SIZE;
    let required = excess / 5;
    
    prove_outcome_diversity(history, skill_type, required)
}
```

---

## Escrow Verification

### Mathematical Notation

$$\text{valid\_escrow}(c) \iff \text{verify}(\varepsilon, s) \land \text{locked}(\varepsilon) \land \text{owner}(\varepsilon) \in \{a_{\text{provider}}, a_{\text{consumer}}\}$$

### Noir Implementation

```noir
// ============================================
// ESCROW VERIFICATION
// ============================================

/// Escrow note data (would be an Aztec private note in production).
/// 
/// Represents locked funds backing a contract stake.
struct EscrowNote {
    /// The amount of tokens locked.
    amount: Field,
    
    /// Owner of the escrow (provider or consumer).
    owner: AgentId,
    
    /// Hash of the note for commitment verification.
    note_hash: Field,
    
    /// Nullifier to prevent double-spending.
    nullifier: Field,
    
    /// Whether the escrow is currently locked.
    is_locked: bool,
}

/// Verifies that an escrow commitment is valid for a contract.
/// 
/// Checks:
/// 1. The commitment matches the escrow note hash
/// 2. The escrow amount matches the contract stake
/// 3. The escrow is currently locked
/// 4. The escrow owner is one of the contract parties
/// 
/// # Arguments
/// * `contract` - The contract to verify escrow for
/// * `escrow` - The escrow note
/// * `provider` - The contract provider's ID
/// 
/// # Returns
/// True if the escrow is valid for this contract.
fn verify_escrow(
    contract: EnhancedContract,
    escrow: EscrowNote,
    provider: AgentId,
) -> bool {
    // 1. Commitment matches note hash
    let commitment_valid = contract.escrow_commitment == escrow.note_hash;
    
    // 2. Amount matches stake
    let amount_valid = escrow.amount >= contract.stake;
    
    // 3. Escrow is locked
    let is_locked = escrow.is_locked;
    
    // 4. Owner is provider or consumer
    let owner_is_provider = escrow.owner.eq(provider);
    let owner_is_consumer = escrow.owner.eq(contract.counterparty);
    let owner_valid = owner_is_provider | owner_is_consumer;
    
    commitment_valid & amount_valid & is_locked & owner_valid
}

/// Proves that all contracts in a history have valid escrow.
/// 
/// This would typically be verified at contract creation time,
/// but can be re-verified in batch for auditing.
/// 
/// # Arguments
/// * `history` - The agent's contract history
/// * `escrows` - Array of escrow notes corresponding to contracts
/// * `skill_type` - Skill type to filter by
/// 
/// # Returns
/// True if all matching contracts have valid escrow.
fn verify_all_escrows(
    history: EnhancedAgentHistory,
    escrows: [EscrowNote; MAX_HISTORY],
    skill_type: SkillType,
) -> bool {
    let mut all_valid = true;
    
    for i in 0..MAX_HISTORY {
        let contract = history.contracts[i];
        if contract.is_active() & contract.is_skill(skill_type) {
            let escrow_valid = verify_escrow(
                contract,
                escrows[i],
                history.agent_id
            );
            all_valid = all_valid & escrow_valid;
        }
    }
    
    all_valid
}
```

---

## Enhanced Trust Calculation

### Mathematical Notation

$$V_t(\text{Agent}(t, h_t)) = \sum_{c \in h_t} \omega(c) \cdot \text{outcome}(c) \cdot \gamma(c) \cdot \nu(c)$$

### Noir Implementation

```noir
// ============================================
// ENHANCED TRUST CALCULATION
// ============================================

/// Computes trust value with all Sybil resistance factors.
/// 
/// V_t = Σ ω(c) × outcome(c) × γ(c) × ν(c)
/// 
/// Where:
/// - ω(c) = base weight (stake, difficulty, recency)
/// - outcome(c) = contract result [-1, +1]
/// - γ(c) = counterparty trust factor [0, 1]
/// - ν(c) = velocity weight [0, 1]
/// 
/// # Arguments
/// * `history` - The agent's contract history
/// * `skill_type` - The skill type to compute trust for
/// * `current_time` - Current timestamp for velocity calculation
/// 
/// # Returns
/// Signed trust value scaled by PRECISION.
fn compute_trust_value_enhanced(
    history: EnhancedAgentHistory,
    skill_type: SkillType,
    current_time: Field,
) -> Signed {
    let mut trust = Signed::zero();
    
    for i in 0..MAX_HISTORY {
        let contract = history.contracts[i];
        
        if contract.is_active() & contract.is_skill(skill_type) {
            // Validate contract bounds
            assert(contract.is_valid());
            assert(verify_weight_bounds_enhanced(contract));
            
            // Base contribution: ω × outcome
            let outcome = contract.outcome();
            let base_magnitude = (contract.weight * outcome.magnitude) / 100;
            let base_contribution = Signed::new(base_magnitude, outcome.is_negative);
            
            // Counterparty factor: γ(c)
            let counterparty_trust = contract.counterparty_trust();
            let gamma = compute_counterparty_factor(counterparty_trust);
            
            // Velocity factor: ν(c)
            let nu = compute_velocity_weight_for_contract(
                history, i as u32, skill_type, current_time
            );
            
            // Final contribution: base × γ × ν
            // All factors are scaled by PRECISION, so we divide twice
            let adjusted_magnitude = fp_mul(fp_mul(base_contribution.magnitude, gamma), nu);
            let final_contribution = Signed::new(adjusted_magnitude, base_contribution.is_negative);
            
            trust = trust.add(final_contribution);
        }
    }
    
    trust
}

/// Validates weight bounds for enhanced contracts.
/// 
/// Same as base implementation but works with EnhancedContract.
fn verify_weight_bounds_enhanced(contract: EnhancedContract) -> bool {
    let min_weight: Field = 1;
    let log_stake = approx_log1p(contract.stake);
    let max_theoretical = fp_mul(log_stake, 3 * PRECISION);
    let min_max_weight = 3 * PRECISION;
    let computed_max = max_theoretical + PRECISION;
    let max_weight = if computed_max > min_max_weight {
        computed_max
    } else {
        min_max_weight
    };
    (contract.weight >= min_weight) & (contract.weight <= max_weight)
}
```

---

## Combined Eligibility Proof

### Mathematical Notation

$$\text{eligible}(a, c) \iff V_t(a) \geq \theta(c) \land \text{plausible}(h_t(a))$$

### Noir Implementation

```noir
// ============================================
// COMBINED ELIGIBILITY PROOF
// ============================================

/// The primary eligibility circuit with all Sybil resistance checks.
/// 
/// Proves:
/// 1. Trust meets threshold: V_t ≥ θ(c)
/// 2. History is plausible: variance check passes
/// 3. History has sufficient size (basic Sybil check)
/// 4. History has sufficient depth (basic Sybil check)
/// 5. History has sufficient counterparty diversity (basic Sybil check)
/// 
/// The verifier learns only: "This agent is eligible" (true/false)
/// The verifier does NOT learn: exact trust, contract details, counterparties
/// 
/// # Arguments
/// * `skill_type` - The skill type for this contract (public)
/// * `contract_stake` - Stake amount for the contract (public)
/// * `contract_difficulty` - Difficulty rating (public)
/// * `min_history_size` - Required minimum contracts (public)
/// * `min_history_depth` - Required minimum days of history (public)
/// * `min_counterparty_diversity` - Required unique counterparties (public)
/// * `current_time` - Current timestamp (public)
/// * `history` - The agent's contract history (private)
/// 
/// # Returns
/// True if all eligibility requirements are met.
fn prove_eligibility_enhanced(
    skill_type: pub Field,
    contract_stake: pub Field,
    contract_difficulty: pub Field,
    min_history_size: pub Field,
    min_history_depth: pub Field,
    min_counterparty_diversity: pub Field,
    current_time: pub Field,
    history: EnhancedAgentHistory,
) -> pub bool {
    let skill = SkillType::new(skill_type);
    
    // 1. Compute trust with all factors
    let trust = compute_trust_value_enhanced(history, skill, current_time);
    
    // 2. Compute threshold
    let threshold = calculate_threshold(contract_stake, contract_difficulty);
    
    // 3. Check trust meets threshold
    let meets_threshold = trust.gte(Signed::from_positive(threshold));
    
    // 4. Check history is plausible (variance)
    let is_plausible = prove_plausible_history_simple(history, skill);
    
    // 5. Basic Sybil checks (from base implementation)
    let meets_size = prove_history_size_enhanced(skill_type, min_history_size, history);
    let meets_depth = prove_history_depth_enhanced(skill_type, min_history_depth, current_time, history);
    let meets_diversity = prove_counterparty_diversity_enhanced(skill_type, min_counterparty_diversity, history);
    
    // All checks must pass
    meets_threshold & is_plausible & meets_size & meets_depth & meets_diversity
}

/// Proves minimum history size (enhanced version).
fn prove_history_size_enhanced(
    skill_type: pub Field,
    minimum: pub Field,
    history: EnhancedAgentHistory,
) -> pub bool {
    let skill = SkillType::new(skill_type);
    let mut count: Field = 0;
    
    for i in 0..MAX_HISTORY {
        let contract = history.contracts[i];
        if contract.is_active() & contract.is_skill(skill) {
            count = count + 1;
        }
    }
    
    count >= minimum
}

/// Proves minimum history depth (enhanced version).
fn prove_history_depth_enhanced(
    skill_type: pub Field,
    min_depth_days: pub Field,
    current_time: pub Field,
    history: EnhancedAgentHistory,
) -> pub bool {
    let skill = SkillType::new(skill_type);
    let required_age = min_depth_days * 86400;  // Convert days to seconds
    let mut oldest_contract: Field = current_time;
    let mut found_any = false;
    
    for i in 0..MAX_HISTORY {
        let contract = history.contracts[i];
        if contract.is_active() & contract.is_skill(skill) {
            found_any = true;
            if contract.completed_at < oldest_contract {
                oldest_contract = contract.completed_at;
            }
        }
    }
    
    if !found_any {
        false
    } else {
        (current_time - oldest_contract) >= required_age
    }
}

/// Proves minimum counterparty diversity (enhanced version).
fn prove_counterparty_diversity_enhanced(
    skill_type: pub Field,
    minimum_unique: pub Field,
    history: EnhancedAgentHistory,
) -> pub bool {
    let skill = SkillType::new(skill_type);
    
    // Collect counterparty IDs
    let mut counterparties: [Field; MAX_HISTORY] = [0; MAX_HISTORY];
    let mut cp_count: u32 = 0;
    
    for i in 0..MAX_HISTORY {
        let contract = history.contracts[i];
        if contract.is_active() & contract.is_skill(skill) {
            counterparties[cp_count] = contract.counterparty.inner;
            cp_count = cp_count + 1;
        }
    }
    
    // Count unique counterparties (O(n²) but bounded)
    let mut unique_count: Field = 0;
    
    for i in 0..MAX_HISTORY {
        if (i as Field) < (cp_count as Field) {
            let mut is_unique = true;
            
            // Check if this counterparty appeared earlier
            for j in 0..MAX_HISTORY {
                if (j as u32) < i {
                    if counterparties[j] == counterparties[i] {
                        is_unique = false;
                    }
                }
            }
            
            if is_unique {
                unique_count = unique_count + 1;
            }
        }
    }
    
    unique_count >= minimum_unique
}
```

---

## Integration Notes

### Aztec-Specific Considerations

When integrating with Aztec's full stack:

1. **Escrow as Private Notes**: Each escrow becomes a private note. The `escrow_commitment` field would be the note hash. Nullifiers prevent double-spending.

2. **Counterparty Trust Snapshots**: When a contract completes, query the counterparty's current trust (via a public function or oracle) and store the snapshot in the contract note.

3. **Velocity Periods**: Use block numbers instead of timestamps for more predictable period boundaries. Convert `VELOCITY_PERIOD_DAYS` to blocks.

4. **Recursive Proofs for Batch Verification**: For histories with many contracts, consider recursive proof composition to manage circuit size.

### Gas Optimization

The enhanced eligibility proof is more expensive than the base version due to:

- Sigmoid computation for each contract
- Velocity rank computation (O(n) per contract, O(n²) total)
- Variance computation (two passes over history)

For production, consider:

1. **Pre-computing ranks**: Store velocity ranks in contract notes at creation time
2. **Simplified variance**: Use the `prove_plausible_history_simple()` variant
3. **Batched verification**: Verify escrows separately from eligibility

### Parameter Governance

The parameters in this implementation are compile-time constants. For a production system:

1. **Make parameters public inputs**: Allow contract posters to specify stricter requirements
2. **Protocol-level minimums**: Enforce minimum values that can't be relaxed
3. **Upgrade path**: Use proxy patterns to update parameter defaults

---

## Summary of New Circuits

| Circuit | Public Inputs | Private Inputs | Output |
|---------|---------------|----------------|--------|
| `compute_counterparty_factor` | — | counterparty_trust | γ value |
| `compute_velocity_weight` | rank | — | ν value |
| `prove_plausible_history` | skill_type | history | bool |
| `verify_escrow` | — | contract, escrow, provider | bool |
| `compute_trust_value_enhanced` | skill_type, current_time | history | trust value |
| `prove_eligibility_enhanced` | many (see above) | history | bool |

---

*This document extends the base Quantum of Trust Noir implementation with Sybil resistance mechanisms.*
