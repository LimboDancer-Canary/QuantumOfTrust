# ADR: Dispute Resolution via Arbitration Contracts

**Status:** Accepted  
**Date:** January 1, 2026  
**Deciders:** Dennis  
**Context:** QoT Professional Network design

---

## Decision

Disputes are resolved through **arbitration contracts** — standard QoT contracts where the deliverable is a binding outcome determination. Arbitrators are Avatars with `dispute_resolution` skill trust. No new primitives are required.

| Aspect | Decision |
|--------|----------|
| Task stake | Fixed amount, funded entirely by customer |
| Provider stake | None — provider stakes labor, not capital |
| Payment on acceptance | 100% of stake — no partial payment option |
| Review trigger | Milestone deadline (not individual task deadlines) |
| Customer action | Accept milestone, dispute specific task(s), or timeout |
| Timeout | Milestone deadline passes → all tasks paid via timeout |
| Dispute granularity | Task level (within milestone) |
| Arbitrator selection | Mutual agreement required |
| Tier 1 | Single arbitrator; fee = 10% of task stake |
| Tier 2 (appeal) | Panel of 3; fee = 15% of remaining pool |
| Fee commitment | When arbitrator(s) accept, not when dispute filed |
| Arbitrator ruling | payout_pct (0-100%): percentage of pool to provider |
| Trust outcome | Derived: (payout_pct - 50) / 50 |
| Evidence | Off-chain, anything goes |
| Deadlock | Timeout per existing contract rules; no fee lost |

---

## Context

The QoT framework requires a mechanism for resolving disputes when customer and provider disagree on task completion. Without dispute resolution:
- Customers have unilateral power over outcomes
- Providers have no recourse against unfair rejection
- System trust degrades as providers avoid risky contracts

The solution must preserve QoT principles:
- No central authority
- Trust earned through verified action
- Same primitives for all participants
- Economic incentives align behavior

---

## The Insight: Arbitration Is Just Another Skill

Dispute resolution is a professional skill like any other. Arbitrators:
- Build reputation through fair rulings
- Are rated by both disputants
- Accumulate `dispute_resolution` trust over time
- Can be filtered by eligibility threshold

This means:
- **No new primitives** — arbitration contracts use existing Contract structure
- **Self-regulating** — bad arbitrators lose trust, good ones gain it
- **Market-driven** — arbitrators compete on reputation
- **Recursive** — the same trust math applies

---

## Stake Model

### Why Provider Doesn't Stake Capital

The customer provides capital. The provider provides labor. Both have skin in the game:

| Party | What They Stake | What They Lose If Things Go Wrong |
|-------|-----------------|-----------------------------------|
| **Customer** | Task stake (capital) | Money (reduced/no refund) |
| **Provider** | Labor (time, effort) | Payment + trust + future opportunity |

Requiring providers to stake capital would:
- Create unnecessary barrier to entry
- Disadvantage skilled providers without capital
- Be redundant — labor commitment already aligns incentives

### Task Stake Mechanics

```
Task created:
  Customer locks: 1000 tokens (task stake)
  Provider locks: 0 tokens (stakes labor instead)

Task accepted (or auto-accepted):
  Provider receives: 1000 tokens
  Provider gains: +1.0 trust outcome
  
Task disputed and arbitrated:
  Arbitrator receives: fee (from task stake)
  Remaining pool: distributed by payout_pct ruling
```

---

## Design

### Task Completion Flow

Milestone-based resolution with customer review at milestone deadline:

```
┌─────────────────────────────────────────────────────────────────────┐
│  MILESTONE CREATED WITH TASKS                                        │
│                                                                      │
│  - Milestone groups related tasks                                    │
│  - Each task has: provider, stake, work deadline                     │
│  - Milestone deadline = max(task deadlines)                          │
│  - Providers work, deliver off-chain                                 │
└─────────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────────┐
│  AT MILESTONE DEADLINE: Customer must act                            │
│                                                                      │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐      │
│  │ ACCEPT          │  │ DISPUTE         │  │ DO NOTHING      │      │
│  │ MILESTONE       │  │ TASK(S)         │  │                 │      │
│  └────────┬────────┘  └────────┬────────┘  └────────┬────────┘      │
│           │                    │                    │               │
│           ▼                    ▼                    ▼               │
│    All task stakes       Disputed tasks       Timeout:              │
│    released to           enter arbitration    All task stakes       │
│    respective            Non-disputed tasks   released              │
│    providers             paid normally                              │
│    outcome = +1.0                                                   │
└─────────────────────────────────────────────────────────────────────┘
```

