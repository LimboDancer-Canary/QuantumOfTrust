# Quantum of Trust: Mathematical Equations in Noir

## Executive Summary

This document translates the mathematical equations from the Quantum of Trust framework into Noir, Aztec's domain-specific language for zero-knowledge circuits. Noir circuits generate cryptographic proofs that verify computations without revealing private inputs.

The critical insight: **we don't compute trust values to return them. We prove statements about trust values without revealing the underlying data.**

### What This Document Covers

1. **ZK-specific constraints** — Fixed-size arrays, field arithmetic, private vs public inputs
2. **Signed arithmetic** — Handling negative trust values in finite field arithmetic
3. **Core types** — `Agent`, `Contract`, and `DAO` as Noir structs
4. **Valuation function** — Trust computation over private history
5. **Eligibility proofs** — The core primitive: prove V_t ≥ θ without revealing history
6. **Contract structure** — Private notes containing contract data
7. **Outcome handling** — Scaled integer representation in finite fields
8. **Weighting function** — Approximations for log, exp, tanh in ZK circuits
9. **History evolution** — Note creation and nullifier patterns
10. **Trust evolution** — Incremental updates with private state
11. **DAO aggregation** — Proving composite trust
12. **Sybil resistance** — How privacy preserves the economic guarantees

### Key Constraints in ZK Circuits

| Aspect | Constraint |
|--------|------------|
| Execution model | Proof generation + verification |
| Data visibility | Private inputs hidden from verifier |
| Array sizes | Fixed at compile time |
| Arithmetic | Finite field (integers mod p) — always positive |
| Iteration | Bounded, unrolled loops |
| Result | Boolean proof statement |

---

## Part One: ZK Fundamentals for Quantum of Trust

### The Prover-Verifier Model

In zero-knowledge systems, there are two parties:

- **Prover**: Holds private data (contract history), generates proof
- **Verifier**: Sees only public inputs and proof, learns nothing else

For Quantum of Trust:

```
Prover (Avatar owner):
  - Knows: Full contract history, exact trust score
  - Proves: "My trust in Engineering ≥ 50"
  - Reveals: Nothing about individual contracts

Verifier (Contract poster, network):
  - Sees: Skill type, threshold, proof validity
  - Learns: Avatar is eligible (or not)
  - Cannot learn: History size, counterparties, stakes, outcomes
```

### Noir Execution Model

Noir compiles to an arithmetic circuit. Every operation becomes constraints over a finite field. Key implications:

1. **No dynamic allocation** — Array sizes fixed at compile time
2. **No unbounded loops** — All iteration counts known statically
3. **Field arithmetic** — All values are elements of a prime field (always positive)
4. **Conditional execution** — Both branches execute; result selected
5. **No early returns** — Use if-else expressions instead

```noir
// This is NOT how Noir works:
fn bad_example(history: Vec<Contract>) { ... }  // Dynamic size - invalid

// This IS how Noir works:
fn good_example(history: [Contract; MAX_HISTORY], count: Field) { ... }  // Fixed size
```

### Public vs Private Inputs

```noir
// Public inputs: visible to verifier (marked with `pub`)
// Private inputs: known only to prover (no marker)
// Public outputs: visible to verifier (return type marked with `pub`)

fn prove_eligibility(
    // Public - verifier sees these
    skill_type: pub Field,
    threshold: pub Field,
    
    // Private - verifier learns nothing about these
    history: [Contract; MAX_HISTORY],
    history_count: Field,
) -> pub bool {
    // Circuit logic here
    // Return value is public output
}
```

---

## Part Two: Signed Arithmetic in Finite Fields

### The Problem

Noir's `Field` type represents elements of a prime field — they are always positive. But trust values can be negative (actively distrusted agents). We need a representation for signed values.

### Solution: Signed Struct with Magnitude and Sign

```noir
/// Represents a signed value in finite field arithmetic.
/// 
/// Since Noir's Field type represents elements of a prime field (always positive),
/// we cannot directly represent negative numbers. This struct tracks sign separately
/// from magnitude, enabling proper signed arithmetic.
/// 
/// # Normalization Invariant
/// Zero is always represented with `is_negative = false`. The constructor `new()`
/// enforces this invariant, which simplifies comparison logic.
/// 
/// # Examples
/// ```
/// let positive = Signed::from_positive(100);  // +100
/// let negative = Signed::from_negative(50);   // -50
/// let sum = positive.add(negative);           // +50
/// ```
struct Signed {
    /// The absolute value of this signed number.
    magnitude: Field,
    
    /// True if this value is negative, false if positive or zero.
    /// Invariant: if magnitude == 0, then is_negative == false.
    is_negative: bool,
}

impl Signed {
    /// Creates a signed zero value.
    /// 
    /// # Returns
    /// A `Signed` representing the value 0.
    fn zero() -> Self {
        Signed { magnitude: 0, is_negative: false }
    }
    
    /// Creates a signed value from a positive Field value.
    /// 
    /// # Arguments
    /// * `value` - The positive value (must be >= 0 conceptually)
    /// 
    /// # Returns
    /// A `Signed` representing the positive value.
    fn from_positive(value: Field) -> Self {
        Signed { magnitude: value, is_negative: false }
    }
    
    /// Creates a signed value from a negative value's magnitude.
    /// 
    /// # Arguments
    /// * `magnitude` - The absolute value of the negative number
    /// 
    /// # Returns
    /// A `Signed` representing -magnitude (or zero if magnitude is 0).
    /// 
    /// # Examples
    /// ```
    /// let neg_fifty = Signed::from_negative(50);  // Represents -50
    /// ```
    fn from_negative(magnitude: Field) -> Self {
        // Normalize: zero is never negative
        if magnitude == 0 {
            Signed { magnitude: 0, is_negative: false }
        } else {
            Signed { magnitude, is_negative: true }
        }
    }
    
    /// Creates a signed value from magnitude and sign flag.
    /// 
    /// This is the primary constructor that enforces the normalization invariant:
    /// zero values always have `is_negative = false`.
    /// 
    /// # Arguments
    /// * `magnitude` - The absolute value
    /// * `is_negative` - True if the value should be negative
    /// 
    /// # Returns
    /// A normalized `Signed` value.
    fn new(magnitude: Field, is_negative: bool) -> Self {
        // Normalization: zero is never negative
        let normalized_negative = if magnitude == 0 { false } else { is_negative };
        Signed { magnitude, is_negative: normalized_negative }
    }
    
    /// Adds two signed values.
    /// 
    /// Handles all combinations of positive and negative operands correctly.
    /// 
    /// # Arguments
    /// * `other` - The value to add to self
    /// 
    /// # Returns
    /// The sum as a normalized `Signed` value.
    /// 
    /// # Examples
    /// ```
    /// let a = Signed::from_positive(100);
    /// let b = Signed::from_negative(30);
    /// let sum = a.add(b);  // 100 + (-30) = 70
    /// assert(sum.magnitude == 70);
    /// assert(!sum.is_negative);
    /// ```
    fn add(self, other: Signed) -> Self {
        if self.is_negative == other.is_negative {
            // Same sign: add magnitudes, keep sign
            // (+a) + (+b) = +(a+b)
            // (-a) + (-b) = -(a+b)
            Signed::new(self.magnitude + other.magnitude, self.is_negative)
        } else if self.magnitude >= other.magnitude {
            // Different signs, self has larger magnitude: keep self's sign
            // (+5) + (-3) = +2, or (-5) + (+3) = -2
            Signed::new(self.magnitude - other.magnitude, self.is_negative)
        } else {
            // Different signs, other has larger magnitude: take other's sign
            // (+3) + (-5) = -2, or (-3) + (+5) = +2
            Signed::new(other.magnitude - self.magnitude, other.is_negative)
        }
    }
    
    /// Subtracts another signed value from this one.
    /// 
    /// Implemented as: a - b = a + (-b)
    /// 
    /// # Arguments
    /// * `other` - The value to subtract from self
    /// 
    /// # Returns
    /// The difference as a normalized `Signed` value.
    fn sub(self, other: Signed) -> Self {
        // a - b = a + (-b)
        let negated = Signed::new(other.magnitude, !other.is_negative);
        self.add(negated)
    }
    
    /// Multiplies two signed values.
    /// 
    /// The sign of the result follows standard rules:
    /// - positive × positive = positive
    /// - negative × negative = positive
    /// - positive × negative = negative
    /// - negative × positive = negative
    /// 
    /// # Arguments
    /// * `other` - The value to multiply with self
    /// 
    /// # Returns
    /// The product as a normalized `Signed` value.
    fn mul(self, other: Signed) -> Self {
        // XOR for sign: same signs → positive, different signs → negative
        Signed::new(
            self.magnitude * other.magnitude,
            self.is_negative != other.is_negative
        )
    }
    
    /// Divides this signed value by another (integer division).
    /// 
    /// # Arguments
    /// * `other` - The divisor (must have non-zero magnitude)
    /// 
    /// # Returns
    /// The quotient as a normalized `Signed` value.
    /// 
    /// # Panics
    /// Asserts if other.magnitude is zero.
    fn div(self, other: Signed) -> Self {
        assert(other.magnitude != 0);  // Division by zero check
        Signed::new(
            self.magnitude / other.magnitude,
            self.is_negative != other.is_negative
        )
    }
    
    /// Checks if self is greater than or equal to other.
    /// 
    /// Comparison rules:
    /// - Any positive >= any negative
    /// - For same sign: compare magnitudes (reversed for negatives)
    /// - Zero is neither positive nor negative, equals zero
    /// 
    /// # Arguments
    /// * `other` - The value to compare against
    /// 
    /// # Returns
    /// True if self >= other.
    fn gte(self, other: Signed) -> bool {
        if !self.is_negative & other.is_negative {
            // positive/zero >= negative: always true
            // (other.is_negative implies other.magnitude > 0 due to normalization)
            true
        } else if self.is_negative & !other.is_negative {
            // negative >= positive/zero: only if both are zero
            (self.magnitude == 0) & (other.magnitude == 0)
        } else if self.is_negative {
            // Both negative: smaller magnitude is greater
            // -2 >= -5 because |-2| <= |-5|
            self.magnitude <= other.magnitude
        } else {
            // Both positive/zero: larger magnitude is greater
            self.magnitude >= other.magnitude
        }
    }
    
    /// Checks if self is strictly greater than other.
    /// 
    /// # Arguments
    /// * `other` - The value to compare against
    /// 
    /// # Returns
    /// True if self > other.
    fn gt(self, other: Signed) -> bool {
        if !self.is_negative & other.is_negative {
            // positive/zero > negative: always true
            // (normalization ensures other.magnitude > 0 when other.is_negative)
            true
        } else if self.is_negative & !other.is_negative {
            // negative > positive/zero: never true
            false
        } else if self.is_negative {
            // Both negative: smaller magnitude is greater
            // -2 > -5 because |-2| < |-5|
            self.magnitude < other.magnitude
        } else {
            // Both positive/zero: larger magnitude is greater
            self.magnitude > other.magnitude
        }
    }
    
    /// Checks if this value is strictly positive (greater than zero).
    /// 
    /// # Returns
    /// True if the value is positive (not zero, not negative).
    fn is_positive(self) -> bool {
        !self.is_negative & (self.magnitude > 0)
    }
    
    /// Checks if this value is zero.
    /// 
    /// # Returns
    /// True if the magnitude is zero.
    fn is_zero(self) -> bool {
        self.magnitude == 0
    }
    
    /// Returns the absolute value as an unsigned Field.
    /// 
    /// # Returns
    /// The magnitude of this signed value.
    fn abs(self) -> Field {
        self.magnitude
    }
    
    /// Negates this signed value.
    /// 
    /// # Returns
    /// A new `Signed` with the opposite sign (zero remains zero).
    fn negate(self) -> Self {
        Signed::new(self.magnitude, !self.is_negative)
    }
}
```

---

## Part Three: Constants and Fixed-Point Arithmetic

### Global Constants

```noir
// ============================================
// ARRAY SIZE CONSTANTS
// Must be fixed at compile time for ZK circuits
// ============================================

/// Maximum number of contracts in an agent's history per skill type.
/// Larger values increase proof size and generation time.
global MAX_HISTORY: u32 = 256;

