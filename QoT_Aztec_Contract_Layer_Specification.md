# QoT Aztec Contract Layer Specification

## Smart Contract Architecture for Quantum of Trust on Aztec

---

## Executive Summary

This document specifies the Aztec smart contract layer for the Quantum of Trust framework. The contracts provide coordination infrastructure: state storage, escrow management, lifecycle enforcement, and trust accumulation.

### The Foundational Principle

**The Avatar is the privacy layer.**

The QoT framework decouples trust from identity through Avatars — persistent pseudonyms that accumulate reputation through verified action. The network sees only Avatars; the humans behind them remain private.

This means:

| Always Private | Public at Avatar Level |
|----------------|------------------------|
| Human identity behind Avatar | Avatar addresses |
| Links between Avatars (same human) | Avatar activity (listings, contracts) |
| Private keys | Trust values per skill type |
| | Contract history (Avatar-to-Avatar) |
| | Stakes and outcomes |

Avatar activity is **meant** to be visible. That's how reputation works. When Avatar 0xABC posts a listing, the network learns "Avatar 0xABC wants Engineering work done" — not "Jane Smith wants Engineering work done."

### Three Contracts

| Contract | Responsibility | State Model |
|----------|----------------|-------------|
| **QoTRegistry** | Projects and contract listings | Public |
| **QoTEscrow** | Stake management, lifecycle, outcomes | Public |
| **QoTAvatar** | Trust computation, eligibility proofs | Hybrid |

---

## Table of Contents