**Milestone as payment gate:** Customer reviews all tasks in the milestone at the milestone deadline. Individual task deadlines are work duration allocations, not review triggers.

**Fixed stake model:** Each task has a fixed stake. On acceptance, provider receives 100% of their task's stake. Partial payment requires arbitration.

See **ADR_Milestone_Payment_Gates.md** for full milestone lifecycle.

### Dispute Scope: Task Level

Disputes occur at the **task level** — the most granular contract unit:

```
Project
└── Implementation Phase
    └── Milestone 1
        ├── T1: Database schema ✓ (accepted)
        ├── T2: Password hashing ✓ (accepted)
        └── T3: Session management ✓ (accepted)
    └── Milestone 2
        ├── T4: OAuth2 integration ⚠️ DISPUTED
        ├── T5: Rate limiting ✓ (accepted)
        └── T6: Security audit ✓ (accepted)
```

At Milestone 2 deadline, customer calls `dispute_tasks(milestone_2, [T4])`:
- T4 enters arbitration
- T5 and T6 paid immediately to their providers

**Why task level:**
- Small, concrete scope ("Did OAuth2 work?" not "Was the milestone good?")
- Lower stakes per dispute
- Specific deliverables to evaluate
- Non-disputed tasks in same milestone paid normally
- Evidence is focused and manageable

---

## Fee Structure

### Tier 1: Single Arbitrator

```
Task Stake: 1000 tokens

Tier 1 Fee: 10% of task stake = 100 tokens
Remaining Pool: 900 tokens

Distribution:
  Arbitrator receives: 100 tokens
  Provider receives: payout_pct% of 900 tokens
  Customer receives: (100 - payout_pct)% of 900 tokens
```

### Tier 2: Panel of Three (Appeal)

```
Remaining after Tier 1: 900 tokens

Tier 2 Fee: 15% of remaining = 135 tokens
Final Pool: 765 tokens

Distribution:
  Panel receives: 135 tokens (45 each)
  Provider receives: payout_pct% of 765 tokens
  Customer receives: (100 - payout_pct)% of 765 tokens
```

### Complete Fee Table

| Resolution | Arbitrator(s) | Pool | Provider (100% win) | Customer (100% win) |
|------------|---------------|------|---------------------|---------------------|
| Accepted | 0 | 1000 | 1000 | 0 |
| Tier 1 | 100 | 900 | 900 | 900 |
| Tier 2 | 235 | 765 | 765 | 765 |

**Key insight:** Both parties lose money by disputing, regardless of outcome. This creates strong mutual incentive to resolve without arbitration. Appealing to Tier 2 costs significantly more — strong incentive to accept Tier 1 ruling.

### Fee Commitment Timing

Fees are committed **when arbitrator(s) accept**, not when dispute is filed:

```
1. Customer records outcome (payout_pct)
2. Provider disputes → no fee committed yet
3. Parties negotiate on arbitrator
   ├── Can't agree → timeout rules apply, no fee lost
   └── Agree on arbitrator
4. Arbitrator accepts → 10% (100 tokens) committed
5. Arbitrator rules
   ├── Both accept ruling → done
   └── Either party appeals
6. Parties agree on panel
   ├── Can't agree → Tier 1 ruling stands, no additional fee
   └── Agree on 3 panel members
7. Panel accepts → 15% of remaining (135 tokens) committed
8. Panel rules → FINAL
```

This means:
- Filing a dispute is free (just initiates negotiation)
- Deadlock during arbitrator selection costs nothing but time
- Fee only committed when someone actually takes on the work

---

## Two-Tier Arbitration