/// Maximum number of members in a DAO.
global MAX_DAO_MEMBERS: u32 = 64;

/// Maximum nesting depth for DAOs containing DAOs.
global MAX_DAO_DEPTH: u32 = 3;

// ============================================
// FIXED-POINT ARITHMETIC CONSTANTS
// Used to simulate decimal arithmetic in finite fields
// ============================================

/// Precision multiplier for fixed-point arithmetic.
/// A "decimal" value is stored as: actual_value * PRECISION.
/// Example: 3.14159 is stored as 3141590.
/// Using 1,000,000 gives us 6 decimal places of precision.
global PRECISION: Field = 1000000;

// ============================================
// OUTCOME REPRESENTATION
// Outcomes use offset encoding to avoid negative fields
// ============================================

/// Offset for outcome representation.
/// Actual outcome = stored_value - OUTCOME_OFFSET.
/// Range [0, 200] represents outcomes [-100, +100].
global OUTCOME_OFFSET: Field = 100;

/// Maximum valid outcome offset value.
global OUTCOME_MAX: Field = 200;

// ============================================
// WEIGHT CALCULATION PARAMETERS
// All values scaled by PRECISION unless noted
// ============================================

/// Half-life for recency decay in days (unscaled).
/// Contracts from 365 days ago contribute half as much as today's.
global RECENCY_HALF_LIFE_DAYS: Field = 365;

/// Scaling factor for counterparty trust in tanh calculation (unscaled).
global COUNTERPARTY_SCALING: Field = 100;

/// Maximum influence of counterparty trust on weight.
/// 0.5 * PRECISION means ±50% adjustment.
global COUNTERPARTY_MAX_INFLUENCE: Field = 500000;

/// Minimum difficulty weight multiplier (0.5 * PRECISION).
global DIFFICULTY_WEIGHT_MIN: Field = 500000;

/// Range of difficulty weight above minimum (1.5 * PRECISION).
/// Total range is [0.5, 2.0] for difficulty [0, 10].
global DIFFICULTY_WEIGHT_RANGE: Field = 1500000;

/// Maximum difficulty rating (unscaled).
global MAX_DIFFICULTY: Field = 10;

// ============================================
// THRESHOLD CALCULATION
// ============================================

/// Minimum threshold factor (0.1 * PRECISION).
/// Ensures even zero-difficulty contracts have some threshold.
global MINIMUM_THRESHOLD_FACTOR: Field = 100000;
```

### Fixed-Point Arithmetic

```noir
/// Multiplies two fixed-point numbers.
/// 
/// Both inputs must be scaled by PRECISION. The result is also scaled by PRECISION.
/// 
/// # Arguments
/// * `a` - First fixed-point number (scaled by PRECISION)
/// * `b` - Second fixed-point number (scaled by PRECISION)
/// 
/// # Returns
/// Product scaled by PRECISION: (a * b) / PRECISION
/// 
/// # Examples
/// ```
/// let half = 500000;        // 0.5 * PRECISION
/// let quarter = 250000;     // 0.25 * PRECISION
/// let result = fp_mul(half, half);  // 0.25 * PRECISION = 250000
/// ```
fn fp_mul(a: Field, b: Field) -> Field {
    (a * b) / PRECISION
}

/// Divides two fixed-point numbers.
/// 
/// Both inputs must be scaled by PRECISION. The result is also scaled by PRECISION.
/// 
/// # Arguments
/// * `a` - Dividend (scaled by PRECISION)
/// * `b` - Divisor (scaled by PRECISION, must be non-zero)
/// 
/// # Returns
/// Quotient scaled by PRECISION: (a * PRECISION) / b
/// 
/// # Panics
/// Asserts if b is zero.
fn fp_div(a: Field, b: Field) -> Field {
    assert(b != 0);
    (a * PRECISION) / b
}

/// Computes the ratio of two raw (unscaled) integers.
/// 
/// Use this when dividing raw integers (like days or counts) to get a
/// fixed-point result.
/// 
/// # Arguments
/// * `numerator` - Raw integer numerator
/// * `denominator` - Raw integer denominator (must be non-zero)
/// 
/// # Returns
/// Ratio scaled by PRECISION: (numerator * PRECISION) / denominator
/// 
/// # Panics
/// Asserts if denominator is zero.
/// 
/// # Examples
/// ```
/// let half = ratio(1, 2);   // Returns 500000 (0.5 * PRECISION)
/// let third = ratio(1, 3);  // Returns 333333 (0.333... * PRECISION)
/// ```
fn ratio(numerator: Field, denominator: Field) -> Field {
    assert(denominator != 0);
    (numerator * PRECISION) / denominator
}

/// Converts a raw integer to fixed-point representation.
/// 
/// # Arguments
/// * `n` - Raw integer value
/// 
/// # Returns
/// Value scaled by PRECISION.
fn to_fp(n: Field) -> Field {
    n * PRECISION
}

/// Converts a fixed-point number to a raw integer (truncates).
/// 
/// # Arguments
/// * `fp` - Fixed-point value (scaled by PRECISION)
/// 
/// # Returns
/// Integer part of the value (fractional part discarded).
fn from_fp(fp: Field) -> Field {
    fp / PRECISION
}
```

### Approximating Transcendental Functions

The weighting function uses `log`, `pow`, and `tanh`. These don't exist natively in finite field arithmetic. We use lookup tables with linear interpolation.

#### Logarithm Approximation

For `log(1 + stake)` used in stake normalization:

```noir
/// Precomputed lookup table X values for log(1 + x).
/// These are the input points where we've precomputed log values.
global LOG_TABLE_X: [Field; 16] = [
    0, 1, 2, 5, 10, 20, 50, 100, 
    200, 500, 1000, 2000, 5000, 10000, 50000, 100000
];

/// Precomputed lookup table Y values for log(1 + x).
/// Each value is log(1 + LOG_TABLE_X[i]) * PRECISION.
global LOG_TABLE_Y: [Field; 16] = [
    0,          // log(1) = 0
    693147,     // log(2) ≈ 0.693147
    1098612,    // log(3) ≈ 1.098612
    1791759,    // log(6) ≈ 1.791759
    2397895,    // log(11) ≈ 2.397895
    3044522,    // log(21) ≈ 3.044522
    3931826,    // log(51) ≈ 3.931826
    4615121,    // log(101) ≈ 4.615121
    5303305,    // log(201) ≈ 5.303305
    6216606,    // log(501) ≈ 6.216606
    6908755,    // log(1001) ≈ 6.908755
    7601402,    // log(2001) ≈ 7.601402
    8517393,    // log(5001) ≈ 8.517393
    9210440,    // log(10001) ≈ 9.210440
    10819878,   // log(50001) ≈ 10.819878
    11512935    // log(100001) ≈ 11.512935
];

/// Approximates log(1 + x) using linear interpolation.
/// 
/// Uses a precomputed lookup table with 16 points spanning [0, 100000].
/// For values beyond the table, extrapolates from the last segment.
/// 
/// # Arguments
/// * `x` - Unscaled stake value (must be >= 0)
/// 
/// # Returns
/// Approximation of log(1 + x) scaled by PRECISION.
/// 
/// # Accuracy
/// Linear interpolation between table points. Maximum error depends on
/// spacing between table entries; typically < 5% for values in table range.
fn approx_log1p(x: Field) -> Field {
    // Find bracketing entries in table
    let mut lower_idx: u32 = 0;
    let mut upper_idx: u32 = 1;
    
    for i in 0..15 {
        if LOG_TABLE_X[i] <= x {
            if LOG_TABLE_X[i + 1] > x {
                lower_idx = i;
                upper_idx = i + 1;
            }
        }
    }
    
    // Handle x beyond table range - use last segment for extrapolation
    if x >= LOG_TABLE_X[15] {
        lower_idx = 14;
        upper_idx = 15;
    }
    
    // Linear interpolation
    let x_lower = LOG_TABLE_X[lower_idx];
    let x_upper = LOG_TABLE_X[upper_idx];
    let y_lower = LOG_TABLE_Y[lower_idx];
    let y_upper = LOG_TABLE_Y[upper_idx];
    
    if x_upper == x_lower {
        y_lower
    } else {
        // Ensure x >= x_lower to prevent underflow
        // (should always be true given the loop logic, but defensive)
        assert(x >= x_lower);
        
        // t = (x - x_lower) / (x_upper - x_lower), scaled by PRECISION
        let t = ratio(x - x_lower, x_upper - x_lower);
        // result = y_lower + t * (y_upper - y_lower)
        y_lower + fp_mul(t, y_upper - y_lower)
    }
}
```

#### Exponential Decay Approximation

For recency weighting `0.5^(days/365)`:

```noir
/// Precomputed lookup table for recency decay: days since completion.
global DECAY_TABLE_DAYS: [Field; 12] = [
    0, 30, 90, 180, 365, 548, 730, 1095, 1460, 1825, 2190, 2555
];

/// Precomputed lookup table for recency decay: 0.5^(days/365) * PRECISION.
global DECAY_TABLE_VALUES: [Field; 12] = [
    1000000,    // 0.5^0 = 1.0
    944061,     // 0.5^(30/365) ≈ 0.944
    835729,     // 0.5^(90/365) ≈ 0.836
    698402,     // 0.5^(180/365) ≈ 0.698
    500000,     // 0.5^1 = 0.5
    353553,     // 0.5^1.5 ≈ 0.354
    250000,     // 0.5^2 = 0.25
    125000,     // 0.5^3 = 0.125
    62500,      // 0.5^4 = 0.0625
    31250,      // 0.5^5 ≈ 0.031
    15625,      // 0.5^6 ≈ 0.016
    7812        // 0.5^7 ≈ 0.008
];

/// Approximates 0.5^(days/365) using linear interpolation.
/// 
/// Models the recency decay where older contracts contribute less to trust.
/// A contract from 365 days ago contributes half as much as one from today.
/// 
/// # Arguments
/// * `days` - Days since contract completion (unscaled integer)
/// 
/// # Returns
/// Decay factor scaled by PRECISION. Range: [~0.008, 1.0] * PRECISION
/// 
/// # Notes
/// Contracts older than ~7 years (2555 days) get the minimum weight.
fn approx_recency_decay(days: Field) -> Field {
    let mut lower_idx: u32 = 0;
    let mut upper_idx: u32 = 1;
    
    for i in 0..11 {
        if DECAY_TABLE_DAYS[i] <= days {
            if DECAY_TABLE_DAYS[i + 1] > days {
                lower_idx = i;
                upper_idx = i + 1;
            }
        }
    }
    
    // Very old contracts get minimum weight
    if days >= DECAY_TABLE_DAYS[11] {
        DECAY_TABLE_VALUES[11]
    } else {
        let d_lower = DECAY_TABLE_DAYS[lower_idx];
        let d_upper = DECAY_TABLE_DAYS[upper_idx];
        let v_lower = DECAY_TABLE_VALUES[lower_idx];
        let v_upper = DECAY_TABLE_VALUES[upper_idx];
        
        if d_upper == d_lower {
            v_lower
        } else {
            // Ensure days >= d_lower to prevent underflow
            assert(days >= d_lower);
            
            // t = (days - d_lower) / (d_upper - d_lower)
            let t = ratio(days - d_lower, d_upper - d_lower);
            // Decay decreases, so: v_lower - t * (v_lower - v_upper)
            v_lower - fp_mul(t, v_lower - v_upper)
        }
    }
}
```

#### Hyperbolic Tangent Approximation

For counterparty trust influence bounding `tanh(trust/100)`:

```noir
/// Precomputed lookup table X values for tanh(x/100).
/// Stored as absolute values; we handle sign separately.
global TANH_TABLE_X: [Field; 11] = [
    0, 10, 25, 50, 100, 150, 200, 300, 400, 500, 1000
];

/// Precomputed lookup table Y values for tanh(x/100) * PRECISION.
global TANH_TABLE_Y: [Field; 11] = [
    0,          // tanh(0) = 0
    99668,      // tanh(0.1) ≈ 0.0997
    244919,     // tanh(0.25) ≈ 0.2449
    462117,     // tanh(0.5) ≈ 0.4621
    761594,     // tanh(1.0) ≈ 0.7616
    905148,     // tanh(1.5) ≈ 0.9051
    964028,     // tanh(2.0) ≈ 0.9640
    995055,     // tanh(3.0) ≈ 0.9951
    999329,     // tanh(4.0) ≈ 0.9993
    999909,     // tanh(5.0) ≈ 0.9999
    1000000     // tanh(10.0) ≈ 1.0
];

