# Quantum of Trust

**Trust Primitives for the Agent Economy**

---

> *The future of trust is not about knowing who someone is. It's about knowing what they can do.*

## Overview

Quantum of Trust (q\<T\>, pronounced "cute") is a mathematical framework for decoupling trust from identity. It addresses two converging crises: the fundamental brokenness of network privacy and the lack of accountability infrastructure for AI agents.

The framework enables:

- **Privacy-preserving reputation** — Build trust through verified action, not identity disclosure
- **AI agent accountability** — Same trust primitives for humans and AI-operated avatars
- **Composable trust networks** — DAOs as recursive trust structures
- **Cryptographic verification** — Zero-knowledge proofs for eligibility without revealing history

## The Core Insight

Traditional systems conflate identity with trust. Your credit score, your social media reputation, your professional credentials—all are tightly coupled to *who you are*. This coupling is the root cause of both privacy violations and the lack of AI accountability infrastructure.

q\<T\> inverts this architecture:

```
Traditional:  Identity → Trust → Participation
q<T>:         Avatar → Verified Action → Trust → Participation
```

The Avatar is the fundamental interface. Identity disclosure becomes optional information you attach, not a baseline requirement.

## Mathematical Foundation

A Quantum of Trust is defined recursively:

$$q\langle T \rangle ::= \text{Agent}(t, h_t) \mid \text{DAO}(\{q\langle T \rangle\})$$

Where an **Agent** has a skill type and history, and a **DAO** contains a set of q\<T\> units.

Trust value maps to all real numbers:

$$V_t: q\langle T \rangle \rightarrow \mathbb{R}$$

- $V_t = 0$ → Unknown, no track record
- $V_t > 0$ → Net positive history, trusted  
- $V_t < 0$ → Net negative history, actively distrusted

The eligibility primitive—the core operation:

$$\text{eligible}(a, c) \iff V_t(a) \geq \theta(c)$$

An Avatar proves they meet a threshold *without revealing their history, counterparties, or exact trust score*.

## Repository Contents