```
┌─────────────────────────────────────────────────────────────────────┐
│                         DISPUTED TASK                                │
│                                                                      │
│  Task Stake: 1000 tokens                                             │
│  Status: DISPUTED                                                    │
└─────────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    TIER 1: Single Arbitrator                         │
│                                                                      │
│  Selection: Mutual agreement between customer and provider           │
│  Eligibility: dispute_resolution trust ≥ threshold                   │
│  Fee: 10% of task stake (100 tokens) → committed on acceptance       │
│  Pool: 900 tokens                                                    │
│                                                                      │
│  Arbitrator determines:                                              │
│    • payout_pct (0-100): percentage of pool provider receives        │
│                                                                      │
│  Results:                                                            │
│    • Provider gets: payout_pct% of 900                               │
│    • Customer gets: (100 - payout_pct)% of 900                       │
│    • Trust outcome: (payout_pct - 50) / 50                           │
│                                                                      │
│  Both parties rate arbitrator → dispute_resolution trust updates     │
└─────────────────────────────────────────────────────────────────────┘
                                │
                                │ Either party appeals?
                                ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    TIER 2: Panel of Three                            │
│                                                                      │
│  Selection: Mutual agreement on all 3 panel members                  │
│  Eligibility: dispute_resolution trust ≥ threshold                   │
│  Fee: 15% of remaining pool (135 tokens) → committed on acceptance   │
│  Pool: 765 tokens                                                    │
│                                                                      │
│  Panel determines (majority 2-of-3):                                 │
│    • payout_pct (0-100): percentage of pool provider receives        │
│                                                                      │
│  Results:                                                            │
│    • Provider gets: payout_pct% of 765                               │
│    • Customer gets: (100 - payout_pct)% of 765                       │
│    • Trust outcome: (payout_pct - 50) / 50                           │
│                                                                      │
│  Both parties rate all 3 panel members                               │
│  FINAL — no further appeal                                           │
└─────────────────────────────────────────────────────────────────────┘
```

### Deadlock Resolution

If parties cannot agree on arbitrator(s):

```
Dispute filed
    │
    ▼
Arbitrator negotiation window
    │
    ├── Agreement reached → proceed to arbitration
    │
    └── No agreement (window expires)
        │
        ▼
    Existing timeout rules apply
    No arbitration fee lost
    Neither party "wins"
```

**Rationale:** This creates mutual incentive to find agreement. Neither party benefits from deadlock — both lose time, both risk the timeout outcome.

---

## Flow Details

### Resolution Paths

```
AT MILESTONE DEADLINE:

Path A — Customer accepts milestone:
  1. Customer calls accept_milestone(milestone_id)
  2. All task stakes released to respective providers
  3. All tasks: outcome = +1.0
  4. Milestone complete

Path B — Customer disputes specific task(s):
  1. Customer calls dispute_tasks(milestone_id, [task_ids], evidence_hash)
  2. Disputed tasks enter DISPUTED state → arbitration begins
  3. Non-disputed tasks paid immediately (outcome = +1.0)
  4. Milestone enters PARTIAL_DISPUTE state

AFTER MILESTONE DEADLINE (no customer action):

Path C — Timeout:
  1. Anyone calls timeout_milestone(milestone_id)
  2. All task stakes released to respective providers
  3. All tasks: outcome = +1.0
  4. Milestone complete (customer forfeited review)
```

### Arbitrator Selection (Tier 1)

```
1. Task in DISPUTED state
2. Both parties submit evidence hash (off-chain evidence, on-chain commitment)
3. Selection process:
   a) Either party proposes arbitrator
   b) Other party accepts or counter-proposes
   c) Repeat until mutual agreement or timeout
4. Agreed arbitrator must meet eligibility (dispute_resolution trust ≥ threshold)
5. Arbitrator accepts → 10% fee committed from task stake
6. Tier 1 begins
```

### Tier 1 Resolution

```
1. Arbitrator reviews:
   - Original task contract details
   - Customer's evidence (off-chain, hash committed)
   - Provider's evidence (off-chain, hash committed)
   
2. Arbitrator determines ONE thing:
   - payout_pct: 0-100 (percentage of pool provider receives)
   
3. System calculates:
   - Provider payout: payout_pct% of 900 tokens
   - Customer refund: (100 - payout_pct)% of 900 tokens
   - Trust outcome: (payout_pct - 50) / 50
   
4. Arbitrator calls resolve_dispute(task_id, payout_pct)

5. Both parties rate arbitrator:
   - Customer rates arbitrator
   - Provider rates arbitrator
   - Arbitrator's dispute_resolution trust updates
   - Counterparty weighting applies

6. Appeal window opens
```

### Appeal to Tier 2

```
1. Either party calls appeal(task_id) within appeal_window_blocks
2. Panel selection:
   a) Parties propose panel members
   b) Must agree on all 3
   c) All 3 must meet eligibility threshold
   d) Tier 1 arbitrator excluded from panel
3. Panel members accept → 15% fee (135 tokens) committed
4. Tier 2 begins
```

### Tier 2 Resolution

```
1. Each panel member independently reviews evidence
2. Each panel member submits their payout_pct (0-100)
3. Final payout_pct determined:
   - If 2+ submit same value → that value applies
   - If all 3 differ → median value applies
4. System calculates:
   - Provider payout: payout_pct% of 765 tokens
   - Customer refund: (100 - payout_pct)% of 765 tokens
   - Trust outcome: (payout_pct - 50) / 50
5. Both parties rate all 3 panel members
6. Resolution is FINAL — no further appeal
```

