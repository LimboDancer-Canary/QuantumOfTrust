# Skill Type Independence Tests (Noir)

These tests verify the fundamental property that trust is computed independently per skill type.
Each skill type has its own separate trust score that is unaffected by contracts in other skills.

---

## Tests

```noir
// ============================================
// SKILL TYPE INDEPENDENCE TESTS
// ============================================

// --------------------------------------------
// Basic Independence Tests
// --------------------------------------------

#[test]
fn test_skill_type_independence_different_skills_isolated() {
    // ═══════════════════════════════════════════════════════════════
    // MATH: V<t>(Agent(t, h<t>)) = Σ ω(c) · outcome(c) for c ∈ h<t>
    //       where h<t> contains only contracts matching skill type t
    //
    // PLAIN ENGLISH: "You have a separate score for each skill you 
    // practice. V<engineering> measures engineering trust, while 
    // V<design> measures design trust—they're completely independent."
    //
    // This test verifies that a contract in skill type 1 does NOT
    // affect the trust calculation for skill type 2.
    // ═══════════════════════════════════════════════════════════════
    
    let mut contracts: [Contract; MAX_HISTORY] = [Contract::empty(); MAX_HISTORY];
    
    // Contract in skill type 1 (engineering) with positive outcome
    contracts[0] = Contract {
        counterparty: AgentId::new(2),
        skill_type: SkillType::new(1),  // Skill type 1
        stake: 1000,
        difficulty: 5,
        outcome_offset: 200,  // +100 outcome (success)
        completed_at: 1000,
        weight: PRECISION,
    };
    
    let history = AgentHistory {
        contracts,
        count: 1,
        agent_id: AgentId::new(1),
    };
    
    // Trust for skill type 1 should be positive
    let skill_1 = SkillType::new(1);
    let trust_1 = compute_trust_value(history, skill_1);
    assert(trust_1.is_positive());
    assert(trust_1.magnitude == PRECISION);
    
    // Trust for skill type 2 should be ZERO (no contracts in this skill)
    let skill_2 = SkillType::new(2);
    let trust_2 = compute_trust_value(history, skill_2);
    assert(trust_2.magnitude == 0);
    assert(!trust_2.is_negative);
}

#[test]
fn test_skill_type_independence_positive_in_one_negative_in_another() {
    // ═══════════════════════════════════════════════════════════════
    // MATH: V<t1> and V<t2> are computed independently
    //       V<t1> > 0 does not imply V<t2> > 0
    //
    // PLAIN ENGLISH: "An Avatar with V<t> = -50 is worse than an 
    // unknown newcomer—they've proven they can't deliver." An agent
    // can be trusted in one skill while being actively distrusted 
    // in another.
    // ═══════════════════════════════════════════════════════════════
    
    let mut contracts: [Contract; MAX_HISTORY] = [Contract::empty(); MAX_HISTORY];
    
    // Success in skill type 1 (engineering)
    contracts[0] = Contract {
        counterparty: AgentId::new(2),
        skill_type: SkillType::new(1),
        stake: 1000,
        difficulty: 5,
        outcome_offset: 200,  // +100 (success)
        completed_at: 1000,
        weight: PRECISION,
    };
    
    // Failure in skill type 2 (design)
    contracts[1] = Contract {
        counterparty: AgentId::new(3),
        skill_type: SkillType::new(2),
        stake: 1000,
        difficulty: 5,
        outcome_offset: 0,    // -100 (failure)
        completed_at: 1001,
        weight: PRECISION,
    };
    
    let history = AgentHistory {
        contracts,
        count: 2,
        agent_id: AgentId::new(1),
    };
    
    // Skill 1: positive trust
    let skill_1 = SkillType::new(1);
    let trust_1 = compute_trust_value(history, skill_1);
    assert(trust_1.is_positive());
    assert(trust_1.magnitude == PRECISION);
    
    // Skill 2: negative trust (completely independent!)
    let skill_2 = SkillType::new(2);
    let trust_2 = compute_trust_value(history, skill_2);
    assert(trust_2.is_negative);
    assert(trust_2.magnitude == PRECISION);
}

#[test]
fn test_skill_type_independence_multiple_contracts_same_skill() {
    // ═══════════════════════════════════════════════════════════════
    // MATH: V<t>(Agent(t, h<t>)) = Σ ω(c) · outcome(c) for c ∈ h<t>
    //       Sum includes ALL contracts matching skill type t
    //
    // PLAIN ENGLISH: "An Agent's trust value equals the sum of 
    // (weight × outcome) for each contract in their history."
    // Multiple contracts in the same skill accumulate.
    // ═══════════════════════════════════════════════════════════════
    
    let mut contracts: [Contract; MAX_HISTORY] = [Contract::empty(); MAX_HISTORY];
    
    // Two successes in skill type 1
    contracts[0] = Contract {
        counterparty: AgentId::new(2),
        skill_type: SkillType::new(1),
        stake: 1000,
        difficulty: 5,
        outcome_offset: 200,  // +100
        completed_at: 1000,
        weight: PRECISION,
    };
    
    contracts[1] = Contract {
        counterparty: AgentId::new(3),
        skill_type: SkillType::new(1),
        stake: 1000,
        difficulty: 5,
        outcome_offset: 200,  // +100
        completed_at: 1001,
        weight: PRECISION,
    };
    
    // One contract in skill type 2 (should not affect skill 1)
    contracts[2] = Contract {
        counterparty: AgentId::new(4),
        skill_type: SkillType::new(2),
        stake: 1000,
        difficulty: 5,
        outcome_offset: 0,    // -100 (failure)
        completed_at: 1002,
        weight: PRECISION,
    };
    
    let history = AgentHistory {
        contracts,
        count: 3,
        agent_id: AgentId::new(1),
    };
    
    // Skill 1: trust = 1.0 + 1.0 = 2.0 (two successes)
    let skill_1 = SkillType::new(1);
    let trust_1 = compute_trust_value(history, skill_1);
    assert(trust_1.is_positive());
    assert(trust_1.magnitude == 2 * PRECISION);
    
    // Skill 2: trust = -1.0 (one failure, unaffected by skill 1)
    let skill_2 = SkillType::new(2);
    let trust_2 = compute_trust_value(history, skill_2);
    assert(trust_2.is_negative);
    assert(trust_2.magnitude == PRECISION);
}

#[test]
fn test_skill_type_independence_query_nonexistent_skill() {
    // ═══════════════════════════════════════════════════════════════
    // MATH: V<t> = 0 when h<t> = ∅ (empty history for skill t)
    //
    // PLAIN ENGLISH: "A trust value of zero means no track record 
    // yet. A newcomer. Might deserve a chance on small contracts."
    // Querying a skill with no contracts returns zero trust.
    // ═══════════════════════════════════════════════════════════════
    
    let mut contracts: [Contract; MAX_HISTORY] = [Contract::empty(); MAX_HISTORY];
    
    // Contracts only in skill types 1 and 2
    contracts[0] = Contract {
        counterparty: AgentId::new(2),
        skill_type: SkillType::new(1),
        stake: 1000,
        difficulty: 5,
        outcome_offset: 200,
        completed_at: 1000,
        weight: PRECISION,
    };
    
    contracts[1] = Contract {
        counterparty: AgentId::new(3),
        skill_type: SkillType::new(2),
        stake: 1000,
        difficulty: 5,
        outcome_offset: 200,
        completed_at: 1001,
        weight: PRECISION,
    };
    
    let history = AgentHistory {
        contracts,
        count: 2,
        agent_id: AgentId::new(1),
    };
    
    // Query skill type 99 (no contracts exist)
    let skill_99 = SkillType::new(99);
    let trust_99 = compute_trust_value(history, skill_99);
    
    // Should return zero (unknown, no track record)
    assert(trust_99.magnitude == 0);
    assert(!trust_99.is_negative);
}

// --------------------------------------------
// Mixed History Tests
// --------------------------------------------

#[test]
fn test_skill_type_independence_mixed_outcomes_per_skill() {
    // ═══════════════════════════════════════════════════════════════
    // MATH: V<t> = Σ ω(c) · outcome(c) for c ∈ h<t>
    //       Each skill type sums its own contracts independently
    //
    // PLAIN ENGLISH: "For each contract: A successful contract with 
    // high weight adds a lot to your trust. A failed contract with 
    // high weight subtracts a lot from your trust."
    // ═══════════════════════════════════════════════════════════════
    
    let mut contracts: [Contract; MAX_HISTORY] = [Contract::empty(); MAX_HISTORY];
    
    // Skill 1: +100, +100, -100 = net +1.0
    contracts[0] = Contract {
        counterparty: AgentId::new(2),
        skill_type: SkillType::new(1),
        stake: 1000,
        difficulty: 5,
        outcome_offset: 200,  // +100
        completed_at: 1000,
        weight: PRECISION,
    };
    contracts[1] = Contract {
        counterparty: AgentId::new(3),
        skill_type: SkillType::new(1),
        stake: 1000,
        difficulty: 5,
        outcome_offset: 200,  // +100
        completed_at: 1001,
        weight: PRECISION,
    };
    contracts[2] = Contract {
        counterparty: AgentId::new(4),
        skill_type: SkillType::new(1),
        stake: 1000,
        difficulty: 5,
        outcome_offset: 0,    // -100
        completed_at: 1002,
        weight: PRECISION,
    };
    
    // Skill 2: -100, -100, +100 = net -1.0
    contracts[3] = Contract {
        counterparty: AgentId::new(5),
        skill_type: SkillType::new(2),
        stake: 1000,
        difficulty: 5,
        outcome_offset: 0,    // -100
        completed_at: 1003,
        weight: PRECISION,
    };
    contracts[4] = Contract {
        counterparty: AgentId::new(6),
        skill_type: SkillType::new(2),
        stake: 1000,
        difficulty: 5,
        outcome_offset: 0,    // -100
        completed_at: 1004,
        weight: PRECISION,
    };
    contracts[5] = Contract {
        counterparty: AgentId::new(7),
        skill_type: SkillType::new(2),
        stake: 1000,
        difficulty: 5,
        outcome_offset: 200,  // +100
        completed_at: 1005,
        weight: PRECISION,
    };
    
    let history = AgentHistory {
        contracts,
        count: 6,
        agent_id: AgentId::new(1),
    };
    
    // Skill 1: +1 +1 -1 = +1.0
    let skill_1 = SkillType::new(1);
    let trust_1 = compute_trust_value(history, skill_1);
    assert(trust_1.is_positive());
    assert(trust_1.magnitude == PRECISION);
    
    // Skill 2: -1 -1 +1 = -1.0
    let skill_2 = SkillType::new(2);
    let trust_2 = compute_trust_value(history, skill_2);
    assert(trust_2.is_negative);
    assert(trust_2.magnitude == PRECISION);
}

#[test]
fn test_skill_type_independence_many_skills() {
    // ═══════════════════════════════════════════════════════════════
    // MATH: V<t1>, V<t2>, V<t3>, ... are all independent
    //
    // PLAIN ENGLISH: "The subscript 't' indicates this is scoped to 
    // a specific skill type. So V<engineering> measures engineering 
    // trust, while V<design> measures design trust."
    // An agent can have different trust levels across many skills.
    // ═══════════════════════════════════════════════════════════════
    
    let mut contracts: [Contract; MAX_HISTORY] = [Contract::empty(); MAX_HISTORY];
    
    // Skill 1: success (+1.0)
    contracts[0] = Contract {
        counterparty: AgentId::new(2),
        skill_type: SkillType::new(1),
        stake: 1000,
        difficulty: 5,
        outcome_offset: 200,
        completed_at: 1000,
        weight: PRECISION,
    };
    
    // Skill 2: failure (-1.0)
    contracts[1] = Contract {
        counterparty: AgentId::new(3),
        skill_type: SkillType::new(2),
        stake: 1000,
        difficulty: 5,
        outcome_offset: 0,
        completed_at: 1001,
        weight: PRECISION,
    };
    
    // Skill 3: partial success (+0.5)
    contracts[2] = Contract {
        counterparty: AgentId::new(4),
        skill_type: SkillType::new(3),
        stake: 1000,
        difficulty: 5,
        outcome_offset: 150,  // +50
        completed_at: 1002,
        weight: PRECISION,
    };
    
    // Skill 4: partial failure (-0.5)
    contracts[3] = Contract {
        counterparty: AgentId::new(5),
        skill_type: SkillType::new(4),
        stake: 1000,
        difficulty: 5,
        outcome_offset: 50,   // -50
        completed_at: 1003,
        weight: PRECISION,
    };
    
    let history = AgentHistory {
        contracts,
        count: 4,
        agent_id: AgentId::new(1),
    };
    
    // Verify each skill has independent trust
    let skill_1 = SkillType::new(1);
    let trust_1 = compute_trust_value(history, skill_1);
    assert(trust_1.is_positive());
    assert(trust_1.magnitude == PRECISION);  // +1.0
    
    let skill_2 = SkillType::new(2);
    let trust_2 = compute_trust_value(history, skill_2);
    assert(trust_2.is_negative);
    assert(trust_2.magnitude == PRECISION);  // -1.0
    
    let skill_3 = SkillType::new(3);
    let trust_3 = compute_trust_value(history, skill_3);
    assert(trust_3.is_positive());
    assert(trust_3.magnitude == PRECISION / 2);  // +0.5
    
    let skill_4 = SkillType::new(4);
    let trust_4 = compute_trust_value(history, skill_4);
    assert(trust_4.is_negative);
    assert(trust_4.magnitude == PRECISION / 2);  // -0.5
    
    // Skill 5: no contracts, should be zero
    let skill_5 = SkillType::new(5);
    let trust_5 = compute_trust_value(history, skill_5);
    assert(trust_5.magnitude == 0);
}

// --------------------------------------------
// Edge Cases
// --------------------------------------------

#[test]
fn test_skill_type_independence_zero_skill_id() {
    // ═══════════════════════════════════════════════════════════════
    // MATH: Skill type 0 is a valid skill identifier
    //
    // PLAIN ENGLISH: "Skill types are Field values representing 
    // categories of work." Skill ID 0 should work like any other.
    // ═══════════════════════════════════════════════════════════════
    
    let mut contracts: [Contract; MAX_HISTORY] = [Contract::empty(); MAX_HISTORY];
    
    // Contract in skill type 0
    contracts[0] = Contract {
        counterparty: AgentId::new(2),
        skill_type: SkillType::new(0),  // Skill type 0
        stake: 1000,
        difficulty: 5,
        outcome_offset: 200,
        completed_at: 1000,
        weight: PRECISION,
    };
    
    // Contract in skill type 1
    contracts[1] = Contract {
        counterparty: AgentId::new(3),
        skill_type: SkillType::new(1),
        stake: 1000,
        difficulty: 5,
        outcome_offset: 0,
        completed_at: 1001,
        weight: PRECISION,
    };
    
    let history = AgentHistory {
        contracts,
        count: 2,
        agent_id: AgentId::new(1),
    };
    
    // Skill 0: positive
    let skill_0 = SkillType::new(0);
    let trust_0 = compute_trust_value(history, skill_0);
    assert(trust_0.is_positive());
    assert(trust_0.magnitude == PRECISION);
    
    // Skill 1: negative (independent from skill 0)
    let skill_1 = SkillType::new(1);
    let trust_1 = compute_trust_value(history, skill_1);
    assert(trust_1.is_negative);
    assert(trust_1.magnitude == PRECISION);
}

#[test]
fn test_skill_type_independence_large_skill_id() {
    // ═══════════════════════════════════════════════════════════════
    // MATH: Skill types are Field values (can be large numbers)
    //
    // PLAIN ENGLISH: "In practice, skill type is a hash of the skill 
    // type string computed off-circuit." Large hash values should
    // work correctly as skill identifiers.
    // ═══════════════════════════════════════════════════════════════
    
    let mut contracts: [Contract; MAX_HISTORY] = [Contract::empty(); MAX_HISTORY];
    
    // Contract with large skill type ID (simulating a hash)
    let large_skill_id: Field = 123456789012345;
    
    contracts[0] = Contract {
        counterparty: AgentId::new(2),
        skill_type: SkillType::new(large_skill_id),
        stake: 1000,
        difficulty: 5,
        outcome_offset: 200,
        completed_at: 1000,
        weight: PRECISION,
    };
    
    let history = AgentHistory {
        contracts,
        count: 1,
        agent_id: AgentId::new(1),
    };
    
    // Should find the contract with matching large skill ID
    let skill_large = SkillType::new(large_skill_id);
    let trust_large = compute_trust_value(history, skill_large);
    assert(trust_large.is_positive());
    assert(trust_large.magnitude == PRECISION);
    
    // Different large ID should return zero
    let skill_other = SkillType::new(large_skill_id + 1);
    let trust_other = compute_trust_value(history, skill_other);
    assert(trust_other.magnitude == 0);
}

#[test]
fn test_skill_type_independence_same_counterparty_different_skills() {
    // ═══════════════════════════════════════════════════════════════
    // MATH: c = (a<provider>, a<consumer>, t, s, d, τ)
    //       Contracts with same counterparty but different skill 
    //       types are still independent.
    //
    // PLAIN ENGLISH: "A contract is defined by six things including 
    // what skill type it involves." The same counterparty can appear
    // in contracts for different skills without cross-contamination.
    // ═══════════════════════════════════════════════════════════════
    
    let mut contracts: [Contract; MAX_HISTORY] = [Contract::empty(); MAX_HISTORY];
    
    // Same counterparty (AgentId 2), different skills, different outcomes
    contracts[0] = Contract {
        counterparty: AgentId::new(2),  // Same counterparty
        skill_type: SkillType::new(1),
        stake: 1000,
        difficulty: 5,
        outcome_offset: 200,  // Success in skill 1
        completed_at: 1000,
        weight: PRECISION,
    };
    
    contracts[1] = Contract {
        counterparty: AgentId::new(2),  // Same counterparty
        skill_type: SkillType::new(2),
        stake: 1000,
        difficulty: 5,
        outcome_offset: 0,    // Failure in skill 2
        completed_at: 1001,
        weight: PRECISION,
    };
    
    let history = AgentHistory {
        contracts,
        count: 2,
        agent_id: AgentId::new(1),
    };
    
    // Skills remain independent despite same counterparty
    let skill_1 = SkillType::new(1);
    let trust_1 = compute_trust_value(history, skill_1);
    assert(trust_1.is_positive());
    
    let skill_2 = SkillType::new(2);
    let trust_2 = compute_trust_value(history, skill_2);
    assert(trust_2.is_negative);
}

// --------------------------------------------
// Weighted Independence Tests
// --------------------------------------------

#[test]
fn test_skill_type_independence_different_weights_per_skill() {
    // ═══════════════════════════════════════════════════════════════
    // MATH: V<t> = Σ ω(c) · outcome(c) for c ∈ h<t>
    //       Weights are applied independently per skill type.
    //
    // PLAIN ENGLISH: "The weight of a contract determines how much 
    // it should count." High-weight contracts in one skill don't 
    // affect low-weight calculations in another skill.
    // ═══════════════════════════════════════════════════════════════
    
    let mut contracts: [Contract; MAX_HISTORY] = [Contract::empty(); MAX_HISTORY];
    
    // Skill 1: high weight success
    contracts[0] = Contract {
        counterparty: AgentId::new(2),
        skill_type: SkillType::new(1),
        stake: 10000,
        difficulty: 10,
        outcome_offset: 200,
        completed_at: 1000,
        weight: 5 * PRECISION,  // High weight: 5.0
    };
    
    // Skill 2: low weight failure
    contracts[1] = Contract {
        counterparty: AgentId::new(3),
        skill_type: SkillType::new(2),
        stake: 100,
        difficulty: 1,
        outcome_offset: 0,
        completed_at: 1001,
        weight: PRECISION / 2,  // Low weight: 0.5
    };
    
    let history = AgentHistory {
        contracts,
        count: 2,
        agent_id: AgentId::new(1),
    };
    
    // Skill 1: trust = 5.0 × 1.0 = 5.0
    let skill_1 = SkillType::new(1);
    let trust_1 = compute_trust_value(history, skill_1);
    assert(trust_1.is_positive());
    assert(trust_1.magnitude == 5 * PRECISION);
    
    // Skill 2: trust = 0.5 × (-1.0) = -0.5
    let skill_2 = SkillType::new(2);
    let trust_2 = compute_trust_value(history, skill_2);
    assert(trust_2.is_negative);
    assert(trust_2.magnitude == PRECISION / 2);
}

#[test]
fn test_skill_type_independence_partial_outcomes() {
    // ═══════════════════════════════════════════════════════════════
    // MATH: outcome(c) ∈ [-1, 1] (continuous range)
    //       V<t> = Σ ω(c) · outcome(c)
    //
    // PLAIN ENGLISH: "This continuous range allows for nuance. +0.5 
    // means good but not perfect. -0.5 means problematic but some 
    // value delivered."
    // ═══════════════════════════════════════════════════════════════
    
    let mut contracts: [Contract; MAX_HISTORY] = [Contract::empty(); MAX_HISTORY];
    
    // Skill 1: +75% outcome
    contracts[0] = Contract {
        counterparty: AgentId::new(2),
        skill_type: SkillType::new(1),
        stake: 1000,
        difficulty: 5,
        outcome_offset: 175,  // +75
        completed_at: 1000,
        weight: PRECISION,
    };
    
    // Skill 2: -25% outcome
    contracts[1] = Contract {
        counterparty: AgentId::new(3),
        skill_type: SkillType::new(2),
        stake: 1000,
        difficulty: 5,
        outcome_offset: 75,   // -25
        completed_at: 1001,
        weight: PRECISION,
    };
    
    let history = AgentHistory {
        contracts,
        count: 2,
        agent_id: AgentId::new(1),
    };
    
    // Skill 1: trust = 1.0 × 0.75 = 0.75
    let skill_1 = SkillType::new(1);
    let trust_1 = compute_trust_value(history, skill_1);
    assert(trust_1.is_positive());
    assert(trust_1.magnitude == (75 * PRECISION) / 100);
    
    // Skill 2: trust = 1.0 × (-0.25) = -0.25
    let skill_2 = SkillType::new(2);
    let trust_2 = compute_trust_value(history, skill_2);
    assert(trust_2.is_negative);
    assert(trust_2.magnitude == (25 * PRECISION) / 100);
}

// --------------------------------------------
// Empty History Edge Cases
// --------------------------------------------

#[test]
fn test_skill_type_independence_empty_history_all_skills_zero() {
    // ═══════════════════════════════════════════════════════════════
    // MATH: V<t> = 0 when h<t> = ∅ for all t
    //
    // PLAIN ENGLISH: "A trust value of zero means no track record 
    // yet." With no contracts at all, every skill type returns zero.
    // ═══════════════════════════════════════════════════════════════
    
    let empty_history = AgentHistory {
        contracts: [Contract::empty(); MAX_HISTORY],
        count: 0,
        agent_id: AgentId::new(1),
    };
    
    // All skill types should return zero
    let skill_0 = SkillType::new(0);
    let trust_0 = compute_trust_value(empty_history, skill_0);
    assert(trust_0.magnitude == 0);
    
    let skill_1 = SkillType::new(1);
    let trust_1 = compute_trust_value(empty_history, skill_1);
    assert(trust_1.magnitude == 0);
    
    let skill_99 = SkillType::new(99);
    let trust_99 = compute_trust_value(empty_history, skill_99);
    assert(trust_99.magnitude == 0);
}

#[test]
fn test_skill_type_independence_neutral_outcomes_dont_affect_other_skills() {
    // ═══════════════════════════════════════════════════════════════
    // MATH: outcome = 0 contributes nothing to V<t>
    //       V<t> = Σ ω(c) · 0 = 0 for neutral outcomes
    //
    // PLAIN ENGLISH: "0 is neutral—partial delivery, or cancelled 
    // by mutual agreement." Neutral outcomes in one skill don't 
    // affect any other skill.
    // ═══════════════════════════════════════════════════════════════
    
    let mut contracts: [Contract; MAX_HISTORY] = [Contract::empty(); MAX_HISTORY];
    
    // Skill 1: neutral outcome
    contracts[0] = Contract {
        counterparty: AgentId::new(2),
        skill_type: SkillType::new(1),
        stake: 1000,
        difficulty: 5,
        outcome_offset: 100,  // 0 (neutral)
        completed_at: 1000,
        weight: PRECISION,
    };
    
    // Skill 2: positive outcome
    contracts[1] = Contract {
        counterparty: AgentId::new(3),
        skill_type: SkillType::new(2),
        stake: 1000,
        difficulty: 5,
        outcome_offset: 200,  // +100
        completed_at: 1001,
        weight: PRECISION,
    };
    
    let history = AgentHistory {
        contracts,
        count: 2,
        agent_id: AgentId::new(1),
    };
    
    // Skill 1: trust = 0 (neutral contributes nothing)
    let skill_1 = SkillType::new(1);
    let trust_1 = compute_trust_value(history, skill_1);
    assert(trust_1.magnitude == 0);
    
    // Skill 2: trust = +1.0 (unaffected by skill 1's neutral)
    let skill_2 = SkillType::new(2);
    let trust_2 = compute_trust_value(history, skill_2);
    assert(trust_2.is_positive());
    assert(trust_2.magnitude == PRECISION);
}
```

