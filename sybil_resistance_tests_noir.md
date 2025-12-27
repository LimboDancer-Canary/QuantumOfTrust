# Sybil Resistance Tests (Noir)

These tests verify the Sybil resistance mechanisms that make it economically
irrational to split activity across multiple fake identities.

---

## Tests

```noir
// ============================================
// SYBIL RESISTANCE TESTS
// ============================================

// --------------------------------------------
// prove_history_size Tests
// --------------------------------------------

#[test]
fn test_prove_history_size_meets_minimum() {
    // ═══════════════════════════════════════════════════════════════
    // MATH: |h<t>(a<honest>)| > |h<t>(a<sybil>)|
    //
    // PLAIN ENGLISH: "The size of an honest Avatar's history is 
    // greater than the size of any individual sybil Avatar's history."
    // An agent with enough contracts passes the minimum check.
    // ═══════════════════════════════════════════════════════════════
    
    let agent = AgentId::new(1);
    let skill = SkillType::new(1);
    let mut history = AgentHistory::empty(agent);
    
    // Add 5 contracts
    for i in 0..5 {
        history.contracts[i] = Contract {
            counterparty: AgentId::new(100 + i as Field),
            skill_type: skill,
            stake: 1000,
            difficulty: 5,
            outcome_offset: 200,
            completed_at: 1000 + i as Field,
            weight: PRECISION,
        };
    }
    history.count = 5;
    
    // Require minimum 5 contracts - should pass
    let result = prove_history_size(1, 5, history);
    assert(result);
}

#[test]
fn test_prove_history_size_exceeds_minimum() {
    // ═══════════════════════════════════════════════════════════════
    // MATH: |h<t>(a)| ≥ minimum
    //
    // PLAIN ENGLISH: "Honest Alice completes 100 contracts."
    // Having more than minimum contracts passes easily.
    // ═══════════════════════════════════════════════════════════════
    
    let agent = AgentId::new(1);
    let skill = SkillType::new(1);
    let mut history = AgentHistory::empty(agent);
    
    // Add 10 contracts
    for i in 0..10 {
        history.contracts[i] = Contract {
            counterparty: AgentId::new(100 + i as Field),
            skill_type: skill,
            stake: 1000,
            difficulty: 5,
            outcome_offset: 200,
            completed_at: 1000 + i as Field,
            weight: PRECISION,
        };
    }
    history.count = 10;
    
    // Require minimum 5 contracts - should pass (10 >= 5)
    let result = prove_history_size(1, 5, history);
    assert(result);
}

#[test]
fn test_prove_history_size_below_minimum() {
    // ═══════════════════════════════════════════════════════════════
    // MATH: |h<t>(a<sybil>)| < minimum
    //
    // PLAIN ENGLISH: "Sybil Bob splits across 5 fake Avatars, each 
    // completes 20 contracts." Each sybil has fewer contracts than 
    // an honest agent would.
    // ═══════════════════════════════════════════════════════════════
    
    let agent = AgentId::new(1);
    let skill = SkillType::new(1);
    let mut history = AgentHistory::empty(agent);
    
    // Add only 3 contracts (sybil has thin history)
    for i in 0..3 {
        history.contracts[i] = Contract {
            counterparty: AgentId::new(100 + i as Field),
            skill_type: skill,
            stake: 1000,
            difficulty: 5,
            outcome_offset: 200,
            completed_at: 1000 + i as Field,
            weight: PRECISION,
        };
    }
    history.count = 3;
    
    // Require minimum 5 contracts - should fail (3 < 5)
    let result = prove_history_size(1, 5, history);
    assert(!result);
}

#[test]
fn test_prove_history_size_empty_history() {
    // ═══════════════════════════════════════════════════════════════
    // MATH: |h<t>(a)| = 0 < minimum
    //
    // PLAIN ENGLISH: "A trust value of zero means no track record."
    // Empty history fails any non-zero minimum.
    // ═══════════════════════════════════════════════════════════════
    
    let agent = AgentId::new(1);
    let history = AgentHistory::empty(agent);
    
    // Require minimum 1 contract - should fail (0 < 1)
    let result = prove_history_size(1, 1, history);
    assert(!result);
}

#[test]
fn test_prove_history_size_zero_minimum() {
    // ═══════════════════════════════════════════════════════════════
    // MATH: |h<t>(a)| ≥ 0 (always true)
    //
    // PLAIN ENGLISH: "Zero minimum is always satisfied."
    // Edge case: any history meets a zero requirement.
    // ═══════════════════════════════════════════════════════════════
    
    let agent = AgentId::new(1);
    let history = AgentHistory::empty(agent);
    
    // Require minimum 0 contracts - should pass (0 >= 0)
    let result = prove_history_size(1, 0, history);
    assert(result);
}

#[test]
fn test_prove_history_size_filters_by_skill() {
    // ═══════════════════════════════════════════════════════════════
    // MATH: |h<t>(a)| counts only contracts for skill type t
    //
    // PLAIN ENGLISH: "You have a separate score for each skill you 
    // practice." Only matching skill contracts are counted.
    // ═══════════════════════════════════════════════════════════════
    
    let agent = AgentId::new(1);
    let skill_1 = SkillType::new(1);
    let skill_2 = SkillType::new(2);
    let mut history = AgentHistory::empty(agent);
    
    // Add 2 contracts for skill 1
    history.contracts[0] = Contract {
        counterparty: AgentId::new(100),
        skill_type: skill_1,
        stake: 1000,
        difficulty: 5,
        outcome_offset: 200,
        completed_at: 1000,
        weight: PRECISION,
    };
    history.contracts[1] = Contract {
        counterparty: AgentId::new(101),
        skill_type: skill_1,
        stake: 1000,
        difficulty: 5,
        outcome_offset: 200,
        completed_at: 1001,
        weight: PRECISION,
    };
    
    // Add 3 contracts for skill 2 (should be ignored)
    history.contracts[2] = Contract {
        counterparty: AgentId::new(102),
        skill_type: skill_2,
        stake: 1000,
        difficulty: 5,
        outcome_offset: 200,
        completed_at: 1002,
        weight: PRECISION,
    };
    history.contracts[3] = Contract {
        counterparty: AgentId::new(103),
        skill_type: skill_2,
        stake: 1000,
        difficulty: 5,
        outcome_offset: 200,
        completed_at: 1003,
        weight: PRECISION,
    };
    history.contracts[4] = Contract {
        counterparty: AgentId::new(104),
        skill_type: skill_2,
        stake: 1000,
        difficulty: 5,
        outcome_offset: 200,
        completed_at: 1004,
        weight: PRECISION,
    };
    history.count = 5;
    
    // Check skill 1: only 2 contracts, require 3 - should fail
    let result = prove_history_size(1, 3, history);
    assert(!result);
    
    // Check skill 2: 3 contracts, require 3 - should pass
    let result2 = prove_history_size(2, 3, history);
    assert(result2);
}

#[test]
fn test_prove_history_size_ignores_inactive_contracts() {
    // ═══════════════════════════════════════════════════════════════
    // MATH: Only active contracts (weight > 0) are counted
    //
    // PLAIN ENGLISH: "Unused slots have weight=0." Inactive contracts
    // in the array are not counted.
    // ═══════════════════════════════════════════════════════════════
    
    let agent = AgentId::new(1);
    let skill = SkillType::new(1);
    let mut history = AgentHistory::empty(agent);
    
    // Add 2 active contracts
    history.contracts[0] = Contract {
        counterparty: AgentId::new(100),
        skill_type: skill,
        stake: 1000,
        difficulty: 5,
        outcome_offset: 200,
        completed_at: 1000,
        weight: PRECISION,
    };
    history.contracts[1] = Contract {
        counterparty: AgentId::new(101),
        skill_type: skill,
        stake: 1000,
        difficulty: 5,
        outcome_offset: 200,
        completed_at: 1001,
        weight: PRECISION,
    };
    
    // Slot 2 is inactive (weight = 0, from empty())
    // history.contracts[2] remains Contract::empty()
    
    history.count = 3;  // Count says 3, but only 2 are active
    
    // Require 3 contracts - should fail (only 2 active)
    let result = prove_history_size(1, 3, history);
    assert(!result);
}

// --------------------------------------------
// prove_history_depth Tests
// --------------------------------------------

#[test]
fn test_prove_history_depth_meets_minimum() {
    // ═══════════════════════════════════════════════════════════════
    // MATH: age(oldest_contract) ≥ minimum_age_days
    //
    // PLAIN ENGLISH: "Sybil identities created recently have shallow 
    // histories." An established agent with old contracts passes.
    // ═══════════════════════════════════════════════════════════════
    
    let agent = AgentId::new(1);
    let skill = SkillType::new(1);
    let mut history = AgentHistory::empty(agent);
    
    // Contract from 100 days ago
    history.contracts[0] = Contract {
        counterparty: AgentId::new(100),
        skill_type: skill,
        stake: 1000,
        difficulty: 5,
        outcome_offset: 200,
        completed_at: 0,  // 100 days ago (current = 100)
        weight: PRECISION,
    };
    history.count = 1;
    
    // Current time = 100, require 90 days - should pass (100 >= 90)
    let result = prove_history_depth(1, 90, 100, history);
    assert(result);
}

#[test]
fn test_prove_history_depth_exceeds_minimum() {
    // ═══════════════════════════════════════════════════════════════
    // MATH: age(oldest_contract) > minimum_age_days
    //
    // PLAIN ENGLISH: "Legitimate agents interact with many different 
    // counterparties" over extended time periods.
    // ═══════════════════════════════════════════════════════════════
    
    let agent = AgentId::new(1);
    let skill = SkillType::new(1);
    let mut history = AgentHistory::empty(agent);
    
    // Oldest contract from 365 days ago
    history.contracts[0] = Contract {
        counterparty: AgentId::new(100),
        skill_type: skill,
        stake: 1000,
        difficulty: 5,
        outcome_offset: 200,
        completed_at: 0,  // 365 days ago
        weight: PRECISION,
    };
    // Recent contract
    history.contracts[1] = Contract {
        counterparty: AgentId::new(101),
        skill_type: skill,
        stake: 1000,
        difficulty: 5,
        outcome_offset: 200,
        completed_at: 360,  // 5 days ago
        weight: PRECISION,
    };
    history.count = 2;
    
    // Current time = 365, require 30 days - should pass (365 >= 30)
    let result = prove_history_depth(1, 30, 365, history);
    assert(result);
}

#[test]
fn test_prove_history_depth_below_minimum() {
    // ═══════════════════════════════════════════════════════════════
    // MATH: age(oldest_contract) < minimum_age_days
    //
    // PLAIN ENGLISH: "Sybil identities created recently have shallow 
    // histories." A new sybil identity fails the age check.
    // ═══════════════════════════════════════════════════════════════
    
    let agent = AgentId::new(1);
    let skill = SkillType::new(1);
    let mut history = AgentHistory::empty(agent);
    
    // Contract from only 10 days ago
    history.contracts[0] = Contract {
        counterparty: AgentId::new(100),
        skill_type: skill,
        stake: 1000,
        difficulty: 5,
        outcome_offset: 200,
        completed_at: 90,  // 10 days ago (current = 100)
        weight: PRECISION,
    };
    history.count = 1;
    
    // Current time = 100, require 30 days - should fail (10 < 30)
    let result = prove_history_depth(1, 30, 100, history);
    assert(!result);
}

#[test]
fn test_prove_history_depth_empty_history() {
    // ═══════════════════════════════════════════════════════════════
    // MATH: No contracts means no age to measure
    //
    // PLAIN ENGLISH: "If no contracts, cannot meet age requirement."
    // Empty history fails any age check.
    // ═══════════════════════════════════════════════════════════════
    
    let agent = AgentId::new(1);
    let history = AgentHistory::empty(agent);
    
    // Current time = 100, require 0 days - should still fail (no contracts)
    let result = prove_history_depth(1, 0, 100, history);
    assert(!result);
}

#[test]
fn test_prove_history_depth_finds_oldest() {
    // ═══════════════════════════════════════════════════════════════
    // MATH: age = current_timestamp - min(completed_at)
    //
    // PLAIN ENGLISH: "This check ensures the agent has contracts 
    // spanning a minimum time period." Finds the oldest contract.
    // ═══════════════════════════════════════════════════════════════
    
    let agent = AgentId::new(1);
    let skill = SkillType::new(1);
    let mut history = AgentHistory::empty(agent);
    
    // Middle contract (50 days ago)
    history.contracts[0] = Contract {
        counterparty: AgentId::new(100),
        skill_type: skill,
        stake: 1000,
        difficulty: 5,
        outcome_offset: 200,
        completed_at: 50,
        weight: PRECISION,
    };
    // Oldest contract (100 days ago) - not first in array
    history.contracts[1] = Contract {
        counterparty: AgentId::new(101),
        skill_type: skill,
        stake: 1000,
        difficulty: 5,
        outcome_offset: 200,
        completed_at: 0,  // Oldest
        weight: PRECISION,
    };
    // Recent contract (10 days ago)
    history.contracts[2] = Contract {
        counterparty: AgentId::new(102),
        skill_type: skill,
        stake: 1000,
        difficulty: 5,
        outcome_offset: 200,
        completed_at: 90,
        weight: PRECISION,
    };
    history.count = 3;
    
    // Current time = 100, require 90 days - should pass
    // Oldest is at timestamp 0, age = 100 - 0 = 100 >= 90
    let result = prove_history_depth(1, 90, 100, history);
    assert(result);
}

#[test]
fn test_prove_history_depth_filters_by_skill() {
    // ═══════════════════════════════════════════════════════════════
    // MATH: Only contracts for skill type t are considered
    //
    // PLAIN ENGLISH: "You have a separate score for each skill."
    // Old contracts for wrong skill don't count.
    // ═══════════════════════════════════════════════════════════════
    
    let agent = AgentId::new(1);
    let skill_1 = SkillType::new(1);
    let skill_2 = SkillType::new(2);
    let mut history = AgentHistory::empty(agent);
    
    // Old contract for skill 2 (100 days ago)
    history.contracts[0] = Contract {
        counterparty: AgentId::new(100),
        skill_type: skill_2,
        stake: 1000,
        difficulty: 5,
        outcome_offset: 200,
        completed_at: 0,
        weight: PRECISION,
    };
    // Recent contract for skill 1 (10 days ago)
    history.contracts[1] = Contract {
        counterparty: AgentId::new(101),
        skill_type: skill_1,
        stake: 1000,
        difficulty: 5,
        outcome_offset: 200,
        completed_at: 90,
        weight: PRECISION,
    };
    history.count = 2;
    
    // Check skill 1: oldest is 10 days, require 30 - should fail
    let result = prove_history_depth(1, 30, 100, history);
    assert(!result);
    
    // Check skill 2: oldest is 100 days, require 30 - should pass
    let result2 = prove_history_depth(2, 30, 100, history);
    assert(result2);
}

#[test]
fn test_prove_history_depth_exactly_at_boundary() {
    // ═══════════════════════════════════════════════════════════════
    // MATH: age == minimum_age_days (boundary case)
    //
    // PLAIN ENGLISH: "Exactly at the minimum age passes."
    // ═══════════════════════════════════════════════════════════════
    
    let agent = AgentId::new(1);
    let skill = SkillType::new(1);
    let mut history = AgentHistory::empty(agent);
    
    // Contract from exactly 30 days ago
    history.contracts[0] = Contract {
        counterparty: AgentId::new(100),
        skill_type: skill,
        stake: 1000,
        difficulty: 5,
        outcome_offset: 200,
        completed_at: 70,  // 30 days ago (current = 100)
        weight: PRECISION,
    };
    history.count = 1;
    
    // Require exactly 30 days - should pass (30 >= 30)
    let result = prove_history_depth(1, 30, 100, history);
    assert(result);
}

// --------------------------------------------
// prove_counterparty_diversity Tests
// --------------------------------------------

#[test]
fn test_prove_counterparty_diversity_meets_minimum() {
    // ═══════════════════════════════════════════════════════════════
    // MATH: |unique_counterparties| ≥ minimum
    //
    // PLAIN ENGLISH: "Legitimate agents interact with many different 
    // counterparties." Having enough unique counterparties passes.
    // ═══════════════════════════════════════════════════════════════
    
    let agent = AgentId::new(1);
    let skill = SkillType::new(1);
    let mut history = AgentHistory::empty(agent);
    
    // 5 contracts with 5 unique counterparties
    history.contracts[0] = Contract {
        counterparty: AgentId::new(100),
        skill_type: skill,
        stake: 1000,
        difficulty: 5,
        outcome_offset: 200,
        completed_at: 1000,
        weight: PRECISION,
    };
    history.contracts[1] = Contract {
        counterparty: AgentId::new(101),
        skill_type: skill,
        stake: 1000,
        difficulty: 5,
        outcome_offset: 200,
        completed_at: 1001,
        weight: PRECISION,
    };
    history.contracts[2] = Contract {
        counterparty: AgentId::new(102),
        skill_type: skill,
        stake: 1000,
        difficulty: 5,
        outcome_offset: 200,
        completed_at: 1002,
        weight: PRECISION,
    };
    history.contracts[3] = Contract {
        counterparty: AgentId::new(103),
        skill_type: skill,
        stake: 1000,
        difficulty: 5,
        outcome_offset: 200,
        completed_at: 1003,
        weight: PRECISION,
    };
    history.contracts[4] = Contract {
        counterparty: AgentId::new(104),
        skill_type: skill,
        stake: 1000,
        difficulty: 5,
        outcome_offset: 200,
        completed_at: 1004,
        weight: PRECISION,
    };
    history.count = 5;
    
    // Require 5 unique counterparties - should pass
    let result = prove_counterparty_diversity(1, 5, history);
    assert(result);
}

#[test]
fn test_prove_counterparty_diversity_exceeds_minimum() {
    // ═══════════════════════════════════════════════════════════════
    // MATH: |unique_counterparties| > minimum
    //
    // PLAIN ENGLISH: "Legitimate agents interact with many different 
    // counterparties." More unique counterparties than required.
    // ═══════════════════════════════════════════════════════════════
    
    let agent = AgentId::new(1);
    let skill = SkillType::new(1);
    let mut history = AgentHistory::empty(agent);
    
    // 5 contracts with 5 unique counterparties
    for i in 0..5 {
        history.contracts[i] = Contract {
            counterparty: AgentId::new(100 + i as Field),
            skill_type: skill,
            stake: 1000,
            difficulty: 5,
            outcome_offset: 200,
            completed_at: 1000 + i as Field,
            weight: PRECISION,
        };
    }
    history.count = 5;
    
    // Require only 3 unique counterparties - should pass (5 >= 3)
    let result = prove_counterparty_diversity(1, 3, history);
    assert(result);
}

#[test]
fn test_prove_counterparty_diversity_below_minimum() {
    // ═══════════════════════════════════════════════════════════════
    // MATH: |unique_counterparties| < minimum
    //
    // PLAIN ENGLISH: "Sybil networks often have concentrated 
    // relationships (fake identities trading with each other)."
    // ═══════════════════════════════════════════════════════════════
    
    let agent = AgentId::new(1);
    let skill = SkillType::new(1);
    let mut history = AgentHistory::empty(agent);
    
    // 5 contracts but only 2 unique counterparties
    history.contracts[0] = Contract {
        counterparty: AgentId::new(100),  // First counterparty
        skill_type: skill,
        stake: 1000,
        difficulty: 5,
        outcome_offset: 200,
        completed_at: 1000,
        weight: PRECISION,
    };
    history.contracts[1] = Contract {
        counterparty: AgentId::new(101),  // Second counterparty
        skill_type: skill,
        stake: 1000,
        difficulty: 5,
        outcome_offset: 200,
        completed_at: 1001,
        weight: PRECISION,
    };
    history.contracts[2] = Contract {
        counterparty: AgentId::new(100),  // Repeat!
        skill_type: skill,
        stake: 1000,
        difficulty: 5,
        outcome_offset: 200,
        completed_at: 1002,
        weight: PRECISION,
    };
    history.contracts[3] = Contract {
        counterparty: AgentId::new(101),  // Repeat!
        skill_type: skill,
        stake: 1000,
        difficulty: 5,
        outcome_offset: 200,
        completed_at: 1003,
        weight: PRECISION,
    };
    history.contracts[4] = Contract {
        counterparty: AgentId::new(100),  // Repeat!
        skill_type: skill,
        stake: 1000,
        difficulty: 5,
        outcome_offset: 200,
        completed_at: 1004,
        weight: PRECISION,
    };
    history.count = 5;
    
    // Require 3 unique counterparties - should fail (only 2)
    let result = prove_counterparty_diversity(1, 3, history);
    assert(!result);
}

#[test]
fn test_prove_counterparty_diversity_single_counterparty() {
    // ═══════════════════════════════════════════════════════════════
    // MATH: |unique_counterparties| = 1
    //
    // PLAIN ENGLISH: "Sybil networks often have concentrated 
    // relationships." All contracts with same counterparty is suspicious.
    // ═══════════════════════════════════════════════════════════════
    
    let agent = AgentId::new(1);
    let skill = SkillType::new(1);
    let mut history = AgentHistory::empty(agent);
    
    // 5 contracts all with same counterparty (sybil pattern)
    for i in 0..5 {
        history.contracts[i] = Contract {
            counterparty: AgentId::new(100),  // Same every time!
            skill_type: skill,
            stake: 1000,
            difficulty: 5,
            outcome_offset: 200,
            completed_at: 1000 + i as Field,
            weight: PRECISION,
        };
    }
    history.count = 5;
    
    // Require 2 unique counterparties - should fail (only 1)
    let result = prove_counterparty_diversity(1, 2, history);
    assert(!result);
}

#[test]
fn test_prove_counterparty_diversity_empty_history() {
    // ═══════════════════════════════════════════════════════════════
    // MATH: |unique_counterparties| = 0 < minimum
    //
    // PLAIN ENGLISH: "Empty history has no counterparties."
    // ═══════════════════════════════════════════════════════════════
    
    let agent = AgentId::new(1);
    let history = AgentHistory::empty(agent);
    
    // Require 1 unique counterparty - should fail (0)
    let result = prove_counterparty_diversity(1, 1, history);
    assert(!result);
}

#[test]
fn test_prove_counterparty_diversity_zero_minimum() {
    // ═══════════════════════════════════════════════════════════════
    // MATH: |unique_counterparties| ≥ 0 (always true)
    //
    // PLAIN ENGLISH: "Zero minimum is always satisfied."
    // ═══════════════════════════════════════════════════════════════
    
    let agent = AgentId::new(1);
    let history = AgentHistory::empty(agent);
    
    // Require 0 unique counterparties - should pass
    let result = prove_counterparty_diversity(1, 0, history);
    assert(result);
}

#[test]
fn test_prove_counterparty_diversity_filters_by_skill() {
    // ═══════════════════════════════════════════════════════════════
    // MATH: Only contracts for skill type t are counted
    //
    // PLAIN ENGLISH: "You have a separate score for each skill."
    // Counterparties from other skills don't count.
    // ═══════════════════════════════════════════════════════════════
    
    let agent = AgentId::new(1);
    let skill_1 = SkillType::new(1);
    let skill_2 = SkillType::new(2);
    let mut history = AgentHistory::empty(agent);
    
    // 2 contracts for skill 1 with 2 unique counterparties
    history.contracts[0] = Contract {
        counterparty: AgentId::new(100),
        skill_type: skill_1,
        stake: 1000,
        difficulty: 5,
        outcome_offset: 200,
        completed_at: 1000,
        weight: PRECISION,
    };
    history.contracts[1] = Contract {
        counterparty: AgentId::new(101),
        skill_type: skill_1,
        stake: 1000,
        difficulty: 5,
        outcome_offset: 200,
        completed_at: 1001,
        weight: PRECISION,
    };
    
    // 3 contracts for skill 2 with different counterparties
    history.contracts[2] = Contract {
        counterparty: AgentId::new(200),
        skill_type: skill_2,
        stake: 1000,
        difficulty: 5,
        outcome_offset: 200,
        completed_at: 1002,
        weight: PRECISION,
    };
    history.contracts[3] = Contract {
        counterparty: AgentId::new(201),
        skill_type: skill_2,
        stake: 1000,
        difficulty: 5,
        outcome_offset: 200,
        completed_at: 1003,
        weight: PRECISION,
    };
    history.contracts[4] = Contract {
        counterparty: AgentId::new(202),
        skill_type: skill_2,
        stake: 1000,
        difficulty: 5,
        outcome_offset: 200,
        completed_at: 1004,
        weight: PRECISION,
    };
    history.count = 5;
    
    // Check skill 1: only 2 unique, require 3 - should fail
    let result = prove_counterparty_diversity(1, 3, history);
    assert(!result);
    
    // Check skill 2: 3 unique, require 3 - should pass
    let result2 = prove_counterparty_diversity(2, 3, history);
    assert(result2);
}

#[test]
fn test_prove_counterparty_diversity_counts_correctly() {
    // ═══════════════════════════════════════════════════════════════
    // MATH: Count unique counterparties exactly
    //
    // PLAIN ENGLISH: "Legitimate agents interact with many different 
    // counterparties." Verify exact counting logic.
    // ═══════════════════════════════════════════════════════════════
    
    let agent = AgentId::new(1);
    let skill = SkillType::new(1);
    let mut history = AgentHistory::empty(agent);
    
    // 6 contracts with 3 unique counterparties (each appears twice)
    history.contracts[0] = Contract {
        counterparty: AgentId::new(100),
        skill_type: skill,
        stake: 1000,
        difficulty: 5,
        outcome_offset: 200,
        completed_at: 1000,
        weight: PRECISION,
    };
    history.contracts[1] = Contract {
        counterparty: AgentId::new(101),
        skill_type: skill,
        stake: 1000,
        difficulty: 5,
        outcome_offset: 200,
        completed_at: 1001,
        weight: PRECISION,
    };
    history.contracts[2] = Contract {
        counterparty: AgentId::new(102),
        skill_type: skill,
        stake: 1000,
        difficulty: 5,
        outcome_offset: 200,
        completed_at: 1002,
        weight: PRECISION,
    };
    history.contracts[3] = Contract {
        counterparty: AgentId::new(100),  // Repeat
        skill_type: skill,
        stake: 1000,
        difficulty: 5,
        outcome_offset: 200,
        completed_at: 1003,
        weight: PRECISION,
    };
    history.contracts[4] = Contract {
        counterparty: AgentId::new(101),  // Repeat
        skill_type: skill,
        stake: 1000,
        difficulty: 5,
        outcome_offset: 200,
        completed_at: 1004,
        weight: PRECISION,
    };
    history.contracts[5] = Contract {
        counterparty: AgentId::new(102),  // Repeat
        skill_type: skill,
        stake: 1000,
        difficulty: 5,
        outcome_offset: 200,
        completed_at: 1005,
        weight: PRECISION,
    };
    history.count = 6;
    
    // Require 3 unique - should pass (exactly 3)
    let result = prove_counterparty_diversity(1, 3, history);
    assert(result);
    
    // Require 4 unique - should fail (only 3)
    let result2 = prove_counterparty_diversity(1, 4, history);
    assert(!result2);
}

// --------------------------------------------
// Combined Sybil Resistance Tests
// --------------------------------------------

#[test]
fn test_sybil_all_checks_pass() {
    // ═══════════════════════════════════════════════════════════════
    // MATH: |h<t>(a<honest>)| > |h<t>(a<sybil>)|
    //
    // PLAIN ENGLISH: "The economics favor consolidation, not 
    // fragmentation." An honest agent passes all Sybil checks.
    // ═══════════════════════════════════════════════════════════════
    
    let agent = AgentId::new(1);
    let skill = SkillType::new(1);
    let mut history = AgentHistory::empty(agent);
    
    // 10 contracts over 100 days with 10 unique counterparties
    for i in 0..10 {
        history.contracts[i] = Contract {
            counterparty: AgentId::new(100 + i as Field),
            skill_type: skill,
            stake: 1000,
            difficulty: 5,
            outcome_offset: 200,
            completed_at: i as Field * 10,  // Spread over 100 days
            weight: PRECISION,
        };
    }
    history.count = 10;
    
    // All checks should pass
    let size_check = prove_history_size(1, 5, history);
    let depth_check = prove_history_depth(1, 50, 100, history);
    let diversity_check = prove_counterparty_diversity(1, 5, history);
    
    assert(size_check);
    assert(depth_check);
    assert(diversity_check);
}

#[test]
fn test_sybil_fails_size_only() {
    // ═══════════════════════════════════════════════════════════════
    // MATH: Sybil with few contracts fails size check
    //
    // PLAIN ENGLISH: "Each sybil gets roughly 1/k as many contracts."
    // ═══════════════════════════════════════════════════════════════
    
    let agent = AgentId::new(1);
    let skill = SkillType::new(1);
    let mut history = AgentHistory::empty(agent);
    
    // Only 2 contracts but old and diverse
    history.contracts[0] = Contract {
        counterparty: AgentId::new(100),
        skill_type: skill,
        stake: 1000,
        difficulty: 5,
        outcome_offset: 200,
        completed_at: 0,  // Old
        weight: PRECISION,
    };
    history.contracts[1] = Contract {
        counterparty: AgentId::new(101),  // Unique
        skill_type: skill,
        stake: 1000,
        difficulty: 5,
        outcome_offset: 200,
        completed_at: 50,
        weight: PRECISION,
    };
    history.count = 2;
    
    // Fails size (2 < 5), passes depth and diversity
    let size_check = prove_history_size(1, 5, history);
    let depth_check = prove_history_depth(1, 50, 100, history);
    let diversity_check = prove_counterparty_diversity(1, 2, history);
    
    assert(!size_check);  // Fails
    assert(depth_check);
    assert(diversity_check);
}

#[test]
fn test_sybil_fails_depth_only() {
    // ═══════════════════════════════════════════════════════════════
    // MATH: New sybil with many contracts fails depth check
    //
    // PLAIN ENGLISH: "Sybil identities created recently have shallow 
    // histories."
    // ═══════════════════════════════════════════════════════════════
    
    let agent = AgentId::new(1);
    let skill = SkillType::new(1);
    let mut history = AgentHistory::empty(agent);
    
    // 5 contracts but all recent (burst activity)
    for i in 0..5 {
        history.contracts[i] = Contract {
            counterparty: AgentId::new(100 + i as Field),
            skill_type: skill,
            stake: 1000,
            difficulty: 5,
            outcome_offset: 200,
            completed_at: 95 + i as Field,  // All in last 10 days
            weight: PRECISION,
        };
    }
    history.count = 5;
    
    // Passes size and diversity, fails depth
    let size_check = prove_history_size(1, 5, history);
    let depth_check = prove_history_depth(1, 30, 100, history);
    let diversity_check = prove_counterparty_diversity(1, 5, history);
    
    assert(size_check);
    assert(!depth_check);  // Fails
    assert(diversity_check);
}

#[test]
fn test_sybil_fails_diversity_only() {
    // ═══════════════════════════════════════════════════════════════
    // MATH: Concentrated relationships indicate Sybil network
    //
    // PLAIN ENGLISH: "Sybil networks often have concentrated 
    // relationships (fake identities trading with each other)."
    // ═══════════════════════════════════════════════════════════════
    
    let agent = AgentId::new(1);
    let skill = SkillType::new(1);
    let mut history = AgentHistory::empty(agent);
    
    // 5 contracts over time but only 2 counterparties
    history.contracts[0] = Contract {
        counterparty: AgentId::new(100),
        skill_type: skill,
        stake: 1000,
        difficulty: 5,
        outcome_offset: 200,
        completed_at: 0,
        weight: PRECISION,
    };
    history.contracts[1] = Contract {
        counterparty: AgentId::new(101),
        skill_type: skill,
        stake: 1000,
        difficulty: 5,
        outcome_offset: 200,
        completed_at: 25,
        weight: PRECISION,
    };
    history.contracts[2] = Contract {
        counterparty: AgentId::new(100),  // Repeat
        skill_type: skill,
        stake: 1000,
        difficulty: 5,
        outcome_offset: 200,
        completed_at: 50,
        weight: PRECISION,
    };
    history.contracts[3] = Contract {
        counterparty: AgentId::new(101),  // Repeat
        skill_type: skill,
        stake: 1000,
        difficulty: 5,
        outcome_offset: 200,
        completed_at: 75,
        weight: PRECISION,
    };
    history.contracts[4] = Contract {
        counterparty: AgentId::new(100),  // Repeat
        skill_type: skill,
        stake: 1000,
        difficulty: 5,
        outcome_offset: 200,
        completed_at: 90,
        weight: PRECISION,
    };
    history.count = 5;
    
    // Passes size and depth, fails diversity
    let size_check = prove_history_size(1, 5, history);
    let depth_check = prove_history_depth(1, 50, 100, history);
    let diversity_check = prove_counterparty_diversity(1, 3, history);
    
    assert(size_check);
    assert(depth_check);
    assert(!diversity_check);  // Fails
}
```