---

## Payout-to-Outcome Mapping

The trust outcome is derived directly from the payout percentage:

```
outcome = (payout_pct - 50) / 50
```

| Payout % | Outcome | Interpretation |
|----------|---------|----------------|
| 0% | -1.0 | Complete failure — full refund to customer |
| 25% | -0.5 | Significant shortfall — mostly refunded |
| 50% | 0.0 | Neutral — pool split evenly |
| 75% | +0.5 | Substantial delivery — mostly paid |
| 100% | +1.0 | Full success — full payment to provider |

This linear mapping ensures:
- Arbitrator makes one decision: payout percentage
- Economic outcome and trust outcome are aligned
- No ambiguity about what "outcome" means

---

## Evidence Handling

Evidence is **off-chain** with **on-chain hash commitment**:

```
┌─────────────────────────────────────────────────────────────────────┐
│                        EVIDENCE MODEL                                │
│                                                                      │
│  On-chain:                                                           │
│    • SHA-256 hash of evidence bundle                                 │
│    • Timestamp of submission                                         │
│    • Submitter signature                                             │
│                                                                      │
│  Off-chain (shared with arbitrator):                                 │
│    • Original task contract terms                                    │
│    • Deliverables (code, documents, etc.)                            │
│    • Communication logs                                              │
│    • Any supporting materials                                        │
│                                                                      │
│  Format: Anything goes                                               │
│    • No required structure                                           │
│    • Arbitrator judges relevance                                     │
│    • Parties responsible for clear presentation                      │
└─────────────────────────────────────────────────────────────────────┘
```

**Delivery mechanism:** Parties share evidence with arbitrator via NIP-17 encrypted DMs or other private channel. The hash commitment proves evidence wasn't modified after submission.

---

## Trust Implications

### For Disputants

| Resolution | Provider Gets | Customer Gets | Trust Outcome |
|------------|---------------|---------------|---------------|
| Customer accepts | 1000 | 0 | +1.0 |
| Timeout (deadline passes) | 1000 | 0 | +1.0 |
| Customer disputes → Tier 1: 100% | 900 | 0 | +1.0 |
| Customer disputes → Tier 1: 75% | 675 | 225 | +0.5 |
| Customer disputes → Tier 1: 50% | 450 | 450 | 0.0 |
| Customer disputes → Tier 1: 25% | 225 | 675 | -0.5 |
| Customer disputes → Tier 1: 0% | 0 | 900 | -1.0 |
| Customer disputes → Tier 2: 100% | 765 | 0 | +1.0 |
| Customer disputes → Tier 2: 0% | 0 | 765 | -1.0 |
| Arbitration deadlock | timeout rules | timeout rules | timeout rules |

The derived outcome affects both parties' skill-specific trust equally.

**Customer's choice:** Accept (provider gets 100%) or dispute (arbitration determines payout). There is no middle ground — if the customer believes the work is unsatisfactory, they must articulate that through arbitration.

### For Arbitrators

| Scenario | Arbitrator Trust Impact |
|----------|------------------------|
| Both parties rate highly | Strong positive signal |
| Winner rates high, loser rates low | Mixed signal (expected) |
| Both parties rate poorly | Strong negative signal |
| Pattern of one-sided rulings | Emerges over time |

**Key insight:** Arbitrators who make defensible rulings — where even the losing party acknowledges fairness — will accumulate the highest trust.

### Counterparty Weighting

Ratings from disputants are weighted by their own trust:

```
arbitrator_trust_update = 
    γ(customer_trust) × customer_rating +
    γ(provider_trust) × provider_rating
```

This means:
- High-trust disputants' ratings matter more
- Low-trust disputants can't tank an arbitrator's reputation unfairly
- Arbitrators have incentive to satisfy high-trust parties

---

## Economic Incentives

### Incentive Alignment

| Party | Incentive |
|-------|-----------|
| **Customer** | Accept satisfactory work; dispute only for genuine failures |
| **Provider** | Deliver quality work; completion triggers acceptance window |
| **Arbitrator** | Rule fairly to build reputation |
| **Appellant** | Appeal only when Tier 1 was clearly wrong |

### Cost of Dispute

| Action | Cost to Initiator | Cost to Other Party |
|--------|-------------------|---------------------|
| File dispute | 0 (just time) | 0 (just time) |
| Tier 1 proceeds | 10% of stake | 10% of stake |
| Appeal to Tier 2 | additional 15% | additional 15% |

