# Blockchain Selection for Quantum of Trust Implementation

## Why We Chose Noir on the Aztec Blockchain

---

## Executive Summary

The Quantum of Trust (q\<T\>) framework requires a blockchain platform that treats privacy as a first-class architectural concern rather than an afterthought. After evaluating the major blockchain ecosystems, we selected **Noir** as our smart contract language and **Aztec** as our target blockchain. This document explains the technical reasoning behind this decision.

Aztec positions itself as a **"private world computer"**—a Layer 2 rollup on Ethereum that enables fully programmable, encrypted smart contracts. This framing captures exactly what q\<T\> needs: not just a blockchain with privacy features, but a complete execution environment where privacy is the default.

The core insight driving our selection: **q\<T\> doesn't just need privacy features—it needs privacy as the default execution model.** Traditional blockchains with optional privacy layers cannot provide the guarantees our framework requires. Aztec's architecture, where all execution is private by default with selective disclosure, maps directly onto our "Avatar-first" design philosophy.

---

## Part One: Requirements Analysis

### What q\<T\> Demands from a Blockchain

The Quantum of Trust framework has non-negotiable technical requirements that eliminate most blockchain platforms from consideration:

#### 1. Native Zero-Knowledge Proof Support

q\<T\>'s core operation is the **eligibility proof**: proving that an Avatar's trust quotient exceeds a threshold *without revealing the underlying contract history*. This isn't an optional feature—it's the fundamental primitive.

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

This requires a platform where ZK proofs are the native execution model, not a bolted-on feature.

#### 2. Private State by Default