/// Approximates tanh(x/100) for a signed value using linear interpolation.
/// 
/// The tanh function maps any real number to the range (-1, 1), making it
/// ideal for bounding the influence of counterparty trust on contract weight.
/// 
/// Uses the property that tanh is an odd function: tanh(-x) = -tanh(x).
/// 
/// # Arguments
/// * `x` - Signed trust value (magnitude is the trust score)
/// 
/// # Returns
/// Signed value in range [-PRECISION, +PRECISION] representing tanh(x/100).
fn approx_tanh_scaled(x: Signed) -> Signed {
    // Work with magnitude (tanh is odd function)
    let abs_x = x.magnitude;
    
    let mut lower_idx: u32 = 0;
    let mut upper_idx: u32 = 1;
    
    for i in 0..10 {
        if TANH_TABLE_X[i] <= abs_x {
            if TANH_TABLE_X[i + 1] > abs_x {
                lower_idx = i;
                upper_idx = i + 1;
            }
        }
    }
    
    let result_magnitude = if abs_x >= TANH_TABLE_X[10] {
        // Saturation: tanh approaches ±1 for large inputs
        TANH_TABLE_Y[10]
    } else {
        let x_lower = TANH_TABLE_X[lower_idx];
        let x_upper = TANH_TABLE_X[upper_idx];
        let y_lower = TANH_TABLE_Y[lower_idx];
        let y_upper = TANH_TABLE_Y[upper_idx];
        
        if x_upper == x_lower {
            y_lower
        } else {
            // Ensure abs_x >= x_lower to prevent underflow
            assert(abs_x >= x_lower);
            
            let t = ratio(abs_x - x_lower, x_upper - x_lower);
            y_lower + fp_mul(t, y_upper - y_lower)
        }
    };
    
    // Apply sign (tanh is odd: tanh(-x) = -tanh(x))
    Signed::new(result_magnitude, x.is_negative)
}
```

---

## Part Four: Core Type Definitions

### Mathematical Notation

$$q\langle T \rangle ::= \text{Agent}(t, h_t) \mid \text{DAO}(\{q\langle T \rangle\})$$

### Noir Implementation

```noir
/// Unique identifier for an Avatar/Agent in the network.
/// 
/// In the full Aztec integration, this would be an AztecAddress.
/// For the core circuit logic, we represent it as a Field.
struct AgentId {
    /// The unique identifier value.
    inner: Field,
}

impl AgentId {
    /// Creates a new AgentId from a Field value.
    /// 
    /// # Arguments
    /// * `id` - The unique identifier value
    fn new(id: Field) -> Self {
        AgentId { inner: id }
    }
    
    /// Checks equality with another AgentId.
    fn eq(self, other: AgentId) -> bool {
        self.inner == other.inner
    }
    
    /// Checks if this is the zero/null agent ID.
    fn is_zero(self) -> bool {
        self.inner == 0
    }
}

/// Skill type identifier.
/// 
/// Represents a category of work (e.g., "engineering", "design", "legal").
/// In practice, this is a hash of the skill type string computed off-circuit.
struct SkillType {
    /// The hashed skill type identifier.
    inner: Field,
}

impl SkillType {
    /// Creates a new SkillType from a Field value.
    /// 
    /// # Arguments
    /// * `id` - The skill type identifier (typically a hash)
    fn new(id: Field) -> Self {
        SkillType { inner: id }
    }
    
    /// Checks equality with another SkillType.
    fn eq(self, other: SkillType) -> bool {
        self.inner == other.inner
    }
}

/// A completed contract in an agent's history.
/// 
/// Contracts are the atomic units of trust building. Each contract represents
/// a completed agreement between a provider and consumer, with an outcome
/// (success/failure) that affects the provider's trust quotient.
/// 
/// # Field Scaling Conventions
/// - `stake`: Unscaled token amount
/// - `difficulty`: Unscaled integer 0-10
/// - `outcome_offset`: Unscaled integer 0-200 (represents -100 to +100)
/// - `completed_at`: Unscaled unix timestamp or block number
/// - `weight`: Scaled by PRECISION (1.0 = 1,000,000)
struct Contract {
    /// The other party in this contract (consumer if this is provider's history).
    counterparty: AgentId,
    
    /// The skill type this contract was executed under.
    skill_type: SkillType,
    
    /// The stake value in tokens (unscaled).
    stake: Field,
    
    /// Difficulty rating from 0 (trivial) to 10 (extremely difficult).
    difficulty: Field,
    
    /// Outcome stored as offset: 0=-100 (failure), 100=0 (partial), 200=+100 (success).
    outcome_offset: Field,
    
    /// Timestamp when the contract was completed.
    completed_at: Field,
    
    /// Pre-computed weight (scaled by PRECISION).
    /// Computed off-circuit for efficiency; see compute_weight() for formula.
    weight: Field,
}

impl Contract {
    /// Creates an empty/inactive contract for unused array slots.
    /// 
    /// Empty contracts have weight=0, which is used as the "inactive" marker.
    /// The outcome is set to neutral (0) for safety.
    fn empty() -> Self {
        Contract {
            counterparty: AgentId::new(0),
            skill_type: SkillType::new(0),
            stake: 0,
            difficulty: 0,
            outcome_offset: OUTCOME_OFFSET,  // Neutral outcome (0)
            completed_at: 0,
            weight: 0,  // weight = 0 indicates inactive slot
        }
    }
    
    /// Gets the actual outcome as a Signed value in range [-100, +100].
    /// 
    /// Converts from offset representation back to signed value.
    fn outcome(self) -> Signed {
        if self.outcome_offset >= OUTCOME_OFFSET {
            Signed::from_positive(self.outcome_offset - OUTCOME_OFFSET)
        } else {
            Signed::from_negative(OUTCOME_OFFSET - self.outcome_offset)
        }
    }
    
    /// Checks if this contract is for a given skill type.
    fn is_skill(self, skill: SkillType) -> bool {
        self.skill_type.eq(skill)
    }
    
    /// Checks if this contract slot is active.
    /// 
    /// Inactive slots have weight=0 and should be skipped in calculations.
    fn is_active(self) -> bool {
        self.weight != 0
    }
    
    /// Computes this contract's contribution to trust: ω(c) · outcome(c).
    /// 
    /// The contribution is:
    /// - Positive if the outcome was successful
    /// - Negative if the outcome was a failure
    /// - Scaled by the pre-computed weight
    /// 
    /// # Returns
    /// Signed value scaled by PRECISION.
    fn trust_contribution(self) -> Signed {
        let outcome = self.outcome();
        // weight is scaled by PRECISION
        // outcome is in [-100, +100]
        // We want result scaled by PRECISION
        // contribution = weight * outcome / 100
        let magnitude = (self.weight * outcome.magnitude) / 100;
        Signed::new(magnitude, outcome.is_negative)
    }
    
    /// Validates that contract fields are within expected bounds.
    fn is_valid(self) -> bool {
        let valid_difficulty = self.difficulty <= MAX_DIFFICULTY;
        let valid_outcome = self.outcome_offset <= OUTCOME_MAX;
        valid_difficulty & valid_outcome
    }
}

/// An agent's complete history for proving eligibility.
/// 
/// Contains a fixed-size array of contracts. Unused slots have weight=0.
/// The `count` field tracks how many slots are actually used.
struct AgentHistory {
    /// Fixed-size array of contracts. Unused slots have weight=0.
    contracts: [Contract; MAX_HISTORY],
    
    /// Number of active contracts in the array.
    count: Field,
    
    /// The agent this history belongs to.
    agent_id: AgentId,
}

impl AgentHistory {
    /// Creates an empty history for a given agent.
    fn empty(agent_id: AgentId) -> Self {
        AgentHistory {
            contracts: [Contract::empty(); MAX_HISTORY],
            count: 0,
            agent_id,
        }
    }
}

/// A DAO's membership for proving composite trust.
/// 
/// DAOs aggregate the trust of their members using a configurable function.
/// Member trust values are pre-computed off-circuit for efficiency.
struct DAOMembership {
    /// Member agent IDs.
    members: [AgentId; MAX_DAO_MEMBERS],
    
    /// Number of actual members (rest of array is unused).
    member_count: Field,
    
    /// Pre-computed trust values for each member.
    member_trust_values: [Signed; MAX_DAO_MEMBERS],
    
    /// Aggregation function: 0=sum, 1=average, 2=minimum, 3=maximum.
    aggregation_type: Field,
}
```

---

## Part Five: Valuation Function

### Mathematical Notation

$$V_t: q\langle T \rangle \rightarrow \mathbb{R}$$

Where:
- $V_t = 0$ → unknown, no track record
- $V_t > 0$ → net positive history, trusted
- $V_t < 0$ → net negative history, actively distrusted

### Noir Implementation

```noir
/// Computes the trust value for an agent in a specific skill type.
/// 
/// This is the core V_t function from the mathematical framework.
/// It sums the weighted outcomes of all contracts in the agent's history
/// that match the specified skill type.
/// 
/// V_t(Agent(t, h_t)) = Σ ω(c) · outcome(c) for all c in h_t
/// 
/// # Arguments
/// * `history` - The agent's complete contract history
/// * `skill_type` - The skill type to compute trust for
/// 
/// # Returns
/// Signed trust value scaled by PRECISION.
/// - Positive: agent has net positive history (trusted)
/// - Zero: agent has no track record in this skill
/// - Negative: agent has net negative history (distrusted)
fn compute_trust_value(
    history: AgentHistory,
    skill_type: SkillType,
) -> Signed {
    let mut trust = Signed::zero();
    
    for i in 0..MAX_HISTORY {
        let contract = history.contracts[i];
        
        // Only count active contracts for the requested skill type
        if contract.is_active() & contract.is_skill(skill_type) {
            // CRITICAL: Validate contract bounds before using to prevent forgery
            // Without this, a malicious prover could use weight=10^20 to pass any threshold
            assert(contract.difficulty <= MAX_DIFFICULTY);
            assert(contract.outcome_offset <= OUTCOME_MAX);
            assert(verify_weight_bounds(contract));
            
            let contribution = contract.trust_contribution();
            trust = trust.add(contribution);
        }
    }
    
    trust
}

/// Proves that an agent's trust value meets or exceeds a threshold.
/// 
/// This is the primary circuit for eligibility checks. The verifier learns
/// only whether the agent is eligible, not their exact trust score or
/// contract history.
/// 
/// # Arguments
/// * `skill_type` - The skill type to check (public)
/// * `threshold` - Minimum trust required (public, scaled by PRECISION)
/// * `history` - The agent's contract history (private)
/// 
/// # Returns
/// True if the agent's trust >= threshold.
fn prove_trust_threshold(
    skill_type: pub Field,
    threshold: pub Field,
    history: AgentHistory,
) -> pub bool {
    let skill = SkillType::new(skill_type);
    let trust = compute_trust_value(history, skill);
    let threshold_signed = Signed::from_positive(threshold);
    trust.gte(threshold_signed)
}