**Key insight:** Dispute is expensive for both parties. Even the "winner" loses 10-23.5% of the original stake. This creates strong incentive to:
1. Customers: Accept satisfactory work
2. Providers: Deliver acceptable quality
3. Both: Settle at Tier 1 rather than appeal

---

## Contract Structure

### Task Contract

```noir
struct TaskContract {
    task_id: Field,
    project_id: Field,
    parent_milestone_id: Field,           // → Milestone (payment gate)
    customer: AztecAddress,
    provider: AztecAddress,
    skill_type: Field,
    stake: Field,                         // Customer-funded task stake (fixed amount)
    difficulty: Field,
    
    // Status tracking
    status: TaskStatus,
    
    // Timing parameters
    deadline: Field,                      // Work duration deadline (planning data)
    appeal_window_blocks: Field,          // Time to appeal Tier 1 ruling
    
    // Arbitration parameters (inherited from project)
    arbitrator_min_trust: Field,          // Eligibility threshold
    
    // Resolution
    outcome: Option<Field>,               // Final outcome (scaled)
    payout_pct: Option<Field>,            // Final payout percentage (100 unless arbitrated)
}

enum TaskStatus {
    Active,              // Work in progress
    Accepted,            // Paid via milestone acceptance or timeout
    Disputed,            // Arbitration in progress
    Resolved,            // Arbitration complete
}
```

**Note:** Task `deadline` is a work duration allocation used for planning and evidence in disputes. The review trigger is the parent milestone's `completion_deadline` (derived from max of task deadlines).
```

### Dispute State

```noir
enum DisputeStatus {
    None,                    // No dispute
    Initiated,               // Customer disputed, awaiting arbitrator selection
    Tier1Pending,            // Arbitrator agreed, awaiting acceptance
    Tier1InProgress,         // Arbitrator reviewing
    Tier1Resolved,           // Tier 1 complete, appeal window open
    AppealInitiated,         // Appeal filed, awaiting panel selection
    Tier2Pending,            // Panel agreed, awaiting acceptance
    Tier2InProgress,         // Panel reviewing
    Resolved,                // Final resolution
    Deadlocked,              // Couldn't agree on arbitrator(s)
}

struct DisputeState {
    status: DisputeStatus,
    initiated_at: Field,
    
    // Evidence commitments
    customer_evidence_hash: Field,
    provider_evidence_hash: Field,
    
    // Tier 1
    tier1_arbitrator: Option<AztecAddress>,
    tier1_accepted_at: Option<Field>,
    tier1_payout_pct: Option<Field>,
    tier1_resolved_at: Option<Field>,
    
    // Tier 2 (if appealed)
    appellant: Option<AztecAddress>,
    appeal_initiated_at: Option<Field>,
    panel: Option<[AztecAddress; 3]>,
    panel_accepted_at: Option<Field>,
    panel_votes: Option<[Field; 3]>,
    tier2_payout_pct: Option<Field>,
}
```

### Arbitration Contract

When an arbitrator accepts, a standard contract is created:

```noir
struct ArbitrationContract {
    contract_type: Field,                  // = ARBITRATION
    disputed_task_id: Field,               // Reference to disputed task
    
    // Arbitrator as provider
    provider: AztecAddress,                // The arbitrator
    skill_type: Field,                     // = DISPUTE_RESOLUTION
    
    // Fee (earned by arbitrator)
    fee: Field,                            // 10% of task stake (Tier 1) or 15% of pool (Tier 2)
    
    // Panel info (if Tier 2)
    is_panel_member: bool,
    panel_position: Option<Field>,         // 0, 1, or 2
}
```

---

## Escrow Operations

### Milestone-Level Entry Points

```noir
// Customer accepts entire milestone — all tasks paid
fn accept_milestone(milestone_id: Field) {
    let milestone = get_milestone(milestone_id);
    assert(msg_sender() == milestone.customer);
    assert(milestone.status == MilestoneStatus::Active);
    assert(current_block() <= milestone.completion_deadline);
    
    // Pay each task's stake to its provider
    for task in milestone.tasks {
        transfer(task.provider, task.stake);
        
        task.status = TaskStatus::Accepted;
        task.outcome = Some(PRECISION);  // +1.0 scaled
        task.payout_pct = Some(100);
        
        // Update provider trust
        update_trust(task.provider, task.skill_type, PRECISION);
    }
    
    // Update customer trust
    update_trust(milestone.customer, SKILL_CUSTOMER_VERIFICATION, PRECISION);
    
    milestone.status = MilestoneStatus::Accepted;
    emit MilestoneAccepted { milestone_id };
}

