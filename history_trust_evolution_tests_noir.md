# History/Trust Evolution Tests (Noir)

These tests verify the history and trust evolution functions that implement
the append-only history model and incremental trust updates.

---

## Tests

```noir
// ============================================
// HISTORY/TRUST EVOLUTION TESTS
// ============================================

// --------------------------------------------
// HistoryState Tests
// --------------------------------------------

#[test]
fn test_history_state_empty() {
    // ═══════════════════════════════════════════════════════════════
    // MATH: h<t>^(0) = ∅ (empty initial history)
    //
    // PLAIN ENGLISH: "A new Avatar starts with an empty history."
    // The HistoryState::empty() function creates a valid initial state.
    // ═══════════════════════════════════════════════════════════════
    
    let state = HistoryState::empty();
    
    assert(state.history_root == 0);
    assert(state.cached_trust.magnitude == 0);
    assert(!state.cached_trust.is_negative);
    assert(state.contract_count == 0);
}

// --------------------------------------------
// add_to_history Tests
// --------------------------------------------

#[test]
fn test_add_to_history_first_contract_success() {
    // ═══════════════════════════════════════════════════════════════
    // MATH: h<t>^(n+1)(a) = h<t>^(n)(a) ∪ {c<n>}
    //       V<t>^(n+1) = V<t>^(n) + ω(c<n>) · outcome(c<n>)
    //
    // PLAIN ENGLISH: "An Avatar's history after completing contract n 
    // equals their previous history plus that new contract." Adding 
    // the first contract to empty history.
    // ═══════════════════════════════════════════════════════════════
    
    let empty_state = HistoryState::empty();
    let skill = SkillType::new(1);
    
    // First contract: success with weight 1.0
    let contract = Contract {
        counterparty: AgentId::new(2),
        skill_type: skill,
        stake: 1000,
        difficulty: 5,
        outcome_offset: 200,  // +100 (success)
        completed_at: 1000,
        weight: PRECISION,    // 1.0
    };
    
    let new_state = add_to_history(empty_state, contract, skill);
    
    // Contract count increased by 1
    assert(new_state.contract_count == 1);
    
    // Trust updated: 0 + (1.0 × 1.0) = 1.0
    assert(new_state.cached_trust.magnitude == PRECISION);
    assert(!new_state.cached_trust.is_negative);
}

#[test]
fn test_add_to_history_first_contract_failure() {
    // ═══════════════════════════════════════════════════════════════
    // MATH: V<t>^(n+1) = V<t>^(n) + ω(c<n>) · outcome(c<n>)
    //       When outcome < 0, trust decreases
    //
    // PLAIN ENGLISH: "If you fail, that failure stays in your history 
    // forever." First contract failure creates negative trust.
    // ═══════════════════════════════════════════════════════════════
    
    let empty_state = HistoryState::empty();
    let skill = SkillType::new(1);
    
    // First contract: failure with weight 1.0
    let contract = Contract {
        counterparty: AgentId::new(2),
        skill_type: skill,
        stake: 1000,
        difficulty: 5,
        outcome_offset: 0,    // -100 (failure)
        completed_at: 1000,
        weight: PRECISION,
    };
    
    let new_state = add_to_history(empty_state, contract, skill);
    
    // Contract count increased
    assert(new_state.contract_count == 1);
    
    // Trust updated: 0 + (1.0 × -1.0) = -1.0
    assert(new_state.cached_trust.magnitude == PRECISION);
    assert(new_state.cached_trust.is_negative);
}

#[test]
fn test_add_to_history_sequential_contracts() {
    // ═══════════════════════════════════════════════════════════════
    // MATH: h<t>^(n+1) = h<t>^(n) ∪ {c<n>}
    //       History is append-only—it only grows, never shrinks.
    //
    // PLAIN ENGLISH: "Every time you complete a contract, it gets 
    // added to your permanent record." Sequential additions accumulate.
    // ═══════════════════════════════════════════════════════════════
    
    let skill = SkillType::new(1);
    let mut state = HistoryState::empty();
    
    // Add first contract: +1.0
    let contract1 = Contract {
        counterparty: AgentId::new(2),
        skill_type: skill,
        stake: 1000,
        difficulty: 5,
        outcome_offset: 200,
        completed_at: 1000,
        weight: PRECISION,
    };
    state = add_to_history(state, contract1, skill);
    
    assert(state.contract_count == 1);
    assert(state.cached_trust.magnitude == PRECISION);
    
    // Add second contract: +1.0
    let contract2 = Contract {
        counterparty: AgentId::new(3),
        skill_type: skill,
        stake: 1000,
        difficulty: 5,
        outcome_offset: 200,
        completed_at: 1001,
        weight: PRECISION,
    };
    state = add_to_history(state, contract2, skill);
    
    assert(state.contract_count == 2);
    assert(state.cached_trust.magnitude == 2 * PRECISION);  // 1.0 + 1.0 = 2.0
    
    // Add third contract: -1.0 (failure)
    let contract3 = Contract {
        counterparty: AgentId::new(4),
        skill_type: skill,
        stake: 1000,
        difficulty: 5,
        outcome_offset: 0,
        completed_at: 1002,
        weight: PRECISION,
    };
    state = add_to_history(state, contract3, skill);
    
    assert(state.contract_count == 3);
    assert(state.cached_trust.magnitude == PRECISION);  // 2.0 - 1.0 = 1.0
    assert(!state.cached_trust.is_negative);
}

#[test]
fn test_add_to_history_trust_goes_negative() {
    // ═══════════════════════════════════════════════════════════════
    // MATH: V<t>^(n+1) = V<t>^(n) + ω(c<n>) · outcome(c<n>)
    //       Trust can transition from positive to negative
    //
    // PLAIN ENGLISH: "A negative value means more failures than 
    // successes—you're actively distrusted." Trust can cross zero.
    // ═══════════════════════════════════════════════════════════════
    
    let skill = SkillType::new(1);
    let mut state = HistoryState::empty();
    
    // Start with one success: +1.0
    let contract1 = Contract {
        counterparty: AgentId::new(2),
        skill_type: skill,
        stake: 1000,
        difficulty: 5,
        outcome_offset: 200,
        completed_at: 1000,
        weight: PRECISION,
    };
    state = add_to_history(state, contract1, skill);
    assert(!state.cached_trust.is_negative);
    
    // Add two failures: -2.0 total
    let contract2 = Contract {
        counterparty: AgentId::new(3),
        skill_type: skill,
        stake: 1000,
        difficulty: 5,
        outcome_offset: 0,
        completed_at: 1001,
        weight: PRECISION,
    };
    state = add_to_history(state, contract2, skill);
    
    let contract3 = Contract {
        counterparty: AgentId::new(4),
        skill_type: skill,
        stake: 1000,
        difficulty: 5,
        outcome_offset: 0,
        completed_at: 1002,
        weight: PRECISION,
    };
    state = add_to_history(state, contract3, skill);
    
    // Net trust: 1.0 - 1.0 - 1.0 = -1.0
    assert(state.contract_count == 3);
    assert(state.cached_trust.magnitude == PRECISION);
    assert(state.cached_trust.is_negative);
}

#[test]
fn test_add_to_history_with_different_weights() {
    // ═══════════════════════════════════════════════════════════════
    // MATH: V<t>^(n+1) = V<t>^(n) + ω(c<n>) · outcome(c<n>)
    //       Different weights produce different contributions
    //
    // PLAIN ENGLISH: "A successful contract with high weight adds a 
    // lot to your trust." Weight determines impact magnitude.
    // ═══════════════════════════════════════════════════════════════
    
    let skill = SkillType::new(1);
    let mut state = HistoryState::empty();
    
    // High weight success: +5.0
    let contract1 = Contract {
        counterparty: AgentId::new(2),
        skill_type: skill,
        stake: 10000,
        difficulty: 10,
        outcome_offset: 200,
        completed_at: 1000,
        weight: 5 * PRECISION,  // 5.0
    };
    state = add_to_history(state, contract1, skill);
    
    assert(state.cached_trust.magnitude == 5 * PRECISION);
    
    // Low weight failure: -0.5
    let contract2 = Contract {
        counterparty: AgentId::new(3),
        skill_type: skill,
        stake: 100,
        difficulty: 1,
        outcome_offset: 0,
        completed_at: 1001,
        weight: PRECISION / 2,  // 0.5
    };
    state = add_to_history(state, contract2, skill);
    
    // Net: 5.0 - 0.5 = 4.5
    assert(state.cached_trust.magnitude == 5 * PRECISION - PRECISION / 2);
    assert(!state.cached_trust.is_negative);
}

#[test]
fn test_add_to_history_partial_outcomes() {
    // ═══════════════════════════════════════════════════════════════
    // MATH: outcome(c) ∈ [-1, 1]
    //       Partial outcomes produce partial contributions
    //
    // PLAIN ENGLISH: "This continuous range allows for nuance. +0.5 
    // means good but not perfect." Partial outcomes adjust trust 
    // proportionally.
    // ═══════════════════════════════════════════════════════════════
    
    let skill = SkillType::new(1);
    let mut state = HistoryState::empty();
    
    // Partial success: outcome = +0.5, weight = 1.0
    // contribution = 1.0 × 0.5 = 0.5
    let contract = Contract {
        counterparty: AgentId::new(2),
        skill_type: skill,
        stake: 1000,
        difficulty: 5,
        outcome_offset: 150,  // +50 out of 100
        completed_at: 1000,
        weight: PRECISION,
    };
    state = add_to_history(state, contract, skill);
    
    // Trust = 0.5 (scaled: PRECISION/2)
    assert(state.cached_trust.magnitude == PRECISION / 2);
    assert(!state.cached_trust.is_negative);
}

// --------------------------------------------
// update_trust Tests
// --------------------------------------------

#[test]
fn test_update_trust_from_zero() {
    // ═══════════════════════════════════════════════════════════════
    // MATH: V<t>^(n+1) = V<t>^(n) + ω(c<n>) · outcome(c<n>)
    //       When V<t>^(n) = 0, new trust equals contribution
    //
    // PLAIN ENGLISH: "Your trust value after completing a contract 
    // equals your previous trust value plus (the contract's weight 
    // times its outcome)."
    // ═══════════════════════════════════════════════════════════════
    
    let current_trust = Signed::zero();
    
    let contract = Contract {
        counterparty: AgentId::new(2),
        skill_type: SkillType::new(1),
        stake: 1000,
        difficulty: 5,
        outcome_offset: 200,  // +100
        completed_at: 1000,
        weight: PRECISION,
    };
    
    let new_trust = update_trust(current_trust, contract);
    
    // 0 + 1.0 = 1.0
    assert(new_trust.magnitude == PRECISION);
    assert(!new_trust.is_negative);
}

#[test]
fn test_update_trust_positive_to_more_positive() {
    // ═══════════════════════════════════════════════════════════════
    // MATH: V<t>^(n+1) = V<t>^(n) + ω(c<n>) · outcome(c<n>)
    //       Success increases positive trust
    //
    // PLAIN ENGLISH: "Succeed at a big contract (+8 weight × +1 
    // outcome) → trust increases by 8."
    // ═══════════════════════════════════════════════════════════════
    
    let current_trust = Signed::from_positive(5 * PRECISION);  // 5.0
    
    let contract = Contract {
        counterparty: AgentId::new(2),
        skill_type: SkillType::new(1),
        stake: 1000,
        difficulty: 5,
        outcome_offset: 200,
        completed_at: 1000,
        weight: 3 * PRECISION,  // 3.0
    };
    
    let new_trust = update_trust(current_trust, contract);
    
    // 5.0 + 3.0 = 8.0
    assert(new_trust.magnitude == 8 * PRECISION);
    assert(!new_trust.is_negative);
}

#[test]
fn test_update_trust_positive_to_less_positive() {
    // ═══════════════════════════════════════════════════════════════
    // MATH: V<t>^(n+1) = V<t>^(n) + ω(c<n>) · outcome(c<n>)
    //       Failure decreases positive trust (but stays positive)
    //
    // PLAIN ENGLISH: "Fail at a small contract (+2 weight × -1 
    // outcome) → trust decreases by 2."
    // ═══════════════════════════════════════════════════════════════
    
    let current_trust = Signed::from_positive(5 * PRECISION);  // 5.0
    
    let contract = Contract {
        counterparty: AgentId::new(2),
        skill_type: SkillType::new(1),
        stake: 1000,
        difficulty: 5,
        outcome_offset: 0,  // -100 (failure)
        completed_at: 1000,
        weight: 2 * PRECISION,  // 2.0
    };
    
    let new_trust = update_trust(current_trust, contract);
    
    // 5.0 - 2.0 = 3.0
    assert(new_trust.magnitude == 3 * PRECISION);
    assert(!new_trust.is_negative);
}

#[test]
fn test_update_trust_positive_to_negative() {
    // ═══════════════════════════════════════════════════════════════
    // MATH: V<t>^(n+1) = V<t>^(n) + ω(c<n>) · outcome(c<n>)
    //       Large failure can flip trust from positive to negative
    //
    // PLAIN ENGLISH: "Each contract moves your score up or down."
    // A big enough failure can cross zero.
    // ═══════════════════════════════════════════════════════════════
    
    let current_trust = Signed::from_positive(2 * PRECISION);  // 2.0
    
    let contract = Contract {
        counterparty: AgentId::new(2),
        skill_type: SkillType::new(1),
        stake: 1000,
        difficulty: 5,
        outcome_offset: 0,  // -100 (failure)
        completed_at: 1000,
        weight: 5 * PRECISION,  // 5.0
    };
    
    let new_trust = update_trust(current_trust, contract);
    
    // 2.0 - 5.0 = -3.0
    assert(new_trust.magnitude == 3 * PRECISION);
    assert(new_trust.is_negative);
}

#[test]
fn test_update_trust_negative_to_more_negative() {
    // ═══════════════════════════════════════════════════════════════
    // MATH: V<t>^(n+1) = V<t>^(n) + ω(c<n>) · outcome(c<n>)
    //       Failure deepens negative trust
    //
    // PLAIN ENGLISH: "An Avatar with V<t> = -50 is worse than an 
    // unknown newcomer—they've proven they can't deliver."
    // ═══════════════════════════════════════════════════════════════
    
    let current_trust = Signed::from_negative(3 * PRECISION);  // -3.0
    
    let contract = Contract {
        counterparty: AgentId::new(2),
        skill_type: SkillType::new(1),
        stake: 1000,
        difficulty: 5,
        outcome_offset: 0,  // -100 (failure)
        completed_at: 1000,
        weight: 2 * PRECISION,  // 2.0
    };
    
    let new_trust = update_trust(current_trust, contract);
    
    // -3.0 - 2.0 = -5.0
    assert(new_trust.magnitude == 5 * PRECISION);
    assert(new_trust.is_negative);
}

#[test]
fn test_update_trust_negative_to_positive() {
    // ═══════════════════════════════════════════════════════════════
    // MATH: V<t>^(n+1) = V<t>^(n) + ω(c<n>) · outcome(c<n>)
    //       Large success can rehabilitate negative trust
    //
    // PLAIN ENGLISH: "Trust changes incrementally." Redemption is 
    // possible with enough successful contracts.
    // ═══════════════════════════════════════════════════════════════
    
    let current_trust = Signed::from_negative(2 * PRECISION);  // -2.0
    
    let contract = Contract {
        counterparty: AgentId::new(2),
        skill_type: SkillType::new(1),
        stake: 1000,
        difficulty: 5,
        outcome_offset: 200,  // +100 (success)
        completed_at: 1000,
        weight: 5 * PRECISION,  // 5.0
    };
    
    let new_trust = update_trust(current_trust, contract);
    
    // -2.0 + 5.0 = 3.0
    assert(new_trust.magnitude == 3 * PRECISION);
    assert(!new_trust.is_negative);
}

#[test]
fn test_update_trust_to_exactly_zero() {
    // ═══════════════════════════════════════════════════════════════
    // MATH: V<t>^(n+1) = V<t>^(n) + ω(c<n>) · outcome(c<n>) = 0
    //       Cancellation to exactly zero
    //
    // PLAIN ENGLISH: "A trust value of zero means no track record."
    // But here it means perfect cancellation of positive and negative.
    // ═══════════════════════════════════════════════════════════════
    
    let current_trust = Signed::from_positive(3 * PRECISION);  // 3.0
    
    let contract = Contract {
        counterparty: AgentId::new(2),
        skill_type: SkillType::new(1),
        stake: 1000,
        difficulty: 5,
        outcome_offset: 0,  // -100 (failure)
        completed_at: 1000,
        weight: 3 * PRECISION,  // 3.0
    };
    
    let new_trust = update_trust(current_trust, contract);
    
    // 3.0 - 3.0 = 0
    assert(new_trust.magnitude == 0);
    assert(!new_trust.is_negative);  // Zero is never negative
}

#[test]
fn test_update_trust_partial_outcome() {
    // ═══════════════════════════════════════════════════════════════
    // MATH: V<t>^(n+1) = V<t>^(n) + ω(c<n>) · outcome(c<n>)
    //       Partial outcomes: +0.6 outcome
    //
    // PLAIN ENGLISH: "Partial success on medium contract (+5 weight 
    // × +0.6 outcome) → trust increases by 3."
    // ═══════════════════════════════════════════════════════════════
    
    let current_trust = Signed::from_positive(10 * PRECISION);  // 10.0
    
    // outcome_offset = 160 means outcome = +60 (scaled: +0.6)
    let contract = Contract {
        counterparty: AgentId::new(2),
        skill_type: SkillType::new(1),
        stake: 1000,
        difficulty: 5,
        outcome_offset: 160,  // +60
        completed_at: 1000,
        weight: 5 * PRECISION,  // 5.0
    };
    
    let new_trust = update_trust(current_trust, contract);
    
    // contribution = 5.0 × 0.6 = 3.0
    // new_trust = 10.0 + 3.0 = 13.0
    assert(new_trust.magnitude == 13 * PRECISION);
    assert(!new_trust.is_negative);
}

// --------------------------------------------
// verify_trust_update Tests
// --------------------------------------------

#[test]
fn test_verify_trust_update_valid_positive_contribution() {
    // ═══════════════════════════════════════════════════════════════
    // MATH: V<t>^(n+1) = V<t>^(n) + ω(c<n>) · outcome(c<n>)
    //       Verification: new_trust == old_trust + contribution
    //
    // PLAIN ENGLISH: "Used when trust updates need to be publicly 
    // verifiable." Verifies correct calculation.
    // ═══════════════════════════════════════════════════════════════
    
    // old_trust = 5.0, contract: weight=2.0, outcome=+100
    // contribution = 2.0 × 1.0 = 2.0
    // new_trust = 5.0 + 2.0 = 7.0
    let result = verify_trust_update(
        5 * PRECISION,   // old_trust_magnitude
        false,           // old_trust_negative
        7 * PRECISION,   // new_trust_magnitude
        false,           // new_trust_negative
        2 * PRECISION,   // contract_weight
        200              // contract_outcome_offset (+100)
    );
    
    assert(result);
}

#[test]
fn test_verify_trust_update_valid_negative_contribution() {
    // ═══════════════════════════════════════════════════════════════
    // MATH: V<t>^(n+1) = V<t>^(n) + ω(c<n>) · outcome(c<n>)
    //       Failure subtracts from trust
    //
    // PLAIN ENGLISH: "A failed contract with high weight subtracts 
    // a lot from your trust."
    // ═══════════════════════════════════════════════════════════════
    
    // old_trust = 5.0, contract: weight=2.0, outcome=-100
    // contribution = 2.0 × -1.0 = -2.0
    // new_trust = 5.0 - 2.0 = 3.0
    let result = verify_trust_update(
        5 * PRECISION,   // old_trust_magnitude
        false,           // old_trust_negative
        3 * PRECISION,   // new_trust_magnitude
        false,           // new_trust_negative
        2 * PRECISION,   // contract_weight
        0                // contract_outcome_offset (-100)
    );
    
    assert(result);
}

#[test]
fn test_verify_trust_update_crossing_zero() {
    // ═══════════════════════════════════════════════════════════════
    // MATH: V<t>^(n+1) = V<t>^(n) + ω(c<n>) · outcome(c<n>)
    //       Trust crossing from positive to negative
    //
    // PLAIN ENGLISH: "Each contract moves your score up or down."
    // ═══════════════════════════════════════════════════════════════
    
    // old_trust = 2.0, contract: weight=5.0, outcome=-100
    // contribution = 5.0 × -1.0 = -5.0
    // new_trust = 2.0 - 5.0 = -3.0
    let result = verify_trust_update(
        2 * PRECISION,   // old_trust_magnitude
        false,           // old_trust_negative (positive 2.0)
        3 * PRECISION,   // new_trust_magnitude
        true,            // new_trust_negative (negative 3.0)
        5 * PRECISION,   // contract_weight
        0                // contract_outcome_offset (-100)
    );
    
    assert(result);
}

#[test]
fn test_verify_trust_update_invalid_wrong_magnitude() {
    // ═══════════════════════════════════════════════════════════════
    // MATH: Verification fails if new_trust ≠ old_trust + contribution
    //
    // PLAIN ENGLISH: "The verifier can check that new_trust = 
    // old_trust + contribution." Invalid updates are rejected.
    // ═══════════════════════════════════════════════════════════════
    
    // old_trust = 5.0, contract: weight=2.0, outcome=+100
    // Expected: 5.0 + 2.0 = 7.0
    // Claimed: 8.0 (WRONG!)
    let result = verify_trust_update(
        5 * PRECISION,   // old_trust_magnitude
        false,           // old_trust_negative
        8 * PRECISION,   // new_trust_magnitude (WRONG - should be 7)
        false,           // new_trust_negative
        2 * PRECISION,   // contract_weight
        200              // contract_outcome_offset (+100)
    );
    
    assert(!result);
}

#[test]
fn test_verify_trust_update_invalid_wrong_sign() {
    // ═══════════════════════════════════════════════════════════════
    // MATH: Verification fails if sign is wrong
    //
    // PLAIN ENGLISH: "Invalid updates are rejected." Wrong sign means
    // the update is fraudulent.
    // ═══════════════════════════════════════════════════════════════
    
    // old_trust = 5.0, contract: weight=2.0, outcome=+100
    // Expected: 5.0 + 2.0 = +7.0 (positive)
    // Claimed: -7.0 (WRONG SIGN!)
    let result = verify_trust_update(
        5 * PRECISION,   // old_trust_magnitude
        false,           // old_trust_negative
        7 * PRECISION,   // new_trust_magnitude (correct)
        true,            // new_trust_negative (WRONG - should be false)
        2 * PRECISION,   // contract_weight
        200              // contract_outcome_offset (+100)
    );
    
    assert(!result);
}

#[test]
fn test_verify_trust_update_partial_outcome() {
    // ═══════════════════════════════════════════════════════════════
    // MATH: contribution = ω × outcome / 100
    //       Partial outcomes work correctly
    //
    // PLAIN ENGLISH: "This continuous range allows for nuance."
    // ═══════════════════════════════════════════════════════════════
    
    // old_trust = 10.0, contract: weight=4.0, outcome=+50 (0.5)
    // contribution = 4.0 × 0.5 = 2.0
    // new_trust = 10.0 + 2.0 = 12.0
    let result = verify_trust_update(
        10 * PRECISION,  // old_trust_magnitude
        false,           // old_trust_negative
        12 * PRECISION,  // new_trust_magnitude
        false,           // new_trust_negative
        4 * PRECISION,   // contract_weight
        150              // contract_outcome_offset (+50)
    );
    
    assert(result);
}

#[test]
fn test_verify_trust_update_from_negative() {
    // ═══════════════════════════════════════════════════════════════
    // MATH: V<t>^(n+1) = V<t>^(n) + ω(c<n>) · outcome(c<n>)
    //       Starting from negative trust
    //
    // PLAIN ENGLISH: "Redemption is possible with enough successful 
    // contracts." Negative trust can improve.
    // ═══════════════════════════════════════════════════════════════
    
    // old_trust = -3.0, contract: weight=2.0, outcome=+100
    // contribution = 2.0 × 1.0 = 2.0
    // new_trust = -3.0 + 2.0 = -1.0
    let result = verify_trust_update(
        3 * PRECISION,   // old_trust_magnitude
        true,            // old_trust_negative (-3.0)
        PRECISION,       // new_trust_magnitude
        true,            // new_trust_negative (-1.0)
        2 * PRECISION,   // contract_weight
        200              // contract_outcome_offset (+100)
    );
    
    assert(result);
}

// --------------------------------------------
// batch_update_trust Tests
// --------------------------------------------

#[test]
fn test_batch_update_trust_empty() {
    // ═══════════════════════════════════════════════════════════════
    // MATH: No contracts means no change to trust
    //
    // PLAIN ENGLISH: "Batch updates trust from multiple contracts."
    // Empty batch leaves trust unchanged.
    // ═══════════════════════════════════════════════════════════════
    
    let initial_trust = Signed::from_positive(5 * PRECISION);
    let contracts: [Contract; MAX_HISTORY] = [Contract::empty(); MAX_HISTORY];
    let skill = SkillType::new(1);
    
    let result = batch_update_trust(initial_trust, contracts, 0, skill);
    
    // No change
    assert(result.magnitude == 5 * PRECISION);
    assert(!result.is_negative);
}

#[test]
fn test_batch_update_trust_multiple_contracts() {
    // ═══════════════════════════════════════════════════════════════
    // MATH: V<t>^(n+k) = V<t>^(n) + Σ ω(c<i>) · outcome(c<i>)
    //       Batch applies multiple updates at once
    //
    // PLAIN ENGLISH: "Useful when initializing from a set of 
    // historical contracts." Processes all contracts in one call.
    // ═══════════════════════════════════════════════════════════════
    
    let initial_trust = Signed::zero();
    let skill = SkillType::new(1);
    
    let mut contracts: [Contract; MAX_HISTORY] = [Contract::empty(); MAX_HISTORY];
    
    // +1.0
    contracts[0] = Contract {
        counterparty: AgentId::new(2),
        skill_type: skill,
        stake: 1000,
        difficulty: 5,
        outcome_offset: 200,
        completed_at: 1000,
        weight: PRECISION,
    };
    
    // +1.0
    contracts[1] = Contract {
        counterparty: AgentId::new(3),
        skill_type: skill,
        stake: 1000,
        difficulty: 5,
        outcome_offset: 200,
        completed_at: 1001,
        weight: PRECISION,
    };
    
    // -1.0
    contracts[2] = Contract {
        counterparty: AgentId::new(4),
        skill_type: skill,
        stake: 1000,
        difficulty: 5,
        outcome_offset: 0,
        completed_at: 1002,
        weight: PRECISION,
    };
    
    let result = batch_update_trust(initial_trust, contracts, 3, skill);
    
    // 0 + 1.0 + 1.0 - 1.0 = 1.0
    assert(result.magnitude == PRECISION);
    assert(!result.is_negative);
}

#[test]
fn test_batch_update_trust_filters_by_skill() {
    // ═══════════════════════════════════════════════════════════════
    // MATH: Only contracts matching skill type are applied
    //       V<t> ignores contracts for other skill types
    //
    // PLAIN ENGLISH: "You have a separate score for each skill you 
    // practice." Batch only applies matching skill contracts.
    // ═══════════════════════════════════════════════════════════════
    
    let initial_trust = Signed::zero();
    let skill_1 = SkillType::new(1);
    let skill_2 = SkillType::new(2);
    
    let mut contracts: [Contract; MAX_HISTORY] = [Contract::empty(); MAX_HISTORY];
    
    // Skill 1: +1.0
    contracts[0] = Contract {
        counterparty: AgentId::new(2),
        skill_type: skill_1,
        stake: 1000,
        difficulty: 5,
        outcome_offset: 200,
        completed_at: 1000,
        weight: PRECISION,
    };
    
    // Skill 2: +1.0 (should be ignored)
    contracts[1] = Contract {
        counterparty: AgentId::new(3),
        skill_type: skill_2,
        stake: 1000,
        difficulty: 5,
        outcome_offset: 200,
        completed_at: 1001,
        weight: PRECISION,
    };
    
    // Skill 1: +1.0
    contracts[2] = Contract {
        counterparty: AgentId::new(4),
        skill_type: skill_1,
        stake: 1000,
        difficulty: 5,
        outcome_offset: 200,
        completed_at: 1002,
        weight: PRECISION,
    };
    
    let result = batch_update_trust(initial_trust, contracts, 3, skill_1);
    
    // Only skill 1 contracts: 0 + 1.0 + 1.0 = 2.0
    assert(result.magnitude == 2 * PRECISION);
    assert(!result.is_negative);
}

#[test]
fn test_batch_update_trust_respects_count() {
    // ═══════════════════════════════════════════════════════════════
    // MATH: Only first 'count' contracts are processed
    //
    // PLAIN ENGLISH: "Number of contracts to process." Contracts 
    // beyond count are ignored.
    // ═══════════════════════════════════════════════════════════════
    
    let initial_trust = Signed::zero();
    let skill = SkillType::new(1);
    
    let mut contracts: [Contract; MAX_HISTORY] = [Contract::empty(); MAX_HISTORY];
    
    // First two: +1.0 each
    contracts[0] = Contract {
        counterparty: AgentId::new(2),
        skill_type: skill,
        stake: 1000,
        difficulty: 5,
        outcome_offset: 200,
        completed_at: 1000,
        weight: PRECISION,
    };
    
    contracts[1] = Contract {
        counterparty: AgentId::new(3),
        skill_type: skill,
        stake: 1000,
        difficulty: 5,
        outcome_offset: 200,
        completed_at: 1001,
        weight: PRECISION,
    };
    
    // Third: should be ignored (count=2)
    contracts[2] = Contract {
        counterparty: AgentId::new(4),
        skill_type: skill,
        stake: 1000,
        difficulty: 5,
        outcome_offset: 200,
        completed_at: 1002,
        weight: 100 * PRECISION,  // Large weight - would be obvious if included
    };
    
    let result = batch_update_trust(initial_trust, contracts, 2, skill);
    
    // Only first 2: 0 + 1.0 + 1.0 = 2.0
    assert(result.magnitude == 2 * PRECISION);
}

#[test]
fn test_batch_update_trust_with_initial_value() {
    // ═══════════════════════════════════════════════════════════════
    // MATH: V<t>^(n+k) = V<t>^(n) + Σ contributions
    //       Starting from non-zero initial trust
    //
    // PLAIN ENGLISH: "Applies multiple contract contributions to an 
    // initial trust value."
    // ═══════════════════════════════════════════════════════════════
    
    let initial_trust = Signed::from_positive(10 * PRECISION);  // 10.0
    let skill = SkillType::new(1);
    
    let mut contracts: [Contract; MAX_HISTORY] = [Contract::empty(); MAX_HISTORY];
    
    // +2.0
    contracts[0] = Contract {
        counterparty: AgentId::new(2),
        skill_type: skill,
        stake: 1000,
        difficulty: 5,
        outcome_offset: 200,
        completed_at: 1000,
        weight: 2 * PRECISION,
    };
    
    // -1.0
    contracts[1] = Contract {
        counterparty: AgentId::new(3),
        skill_type: skill,
        stake: 1000,
        difficulty: 5,
        outcome_offset: 0,
        completed_at: 1001,
        weight: PRECISION,
    };
    
    let result = batch_update_trust(initial_trust, contracts, 2, skill);
    
    // 10.0 + 2.0 - 1.0 = 11.0
    assert(result.magnitude == 11 * PRECISION);
    assert(!result.is_negative);
}
```