/// Proves that an agent's trust value falls within a specified range.
/// 
/// Allows more nuanced disclosure than simple threshold proofs.
/// Example: "My trust is between 50 and 100" without revealing exact value.
/// 
/// # Arguments
/// * `skill_type` - The skill type to check (public)
/// * `lower_bound` - Minimum of range (public)
/// * `lower_is_negative` - Whether lower bound is negative (public)
/// * `upper_bound` - Maximum of range (public)
/// * `upper_is_negative` - Whether upper bound is negative (public)
/// * `history` - The agent's contract history (private)
/// 
/// # Returns
/// True if lower_bound <= trust <= upper_bound.
/// 
/// # Panics
/// Asserts if lower_bound > upper_bound (invalid range).
fn prove_trust_range(
    skill_type: pub Field,
    lower_bound: pub Field,
    lower_is_negative: pub bool,
    upper_bound: pub Field,
    upper_is_negative: pub bool,
    history: AgentHistory,
) -> pub bool {
    let skill = SkillType::new(skill_type);
    let trust = compute_trust_value(history, skill);
    
    let lower = Signed::new(lower_bound, lower_is_negative);
    let upper = Signed::new(upper_bound, upper_is_negative);
    
    // Validate that lower <= upper (i.e., upper >= lower)
    assert(upper.gte(lower));
    
    trust.gte(lower) & upper.gte(trust)
}
```

---

## Part Six: Agent Trust Value Calculation

### Mathematical Notation

$$V_t(\text{Agent}(t, h_t)) = \sum_{c \in h_t} \omega(c) \cdot \text{outcome}(c)$$

### Noir Implementation

```noir
/// Computes trust value with detailed breakdown for debugging/analysis.
/// 
/// Returns the total trust plus separate sums of positive and negative
/// contributions, and the count of matching contracts.
/// 
/// # Arguments
/// * `history` - The agent's contract history
/// * `skill_type` - The skill type to analyze
/// 
/// # Returns
/// Tuple of (total_trust, positive_sum, negative_sum, contract_count)
/// 
/// # Panics
/// Asserts if any contract has invalid difficulty, outcome, or weight bounds.
fn compute_trust_value_detailed(
    history: AgentHistory,
    skill_type: SkillType,
) -> (Signed, Signed, Signed, Field) {
    let mut trust = Signed::zero();
    let mut positive_sum = Signed::zero();
    let mut negative_sum = Signed::zero();
    let mut matching_count: Field = 0;
    
    for i in 0..MAX_HISTORY {
        let contract = history.contracts[i];
        
        if contract.is_active() & contract.is_skill(skill_type) {
            // Validate contract bounds before using
            assert(contract.difficulty <= MAX_DIFFICULTY);
            assert(contract.outcome_offset <= OUTCOME_MAX);
            assert(verify_weight_bounds(contract));
            
            let contribution = contract.trust_contribution();
            trust = trust.add(contribution);
            matching_count = matching_count + 1;
            
            if contribution.is_positive() {
                positive_sum = positive_sum.add(contribution);
            } else if contribution.is_negative {
                negative_sum = negative_sum.add(contribution);
            }
        }
    }
    
    (trust, positive_sum, negative_sum, matching_count)
}

/// Proves that an agent has net positive trust (more successes than failures).
/// 
/// # Arguments
/// * `skill_type` - The skill type to check (public)
/// * `history` - The agent's contract history (private)
/// 
/// # Returns
/// True if trust > 0.
fn prove_positive_trust(
    skill_type: pub Field,
    history: AgentHistory,
) -> pub bool {
    let skill = SkillType::new(skill_type);
    let trust = compute_trust_value(history, skill);
    trust.is_positive()
}

/// Proves that an agent has at least a minimum number of contracts.
/// 
/// This is a Sybil resistance signal: agents with more history are less
/// likely to be fake identities created to game the system.
/// 
/// # Arguments
/// * `skill_type` - The skill type to check (public)
/// * `minimum_contracts` - Required minimum number of contracts (public)
/// * `history` - The agent's contract history (private)
/// 
/// # Returns
/// True if the agent has >= minimum_contracts for this skill type.
fn prove_minimum_history(
    skill_type: pub Field,
    minimum_contracts: pub Field,
    history: AgentHistory,
) -> pub bool {
    let skill = SkillType::new(skill_type);
    let mut count: Field = 0;
    
    for i in 0..MAX_HISTORY {
        let contract = history.contracts[i];
        if contract.is_active() & contract.is_skill(skill) {
            count = count + 1;
        }
    }
    
    count >= minimum_contracts
}
```

---

## Part Seven: DAO Trust Value Calculation

### Mathematical Notation

$$V_t(\text{DAO}(S)) = \Phi\left(\{V_t(q) : q \in S\}\right)$$

Where Φ is an aggregation function (sum, average, min, max).

### Noir Implementation

```noir
/// Aggregates member trust values by summation.
/// 
/// Total capability interpretation: the DAO can handle work equal to
/// the combined capacity of all members.
/// 
/// # Arguments
/// * `values` - Array of member trust values
/// * `count` - Number of actual members
/// 
/// # Returns
/// Sum of all member trust values.
fn aggregate_sum(values: [Signed; MAX_DAO_MEMBERS], count: Field) -> Signed {
    let mut sum = Signed::zero();
    for i in 0..MAX_DAO_MEMBERS {
        if (i as Field) < count {
            sum = sum.add(values[i]);
        }
    }
    sum
}

/// Aggregates member trust values by averaging.
/// 
/// Mean reliability interpretation: represents the expected reliability
/// of a randomly selected member.
/// 
/// # Arguments
/// * `values` - Array of member trust values
/// * `count` - Number of actual members
/// 
/// # Returns
/// Average of member trust values, or zero if count is 0.
fn aggregate_average(values: [Signed; MAX_DAO_MEMBERS], count: Field) -> Signed {
    if count == 0 {
        Signed::zero()
    } else {
        let sum = aggregate_sum(values, count);
        // Divide magnitude by count (preserves sign)
        Signed::new(sum.magnitude / count, sum.is_negative)
    }
}

/// Aggregates member trust values by taking the minimum.
/// 
/// Weakest-link interpretation: the DAO is only as strong as its weakest
/// member. Appropriate for security-critical applications.
/// 
/// # Arguments
/// * `values` - Array of member trust values
/// * `count` - Number of actual members
/// 
/// # Returns
/// Minimum member trust value, or zero if count is 0.
fn aggregate_minimum(values: [Signed; MAX_DAO_MEMBERS], count: Field) -> Signed {
    if count == 0 {
        Signed::zero()
    } else {
        let mut min_val = values[0];
        
        for i in 1..MAX_DAO_MEMBERS {
            if (i as Field) < count {
                // Update min if current value is less than current minimum
                // values[i] < min_val  ⟺  !values[i].gte(min_val)
                if !values[i].gte(min_val) {
                    min_val = values[i];
                }
            }
        }
        min_val
    }
}

/// Aggregates member trust values by taking the maximum.
/// 
/// Best-member interpretation: the DAO's capability is defined by its
/// strongest member.
/// 
/// # Arguments
/// * `values` - Array of member trust values
/// * `count` - Number of actual members
/// 
/// # Returns
/// Maximum member trust value, or zero if count is 0.
fn aggregate_maximum(values: [Signed; MAX_DAO_MEMBERS], count: Field) -> Signed {
    if count == 0 {
        Signed::zero()
    } else {
        let mut max_val = values[0];
        
        for i in 1..MAX_DAO_MEMBERS {
            if (i as Field) < count {
                if values[i].gt(max_val) {
                    max_val = values[i];
                }
            }
        }
        max_val
    }
}

/// Computes DAO trust value using the configured aggregation function.
/// 
/// The aggregation type determines how member trust values combine:
/// - 0: Sum (total capability)
/// - 1: Average (mean reliability)
/// - 2: Minimum (weakest link)
/// - 3: Maximum (strongest member)
/// 
/// # Arguments
/// * `membership` - The DAO membership with pre-computed member trust values
/// 
/// # Returns
/// Aggregated trust value for the DAO.
fn compute_dao_trust(membership: DAOMembership) -> Signed {
    let values = membership.member_trust_values;
    let count = membership.member_count;
    
    if membership.aggregation_type == 0 {
        aggregate_sum(values, count)
    } else if membership.aggregation_type == 1 {
        aggregate_average(values, count)
    } else if membership.aggregation_type == 2 {
        aggregate_minimum(values, count)
    } else {
        aggregate_maximum(values, count)
    }
}

/// Proves that a DAO's aggregated trust meets or exceeds a threshold.
/// 
/// # Arguments
/// * `threshold` - Minimum required trust (public, scaled by PRECISION)
/// * `membership` - DAO membership with pre-computed member trusts (private)
/// 
/// # Returns
/// True if the DAO's aggregated trust >= threshold.
/// 
/// # Notes
/// Member trust values must be pre-computed and committed. In a full
/// implementation, each member would provide a proof of their trust value,
/// and this circuit would verify those proofs before aggregating.
fn prove_dao_trust_threshold(
    threshold: pub Field,
    membership: DAOMembership,
) -> pub bool {
    let dao_trust = compute_dao_trust(membership);
    let threshold_signed = Signed::from_positive(threshold);
    dao_trust.gte(threshold_signed)
}
```

---

## Part Eight: Contract Definition

### Mathematical Notation

$$c = (a_{\text{provider}}, a_{\text{consumer}}, t, s, d, \tau)$$

### Noir Implementation

```noir
/// Creates a contract with proper outcome offset encoding.
/// 
/// Converts a signed outcome value to the offset representation used
/// for storage in the Contract struct.
/// 
/// # Arguments
/// * `counterparty` - The other party in the contract
/// * `skill_type` - The skill type for this contract
/// * `stake` - Token amount at stake (unscaled)
/// * `difficulty` - Difficulty rating 0-10 (unscaled)
/// * `outcome` - Signed outcome value in range [-100, +100]
/// * `completed_at` - Completion timestamp
/// * `weight` - Pre-computed weight (scaled by PRECISION)
/// 
/// # Returns
/// A Contract with properly encoded outcome_offset.
/// 
/// # Panics
/// Asserts if difficulty > MAX_DIFFICULTY or outcome magnitude > 100.
fn create_contract(
    counterparty: AgentId,
    skill_type: SkillType,
    stake: Field,
    difficulty: Field,
    outcome: Signed,
    completed_at: Field,
    weight: Field,
) -> Contract {
    // Validate inputs to prevent invalid contracts
    assert(difficulty <= MAX_DIFFICULTY);
    assert(outcome.magnitude <= 100);  // Outcome must be in [-100, +100]
    
    // Convert signed outcome to offset representation
    let outcome_offset = if outcome.is_negative {
        OUTCOME_OFFSET - outcome.magnitude
    } else {
        OUTCOME_OFFSET + outcome.magnitude
    };
    
    Contract {
        counterparty,
        skill_type,
        stake,
        difficulty,
        outcome_offset,
        completed_at,
        weight,
    }
}

/// Validates that a contract's fields are within expected bounds.
/// 
/// Checks:
/// - Difficulty is in range [0, 10]
/// - Outcome offset is in range [0, 200]
/// - Weight is positive (active contract)
/// 
/// # Arguments
/// * `contract` - The contract to validate
/// 
/// # Returns
/// True if all fields are valid.
fn validate_contract(contract: Contract) -> bool {
    let valid_difficulty = contract.difficulty <= MAX_DIFFICULTY;
    let valid_outcome = contract.outcome_offset <= OUTCOME_MAX;
    let valid_weight = contract.weight > 0;
    
    valid_difficulty & valid_outcome & valid_weight
}
```

---

## Part Nine: Outcome Function

### Mathematical Notation

$$\text{outcome}(c) \in [-1, 1]$$

Continuous range allowing partial success/failure. Discrete {-1, 0, 1} as special case.

### Noir Implementation

```noir
/// Outcome constant: complete failure (-100 as offset 0).
global OUTCOME_FAILURE: Field = 0;

/// Outcome constant: partial/neutral (0 as offset 100).
global OUTCOME_PARTIAL: Field = 100;

/// Outcome constant: complete success (+100 as offset 200).
global OUTCOME_SUCCESS: Field = 200;

/// Calculates a partial outcome from completion and quality metrics.
/// 
/// Combines completion percentage and quality score into a single outcome
/// value. Both inputs contribute equally to the final outcome.
/// 
/// # Arguments
/// * `completion_percentage` - How much of the work was completed (0-100)
/// * `quality_score` - Quality rating of completed work (0-100)
/// 
/// # Returns
/// Outcome offset in range [0, 200].
/// 
/// # Panics
/// Asserts if either input exceeds 100.
/// 
/// # Examples
/// ```
/// // 100% complete, 100% quality → +100 outcome (offset 200)
/// let perfect = calculate_partial_outcome(100, 100);
/// assert(perfect == 200);
/// 
/// // 50% complete, 50% quality → 0 outcome (offset 100)
/// let partial = calculate_partial_outcome(50, 50);
/// assert(partial == 100);
/// 
/// // 0% complete, 0% quality → -100 outcome (offset 0)
/// let failure = calculate_partial_outcome(0, 0);
/// assert(failure == 0);
/// ```
fn calculate_partial_outcome(
    completion_percentage: Field,
    quality_score: Field,
) -> Field {
    assert(completion_percentage <= 100);
    assert(quality_score <= 100);
    
    // Average of completion and quality: [0, 100]
    let raw_score = (completion_percentage + quality_score) / 2;
    
    // Map [0, 100] to [0, 200] offset representation
    raw_score * 2
}