---

## Summary of Test Coverage

| Test Category | Count | Math Equation |
|---------------|-------|---------------|
| Basic Independence | 4 | `V<t>(Agent(t, h<t>)) = Σ ω(c) · outcome(c)` |
| Mixed History | 2 | `V<t1>, V<t2>, ... independent` |
| Edge Cases | 4 | Skill ID 0, large IDs, same counterparty |
| Weighted Independence | 2 | `ω(c) · outcome(c)` per skill |
| Empty History | 2 | `V<t> = 0` when `h<t> = ∅` |

**Total: 14 tests**

---

## Equations Referenced

| # | Equation | Plain English |
|---|----------|---------------|
| 2 | `V<t>: q<T> → ℝ` | Trust value function takes a q<T> and produces a real number |
| 3 | `V<t> = 0` (unknown), `V<t> > 0` (trusted), `V<t> < 0` (distrusted) | Trust value meanings |
| 4 | `V<t>(Agent(t, h<t>)) = Σ ω(c) · outcome(c)` | Trust = sum of weighted outcomes |
| 5 | `c = (a<provider>, a<consumer>, t, s, d, τ)` | Contract structure |

---

## Properties Verified

1. **Skill isolation**: Contracts in skill A don't affect trust in skill B
2. **Independent signs**: Positive trust in one skill, negative in another
3. **Zero for unknown**: No contracts in a skill returns zero trust
4. **Accumulation per skill**: Multiple contracts in same skill sum correctly
5. **Weight independence**: High weights in one skill don't leak to others
6. **Partial outcome independence**: Fractional outcomes stay in their skill
7. **Edge case handling**: Skill ID 0, large IDs, empty history