---

## Summary of Test Coverage

| Test Category | Count | Math Equation |
|---------------|-------|---------------|
| HistoryState | 1 | Initial state |
| add_to_history | 6 | `h<t>^(n+1) = h<t>^(n) ∪ {c<n>}` |
| update_trust | 8 | `V<t>^(n+1) = V<t>^(n) + ω(c) · outcome(c)` |
| verify_trust_update | 7 | Verification of updates |
| batch_update_trust | 5 | Batch processing |

**Total: 27 tests**

---

## Equations Referenced

| # | Equation | Plain English |
|---|----------|---------------|
| 10 | `h<t>^(n+1) = h<t>^(n) ∪ {c<n>}` | History grows with each contract |
| 11 | `V<t>^(n+1) = V<t>^(n) + ω(c<n>) · outcome(c<n>)` | Trust updates incrementally |

---

## Properties Verified

1. **Append-only history**: Contracts are added, never removed
2. **Incremental trust**: Trust updates are additive
3. **Sign transitions**: Trust can cross zero in either direction
4. **Partial outcomes**: Fractional outcomes produce fractional changes
5. **Verification correctness**: Invalid updates are rejected
6. **Batch processing**: Multiple contracts processed correctly
7. **Skill filtering**: Only matching skill types are processed
8. **Count parameter**: Only specified contracts are processed