/// Checks if a contract outcome represents success (positive).
fn is_success(contract: Contract) -> bool {
    contract.outcome_offset > OUTCOME_OFFSET
}

/// Checks if a contract outcome represents failure (negative).
fn is_failure(contract: Contract) -> bool {
    contract.outcome_offset < OUTCOME_OFFSET
}

/// Checks if a contract outcome is neutral (zero).
fn is_neutral(contract: Contract) -> bool {
    contract.outcome_offset == OUTCOME_OFFSET
}
```

---

## Part Ten: Weighting Function

### Mathematical Notation

$$\omega(c) = f\big(s(c),\ d(c),\ V_t(a_{\text{consumer}}),\ \text{recency}(c)\big)$$

### Noir Implementation

```noir
/// Computes the weight for a contract (in-circuit version).
/// 
/// The weight determines how much a contract contributes to trust.
/// Higher weights mean the contract provides more signal. Factors:
/// 
/// 1. **Stake**: log(1 + stake) — higher stakes = more signal
/// 2. **Difficulty**: 0.5 + (difficulty/10) × 1.5 — harder = more signal
/// 3. **Counterparty trust**: 1 ± tanh(trust/100) × 0.5 — trusted counterparties matter more
/// 4. **Recency**: 0.5^(days/365) — recent contracts matter more
/// 
/// # Arguments
/// * `stake` - Token amount (unscaled)
/// * `difficulty` - Difficulty rating 0-10 (unscaled)
/// * `counterparty_trust` - Counterparty's trust value (scaled by PRECISION)
/// * `days_since_completion` - Age of contract in days (unscaled)
/// 
/// # Returns
/// Weight value scaled by PRECISION.
/// 
/// # Notes
/// In practice, weights are computed off-circuit at contract completion
/// time and stored in the Contract struct. This function documents the
/// formula and can be used for verification.
fn compute_weight(
    stake: Field,
    difficulty: Field,
    counterparty_trust: Signed,
    days_since_completion: Field,
) -> Field {
    // 1. Stake weight: log(1 + stake)
    let stake_weight = approx_log1p(stake);
    
    // 2. Difficulty weight: maps [0,10] to [0.5, 2.0]
    let difficulty_normalized = ratio(difficulty, MAX_DIFFICULTY);
    let difficulty_weight = DIFFICULTY_WEIGHT_MIN + 
                           fp_mul(difficulty_normalized, DIFFICULTY_WEIGHT_RANGE);
    
    // 3. Counterparty trust weight: maps any trust to [0.5, 1.5]
    let tanh_value = approx_tanh_scaled(counterparty_trust);
    let counterparty_weight = if tanh_value.is_negative {
        PRECISION - fp_mul(tanh_value.magnitude, COUNTERPARTY_MAX_INFLUENCE)
    } else {
        PRECISION + fp_mul(tanh_value.magnitude, COUNTERPARTY_MAX_INFLUENCE)
    };
    
    // 4. Recency weight: 0.5^(days/365)
    let recency_weight = approx_recency_decay(days_since_completion);
    
    // Combine multiplicatively
    let mut weight = stake_weight;
    weight = fp_mul(weight, difficulty_weight);
    weight = fp_mul(weight, counterparty_weight);
    weight = fp_mul(weight, recency_weight);
    
    // Ensure minimum weight for numerical stability
    if weight < 1 {
        1
    } else {
        weight
    }
}

/// Computes weight when counterparty trust is unknown or zero.
/// 
/// Convenience wrapper that uses neutral counterparty trust.
fn compute_weight_simple(
    stake: Field,
    difficulty: Field,
    days_since_completion: Field,
) -> Field {
    compute_weight(stake, difficulty, Signed::zero(), days_since_completion)
}

/// Verifies that a pre-computed weight is within reasonable bounds.
/// 
/// Prevents malicious weight inflation by checking against theoretical
/// maximum based on the contract's stake.
/// 
/// Maximum theoretical weight occurs when:
/// - stake_weight = log(1 + stake)
/// - difficulty_weight = 2.0 (max difficulty)
/// - counterparty_weight = 1.5 (max positive influence)
/// - recency_weight = 1.0 (completed today)
/// 
/// So max ≈ log(1 + stake) × 3.0
/// 
/// For zero/low stake contracts, we use a fixed reasonable maximum
/// since log(1) = 0 would otherwise allow no weight at all.
/// 
/// # Arguments
/// * `contract` - The contract with pre-computed weight
/// 
/// # Returns
/// True if the weight is within reasonable bounds.
fn verify_weight_bounds(contract: Contract) -> bool {
    let min_weight: Field = 1;
    
    // Maximum: log(1 + stake) * 3 + buffer
    let log_stake = approx_log1p(contract.stake);
    let max_theoretical = fp_mul(log_stake, 3 * PRECISION);
    
    // For zero/low stake, use minimum sensible maximum (3.0 scaled)
    // This handles the case where log(1 + 0) = 0
    let min_max_weight = 3 * PRECISION;
    
    // Add 1.0 (scaled) as buffer for rounding errors
    let computed_max = max_theoretical + PRECISION;
    
    // Use the larger of computed max or minimum max
    let max_weight = if computed_max > min_max_weight {
        computed_max
    } else {
        min_max_weight
    };
    
    (contract.weight >= min_weight) & (contract.weight <= max_weight)
}
```

---

## Part Eleven: History Evolution

### Mathematical Notation

$$h_t^{(n+1)}(a) = h_t^{(n)}(a) \cup \{c_n\}$$

### Noir Implementation

In Aztec, state evolution uses the "note" pattern with nullifiers. This is conceptual — actual implementation uses Aztec.nr primitives.

```noir
/// Represents a note containing a single contract.
/// 
/// In Aztec's private state model, each contract is stored as an
/// encrypted note. The owner can prove they own the note without
/// revealing its contents.
struct ContractNote {
    /// The contract data.
    contract: Contract,
    
    /// Owner's address (who can read/spend this note).
    owner: Field,
    
    /// Random value for note hiding (prevents note identification).
    randomness: Field,
}

/// Represents the agent's history state.
/// 
/// Combines a Merkle root (for efficient membership proofs) with
/// cached aggregate values for efficiency.
struct HistoryState {
    /// Root of Merkle tree containing all contract notes.
    history_root: Field,
    
    /// Current trust value (cached to avoid recomputation).
    cached_trust: Signed,
    
    /// Total number of contracts in history.
    contract_count: Field,
}

impl HistoryState {
    /// Creates an empty history state.
    fn empty() -> Self {
        HistoryState {
            history_root: 0,
            cached_trust: Signed::zero(),
            contract_count: 0,
        }
    }
}

/// Adds a contract to history and updates cached trust.
/// 
/// In Aztec, this would create a new note and nullify the old state.
/// The trust value is updated incrementally for efficiency.
/// 
/// # Arguments
/// * `old_state` - Current history state
/// * `new_contract` - Contract to add
/// * `skill_type` - Expected skill type (for validation)
/// 
/// # Returns
/// New history state with updated trust and count.
/// 
/// # Panics
/// Asserts if the contract's skill type doesn't match, or if contract
/// has invalid difficulty, outcome, or weight bounds.
fn add_to_history(
    old_state: HistoryState,
    new_contract: Contract,
    skill_type: SkillType,
) -> HistoryState {
    // Verify new contract is for the correct skill type
    assert(new_contract.is_skill(skill_type));
    
    // Validate contract bounds before using
    assert(new_contract.difficulty <= MAX_DIFFICULTY);
    assert(new_contract.outcome_offset <= OUTCOME_MAX);
    assert(verify_weight_bounds(new_contract));
    
    // Compute new trust value incrementally
    // V_t^(n+1) = V_t^(n) + ω(c_n) · outcome(c_n)
    let contribution = new_contract.trust_contribution();
    let new_trust = old_state.cached_trust.add(contribution);
    
    HistoryState {
        history_root: 0,  // Placeholder; would be computed from updated tree
        cached_trust: new_trust,
        contract_count: old_state.contract_count + 1,
    }
}
```

---

## Part Twelve: Trust Evolution

### Mathematical Notation

$$V_t^{(n+1)}(a) = V_t^{(n)}(a) + \omega(c_n) \cdot \text{outcome}(c_n)$$

### Noir Implementation

```noir
/// Updates trust incrementally when a new contract completes.
/// 
/// This is more efficient than recomputing from the entire history.
/// 
/// # Arguments
/// * `current_trust` - Current trust value (scaled by PRECISION)
/// * `new_contract` - The newly completed contract
/// 
/// # Returns
/// Updated trust value.
/// 
/// # Panics
/// Asserts if contract has invalid difficulty, outcome, or weight bounds.
fn update_trust(
    current_trust: Signed,
    new_contract: Contract,
) -> Signed {
    // Validate contract bounds before using
    assert(new_contract.difficulty <= MAX_DIFFICULTY);
    assert(new_contract.outcome_offset <= OUTCOME_MAX);
    assert(verify_weight_bounds(new_contract));
    
    current_trust.add(new_contract.trust_contribution())
}

/// Verifies that a trust update was computed correctly.
/// 
/// Used when trust updates need to be publicly verifiable. The verifier
/// can check that new_trust = old_trust + contribution.
/// 
/// # Arguments
/// * `old_trust_magnitude` - Previous trust magnitude (public)
/// * `old_trust_negative` - Whether previous trust was negative (public)
/// * `new_trust_magnitude` - New trust magnitude (public)
/// * `new_trust_negative` - Whether new trust is negative (public)
/// * `contract_weight` - Weight of the new contract (public)
/// * `contract_outcome_offset` - Outcome of the new contract (public)
/// 
/// # Returns
/// True if new_trust = old_trust + (weight × outcome / 100).
/// 
/// # Panics
/// Asserts if contract_outcome_offset > OUTCOME_MAX (invalid outcome).
fn verify_trust_update(
    old_trust_magnitude: pub Field,
    old_trust_negative: pub bool,
    new_trust_magnitude: pub Field,
    new_trust_negative: pub bool,
    contract_weight: pub Field,
    contract_outcome_offset: pub Field,
) -> pub bool {
    // Validate public inputs to prevent invalid outcomes
    assert(contract_outcome_offset <= OUTCOME_MAX);
    
    let old_trust = Signed::new(old_trust_magnitude, old_trust_negative);
    let new_trust = Signed::new(new_trust_magnitude, new_trust_negative);
    
    // Compute expected contribution from the contract
    let outcome = if contract_outcome_offset >= OUTCOME_OFFSET {
        Signed::from_positive(contract_outcome_offset - OUTCOME_OFFSET)
    } else {
        Signed::from_negative(OUTCOME_OFFSET - contract_outcome_offset)
    };
    
    let contribution_magnitude = (contract_weight * outcome.magnitude) / 100;
    let contribution = Signed::new(contribution_magnitude, outcome.is_negative);
    
    // Verify: new_trust == old_trust + contribution
    let expected = old_trust.add(contribution);
    (expected.magnitude == new_trust.magnitude) & 
    (expected.is_negative == new_trust.is_negative)
}