// Customer disputes specific tasks — non-disputed tasks paid immediately
fn dispute_tasks(milestone_id: Field, task_ids: [Field], evidence_hash: Field) {
    let milestone = get_milestone(milestone_id);
    assert(msg_sender() == milestone.customer);
    assert(milestone.status == MilestoneStatus::Active);
    assert(current_block() <= milestone.completion_deadline);
    
    for task in milestone.tasks {
        if task_ids.contains(task.task_id) {
            // Disputed task enters arbitration
            task.status = TaskStatus::Disputed;
            task.dispute_state = Some(DisputeState {
                status: DisputeStatus::Initiated,
                initiated_at: current_block(),
                customer_evidence_hash: evidence_hash,
                ..Default::default()
            });
        } else {
            // Non-disputed task paid immediately
            transfer(task.provider, task.stake);
            
            task.status = TaskStatus::Accepted;
            task.outcome = Some(PRECISION);
            task.payout_pct = Some(100);
            
            update_trust(task.provider, task.skill_type, PRECISION);
        }
    }
    
    milestone.status = MilestoneStatus::PartialDispute;
    emit TasksDisputed { milestone_id, task_ids, evidence_hash };
}

// Timeout — all tasks paid after milestone deadline passes
fn timeout_milestone(milestone_id: Field) {
    let milestone = get_milestone(milestone_id);
    assert(milestone.status == MilestoneStatus::Active);
    assert(current_block() > milestone.completion_deadline);
    
    // Pay all tasks — customer forfeited their review
    for task in milestone.tasks {
        transfer(task.provider, task.stake);
        
        task.status = TaskStatus::Accepted;
        task.outcome = Some(PRECISION);
        task.payout_pct = Some(100);
        
        update_trust(task.provider, task.skill_type, PRECISION);
    }
    
    update_trust(milestone.customer, SKILL_CUSTOMER_VERIFICATION, PRECISION);
    
    milestone.status = MilestoneStatus::TimedOut;
    emit MilestoneTimedOut { milestone_id };
}
```

### Task-Level Arbitration (once dispute filed)

```noir
// Provider submits evidence (in response to customer's dispute)
fn submit_provider_evidence(task_id: Field, evidence_hash: Field) {
    assert(msg_sender() == task.provider);
    assert(task.status == TaskStatus::Disputed);
    
    task.dispute_state.provider_evidence_hash = evidence_hash;
    
    emit ProviderEvidenceSubmitted { task_id, evidence_hash };
}

// Arbitrator accepts — fee committed
fn accept_arbitration(task_id: Field) {
    let arbitrator = msg_sender();
    assert(task.status == TaskStatus::Disputed);
    assert(task.dispute_state.tier1_arbitrator == Some(arbitrator));
    assert(get_trust(arbitrator, SKILL_DISPUTE_RESOLUTION) >= task.arbitrator_min_trust);
    
    task.dispute_state.status = DisputeStatus::Tier1InProgress;
    task.dispute_state.tier1_accepted_at = Some(current_block());
    
    // Fee is committed but not transferred yet
    let fee = task.stake * 10 / 100;  // 10%
    
    emit ArbitrationAccepted { task_id, arbitrator, fee };
}

// Arbitrator resolves Tier 1
fn resolve_tier1(task_id: Field, payout_pct: Field) {
    assert(payout_pct <= 100);
    assert(msg_sender() == task.dispute_state.tier1_arbitrator.unwrap());
    assert(task.dispute_state.status == DisputeStatus::Tier1InProgress);
    
    // Calculate amounts
    let fee = task.stake * 10 / 100;           // 10% to arbitrator
    let pool = task.stake - fee;                // 90% remaining
    let provider_payout = pool * payout_pct / 100;
    let customer_refund = pool - provider_payout;
    
    // Transfer funds
    transfer(task.dispute_state.tier1_arbitrator.unwrap(), fee);
    transfer(task.provider, provider_payout);
    transfer(task.customer, customer_refund);
    
    // Calculate outcome: (payout_pct - 50) / 50, scaled
    let outcome = (payout_pct as i64 - 50) * PRECISION as i64 / 50;
    
    task.dispute_state.status = DisputeStatus::Tier1Resolved;
    task.dispute_state.tier1_payout_pct = Some(payout_pct);
    task.dispute_state.tier1_resolved_at = Some(current_block());
    
    // Don't finalize yet — appeal window opens
    emit Tier1Resolved { task_id, payout_pct, fee, provider_payout, customer_refund };
}

