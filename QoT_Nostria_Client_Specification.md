# QoT Nostria Client Specification

*Technical specification for integrating Quantum of Trust into the Nostria Nostr client*

**Version:** 1.0  
**Date:** January 1, 2026  
**Status:** Draft

---

## 1. Overview

### 1.1 Purpose

This specification defines the technical architecture for adding QoT Professional Network functionality to Nostria, an Angular-based Nostr client. The integration enables pseudonymous professional reputation tracking through verified contract outcomes.

### 1.2 Scope

This specification covers:
- Client-side service architecture
- Angular component structure
- Nostr event schemas for discovery
- Aztec blockchain integration
- ZK circuit integration for client-side computation
- Performance and privacy requirements

This specification does **not** cover:
- Aztec smart contract internals (see QoT_Aztec_Contract_Layer_Specification.md)
- Relay-side ZK verification (see ZK_Relay_Integration_Spec.md)
- UX flows and wireframes (see QoT_Professional_Network_UX_Analysis.md)

### 1.3 Architectural Principles

| Principle | Description |
|-----------|-------------|
| **Aztec is source of truth** | Trust state, contracts, and escrow live on the Aztec blockchain |
| **Nostr is discovery layer** | Listings, proofs, and negotiation flow through Nostr |
| **Separate pages** | QoT pages are architecturally separate from existing Nostria features |
| **Shared identity** | Same npub/keypair maps to Aztec Avatar address |
| **Client-side proving** | ZK proofs generated locally using compiled Noir circuits |

### 1.4 Key Dependencies

| Package | Purpose | Size |
|---------|---------|------|
| `@aztec/aztec.js` | Blockchain RPC client | ~200KB |
| `@qot/circuits-wasm` | Trust computation (compiled from Noir) | ~500KB |
| `@aztec/bb.js` | Barretenberg proving backend | ~2MB WASM |
| `@noir-lang/noir_js` | Noir circuit execution | ~300KB |
| `nostr-tools` | Nostr protocol (existing) | — |

---

## 2. Architecture

### 2.1 System Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                        Aztec Blockchain                             │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────────────┐  │
│  │ QoTRegistry │  │ QoTEscrow   │  │ QoTAvatar                   │  │
│  │ - Projects  │  │ - Stakes    │  │ - Trust state               │  │
│  │ - Phases    │  │ - Lifecycle │  │ - Contract history          │  │
│  │ - Listings  │  │ - Outcomes  │  │ - Eligibility verification  │  │
│  └─────────────┘  └─────────────┘  └─────────────────────────────┘  │
│               SOURCE OF TRUTH FOR ALL TRUST DATA                    │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│                      Nostria Client                                 │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────────────┐  │
│  │ QotService  │  │ ZkProving   │  │ AztecService                │  │
│  │ (coord)     │──│ Service     │──│ (blockchain RPC)            │  │
│  └─────────────┘  └─────────────┘  └─────────────────────────────┘  │
│                              │                                      │
│                              ▼                                      │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                   qot_circuits.nr (WASM)                    │   │
│  │  - compute_trust_value()    - prove_eligibility()           │   │
│  │  - check_sybil_compliance() - compute_weight()              │   │
│  └─────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│                         Nostr Network                               │
│  - Discovery (listings, profiles)                                   │
│  - Proof sharing (ZK eligibility proofs)                            │
│  - Contract negotiation (NIP-17 encrypted DMs)                      │
│  - NOT the source of truth (mirrors on-chain state)                 │
└─────────────────────────────────────────────────────────────────────┘
```

### 2.2 Data Location

| Data | Authoritative Source | Nostr Role |
|------|---------------------|------------|
| Trust scores | QoTAvatar contract | Cached/mirrored for discovery |
| Contract state | QoTEscrow contract | Notifications, negotiation |
| Listings | QoTRegistry contract | Discovery, search |
| ZK proofs | Generated client-side | Shared for efficient verification |
| Contract history | QoTAvatar contract | Not on Nostr (on-chain sufficient) |
| Stakes/escrow | QoTEscrow contract | Not on Nostr |
| Milestone state | QoTEscrow contract | Progress notifications |

### 2.3 Why Aztec

Aztec provides **perfect security via on-chain ZK proofs**. All state transitions are verified by zero-knowledge circuits, ensuring that trust computations are mathematically guaranteed correct—not merely consensus-agreed.

| Capability | Mechanism |
|------------|-----------|
| **ZK-verified computation** | All trust math proven correct on-chain via Noir circuits |
| **Private state** | Contract history encrypted; only proofs are public |
| **Verified outcomes** | Mutual sign-off on-chain; stakes at risk |
| **Enforced escrow** | Funds locked until outcome recorded |
| **Computed trust** | `QoTAvatar.compute_trust_value()` from verified history |
| **Sybil resistance** | Velocity/variance checks in contract logic |
| **Efficient proofs** | Client-generated proofs reference on-chain state as public inputs |

---

## 3. Service Layer

### 3.1 QotService

**Purpose:** Central coordinator for QoT operations, bridging Aztec blockchain (source of truth) and Nostr (discovery layer).

```typescript
@Injectable({ providedIn: 'root' })
export class QotService {
  // === Reactive State (Signals) ===
  myTrustScores: Signal<Map<string, TrustScore>>;
  myContractHistory: Signal<ContractHistoryEntry[]>;
  customerProfile: Signal<CustomerProfile | null>;
  