/// Batch updates trust from multiple contracts.
/// 
/// Applies multiple contract contributions to an initial trust value.
/// Useful when initializing from a set of historical contracts.
/// 
/// # Arguments
/// * `initial_trust` - Starting trust value
/// * `contracts` - Array of contracts to apply
/// * `count` - Number of contracts to process
/// * `skill_type` - Only apply contracts matching this skill type
/// 
/// # Returns
/// Updated trust value after all applicable contracts.
/// 
/// # Panics
/// Asserts if any contract has invalid difficulty, outcome, or weight bounds.
fn batch_update_trust(
    initial_trust: Signed,
    contracts: [Contract; MAX_HISTORY],
    count: Field,
    skill_type: SkillType,
) -> Signed {
    let mut trust = initial_trust;
    
    for i in 0..MAX_HISTORY {
        if (i as Field) < count {
            let contract = contracts[i];
            if contract.is_active() & contract.is_skill(skill_type) {
                // Validate contract bounds before using
                assert(contract.difficulty <= MAX_DIFFICULTY);
                assert(contract.outcome_offset <= OUTCOME_MAX);
                assert(verify_weight_bounds(contract));
                
                trust = trust.add(contract.trust_contribution());
            }
        }
    }
    
    trust
}
```

---

## Part Thirteen: Eligibility Function

### Mathematical Notation

$$\text{eligible}(a, c) \iff V_t(a) \geq \theta(c)$$

Where:
$$\theta(c) = \log(1 + s(c)) \cdot d(c)$$

### Noir Implementation

```noir
/// Calculates the trust threshold for a contract.
/// 
/// Higher stakes and difficulty require higher trust to participate.
/// The formula is: θ = max(log(1+stake) × 0.1, log(1+stake) × difficulty)
/// 
/// The minimum threshold (10% of stake factor) ensures that even
/// zero-difficulty contracts require some baseline trust.
/// 
/// # Arguments
/// * `stake` - Token amount at stake (unscaled)
/// * `difficulty` - Difficulty rating 0-10 (unscaled)
/// 
/// # Returns
/// Threshold value scaled by PRECISION.
/// 
/// # Panics
/// Asserts if difficulty > MAX_DIFFICULTY.
fn calculate_threshold(stake: Field, difficulty: Field) -> Field {
    // Validate difficulty to prevent overflow
    assert(difficulty <= MAX_DIFFICULTY);
    
    let stake_factor = approx_log1p(stake);
    
    // Minimum threshold: 10% of stake factor
    let minimum_threshold = fp_mul(stake_factor, MINIMUM_THRESHOLD_FACTOR);
    
    // Standard threshold: stake_factor × difficulty
    let difficulty_threshold = fp_mul(stake_factor, to_fp(difficulty));
    
    // Return the higher of the two
    if minimum_threshold > difficulty_threshold {
        minimum_threshold
    } else {
        difficulty_threshold
    }
}

/// The core eligibility proof circuit.
/// 
/// This is THE primary circuit for the Quantum of Trust framework.
/// It proves an agent can participate in a contract without revealing
/// anything about their history.
/// 
/// The verifier learns only: "This agent is eligible for this contract"
/// The verifier does NOT learn: contract count, specific outcomes, 
/// counterparties, stakes, or exact trust score.
/// 
/// # Arguments
/// * `skill_type` - The skill type for this contract (public)
/// * `contract_stake` - Stake amount for the contract (public)
/// * `contract_difficulty` - Difficulty rating for the contract (public)
/// * `history` - The agent's contract history (private)
/// 
/// # Returns
/// True if V_t(agent) >= θ(contract).
fn prove_eligibility(
    skill_type: pub Field,
    contract_stake: pub Field,
    contract_difficulty: pub Field,
    history: AgentHistory,
) -> pub bool {
    let skill = SkillType::new(skill_type);
    
    // Compute agent's trust value (remains private)
    let trust = compute_trust_value(history, skill);
    
    // Compute threshold for this contract
    let threshold = calculate_threshold(contract_stake, contract_difficulty);
    
    // Prove: V_t(agent) ≥ θ(contract)
    trust.gte(Signed::from_positive(threshold))
}

/// Extended eligibility proof with Sybil resistance.
/// 
/// In addition to meeting the trust threshold, requires the agent to
/// have a minimum number of contracts. This makes Sybil attacks more
/// expensive since each fake identity needs to complete real contracts.
/// 
/// # Arguments
/// * `skill_type` - The skill type for this contract (public)
/// * `contract_stake` - Stake amount for the contract (public)
/// * `contract_difficulty` - Difficulty rating for the contract (public)
/// * `minimum_history_size` - Required minimum contracts (public)
/// * `history` - The agent's contract history (private)
/// 
/// # Returns
/// True if trust >= threshold AND contract_count >= minimum_history_size.
fn prove_eligibility_extended(
    skill_type: pub Field,
    contract_stake: pub Field,
    contract_difficulty: pub Field,
    minimum_history_size: pub Field,
    history: AgentHistory,
) -> pub bool {
    let skill = SkillType::new(skill_type);
    
    // Basic eligibility check
    let trust = compute_trust_value(history, skill);
    let threshold = calculate_threshold(contract_stake, contract_difficulty);
    let meets_threshold = trust.gte(Signed::from_positive(threshold));
    
    // History size requirement (Sybil resistance)
    let mut history_count: Field = 0;
    for i in 0..MAX_HISTORY {
        if history.contracts[i].is_active() & 
           history.contracts[i].is_skill(skill) {
            history_count = history_count + 1;
        }
    }
    let meets_history_requirement = history_count >= minimum_history_size;
    
    meets_threshold & meets_history_requirement
}
```

---

## Part Fourteen: Convergence Criterion (Validation)

### Mathematical Notation

$$\lim_{n \to \infty} \text{Corr}\big(V_t^{(n)}(a), R_t(a)\big) = 1$$

### Noir Implementation

```noir
/// Proves that agent trust values have certain statistical properties.
/// 
/// Used in network validation (not individual eligibility). Allows proving
/// aggregate properties of a population without revealing individual values.
/// 
/// # Arguments
/// * `claimed_mean` - Claimed mean trust value (public)
/// * `claimed_variance` - Claimed variance (public)
/// * `tolerance` - Acceptable deviation from claims (public)
/// * `agent_trust_magnitudes` - Array of trust magnitudes (private)
/// * `agent_count` - Number of agents (private)
/// 
/// # Returns
/// True if actual statistics are within tolerance of claimed values.
/// 
/// # Notes
/// This simplified version uses unsigned magnitudes. A complete
/// implementation would need to handle signed values properly.
fn prove_population_statistics(
    claimed_mean: pub Field,
    claimed_variance: pub Field,
    tolerance: pub Field,
    agent_trust_magnitudes: [Field; MAX_DAO_MEMBERS],
    agent_count: Field,
) -> pub bool {
    // Guard against division by zero
    if agent_count == 0 {
        // Empty population: only valid if claims are zero
        (claimed_mean == 0) & (claimed_variance == 0)
    } else {
        // Compute actual mean
        let mut sum: Field = 0;
        for i in 0..MAX_DAO_MEMBERS {
            if (i as Field) < agent_count {
                sum = sum + agent_trust_magnitudes[i];
            }
        }
        let actual_mean = sum / agent_count;
        
        // Compute actual variance (using absolute difference)
        let mut variance_sum: Field = 0;
        for i in 0..MAX_DAO_MEMBERS {
            if (i as Field) < agent_count {
                let diff = if agent_trust_magnitudes[i] >= actual_mean {
                    agent_trust_magnitudes[i] - actual_mean
                } else {
                    actual_mean - agent_trust_magnitudes[i]
                };
                variance_sum = variance_sum + (diff * diff);
            }
        }
        let actual_variance = variance_sum / agent_count;
        
        // Check claims are within tolerance
        let mean_diff = if actual_mean >= claimed_mean { 
            actual_mean - claimed_mean 
        } else { 
            claimed_mean - actual_mean 
        };
        
        let variance_diff = if actual_variance >= claimed_variance {
            actual_variance - claimed_variance
        } else {
            claimed_variance - actual_variance
        };
        
        (mean_diff <= tolerance) & (variance_diff <= tolerance)
    }
}
```

---

## Part Fifteen: Sybil Resistance Analysis

### Mathematical Notation

$$|h_t(a_{\text{honest}})| > |h_t(a_{\text{sybil}_i})| \quad \forall i$$

### Noir Implementation

```noir
/// Proves that an agent's history size exceeds a threshold.
/// 
/// Sybil attacks involve creating multiple fake identities. Since each
/// identity must complete real contracts to build trust, splitting
/// activity across N identities means each has ~1/N the history of
/// an honest single-identity agent.
/// 
/// # Note
/// This is identical to `prove_minimum_history` from Part Six. Both are
/// included for pedagogical clarity in their respective contexts.
/// 
/// # Arguments
/// * `skill_type` - The skill type to check (public)
/// * `minimum_contracts` - Required minimum contracts (public)
/// * `history` - The agent's contract history (private)
/// 
/// # Returns
/// True if the agent has >= minimum_contracts for this skill type.
fn prove_history_size(
    skill_type: pub Field,
    minimum_contracts: pub Field,
    history: AgentHistory,
) -> pub bool {
    let skill = SkillType::new(skill_type);
    let mut count: Field = 0;
    
    for i in 0..MAX_HISTORY {
        let contract = history.contracts[i];
        if contract.is_active() & contract.is_skill(skill) {
            count = count + 1;
        }
    }
    
    count >= minimum_contracts
}

/// Proves that an agent has been active for a minimum duration.
/// 
/// Sybil identities created recently have shallow histories. This check
/// ensures the agent has contracts spanning a minimum time period.
/// 
/// # Arguments
/// * `skill_type` - The skill type to check (public)
/// * `minimum_age_days` - Required minimum age in days (public)
/// * `current_timestamp` - Current time for age calculation (public)
/// * `history` - The agent's contract history (private)
/// 
/// # Returns
/// True if oldest contract is >= minimum_age_days old.
/// 
/// # Panics
/// Asserts if any contract has completed_at > current_timestamp (prevents underflow attacks).
fn prove_history_depth(
    skill_type: pub Field,
    minimum_age_days: pub Field,
    current_timestamp: pub Field,
    history: AgentHistory,
) -> pub bool {
    let skill = SkillType::new(skill_type);
    
    // Find oldest contract
    let mut oldest_timestamp = current_timestamp;
    let mut has_contracts = false;
    
    for i in 0..MAX_HISTORY {
        let contract = history.contracts[i];
        if contract.is_active() & contract.is_skill(skill) {
            // Prevent underflow attack: contract cannot be from the future
            assert(contract.completed_at <= current_timestamp);
            
            has_contracts = true;
            if contract.completed_at < oldest_timestamp {
                oldest_timestamp = contract.completed_at;
            }
        }
    }
    
    // If no contracts, cannot meet age requirement
    if !has_contracts {
        false
    } else {
        // Safe subtraction: we've verified oldest_timestamp <= current_timestamp
        let age = current_timestamp - oldest_timestamp;
        age >= minimum_age_days
    }
}

/// Proves that an agent has contracts with diverse counterparties.
/// 
/// Sybil networks often have concentrated relationships (fake identities
/// trading with each other). Legitimate agents interact with many
/// different counterparties.
/// 
/// # Arguments
/// * `skill_type` - The skill type to check (public)
/// * `minimum_unique_counterparties` - Required unique counterparties (public)
/// * `history` - The agent's contract history (private)
/// 
/// # Returns
/// True if the agent has >= minimum_unique_counterparties.
/// 
/// # Complexity Warning
/// This function is O(n²) where n = MAX_HISTORY. For MAX_HISTORY = 256,
/// this creates ~65,000 constraint iterations. Consider alternatives:
/// - Use smaller MAX_HISTORY for this specific proof
/// - Use Merkle set membership proofs
/// - Move diversity checking off-circuit
fn prove_counterparty_diversity(
    skill_type: pub Field,
    minimum_unique_counterparties: pub Field,
    history: AgentHistory,
) -> pub bool {
    let skill = SkillType::new(skill_type);
    
    // Track unique counterparties using a seen array
    // We use u32 for indexing since Noir requires integer types for array indices
    let mut unique_count: Field = 0;
    let mut seen: [Field; MAX_HISTORY] = [0; MAX_HISTORY];
    let mut seen_count: u32 = 0;
    
    for i in 0..MAX_HISTORY {
        let contract = history.contracts[i];
        if contract.is_active() & contract.is_skill(skill) {
            let counterparty = contract.counterparty.inner;
            
            // Check if we've already seen this counterparty
            let mut is_new = true;
            for j in 0..MAX_HISTORY {
                if j < seen_count {
                    if seen[j] == counterparty {
                        is_new = false;
                    }
                }
            }
            
            if is_new {
                // Use conditional assignment since we can't index with runtime value
                // This writes to all positions but only the one at seen_count matters
                for k in 0..MAX_HISTORY {
                    if (k as u32) == seen_count {
                        seen[k] = counterparty;
                    }
                }
                seen_count = seen_count + 1;
                unique_count = unique_count + 1;
            }
        }
    }
    
    unique_count >= minimum_unique_counterparties
}
```

---

## Part Sixteen: Complete Example Circuit

Here's a complete, production-style circuit combining the key elements:

```noir
// ============================================
// CONSTANTS
// ============================================

