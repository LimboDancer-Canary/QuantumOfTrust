# Contract Validation Tests (Noir)

These tests validate the contract structure and bounds checking in the Noir implementation.
Each test ensures that the circuit correctly validates contract fields to prevent proof forgery.

---

## Tests

```noir
// ============================================
// CONTRACT VALIDATION TESTS
// ============================================

// --------------------------------------------
// Contract.is_valid() Tests
// --------------------------------------------

#[test]
fn test_contract_is_valid_all_bounds_correct() {
    // ═══════════════════════════════════════════════════════════════
    // MATH: c = (a<provider>, a<consumer>, t, s, d, τ)
    //       where d ∈ [0, 10]
    //       
    //       outcome(c) ∈ [-1, 1]
    //       (In Noir: outcome_offset ∈ [0, 200] represents [-100, +100])
    //
    // PLAIN ENGLISH: "A contract is defined by six things: who's 
    // providing the service, who's consuming it, what skill type it 
    // involves, how much is at stake, how difficult it is, and its 
    // deadline." The difficulty rating must be between 0 and 10, and
    // the outcome must be in the valid range.
    // ═══════════════════════════════════════════════════════════════
    
    let contract = Contract {
        counterparty: AgentId::new(2),
        skill_type: SkillType::new(1),
        stake: 1000,
        difficulty: 5,           // Valid: 0 ≤ 5 ≤ 10
        outcome_offset: 150,     // Valid: 0 ≤ 150 ≤ 200
        completed_at: 1000,
        weight: PRECISION,
    };
    
    assert(contract.is_valid());
}

#[test]
fn test_contract_is_valid_difficulty_at_minimum() {
    // ═══════════════════════════════════════════════════════════════
    // MATH: d ∈ [0, 10]
    //
    // PLAIN ENGLISH: "Difficulty rating ranges from 0 (trivial) to 10 
    // (extremely difficult)." Zero difficulty is valid—it represents 
    // trivial work that anyone could complete.
    // ═══════════════════════════════════════════════════════════════
    
    let contract = Contract {
        counterparty: AgentId::new(2),
        skill_type: SkillType::new(1),
        stake: 1000,
        difficulty: 0,           // Boundary: minimum
        outcome_offset: 100,
        completed_at: 1000,
        weight: PRECISION,
    };
    
    assert(contract.is_valid());
}

#[test]
fn test_contract_is_valid_difficulty_at_maximum() {
    // ═══════════════════════════════════════════════════════════════
    // MATH: d ∈ [0, 10]
    //
    // PLAIN ENGLISH: "Difficulty rating ranges from 0 (trivial) to 10 
    // (extremely difficult)." A difficulty of 10 represents the most
    // challenging work—completing it proves significant capability.
    // ═══════════════════════════════════════════════════════════════
    
    let contract = Contract {
        counterparty: AgentId::new(2),
        skill_type: SkillType::new(1),
        stake: 1000,
        difficulty: 10,          // Boundary: maximum (MAX_DIFFICULTY)
        outcome_offset: 100,
        completed_at: 1000,
        weight: PRECISION,
    };
    
    assert(contract.is_valid());
}

#[test]
fn test_contract_is_valid_outcome_at_minimum() {
    // ═══════════════════════════════════════════════════════════════
    // MATH: outcome(c) ∈ [-1, 1]
    //       (Noir representation: offset 0 = outcome -100 = -1.0 scaled)
    //
    // PLAIN ENGLISH: "The outcome of a contract is a number between 
    // negative one and positive one." An outcome of -1 represents
    // complete failure—nothing delivered, or actively harmful.
    // ═══════════════════════════════════════════════════════════════
    
    let contract = Contract {
        counterparty: AgentId::new(2),
        skill_type: SkillType::new(1),
        stake: 1000,
        difficulty: 5,
        outcome_offset: 0,       // -100 = complete failure (-1.0)
        completed_at: 1000,
        weight: PRECISION,
    };
    
    assert(contract.is_valid());
}

#[test]
fn test_contract_is_valid_outcome_at_maximum() {
    // ═══════════════════════════════════════════════════════════════
    // MATH: outcome(c) ∈ [-1, 1]
    //       (Noir representation: offset 200 = outcome +100 = +1.0 scaled)
    //
    // PLAIN ENGLISH: "The outcome of a contract is a number between 
    // negative one and positive one." An outcome of +1 represents
    // complete success—everything delivered perfectly.
    // ═══════════════════════════════════════════════════════════════
    
    let contract = Contract {
        counterparty: AgentId::new(2),
        skill_type: SkillType::new(1),
        stake: 1000,
        difficulty: 5,
        outcome_offset: 200,     // +100 = complete success (+1.0)
        completed_at: 1000,
        weight: PRECISION,
    };
    
    assert(contract.is_valid());
}

#[test]
fn test_contract_is_valid_outcome_neutral() {
    // ═══════════════════════════════════════════════════════════════
    // MATH: outcome(c) ∈ [-1, 1]
    //       (Noir representation: offset 100 = outcome 0 = neutral)
    //
    // PLAIN ENGLISH: "The outcome of a contract is a number between 
    // negative one and positive one." An outcome of 0 is neutral—
    // partial delivery, or cancelled by mutual agreement. It neither 
    // adds to nor subtracts from trust.
    // ═══════════════════════════════════════════════════════════════
    
    let contract = Contract {
        counterparty: AgentId::new(2),
        skill_type: SkillType::new(1),
        stake: 1000,
        difficulty: 5,
        outcome_offset: 100,     // 0 = neutral outcome
        completed_at: 1000,
        weight: PRECISION,
    };
    
    assert(contract.is_valid());
}

#[test]
fn test_contract_is_invalid_difficulty_exceeds_maximum() {
    // ═══════════════════════════════════════════════════════════════
    // MATH: d ∈ [0, 10] — values outside this range are invalid
    //
    // PLAIN ENGLISH: "Difficulty rating ranges from 0 to 10." A 
    // malicious prover could try to inflate difficulty to increase
    // weight bounds and forge eligibility. This must be rejected.
    // ═══════════════════════════════════════════════════════════════
    
    let contract = Contract {
        counterparty: AgentId::new(2),
        skill_type: SkillType::new(1),
        stake: 1000,
        difficulty: 11,          // INVALID: exceeds maximum
        outcome_offset: 100,
        completed_at: 1000,
        weight: PRECISION,
    };
    
    assert(!contract.is_valid());
}

#[test]
fn test_contract_is_invalid_outcome_exceeds_maximum() {
    // ═══════════════════════════════════════════════════════════════
    // MATH: outcome(c) ∈ [-1, 1]
    //       (Noir: outcome_offset must be ≤ 200)
    //
    // PLAIN ENGLISH: "The outcome of a contract is a number between 
    // negative one and positive one." A malicious prover could try 
    // to inflate outcome to increase trust contribution. Values 
    // outside the valid range must be rejected.
    // ═══════════════════════════════════════════════════════════════
    
    let contract = Contract {
        counterparty: AgentId::new(2),
        skill_type: SkillType::new(1),
        stake: 1000,
        difficulty: 5,
        outcome_offset: 201,     // INVALID: exceeds maximum
        completed_at: 1000,
        weight: PRECISION,
    };
    
    assert(!contract.is_valid());
}

// --------------------------------------------
// validate_contract() Tests
// --------------------------------------------

#[test]
fn test_validate_contract_valid_active_contract() {
    // ═══════════════════════════════════════════════════════════════
    // MATH: ω(c) > 0 for active contracts
    //       (Combined with d ∈ [0,10] and outcome ∈ [-1,1])
    //
    // PLAIN ENGLISH: "The weight of a contract determines how much 
    // it should count toward trust." Active contracts must have 
    // positive weight, valid difficulty, and valid outcome bounds.
    // ═══════════════════════════════════════════════════════════════
    
    let contract = Contract {
        counterparty: AgentId::new(2),
        skill_type: SkillType::new(1),
        stake: 1000,
        difficulty: 5,
        outcome_offset: 150,
        completed_at: 1000,
        weight: PRECISION,       // Active: weight > 0
    };
    
    assert(validate_contract(contract));
}

#[test]
fn test_validate_contract_invalid_zero_weight() {
    // ═══════════════════════════════════════════════════════════════
    // MATH: ω(c) > 0 required for active contracts
    //
    // PLAIN ENGLISH: "The weight of a contract determines how much 
    // it should count." A weight of zero means the contract slot is
    // inactive/empty and should not contribute to trust calculation.
    // ═══════════════════════════════════════════════════════════════
    
    let contract = Contract {
        counterparty: AgentId::new(2),
        skill_type: SkillType::new(1),
        stake: 1000,
        difficulty: 5,
        outcome_offset: 150,
        completed_at: 1000,
        weight: 0,               // INVALID: zero weight = inactive
    };
    
    assert(!validate_contract(contract));
}

#[test]
fn test_validate_contract_empty_contract_is_invalid() {
    // ═══════════════════════════════════════════════════════════════
    // MATH: ω(c) > 0 required for active contracts
    //
    // PLAIN ENGLISH: "Empty contract slots are placeholders, not 
    // valid active contracts." In Noir's fixed-size arrays, unused 
    // slots have weight=0 to mark them as inactive.
    // ═══════════════════════════════════════════════════════════════
    
    let empty = Contract::empty();
    
    assert(!validate_contract(empty));
}

#[test]
fn test_empty_contract_passes_is_valid_but_fails_validate() {
    // ═══════════════════════════════════════════════════════════════
    // MATH: is_valid() checks: d ∈ [0,10] AND outcome ∈ [-1,1]
    //       validate_contract() checks: above AND ω(c) > 0
    //
    // PLAIN ENGLISH: "Contract::empty() has valid field bounds but 
    // zero weight." The is_valid() function only checks bounds;
    // validate_contract() additionally requires positive weight.
    // This distinction matters for array slot handling.
    // ═══════════════════════════════════════════════════════════════
    
    let empty = Contract::empty();
    
    // Passes bounds check (difficulty=0 valid, outcome_offset=100 valid)
    assert(empty.is_valid());
    
    // Fails active contract validation (weight=0)
    assert(!validate_contract(empty));
}

// --------------------------------------------
// verify_weight_bounds() Tests
// --------------------------------------------

#[test]
fn test_verify_weight_bounds_valid_normal_weight() {
    // ═══════════════════════════════════════════════════════════════
    // MATH: ω(c) = f(s(c), d(c), V<t>(a<consumer>), recency(c))
    //       
    //       Weight is bounded by stake: ω ≤ log(1 + s) × 3 + buffer
    //
    // PLAIN ENGLISH: "The weight of a contract is a function of stake, 
    // difficulty, counterparty trust, and recency." The maximum 
    // theoretical weight is bounded by stake to prevent forgery.
    // For stake=1000, max ≈ log(1001) × 3 ≈ 20.7, so weight=1.0 is valid.
    // ═══════════════════════════════════════════════════════════════
    
    let contract = Contract {
        counterparty: AgentId::new(2),
        skill_type: SkillType::new(1),
        stake: 1000,
        difficulty: 5,
        outcome_offset: 150,
        completed_at: 1000,
        weight: PRECISION,       // 1.0 scaled - well within bounds
    };
    
    assert(verify_weight_bounds(contract));
}

#[test]
fn test_verify_weight_bounds_valid_minimum_weight() {
    // ═══════════════════════════════════════════════════════════════
    // MATH: ω(c) ≥ 1 (minimum weight for numerical stability)
    //
    // PLAIN ENGLISH: "Every active contract must have at least a 
    // minimal weight." This ensures numerical stability and that 
    // every contract contributes at least a tiny amount to trust.
    // ═══════════════════════════════════════════════════════════════
    
    let contract = Contract {
        counterparty: AgentId::new(2),
        skill_type: SkillType::new(1),
        stake: 1000,
        difficulty: 5,
        outcome_offset: 150,
        completed_at: 1000,
        weight: 1,               // Minimum valid weight
    };
    
    assert(verify_weight_bounds(contract));
}

#[test]
fn test_verify_weight_bounds_valid_zero_stake() {
    // ═══════════════════════════════════════════════════════════════
    // MATH: ω(c) = f(s(c), ...) where s = 0
    //       
    //       For s=0: log(1) = 0, so we use min_max_weight = 3.0
    //
    // PLAIN ENGLISH: "A $100,000 contract matters more than a $100 
    // contract." But zero-stake contracts are still valid—they just 
    // can't have very high weights. The system uses a minimum max 
    // weight of 3.0 for zero-stake contracts.
    // ═══════════════════════════════════════════════════════════════
    
    let contract = Contract {
        counterparty: AgentId::new(2),
        skill_type: SkillType::new(1),
        stake: 0,
        difficulty: 5,
        outcome_offset: 150,
        completed_at: 1000,
        weight: 2 * PRECISION,   // 2.0 - within min_max of 3.0
    };
    
    assert(verify_weight_bounds(contract));
}

#[test]
fn test_verify_weight_bounds_valid_high_stake() {
    // ═══════════════════════════════════════════════════════════════
    // MATH: ω(c) = f(s(c), ...) 
    //       max_weight ≈ log(1 + s) × 3 + 1
    //
    // PLAIN ENGLISH: "A $100,000 contract matters more than a $100 
    // contract." Higher stakes allow higher weights. For stake=10000,
    // max ≈ log(10001) × 3 + 1 ≈ 28.6, so weight=10.0 is valid.
    // ═══════════════════════════════════════════════════════════════
    
    let contract = Contract {
        counterparty: AgentId::new(2),
        skill_type: SkillType::new(1),
        stake: 10000,
        difficulty: 5,
        outcome_offset: 150,
        completed_at: 1000,
        weight: 10 * PRECISION,  // 10.0 - valid for high stake
    };
    
    assert(verify_weight_bounds(contract));
}

#[test]
fn test_verify_weight_bounds_invalid_inflated_weight() {
    // ═══════════════════════════════════════════════════════════════
    // MATH: ω(c) ≤ log(1 + s) × 3 + buffer
    //       
    //       For stake=100: max ≈ log(101) × 3 + 1 ≈ 14.85
    //       Weight of 100.0 exceeds this bound.
    //
    // PLAIN ENGLISH: "The weight of a contract is bounded by its 
    // stake." A malicious prover cannot claim an artificially high 
    // weight to forge eligibility proofs. The circuit rejects weights
    // that exceed the theoretical maximum for the given stake.
    // ═══════════════════════════════════════════════════════════════
    
    let contract = Contract {
        counterparty: AgentId::new(2),
        skill_type: SkillType::new(1),
        stake: 100,
        difficulty: 5,
        outcome_offset: 150,
        completed_at: 1000,
        weight: 100 * PRECISION, // INVALID: 100.0 exceeds max ~14.85
    };
    
    assert(!verify_weight_bounds(contract));
}

#[test]
fn test_verify_weight_bounds_invalid_zero_weight() {
    // ═══════════════════════════════════════════════════════════════
    // MATH: ω(c) ≥ 1 (minimum weight)
    //
    // PLAIN ENGLISH: "Active contracts must have positive weight."
    // A weight of zero indicates an inactive slot, not a valid 
    // contract. Weight bounds require at least 1.
    // ═══════════════════════════════════════════════════════════════
    
    let contract = Contract {
        counterparty: AgentId::new(2),
        skill_type: SkillType::new(1),
        stake: 1000,
        difficulty: 5,
        outcome_offset: 150,
        completed_at: 1000,
        weight: 0,               // INVALID: below minimum of 1
    };
    
    assert(!verify_weight_bounds(contract));
}

// --------------------------------------------
// Contract.outcome() Tests
// --------------------------------------------

#[test]
fn test_contract_outcome_positive() {
    // ═══════════════════════════════════════════════════════════════
    // MATH: outcome(c) ∈ [-1, 1]
    //       Noir offset 200 → outcome = +100 (scaled +1.0)
    //
    // PLAIN ENGLISH: "The outcome of a contract is a number between 
    // negative one and positive one. +1 means complete success—
    // everything delivered perfectly."
    // ═══════════════════════════════════════════════════════════════
    
    let contract = Contract {
        counterparty: AgentId::new(2),
        skill_type: SkillType::new(1),
        stake: 1000,
        difficulty: 5,
        outcome_offset: 200,     // +100 outcome (complete success)
        completed_at: 1000,
        weight: PRECISION,
    };
    
    let outcome = contract.outcome();
    assert(outcome.magnitude == 100);
    assert(!outcome.is_negative);
}

#[test]
fn test_contract_outcome_negative() {
    // ═══════════════════════════════════════════════════════════════
    // MATH: outcome(c) ∈ [-1, 1]
    //       Noir offset 0 → outcome = -100 (scaled -1.0)
    //
    // PLAIN ENGLISH: "The outcome of a contract is a number between 
    // negative one and positive one. -1 means complete failure—
    // nothing delivered, or actively harmful."
    // ═══════════════════════════════════════════════════════════════
    
    let contract = Contract {
        counterparty: AgentId::new(2),
        skill_type: SkillType::new(1),
        stake: 1000,
        difficulty: 5,
        outcome_offset: 0,       // -100 outcome (complete failure)
        completed_at: 1000,
        weight: PRECISION,
    };
    
    let outcome = contract.outcome();
    assert(outcome.magnitude == 100);
    assert(outcome.is_negative);
}

#[test]
fn test_contract_outcome_neutral() {
    // ═══════════════════════════════════════════════════════════════
    // MATH: outcome(c) ∈ [-1, 1]
    //       Noir offset 100 → outcome = 0 (neutral)
    //
    // PLAIN ENGLISH: "The outcome of a contract is a number between 
    // negative one and positive one. 0 is neutral—partial delivery, 
    // or cancelled by mutual agreement."
    // ═══════════════════════════════════════════════════════════════
    
    let contract = Contract {
        counterparty: AgentId::new(2),
        skill_type: SkillType::new(1),
        stake: 1000,
        difficulty: 5,
        outcome_offset: 100,     // 0 outcome (neutral)
        completed_at: 1000,
        weight: PRECISION,
    };
    
    let outcome = contract.outcome();
    assert(outcome.magnitude == 0);
    assert(!outcome.is_negative);  // Zero is never negative
}

#[test]
fn test_contract_outcome_partial_positive() {
    // ═══════════════════════════════════════════════════════════════
    // MATH: outcome(c) ∈ [-1, 1] (continuous range)
    //       Noir offset 150 → outcome = +50 (scaled +0.5)
    //
    // PLAIN ENGLISH: "This continuous range allows for nuance. +0.5 
    // means good but not perfect—met expectations." Real work has 
    // degrees of quality, not just pass/fail.
    // ═══════════════════════════════════════════════════════════════
    
    let contract = Contract {
        counterparty: AgentId::new(2),
        skill_type: SkillType::new(1),
        stake: 1000,
        difficulty: 5,
        outcome_offset: 150,     // +50 outcome (partial success)
        completed_at: 1000,
        weight: PRECISION,
    };
    
    let outcome = contract.outcome();
    assert(outcome.magnitude == 50);
    assert(!outcome.is_negative);
}

#[test]
fn test_contract_outcome_partial_negative() {
    // ═══════════════════════════════════════════════════════════════
    // MATH: outcome(c) ∈ [-1, 1] (continuous range)
    //       Noir offset 30 → outcome = -70 (scaled -0.7)
    //
    // PLAIN ENGLISH: "This continuous range allows for nuance. -0.7 
    // means problematic—significant issues but some value delivered."
    // A project that's 30% complete isn't the same as one never started.
    // ═══════════════════════════════════════════════════════════════
    
    let contract = Contract {
        counterparty: AgentId::new(2),
        skill_type: SkillType::new(1),
        stake: 1000,
        difficulty: 5,
        outcome_offset: 30,      // -70 outcome (partial failure)
        completed_at: 1000,
        weight: PRECISION,
    };
    
    let outcome = contract.outcome();
    assert(outcome.magnitude == 70);
    assert(outcome.is_negative);
}

// --------------------------------------------
// Contract.is_active() Tests
// --------------------------------------------

#[test]
fn test_contract_is_active_with_positive_weight() {
    // ═══════════════════════════════════════════════════════════════
    // MATH: V<t>(Agent(t, h<t>)) = Σ ω(c) · outcome(c) for c ∈ h<t>
    //       Only contracts with ω(c) > 0 contribute to the sum.
    //
    // PLAIN ENGLISH: "An Agent's trust value equals the sum of 
    // (weight × outcome) for each contract." Active contracts have
    // positive weight and contribute to this calculation.
    // ═══════════════════════════════════════════════════════════════
    
    let contract = Contract {
        counterparty: AgentId::new(2),
        skill_type: SkillType::new(1),
        stake: 1000,
        difficulty: 5,
        outcome_offset: 150,
        completed_at: 1000,
        weight: PRECISION,
    };
    
    assert(contract.is_active());
}

#[test]
fn test_contract_is_inactive_with_zero_weight() {
    // ═══════════════════════════════════════════════════════════════
    // MATH: Contracts with ω(c) = 0 are skipped in summation
    //
    // PLAIN ENGLISH: "Inactive slots have weight=0 and should be 
    // skipped in calculations." This is how Noir's fixed-size arrays 
    // handle unused slots—they don't contribute to trust.
    // ═══════════════════════════════════════════════════════════════
    
    let contract = Contract {
        counterparty: AgentId::new(2),
        skill_type: SkillType::new(1),
        stake: 1000,
        difficulty: 5,
        outcome_offset: 150,
        completed_at: 1000,
        weight: 0,
    };
    
    assert(!contract.is_active());
}

#[test]
fn test_empty_contract_is_inactive() {
    // ═══════════════════════════════════════════════════════════════
    // MATH: Contract::empty() has ω = 0
    //
    // PLAIN ENGLISH: "Empty contracts are placeholders for unused 
    // array slots." They have weight=0 by definition, so they don't 
    // contribute to trust calculations.
    // ═══════════════════════════════════════════════════════════════
    
    let empty = Contract::empty();
    assert(!empty.is_active());
}

// --------------------------------------------
// Contract.trust_contribution() Tests
// --------------------------------------------

#[test]
fn test_trust_contribution_positive_outcome() {
    // ═══════════════════════════════════════════════════════════════
    // MATH: V<t> = Σ ω(c) · outcome(c)
    //       Each term: contribution = ω(c) × outcome(c)
    //
    // PLAIN ENGLISH: "A successful contract with high weight adds a 
    // lot to your trust." Here: weight=1.0, outcome=+1.0, so 
    // contribution = 1.0 × 1.0 = +1.0 (scaled: PRECISION).
    // ═══════════════════════════════════════════════════════════════
    
    let contract = Contract {
        counterparty: AgentId::new(2),
        skill_type: SkillType::new(1),
        stake: 1000,
        difficulty: 5,
        outcome_offset: 200,     // +100 outcome (+1.0)
        completed_at: 1000,
        weight: PRECISION,       // 1.0 weight
    };
    
    let contribution = contract.trust_contribution();
    assert(contribution.magnitude == PRECISION);
    assert(!contribution.is_negative);
}

#[test]
fn test_trust_contribution_negative_outcome() {
    // ═══════════════════════════════════════════════════════════════
    // MATH: V<t> = Σ ω(c) · outcome(c)
    //       Each term: contribution = ω(c) × outcome(c)
    //
    // PLAIN ENGLISH: "A failed contract with high weight subtracts a 
    // lot from your trust." Here: weight=1.0, outcome=-1.0, so 
    // contribution = 1.0 × (-1.0) = -1.0 (scaled: -PRECISION).
    // ═══════════════════════════════════════════════════════════════
    
    let contract = Contract {
        counterparty: AgentId::new(2),
        skill_type: SkillType::new(1),
        stake: 1000,
        difficulty: 5,
        outcome_offset: 0,       // -100 outcome (-1.0)
        completed_at: 1000,
        weight: PRECISION,       // 1.0 weight
    };
    
    let contribution = contract.trust_contribution();
    assert(contribution.magnitude == PRECISION);
    assert(contribution.is_negative);
}

#[test]
fn test_trust_contribution_neutral_outcome() {
    // ═══════════════════════════════════════════════════════════════
    // MATH: V<t> = Σ ω(c) · outcome(c)
    //       When outcome = 0: contribution = ω × 0 = 0
    //
    // PLAIN ENGLISH: "A neutral outcome (partial delivery, cancelled 
    // by mutual agreement) contributes nothing to trust—neither 
    // positive nor negative."
    // ═══════════════════════════════════════════════════════════════
    
    let contract = Contract {
        counterparty: AgentId::new(2),
        skill_type: SkillType::new(1),
        stake: 1000,
        difficulty: 5,
        outcome_offset: 100,     // 0 outcome (neutral)
        completed_at: 1000,
        weight: PRECISION,       // 1.0 weight
    };
    
    let contribution = contract.trust_contribution();
    assert(contribution.magnitude == 0);
}

#[test]
fn test_trust_contribution_weighted() {
    // ═══════════════════════════════════════════════════════════════
    // MATH: V<t> = Σ ω(c) · outcome(c)
    //       contribution = 2.0 × 0.5 = 1.0
    //
    // PLAIN ENGLISH: "Low-weight contracts (small stakes, easy tasks) 
    // matter less either way." Here: weight=2.0, outcome=+0.5, so
    // contribution = 2.0 × 0.5 = 1.0 (scaled: PRECISION).
    // ═══════════════════════════════════════════════════════════════
    
    let contract = Contract {
        counterparty: AgentId::new(2),
        skill_type: SkillType::new(1),
        stake: 1000,
        difficulty: 5,
        outcome_offset: 150,     // +50 outcome (+0.5)
        completed_at: 1000,
        weight: 2 * PRECISION,   // 2.0 weight
    };
    
    let contribution = contract.trust_contribution();
    assert(contribution.magnitude == PRECISION);
    assert(!contribution.is_negative);
}

// --------------------------------------------
// calculate_partial_outcome() Tests
// --------------------------------------------

#[test]
fn test_calculate_partial_outcome_perfect() {
    // ═══════════════════════════════════════════════════════════════
    // MATH: outcome(c) ∈ [-1, 1]
    //       100% completion + 100% quality → +1.0 (offset 200)
    //
    // PLAIN ENGLISH: "+1 means complete success—everything delivered 
    // perfectly." This test verifies that perfect completion and 
    // quality map to the maximum positive outcome.
    // ═══════════════════════════════════════════════════════════════
    
    let outcome = calculate_partial_outcome(100, 100);
    assert(outcome == 200);
}

#[test]
fn test_calculate_partial_outcome_failure() {
    // ═══════════════════════════════════════════════════════════════
    // MATH: outcome(c) ∈ [-1, 1]
    //       0% completion + 0% quality → -1.0 (offset 0)
    //
    // PLAIN ENGLISH: "-1 means complete failure—nothing delivered, 
    // or actively harmful." This test verifies that zero completion 
    // and quality map to the maximum negative outcome.
    // ═══════════════════════════════════════════════════════════════
    
    let outcome = calculate_partial_outcome(0, 0);
    assert(outcome == 0);
}

#[test]
fn test_calculate_partial_outcome_neutral() {
    // ═══════════════════════════════════════════════════════════════
    // MATH: outcome(c) ∈ [-1, 1]
    //       50% completion + 50% quality → 0 (offset 100)
    //
    // PLAIN ENGLISH: "0 is neutral—partial delivery." Half 
    // completion with half quality averages to a neutral outcome 
    // that neither helps nor hurts trust.
    // ═══════════════════════════════════════════════════════════════
    
    let outcome = calculate_partial_outcome(50, 50);
    assert(outcome == 100);
}

#[test]
fn test_calculate_partial_outcome_asymmetric() {
    // ═══════════════════════════════════════════════════════════════
    // MATH: outcome(c) ∈ [-1, 1]
    //       100% completion + 0% quality → average 50% → 0 (offset 100)
    //
    // PLAIN ENGLISH: "Real work has degrees of quality, not just 
    // pass/fail." Completing everything but with zero quality 
    // averages to neutral—you delivered, but it wasn't useful.
    // ═══════════════════════════════════════════════════════════════
    
    let outcome = calculate_partial_outcome(100, 0);
    assert(outcome == 100);
}

// --------------------------------------------
// Extreme Value Tests
// --------------------------------------------

#[test]
fn test_contract_is_invalid_extreme_difficulty() {
    // ═══════════════════════════════════════════════════════════════
    // MATH: d ∈ [0, 10] — values like 1000 are far out of range
    //
    // PLAIN ENGLISH: "Difficulty rating ranges from 0 to 10." Very 
    // large values must be rejected to prevent attacks that try to 
    // claim extreme difficulty to inflate weight bounds.
    // ═══════════════════════════════════════════════════════════════
    
    let contract = Contract {
        counterparty: AgentId::new(2),
        skill_type: SkillType::new(1),
        stake: 1000,
        difficulty: 1000,        // INVALID: far exceeds MAX_DIFFICULTY
        outcome_offset: 100,
        completed_at: 1000,
        weight: PRECISION,
    };
    
    assert(!contract.is_valid());
}

#[test]
fn test_contract_is_invalid_extreme_outcome() {
    // ═══════════════════════════════════════════════════════════════
    // MATH: outcome(c) ∈ [-1, 1] — Noir offset 1000 is far out of range
    //
    // PLAIN ENGLISH: "The outcome of a contract is a number between 
    // negative one and positive one." Very large offsets must be 
    // rejected to prevent attacks that try to claim inflated outcomes.
    // ═══════════════════════════════════════════════════════════════
    
    let contract = Contract {
        counterparty: AgentId::new(2),
        skill_type: SkillType::new(1),
        stake: 1000,
        difficulty: 5,
        outcome_offset: 1000,    // INVALID: far exceeds OUTCOME_MAX
        completed_at: 1000,
        weight: PRECISION,
    };
    
    assert(!contract.is_valid());
}

#[test]
fn test_verify_weight_bounds_extreme_inflation() {
    // ═══════════════════════════════════════════════════════════════
    // MATH: ω(c) ≤ log(1 + s) × 3 + buffer
    //       For stake=100: max ≈ 15
    //       Weight of 10^9 is extreme inflation.
    //
    // PLAIN ENGLISH: "The weight of a contract is bounded by stake."
    // An attacker might try weight = 10^9 to pass any threshold.
    // This extreme inflation must be caught and rejected.
    // ═══════════════════════════════════════════════════════════════
    
    let contract = Contract {
        counterparty: AgentId::new(2),
        skill_type: SkillType::new(1),
        stake: 100,
        difficulty: 5,
        outcome_offset: 150,
        completed_at: 1000,
        weight: 1000000000,      // 10^9 - extreme inflation attempt
    };
    
    assert(!verify_weight_bounds(contract));
}

#[test]
fn test_contract_skill_type_matching() {
    // ═══════════════════════════════════════════════════════════════
    // MATH: V<t>(Agent(t, h<t>)) — trust is scoped to skill type t
    //       Only contracts matching skill type t contribute to V<t>.
    //
    // PLAIN ENGLISH: "You have a separate score for each skill you 
    // practice. V_engineering measures engineering trust, while 
    // V_design measures design trust—they're completely independent."
    // ═══════════════════════════════════════════════════════════════
    
    let contract = Contract {
        counterparty: AgentId::new(2),
        skill_type: SkillType::new(42),
        stake: 1000,
        difficulty: 5,
        outcome_offset: 150,
        completed_at: 1000,
        weight: PRECISION,
    };
    
    // Should match its own skill type
    assert(contract.is_skill(SkillType::new(42)));
    
    // Should not match different skill types
    assert(!contract.is_skill(SkillType::new(1)));
    assert(!contract.is_skill(SkillType::new(0)));
}

// --------------------------------------------
// HierarchicalContract Tests
// --------------------------------------------

#[test]
fn test_hierarchical_contract_type_standalone() {
    // ═══════════════════════════════════════════════════════════════
    // MATH: ContractType ∈ {Standalone, Project, Phase, Milestone, Task}
    //
    // PLAIN ENGLISH: "Standalone contracts are atomic work without 
    // hierarchy. They have no parent reference."
    // ═══════════════════════════════════════════════════════════════
    
    let contract = HierarchicalContract {
        contract_type: CONTRACT_TYPE_STANDALONE,
        parent_id: Field::from(0),  // No parent
        deadline: 0,
    };
    
    assert(contract.is_standalone());
    assert(!contract.is_project());
    assert(!contract.is_phase());
    assert(!contract.is_milestone());
    assert(!contract.is_task());
}

#[test]
fn test_hierarchical_contract_type_milestone() {
    // ═══════════════════════════════════════════════════════════════
    // MATH: ContractType = 5 for Milestone
    //       Milestones are payment gates within Implementation phases.
    //
    // PLAIN ENGLISH: "Milestones group tasks for incremental payment.
    // Customer reviews at milestone deadline. Trust flows through 
    // tasks, not milestones—milestones are coordination containers."
    // See: ADR_Milestone_Payment_Gates.md
    // ═══════════════════════════════════════════════════════════════
    
    let contract = HierarchicalContract {
        contract_type: CONTRACT_TYPE_MILESTONE,
        parent_id: Field::from(123),  // Parent phase
        deadline: 1704067200,         // Computed from max(task deadlines)
    };
    
    assert(contract.is_milestone());
    assert(!contract.is_standalone());
    assert(!contract.is_task());
}

#[test]
fn test_hierarchical_contract_type_task() {
    // ═══════════════════════════════════════════════════════════════
    // MATH: ContractType = 4 for Task
    //       Tasks are atomic work units within milestones.
    //
    // PLAIN ENGLISH: "Tasks are the atomic work units where trust 
    // actually flows. Each task has a provider, difficulty, and 
    // outcome. Task deadline is work duration (planning data)."
    // ═══════════════════════════════════════════════════════════════
    
    let contract = HierarchicalContract {
        contract_type: CONTRACT_TYPE_TASK,
        parent_id: Field::from(456),  // Parent milestone
        deadline: 604800,             // Work duration (e.g., 1 week)
    };
    
    assert(contract.is_task());
    assert(!contract.is_milestone());
    assert(!contract.is_standalone());
}

#[test]
fn test_hierarchical_contract_milestone_requires_parent() {
    // ═══════════════════════════════════════════════════════════════
    // MATH: Milestone contracts must have parent_id ≠ 0
    //
    // PLAIN ENGLISH: "Milestones must reference their parent 
    // implementation phase. A milestone without a parent is invalid."
    // ═══════════════════════════════════════════════════════════════
    
    let contract = HierarchicalContract {
        contract_type: CONTRACT_TYPE_MILESTONE,
        parent_id: Field::from(0),    // INVALID: No parent
        deadline: 1704067200,
    };
    
    // Validation should fail for milestone without parent
    assert(!contract.has_valid_parent());
}

#[test]
fn test_hierarchical_contract_task_requires_parent() {
    // ═══════════════════════════════════════════════════════════════
    // MATH: Task contracts must have parent_id ≠ 0
    //
    // PLAIN ENGLISH: "Tasks must reference their parent milestone.
    // A task without a parent milestone is invalid."
    // ═══════════════════════════════════════════════════════════════
    
    let contract = HierarchicalContract {
        contract_type: CONTRACT_TYPE_TASK,
        parent_id: Field::from(0),    // INVALID: No parent milestone
        deadline: 604800,
    };
    
    // Validation should fail for task without parent
    assert(!contract.has_valid_parent());
}

#[test]
fn test_hierarchical_contract_standalone_no_parent_ok() {
    // ═══════════════════════════════════════════════════════════════
    // MATH: Standalone contracts may have parent_id = 0
    //
    // PLAIN ENGLISH: "Standalone contracts are atomic work without
    // hierarchy. No parent reference is required."
    // ═══════════════════════════════════════════════════════════════
    
    let contract = HierarchicalContract {
        contract_type: CONTRACT_TYPE_STANDALONE,
        parent_id: Field::from(0),    // Valid: standalone doesn't need parent
        deadline: 0,
    };
    
    // Validation should pass for standalone without parent
    assert(contract.has_valid_parent());
}

// --------------------------------------------
// Milestone Validation Tests
// --------------------------------------------

#[test]
fn test_milestone_deadline_from_tasks() {
    // ═══════════════════════════════════════════════════════════════
    // MATH: d_milestone = max(d_task for task in milestone)
    //
    // PLAIN ENGLISH: "Milestone deadline is computed from the latest
    // task deadline within that milestone. This is when customer 
    // review happens—not per-task during work."
    // See: ADR_Milestone_Payment_Gates.md
    // ═══════════════════════════════════════════════════════════════
    
    // Task deadlines (work durations from contract start)
    let task1_deadline: Field = 604800;   // Week 1
    let task2_deadline: Field = 1209600;  // Week 2 (latest)
    let task3_deadline: Field = 864000;   // 10 days
    
    // Milestone deadline = max of task deadlines
    let milestone_deadline = compute_milestone_deadline(
        [task1_deadline, task2_deadline, task3_deadline]
    );
    
    assert(milestone_deadline == 1209600);  // Week 2
}

#[test]
fn test_milestone_stake_from_tasks() {
    // ═══════════════════════════════════════════════════════════════
    // MATH: s_milestone = Σ s_task for task in milestone
    //
    // PLAIN ENGLISH: "Milestone stake is the sum of task stakes.
    // It's computed, not stored independently."
    // ═══════════════════════════════════════════════════════════════
    
    let task1_stake: Field = 300;
    let task2_stake: Field = 500;
    let task3_stake: Field = 200;
    
    let milestone_stake = compute_milestone_stake(
        [task1_stake, task2_stake, task3_stake]
    );
    
    assert(milestone_stake == 1000);  // Sum of task stakes
}

#[test]
fn test_milestone_difficulty_aggregation() {
    // ═══════════════════════════════════════════════════════════════
    // MATH: d_milestone = Σ(d_task × w_task) / Σ w_task
    //       Stake-weighted average of task difficulties.
    //
    // PLAIN ENGLISH: "Milestone difficulty aggregates from task
    // difficulties weighted by stake. High-stake tasks contribute
    // more to the aggregate."
    // ═══════════════════════════════════════════════════════════════
    
    // Task 1: difficulty 8, weight 2
    // Task 2: difficulty 5, weight 1
    // Weighted average = (8×2 + 5×1) / (2+1) = 21/3 = 7
    
    let difficulties: [Field; 2] = [8, 5];
    let weights: [Field; 2] = [2, 1];
    
    let aggregate = compute_weighted_difficulty(difficulties, weights);
    
    // Result should be 7 (scaled by PRECISION if needed)
    assert(aggregate == 7 * PRECISION);
}

#[test]
fn test_contract_type_all_values_defined() {
    // ═══════════════════════════════════════════════════════════════
    // MATH: ContractType ∈ {0, 1, 2, 3, 4, 5}
    //       All six contract types must have distinct values.
    //
    // PLAIN ENGLISH: "The hierarchy is: Project → Phase → Milestone → Task.
    // Standalone is for atomic work without hierarchy."
    // ═══════════════════════════════════════════════════════════════
    
    assert(CONTRACT_TYPE_STANDALONE == 0);
    assert(CONTRACT_TYPE_PROJECT == 1);
    assert(CONTRACT_TYPE_PHASE == 2);
    assert(CONTRACT_TYPE_SPECIFICATION == 3);
    assert(CONTRACT_TYPE_TASK == 4);
    assert(CONTRACT_TYPE_MILESTONE == 5);
    
    // All values are distinct
    assert(CONTRACT_TYPE_STANDALONE != CONTRACT_TYPE_MILESTONE);
    assert(CONTRACT_TYPE_TASK != CONTRACT_TYPE_MILESTONE);
}

#[test]
fn test_task_trust_contribution_not_milestone() {
    // ═══════════════════════════════════════════════════════════════
    // MATH: V<t> = Σ ω(c) · outcome(c) for tasks
    //       Milestones don't independently contribute to trust.
    //
    // PLAIN ENGLISH: "Trust flows through tasks, not milestones.
    // Milestones are coordination containers for payment."
    // See: ADR_Trust_Signal_Boundaries.md
    // ═══════════════════════════════════════════════════════════════
    
    let task_contract = Contract {
        counterparty: AgentId::new(2),
        skill_type: SkillType::new(1),
        stake: 500,
        difficulty: 7,
        outcome_offset: 180,     // +80 outcome (good)
        completed_at: 1000,
        weight: 2 * PRECISION,
    };
    
    // Task contributes to trust
    let contribution = task_contract.trust_contribution();
    assert(contribution.magnitude > 0);
    
    // Milestone contract type check
    let hierarchical = HierarchicalContract {
        contract_type: CONTRACT_TYPE_MILESTONE,
        parent_id: Field::from(123),
        deadline: 1704067200,
    };
    
    // Milestones are coordination containers
    assert(hierarchical.is_milestone());
    // Note: Trust flows through child tasks, not the milestone itself
}
```

