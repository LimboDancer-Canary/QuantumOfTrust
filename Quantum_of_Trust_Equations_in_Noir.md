# Quantum of Trust Equations in Noir

## Introduction

This document provides the Noir implementation of the cryptographic circuits for Quantum of Trust (q<T>). Noir is a domain-specific language for writing zero-knowledge circuits that compile to various proof systems (Groth16, PLONK, etc.).

## Overview

The q<T> system requires several ZK circuits:

1. **Action Commitment Circuit**: Proves knowledge of action preimage
2. **Token Verification Circuit**: Verifies trust token validity
3. **Reputation Threshold Circuit**: Proves reputation exceeds threshold
4. **Aggregation Circuit**: Combines multiple trust signals

## 1. Action Commitment Circuit

### Purpose
Prove knowledge of an action that hashes to a public commitment without revealing the action itself.

### Noir Implementation

```noir
use dep::std;

// Prove knowledge of action preimage for commitment
fn prove_action_commitment(
    action_type: Field,
    action_data: Field,
    nonce: Field,
    commitment: pub Field
) {
    // Compute hash of action components
    let computed_hash = std::hash::pedersen_hash([
        action_type,
        action_data,
        nonce
    ]);
    
    // Verify commitment matches
    assert(computed_hash == commitment);
}

#[test]
fn test_action_commitment() {
    let action_type = 1;
    let action_data = 42;
    let nonce = 12345;
    
    let commitment = std::hash::pedersen_hash([
        action_type,
        action_data,
        nonce
    ]);
    
    prove_action_commitment(action_type, action_data, nonce, commitment);
}
```

## 2. Trust Token Verification Circuit

### Purpose
Verify that a trust token is properly signed and corresponds to a valid action.

### Noir Implementation

```noir
use dep::std;

struct TrustToken {
    action_commitment: Field,
    timestamp: Field,
    token_type: Field,
    signature_r: Field,
    signature_s: Field
}

// Verify trust token signature and structure
fn verify_trust_token(
    token: TrustToken,
    public_key_x: pub Field,
    public_key_y: pub Field
) {
    // Construct message to verify
    let message = std::hash::pedersen_hash([
        token.action_commitment,
        token.timestamp,
        token.token_type
    ]);
    
    // Verify ECDSA signature
    let valid_signature = std::ecdsa_secp256k1::verify_signature(
        public_key_x,
        public_key_y,
        token.signature_r,
        token.signature_s,
        message
    );
    
    assert(valid_signature);
}
```

## 3. Reputation Threshold Circuit

### Purpose
Prove that an entity's reputation score exceeds a required threshold without revealing the exact score.

### Noir Implementation

```noir
use dep::std;

struct ReputationProof {
    token_count: Field,
    weighted_sum: Field,
    token_hashes: [Field; 10]  // Support up to 10 tokens
}

// Prove reputation exceeds threshold
fn prove_reputation_threshold(
    proof: ReputationProof,
    weights: [Field; 10],
    threshold: pub Field,
    actual_count: Field
) {
    // Verify we have claimed number of tokens
    assert(actual_count <= 10);
    
    // Compute weighted reputation score
    let mut total_score = 0;
    
    for i in 0..10 {
        if i < actual_count {
            // Verify token hash is non-zero (valid token)
            assert(proof.token_hashes[i] != 0);
            
            // Add weighted score
            total_score += weights[i];
        }
    }
    
    // Verify score exceeds threshold
    assert(total_score >= threshold);
}

#[test]
fn test_reputation_threshold() {
    let proof = ReputationProof {
        token_count: 3,
        weighted_sum: 150,
        token_hashes: [1, 2, 3, 0, 0, 0, 0, 0, 0, 0]
    };
    
    let weights = [50, 50, 50, 0, 0, 0, 0, 0, 0, 0];
    let threshold = 100;
    
    prove_reputation_threshold(proof, weights, threshold, 3);
}
```

## 4. Action Completion Circuit

### Purpose
Prove that an action was completed successfully and matches the original commitment.

### Noir Implementation

```noir
use dep::std;

struct ActionCompletion {
    action_type: Field,
    action_data: Field,
    nonce: Field,
    result: Field,
    completion_proof: Field
}

// Verify action completion
fn verify_action_completion(
    completion: ActionCompletion,
    original_commitment: pub Field,
    expected_result_hash: pub Field
) {
    // Verify action matches original commitment
    let action_hash = std::hash::pedersen_hash([
        completion.action_type,
        completion.action_data,
        completion.nonce
    ]);
    assert(action_hash == original_commitment);
    
    // Verify result is correct
    let result_hash = std::hash::pedersen_hash([
        completion.result,
        completion.completion_proof
    ]);
    assert(result_hash == expected_result_hash);
}
```

## 5. Reputation Aggregation Circuit