global MAX_HISTORY: u32 = 256;
global PRECISION: Field = 1000000;
global OUTCOME_OFFSET: Field = 100;
global OUTCOME_MAX: Field = 200;
global MAX_DIFFICULTY: Field = 10;
global MINIMUM_THRESHOLD_FACTOR: Field = 100000;

// ============================================
// SIGNED ARITHMETIC
// ============================================

/// Signed integer representation for finite field arithmetic.
/// See Part Two for full documentation.
struct Signed {
    magnitude: Field,
    is_negative: bool,
}

impl Signed {
    fn zero() -> Self {
        Signed { magnitude: 0, is_negative: false }
    }
    
    fn from_positive(value: Field) -> Self {
        Signed { magnitude: value, is_negative: false }
    }
    
    fn from_negative(magnitude: Field) -> Self {
        if magnitude == 0 {
            Signed { magnitude: 0, is_negative: false }
        } else {
            Signed { magnitude, is_negative: true }
        }
    }
    
    fn new(magnitude: Field, is_negative: bool) -> Self {
        let normalized_negative = if magnitude == 0 { false } else { is_negative };
        Signed { magnitude, is_negative: normalized_negative }
    }
    
    fn add(self, other: Signed) -> Self {
        if self.is_negative == other.is_negative {
            Signed::new(self.magnitude + other.magnitude, self.is_negative)
        } else if self.magnitude >= other.magnitude {
            Signed::new(self.magnitude - other.magnitude, self.is_negative)
        } else {
            Signed::new(other.magnitude - self.magnitude, other.is_negative)
        }
    }
    
    fn gte(self, other: Signed) -> bool {
        if !self.is_negative & other.is_negative {
            true
        } else if self.is_negative & !other.is_negative {
            (self.magnitude == 0) & (other.magnitude == 0)
        } else if self.is_negative {
            self.magnitude <= other.magnitude
        } else {
            self.magnitude >= other.magnitude
        }
    }
    
    fn gt(self, other: Signed) -> bool {
        if !self.is_negative & other.is_negative {
            true
        } else if self.is_negative & !other.is_negative {
            false
        } else if self.is_negative {
            self.magnitude < other.magnitude
        } else {
            self.magnitude > other.magnitude
        }
    }
    
    fn is_positive(self) -> bool {
        !self.is_negative & (self.magnitude > 0)
    }
    
    fn is_zero(self) -> bool {
        self.magnitude == 0
    }
}

// ============================================
// FIXED-POINT ARITHMETIC
// ============================================

fn fp_mul(a: Field, b: Field) -> Field {
    (a * b) / PRECISION
}

fn ratio(numerator: Field, denominator: Field) -> Field {
    assert(denominator != 0);
    (numerator * PRECISION) / denominator
}

fn to_fp(n: Field) -> Field {
    n * PRECISION
}

// ============================================
// TYPES
// ============================================

struct AgentId {
    inner: Field,
}

impl AgentId {
    fn new(id: Field) -> Self {
        AgentId { inner: id }
    }
}

struct SkillType {
    inner: Field,
}

impl SkillType {
    fn new(id: Field) -> Self {
        SkillType { inner: id }
    }
    
    fn eq(self, other: SkillType) -> bool {
        self.inner == other.inner
    }
}

struct Contract {
    counterparty: AgentId,
    skill_type: SkillType,
    stake: Field,
    difficulty: Field,
    outcome_offset: Field,
    completed_at: Field,
    weight: Field,
}

impl Contract {
    fn empty() -> Self {
        Contract {
            counterparty: AgentId::new(0),
            skill_type: SkillType::new(0),
            stake: 0,
            difficulty: 0,
            outcome_offset: OUTCOME_OFFSET,
            completed_at: 0,
            weight: 0,
        }
    }
    
    fn outcome(self) -> Signed {
        if self.outcome_offset >= OUTCOME_OFFSET {
            Signed::from_positive(self.outcome_offset - OUTCOME_OFFSET)
        } else {
            Signed::from_negative(OUTCOME_OFFSET - self.outcome_offset)
        }
    }
    
    fn is_skill(self, skill: SkillType) -> bool {
        self.skill_type.eq(skill)
    }
    
    fn is_active(self) -> bool {
        self.weight != 0
    }
    
    fn trust_contribution(self) -> Signed {
        let outcome = self.outcome();
        let magnitude = (self.weight * outcome.magnitude) / 100;
        Signed::new(magnitude, outcome.is_negative)
    }
}

struct AgentHistory {
    contracts: [Contract; MAX_HISTORY],
    count: Field,
    agent_id: AgentId,
}

// ============================================
// LOG APPROXIMATION
// ============================================

global LOG_TABLE_X: [Field; 16] = [
    0, 1, 2, 5, 10, 20, 50, 100, 
    200, 500, 1000, 2000, 5000, 10000, 50000, 100000
];

global LOG_TABLE_Y: [Field; 16] = [
    0, 693147, 1098612, 1791759, 2397895, 3044522, 3931826, 4615121,
    5303305, 6216606, 6908755, 7601402, 8517393, 9210440, 10819878, 11512935
];

fn approx_log1p(x: Field) -> Field {
    let mut lower_idx: u32 = 0;
    let mut upper_idx: u32 = 1;
    
    for i in 0..15 {
        if LOG_TABLE_X[i] <= x {
            if LOG_TABLE_X[i + 1] > x {
                lower_idx = i;
                upper_idx = i + 1;
            }
        }
    }
    
    if x >= LOG_TABLE_X[15] {
        lower_idx = 14;
        upper_idx = 15;
    }
    
    let x_lower = LOG_TABLE_X[lower_idx];
    let x_upper = LOG_TABLE_X[upper_idx];
    let y_lower = LOG_TABLE_Y[lower_idx];
    let y_upper = LOG_TABLE_Y[upper_idx];
    
    if x_upper == x_lower {
        y_lower
    } else {
        assert(x >= x_lower);
        let t = ratio(x - x_lower, x_upper - x_lower);
        y_lower + fp_mul(t, y_upper - y_lower)
    }
}

// ============================================
// WEIGHT VALIDATION
// ============================================