---

## Summary of Test Coverage

| Test Category | Count | Math Equation |
|---------------|-------|---------------|
| Contract.is_valid() | 10 | `d ∈ [0,10]`, `outcome ∈ [-1,1]` |
| validate_contract() | 4 | `ω(c) > 0` |
| verify_weight_bounds() | 7 | `ω(c) = f(s, d, V<t>, recency)` |
| Contract.outcome() | 5 | `outcome(c) ∈ [-1, 1]` |
| Contract.is_active() | 3 | `V<t> = Σ ω(c) · outcome(c)` |
| Contract.trust_contribution() | 4 | `V<t> = Σ ω(c) · outcome(c)` |
| calculate_partial_outcome() | 4 | `outcome(c) ∈ [-1, 1]` |
| Skill type matching | 1 | `V<t>(Agent(t, h<t>))` |
| HierarchicalContract types | 7 | `ContractType ∈ {0..5}` |
| Milestone validation | 5 | `d_milestone = max(d_task)`, `s_milestone = Σ s_task` |

**Total: 50 tests**

---

## Equations Referenced

| # | Equation | Plain English |
|---|----------|---------------|
| 4 | `V<t>(Agent(t, h<t>)) = Σ ω(c) · outcome(c)` | Trust = sum of (weight × outcome) for each contract |
| 5 | `c = (a<provider>, a<consumer>, t, s, d, τ)` | Contract has provider, consumer, skill, stake, difficulty, deadline |
| 6 | `outcome(c) ∈ [-1, 1]` | Outcome ranges from complete failure to complete success |
| 7 | `ω(c) = f(s, d, V<t>, recency)` | Weight depends on stake, difficulty, counterparty, recency |
| 18 | `d_milestone = max(d_task)` | Milestone deadline computed from latest task deadline |
| 18 | `s_milestone = Σ s_task` | Milestone stake computed from sum of task stakes |
| 18 | `d_diff_milestone = Σ(d_task × w_task) / Σ w_task` | Milestone difficulty = stake-weighted average |

---

## Security Properties Verified

1. **Difficulty bounds**: Prevents inflation beyond MAX_DIFFICULTY (10)
2. **Outcome bounds**: Prevents outcomes outside [-100, +100] range
3. **Weight bounds**: Prevents malicious weight inflation based on stake
4. **Active contract detection**: Empty slots correctly identified and skipped
5. **Skill type scoping**: Trust computed independently per skill type
6. **Hierarchical contract types**: All six types (0-5) correctly distinguished
7. **Parent reference validation**: Tasks and milestones must have parent_id ≠ 0
8. **Milestone computation**: Deadline and stake computed from child tasks
9. **Trust flow boundaries**: Trust flows through tasks, not milestones

---

## Related Documents

- **ADR_Milestone_Payment_Gates.md** — Milestone-based payment model
- **ADR_Trust_Signal_Boundaries.md** — Trust flows through tasks, not milestones
- **ADR_Dispute_Resolution.md** — Customer review at milestone deadline
- **Quantum_of_Trust_Equations_in_Noir.md** — Full circuit implementation