// Appeal Tier 1 ruling
fn appeal(task_id: Field) {
    assert(task.dispute_state.status == DisputeStatus::Tier1Resolved);
    assert(current_block() <= task.dispute_state.tier1_resolved_at.unwrap() + task.appeal_window_blocks);
    assert(msg_sender() == task.customer || msg_sender() == task.provider);
    
    task.dispute_state.status = DisputeStatus::AppealInitiated;
    task.dispute_state.appellant = Some(msg_sender());
    task.dispute_state.appeal_initiated_at = Some(current_block());
    
    emit AppealInitiated { task_id, appellant: msg_sender() };
}

// Panel accepts — additional fee committed
fn accept_panel_arbitration(task_id: Field) {
    let panelist = msg_sender();
    assert(task.dispute_state.status == DisputeStatus::AppealInitiated);
    assert(task.dispute_state.panel.unwrap().contains(&panelist));
    
    // Track panel acceptance (need all 3)
    // ... acceptance tracking logic ...
    
    // When all 3 accept:
    task.dispute_state.status = DisputeStatus::Tier2InProgress;
    task.dispute_state.panel_accepted_at = Some(current_block());
    
    // Tier 2 fee calculated from remaining pool after Tier 1
    let tier1_fee = task.stake * 10 / 100;
    let remaining = task.stake - tier1_fee;
    let tier2_fee = remaining * 15 / 100;  // 15% of remaining
    
    emit PanelAccepted { task_id, tier2_fee };
}

// Panel member submits vote
fn submit_panel_vote(task_id: Field, payout_pct: Field) {
    assert(payout_pct <= 100);
    let panelist = msg_sender();
    assert(task.dispute_state.status == DisputeStatus::Tier2InProgress);
    
    let position = get_panel_position(task_id, panelist);
    task.dispute_state.panel_votes[position] = payout_pct;
    
    // Check if all 3 votes in
    if all_votes_submitted(task_id) {
        finalize_tier2(task_id);
    }
}

// Finalize Tier 2 — called automatically when all votes in
fn finalize_tier2(task_id: Field) {
    let votes = task.dispute_state.panel_votes.unwrap();
    
    // Determine final payout_pct (majority or median)
    let final_payout_pct = if votes[0] == votes[1] || votes[0] == votes[2] {
        votes[0]
    } else if votes[1] == votes[2] {
        votes[1]
    } else {
        // All different — use median
        median(votes)
    };
    
    // Calculate amounts
    let tier1_fee = task.stake * 10 / 100;
    let after_tier1 = task.stake - tier1_fee;
    let tier2_fee = after_tier1 * 15 / 100;
    let pool = after_tier1 - tier2_fee;          // 765 for 1000 stake
    
    let provider_payout = pool * final_payout_pct / 100;
    let customer_refund = pool - provider_payout;
    
    // Transfer funds
    let per_panelist = tier2_fee / 3;
    for panelist in task.dispute_state.panel.unwrap() {
        transfer(panelist, per_panelist);
    }
    transfer(task.provider, provider_payout);
    transfer(task.customer, customer_refund);
    
    // Calculate outcome
    let outcome = (final_payout_pct as i64 - 50) * PRECISION as i64 / 50;
    
    task.status = TaskStatus::Resolved;
    task.outcome = Some(outcome);
    task.payout_pct = Some(final_payout_pct);
    task.dispute_state.status = DisputeStatus::Resolved;
    task.dispute_state.tier2_payout_pct = Some(final_payout_pct);
    
    // Update trust
    update_trust(task.provider, task.skill_type, outcome);
    update_trust(task.customer, SKILL_CUSTOMER_VERIFICATION, outcome);
    
    emit Tier2Resolved { 
        task_id, 
        payout_pct: final_payout_pct, 
        votes,
        tier2_fee,
        provider_payout, 
        customer_refund 
    };
}