fn verify_weight_bounds(contract: Contract) -> bool {
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

// ============================================
// CORE TRUST COMPUTATION
// ============================================

fn compute_trust_value(
    history: AgentHistory,
    skill_type: SkillType,
) -> Signed {
    let mut trust = Signed::zero();
    
    for i in 0..MAX_HISTORY {
        let contract = history.contracts[i];
        if contract.is_active() & contract.is_skill(skill_type) {
            // CRITICAL: Validate contract bounds before using
            assert(contract.difficulty <= MAX_DIFFICULTY);
            assert(contract.outcome_offset <= OUTCOME_MAX);
            assert(verify_weight_bounds(contract));
            
            trust = trust.add(contract.trust_contribution());
        }
    }
    
    trust
}

fn calculate_threshold(stake: Field, difficulty: Field) -> Field {
    // Validate difficulty to prevent overflow
    assert(difficulty <= MAX_DIFFICULTY);
    
    let stake_factor = approx_log1p(stake);
    let minimum_threshold = fp_mul(stake_factor, MINIMUM_THRESHOLD_FACTOR);
    let difficulty_threshold = fp_mul(stake_factor, to_fp(difficulty));
    
    if minimum_threshold > difficulty_threshold {
        minimum_threshold
    } else {
        difficulty_threshold
    }
}

// ============================================
// DAO AGGREGATION FUNCTIONS
// ============================================

global MAX_DAO_MEMBERS: u32 = 64;

fn aggregate_sum(values: [Signed; MAX_DAO_MEMBERS], count: Field) -> Signed {
    let mut sum = Signed::zero();
    for i in 0..MAX_DAO_MEMBERS {
        if (i as Field) < count {
            sum = sum.add(values[i]);
        }
    }
    sum
}

fn aggregate_average(values: [Signed; MAX_DAO_MEMBERS], count: Field) -> Signed {
    if count == 0 {
        Signed::zero()
    } else {
        let sum = aggregate_sum(values, count);
        Signed::new(sum.magnitude / count, sum.is_negative)
    }
}

fn aggregate_minimum(values: [Signed; MAX_DAO_MEMBERS], count: Field) -> Signed {
    if count == 0 {
        Signed::zero()
    } else {
        let mut min_val = values[0];
        
        for i in 1..MAX_DAO_MEMBERS {
            if (i as Field) < count {
                // Update min if current value is less than current minimum
                // values[i] < min_val  ⟺  !values[i].gte(min_val)
                if !values[i].gte(min_val) {
                    min_val = values[i];
                }
            }
        }
        min_val
    }
}

fn aggregate_maximum(values: [Signed; MAX_DAO_MEMBERS], count: Field) -> Signed {
    if count == 0 {
        Signed::zero()
    } else {
        let mut max_val = values[0];
        
        for i in 1..MAX_DAO_MEMBERS {
            if (i as Field) < count {
                if values[i].gt(max_val) {
                    max_val = values[i];
                }
            }
        }
        max_val
    }
}

// ============================================
// MAIN ENTRY POINT
// ============================================

/// Main eligibility proof circuit.
/// 
/// Proves that the agent's trust value for the given skill type
/// meets or exceeds the threshold for a contract with the given
/// stake and difficulty.
fn main(
    skill_type_inner: pub Field,
    contract_stake: pub Field,
    contract_difficulty: pub Field,
    history: AgentHistory,
) -> pub bool {
    let skill_type = SkillType::new(skill_type_inner);
    
    let trust = compute_trust_value(history, skill_type);
    let threshold = calculate_threshold(contract_stake, contract_difficulty);
    
    trust.gte(Signed::from_positive(threshold))
}

// ============================================
// TESTS
// ============================================

#[test]
fn test_signed_zero() {
    let zero = Signed::zero();
    assert(zero.magnitude == 0);
    assert(!zero.is_negative);
    assert(zero.is_zero());
    assert(!zero.is_positive());
}

#[test]
fn test_signed_positive() {
    let pos = Signed::from_positive(100);
    assert(pos.magnitude == 100);
    assert(!pos.is_negative);
    assert(pos.is_positive());
}

#[test]
fn test_signed_negative() {
    let neg = Signed::from_negative(50);
    assert(neg.magnitude == 50);
    assert(neg.is_negative);
    assert(!neg.is_positive());
}

#[test]
fn test_signed_add_same_sign() {
    // Positive + Positive
    let a = Signed::from_positive(100);
    let b = Signed::from_positive(50);
    let sum = a.add(b);
    assert(sum.magnitude == 150);
    assert(!sum.is_negative);
    
    // Negative + Negative
    let c = Signed::from_negative(100);
    let d = Signed::from_negative(50);
    let sum2 = c.add(d);
    assert(sum2.magnitude == 150);
    assert(sum2.is_negative);
}

#[test]
fn test_signed_add_different_sign() {
    // Positive + Negative (positive result)
    let a = Signed::from_positive(100);
    let b = Signed::from_negative(30);
    let sum = a.add(b);
    assert(sum.magnitude == 70);
    assert(!sum.is_negative);
    
    // Positive + Negative (negative result)
    let c = Signed::from_positive(30);
    let d = Signed::from_negative(100);
    let sum2 = c.add(d);
    assert(sum2.magnitude == 70);
    assert(sum2.is_negative);
}

#[test]
fn test_signed_add_to_zero() {
    let a = Signed::from_positive(50);
    let b = Signed::from_negative(50);
    let sum = a.add(b);
    assert(sum.magnitude == 0);
    assert(!sum.is_negative);  // Zero should not be negative
}

#[test]
fn test_signed_gte() {
    let pos = Signed::from_positive(100);
    let neg = Signed::from_negative(50);
    let zero = Signed::zero();
    
    assert(pos.gte(neg));    // 100 >= -50
    assert(pos.gte(zero));   // 100 >= 0
    assert(zero.gte(neg));   // 0 >= -50
    assert(!neg.gte(pos));   // -50 >= 100 is false
    assert(!neg.gte(zero));  // -50 >= 0 is false
}

#[test]
fn test_empty_history() {
    let empty_history = AgentHistory {
        contracts: [Contract::empty(); MAX_HISTORY],
        count: 0,
        agent_id: AgentId::new(1),
    };
    
    let skill = SkillType::new(1);
    let trust = compute_trust_value(empty_history, skill);
    
    assert(trust.magnitude == 0);
    assert(!trust.is_negative);
}

#[test]
fn test_positive_trust() {
    let mut contracts: [Contract; MAX_HISTORY] = [Contract::empty(); MAX_HISTORY];
    
    contracts[0] = Contract {
        counterparty: AgentId::new(2),
        skill_type: SkillType::new(1),
        stake: 1000,
        difficulty: 5,
        outcome_offset: 200,  // +100 outcome (success)
        completed_at: 1000,
        weight: PRECISION,    // Weight of 1.0
    };
    
    let history = AgentHistory {
        contracts,
        count: 1,
        agent_id: AgentId::new(1),
    };
    
    let skill = SkillType::new(1);
    let trust = compute_trust_value(history, skill);
    
    assert(trust.is_positive());
    // Trust = 1.0 * 100 / 100 = 1.0 (scaled = PRECISION)
    assert(trust.magnitude == PRECISION);
}

#[test]
fn test_negative_trust() {
    let mut contracts: [Contract; MAX_HISTORY] = [Contract::empty(); MAX_HISTORY];
    
    contracts[0] = Contract {
        counterparty: AgentId::new(2),
        skill_type: SkillType::new(1),
        stake: 1000,
        difficulty: 5,
        outcome_offset: 0,    // -100 outcome (failure)
        completed_at: 1000,
        weight: PRECISION,
    };
    
    let history = AgentHistory {
        contracts,
        count: 1,
        agent_id: AgentId::new(1),
    };
    
    let skill = SkillType::new(1);
    let trust = compute_trust_value(history, skill);
    
    assert(trust.is_negative);
    assert(trust.magnitude == PRECISION);
}

#[test]
fn test_cancellation() {
    let mut contracts: [Contract; MAX_HISTORY] = [Contract::empty(); MAX_HISTORY];
    
    // Success: +100 outcome
    contracts[0] = Contract {
        counterparty: AgentId::new(2),
        skill_type: SkillType::new(1),
        stake: 1000,
        difficulty: 5,
        outcome_offset: 200,
        completed_at: 1000,
        weight: PRECISION,
    };
    
    // Failure: -100 outcome
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
    
    let skill = SkillType::new(1);
    let trust = compute_trust_value(history, skill);
    
    // +1.0 + (-1.0) = 0
    assert(trust.is_zero());
}

#[test]
fn test_eligibility_pass() {
    let mut contracts: [Contract; MAX_HISTORY] = [Contract::empty(); MAX_HISTORY];
    
    // Add 10 successful contracts with weight 1.0 each
    for i in 0..10 {
        contracts[i] = Contract {
            counterparty: AgentId::new(2),
            skill_type: SkillType::new(1),
            stake: 1000,
            difficulty: 5,
            outcome_offset: 200,
            completed_at: 1000,
            weight: PRECISION,
        };
    }
    
    let history = AgentHistory {
        contracts,
        count: 10,
        agent_id: AgentId::new(1),
    };
    
    // Trust = 10 * PRECISION = 10,000,000
    // Threshold for stake=100, difficulty=1:
    //   log(101) ≈ 4.615 → 4615121 scaled
    //   threshold = 4615121 * 1 = 4615121
    // 10,000,000 >= 4,615,121 → eligible
    let result = main(1, 100, 1, history);
    assert(result);
}

#[test]
fn test_eligibility_fail() {
    let empty_history = AgentHistory {
        contracts: [Contract::empty(); MAX_HISTORY],
        count: 0,
        agent_id: AgentId::new(1),
    };
    
    // Zero trust cannot meet any positive threshold
    let result = main(1, 1000, 5, empty_history);
    assert(!result);
}

#[test]
fn test_threshold_boundary() {
    let mut contracts: [Contract; MAX_HISTORY] = [Contract::empty(); MAX_HISTORY];
    
    // Create exactly enough trust to meet threshold
    // For stake=10, difficulty=1: threshold = log(11) * 1 ≈ 2.398 * PRECISION ≈ 2397895
    // Need 2.398 trust, so ~2.4 weight worth of +100 outcomes
    // With weight=PRECISION, each +100 outcome contributes PRECISION to trust
    // So we need ~2.4 contracts, round up to 3
    for i in 0..3 {
        contracts[i] = Contract {
            counterparty: AgentId::new(2),
            skill_type: SkillType::new(1),
            stake: 1000,
            difficulty: 5,
            outcome_offset: 200,
            completed_at: 1000,
            weight: PRECISION,
        };
    }
    
    let history = AgentHistory {
        contracts,
        count: 3,
        agent_id: AgentId::new(1),
    };
    
    // Trust = 3 * PRECISION = 3,000,000
    // Threshold for stake=10, difficulty=1 ≈ 2,397,895
    // 3,000,000 >= 2,397,895 → eligible
    let result = main(1, 10, 1, history);
    assert(result);
}

// ============================================
// DAO AGGREGATION TESTS
// ============================================

#[test]
fn test_aggregate_minimum_finds_smallest() {
    // Test that aggregate_minimum correctly finds the minimum value
    let mut values: [Signed; MAX_DAO_MEMBERS] = [Signed::zero(); MAX_DAO_MEMBERS];
    values[0] = Signed::from_positive(100);  // 100
    values[1] = Signed::from_positive(50);   // 50 <- minimum
    values[2] = Signed::from_positive(75);   // 75
    
    let result = aggregate_minimum(values, 3);
    assert(result.magnitude == 50);
    assert(!result.is_negative);
}

#[test]
fn test_aggregate_minimum_with_negatives() {
    // Test minimum with negative values: -30 < -10 < 20
    let mut values: [Signed; MAX_DAO_MEMBERS] = [Signed::zero(); MAX_DAO_MEMBERS];
    values[0] = Signed::from_positive(20);   // 20
    values[1] = Signed::from_negative(10);   // -10
    values[2] = Signed::from_negative(30);   // -30 <- minimum
    
    let result = aggregate_minimum(values, 3);
    assert(result.magnitude == 30);
    assert(result.is_negative);  // -30
}

#[test]
fn test_aggregate_maximum_finds_largest() {
    // Test that aggregate_maximum correctly finds the maximum value
    let mut values: [Signed; MAX_DAO_MEMBERS] = [Signed::zero(); MAX_DAO_MEMBERS];
    values[0] = Signed::from_positive(50);   // 50
    values[1] = Signed::from_positive(100);  // 100 <- maximum
    values[2] = Signed::from_positive(75);   // 75
    
    let result = aggregate_maximum(values, 3);
    assert(result.magnitude == 100);
    assert(!result.is_negative);
}

#[test]
fn test_aggregate_maximum_with_negatives() {
    // Test maximum with negative values: -30 < -10 < 20
    let mut values: [Signed; MAX_DAO_MEMBERS] = [Signed::zero(); MAX_DAO_MEMBERS];
    values[0] = Signed::from_negative(30);   // -30
    values[1] = Signed::from_negative(10);   // -10
    values[2] = Signed::from_positive(20);   // 20 <- maximum
    
    let result = aggregate_maximum(values, 3);
    assert(result.magnitude == 20);
    assert(!result.is_negative);  // +20
}

#[test]
fn test_aggregate_empty() {
    // Empty DAO should return zero for all aggregations
    let values: [Signed; MAX_DAO_MEMBERS] = [Signed::zero(); MAX_DAO_MEMBERS];
    
    let sum = aggregate_sum(values, 0);
    assert(sum.is_zero());
    
    let avg = aggregate_average(values, 0);
    assert(avg.is_zero());
    
    let min = aggregate_minimum(values, 0);
    assert(min.is_zero());
    
    let max = aggregate_maximum(values, 0);
    assert(max.is_zero());
}
```

---

## Summary of Equation Mappings

| Math Notation | Noir Implementation |
|---------------|---------------------|
| $q\langle T \rangle$ | `AgentHistory` or `DAOMembership` structs |
| $\text{Agent}(t, h_t)$ | `AgentHistory { contracts, count, agent_id }` |
| $\text{DAO}(\{q\langle T \rangle\})$ | `DAOMembership { members, member_trust_values, aggregation_type }` |
| $V_t(q)$ | `compute_trust_value(history, skill_type) -> Signed` |
| $\sum_{c \in h_t} \omega(c) \cdot \text{outcome}(c)$ | Loop over `contracts` array, sum `trust_contribution()` |
| $\Phi(\{V_t(q) : q \in S\})$ | `compute_dao_trust()` with aggregation functions |
| $h_t^{(n+1)} = h_t^{(n)} \cup \{c_n\}$ | `add_to_history()` / Note creation pattern |
| $V_t^{(n+1)} = V_t^{(n)} + \omega(c_n) \cdot \text{outcome}(c_n)$ | `update_trust()` using `Signed.add()` |
| $\omega(c) = f(s, d, V_t, \text{recency})$ | `compute_weight()` or pre-computed in `Contract.weight` |
| $\text{eligible}(a, c) \iff V_t(a) \geq \theta(c)$ | `prove_eligibility()` — THE core circuit |
| $\theta(c) = \log(1+s) \cdot d$ | `calculate_threshold()` with `approx_log1p()` |
| $\text{Corr}(V_t^{(n)}, R_t)$ | Off-circuit simulation; `prove_population_statistics()` for bounds |
| $\|h_t(a)\|$ comparisons | `prove_history_size()`, `prove_history_depth()` |

---

## Key Architectural Decisions

### 1. Signed Arithmetic via Struct

Since Noir's `Field` type is always positive (elements of a prime field), we use a `Signed` struct with separate magnitude and sign. This correctly handles negative trust values. The normalization invariant (zero is never negative) simplifies comparison logic.

### 2. Pre-computed Weights with Mandatory Validation

The weighting function involves transcendental functions (log, exp, tanh) that are expensive in ZK. We compute weights off-circuit at contract completion time and store them in the Contract struct. **Critically, `compute_trust_value()` validates all contract bounds (difficulty, outcome, weight) before processing to prevent malicious provers from forging inflated trust scores.** Without this validation, an attacker could pass any eligibility threshold by providing contracts with artificially high weights.

### 3. Fixed-Size Arrays

All arrays have compile-time fixed sizes. Unused slots are marked with `weight = 0`. This is a fundamental ZK constraint that enables circuit compilation.

### 4. Scaled Integer Arithmetic

All "decimal" values use fixed-point representation: `actual_value * PRECISION`. The `ratio()` function handles division of raw integers to produce properly scaled results, avoiding the double-scaling bug in naive implementations.

### 5. Privacy-First Design

The default is maximum privacy. All history details are private inputs. Public outputs are boolean (eligible/not eligible) or bounded ranges. The verifier learns nothing about contract count, outcomes, counterparties, or exact trust score.

### 6. Incremental Trust Updates

Trust evolution uses the incremental formula `V_t^(n+1) = V_t^(n) + contribution`. Full history recomputation is only needed for proofs, not for state updates. This enables efficient caching in the `HistoryState` struct.

---

## Aztec-Specific Integration Notes

When integrating with Aztec's full stack:

1. **Notes**: Each Contract becomes a private note owned by the Avatar
2. **Nullifiers**: Reading/spending notes requires nullifier generation to prevent double-spending
3. **Public/Private Function Split**: Some operations may need public functions for state commitment
4. **Recursive Proofs**: DAO trust may require recursive proof verification for member proofs
5. **AztecAddress**: Replace `AgentId` with proper Aztec address types from `aztec.nr`

This document provides the mathematical core. Full Aztec integration requires additional infrastructure covered in Aztec.nr documentation.