| Document | Description |
|----------|-------------|
| [**QuantumOfTrust_v10.md**](./QuantumOfTrust_v10.md) | The complete framework whitepaper. Covers the problem, the Avatar-first architecture, mathematical formalization, composable trust, and the human-AI bridge. |
| [**The_Quantum_of_Trust_Math_Equations_in_Plain_English.md**](./The_Quantum_of_Trust_Math_Equations_in_Plain_English.md) | Every equation explained in plain English with examples. |
| [**Quantum_of_Trust_Equations_in_Noir.md**](./Quantum_of_Trust_Equations_in_Noir.md) | Full implementation in Noir (Aztec's ZK circuit language). Covers signed arithmetic, eligibility proofs, DAO aggregation, and Sybil resistance. |
| [**Quantum_of_Trust_Equations_in_CSharp.md**](./Quantum_of_Trust_Equations_in_CSharp.md) | C# implementation for traditional systems. |
| [**Sybil_Resistance_Architecture.md**](./Sybil_Resistance_Architecture.md) | Detailed design rationale for defense-in-depth against Sybil attacks. |
| [**Blockchain_Selection_for_Quantum_of_Trust_Implementation.md**](./Blockchain_Selection_for_Quantum_of_Trust_Implementation.md) | Technical rationale for choosing Noir on Aztec. |

## Key Concepts

### Avatar-First Architecture

All network participation is Avatar-mediated. The network only "sees" Avatars—it has no concept of "human" or "AI". This architectural choice enables:

- **Privacy by default** — Identity disclosure is additive, not baseline
- **AI equivalence** — Human-operated and AI-operated avatars are structurally identical
- **Multiple personas** — One principal can operate multiple Avatars across contexts
- **Accountability** — Keys trace back to human principals without identity disclosure

### Skill-Scoped Trust

Trust quotients are independent per skill type:

```
Jane's Avatars:
├── Engineering Avatar: V_engineering = 85 cutes
└── Design Avatar:      V_design = -12 cutes
```

Success in one domain doesn't inflate standing in another. Failure in one doesn't contaminate success in another.

### Zero-Knowledge Eligibility

The core cryptographic primitive proves statements about trust without revealing underlying data:

```
Prover knows:     Full contract history, exact trust score
Prover proves:    "My trust in Engineering ≥ 50"
Verifier learns:  Avatar is eligible (boolean)
Verifier cannot:  See history size, counterparties, stakes, outcomes
```

### Composable DAOs

DAOs are themselves q\<T\> units, enabling recursive composition:

$$V_t(\text{DAO}(S)) = \Phi\left(\{V_t(q) : q \in S\}\right)$$

Where Φ is a configurable aggregation function (sum, average, minimum, maximum).

## Implementation Status

The Noir implementation provides production-ready circuits for:

- ✅ Signed arithmetic in finite fields
- ✅ Fixed-point arithmetic for weighted calculations
- ✅ Trust value computation from private history
- ✅ Eligibility threshold proofs
- ✅ DAO aggregation functions
- ✅ Sybil resistance checks (history size, depth, counterparty diversity)
- ✅ Contract validation to prevent proof forgery
- ✅ Comprehensive test suite

## Sybil Resistance (Active Development)

The framework implements defense-in-depth against Sybil attacks through four complementary mechanisms:

- **Economic escrow** — Contract stakes require locked funds, creating real cost for fake contracts
- **Counterparty trust weighting** — Contracts with low-trust counterparties contribute less to your score
- **Outcome variance requirements** — Suspiciously uniform histories are flagged as implausible
- **Temporal velocity limits** — Trust accumulation is rate-limited to prevent burst attacks

These mechanisms compose to make Sybil attacks economically irrational while preserving accessibility for legitimate newcomers. Parameter tuning and additional hardening are ongoing.

**Mathematical formulation:**

| Mechanism | Formula |
|-----------|---------|
| Counterparty Factor | $\gamma(c) = \sigma(V_t(\text{counterparty}) / \lambda)$ |
| Velocity Weight | $\nu(c) = 1 / (1 + k \cdot \max(0, \text{rank} - N))$ |
| Variance Check | $\text{plausible}(h) \iff \|h\| < N_{\min} \lor \text{var}(\text{outcomes}) \geq \varepsilon$ |

The enhanced trust calculation becomes:

$$V_t(\text{Agent}) = \sum_{c \in h_t} \omega(c) \cdot \text{outcome}(c) \cdot \gamma(c) \cdot \nu(c)$$

See [Sybil_Resistance_Architecture.md](./Sybil_Resistance_Architecture.md) for detailed design rationale and attack scenario analysis.

## Why Aztec/Noir?

q\<T\> requires privacy as the default execution model, not an optional layer. Aztec provides:

- **Private state by default** — Encrypted UTXO-like "notes" for contract histories
- **Client-side proving** — Users generate proofs locally; sequencers never see private data
- **Hybrid state model** — Private for histories, public for contract listings
- **Recursive proofs** — DAO trust aggregation without revealing member values

See [Blockchain_Selection_for_Quantum_of_Trust_Implementation.md](./Blockchain_Selection_for_Quantum_of_Trust_Implementation.md) for the full platform evaluation.

## Use Cases

### For Humans

- Maintain professional reputation without doxxing
- Build independent trust across multiple contexts
- Transfer earned reputation (Avatar trading)
- Participate in DAOs with privacy-preserving governance

### For AI Agents

- Objective, measurable reputation for agent services
- Accountability that traces to human principals
- Market infrastructure for agent capabilities
- Trust-gated access to high-stakes contracts

### For Organizations

- Form DAOs around verified capabilities
- Aggregate trust for collective contracting
- Governance weighted by demonstrated competence
- Sybil-resistant membership verification

## Getting Started

### Reading Order

1. Start with **QuantumOfTrust_v10.md** for the conceptual framework
2. Review **The_Quantum_of_Trust_Math_Equations_in_Plain_English.md** for accessible explanations
3. Study **Sybil_Resistance_Architecture.md** for security mechanisms
4. Review **Blockchain_Selection_for_Quantum_of_Trust_Implementation.md** for platform rationale
5. Study **Quantum_of_Trust_Equations_in_Noir.md** for implementation details

### For Developers

The Noir implementation requires:

- [Nargo](https://noir-lang.org/docs/getting_started/installation/) — Noir's CLI toolchain
- [Aztec Sandbox](https://docs.aztec.network/developers/sandbox) — Local development environment

```bash
# Compile Noir circuits
nargo compile

# Run tests
nargo test

# Generate proof
nargo prove

# Verify proof
nargo verify
```

## Contributing

This framework is under active development. We welcome:

- **Conceptual feedback** on the whitepaper
- **Implementation improvements** to the Noir circuits
- **Security analysis** of the cryptographic primitives
- **Use case exploration** for specific domains

Please open an issue to discuss proposed changes before submitting pull requests.

## License

[To be determined — suggest MIT or Apache 2.0 for maximum adoption]

## Acknowledgments

This work builds on foundational research in:

- Zero-knowledge proof systems
- Decentralized identity
- Reputation networks
- Cryptographic privacy

Special thanks to the Aztec team for Noir and the broader ZK research community.

---

*The author invites criticism, comments, and questions as we continue to develop these ideas together.*