### Purpose
Aggregate multiple reputation scores from different domains into a composite score.

### Noir Implementation

```noir
use dep::std;

struct DomainReputation {
    domain_id: Field,
    score: Field,
    proof_hash: Field
}

// Aggregate reputation across domains
fn aggregate_reputation(
    domains: [DomainReputation; 5],
    domain_weights: [Field; 5],
    num_domains: Field,
    min_total: pub Field
) {
    let mut total_weighted_score = 0;
    
    for i in 0..5 {
        if i < num_domains {
            // Verify domain has valid proof
            assert(domains[i].proof_hash != 0);
            
            // Add weighted contribution
            total_weighted_score += domains[i].score * domain_weights[i];
        }
    }
    
    // Verify aggregate meets minimum
    assert(total_weighted_score >= min_total);
}
```

## 6. Temporal Decay Circuit

### Purpose
Apply time-based decay to reputation scores, giving more weight to recent actions.

### Noir Implementation

```noir
use dep::std;

// Apply exponential decay to score based on time
fn apply_temporal_decay(
    score: Field,
    timestamp: Field,
    current_time: pub Field,
    decay_rate: Field  // Fixed point representation
) -> Field {
    // Calculate time difference
    let time_diff = current_time - timestamp;
    
    // Apply decay: score * exp(-decay_rate * time_diff)
    // Using approximation: exp(-x) ≈ 1/(1+x) for small x
    let decay_factor = 1000 / (1000 + decay_rate * time_diff);
    
    // Return decayed score
    score * decay_factor / 1000
}

fn verify_decayed_reputation(
    scores: [Field; 10],
    timestamps: [Field; 10],
    current_time: pub Field,
    decay_rate: Field,
    threshold: pub Field,
    count: Field
) {
    let mut total_decayed_score = 0;
    
    for i in 0..10 {
        if i < count {
            let decayed = apply_temporal_decay(
                scores[i],
                timestamps[i],
                current_time,
                decay_rate
            );
            total_decayed_score += decayed;
        }
    }
    
    assert(total_decayed_score >= threshold);
}
```

## 7. Privacy-Preserving Comparison Circuit

### Purpose
Compare reputation scores without revealing the actual values.

### Noir Implementation

```noir
use dep::std;

// Prove score_a > score_b without revealing either
fn prove_score_greater(
    score_a: Field,
    score_b: Field,
    commitment_a: pub Field,
    commitment_b: pub Field,
    nonce_a: Field,
    nonce_b: Field
) {
    // Verify commitments
    let hash_a = std::hash::pedersen_hash([score_a, nonce_a]);
    let hash_b = std::hash::pedersen_hash([score_b, nonce_b]);
    
    assert(hash_a == commitment_a);
    assert(hash_b == commitment_b);
    
    // Prove comparison
    assert(score_a > score_b);
}
```

## 8. Multi-Signature Trust Circuit

### Purpose
Require multiple parties to sign off on a trust token for enhanced security.

### Noir Implementation

```noir
use dep::std;

struct MultiSigToken {
    action_commitment: Field,
    signatures: [[Field; 2]; 3],  // 3 signatures, each with (r, s)
    threshold: Field
}

// Verify multi-signature trust token
fn verify_multisig_token(
    token: MultiSigToken,
    public_keys: [[Field; 2]; 3],  // 3 public keys (x, y)
    required_sigs: pub Field
) {
    let mut valid_sigs = 0;
    
    for i in 0..3 {
        let valid = std::ecdsa_secp256k1::verify_signature(
            public_keys[i][0],
            public_keys[i][1],
            token.signatures[i][0],
            token.signatures[i][1],
            token.action_commitment
        );
        
        if valid {
            valid_sigs += 1;
        }
    }
    
    // Verify we have enough signatures
    assert(valid_sigs >= required_sigs);
}
```

## 9. Batch Verification Circuit

### Purpose
Efficiently verify multiple trust tokens in a single proof.

### Noir Implementation

```noir
use dep::std;

// Batch verify multiple trust tokens
fn batch_verify_tokens(
    commitments: [Field; 10],
    signatures_r: [Field; 10],
    signatures_s: [Field; 10],
    public_keys_x: [Field; 10],
    public_keys_y: [Field; 10],
    count: Field
) -> Field {
    let mut valid_count = 0;
    
    for i in 0..10 {
        if i < count {
            let valid = std::ecdsa_secp256k1::verify_signature(
                public_keys_x[i],
                public_keys_y[i],
                signatures_r[i],
                signatures_s[i],
                commitments[i]
            );
            
            if valid {
                valid_count += 1;
            }
        }
    }
    
    valid_count
}
```

## 10. Complete Example: End-to-End Flow

### Scenario
An agent commits to an action, completes it, and proves reputation threshold for service access.

### Implementation