1. [The Avatar Privacy Model](#the-avatar-privacy-model)
2. [Governance Model](#governance-model)
3. [Protocol Constants](#protocol-constants)
4. [Circuit Library Dependency](#circuit-library-dependency)
5. [QoTRegistry Contract](#qotregistry-contract)
6. [QoTEscrow Contract](#qotescrow-contract)
7. [QoTAvatar Contract](#qotavatar-contract)
8. [Interaction Flows](#interaction-flows)
9. [Privacy Boundaries](#privacy-boundaries)
10. [Error Handling](#error-handling)
11. [Open Questions](#open-questions)

---

## The Avatar Privacy Model

### From the Whitepaper

> "The network only 'sees' Avatars. The network has no concept of 'human.' It only knows Avatars with skill types, histories, and trust quotients. What's behind the Avatar—human directly, human via AI, consortium of humans, whatever—is opaque to the network by design."

The Avatar is not a mask over identity. It is the **fundamental mode of participation**. Even users who operate under their real names do so through an Avatar that happens to bear that name.

### What This Means for Smart Contracts

**Public on-chain (Avatar level):**
- Avatar posts listing → visible
- Avatar accepts contract → visible
- Avatar's stake amount → visible
- Contract outcome → visible
- Avatar's trust quotient → visible (or efficiently provable)

**Private (human level):**
- Which human controls Avatar 0xABC → private
- That Avatar 0xABC and 0xDEF are controlled by the same human → private
- The private keys → private

### Why Avatar Activity Must Be Visible

Reputation requires observable history. From the whitepaper:

> "Trust value is computed by a function... For an Agent, value derives from contract history within a specific skill context."

If contract history were hidden, trust couldn't be computed or verified. The entire framework depends on Avatars having visible track records.

### The Role of Zero-Knowledge Proofs

ZK proofs in QoT serve **efficiency**, not Avatar-level privacy:

| Use Case | What ZK Proves | Why ZK (not because it's secret) |
|----------|----------------|----------------------------------|
| Eligibility check | "My trust ≥ threshold" | Avoid scanning full history on-chain |
| Sybil compliance | "My history meets variance/velocity requirements" | Complex computation done off-chain |
| DAO membership | "I'm a member of DAO X" | Efficient verification |

A verifier *could* compute these by scanning the Avatar's public history. ZK proofs make verification efficient, not private.

---

## Governance Model

### Design Philosophy

QoT follows a **governance-minimized** design: protocol rules are encoded in immutable smart contracts. Changes require deploying new contract versions and voluntary user migration.

| Aspect | Approach | Rationale |
|--------|----------|-----------|
| **Skill types** | Permissionless (raw Field values) | Social convention, not protocol authority |
| **Stake minimums** | Per-listing (customer decides) | Market determines appropriate stakes |
| **Timeouts** | Per-listing within hardcoded bounds | Flexibility with safety rails |
| **Sybil parameters** | Hardcoded floor + per-project stricter-only | Security baseline with customization |
| **Contract upgrades** | Immutable deploy, voluntary migration | No admin keys to compromise |
| **Admin keys** | **None** | No capture vector |

### Skill Types Are Just Numbers

The blockchain doesn't need to know what "Engineering" means. Skill types are raw `Field` values:

```
skill_type = 1          // "Engineering" by convention
skill_type = 1000001    // Reserved: Customer commitment
skill_type = 0xDEADBEEF // Someone's custom skill type
```

Front-ends and indexers maintain human-readable mappings. Protocol doesn't validate or restrict.

### Project-Level Sybil Parameters

Sybil resistance parameters have a **hardcoded floor** but can be **tightened per-project**:

```
Protocol Floor (hardcoded, immutable)
         │
         ▼
    ┌─────────┐
    │ Project │ ← Customer can tighten here
    └────┬────┘
         │ (inherited by all subcontracts)
         ▼
   ┌─────┴─────┐
   │  Phases   │
   └─────┬─────┘
         │
         ▼
   ┌─────┴─────┐
   │   Tasks   │
   └───────────┘
```

Parameters can only be made **stricter**, never looser than protocol floor.

---

## Protocol Constants

### Hardcoded Immutable Values

```noir
// =============================================================================
// SYBIL RESISTANCE FLOOR (minimum security, can only be tightened)
// =============================================================================

/// Counterparty weight sigmoid spread
/// Lower = stricter (sharper discrimination against low-trust counterparties)
global FLOOR_LAMBDA: Field = 50;

/// Maximum full-weight contracts per velocity period
/// Lower = stricter (fewer contracts count at full weight)
global FLOOR_VELOCITY_ALLOWANCE: Field = 10;

/// Minimum velocity measurement window (blocks, ~7 days)
/// Higher = stricter (same allowance spread over longer period)
global FLOOR_VELOCITY_PERIOD: Field = 50400;

/// Minimum outcome variance threshold (scaled by PRECISION)
/// Higher = stricter (more variance required in history)
global FLOOR_VARIANCE_THRESHOLD: Field = 100;  // 0.1 scaled

/// Minimum contracts before variance check applies
/// Lower = stricter (variance checked earlier)
global FLOOR_VARIANCE_MIN_HISTORY: Field = 10;


// =============================================================================
// TIMEOUT BOUNDS
// =============================================================================

/// Minimum timeout period (~30 minutes)
global MIN_TIMEOUT_BLOCKS: Field = 100;

/// Maximum timeout period (~6 months)
global MAX_TIMEOUT_BLOCKS: Field = 500000;


// =============================================================================
// ARITHMETIC CONSTANTS
// =============================================================================

/// Precision for fixed-point arithmetic (10^6)
global PRECISION: Field = 1000000;

/// Maximum outcome value (+1.0 scaled)
global OUTCOME_MAX: Field = 1000000;

/// Maximum weight value
global MAX_WEIGHT: Field = 10000000;

/// Maximum difficulty rating (10.0 scaled)
global MAX_DIFFICULTY: Field = 10000000;


// =============================================================================
// ARRAY BOUNDS (ZK circuit constraints)
// =============================================================================

/// Maximum contracts in history for proof generation
global MAX_HISTORY: u32 = 1000;

/// Maximum phases per project
global MAX_PHASES: u32 = 20;

/// Maximum tasks per phase
global MAX_TASKS: u32 = 100;

/// History size below which on-chain eligibility check is used
/// Above this threshold, ZK proof required for efficiency
global SMALL_HISTORY_THRESHOLD: Field = 50;

/// Age at which contracts can be pruned (~7 years at 12s blocks)
/// Contracts older than this have zero trust potency due to recency decay
global PRUNING_THRESHOLD_BLOCKS: Field = 18396000;

/// Grace period for other party to convert unilateral to mutual cancellation (~24 hours)
global CANCELLATION_GRACE_PERIOD: Field = 7200;

/// Reserved skill type for customer commitment tracking
global SKILL_TYPE_CUSTOMER_COMMITMENT: Field = 1000001;
```

### Sybil Parameters Struct

```noir
/// Sybil resistance parameters for a project or standalone contract.
/// All values must be stricter-or-equal to protocol floor.
struct SybilParameters {
    lambda: Field,
    velocity_allowance: Field,
    velocity_period: Field,
    variance_threshold: Field,
    variance_min_history: Field,
}

impl SybilParameters {
    /// Returns protocol floor parameters (default, least strict allowed)
    fn protocol_floor() -> Self {
        SybilParameters {
            lambda: FLOOR_LAMBDA,
            velocity_allowance: FLOOR_VELOCITY_ALLOWANCE,
            velocity_period: FLOOR_VELOCITY_PERIOD,
            variance_threshold: FLOOR_VARIANCE_THRESHOLD,
            variance_min_history: FLOOR_VARIANCE_MIN_HISTORY,
        }
    }
    
    /// Validates that all parameters are stricter-or-equal to floor
    fn is_valid(self) -> bool {
        (self.lambda <= FLOOR_LAMBDA) &
        (self.velocity_allowance <= FLOOR_VELOCITY_ALLOWANCE) &
        (self.velocity_period >= FLOOR_VELOCITY_PERIOD) &
        (self.variance_threshold >= FLOOR_VARIANCE_THRESHOLD) &
        (self.variance_min_history <= FLOOR_VARIANCE_MIN_HISTORY)
    }
    
    /// Check if self is stricter-or-equal to other
    fn is_stricter_or_equal(self, other: SybilParameters) -> bool {
        (self.lambda <= other.lambda) &
        (self.velocity_allowance <= other.velocity_allowance) &
        (self.velocity_period >= other.velocity_period) &
        (self.variance_threshold >= other.variance_threshold) &
        (self.variance_min_history <= other.variance_min_history)
    }
}
```

---

## Circuit Library Dependency

### Two Layers of Noir Code

The QoT implementation consists of two distinct layers:

| Layer | File(s) | Purpose |
|-------|---------|---------|
| **Coordination** | `QoTRegistry.nr`, `QoTEscrow.nr`, `QoTAvatar.nr` | Smart contracts — state storage, escrow, lifecycle |
| **Computation** | `qot_circuits.nr` | Circuit library — trust math, weights, Sybil checks |

The smart contracts import and call functions from the circuit library. The library handles the mathematical computations defined in the research documents.

### Research Documents vs. Production Library

Two research documents contain circuit implementations:

| Document | Contents |
|----------|----------|
| `Quantum_of_Trust_Equations_in_Noir.md` | Core trust math: `Signed` arithmetic, `compute_trust_value()`, eligibility proofs, DAO aggregation, math approximations |
| `Sybil_Resistance_Circuits_Noir.md` | Defense mechanisms: counterparty weighting, velocity limits, variance checks, enhanced eligibility |

These are **research libraries** — they document the mathematical translations and contain tested implementations. The production library (`qot_circuits.nr`) consolidates and organizes these into a single importable module.

### Production Library: `qot_circuits.nr`

The contracts import from a single library file:

```noir
use dep::qot_circuits::{
    // Signed arithmetic
    Signed,
    
    // Trust computation
    compute_trust_value,
    compute_trust_contribution,
    
    // Weight computation  
    compute_weight,
    compute_weight_with_recency,
    verify_weight_bounds,
    compute_cancellation_weight,
    
    // Eligibility
    calculate_threshold,
    verify_eligibility,
    prove_eligibility,  // ZK version
    
    // Sybil resistance
    SybilParameters,
    check_sybil_compliance,
    compute_counterparty_factor,
    compute_velocity_weight,
    compute_outcome_variance,
    count_unique_counterparties,
    count_recent_contracts,
    prove_plausible_history,
    
    // Math primitives
    approx_log1p,
    approx_sigmoid,
    fp_mul,
    ratio,
    
    // Validation
    is_valid_outcome,
    
    // Constants (re-exported)
    PRECISION,
    MAX_DIFFICULTY,
    OUTCOME_MAX,
};
```

### Functions Called by Contracts

| Contract | Function Called | Source |
|----------|-----------------|--------|
| **QoTAvatar** | | |
| | `compute_trust_value()` | Core trust computation |
| | `compute_weight()` | Weight from stake, difficulty, counterparty |
| | `compute_weight_with_recency()` | Weight including time decay |
| | `compute_trust_contribution()` | Single contract's trust delta |
| | `verify_eligibility()` | Public state eligibility check |
| | `prove_eligibility()` | ZK proof generation |
| | `check_sybil_compliance()` | All Sybil checks combined |
| | `compute_outcome_variance()` | For variance Sybil check |
| | `count_unique_counterparties()` | For diversity Sybil check |
| | `count_recent_contracts()` | For velocity Sybil check |
| | `compute_cancellation_weight()` | Weight for cancellation trust impact |
| | `Signed` arithmetic | All trust value operations |
| **QoTEscrow** | | |
| | `is_valid_outcome()` | Validate outcome in range |
| **QoTRegistry** | | |
| | `SybilParameters.is_valid()` | Validate against protocol floor |
| | `SybilParameters.is_stricter_or_equal()` | Compare parameter sets |

### What the Library Must Provide

Based on contract requirements, `qot_circuits.nr` must export:

**Types:**
```noir
struct Signed { magnitude: Field, is_negative: bool }
struct SybilParameters { lambda, velocity_allowance, velocity_period, variance_threshold, variance_min_history }
struct ContractHistoryEntry { ... }  // For internal computation
```

**Core Functions:**
```noir
// Trust
fn compute_trust_value(history: [...], skill_type: Field) -> Signed
fn compute_trust_contribution(weight: Field, outcome: Signed) -> Signed

// Weight  
fn compute_weight(stake: Field, difficulty: Field, counterparty_trust: Signed) -> Field
fn compute_weight_with_recency(entry: ContractHistoryEntry, current_block: Field) -> Field
fn compute_cancellation_weight(stake: Field) -> Field
fn verify_weight_bounds(weight: Field, stake: Field) -> bool

// Eligibility
fn calculate_threshold(stake: Field, difficulty: Field) -> Field
fn verify_eligibility(trust: Signed, threshold: Field, params: SybilParameters, history: [...]) -> bool
fn prove_eligibility(skill_type: pub Field, threshold: pub Field, params: pub SybilParameters, history: [...]) -> pub bool

// Sybil Resistance
fn check_sybil_compliance(history: [...], skill_type: Field, params: SybilParameters) -> bool
fn compute_counterparty_factor(counterparty_trust: Signed, lambda: Field) -> Field
fn compute_velocity_weight(rank: Field, allowance: Field, period: Field) -> Field
fn compute_outcome_variance(history: [...], skill_type: Field) -> Field
fn count_unique_counterparties(history: [...], skill_type: Field) -> Field
fn count_recent_contracts(history: [...], skill_type: Field, period: Field) -> Field
fn prove_plausible_history(history: [...], skill_type: Field, min_variance: Field) -> bool

// Validation
fn is_valid_outcome(outcome: Field) -> bool

// Math
fn approx_log1p(x: Field) -> Field
fn approx_sigmoid(x: Signed, scale: Field) -> Field
fn fp_mul(a: Field, b: Field) -> Field
fn ratio(numerator: Field, denominator: Field) -> Field
```

### Recency Decay Function

One function not fully specified in either research document: **recency decay** for the 7-year pruning threshold.

The weight function includes recency:

$$\omega(c) = f(s, d, V_t(\text{counterparty}), \text{recency})$$

For pruning, we need `compute_weight_with_recency()` that returns 0 for contracts older than 7 years:

```noir
/// Computes weight with recency decay applied.
/// Returns 0 for contracts older than PRUNING_THRESHOLD_BLOCKS.
fn compute_weight_with_recency(
    entry: ContractHistoryEntry,
    current_block: Field,
) -> Field {
    let age = current_block - entry.timestamp;
    
    // Contracts older than 7 years have zero weight
    if age >= PRUNING_THRESHOLD_BLOCKS {
        return 0;
    }
    
    // Apply recency decay curve
    // Full weight at age 0, decays to 0 at PRUNING_THRESHOLD_BLOCKS
    let decay_factor = (PRUNING_THRESHOLD_BLOCKS - age) * PRECISION / PRUNING_THRESHOLD_BLOCKS;
    
    fp_mul(entry.weight, decay_factor)
}
```

This function must be added to the circuit library.

### Library Organization

Recommended module structure for `qot_circuits.nr`:

```
qot_circuits/
├── lib.nr              # Re-exports all public items
├── signed.nr           # Signed arithmetic
├── types.nr            # SybilParameters, ContractHistoryEntry
├── math.nr             # approx_log1p, approx_sigmoid, fp_mul, ratio
├── weight.nr           # compute_weight, recency decay
├── trust.nr            # compute_trust_value, compute_trust_contribution
├── eligibility.nr      # calculate_threshold, verify_eligibility, prove_eligibility
├── sybil.nr            # All Sybil resistance functions
└── validation.nr       # is_valid_outcome, verify_weight_bounds
```

Or as a single file for simplicity during initial development.

### Gap Analysis: Research Documents vs. Contract Requirements

The following functions are **required by contracts** but **not fully implemented** in the research documents:

| Required Function | Status | Notes |
|-------------------|--------|-------|
| `compute_weight_with_recency()` | **GAP** | Recency decay exists (`approx_recency_decay`) but not combined with weight |
| `compute_cancellation_weight()` | **GAP** | New requirement for cancellation trust impact |
| `check_sybil_compliance()` | **GAP** | Individual checks exist; need combined function |
| `count_recent_contracts()` | **GAP** | Velocity checks exist but not as standalone counter |
| `is_valid_outcome()` | **GAP** | Validation logic exists in `Contract.is_valid()` but not standalone |
| `SybilParameters` struct | **GAP** | Defined in contract spec, not in circuit library |
| `verify_eligibility()` | **PARTIAL** | `prove_eligibility()` exists; need non-ZK version for small histories |

The following are **fully covered** in research documents:

| Function | Source Document |
|----------|-----------------|
| `Signed` struct + arithmetic | Equations |
| `compute_trust_value()` | Equations |
| `compute_weight()` | Equations |
| `verify_weight_bounds()` | Equations |
| `calculate_threshold()` | Equations |
| `prove_eligibility()` | Equations |
| `approx_log1p()` | Equations |
| `fp_mul()`, `ratio()` | Equations |
| `compute_counterparty_factor()` | Sybil Resistance |
| `compute_velocity_weight()` | Sybil Resistance |
| `compute_outcome_variance()` | Sybil Resistance |
| `prove_plausible_history()` | Sybil Resistance |
| `approx_sigmoid()` | Sybil Resistance |
| `count_unique_counterparties()` | Equations (as `prove_counterparty_diversity`) |

### Functions to Add to `qot_circuits.nr`

These must be implemented when creating the production library:

```noir
/// SybilParameters struct (matches contract spec)
struct SybilParameters {
    lambda: Field,
    velocity_allowance: Field,
    velocity_period: Field,
    variance_threshold: Field,
    variance_min_history: Field,
}

impl SybilParameters {
    fn protocol_floor() -> Self { ... }
    fn is_valid(self) -> bool { ... }
    fn is_stricter_or_equal(self, other: Self) -> bool { ... }
}

/// Weight with recency decay applied
fn compute_weight_with_recency(
    base_weight: Field,
    contract_timestamp: Field,
    current_block: Field,
) -> Field {
    let age = current_block - contract_timestamp;
    
    if age >= PRUNING_THRESHOLD_BLOCKS {
        return 0;
    }
    
    let decay = approx_recency_decay(age);
    fp_mul(base_weight, decay)
}

/// Weight calculation for cancellation (no counterparty factor)
fn compute_cancellation_weight(stake: Field) -> Field {
    // Cancellation weight based on stake only
    // No difficulty (cancellation isn't "work")
    // No counterparty factor (self-inflicted)
    approx_log1p(stake)
}

/// Combined Sybil compliance check
fn check_sybil_compliance(
    history: [ContractHistoryEntry; MAX_HISTORY],
    history_count: Field,
    skill_type: Field,
    params: SybilParameters,
    current_block: Field,
) -> bool {
    // 1. Velocity check
    let recent = count_recent_contracts(history, history_count, skill_type, params.velocity_period, current_block);
    let velocity_ok = recent <= params.velocity_allowance;
    
    // 2. Variance check (if enough history)
    let variance_ok = if history_count >= params.variance_min_history {
        let variance = compute_outcome_variance(history, history_count, skill_type);
        variance >= params.variance_threshold
    } else {
        true  // Exempt if insufficient history
    };
    
    // 3. Counterparty diversity is implicit in counterparty weighting
    
    velocity_ok & variance_ok
}

/// Count contracts in recent period
fn count_recent_contracts(
    history: [ContractHistoryEntry; MAX_HISTORY],
    history_count: Field,
    skill_type: Field,
    period_blocks: Field,
    current_block: Field,
) -> Field {
    let period_start = current_block - period_blocks;
    let mut count: Field = 0;
    
    for i in 0..MAX_HISTORY {
        if (i as Field) < history_count {
            let entry = history[i];
            if entry.skill_type == skill_type {
                if entry.timestamp >= period_start {
                    count = count + 1;
                }
            }
        }
    }
    
    count
}

/// Validate outcome is in range
fn is_valid_outcome(outcome: Field) -> bool {
    // Outcome stored as offset: 0 = -1.0, PRECISION = 0, 2*PRECISION = +1.0
    outcome <= (2 * PRECISION)
}

/// Non-ZK eligibility verification (for small histories)
fn verify_eligibility(
    trust: Signed,
    threshold: Field,
    history: [ContractHistoryEntry; MAX_HISTORY],
    history_count: Field,
    skill_type: Field,
    params: SybilParameters,
    current_block: Field,
) -> bool {
    // Check trust threshold
    let threshold_signed = Signed::from_positive(threshold);
    let meets_threshold = trust.gte(threshold_signed);
    
    // Check Sybil compliance
    let sybil_ok = check_sybil_compliance(history, history_count, skill_type, params, current_block);
    
    meets_threshold & sybil_ok
}
```

### Purpose

Public registry of projects and contract listings. All listing data is public at the Avatar level — this enables discovery and reputation visibility.

### State Variables

```noir
/// Project definition
struct Project {
    /// Unique project identifier
    project_id: Field,
    
    /// Project creator (customer Avatar)
    customer: AztecAddress,
    
    /// Total stake allocated to project
    total_stake: Field,
    
    /// Sybil resistance parameters (inherited by all subcontracts)
    sybil_params: SybilParameters,
    
    /// Project-level deadlines
    acceptance_deadline: Field,
    completion_deadline: Field,
    
    /// Whether project is active
    is_active: bool,
    
    /// Number of phases created
    phase_count: Field,
}

/// Contract listing (fully public)
struct ContractListing {
    /// Unique listing identifier
    listing_id: Field,
    
    /// Customer Avatar posting the listing
    customer: AztecAddress,
    
    /// Required skill type
    skill_type: Field,
    
    /// Minimum trust threshold for eligibility
    threshold: Field,
    
    /// Customer's committed stake
    customer_stake: Field,
    
    /// Required provider stake
    required_provider_stake: Field,
    
    /// Deadlines
    acceptance_deadline: Field,
    completion_deadline: Field,
    
    /// Sybil parameters (from project or standalone)
    sybil_params: SybilParameters,
    
    /// Contract type (standalone, specification, planning, implementation, task)
    contract_type: Field,
    
    /// Reference to parent project (0 for standalone)
    project_id: Field,
    
    /// Parent contract reference (for tasks within phases)
    parent_ref: Field,
    
    /// Listing state
    is_open: bool,
    
    /// Provider Avatar (set when accepted)
    provider: AztecAddress,
}
```

### Public Functions

```noir
#[aztec(public)]
fn create_project(
    project_id: Field,
    total_stake: Field,
    sybil_params: SybilParameters,
    acceptance_deadline: Field,
    completion_deadline: Field,
) {
    // Project must not already exist
    assert(!storage.projects.at(project_id).read().is_active);
    
    // Sybil params must be valid (stricter-or-equal to floor)
    assert(sybil_params.is_valid());
    
    // Deadlines must be valid
    assert(acceptance_deadline > context.block_number());
    assert(completion_deadline > acceptance_deadline);
    assert(completion_deadline - context.block_number() <= MAX_TIMEOUT_BLOCKS);
    
    // Store project
    storage.projects.at(project_id).write(Project {
        project_id,
        customer: context.msg_sender(),
        total_stake,
        sybil_params,
        acceptance_deadline,
        completion_deadline,
        is_active: true,
        phase_count: 0,
    });
    
    // Lock stake in escrow
    QoTEscrow::at(storage.escrow_contract.read()).lock_stake(
        context.msg_sender(),
        project_id,
        total_stake,
    );
    
    emit_project_created(project_id, context.msg_sender(), total_stake);
}

#[aztec(public)]
fn create_listing(
    listing_id: Field,
    skill_type: Field,
    threshold: Field,
    customer_stake: Field,
    required_provider_stake: Field,
    acceptance_deadline: Field,
    completion_deadline: Field,
    sybil_params: SybilParameters,
    contract_type: Field,
    project_id: Field,
    parent_ref: Field,
) {
    // Listing must not exist
    assert(storage.listings.at(listing_id).read().customer == AztecAddress::zero());
    
    // If part of project, validate against project params
    if project_id != 0 {
        let project = storage.projects.at(project_id).read();
        assert(project.is_active);
        assert(context.msg_sender() == project.customer);
        assert(sybil_params.is_stricter_or_equal(project.sybil_params));
        assert(acceptance_deadline <= project.acceptance_deadline);
        assert(completion_deadline <= project.completion_deadline);
    } else {
        // Standalone: validate against protocol floor
        assert(sybil_params.is_valid());
    }
    
    // Validate deadlines
    assert(acceptance_deadline > context.block_number());
    assert(completion_deadline > acceptance_deadline);
    assert(completion_deadline - context.block_number() >= MIN_TIMEOUT_BLOCKS);
    assert(completion_deadline - context.block_number() <= MAX_TIMEOUT_BLOCKS);
    
    // Store listing
    storage.listings.at(listing_id).write(ContractListing {
        listing_id,
        customer: context.msg_sender(),
        skill_type,
        threshold,
        customer_stake,
        required_provider_stake,
        acceptance_deadline,
        completion_deadline,
        sybil_params,
        contract_type,
        project_id,
        parent_ref,
        is_open: true,
        provider: AztecAddress::zero(),
    });
    
    // Lock stake in escrow (if standalone, not already locked via project)
    if project_id == 0 {
        QoTEscrow::at(storage.escrow_contract.read()).lock_stake(
            context.msg_sender(),
            listing_id,
            customer_stake,
        );
    }
    
    emit_listing_created(listing_id, context.msg_sender(), skill_type, threshold);
}

#[aztec(public)]
fn accept_listing(
    listing_id: Field,
    provider_stake: Field,
    difficulty: Field,
) {
    let mut listing = storage.listings.at(listing_id).read();
    
    // Listing must be open
    assert(listing.is_open);
    
    // Acceptance deadline not passed
    assert(context.block_number() <= listing.acceptance_deadline);
    
    // Provider stake must meet requirement
    assert(provider_stake >= listing.required_provider_stake);
    
    // Provider must not be customer
    assert(context.msg_sender() != listing.customer);
    
    // Difficulty must be valid
    assert(difficulty > 0);
    assert(difficulty <= MAX_DIFFICULTY);
    
    // Verify eligibility (trust >= threshold)
    // Hybrid approach: direct check for small histories, ZK proof for large
    let history_size = QoTAvatar::at(storage.avatar_contract.read()).get_contract_count(
        context.msg_sender(),
        listing.skill_type,
    );
    
    if history_size < SMALL_HISTORY_THRESHOLD {
        assert(QoTAvatar::at(storage.avatar_contract.read()).verify_eligibility(
            context.msg_sender(),
            listing.skill_type,
            listing.threshold,
            listing.sybil_params,
        ));
    } else {
        // For large histories, caller must provide ZK proof via separate function
        // This simplified version assumes proof was pre-verified
        assert(storage.verified_eligibility.at(context.msg_sender()).at(listing_id).read());
    }
    
    // Lock provider stake
    QoTEscrow::at(storage.escrow_contract.read()).lock_stake(
        context.msg_sender(),
        listing_id,
        provider_stake,
    );
    
    // Update listing
    listing.is_open = false;
    listing.provider = context.msg_sender();
    storage.listings.at(listing_id).write(listing);
    
    // Create active contract in escrow (difficulty committed here)
    QoTEscrow::at(storage.escrow_contract.read()).create_contract(
        listing_id,
        listing.customer,
        context.msg_sender(),
        listing.skill_type,
        listing.customer_stake,
        provider_stake,
        difficulty,
        listing.completion_deadline,
        listing.sybil_params,
    );
    
    emit_listing_accepted(listing_id, context.msg_sender(), provider_stake, difficulty);
}

#[aztec(public)]
fn cancel_listing(listing_id: Field) {
    let mut listing = storage.listings.at(listing_id).read();
    
    // Only customer can cancel
    assert(context.msg_sender() == listing.customer);
    
    // Must still be open (not accepted)
    assert(listing.is_open);
    
    // Close listing
    listing.is_open = false;
    storage.listings.at(listing_id).write(listing);
    
    // Release customer stake (if standalone)
    if listing.project_id == 0 {
        QoTEscrow::at(storage.escrow_contract.read()).release_stake(
            listing.customer,
            listing_id,
            listing.customer_stake,
        );
    }
    
    emit_listing_cancelled(listing_id);
}
```

### View Functions

```noir
#[aztec(public)]
fn get_project(project_id: Field) -> Project {
    storage.projects.at(project_id).read()
}

#[aztec(public)]
fn get_listing(listing_id: Field) -> ContractListing {
    storage.listings.at(listing_id).read()
}

#[aztec(public)]
fn is_listing_open(listing_id: Field) -> bool {
    let listing = storage.listings.at(listing_id).read();
    listing.is_open & (context.block_number() <= listing.acceptance_deadline)
}
```

---

## QoTEscrow Contract

### Purpose

Manages stakes and contract lifecycle. All state is public at the Avatar level — observers can see which Avatars have stakes locked and contract outcomes.

### State Variables

```noir
/// Active contract (created when listing is accepted)
struct ActiveContract {
    /// Unique contract identifier (same as listing_id)
    contract_id: Field,
    
    /// Customer Avatar
    customer: AztecAddress,
    
    /// Provider Avatar
    provider: AztecAddress,
    
    /// Skill type
    skill_type: Field,
    
    /// Stakes
    customer_stake: Field,
    provider_stake: Field,
    
    /// Deadline
    deadline: Field,
    
    /// Sybil parameters
    sybil_params: SybilParameters,
    
    /// Contract state
    state: ContractState,
    
    /// Outcome (set when recorded)
    outcome: Field,  // Signed, scaled by PRECISION
    
    /// Difficulty (set by provider at acceptance or during execution)
    difficulty: Field,
    
    /// Timestamps
    created_at: Field,
    completed_at: Field,
}

enum ContractState {
    Active,      // Work in progress
    Completed,   // Outcome recorded, trust updated
    Cancelled,   // Mutual cancellation
    TimedOut,    // Deadline passed without outcome
}

/// Escrow entry
struct EscrowEntry {
    owner: AztecAddress,
    amount: Field,
    reference_id: Field,  // listing_id or project_id
    is_locked: bool,
}

/// Cancellation request tracking
struct CancelRequest {
    initiator: AztecAddress,
    initiated_at: Field,
}
```

### Public Functions

```noir
#[aztec(public)]
fn lock_stake(
    owner: AztecAddress,
    reference_id: Field,
    amount: Field,
) {
    // Only registry can call
    assert(context.msg_sender() == storage.registry_contract.read());
    
    // Transfer tokens to escrow
    // (Token contract integration - implementation depends on Aztec token standards)
    transfer_to_escrow(owner, amount);
    
    // Record escrow entry
    let entry_id = compute_entry_id(owner, reference_id);
    storage.escrow_entries.at(entry_id).write(EscrowEntry {
        owner,
        amount,
        reference_id,
        is_locked: true,
    });
    
    emit_stake_locked(owner, reference_id, amount);
}

#[aztec(public)]
fn create_contract(
    contract_id: Field,
    customer: AztecAddress,
    provider: AztecAddress,
    skill_type: Field,
    customer_stake: Field,
    provider_stake: Field,
    difficulty: Field,
    deadline: Field,
    sybil_params: SybilParameters,
) {
    // Only registry can call
    assert(context.msg_sender() == storage.registry_contract.read());
    
    // Contract must not exist
    assert(storage.contracts.at(contract_id).read().state == ContractState::default());
    
    storage.contracts.at(contract_id).write(ActiveContract {
        contract_id,
        customer,
        provider,
        skill_type,
        customer_stake,
        provider_stake,
        deadline,
        sybil_params,
        state: ContractState::Active,
        outcome: 0,
        difficulty,  // Set at contract creation (provider's assessment)
        created_at: context.block_number(),
        completed_at: 0,
    });
    
    emit_contract_created(contract_id, customer, provider, skill_type, difficulty);
}

#[aztec(public)]
fn record_outcome(
    contract_id: Field,
    outcome: Field,
) {
    let mut contract = storage.contracts.at(contract_id).read();
    
    // Only customer can record outcome
    assert(context.msg_sender() == contract.customer);
    
    // Contract must be active
    assert(contract.state == ContractState::Active);
    
    // Deadline must not have passed
    assert(context.block_number() <= contract.deadline);
    
    // Outcome must be in valid range [-PRECISION, +PRECISION]
    // Note: Using Field, negative values represented differently
    assert(is_valid_outcome(outcome));
    
    // Record outcome
    contract.outcome = outcome;
    contract.state = ContractState::Completed;
    contract.completed_at = context.block_number();
    storage.contracts.at(contract_id).write(contract);
    
    // Update trust for both parties
    QoTAvatar::at(storage.avatar_contract.read()).record_contract_completion(
        contract.provider,
        contract.customer,
        contract.skill_type,
        contract.provider_stake,
        contract.difficulty,
        outcome,
        contract.sybil_params,
    );
    
    // Release stakes
    release_stake(contract.customer, contract_id, contract.customer_stake);
    release_stake(contract.provider, contract_id, contract.provider_stake);
    
    emit_outcome_recorded(contract_id, outcome);
}

#[aztec(public)]
fn timeout_contract(contract_id: Field) {
    let mut contract = storage.contracts.at(contract_id).read();
    
    // Contract must be active
    assert(contract.state == ContractState::Active);
    
    // Deadline must have passed
    assert(context.block_number() > contract.deadline);
    
    // Apply timeout: provider forfeits stake to customer
    contract.state = ContractState::TimedOut;
    storage.contracts.at(contract_id).write(contract);
    
    // Release customer stake back to customer
    release_stake(contract.customer, contract_id, contract.customer_stake);
    
    // Transfer provider stake to customer (penalty)
    transfer_stake(contract.provider, contract.customer, contract_id, contract.provider_stake);
    
    emit_contract_timed_out(contract_id);
}

#[aztec(public)]
fn cancel_contract(contract_id: Field) {
    let mut contract = storage.contracts.at(contract_id).read();
    
    // Contract must be active
    assert(contract.state == ContractState::Active);
    
    let caller = context.msg_sender();
    assert(caller == contract.customer | caller == contract.provider);
    
    // Check if other party has already initiated cancellation
    let cancel_state = storage.cancel_requests.at(contract_id).read();
    
    if cancel_state.initiator != AztecAddress::zero() {
        // Other party already initiated - this is mutual cancellation
        // Both parties receive mitigated negative trust
        let mutual_outcome = -100000;  // -0.1 scaled
        
        QoTAvatar::at(storage.avatar_contract.read()).record_cancellation(
            contract.customer,
            SKILL_TYPE_CUSTOMER_COMMITMENT,
            contract.customer_stake,
            mutual_outcome,
        );
        
        QoTAvatar::at(storage.avatar_contract.read()).record_cancellation(
            contract.provider,
            contract.skill_type,
            contract.provider_stake,
            mutual_outcome,
        );
        
        // Finalize cancellation
        contract.state = ContractState::Cancelled;
        storage.contracts.at(contract_id).write(contract);
        
        // Return stakes to both parties
        release_stake(contract.customer, contract_id, contract.customer_stake);
        release_stake(contract.provider, contract_id, contract.provider_stake);
        
        emit_contract_cancelled(contract_id, true);  // mutual = true
    } else {
        // First cancellation request - unilateral
        // Cancelling party receives negative trust
        let unilateral_outcome = -300000;  // -0.3 scaled
        
        if caller == contract.customer {
            QoTAvatar::at(storage.avatar_contract.read()).record_cancellation(
                contract.customer,
                SKILL_TYPE_CUSTOMER_COMMITMENT,
                contract.customer_stake,
                unilateral_outcome,
            );
        } else {
            QoTAvatar::at(storage.avatar_contract.read()).record_cancellation(
                contract.provider,
                contract.skill_type,
                contract.provider_stake,
                unilateral_outcome,
            );
        }
        
        // Record cancellation request - give other party window to make it mutual
        storage.cancel_requests.at(contract_id).write(CancelRequest {
            initiator: caller,
            initiated_at: context.block_number(),
        });
        
        emit_cancellation_initiated(contract_id, caller);
    }
}

/// Finalize a unilateral cancellation after grace period
#[aztec(public)]
fn finalize_cancellation(contract_id: Field) {
    let contract = storage.contracts.at(contract_id).read();
    let cancel_state = storage.cancel_requests.at(contract_id).read();
    
    // Must have pending cancellation
    assert(cancel_state.initiator != AztecAddress::zero());
    
    // Grace period must have passed (e.g., 24 hours for other party to make mutual)
    assert(context.block_number() > cancel_state.initiated_at + CANCELLATION_GRACE_PERIOD);
    
    // Contract still active (not already finalized as mutual)
    assert(contract.state == ContractState::Active);
    
    // Finalize as unilateral
    let mut contract = contract;
    contract.state = ContractState::Cancelled;
    storage.contracts.at(contract_id).write(contract);
    
    // Return stakes to both parties
    release_stake(contract.customer, contract_id, contract.customer_stake);
    release_stake(contract.provider, contract_id, contract.provider_stake);
    
    emit_contract_cancelled(contract_id, false);  // mutual = false
}

#[aztec(public)]
fn release_stake(
    owner: AztecAddress,
    reference_id: Field,
    amount: Field,
) {
    let entry_id = compute_entry_id(owner, reference_id);
    let mut entry = storage.escrow_entries.at(entry_id).read();
    
    assert(entry.is_locked);
    assert(entry.amount == amount);
    
    entry.is_locked = false;
    storage.escrow_entries.at(entry_id).write(entry);
    
    // Transfer tokens back to owner
    transfer_from_escrow(owner, amount);
    
    emit_stake_released(owner, reference_id, amount);
}
```

### View Functions

```noir
#[aztec(public)]
fn get_contract(contract_id: Field) -> ActiveContract {
    storage.contracts.at(contract_id).read()
}

#[aztec(public)]
fn get_escrow_balance(owner: AztecAddress, reference_id: Field) -> Field {
    let entry_id = compute_entry_id(owner, reference_id);
    let entry = storage.escrow_entries.at(entry_id).read();
    if entry.is_locked { entry.amount } else { 0 }
}
```

---

## QoTAvatar Contract

### Purpose

Manages Avatar trust state and generates eligibility proofs. This is where the Noir circuits execute.

**Hybrid privacy model:**
- Trust values and history are **publicly visible** (can be queried)
- Eligibility proofs provide **efficient verification** (ZK proves trust ≥ threshold without scanning full history)
- History details support **incremental updates** (each contract adds to history)

### State Variables

```noir
/// Avatar trust state
struct AvatarState {
    /// Avatar address
    owner: AztecAddress,
    
    /// Trust values per skill type (public, queryable)
    /// Map: skill_type -> trust value (Signed, scaled)
    trust_values: Map<Field, Field>,
    
    /// Contract count per skill type
    contract_counts: Map<Field, Field>,
    
    /// Oldest contract timestamp (for history depth)
    oldest_contract: Field,
}

/// Contract history entry (public)
struct ContractHistoryEntry {
    /// Contract identifier
    contract_id: Field,
    
    /// Counterparty Avatar
    counterparty: AztecAddress,
    
    /// Skill type
    skill_type: Field,
    
    /// Stake
    stake: Field,
    
    /// Difficulty
    difficulty: Field,
    
    /// Outcome [-1, +1] scaled
    outcome: Field,
    
    /// Pre-computed weight
    weight: Field,
    
    /// Timestamp
    timestamp: Field,
}
```

### Public Functions

```noir
#[aztec(public)]
fn record_contract_completion(
    provider: AztecAddress,
    customer: AztecAddress,
    skill_type: Field,
    stake: Field,
    difficulty: Field,
    outcome: Field,
    sybil_params: SybilParameters,
) {
    // Only escrow can call
    assert(context.msg_sender() == storage.escrow_contract.read());
    
    // Compute weight for this contract
    let customer_trust = storage.avatar_states.at(customer).read()
        .trust_values.at(skill_type).read();
    let weight = compute_weight(stake, difficulty, customer_trust);
    
    // Compute trust contribution
    let contribution = compute_trust_contribution(weight, outcome, sybil_params);
    
    // Update provider's trust
    update_avatar_trust(provider, skill_type, contribution, context.block_number());
    
    // Record in provider's history
    let entry = ContractHistoryEntry {
        contract_id: context.transaction_id(),  // Or passed in
        counterparty: customer,
        skill_type,
        stake,
        difficulty,
        outcome,
        weight,
        timestamp: context.block_number(),
    };
    storage.contract_histories.at(provider).push(entry);
    
    // Update customer's trust (for customer skill types)
    // Customer behaviors tracked separately
    update_customer_trust(customer, provider, stake, outcome, context.block_number());
    
    emit_trust_updated(provider, skill_type, contribution);
}

#[aztec(public)]
fn verify_eligibility(
    avatar: AztecAddress,
    skill_type: Field,
    threshold: Field,
    sybil_params: SybilParameters,
) -> bool {
    let state = storage.avatar_states.at(avatar).read();
    
    // Get current trust value
    let trust = state.trust_values.at(skill_type).read();
    
    // Check threshold
    if trust < threshold {
        return false;
    }
    
    // Check Sybil resistance requirements
    if !check_sybil_compliance(avatar, skill_type, sybil_params) {
        return false;
    }
    
    true
}

#[aztec(public)]
fn get_trust_value(avatar: AztecAddress, skill_type: Field) -> Field {
    storage.avatar_states.at(avatar).read()
        .trust_values.at(skill_type).read()
}

#[aztec(public)]
fn get_contract_count(avatar: AztecAddress, skill_type: Field) -> Field {
    storage.avatar_states.at(avatar).read()
        .contract_counts.at(skill_type).read()
}

#[aztec(public)]
fn get_history_depth(avatar: AztecAddress) -> Field {
    let state = storage.avatar_states.at(avatar).read();
    if state.oldest_contract == 0 {
        0
    } else {
        context.block_number() - state.oldest_contract
    }
}
```

### Private Functions (for efficient ZK proofs)

```noir
/// Generate eligibility proof without scanning full history on-chain
/// This is more efficient than public verification for large histories
#[aztec(private)]
fn prove_eligibility(
    skill_type: pub Field,
    threshold: pub Field,
    sybil_params: pub SybilParameters,
) -> pub bool {
    // Load private state (Avatar's full history)
    let history = load_history(context.msg_sender(), skill_type);
    
    // Compute trust value from history
    let trust = compute_trust_value(history, skill_type);
    
    // Check threshold
    if !trust.gte(Signed::from_field(threshold)) {
        return false;
    }
    
    // Check Sybil compliance
    if !check_sybil_compliance_private(history, sybil_params) {
        return false;
    }
    
    true
}

/// Prove trust falls within a range (for selective disclosure)
#[aztec(private)]
fn prove_trust_range(
    skill_type: pub Field,
    min_trust: pub Field,
    max_trust: pub Field,
) -> pub bool {
    let history = load_history(context.msg_sender(), skill_type);
    let trust = compute_trust_value(history, skill_type);
    
    trust.gte(Signed::from_field(min_trust)) & 
    trust.lte(Signed::from_field(max_trust))
}

/// Prune contracts older than 7 years (zero trust potency)
#[aztec(public)]
fn prune_old_contracts(contract_ids: [Field; N]) {
    let avatar = context.msg_sender();
    
    for contract_id in contract_ids {
        let entry = storage.contract_histories.at(avatar).get(contract_id);
        
        // Must be old enough (7+ years)
        let age = context.block_number() - entry.timestamp;
        assert(age >= PRUNING_THRESHOLD_BLOCKS);
        
        // Verify weight would be zero (recency decay check)
        let weight = compute_weight_with_recency(entry, context.block_number());
        assert(weight == 0);
        
        // Remove from active history
        storage.contract_histories.at(avatar).remove(contract_id);
    }
    
    emit_history_pruned(avatar, contract_ids.len());
}

/// Record a cancellation's trust impact
#[aztec(public)]
fn record_cancellation(
    avatar: AztecAddress,
    skill_type: Field,
    stake: Field,
    outcome: Field,
) {
    // Only escrow can call
    assert(context.msg_sender() == storage.escrow_contract.read());
    
    // Compute weight (using protocol floor for lambda since no counterparty trust)
    let weight = compute_cancellation_weight(stake);
    
    // Apply trust impact
    let contribution = (weight * outcome) / PRECISION;
    update_avatar_trust(avatar, skill_type, contribution, context.block_number());
    
    emit_cancellation_recorded(avatar, skill_type, outcome);
}
```

### Internal Functions

```noir
fn update_avatar_trust(
    avatar: AztecAddress,
    skill_type: Field,
    contribution: Field,
    timestamp: Field,
) {
    let mut state = storage.avatar_states.at(avatar).read();
    
    // Update trust value
    let current = state.trust_values.at(skill_type).read();
    state.trust_values.at(skill_type).write(add_signed(current, contribution));
    
    // Update contract count
    let count = state.contract_counts.at(skill_type).read();
    state.contract_counts.at(skill_type).write(count + 1);
    
    // Update oldest contract if first
    if state.oldest_contract == 0 {
        state.oldest_contract = timestamp;
    }
    
    storage.avatar_states.at(avatar).write(state);
}

fn check_sybil_compliance(
    avatar: AztecAddress,
    skill_type: Field,
    params: SybilParameters,
) -> bool {
    let state = storage.avatar_states.at(avatar).read();
    let history = storage.contract_histories.at(avatar);
    
    // Check counterparty diversity
    let unique_counterparties = count_unique_counterparties(history, skill_type);
    
    // Check velocity (contracts in recent period)
    let recent_count = count_recent_contracts(history, skill_type, params.velocity_period);
    
    // Check variance (if enough history)
    let contract_count = state.contract_counts.at(skill_type).read();
    if contract_count >= params.variance_min_history {
        let variance = compute_outcome_variance(history, skill_type);
        if variance < params.variance_threshold {
            return false;
        }
    }
    
    // All checks passed
    true
}

fn compute_weight(stake: Field, difficulty: Field, counterparty_trust: Field) -> Field {
    // ω(c) = f(stake, difficulty, counterparty_trust, recency)
    // Simplified version - full implementation in Noir circuits doc
    let stake_factor = log_approx(1 + stake);
    let difficulty_factor = difficulty;
    let counterparty_factor = sigmoid(counterparty_trust, FLOOR_LAMBDA);
    
    (stake_factor * difficulty_factor * counterparty_factor) / PRECISION
}

fn compute_trust_contribution(weight: Field, outcome: Field, params: SybilParameters) -> Field {
    // contribution = weight * outcome * velocity_factor
    // Full implementation includes all Sybil factors
    (weight * outcome) / PRECISION
}
```

---

## Interaction Flows

### Flow 1: Post and Accept Listing

```
Customer Avatar              Registry                Escrow                Provider Avatar
      │                         │                      │                        │
      │ create_listing()        │                      │                        │
      │────────────────────────>│                      │                        │
      │                         │ lock_stake()         │                        │
      │                         │─────────────────────>│                        │
      │                         │                      │                        │
      │                    [listing visible on-chain]                           │
      │                         │                      │                        │
      │                         │                      │    accept_listing()    │
      │                         │<─────────────────────────────────────────────│
      │                         │                      │                        │
      │                         │ verify_eligibility() │                        │
      │                         │─────────────────────────────────────────────>│
      │                         │                      │                        │
      │                         │ lock_stake()         │                        │
      │                         │─────────────────────>│                        │
      │                         │                      │                        │
      │                         │ create_contract()    │                        │
      │                         │─────────────────────>│                        │
      │                         │                      │                        │
      │                    [contract active, both stakes locked]                │
```

### Flow 2: Complete Contract

```
Provider Avatar              Customer Avatar           Escrow                 Avatar Contract
      │                           │                      │                        │
      │   [delivers work]         │                      │                        │
      │─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─>│                      │                        │
      │                           │                      │                        │
      │                           │ record_outcome()     │                        │
      │                           │─────────────────────>│                        │
      │                           │                      │                        │
      │                           │                      │ record_contract_       │
      │                           │                      │ completion()           │
      │                           │                      │───────────────────────>│
      │                           │                      │                        │
      │                           │                      │ [trust updated for     │
      │                           │                      │  both avatars]         │
      │                           │                      │                        │
      │                           │                      │ release stakes         │
      │                           │                      │                        │
      │   [stake returned]        │   [stake returned]   │                        │
      │<─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─│                        │
```

### Flow 3: Timeout

```
Anyone                         Escrow                 Customer Avatar
   │                              │                        │
   │ timeout_contract()           │                        │
   │ (after deadline)             │                        │
   │─────────────────────────────>│                        │
   │                              │                        │
   │                              │ release customer stake │
   │                              │───────────────────────>│
   │                              │                        │
   │                              │ transfer provider      │
   │                              │ stake to customer      │
   │                              │───────────────────────>│
   │                              │                        │
   │                         [no trust update - no outcome recorded]
```

---

## Privacy Boundaries

### The Correct Model

Privacy in QoT protects **human identity**, not **Avatar activity**.

### What's Always Private

| Data | Why Private |
|------|-------------|
| Human identity behind Avatar | Core QoT principle — decouple trust from identity |
| Links between Avatars (same human) | Multiple Avatars are natural; linkability would defeat purpose |
| Private keys | Cryptographic security |

### What's Public (Avatar Level)

| Data | Why Public |
|------|------------|
| Avatar addresses | Avatars are persistent pseudonyms |
| Contract listings (all parameters) | Discovery requires visibility |
| Who posted / who accepted | Avatar activity is reputation |
| Stakes committed | Economic commitment is part of the signal |
| Contract outcomes | Trust computation requires visible outcomes |
| Trust values per skill type | This is q⟨T⟩ — the whole point |
| Contract history | How trust is verified and computed |

### What ZK Proofs Provide

ZK proofs in QoT provide **efficiency**, not secrecy:

| Proof | What It Does | Why Useful |
|-------|--------------|------------|
| Eligibility proof | Proves trust ≥ threshold | Avoids scanning full history on-chain |
| Sybil compliance proof | Proves history meets requirements | Complex computation done off-chain |
| Range proof | Proves trust in [min, max] | Optional selective disclosure |

A verifier could compute eligibility by reading the Avatar's public history. ZK proofs make this efficient for large histories.

### Edge Cases

**"I want my Avatar activity to be private"**

Create a new Avatar. Each Avatar has an independent, visible history. If you want unlinkable activity, operate through separate Avatars. The human behind them remains private.

**"I want to prove eligibility without revealing my trust value"**

Use a ZK eligibility proof. This proves trust ≥ threshold without revealing the exact value. The threshold comparison result is public; the exact trust value can remain unqueried.

---

## Error Handling

### Revert Conditions

| Function | Revert Condition | Error Code |
|----------|------------------|------------|
| `create_project` | Project exists | `PROJECT_EXISTS` |
| `create_project` | Sybil params invalid | `INVALID_SYBIL_PARAMS` |
| `create_project` | Deadline invalid | `INVALID_DEADLINE` |
| `create_listing` | Listing exists | `LISTING_EXISTS` |
| `create_listing` | Not project customer | `UNAUTHORIZED` |
| `create_listing` | Sybil params not strict enough | `SYBIL_PARAMS_TOO_LOOSE` |
| `accept_listing` | Listing not open | `LISTING_NOT_OPEN` |
| `accept_listing` | Deadline passed | `DEADLINE_PASSED` |
| `accept_listing` | Insufficient stake | `INSUFFICIENT_STAKE` |
| `accept_listing` | Eligibility check failed | `INELIGIBLE` |
| `accept_listing` | Provider is customer | `SELF_CONTRACT` |
| `accept_listing` | Invalid difficulty | `INVALID_DIFFICULTY` |
| `accept_listing` | ZK proof required but not verified | `PROOF_REQUIRED` |
| `record_outcome` | Not customer | `UNAUTHORIZED` |
| `record_outcome` | Contract not active | `INVALID_STATE` |
| `record_outcome` | Deadline passed | `DEADLINE_PASSED` |
| `record_outcome` | Invalid outcome value | `INVALID_OUTCOME` |
| `timeout_contract` | Contract not active | `INVALID_STATE` |
| `timeout_contract` | Deadline not passed | `DEADLINE_NOT_REACHED` |
| `cancel_contract` | Contract not active | `INVALID_STATE` |
| `cancel_contract` | Not party to contract | `UNAUTHORIZED` |
| `finalize_cancellation` | No pending cancellation | `NO_PENDING_CANCELLATION` |
| `finalize_cancellation` | Grace period not passed | `GRACE_PERIOD_ACTIVE` |
| `prune_old_contracts` | Not Avatar owner | `UNAUTHORIZED` |
| `prune_old_contracts` | Contract too recent | `CONTRACT_TOO_RECENT` |
| `prune_old_contracts` | Contract has non-zero weight | `CONTRACT_STILL_ACTIVE` |

### Gas Considerations

| Operation | Relative Cost | Notes |
|-----------|---------------|-------|
| `create_project` | Medium | State writes + escrow lock |
| `create_listing` | Medium | State writes |
| `accept_listing` | High | Eligibility verification + escrow + difficulty |
| `record_outcome` | High | Trust updates for both parties |
| `timeout_contract` | Medium | State update + transfers |
| `cancel_contract` | Medium | Trust update + state change |
| `finalize_cancellation` | Low | State finalization + transfers |
| `prove_eligibility` (ZK) | High | Off-chain proof generation |
| `prune_old_contracts` | Low | Storage cleanup |

---

## Open Questions

All major design questions have been resolved.

### Decided

#### 1. Eligibility Verification: Hybrid (Option C)

**Decision**: Public state query for small histories, ZK proof for large histories.

For Avatars with fewer than ~50 contracts in a skill type, on-chain verification by reading public state is efficient enough. For larger histories, require a ZK eligibility proof to avoid expensive on-chain computation.

```noir
#[aztec(public)]
fn accept_listing(listing_id: Field, provider_stake: Field, eligibility_proof: Option<Proof>) {
    let listing = storage.listings.at(listing_id).read();
    let history_size = QoTAvatar::get_contract_count(context.msg_sender(), listing.skill_type);
    
    if history_size < SMALL_HISTORY_THRESHOLD {
        // Direct verification for small histories
        assert(QoTAvatar::verify_eligibility(
            context.msg_sender(),
            listing.skill_type,
            listing.threshold,
            listing.sybil_params,
        ));
    } else {
        // Require ZK proof for large histories
        assert(eligibility_proof.is_some());
        assert(verify_eligibility_proof(eligibility_proof.unwrap(), listing));
    }
    
    // ... rest of acceptance logic
}
```

#### 2. Difficulty Assessment: At Acceptance (Option A)

**Decision**: Provider assesses and commits to difficulty at the moment of accepting the contract.

This ensures difficulty is locked before work begins, preventing gaming after the fact. The provider evaluates the listing requirements and stakes their assessment as part of acceptance.

```noir
#[aztec(public)]
fn accept_listing(
    listing_id: Field,
    provider_stake: Field,
    difficulty: Field,  // Provider's difficulty assessment
    eligibility_proof: Option<Proof>,
) {
    // ... validation ...
    
    assert(difficulty <= MAX_DIFFICULTY);
    assert(difficulty > 0);  // Must provide assessment
    
    // Difficulty locked at contract creation
    QoTEscrow::create_contract(
        listing_id,
        listing.customer,
        context.msg_sender(),
        listing.skill_type,
        listing.customer_stake,
        provider_stake,
        difficulty,  // Committed here
        listing.completion_deadline,
        listing.sybil_params,
    );
}
```

#### 3. Cancellation: Unilateral with Consequences (Option D)

**Decision**: Either party can cancel unilaterally, but cancellation carries negative trust consequences. Mutual consent mitigates the consequences for both parties.

**Unilateral cancellation:**
- Cancelling party receives negative trust impact (treated as partial failure)
- Non-cancelling party receives no trust impact
- Both stakes returned (no economic penalty, only reputational)

**Mutual cancellation:**
- Both parties receive smaller negative trust impact (shared responsibility)
- Reflects that sometimes contracts don't work out through no one's fault

```noir
#[aztec(public)]
fn cancel_contract(contract_id: Field) {
    let mut contract = storage.contracts.at(contract_id).read();
    assert(contract.state == ContractState::Active);
    
    let caller = context.msg_sender();
    assert(caller == contract.customer | caller == contract.provider);
    
    // Check if other party has already initiated cancellation
    let other_initiated = storage.cancel_requests.at(contract_id).read();
    
    if other_initiated {
        // Mutual cancellation - mitigated consequences for both
        apply_mutual_cancellation_trust(contract);
    } else if caller == contract.customer {
        // Customer unilateral - customer takes the hit
        apply_unilateral_cancellation_trust(contract, contract.customer);
        storage.cancel_requests.at(contract_id).write(true);
        // Give provider window to also cancel for mutual
        return;  // Don't finalize yet
    } else {
        // Provider unilateral - provider takes the hit
        apply_unilateral_cancellation_trust(contract, contract.provider);
        storage.cancel_requests.at(contract_id).write(true);
        return;  // Don't finalize yet
    }
    
    // Finalize cancellation
    contract.state = ContractState::Cancelled;
    storage.contracts.at(contract_id).write(contract);
    
    // Return stakes to both parties
    release_stake(contract.customer, contract_id, contract.customer_stake);
    release_stake(contract.provider, contract_id, contract.provider_stake);
}

fn apply_unilateral_cancellation_trust(contract: ActiveContract, canceller: AztecAddress) {
    // Canceller receives negative trust (e.g., outcome = -0.3)
    let cancellation_outcome = -300000;  // -0.3 scaled
    
    QoTAvatar::record_cancellation(
        canceller,
        contract.skill_type,
        contract.customer_stake + contract.provider_stake,
        cancellation_outcome,
    );
}

fn apply_mutual_cancellation_trust(contract: ActiveContract) {
    // Both receive smaller negative trust (e.g., outcome = -0.1 each)
    let mutual_cancellation_outcome = -100000;  // -0.1 scaled
    
    QoTAvatar::record_cancellation(
        contract.customer,
        SKILL_TYPE_CUSTOMER_COMMITMENT,
        contract.customer_stake,
        mutual_cancellation_outcome,
    );
    
    QoTAvatar::record_cancellation(
        contract.provider,
        contract.skill_type,
        contract.provider_stake,
        mutual_cancellation_outcome,
    );
}
```

#### 4. Token Integration: Single Token, Extensible Design (Option C)

**Decision**: Launch with a single token (native Aztec token or designated stablecoin). Design interfaces to support future multi-token extension.

```noir
/// Current: single token address, hardcoded at deployment
global STAKE_TOKEN: AztecAddress = /* deployed token address */;

/// Future-proofing: escrow functions accept token parameter
/// but currently assert it matches STAKE_TOKEN
#[aztec(public)]
fn lock_stake(
    owner: AztecAddress,
    reference_id: Field,
    amount: Field,
    token: AztecAddress,  // For future multi-token support
) {
    // Current: only one token allowed
    assert(token == STAKE_TOKEN);
    
    // Future: validate token is in whitelist
    // assert(storage.allowed_tokens.contains(token));
    
    transfer_to_escrow(owner, amount, token);
    // ...
}
```

#### 5. History Pruning: Allowed After 7 Years

**Decision**: Contracts older than 7 years have zero trust potency (per recency decay). Avatars may prune these contracts from their history to reduce storage and proof costs.

**Rationale**: If a contract contributes zero to trust computation, retaining it serves no purpose. Pruning is optional — Avatars with small histories may keep full records.

**Safeguards**:
- Only contracts with zero computed weight can be pruned
- Pruning is Avatar-initiated (not automatic)
- Contract existence remains verifiable via on-chain events (immutable)
- Pruned contracts still count toward "total contracts ever" for Sybil metrics if needed

```noir
global PRUNING_THRESHOLD_BLOCKS: Field = 18396000;  // ~7 years at 12s blocks

#[aztec(public)]
fn prune_old_contracts(avatar: AztecAddress, contract_ids: [Field; N]) {
    // Only Avatar owner can prune their history
    assert(context.msg_sender() == avatar);
    
    for contract_id in contract_ids {
        let entry = storage.contract_histories.at(avatar).get(contract_id);
        
        // Must be old enough
        let age = context.block_number() - entry.timestamp;
        assert(age >= PRUNING_THRESHOLD_BLOCKS);
        
        // Verify weight would be zero (recency decay check)
        let weight = compute_weight_with_recency(entry, context.block_number());
        assert(weight == 0);
        
        // Remove from active history
        storage.contract_histories.at(avatar).remove(contract_id);
    }
    
    emit_history_pruned(avatar, contract_ids.len());
}
```

### Remaining Open

None. All major design questions resolved.

---

## Related Documents

| Document | Relationship |
|----------|--------------|
| **QuantumOfTrust_v10.md** | Framework whitepaper — conceptual foundation |
| **Quantum_of_Trust_Equations_in_Noir.md** | Research library — core circuit implementations |
| **Sybil_Resistance_Circuits_Noir.md** | Research library — Sybil defense circuits |
| **qot_circuits.nr** | Production library — consolidated circuits for contract use (to be created) |
| **Blockchain_Selection_for_Quantum_of_Trust_Implementation.md** | Why Aztec/Noir |
| **Sybil_Resistance_Architecture.md** | Defense mechanism design |
| **ADR_Subcontract_Architecture.md** | Hierarchical contract pattern |

---

## Appendix: Aztec.nr Integration Notes

### Required Imports

```noir
use dep::aztec::{
    context::{PublicContext, PrivateContext},
    state_vars::{Map, PublicState},
    types::address::AztecAddress,
};
```

### Storage Declaration

```noir
#[aztec(storage)]
struct Storage {
    // Registry
    projects: Map<Field, PublicState<Project>>,
    listings: Map<Field, PublicState<ContractListing>>,
    verified_eligibility: Map<AztecAddress, Map<Field, PublicState<bool>>>,  // For ZK proof caching
    
    // Escrow
    contracts: Map<Field, PublicState<ActiveContract>>,
    escrow_entries: Map<Field, PublicState<EscrowEntry>>,
    cancel_requests: Map<Field, PublicState<CancelRequest>>,
    
    // Avatar
    avatar_states: Map<AztecAddress, PublicState<AvatarState>>,
    contract_histories: Map<AztecAddress, Vec<ContractHistoryEntry>>,
    
    // Cross-contract references
    registry_contract: PublicState<AztecAddress>,
    escrow_contract: PublicState<AztecAddress>,
    avatar_contract: PublicState<AztecAddress>,
}
```

### Events

```noir
// Registry events
event ProjectCreated { project_id: Field, customer: AztecAddress, stake: Field }
event ListingCreated { listing_id: Field, customer: AztecAddress, skill_type: Field, threshold: Field }
event ListingAccepted { listing_id: Field, provider: AztecAddress, stake: Field, difficulty: Field }
event ListingCancelled { listing_id: Field }

// Escrow events
event StakeLocked { owner: AztecAddress, reference_id: Field, amount: Field }
event StakeReleased { owner: AztecAddress, reference_id: Field, amount: Field }
event ContractCreated { contract_id: Field, customer: AztecAddress, provider: AztecAddress, skill_type: Field, difficulty: Field }
event OutcomeRecorded { contract_id: Field, outcome: Field }
event ContractTimedOut { contract_id: Field }
event CancellationInitiated { contract_id: Field, initiator: AztecAddress }
event ContractCancelled { contract_id: Field, mutual: bool }

// Avatar events
event TrustUpdated { avatar: AztecAddress, skill_type: Field, contribution: Field }
event CancellationRecorded { avatar: AztecAddress, skill_type: Field, outcome: Field }
event HistoryPruned { avatar: AztecAddress, contracts_removed: Field }
```

---

*This specification defines the target architecture for QoT on Aztec. Implementation will proceed iteratively, validating against Aztec's APIs as they stabilize.*