Every Avatar maintains private state:
- Contract history (who they've worked with, outcomes, stakes)
- Trust quotients per skill type
- DAO memberships and roles

This state must remain encrypted on-chain. Public blockchains where all state is visible (Ethereum, Solana, etc.) cannot support this without complex and gas-intensive encryption schemes.

#### 3. Composable Privacy

q\<T\> supports hierarchical structures:
- Avatars contain contract histories
- DAOs contain Avatars (and other DAOs)
- Trust aggregation functions operate over private member data

The blockchain must support **recursive proofs**—proving statements about proofs—to enable DAO-level trust computation without revealing individual member trust values.

#### 4. Programmable Disclosure

Privacy in q\<T\> isn't absolute—it's *controlled*. An Avatar might choose to:
- Prove eligibility for a contract (binary disclosure)
- Prove trust falls within a range (bounded disclosure)
- Reveal specific contracts to a counterparty (selective disclosure)
- Attach real-name identity to their Avatar (full disclosure)

The platform must support fine-grained control over what information is revealed and to whom.

#### 5. Smart Contract Automation

Contract execution, outcome recording, and trust updates must be automated and trustless. The platform needs:
- Turing-complete (or sufficiently expressive) smart contracts
- Atomic transactions (stake + contract + outcome as single unit)
- Time-based triggers (deadlines, recency decay)

#### 6. Sybil Resistance Economics

The platform's fee structure must make Sybil attacks economically unattractive. Creating fake Avatars to game trust quotients should cost more than honest participation.

---

## Part Two: Platform Evaluation

### Platforms Considered

| Platform | Privacy Model | ZK Support | Smart Contracts | Verdict |
|----------|---------------|------------|-----------------|---------|
| Ethereum | Public by default | zkEVMs (external) | Full EVM | ❌ Privacy bolt-on |
| Solana | Public by default | None native | Limited | ❌ No ZK, public state |
| Polygon zkEVM | Public with ZK rollup | Validity proofs | EVM compatible | ❌ Public state, ZK for scaling only |
| zkSync | Public with ZK rollup | Validity proofs | EVM compatible | ❌ Public state, ZK for scaling only |
| Mina | ZK-native | SNARKs | Limited (zkApps) | ⚠️ Limited expressiveness |
| Aleo | Private by default | ZK-native | Leo language | ⚠️ Newer ecosystem |
| **Aztec** | **Private by default** | **ZK-native (Noir)** | **Full programmability** | ✅ **Selected** |

### Why Not Ethereum + Privacy Layer?

Ethereum's architecture fundamentally assumes public state. Privacy solutions like Tornado Cash or Aztec Connect (the bridge) operate *on top of* Ethereum's public ledger. This creates problems:

1. **Metadata leakage**: Transaction timing, gas costs, and interaction patterns are visible
2. **Complexity overhead**: Every private operation requires explicit encryption/decryption
3. **Gas costs**: ZK proof verification is expensive on L1
4. **Composability limits**: Private contracts can't easily interact with public DeFi

For q\<T\>, we don't want privacy as a feature—we want it as the default.

### Why Not zkSync/Polygon zkEVM?

These platforms use ZK proofs for **scaling** (validity proofs for rollups), not for **privacy**. All transaction data is still publicly visible on the rollup; the ZK proofs merely attest that the rollup operator executed transactions correctly.

This is the opposite of what q\<T\> needs. We need ZK proofs to hide data from verifiers, not to compress public data for cheaper verification.

### Why Not Mina?

Mina's zkApps are interesting—the entire blockchain state is compressed into a single ZK proof. However:

1. **Limited expressiveness**: zkApps are constrained compared to general-purpose smart contracts
2. **No private state storage**: Mina doesn't have a native model for encrypted on-chain state
3. **Ecosystem maturity**: Developer tooling and documentation lag behind Aztec

### Why Not Aleo?

Aleo is actually a strong candidate with similar goals to Aztec. We considered it seriously. The deciding factors for Aztec:

1. **Ethereum alignment**: Aztec is building toward Ethereum L2, enabling future interoperability with DeFi
2. **Noir's maturity**: Noir has more comprehensive documentation and tooling than Leo
3. **Note model**: Aztec's UTXO-like "note" model maps cleanly onto q\<T\>'s contract history structure
4. **Team track record**: Aztec team has shipped multiple production systems (Aztec Connect, earlier versions)

---

## Part Three: Why Aztec

### Architecture Alignment

Aztec's architecture mirrors q\<T\>'s design philosophy at a fundamental level:

| q\<T\> Concept | Aztec Equivalent |
|----------------|------------------|
| Avatar (private identity) | Account with private state |
| Contract history | Private notes (UTXO-like model) |
| Trust quotient | Computed from private notes |
| Eligibility proof | Noir circuit proving threshold |
| DAO membership | Note containing member list |
| Selective disclosure | Public/private function split |

This isn't coincidence—both systems are built on the principle that **privacy should be default, disclosure should be explicit**.

### Hybrid State Model

Aztec uniquely supports both private and public state:

- **Private State**: An encrypted UTXO-like model for sensitive data. Each piece of private state is a "note" that only the owner can decrypt and spend.
- **Public State**: An account-based model similar to Ethereum for data that should be visible (contract registries, public parameters, etc.).

This hybrid approach maps perfectly onto q\<T\>:
- **Private**: Avatar trust quotients, contract histories, DAO memberships
- **Public**: Contract listings, skill type definitions, threshold requirements

### Client-Side Proving

A critical architectural feature: **users execute private contract logic locally on their own devices**, generating ZK proofs without revealing underlying data to anyone—not even Aztec's sequencers.

For q\<T\>, this means:
1. Avatar owners compute their trust values locally
2. The proof of "trust ≥ threshold" is generated on the user's device
3. Only the proof (not the history) is submitted to the network
4. Sequencers verify proofs without ever seeing private data

This is fundamentally different from systems where a trusted server performs computation on your behalf. In Aztec, **you don't trust anyone with your data**—cryptography enforces privacy.

### The Note Model

Aztec uses a UTXO-like model called "notes" for private state:

```
Note = {
  owner: AztecAddress,      // Who can read/spend this note
  data: encrypted_payload,  // The actual content
  nullifier: hash,          // Prevents double-spending
}
```

For q\<T\>, each completed contract becomes a note:

```
ContractNote = {
  owner: Avatar's AztecAddress,
  data: {
    counterparty: AgentId,
    skill_type: SkillType,
    stake: Field,
    difficulty: Field,
    outcome_offset: Field,
    weight: Field,
  },
  nullifier: derived from note + owner's key
}
```

This model provides:

1. **Privacy**: Note contents are encrypted; only the owner can decrypt
2. **Ownership**: Notes are bound to addresses via cryptographic keys
3. **Non-fungibility**: Each contract is a distinct note (unlike fungible tokens)
4. **Spendability**: Notes can be "spent" (nullified) when updating trust state

### Private vs Public Functions

Aztec contracts have two types of functions:

```noir
// Private function - executes locally, generates proof
#[aztec(private)]
fn prove_eligibility(skill_type: Field, threshold: Field) -> bool {
    // Access private notes
    // Compute trust value
    // Return eligibility (public output)
}

// Public function - executes on sequencer, visible to all
#[aztec(public)]
fn record_contract_completion(contract_id: Field, outcome: Field) {
    // Update public registry
    // Emit events
}
```

This split maps directly onto q\<T\>'s disclosure model:
- Private functions for eligibility proofs, trust computation
- Public functions for contract registration, outcome finalization

### Recursive Proofs for DAOs

DAO trust aggregation requires proving statements about member trust values without revealing them:

$$V_t(\text{DAO}(S)) = \Phi\left(\{V_t(q) : q \in S\}\right)$$

Aztec supports recursive proof verification:

1. Each member generates a proof of their trust value (or a range proof)
2. The DAO aggregation circuit takes these proofs as inputs
3. The aggregation circuit verifies member proofs and computes aggregate trust
4. Final output: "This DAO's aggregate trust ≥ threshold"

No individual member's exact trust value is revealed—not even to other DAO members.

---

## Part Four: Why Noir

### Language Design Philosophy

Noir is a high-level domain-specific language (DSL) that abstracts the complex mathematics of zero-knowledge proofs. Developed by Aztec Labs, it's designed specifically for ZK circuits while remaining accessible to developers familiar with modern systems languages.

Unlike general-purpose languages adapted for ZK (Solidity + ZK coprocessors), Noir's semantics align with circuit constraints:

| Noir Feature | ZK Circuit Requirement |
|--------------|------------------------|
| Fixed-size arrays | Circuits have static structure |
| Bounded loops | Loop unrolling at compile time |
| Field type | Native finite field arithmetic |
| No dynamic allocation | Deterministic memory layout |
| Explicit public/private | Clear prover/verifier separation |

### Rust-Inspired Syntax

Noir's syntax is heavily inspired by Rust, making it immediately familiar to systems programmers:

```noir
struct Signed {
    magnitude: Field,
    is_negative: bool,
}

impl Signed {
    fn add(self, other: Signed) -> Self {
        // Rust-like method syntax
        if self.is_negative == other.is_negative {
            Signed::new(self.magnitude + other.magnitude, self.is_negative)
        } else {
            // Pattern matching, immutability by default
            ...
        }
    }
}
```

This lowers the barrier to entry significantly compared to circuit-specific languages.

### Backend Agnostic: A Universal Adapter

While Noir is the native language for Aztec, it's designed as a **"universal adapter"** that can compile circuits to work with various proving systems:

```
Noir Source Code
      ↓
    Nargo (compiler)
      ↓
    ACIR (Abstract Circuit Intermediate Representation)
      ↓
    Backend-specific constraints
      ↓
    [Barretenberg | Plonk | Groth16 | ...]
```

**ACIR** (Abstract Circuit Intermediate Representation) is the key abstraction. Noir compiles to ACIR, which can then be transformed into specific cryptographic constraints for different proving systems.

This means:
- **Portability**: Our q\<T\> circuits could theoretically target other ZK platforms
- **Future-proofing**: As proving systems improve, we benefit without rewriting code
- **Optimization**: Different backends can be chosen for different trade-offs (proof size vs. proving time)

### Development Tooling

Noir provides production-ready development tools:

**Nargo**: The command-line interface for Noir development
- `nargo compile` — Compile Noir programs to ACIR
- `nargo prove` — Generate ZK proofs
- `nargo verify` — Verify proofs
- `nargo test` — Run unit tests without proof generation

**NoirJS**: Browser and mobile integration
- Generate and verify ZK proofs directly in web browsers
- Enables client-side proving for web applications
- Critical for q\<T\>'s user-facing interfaces

### Type Safety

Noir's type system catches errors at compile time that would be vulnerabilities in other languages:

```noir
// Noir enforces that private inputs stay private
fn prove_threshold(
    threshold: pub Field,      // Verifier sees this
    history: [Contract; 256],  // Verifier learns nothing
) -> pub bool {
    // Compiler ensures 'history' never leaks to public outputs
}
```

### Developer Experience

Compared to raw circuit languages (R1CS, Plonk constraints), Noir provides:

1. **Familiar syntax**: Rust-like, easy for systems programmers
2. **Standard library**: Common operations pre-implemented
3. **Testing framework**: Unit tests run without proof generation
4. **Debugging**: Error messages reference source code, not constraint indices
5. **Documentation**: Comprehensive guides and examples

### The Complete Example

Our Noir implementation spans 3,000+ lines covering:

- Signed arithmetic for negative trust values
- Fixed-point arithmetic for weighted calculations
- Contract validation to prevent proof forgery
- DAO aggregation functions (sum, average, min, max)
- Sybil resistance checks (history size, depth, diversity)
- Full test suite

This implementation demonstrates that Noir is expressive enough for complex financial/reputation logic, not just simple token transfers.

---

## Part Five: Technical Deep Dive

### How Eligibility Proofs Work

The core q\<T\> operation in Noir:

```noir
fn prove_eligibility(
    skill_type: pub Field,
    contract_stake: pub Field,
    contract_difficulty: pub Field,
    history: AgentHistory,  // Private - 256 contracts max
) -> pub bool {
    let skill = SkillType::new(skill_type);
    
    // Compute trust from private history
    let trust = compute_trust_value(history, skill);
    
    // Compute threshold from public contract parameters
    let threshold = calculate_threshold(contract_stake, contract_difficulty);
    
    // Return eligibility (only output verifier sees)
    trust.gte(Signed::from_positive(threshold))
}
```

What the verifier learns: "This Avatar is eligible for this contract."

What the verifier does NOT learn:
- How many contracts are in the Avatar's history
- Who the Avatar has worked with
- What outcomes those contracts had
- The Avatar's exact trust score
- Whether the Avatar barely passed or greatly exceeded the threshold

### Handling Negative Trust

Trust values can be negative (actively distrusted agents). Noir's `Field` type is always positive (finite field elements), so we implement signed arithmetic:

```noir
struct Signed {
    magnitude: Field,
    is_negative: bool,
}

impl Signed {
    fn add(self, other: Signed) -> Self {
        if self.is_negative == other.is_negative {
            // Same sign: add magnitudes
            Signed::new(self.magnitude + other.magnitude, self.is_negative)
        } else if self.magnitude >= other.magnitude {
            // Different signs: subtract, keep larger's sign
            Signed::new(self.magnitude - other.magnitude, self.is_negative)
        } else {
            Signed::new(other.magnitude - self.magnitude, other.is_negative)
        }
    }
}
```

### Security: Mandatory Validation

A critical security requirement: provers provide private inputs, but we must validate those inputs are well-formed. Without validation, a malicious prover could forge trust scores:

```noir
fn compute_trust_value(history: AgentHistory, skill_type: SkillType) -> Signed {
    let mut trust = Signed::zero();
    
    for i in 0..MAX_HISTORY {
        let contract = history.contracts[i];
        
        if contract.is_active() & contract.is_skill(skill_type) {
            // CRITICAL: Validate before using
            assert(contract.difficulty <= MAX_DIFFICULTY);
            assert(contract.outcome_offset <= OUTCOME_MAX);
            assert(verify_weight_bounds(contract));
            
            trust = trust.add(contract.trust_contribution());
        }
    }
    
    trust
}
```

Without these assertions, an attacker could pass `weight = 10^20` and bypass any threshold.

---

## Part Six: Aztec Roadmap Alignment

### Current Status (Late 2025)

Aztec has reached a major milestone with the launch of its production network:

- ✅ **Ignition Chain**: Aztec's first fully decentralized L2 mainnet on Ethereum, launched late 2025
- ✅ **Noir language**: Stable and well-documented; Aztec's entire core cryptography stack has been rewritten in Noir for improved safety and auditability
- ✅ **AZTEC Token**: Launched late 2025 for staking, governance, and incentivizing network operators (sequencers and provers)
- ✅ **Private/public function model**: Production-ready
- ✅ **Note encryption and nullifiers**: Production-ready
- ✅ **Client-side proving**: Users can generate proofs locally

### Network Economics

The AZTEC token creates aligned incentives for network participants:

| Role | Incentive |
|------|-----------|
| **Sequencers** | Earn tokens for ordering and batching transactions |
| **Provers** | Earn tokens for generating validity proofs |
| **Stakers** | Secure the network and earn rewards |
| **Governance** | Token holders vote on protocol upgrades |

For q\<T\>, this means:
- Sustainable infrastructure for eligibility proof verification
- Economic security for the trust network
- Decentralized operation without single points of failure

### Why Build Now?

With Ignition Chain live, building on Aztec offers:

1. **Production deployment**: q\<T\> can launch on a live, decentralized network
2. **First-mover positioning**: q\<T\> can be a flagship Aztec application demonstrating privacy-preserving reputation
3. **Ecosystem growth**: Aztec's DeFi and identity ecosystems provide natural integration points
4. **Battle-tested infrastructure**: Ignition Chain's launch validates the technology stack

### Aztec Ecosystem Alignment

Aztec's documented use cases align naturally with q\<T\>:

| Aztec Use Case | q\<T\> Application |
|----------------|-------------------|
| **Private DeFi** | Shielded reputation-based lending (trust score as collateral) |
| **Anonymous Voting** | DAO governance where votes are weighted by trust |
| **Confidential Identity** | Avatar-based identity with selective disclosure |
| **Gaming with Hidden Information** | Agent marketplaces with private capability proofs |

q\<T\> extends Aztec's privacy primitives into the reputation domain—a natural fit for the "private world computer" vision.

### Migration Path

With Ignition Chain live, our migration path is straightforward:

1. **Development**: Build and test on Aztec Sandbox (local environment)
2. **Staging**: Deploy to testnet for integration testing
3. **Production**: Deploy to Ignition Chain mainnet

Noir's backend-agnostic design provides additional insurance:
- Noir circuits can target other backends (Barretenberg, custom provers)
- Core logic separates from Aztec-specific integration
- Mathematical framework is blockchain-agnostic

---

## Part Seven: Risk Analysis

### Aztec-Specific Risks

| Risk | Severity | Mitigation |
|------|----------|------------|
| Network stability (new mainnet) | Medium | Monitor Ignition Chain performance; maintain testnet fallback |
| Breaking changes | Low | Noir is stable; core crypto rewritten in Noir for auditability |
| Ecosystem size | Low | Privacy apps have strong product-market fit; ecosystem growing |
| Sequencer/prover availability | Low | Token incentives align operator economics |

### ZK-Specific Risks

| Risk | Severity | Mitigation |
|------|----------|------------|
| Proof generation time | Medium | Optimize circuits; use client-side proving |
| Circuit bugs | High | Extensive testing; formal verification where possible |
| Cryptographic breaks | Low | Use battle-tested primitives; follow cryptography research |

### General Risks

| Risk | Severity | Mitigation |
|------|----------|------------|
| Regulatory uncertainty | Medium | Privacy-preserving != anonymous; support compliance features |
| User adoption | Medium | Focus on AI agent use case where privacy is clearly valuable |

---

## Conclusion

We chose Noir on Aztec because **no other platform treats privacy as a first-class architectural concern while providing the programmability q\<T\> requires**.

The decision matrix:

| Requirement | Aztec/Noir | Best Alternative | Gap |
|-------------|------------|------------------|-----|
| Private state by default | ✅ Native | Aleo | Minor (ecosystem maturity) |
| ZK eligibility proofs | ✅ Native | Ethereum + coprocessor | Major (complexity) |
| Recursive proofs (DAOs) | ✅ Supported | Mina | Major (expressiveness) |
| Programmable disclosure | ✅ Private/public split | Secret Network | Moderate (trust model) |
| Ethereum alignment | ✅ L2 roadmap | Aleo | Moderate (interop) |
| Developer experience | ✅ Rust-like Noir | Cairo | Minor (learning curve) |

The Quantum of Trust framework and Aztec share a fundamental insight: **privacy should be the default, not an afterthought**. This philosophical alignment, combined with Aztec's technical capabilities and roadmap, makes it the clear choice for q\<T\> implementation.

---

## Appendix: Resources

### Aztec Documentation
- [Aztec Docs](https://docs.aztec.network/)
- [Noir Language Reference](https://noir-lang.org/docs)
- [Aztec Sandbox Setup](https://docs.aztec.network/developers/sandbox)
- [Ignition Chain](https://aztec.network/) — Mainnet information

### Development Tools
- **Nargo** — CLI for compiling, proving, and verifying Noir programs
- **NoirJS** — Browser/mobile library for client-side proof generation
- **Aztec Sandbox** — Local development environment

### q\<T\> Implementation
- `Quantum_of_Trust_Equations_in_Noir.md` — Complete Noir implementation (3,000+ lines)
- `QuantumOfTrust_v10.md` — Framework whitepaper

### Background Reading
- [ZK-SNARKs Explained](https://z.cash/technology/zksnarks/)
- [Aztec's Private Execution Model](https://docs.aztec.network/aztec/concepts/state_model)
- [UTXO vs Account Models](https://docs.aztec.network/aztec/concepts/accounts)
- [ACIR Specification](https://noir-lang.org/docs/explainers/acir)

---

*This document represents our current assessment as of the implementation date. Blockchain technology evolves rapidly; we will revisit this analysis as the ecosystem matures.*