  // === Dependencies ===
  constructor(
    private aztec: AztecService,           // Blockchain reads/writes
    private circuits: QotCircuitsService,  // Local computation
    private nostr: NostrService,           // Discovery, proof sharing
    private cache: TrustCacheService       // Local caching
  );
  
  // === Trust Score Operations (reads from Aztec) ===
  fetchMyTrustScores(domain?: string): Promise<TrustScore[]>;
  fetchAvatarTrustScores(pubkey: string, domain?: string): Promise<TrustScore[]>;
  
  // === Contract Operations (writes to Aztec) ===
  createContract(params: ContractParams): Promise<Contract>;
  acceptListing(listingId: string, stake: number, difficulty: number): Promise<void>;
  recordOutcome(contractId: string, outcome: number): Promise<void>;
  
  // === Milestone Operations ===
  getMilestoneState(contractId: string, milestoneId: number): Promise<MilestoneState>;
  submitMilestoneReview(contractId: string, milestoneId: number, review: MilestoneReview): Promise<void>;
  disputeTask(contractId: string, milestoneId: number, taskId: string, evidence: Evidence): Promise<void>;
  
  // === ZK Proof Operations ===
  generateEligibilityProof(domain: string, threshold: number): Promise<ProofResult>;
  verifyProof(proofEvent: NostrEvent): Promise<boolean>;
  
  // === Nostr Event Publishing ===
  publishProofToNostr(proof: Uint8Array, domain: string, threshold: number): Promise<NostrEvent>;
}
```

**Event Handling:** Subscribe to QoT events (kind 30078) with `qt-type` tag filtering.

### 3.2 AztecService

**Purpose:** Aztec blockchain RPC client for contract interaction.

```typescript
@Injectable({ providedIn: 'root' })
export class AztecService {
  // === Connection State ===
  rpcUrl: Signal<string>;
  connected: Signal<boolean>;
  currentAvatar: Signal<AztecAddress | null>;
  
  // === QoTAvatar Reads ===
  getTrustScore(avatar: AztecAddress, skillType: Field): Promise<Signed>;
  getContractHistory(avatar: AztecAddress): Promise<ContractHistoryEntry[]>;
  getAvatarState(avatar: AztecAddress): Promise<AvatarState>;
  
  // === QoTRegistry Reads ===
  getListing(listingId: Field): Promise<ContractListing>;
  getOpenListings(skillType?: Field): Promise<ContractListing[]>;
  getProject(projectId: Field): Promise<Project>;
  getPhase(phaseId: Field): Promise<Phase>;
  getMilestone(milestoneId: Field): Promise<Milestone>;
  