---

## Summary of Test Coverage

| Test Category | Count | What It Tests |
|---------------|-------|---------------|
| prove_history_size | 7 | Minimum contract count |
| prove_history_depth | 7 | Minimum history age |
| prove_counterparty_diversity | 8 | Unique counterparties |
| Combined Sybil checks | 4 | All checks together |

**Total: 26 tests**

---

## Equation Referenced

| # | Equation | Plain English |
|---|----------|---------------|
| 13 | `\|h<t>(a<honest>)\| > \|h<t>(a<sybil>)\| ∀i` | Honest history size exceeds any sybil's |

---

## Properties Verified

1. **History size**: Count of contracts for a skill type
2. **History depth**: Age of oldest contract (temporal span)
3. **Counterparty diversity**: Count of unique counterparties
4. **Skill filtering**: All checks respect skill type boundaries
5. **Empty history handling**: Fails appropriately for empty histories
6. **Boundary conditions**: Exact equality at thresholds
7. **Combined attacks**: Sybils can fail on any dimension

---

## Why Sybils Fail

| Attack Pattern | Detection Mechanism |
|----------------|---------------------|
| Split activity across k identities | Each gets ~1/k contracts (fails size) |
| Create new identities recently | Shallow history (fails depth) |
| Fake identities trade with each other | Concentrated relationships (fails diversity) |