// Finalize Tier 1 if no appeal
fn finalize_tier1(task_id: Field) {
    assert(task.dispute_state.status == DisputeStatus::Tier1Resolved);
    assert(current_block() > task.dispute_state.tier1_resolved_at.unwrap() + task.appeal_window_blocks);
    
    let payout_pct = task.dispute_state.tier1_payout_pct.unwrap();
    let outcome = (payout_pct as i64 - 50) * PRECISION as i64 / 50;
    
    task.status = TaskStatus::Resolved;
    task.outcome = Some(outcome);
    task.dispute_state.status = DisputeStatus::Resolved;
    
    // Update trust
    update_trust(task.provider, task.skill_type, outcome);
    update_trust(task.customer, SKILL_CUSTOMER_VERIFICATION, outcome);
    
    emit Tier1Finalized { task_id, payout_pct, outcome };
}
```

---

## Edge Cases
## Edge Cases

### Edge Case 1: Arbitrator Becomes Unavailable

**Scenario:** Agreed arbitrator goes offline after accepting.

**Resolution:** Arbitration has its own timeout. If arbitrator doesn't rule within deadline:
- Parties can agree on replacement
- If no agreement, dispute deadlocks → timeout rules apply

### Edge Case 2: Panel Disagreement

**Scenario:** All 3 panel members submit different payout percentages.

**Resolution:** Median value applies.

Example: Panel submits [40%, 60%, 80%] → Final payout = 60%

### Edge Case 3: No Eligible Arbitrators

**Scenario:** No arbitrators meet the trust threshold.

**Resolution:** 
- Parties can mutually agree to lower threshold
- If no arbitrator found within window, dispute deadlocks
- Deadlock → timeout rules apply

### Edge Case 4: Dispute on Multiple Tasks in Same Milestone

**Scenario:** Customer disputes several tasks within one milestone.

**Resolution:** Each task is disputed independently. Parties may agree to:
- Use same arbitrator for efficiency
- Bundle evidence across tasks
- But each task has its own ruling and fee

Non-disputed tasks in the milestone are paid immediately.

### Edge Case 5: Provider Abandons Task

**Scenario:** Provider abandons task without completing work.

**Resolution:** At milestone deadline, customer disputes the abandoned task. With no deliverables, arbitrator will likely rule 0% payout. Other completed tasks in the milestone are paid normally.

---

## Consequences

### Positive

1. **No new primitives** — Uses existing contract structure
2. **Self-regulating** — Bad arbitrators lose trust
3. **Decentralized** — No central authority
4. **Aligned incentives** — Everyone benefits from honest behavior
5. **Fair to both parties** — Provider can't be unfairly rejected
6. **Expensive to abuse** — Both parties lose 10-23.5% in any dispute

### Negative

1. **Complexity** — Dispute flow is more complex than simple accept
2. **Bootstrapping** — Initial arbitrator pool may be thin
3. **Cost** — Even "winners" lose money to arbitration fees

### Mitigations

- Complexity hidden in UX; most tasks never dispute
- Early arbitrators can emerge from trusted community members
- Fee cost is a feature: discourages frivolous disputes

---

## Constants

```noir
// Dispute resolution skill type (reserved)
global SKILL_DISPUTE_RESOLUTION: Field = 2000001;

// Fee percentages
global TIER1_FEE_PCT: Field = 10;    // 10% of task stake
global TIER2_FEE_PCT: Field = 15;    // 15% of remaining pool

// Arbitration timing (in blocks, ~12 second blocks)
global DEFAULT_APPEAL_WINDOW: Field = 25200;        // ~3.5 days to appeal Tier 1
global DEFAULT_ARBITRATION_TIMEOUT: Field = 100800; // ~14 days for arbitrator to rule
```

Note: Milestone `completion_deadline` is derived from max(task deadlines). Task deadlines are work duration allocations set during Planning phase.

---

## Related Documents

- **ADR_Milestone_Payment_Gates.md** — Milestone lifecycle and payment flow
- **ADR_Subcontract_Architecture.md** — Recursive contract structure
- **ADR_Trust_Signal_Boundaries.md** — What affects trust calculations
- **QoT_Aztec_Contract_Layer_Specification.md** — Contract lifecycle
- **The_Difficulty_of_Assessing_Difficulty.md** — Task decomposition rationale

---

## Summary

Dispute resolution is another contracted skill. Arbitrators build `dispute_resolution` trust through fair rulings. Two-tier arbitration (single → panel of 3) provides recourse without endless appeals. Mutual agreement on arbitrators prevents gaming; deadlock falls back to existing timeout rules.

**The arbitrator decides one thing:** payout percentage (0-100%). Stake distribution and trust outcome are derived automatically.

**Both parties lose money in any dispute** — 10% at Tier 1, 23.5% total if appealed to Tier 2. This creates strong incentive for customers to accept satisfactory work rather than dispute.

**Milestone-based resolution.** Customer reviews tasks at milestone deadline — accepting the milestone, disputing specific tasks, or letting it timeout. Non-disputed tasks are paid immediately; disputed tasks enter arbitration. Stake is fixed per task — there is no partial payment without arbitration. Provider stakes labor, not capital.

No new primitives. Same trust math. Same contract structure. Disputes are just another type of work.