  // === QoTEscrow Reads ===
  getContractState(contractId: Field): Promise<ActiveContract>;
  getEscrowBalance(contractId: Field): Promise<EscrowEntry>;
  getMilestoneState(contractId: Field, milestoneId: Field): Promise<MilestoneState>;
  getTaskState(taskId: Field): Promise<TaskState>;
  
  // === Write Operations (require signing) ===
  createListing(params: ListingParams): Promise<Field>;
  acceptListing(listingId: Field, stake: Field, difficulty: Field): Promise<Field>;
  recordOutcome(contractId: Field, outcome: Field): Promise<void>;
  initiateCancel(contractId: Field): Promise<void>;
  
  // === Milestone Operations ===
  acceptMilestone(contractId: Field, milestoneId: Field): Promise<void>;
  disputeTask(contractId: Field, taskId: Field, evidenceHash: Field): Promise<void>;
  
  // === ZK Proof Submission ===
  submitEligibilityProof(proof: Uint8Array, publicInputs: Field[]): Promise<void>;
}
```

### 3.3 QotCircuitsService

**Purpose:** Wrapper for compiled Noir circuits (WASM). Provides client-side trust computation and ZK proof generation.

```typescript
@Injectable({ providedIn: 'root' })
export class QotCircuitsService {
  private circuits: CompiledCircuits | null;
  
  // === Lazy WASM Loading ===
  loadCircuits(): Promise<void>;
  
  // === Trust Computation (mirrors on-chain logic) ===
  computeTrustValue(
    history: ContractHistoryEntry[],
    skillType: Field,
    currentBlock: Field
  ): Signed;
  
  computeWeight(
    stake: Field,
    difficulty: Field,
    counterpartyTrust: Signed,
    lambda: Field
  ): Field;
  
  // === Eligibility Checks ===
  verifyEligibility(
    trust: Signed,
    threshold: Field,
    history: ContractHistoryEntry[],
    skillType: Field,
    sybilParams: SybilParameters,
    currentBlock: Field
  ): boolean;
  
  // === ZK Proof Generation ===
  proveEligibility(
    privateInputs: {
      history: ContractHistoryEntry[];
      trust: Signed;
    },
    publicInputs: {
      skillType: Field;
      threshold: Field;
      sybilParams: SybilParameters;
    }
  ): Promise<Uint8Array>;
  
  // === Sybil Checks ===
  checkSybilCompliance(
    history: ContractHistoryEntry[],
    skillType: Field,
    params: SybilParameters,
    currentBlock: Field
  ): boolean;
}
```

### 3.4 ZkProvingService

**Purpose:** Generic Web Worker wrapper for ZK proof generation. QotCircuitsService delegates proving operations here.

```typescript
@Injectable({ providedIn: 'root' })
export class ZkProvingService {
  // === State ===
  isProving: Signal<boolean>;
  
  // === Proof Operations ===
  generateProof(
    circuitWasm: Uint8Array,
    witnessData: Record<string, unknown>
  ): Promise<Uint8Array>;
  
  verifyProof(
    verificationKey: Uint8Array,
    proof: Uint8Array,
    publicInputs: Record<string, unknown>
  ): Promise<boolean>;
}
```

**Implementation Notes:**
- Uses Web Workers for non-blocking computation (~2-5 seconds)
- Worker created lazily on first proof request
- WASM cached in Service Worker after first load

### 3.5 TrustCacheService

**Purpose:** IndexedDB caching for trust scores with signal updates.

```typescript
@Injectable({ providedIn: 'root' })
export class TrustCacheService {
  // === State ===
  trustScoreCache: Signal<Map<string, TrustScore[]>>;
  
  // === Cache Operations ===
  cacheScores(pubkey: string, scores: TrustScore[]): Promise<void>;
  getCachedScores(pubkey: string): Promise<TrustScore[] | null>;
  
  // === Two-Phase Loading (matches Nostria's feed caching) ===
  loadWithFallback(pubkey: string): Promise<TrustScore[]>;
  