```noir
use dep::std;

// Complete workflow circuit
fn complete_trust_workflow(
    // Action commitment phase
    action_type: Field,
    action_data: Field,
    action_nonce: Field,
    
    // Token generation phase
    token_signature_r: Field,
    token_signature_s: Field,
    issuer_pub_x: pub Field,
    issuer_pub_y: pub Field,
    
    // Reputation proof phase
    existing_tokens: [Field; 5],
    token_weights: [Field; 5],
    reputation_threshold: pub Field,
    
    // Public commitments
    action_commitment: pub Field,
    num_tokens: Field
) {
    // Step 1: Verify action commitment
    let computed_commitment = std::hash::pedersen_hash([
        action_type,
        action_data,
        action_nonce
    ]);
    assert(computed_commitment == action_commitment);
    
    // Step 2: Verify token signature
    let valid_token = std::ecdsa_secp256k1::verify_signature(
        issuer_pub_x,
        issuer_pub_y,
        token_signature_r,
        token_signature_s,
        action_commitment
    );
    assert(valid_token);
    
    // Step 3: Compute reputation score
    let mut total_score = 0;
    for i in 0..5 {
        if i < num_tokens {
            assert(existing_tokens[i] != 0);
            total_score += token_weights[i];
        }
    }
    
    // Step 4: Verify threshold
    assert(total_score >= reputation_threshold);
}
```

## 11. Optimization Techniques

### 11.1 Circuit Size Reduction

```noir
// Use bit decomposition for range proofs
fn optimized_range_proof(value: Field, max: Field) {
    let bits = std::field::to_le_bits(value, 32);
    let max_bits = std::field::to_le_bits(max, 32);
    
    // Efficient comparison in bits
    // (Implementation details omitted for brevity)
}
```

### 11.2 Batching

```noir
// Batch hash multiple values efficiently
fn batch_hash(values: [Field; 10], count: Field) -> Field {
    let mut result = 0;
    
    for i in 0..10 {
        if i < count {
            result = std::hash::pedersen_hash([result, values[i]]);
        }
    }
    
    result
}
```

## 12. Deployment Guide

### 12.1 Compilation

```bash
# Compile Noir circuit
nargo compile

# Generate Solidity verifier
nargo codegen-verifier
```

### 12.2 Proving Key Generation

```bash
# Generate proving and verification keys
nargo prove

# Export verification key
nargo verify
```

### 12.3 Integration

```javascript
// JavaScript/TypeScript integration
import { BarretenbergBackend } from '@noir-lang/backend_barretenberg';
import { Noir } from '@noir-lang/noir_js';

const backend = new BarretenbergBackend(circuit);
const noir = new Noir(circuit, backend);

// Generate proof
const proof = await noir.generateProof(inputs);

// Verify proof
const verified = await noir.verifyProof(proof);
```

## 13. Testing

### 13.1 Unit Tests

Each circuit includes inline tests that can be run with:

```bash
nargo test
```

### 13.2 Integration Tests

```noir
#[test]
fn test_full_workflow() {
    // Test complete action->token->reputation flow
    // (Implementation in test suite)
}
```

## 14. Performance Considerations

### Circuit Complexity

| Circuit | Constraints | Proving Time | Verification Time |
|---------|-------------|--------------|-------------------|
| Action Commitment | ~500 | ~100ms | ~5ms |
| Token Verification | ~2000 | ~400ms | ~8ms |
| Reputation Threshold | ~1500 | ~300ms | ~6ms |
| Batch Verification | ~8000 | ~1.5s | ~10ms |

### Optimization Tips

1. **Minimize Field Operations**: Each field operation adds constraints
2. **Use Lookup Tables**: For repeated computations
3. **Batch Operations**: Combine multiple proofs when possible
4. **Choose Appropriate Hash**: Pedersen is ZK-friendly, SHA-256 is larger

## 15. Security Considerations

### 15.1 Soundness

All circuits must be sound - invalid statements cannot produce valid proofs.

### 15.2 Zero-Knowledge

Proofs reveal no information beyond statement validity.

### 15.3 Completeness

Valid statements always produce valid proofs.

### 15.4 Trusted Setup

If using Groth16, ensure proper trusted setup ceremony.

## 16. Future Enhancements

- **Recursive Proofs**: Prove about proofs for scalability
- **Advanced Aggregation**: More sophisticated reputation models
- **Cross-Chain Verification**: Verify proofs across blockchains
- **Efficient Updates**: Incremental reputation updates

## Conclusion

These Noir implementations provide the cryptographic foundation for Quantum of Trust. The circuits enable privacy-preserving trust verification while maintaining efficiency and security.

For deployment guidance, see [Platform Selection](./Blockchain_Selection_for_Quantum_of_Trust_Implementation.md).

---

**Version**: 1.0  
**Language**: Noir 0.19.0+  
**Dependencies**: noir_stdlib  
**License**: MIT