  // === Cache Invalidation ===
  invalidateForPubkey(pubkey: string): Promise<void>;
  invalidateAll(): Promise<void>;
}
```

### 3.6 ContractStateService

**Purpose:** Local cache of on-chain contract state. Provides signal-based reactivity for UI updates.

```typescript
@Injectable({ providedIn: 'root' })
export class ContractStateService {
  // === State ===
  activeContracts: Signal<Contract[]>;
  pendingSignOffs: Signal<PendingSignOff[]>;
  pendingMilestoneReviews: Signal<PendingMilestoneReview[]>;
  
  // === Sync from Aztec ===
  refreshContracts(): Promise<void>;
  refreshContract(contractId: string): Promise<void>;
  
  // === Computed State ===
  getContractsByStatus(status: ContractStatus): Contract[];
  getMilestonesAwaitingReview(): Milestone[];
  getDisputedTasks(): Task[];
  
  // === Subscriptions ===
  subscribeToContractUpdates(contractId: string): Observable<ContractUpdate>;
  subscribeToMilestoneDeadlines(): Observable<MilestoneDeadlineAlert>;
}
```

---

## 4. Component Layer

### 4.1 Route Structure

All QoT routes live under `/qot/*` namespace:

```typescript
// src/app/app.routes.ts
{
  path: 'qot',
  children: [
    { path: '', redirectTo: 'dashboard', pathMatch: 'full' },
    { path: 'dashboard', loadComponent: () => import('./pages/qot/dashboard/dashboard.component') },
    { path: 'marketplace', loadComponent: () => import('./pages/qot/marketplace/marketplace.component') },
    { path: 'contract/:id', loadComponent: () => import('./pages/qot/contract/contract.component') },
    { path: 'contract/:id/milestone/:milestoneId', loadComponent: () => import('./pages/qot/milestone-review/milestone-review.component') },
    { path: 'prove/:domain', loadComponent: () => import('./pages/qot/prove/prove.component') },
    { path: 'avatar/:pubkey', loadComponent: () => import('./pages/qot/avatar/avatar.component') },
    { path: 'customer/:pubkey', loadComponent: () => import('./pages/qot/customer/customer.component') },
    { path: 'dao/:id', loadComponent: () => import('./pages/qot/dao/dao.component') },
    { path: 'calibration', loadComponent: () => import('./pages/qot/calibration/calibration.component') },
    { path: 'dispute/:taskId', loadComponent: () => import('./pages/qot/dispute/dispute.component') }
  ]
}
```

### 4.2 Component Hierarchy

```
src/app/pages/qot/
├── dashboard/
│   ├── dashboard.component.ts          # Main trust dashboard
│   ├── trust-score-card/               # Per-domain trust display
│   └── contract-history/               # Private contract list
├── marketplace/
│   ├── marketplace.component.ts        # Contract browsing
│   ├── contract-card/                  # Contract preview card
│   └── filters/                        # Eligibility filters
├── contract/
│   ├── contract.component.ts           # Contract detail view
│   ├── phase-timeline/                 # Phase visualization
│   ├── milestone-list/                 # Milestone payment gates
│   ├── task-list/                      # Task breakdown within milestone
│   ├── sign-off-dialog/                # Mutual sign-off
│   └── amendment-dialog/               # Mid-contract changes
├── milestone-review/
│   ├── milestone-review.component.ts   # Customer milestone review
│   ├── task-outcome-card/              # Per-task outcome display
│   └── dispute-dialog/                 # Task dispute initiation
├── prove/
│   └── prove.component.ts              # ZK proof generation UI
├── avatar/
│   ├── avatar.component.ts             # Public avatar view
│   └── published-work/                 # Mutual consent contracts
├── customer/
│   └── customer.component.ts           # Customer metrics view
├── dao/
│   └── dao.component.ts                # DAO aggregate view
├── calibration/
│   └── calibration.component.ts        # Provider accuracy tracking
└── dispute/
    ├── dispute.component.ts            # Dispute management
    ├── evidence-upload/                # Evidence submission
    └── arbitrator-selection/           # Arbitrator proposal
```

### 4.3 Key Page Specifications

#### Trust Dashboard

| Element | Data Source | Update Trigger |
|---------|-------------|----------------|
| Trust scores by domain | `QotService.myTrustScores` | Aztec block sync |
| Contract history | `QotService.myContractHistory` | Aztec block sync |
| Customer profile | `QotService.customerProfile` | Aztec block sync |
| Pending milestone reviews | `ContractStateService.pendingMilestoneReviews` | Milestone deadline approach |

#### Contract Detail

| Element | Data Source | Update Trigger |
|---------|-------------|----------------|
| Contract terms | `AztecService.getContractState()` | On navigation |
| Phase timeline | `AztecService.getPhase()` | Phase completion |
| Milestone list | `AztecService.getMilestone()` | Milestone state change |
| Task breakdown | `AztecService.getTaskState()` | Task completion |

#### Milestone Review

| Element | Data Source | Actions |
|---------|-------------|---------|
| Tasks in milestone | `AztecService.getMilestoneState()` | — |
| Task outcomes | Computed from task state | Accept / Dispute |
| Deadline countdown | Milestone deadline from Aztec | — |
| Dispute form | Local state | Submit evidence |

---

## 5. Nostr Event Schema

> **Note:** These events are for **discovery and proof sharing**, not source of truth. Authoritative state lives on Aztec.

### 5.1 Event Types

All QoT events use kind 30078 with d-tag namespacing:

| d-tag Namespace | Purpose | Content |
|-----------------|---------|---------|
| `qt:score:<domain>` | Trust score | NIP-44 encrypted score data |
| `qt:proof:eligibility:<domain>` | ZK eligibility proof | Base64 Noir proof |
| `qt:contract:<id>` | Contract definition | Encrypted contract terms |
| `qt:phase:<contract>:<phase>` | Phase outcome | Encrypted outcome data |
| `qt:milestone:<contract>:<milestone>` | Milestone status | Milestone progress |
| `qt:customer:<pubkey>` | Customer profile | Customer metrics |
| `qt:circuit:<id>` | Circuit metadata | Public circuit info |

### 5.2 Trust Score Event

```json
{
  "kind": 30078,
  "pubkey": "<avatar-pubkey>",
  "created_at": 1704067200,
  "tags": [
    ["d", "qt:score:software_engineering"],
    ["qt-type", "trust-score"],
    ["qt-domain", "software_engineering"],
    ["qt-count", "47"],
    ["qt-version", "1.0.0"]
  ],
  "content": "<NIP-44 encrypted: { value: 78.3, history_depth: 47, updated: 1704067200 }>",
  "sig": "<schnorr-signature>"
}
```

### 5.3 Contract Listing Event

```json
{
  "kind": 30078,
  "pubkey": "<customer-pubkey>",
  "created_at": 1704067200,
  "tags": [
    ["d", "qt:contract:<aztec-contract-id>"],
    ["qt-type", "contract"],
    ["qt-domain", "software_engineering"],
    ["qt-provider", "<provider-pubkey>"],
    ["qt-status", "active"],
    ["qt-escrow", "1000000"],
    ["qt-aztec-ref", "<aztec-contract-address>"]
  ],
  "content": "<NIP-44 encrypted contract terms>",
  "sig": "<schnorr-signature>"
}
```

### 5.4 ZK Eligibility Proof Event

```json
{
  "kind": 30078,
  "pubkey": "<avatar-pubkey>",
  "created_at": 1704067200,
  "tags": [
    ["d", "qt:proof:eligibility:software_engineering"],
    ["qt-type", "zk-proof"],
    ["qt-circuit", "eligibility-v1"],
    ["qt-domain", "software_engineering"],
    ["qt-threshold", "75"],
    ["qt-vk", "<blossom-sha256-hash>"],
    ["nonce", "8192", "16"]
  ],
  "content": "<base64-encoded-noir-proof>",
  "sig": "<schnorr-signature>"
}
```

### 5.5 Milestone Status Event

```json
{
  "kind": 30078,
  "pubkey": "<contract-customer-pubkey>",
  "created_at": 1704067200,
  "tags": [
    ["d", "qt:milestone:<contract-id>:<milestone-id>"],
    ["qt-type", "milestone-status"],
    ["qt-contract", "<aztec-contract-id>"],
    ["qt-milestone", "2"],
    ["qt-status", "pending_review"],
    ["qt-deadline", "1704153600"],
    ["qt-task-count", "4"],
    ["qt-aztec-ref", "<aztec-milestone-address>"]
  ],
  "content": "<NIP-44 encrypted: { tasks: [...], deadline: timestamp }>",
  "sig": "<schnorr-signature>"
}
```

---

## 6. Data Flows

### 6.1 Reading Trust Scores

```
User opens Trust Dashboard
  → QotService.fetchMyTrustScores()
    → AztecService.getAvatarState(myAvatar)
      → RPC call to Aztec node
      → Returns trust values from QoTAvatar contract
    → Update myTrustScores signal
    → UI reactively updates
```

### 6.2 Generating Eligibility Proof

```
User clicks "Prove Eligibility ≥75"
  → QotService.generateEligibilityProof("software_engineering", 75)
    → QotCircuitsService.loadCircuits() [lazy WASM load]
    → AztecService.getContractHistory(myAvatar) [get private witness data]
    → QotCircuitsService.proveEligibility() [2-5 second computation in Web Worker]
    → AztecService.submitEligibilityProof() [optional: cache on-chain]
    → NostrService.publishProofEvent() [share for discovery]
  → UI shows shareable proof link
```

### 6.3 Accepting a Contract

```
Provider clicks "Accept Contract"
  → QotService.acceptListing(listingId, stake, difficulty)
    → AztecService.acceptListing() 
      → Transaction to QoTRegistry.accept_listing()
      → QoTEscrow.lock_stake() called internally
      → QoTAvatar.verify_eligibility() called internally
    → On success: ContractStateService.refreshContract()
    → NostrService.sendContractNotification() [optional DM to customer]
```

### 6.4 Milestone Review Flow

```
Customer opens Milestone Review page
  → ContractStateService.getMilestoneState(contractId, milestoneId)
    → AztecService.getMilestoneState() [from QoTEscrow]
    → Returns tasks, their states, deadline
  → UI shows task outcomes with Accept/Dispute options

Customer accepts all tasks:
  → QotService.submitMilestoneReview(contractId, milestoneId, { acceptAll: true })
    → AztecService.acceptMilestone()
      → QoTEscrow.accept_milestone()
      → Payment released to providers
      → Trust updated for each task
    → ContractStateService.refreshContract()

Customer disputes specific task:
  → QotService.disputeTask(contractId, milestoneId, taskId, evidence)
    → AztecService.disputeTask()
      → QoTEscrow.initiate_dispute()
      → Non-disputed tasks paid immediately
      → Disputed task enters arbitration
    → Navigate to dispute management page
```

### 6.5 Dispute Resolution Flow

```
Dispute initiated:
  → Customer uploads evidence hash to Aztec
  → Provider has 72 hours to respond with counter-evidence
  → Both parties propose arbitrators

Arbitrator selected:
  → First mutually acceptable arbitrator assigned
  → Arbitrator reviews evidence
  → Arbitrator sets payout percentage (0-100%)

Appeal (if either party disagrees):
  → 15% fee required
  → 3-arbitrator panel reviews
  → Majority decision is final

Deadlock (no mutual arbitrator):
  → Stakes returned to both parties
  → Task outcome = 0 (neutral)
```

---

## 7. Type Definitions

### 7.1 Core Types

```typescript
interface TrustScore {
  value: number;           // Scaled trust value
  historyDepth: number;    // Number of contracts
  lastUpdated: number;     // Block number
}

interface ContractHistoryEntry {
  contractId: Field;
  skillType: Field;
  stake: Field;
  difficulty: Field;
  outcome: Signed;
  completedAt: Field;
  counterparty: AztecAddress;
  weight: Field;
}

interface CustomerProfile {
  commitmentRate: number;       // completed / initiated
  fundingReliability: number;   // on-time / total
  verificationIntegrity: number; // rating variance
  scopeStability: number;       // tasks as planned / total
  timelineRealism: number;      // planned vs actual accuracy
}
```

### 7.2 Contract Types

```typescript
interface Contract {
  id: Field;
  customer: AztecAddress;
  provider: AztecAddress;
  skillType: Field;
  stake: Field;
  difficulty: Field;
  status: ContractStatus;
  phases: Phase[];
}

interface Phase {
  id: Field;
  phaseType: PhaseType;        // Specification | Planning | Implementation
  provider: AztecAddress;
  stake: Field;
  outcome: Signed | null;
  milestones: Milestone[];     // Only for Implementation phase
}

interface Milestone {
  id: Field;
  parentPhaseId: Field;
  tasks: Task[];
  deadline: number;            // Computed: max(task deadlines)
  stake: number;               // Computed: sum(task stakes)
  status: MilestoneStatus;     // Pending | AwaitingReview | Accepted | Disputed
}

interface Task {
  id: Field;
  parentMilestoneId: Field;
  provider: AztecAddress;
  weight: number;
  difficulty: number;
  deadline: number;            // Work duration (planning data)
  outcome: Signed | null;
  status: TaskStatus;
}

enum ContractStatus {
  Created = 0,
  Active = 1,
  AwaitingReview = 2,
  Completed = 3,
  Disputed = 4,
  Cancelled = 5
}

enum MilestoneStatus {
  Pending = 0,
  InProgress = 1,
  AwaitingReview = 2,
  Accepted = 3,
  PartiallyDisputed = 4
}
```

### 7.3 Proof Types

```typescript
interface ProofResult {
  proof: Uint8Array;
  publicInputs: Field[];
  nevent: string;              // Nostr event ID for sharing
}

interface EligibilityProof {
  domain: string;
  threshold: number;
  proof: Uint8Array;
  verificationKey: string;     // Blossom hash
  generatedAt: number;
}

interface SybilParameters {
  minCounterparties: Field;
  maxVelocity: Field;
  minOutcomeVariance: Field;
  maxSameCounterpartyRatio: Field;
}
```

---

## 8. Performance Requirements

### 8.1 Targets

| Metric | Target | Measurement |
|--------|--------|-------------|
| Proof generation | < 5 seconds | Browser performance API |
| Trust score load (cached) | < 500ms | Time to first paint |
| Trust score load (uncached) | < 2 seconds | Aztec RPC round-trip |
| Contract state transition | < 2 seconds | Aztec transaction confirmation |
| Bundle size increase (non-ZK pages) | < 500KB | Build output |
| ZK WASM first load | < 3 seconds | Service worker metrics |
| ZK WASM cached load | < 500ms | Service worker metrics |

### 8.2 Optimization Strategies

**Lazy Loading:**
- ZK dependencies loaded only when user navigates to `/qot/prove/*`
- Code-split ZK-related components
- Service worker caches WASM files

```typescript
// Lazy load bb.js only when needed
async loadNoirBackend() {
  if (!this.backend) {
    const { UltraHonkBackend } = await import('@aztec/bb.js');
    this.backend = UltraHonkBackend;
  }
  return this.backend;
}
```

**Web Worker Strategy:**
- All ZK proving runs in Web Worker to avoid UI blocking
- Worker created lazily on first proof request
- Progress updates streamed to main thread

**Caching Strategy:**
- Trust scores cached in IndexedDB with TTL
- Two-phase loading: show cached data immediately, refresh in background
- Contract state cached with invalidation on block updates

---

## 9. Privacy Requirements

### 9.1 Privacy Mitigations

Per Nostr_QoT_Gap_Analysis.md:

| Threat | Mitigation |
|--------|------------|
| Timing correlation | Random delay (0-24h) before proof publication |
| Exact trust inference | Offer preset thresholds (25, 50, 75, 100) only |
| Activity pattern analysis | Option to batch-publish proofs |
| Contract detail exposure | NIP-44 encryption for all content fields |

### 9.2 Implementation

```typescript
// Timing obfuscation
async publishWithDelay(event: NostrEvent, maxDelayHours = 24) {
  const delay = Math.random() * maxDelayHours * 60 * 60 * 1000;
  setTimeout(() => this.publish(event), delay);
}

// Coarse thresholds only
const ALLOWED_THRESHOLDS = [25, 50, 75, 100];

validateThreshold(threshold: number): number {
  const nearest = ALLOWED_THRESHOLDS.reduce((prev, curr) => 
    Math.abs(curr - threshold) < Math.abs(prev - threshold) ? curr : prev
  );
  return nearest;
}
```

### 9.3 Data Visibility

| Data | Visibility | Notes |
|------|------------|-------|
| Trust scores (aggregate) | Public via Nostr | Domain + count only |
| Exact trust value | Private | Revealed only via ZK threshold proofs |
| Contract history | Private | On-chain but encrypted |
| Contract details | Mutual consent | Both parties must agree to publish |
| Counterparty identity | Private | Never revealed without consent |
| Milestone/task breakdown | Private | Internal to contract parties |

---

## 10. Integration Points

### 10.1 Navigation

Add QoT to existing Nostria sidenav:

```typescript
// In app.ts navLinks array
{ 
  path: '/qot', 
  icon: 'verified_user', 
  title: $localize`Professional Trust`,
  tooltip: $localize`QoT Trust Network`
}
```

### 10.2 Profile Header (Optional)

Non-invasive trust badge for profiles:

```typescript
// In profile-header.component.ts
hasQotProfile = computed(() => {
  return this.qotService.hasProfile(this.user().pubkey);
});

// Template
@if (hasQotProfile()) {
  <button mat-icon-button 
    [routerLink]="['/qot/avatar', user().pubkey]"
    matTooltip="View Trust Profile">
    <mat-icon>verified_user</mat-icon>
  </button>
}
```

### 10.3 Direct Message Integration

Contract negotiation via existing NIP-17 DM system:

```typescript
async initiateContractNegotiation(counterparty: string, proposal: ContractProposal) {
  const content = JSON.stringify({
    type: 'qot:contract-proposal',
    proposal
  });
  
  await this.nostr.sendEncryptedDM(counterparty, content);
}
```

### 10.4 Relay Strategy

QoT events published to both user relays and QoT verifier relays:

```typescript
async publishQotEvent(event: NostrEvent) {
  const userRelays = this.accountRelayService.getWriteRelays();
  const qotRelays = this.config.get('qot.verifierRelays');
  
  await Promise.all([
    ...userRelays.map(r => this.relayPool.publish(r, event)),
    ...qotRelays.map(r => this.relayPool.publish(r, event))
  ]);
}
```

---

## 11. Open Questions

| Question | Options | Status |
|----------|---------|--------|
| Aztec RPC endpoint | Self-hosted node vs. public endpoint | TBD |
| Gas/fee payment | User pays vs. sponsored transactions | TBD |
| Proof caching | On-chain (verified_eligibility mapping) vs. Nostr-only | TBD |
| History sync | Full history from Aztec vs. incremental updates | TBD |
| Testnet for development | Aztec sandbox vs. public testnet | TBD |

---

## 12. Related Documents

- **QoT_Nostria_Component_Templates.md** — Angular templates and implementation details
- **QoT_Nostria_Implementation_Plan.md** — Implementation timeline and phases
- **QoT_Aztec_Contract_Layer_Specification.md** — Smart contract architecture
- **QoT_Professional_Network_UX_Analysis.md** — UX flows and wireframes
- **ZK_Relay_Integration_Spec.md** — Relay-side ZK verification
- **Nostr_QoT_Gap_Analysis.md** — Technical gap resolutions
- **ADR_Milestone_Payment_Gates.md** — Milestone-based payment model
- **ADR_Dispute_Resolution.md** — Deadline-based dispute resolution
- **ADR_Subcontract_Architecture.md** — Contract decomposition patterns

---

*This specification defines the technical architecture for QoT integration with Nostria. Implementation details and timeline are documented separately in QoT_Nostria_Implementation_Plan.md.*
