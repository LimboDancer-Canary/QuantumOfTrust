# Extending Nostria with QoT Professional Network Pages

*An analysis of client-side UX for Quantum of Trust as professional infrastructure*

---

## Executive Summary

LinkedIn's core value proposition--professional reputation enabling opportunity access--maps remarkably well to QoT's architecture. But where LinkedIn conflates identity with reputation (your legal name IS your credential), QoT can deliver the same value while preserving pseudonymity. This analysis explores how a Nostr-based client like Nostria could become **the professional network for the pseudonymous economy**.

The key insight: **LinkedIn is fundamentally a trust-gating system**. Recruiters filter by work history, endorsements, and network proximity. QoT formalizes this into mathematical eligibility--and enables it without identity disclosure.

---

## Part One: What LinkedIn Actually Does

Before designing the QoT version, let's be precise about LinkedIn's primitives:

| LinkedIn Feature | Underlying Function | Trust Signal |
|------------------|---------------------|--------------|
| Work History | Sequence of employment contracts | Someone hired you; you didn't get fired |
| Skills & Endorsements | Peer attestations | N people vouch for competency X |
| Recommendations | Detailed testimonials | Past counterparties describe outcomes |
| Job Applications | Eligibility filtering | "5+ years experience" = trust threshold |
| Company Pages | Organizational identity | Aggregate credibility of employees |
| Connections | Social graph | Proximity to trusted nodes |
| Posts & Articles | Signal without stakes | Low-weight trust indicators |

**Critical observation**: LinkedIn's trust signals are *self-attested* or *socially attested*, not *contract-verified*. You claim you worked at Google; maybe a Google employee endorses you; but the network has no cryptographic proof of outcomes.

QoT can do better: **trust from verified contract execution, not claims**.

---

## Part Two: Mapping LinkedIn to QoT Architecture

### Profile -> Avatar with Skill-Scoped Trust

A Nostria profile already shows:
- **npub**: Cryptographic identity
- **Display name**: "SondreB"
- **NIP-05**: sondreb@nostria.app (human-readable)
- **Lightning address**: Payment integration
- **Bio**: Unverified self-description

**QoT enhancement**: Add *computed* trust indicators per skill domain:

```
+-----------------------------------------------------------------+
|  SondreB +                                       [Follow] [Zap] |
|  sondreb@nostria.app                                            |
|                                                                 |
|  "Founder of Nostria. Founder of Liberstad."                    |
|                                                                 |
|  +-------------------------------------------------------------+|
|  |  VERIFIED CAPABILITIES                                      ||
|  |                                                             ||
|  |  TypeScript Development    ################....  78.3 cutes ||
|  |    47 completed contracts . 12 unique clients               ||
|  |                                                             ||
|  |  Product Management        ########............  42.1 cutes ||
|  |    23 completed contracts . 8 unique clients                ||
|  |                                                             ||
|  |  [Prove Eligibility: >=50 cutes in TypeScript]               ||
|  +-------------------------------------------------------------||
|                                                                 |
|  Following 307 . Relays 4 . Contracts 70 . DAO Memberships 3   |
+-----------------------------------------------------------------+
```

**Key differences from LinkedIn**:
- Trust values are *computed from verified contract outcomes*, not self-claims
- "Prove Eligibility" button generates ZK proof for specific threshold
- History is cryptographically committed, not editable

### Work History -> Contract Outcomes (Privacy-First)

LinkedIn shows a sequence of job titles and companies. QoT shows **aggregate trust per skill**--individual contracts are private by default.

**Public profile (visible to all):**
```
+-----------------------------------------------------------------+
|  SondreB                                         [Follow] [Zap] |
|  sondreb@nostria.app                                            |
|                                                                 |
|  VERIFIED SKILLS                                                |
|  TypeScript Development     ################....  78.3 cutes   |
|    47 contracts completed                                       |
|  System Architecture        ############........  42.1 cutes   |
|    23 contracts completed                                       |
|  Requirements Analysis      ########............  38.7 cutes   |
|    31 contracts completed                                       |
|                                                                 |
|  [Prove Eligibility >=50 in TypeScript]                         |
|                                                                 |
|  PUBLISHED WORK (both parties consented)                        |
|  +------------------------------------------------------------+ |
|  |  Nostria Relay Integration | with Nostria DAO | +0.94      | |
|  |  ZK Verifier Plugin        | with ########   | +0.88      | |
|  +------------------------------------------------------------+ |
+-----------------------------------------------------------------+
```

**Key privacy properties:**
- Trust scores are public (aggregate outcome)
- Contract count is public (aggregate only)
- Individual contract details require **mutual publication consent**
- Counterparty identity never revealed without their consent
- "Published Work" section shows only mutually-disclosed contracts

**Private dashboard (visible only to Avatar owner):**
```
+-----------------------------------------------------------------+
|  MY CONTRACT HISTORY (Private)                                  |
|                                                                 |
|  +------------------------------------------------------------+ |
|  |  Dec 15 | TypeScript | +0.92 | +8.7 cutes |  Published   | |
|  |  Dec 02 | Architecture | +0.85 | +4.2 cutes | + Private   | |
|  |  Nov 28 | TypeScript | -0.35 | -2.1 cutes | + Private     | |
|  |  Nov 15 | Requirements | +0.78 | +3.1 cutes | + Private   | |
|  +------------------------------------------------------------+ |
|                                                                 |
|  [Request Publication] -> sends consent request to counterparty  |
+-----------------------------------------------------------------+
```

The Avatar owner sees all their contracts. Negative outcomes are visible here and computed into trust--you can't delete a failed project from your history. But whether others see specific contracts depends on mutual consent.

### Customer Profile  Avatar with Consumer Trust

Just as providers earn trust from contract outcomes, **customers earn trust from their behavior** as contract consumers. This creates bidirectional accountability.

**Customer trust dashboard (visible when viewing a potential customer):**

```

  Acme Corp                                      [Follow] [Message]  
  acme@nostria.app                                                   
                                                                     
  "Building the future of decentralized commerce"                    
                                                                     
    
    CUSTOMER BEHAVIOR METRICS                                      
                                                                   
    Project Commitment        95% (19/20)     
      Projects completed vs initiated                              
                                                                   
    Escrow Discipline         92%             
      On-time funding . Prompt release                             
                                                                   
    Verification Integrity    78%             
      Rating variance: 0.32 (healthy discrimination)               
                                                                   
    Scope Stability           68%             
      Tasks as planned: 34/50                                      
                                                                   
    Timeline Realism          62%             
      Avg deadline accuracy: -12% (tends to underestimate)         
                                                                   
    Spec Quality              88%             
      Impl outcomes for approved specs                             
    
                                                                     
  RECENT PROJECTS (published with consent)                           
    
    Nostr Relay v2         3 phases  Completed  All +0.8+      
    Mobile App MVP         2 phases  In Progress                
    

```

**Customer trust dimensions explained:**

| Metric | Source | What It Signals |
|--------|--------|------------------|
| **Project Commitment** | completed / initiated | Reliable partner who sees projects through |
| **Escrow Discipline** | On-time funding, prompt release | Respects financial agreements |
| **Verification Integrity** | Rating variance  0.35 ideal | Discriminating, not rubber-stamping |
| **Scope Stability** | Tasks as planned / total | Does not constantly change requirements |
| **Timeline Realism** | Planned vs actual duration | Sets achievable deadlines |
| **Spec Quality** | Impl outcomes | Specs they approve lead to good outcomes |

**Provider view before accepting a contract:**

```

  CONTRACT OFFER                                                     
                                                                     
  From: Acme Corp                                                    
  Skill Type: TypeScript Development                                 
  Stake: 5,000 sats                                                  
  Deadline: 30 days                                                  
                                                                     
    
    CUSTOMER TRUST SUMMARY                                         
                                                                   
    Overall:   78/100                         
                                                                   
      Scope Stability: 68% (moderate scope creep risk)           
      Timeline Realism: 62% (tends to underestimate)             
      Escrow Discipline: 92% (reliable payments)                  
      Project Commitment: 95% (finishes what they start)          
                                                                   
    Verification Weight: 0.85x                                     
      (Their ratings count 85% due to verification integrity)      
    
                                                                     
  [View Tasks]  [Accept & Assess Difficulty]  [Decline]              

```

**Key insight:** Providers can make informed decisions about customers just as customers evaluate providers. The **verification weight** shows how much this customer's eventual rating will count—rubber-stamping or erratic customers have reduced influence on provider trust.

**Note on difficulty:** The contract offer does not include a difficulty rating. Difficulty is assessed at the task level by the provider during acceptance. The provider reviews the task breakdown (from the Planning phase), assesses difficulty for each task, and can request task refinement before committing. See [The Difficulty of Assessing Difficulty](./The_Difficulty_of_Assessing_Difficulty.md).


### Endorsements -> Removed (Contract Outcomes Are Sufficient)

LinkedIn has endorsements ("Alice endorsed Bob for JavaScript"). QoT does not.

The contract outcome IS the attestation. Both parties signing off on phase completion is the verification. A separate endorsement layer would be either redundant (if limited to counterparties) or gameable (if open to anyone).

### Company Pages -> DAO Profiles

Organizations become DAOs with aggregate trust:

```
+-----------------------------------------------------------------+
|  DAO: Nostria Development Collective                            |
|  dao://nostria-dev                                              |
|                                                                 |
|  Aggregation: Average (mean member capability)                  |
|                                                                 |
|  +-------------------------------------------------------------+|
|  |  COLLECTIVE TRUST                                           ||
|  |                                                             ||
|  |  TypeScript Development    ####################  92.7 cutes ||
|  |    (avg of 5 members)                                       ||
|  |                                                             ||
|  |  UI/UX Design              ############........  61.2 cutes ||
|  |    (avg of 3 members)                                       ||
|  +-------------------------------------------------------------||
|                                                                 |
|  Members (5):                                                   |
|  - SondreB (78.3 cutes TS)                                     |
|  - #### (102.1 cutes TS)                                       |
|  - #### (95.4 cutes TS)                                        |
|  - #### (87.2 cutes TS)                                        |
|  - #### (100.5 cutes TS)                                       |
|                                                                 |
|  [View DAO Contract History] [Apply for Membership]             |
+-----------------------------------------------------------------+
```

**DAO aggregation choices**:
- **Sum**: Total collective capability (capacity-focused)
- **Average**: Mean reliability (quality-focused)
- **Minimum**: Weakest-link analysis (security-focused)
- **Maximum**: Strongest member (best-case capability)

---

## Part Three: Contract Lifecycle Phases

The current QoT model treats contracts as atomic: they exist, then they have an outcome. But professional work is *phased*. A software project typically moves through:

1. **Specification** -> Agreement on scope and deliverables
2. **Planning** -> Implementation approach, task breakdown, milestone groupings
3. **Implementation** -> Actual work, with milestone-based payment gates
4. **Verification** -> Customer acceptance (tests passing, demo accepted)

The first three phases represent *trust-relevant events* for providers. Completing specification demonstrates requirements analysis skill. Completing planning demonstrates architecture skill. Completing implementation demonstrates development skill.

Verification is different--it's the customer's acceptance mechanism that determines the implementation outcome.

**Milestone-based payment**: Within Implementation, tasks are grouped into milestones. Customer reviews at milestone deadline (not per-task): accept all tasks, dispute specific tasks, or timeout. This creates incremental payment gates that reduce provider risk. See **ADR_Milestone_Payment_Gates.md**.

### Proposed: Multi-Phase Contract Structure

**Design Principle**: QoT captures *trust-relevant outcomes*, not project management details. The contract lifecycle exists in the real world; the framework records only the signals that flow into trust calculations.

**Multi-provider contracts**: Different Avatars may perform different phases. Each phase has its own provider who earns trust for that skill.

**Verification is special**: The customer (or their delegate) performs verification. This is the acceptance mechanism that determines the implementation outcome—not a trust-earning phase for a provider. However, the customer earns trust from their verification integrity (how well they rate).

```
c_phased = (
  consumer: a_customer,
  s,                   // Total stake  
  d,                   // Aggregate difficulty (from task difficulties)
  τ,                   // Escrow commitment
  V_consumer,          // Consumer's trust at contract creation
  
  phases: [
    {
      phase_type: "specification",
      provider: a_business_analyst,
      skill_type: t_requirements,
      stake_portion: 0.20,
      outcome: +0.95,
      signed_by: [a_business_analyst, a_customer],
      completed_at: timestamp
    },
    {
      phase_type: "planning", 
      provider: a_architect,
      skill_type: t_architecture,
      stake_portion: 0.20,
      outcome: +0.85,
      signed_by: [a_architect, a_customer],
      planned_task_count: 12,
      completed_at: timestamp
    },
    {
      phase_type: "implementation",
      provider: a_developer,           // Lead provider (may delegate tasks)
      skill_type: t_development,
      stake_portion: 0.60,
      outcome: +0.90,                  // Aggregate from milestone/task outcomes
      signed_by: [a_developer, a_customer],
      
      milestones: [                    // Payment gates within implementation
        {
          milestone_id: 1,
          tasks: [task_1, task_2, task_3],
          deadline: timestamp_1,       // max(task deadlines) in this milestone
          status: "completed",
          stake: computed_from_tasks   // sum(task stakes)
        },
        {
          milestone_id: 2,
          tasks: [task_4, task_5, task_6],
          deadline: timestamp_2,
          status: "in_progress",
          stake: computed_from_tasks
        }
      ],
      
      tasks_completed_as_planned: 10,
      tasks_deviated: 2,
      completed_at: timestamp
    }
  ],
  
  verification: {
    performed_by: a_customer,      // Or delegate acting as customer's agent
    criteria_met: [true, true, true],
    implementation_outcome: +0.90  // This value flows to implementation phase
  }
)
```

**Trust flows**:
- `a_business_analyst` earns trust in `t_requirements`
- `a_architect` earns trust in `t_architecture`  
- `a_developer` earns trust in `t_development`
- `a_customer` earns trust from their behavior: commitment (completing vs abandoning), escrow discipline (timely funding/release), verification integrity (rating quality), and scope stability
- If customer delegates verification, that's either internal (no trust flow) or a separate contract

**What flows into trust calculations:**
- Phase outcomes (quality of deliverable)
- Mutual sign-off events (commitment points)
- Planning accuracy ratio (tasks_completed_as_planned / planned_task_count)

**What is recorded but NOT trust-weighted:**
- Publication consent (privacy control, not trust signal)
- Verification details (acceptance mechanism, not provider work)

**What does NOT belong in QoT:**
- Task descriptions or contents
- Document contents or artifacts
- Communication logs
- Detailed project management data

The mutual sign-off requirement is what prevents gaming. A provider can't pad with trivial tasks because the consumer must approve the plan. A consumer can't claim "this isn't what I wanted" if they signed the specification. The adversarial dynamic between parties provides validation.

**Publication consent** follows the same mutual sign-off pattern but controls *visibility*, not trust weight. For multi-provider contracts, publication requires consent from consumer AND any provider whose phase would be revealed.

**QA as a separate contract**: Quality Assurance is a valid skill type. QA professionals earn trust through their own contracts--not as a phase within someone else's build contract:

```
Contract A: Build the thing
  consumer: a_customer
  phases: [spec, planning, implementation]
  verification: performed by a_customer
  
Contract B: Test the thing (separate contract)
  consumer: a_customer
  provider: a_qa_professional
  phases: [
    { spec: "test plan scope" },
    { planning: "test strategy" },
    { implementation: "execute tests, report findings" }
  ]
  verification: performed by a_customer (did QA do good work)
```

The QA professional earns trust in "Quality Assurance" skill based on how well they performed their testing work--as judged by the customer. The outcome of Contract B may inform the customer's verification decision on Contract A, but that's operational coordination, not part of the trust framework.

### Benefits of Multi-Phase Contracts

**1. Granular Trust Attribution**

Each phase contributes to its provider's skill type:
```
For each phase p in c where signed_by includes provider and consumer:
  V_{skill(p)}(provider(p)) += (p) . outcome(p) . (c) . (c)
```

A single contract can distribute trust across multiple Avatars in their respective domains.

**2. Specialist Collaboration**

Different Avatars can contribute their expertise:
- Business analyst delivers specification -> earns Requirements trust
- Architect delivers plan -> earns Architecture trust
- Developer delivers implementation -> earns Development trust

The consumer coordinates, but each specialist earns trust independently.

**3. Early Exit with Partial Credit**

If implementation fails but spec/planning succeeded, those providers retain positive outcomes. Trust reflects actual contribution per provider, not all-or-nothing for a single Avatar.

**4. Planning Accuracy as Trust Signal**

```
planning_accuracy(c) = tasks_completed_as_planned / planned_task_count
```

This ratio becomes a trust signal for **Project Planning** skill. The planner (architect) who consistently delivers accurate plans has demonstrated predictability--valuable independent of who implements.

**5. Mutual Sign-Off Prevents Gaming**

Each phase requires sign-off from that phase's provider AND the consumer. This creates natural validation:
- Provider can't pad with trivial tasks (consumer rejects the plan)
- Provider can't claim unrealistic baselines (they proposed the plan)
- Consumer can't claim surprise (they signed the specification)

**6. Verification as Acceptance Mechanism**

The consumer (or delegate) performs verification. This determines the implementation outcome. Verification is not a trust-earning phase like provider work, but customers earn trust from verification integrity—how discriminatingly and fairly they rate.

### Phase State Machine (Trust-Relevant Transitions Only)

The framework tracks state transitions that affect trust calculations. Three phases earn trust for providers; verification is the customer's acceptance mechanism.

```
+----------+
| CREATED  |  Contract exists, escrow locked
+----+-----+
     | Provider + Consumer sign specification
     
+----------+
| SPEC_OK  |  +trust(requirements) for spec provider
+----+-----+
     | Provider + Consumer sign plan (commits task baseline)
     
+----------+
| PLAN_OK  |  +trust(architecture) for planning provider
+----+-----+
     | Provider submits implementation
     
+----------+
|IMPL_READY|  Awaiting verification
+----+-----+
     | Consumer (or delegate) performs verification
     
+----------+
| VERIFIED |  Implementation outcome determined
+----+-----+
     | Provider + Consumer sign off on outcome
     
+----------+
| COMPLETE |  +trust(development) + planning_accuracy, escrow released
+----------+

At any transition:
     | Dispute raised
     
+----------+
| DISPUTED |  Resolution determines final outcomes
+----------+
```

**What the framework records per phase transition:**
- Phase skill type
- Provider for that phase
- Outcome value [-1, +1]
- Sign-off from provider + consumer
- Timestamp
- For PLAN_OK: committed task count
- For COMPLETE: tasks completed as planned

**What the framework does NOT track:**
- Deadlines (operational, not trust-relevant)
- Artifact contents (external concern)
- Work-in-progress states (only signed completions matter)
- Verification details (acceptance mechanism, not provider work)

---

## Part Four: UX Flows for Professional Network

### Flow 1: Creating a Gig/Contract Request

*The "Job Posting" equivalent*

```
+-----------------------------------------------------------------+
|  CREATE NEW CONTRACT REQUEST                                    |
+-----------------------------------------------------------------+
|                                                                 |
|  Title: [Build Nostr Relay with Trust Filtering              ] |
|                                                                 |
|  Primary Skill: [TypeScript Development        ]               |
|                                                                 |
|  PHASES                                                         |
|  +------------------------------------------------------------+ |
|  | + Specification                                            | |
|  |   Skill: [Requirements Analysis  ]                        | |
|  |   Deliverables: [ ] Scope document  [ ] API specification  | |
|  |   Deadline: [Jan 15, 2026]   Stake portion: [20%]          | |
|  +------------------------------------------------------------+ |
|  | + Planning                                                 | |
|  |   Skill: [System Architecture    ]                        | |
|  |   Deliverables: [ ] Architecture doc  [ ] Task breakdown   | |
|  |   Deadline: [Jan 30, 2026]   Stake portion: [20%]          | |
|  +------------------------------------------------------------+ |
|  | + Implementation                                           | |
|  |   Skill: [TypeScript Development ]                        | |
|  |   Milestones (payment gates):                              | |
|  |     M1: [Core Features   ] Deadline: [Feb 15] Tasks: [4]   | |
|  |     M2: [Integration     ] Deadline: [Mar 01] Tasks: [3]   | |
|  |     M3: [Production Ready] Deadline: [Mar 15] Tasks: [2]   | |
|  |   Deadline: [Mar 15, 2026]   Stake portion: [60%]          | |
|  |   Note: Customer reviews at milestone deadlines            | |
|  +------------------------------------------------------------+ |
|                                                                 |
|  VERIFICATION (Customer Acceptance)                             |
|  +------------------------------------------------------------+ |
|  |   Performed by: [You] or [Delegate: ____________]          | |
|  |   Criteria: [ ] Tests passing  [ ] Demo accepted           | |
|  |   Note: Verification determines implementation outcome      | |
|  +------------------------------------------------------------+ |
|                                                                 |
|  ELIGIBILITY REQUIREMENTS                                       |
|  Minimum trust: [50    ] cutes in primary skill                |
|  Minimum history: [10  ] contracts                             |
|  Minimum diversity: [5 ] unique counterparties                 |
|                                                                 |
|  ECONOMICS                                                      |
|  Total stake: [5,000 sats]                                     |
|  Escrow required: [+ Yes]                                      |
|                                                                 |
|  [Preview] [Save Draft] [Publish Contract Request]              |
+-----------------------------------------------------------------+
```

### Flow 2: Applying to a Contract (Proving Eligibility)

```
+-----------------------------------------------------------------+
|  APPLY TO CONTRACT                                              |
+-----------------------------------------------------------------+
|                                                                 |
|  Contract: Build Nostr Relay with Trust Filtering               |
|  Client: ######## (67.2 cutes)                                 |
|  Required: >=50 cutes in TypeScript Development                  |
|                                                                 |
|  YOUR ELIGIBILITY                                               |
|  +------------------------------------------------------------+ |
|  | TypeScript Development                                     | |
|  | Your trust: 78.3 cutes                                     | |
|  | Required:   50.0 cutes                                     | |
|  | Status: + ELIGIBLE                                         | |
|  |                                                            | |
|  | History checks:                                            | |
|  | - Contracts: 47 (required: 10) +                           | |
|  | - Unique counterparties: 12 (required: 5) +                | |
|  | - Outcome variance: 0.23 (required: 0.10) +                | |
|  +------------------------------------------------------------+ |
|                                                                 |
|  GENERATE ELIGIBILITY PROOF                                     |
|                                                                 |
|  This proof will verify:                                        |
|  - Your trust >= 50 cutes (without revealing exact value)        |
|  - Your history meets depth/diversity requirements              |
|  - Your history passes plausibility checks                      |
|                                                                 |
|  Privacy: Your contract history, counterparties, and exact      |
|  trust score will NOT be revealed to the client.                |
|                                                                 |
|  [Generate ZK Proof]                                            |
|                                                                 |
|  ***********************************************************   |
|                                                                 |
|  APPLICATION MESSAGE                                            |
|  +------------------------------------------------------------+ |
|  | I've built several Nostr relays including the strfry      | |
|  | plugin for ZK verification. Happy to share references...  | |
|  +------------------------------------------------------------+ |
|                                                                 |
|  [Cancel] [Submit Application with Proof]                       |
+-----------------------------------------------------------------+
```

### Flow 3: Phase Approval Workflow

*When a deliverable is submitted*

```
+-----------------------------------------------------------------+
|  PHASE REVIEW: Specification                                    |
+-----------------------------------------------------------------+
|                                                                 |
|  Contract: Build Nostr Relay with Trust Filtering               |
|  Provider: ########                                            |
|  Phase: Specification (15% of stake: 750 sats)                 |
|                                                                 |
|  SUBMITTED DELIVERABLES                                         |
|  +------------------------------------------------------------+ |
|  |  scope_document_v1.pdf                                   | |
|  |    Hash: bafybeig...xyz                                    | |
|  |    Submitted: Dec 28, 2025 14:32 UTC                       | |
|  |    [View Document]                                         | |
|  +------------------------------------------------------------+ |
|  |  api_specification.yaml                                  | |
|  |    Hash: bafybeih...abc                                    | |
|  |    Submitted: Dec 28, 2025 14:32 UTC                       | |
|  |    [View Document]                                         | |
|  +------------------------------------------------------------+ |
|                                                                 |
|  APPROVAL STATUS                                                |
|  Provider (########): + Approved                               |
|  Consumer (You):  Pending                                     |
|                                                                 |
|  YOUR ASSESSMENT                                                |
|  +------------------------------------------------------------+ |
|  | Outcome rating: [-1.0] -------------------- [+1.0]          | |
|  |                 Failure      0.85      Complete success    | |
|  |                                                            | |
|  | Comments (private):                                        | |
|  | +--------------------------------------------------------+ | |
|  | | Thorough spec, minor clarifications needed on auth...  | | |
|  | +--------------------------------------------------------+ | |
|  +------------------------------------------------------------+ |
|                                                                 |
|  Upon approval:                                                 |
|  - 750 sats released from escrow to provider                   |
|  - Phase outcome (+0.85) recorded to blockchain                |
|  - Provider earns ~2.3 cutes in Requirements Analysis          |
|  - Contract advances to Planning phase                          |
|                                                                 |
|  [Request Changes] [Approve Phase]                              |
+-----------------------------------------------------------------+
```

### Flow 4: Trust Dashboard (The New "Profile")

```
+-----------------------------------------------------------------+
|  MY TRUST DASHBOARD                                             |
+-----------------------------------------------------------------+
|                                                                 |
|  TRUST OVERVIEW                          Last updated: 5m ago  |
|  +------------------------------------------------------------+ |
|  |                                                            | |
|  |  TypeScript Dev  ########################..  87.2 cutes   | |
|  |  System Arch     ################..........  54.1 cutes   | |
|  |  Requirements    ############..............  38.7 cutes   | |
|  |  Proj Planning   ########..................  22.3 cutes   | |
|  |                                                            | |
|  |  Total weighted: 202.3 cutes across 4 skill domains        | |
|  +------------------------------------------------------------+ |
|                                                                 |
|  TRUST HISTORY                                                  |
|  +------------------------------------------------------------+ |
|  |       ^                                                    | |
|  |    90 |                                      ------          | |
|  |       |                              ----------'              | |
|  |    60 |                    ------------'                      | |
|  |       |          ------------'                                | |
|  |    30 |  ----------'                                          | |
|  |       |                                                    | |
|  |     0 +------------------------------------------------    | |
|  |       Jun  Jul  Aug  Sep  Oct  Nov  Dec                    | |
|  |                                                            | |
|  |  [TypeScript --] [Architecture --] [All Skills --]            | |
|  +------------------------------------------------------------+ |
|                                                                 |
|  ACTIVE CONTRACTS (3)                                           |
|  +------------------------------------------------------------+ |
|  | - Relay Trust Filtering | Phase: Implementation | 67%      | |
|  | - NIP-XX Proposal       | Phase: Specification  | 90%      | |
|  | - API Integration       | Phase: Planning       | 45%      | |
|  +------------------------------------------------------------+ |
|                                                                 |
|  RECENT TRUST CHANGES                                           |
|  - Dec 27: +4.2 cutes (TypeScript) -> Phase completed           |
|  - Dec 22: +3.1 cutes (Architecture) -> Contract completed      |
|  - Dec 18: -0.5 cutes (TypeScript) -> Partial outcome           |
|                                                                 |
|  [Export Proof] [View Full History] [Privacy Settings]          |
+-----------------------------------------------------------------+
```

---


### Flow 5: Team-Based Implementation (Task Roster)

When an Implementation phase has multiple providers via task decomposition, tasks are grouped into milestones:

```

  PROJECT: Build Nostr Relay                                         

                                                                     
  IMPLEMENTATION PHASE  Team Roster                                 
  Total Stake: 3,000 sats . Final Deadline: Mar 15                   
  Phase Difficulty: 7.2 (aggregate from tasks)                       
                                                                     
    
    MILESTONE 1: Core Relay    Deadline: Feb 01    Status: Complete  
    Stake: 1,200 sats                                                
       
    Task                    Provider   Difficulty  Weight  Status   
       
    WebSocket Handler       Alice         8         2.0    Complete 
      Stake: 800 sats . Outcome: +0.95                               
                                                                     
    Event Storage           Bob           7         1.0    Complete  
      Stake: 400 sats . Outcome: +0.92                               
       
    ✓ Customer accepted all tasks . Payment released                 
    
                                                                     
    MILESTONE 2: NIP Compliance    Deadline: Mar 01    Status: Active
    Stake: 1,000 sats                                                
       
    Task                    Provider   Difficulty  Weight  Status   
       
    NIP-01 Compliance       Alice         6         1.0    Complete 
      Stake: 500 sats . Outcome: +0.88                               
                                                                     
    NIP-09 Deletions        Bob           5         1.0    In Progress
      Stake: 500 sats . 67% complete                                 
       
    ⏳ Awaiting milestone deadline for customer review                
    
                                                                     
    MILESTONE 3: Production   Deadline: Mar 15    Status: Pending    
    Stake: 800 sats                                                  
       
    Task                    Provider   Difficulty  Weight  Status   
       
    Performance Tuning      Alice         7         1.0    Pending  
      Stake: 800 sats                                                
    
                                                                     
  TEAM TRUST CONTRIBUTIONS (projected)                               
    
    Alice (Tasks 1, 3, 5)                                             
      TypeScript: +9.4 cutes (from WebSocket, NIP-01, Perf)          
                                                                     
    Bob (Tasks 2, 4)                                                  
      TypeScript: +5.2 cutes (from Event Storage, NIP-09)            
    
                                                                     
  Planning Accuracy: 4/5 tasks as planned (80%)                       
     Architect earns Project Estimation trust                        
                                                                     

```

**Key team implementation properties:**
- Tasks are grouped into milestones (payment gates)
- Each milestone has a deadline computed from max(task deadlines)
- Customer reviews at milestone deadline, not per-task
- Non-disputed tasks in a milestone are paid immediately upon acceptance
- Disputed tasks enter arbitration (see ADR_Dispute_Resolution.md)
- Each task is a sub-subcontract with its own provider
- Each task has its own difficulty rating (assessed by the task provider at acceptance)
- Phase difficulty aggregates from task difficulties via stake-weighted average
- Each provider earns trust only from their assigned tasks
- Planning accuracy measured against original task breakdown

### Flow 6: Provider Acceptance (Difficulty Assessment)

When a provider considers accepting an Implementation contract, they review the task breakdown from the Planning phase and assess difficulty for each task.

```

  PROVIDER ACCEPTANCE                                                 
                                                                     
  Contract: Authentication System Implementation                      
  Customer: Acme Corp (78/100 trust)                                 
  Phase Stake: 5,000 sats . Deadline: 30 days                        
                                                                     
    
    TASK BREAKDOWN (from Planning phase)                            
                                                                    
    Task                          Stake    Your Difficulty Rating   
       
    T1: Database schema design     500        [ 3 ] ▼               
    T2: Password hashing impl      750        [ 4 ] ▼               
    T3: Session management       1,000        [ 5 ] ▼               
    T4: OAuth2 integration       1,250        [ 7 ] ▼               
    T5: Rate limiting              750        [ 4 ] ▼               
    T6: Security audit             750        [ 6 ] ▼               
                                                                    
    
                                                                     
  AGGREGATE DIFFICULTY: 5.05                                         
    (stake-weighted average of task difficulties)                    
                                                                     
  THRESHOLD: 32.4 cutes                                              
    Your TypeScript trust: 78.3 cutes ✓ Eligible                    
                                                                     
  [Request Task Refinement]  [Accept Contract]  [Decline]            

```

**Key elements:**
- Tasks come from Planning phase (customer's responsibility)
- Provider inputs difficulty (0-10) for each task based on their expertise
- Aggregate difficulty calculated automatically (stake-weighted average)
- Threshold updates based on aggregate difficulty
- Provider can request refinement if tasks are too vague

### Flow 7: Task Refinement Request

If a provider identifies tasks that are too vague or too broad to assess accurately, they can request refinement before accepting.

```

  TASK REFINEMENT REQUEST                                            
                                                                     
  Contract: Authentication System Implementation                      
  To: Acme Corp                                                      
                                                                     
    
    TASKS REQUIRING REFINEMENT                                      
                                                                    
    T4: OAuth2 integration (1,250 sats)                             
                                                                    
    Provider's concern:                                              
    ┌─────────────────────────────────────────────────────────────┐ 
    │ This task is too broad to assess accurately. OAuth2 has     │ 
    │ many variations. I need this broken into subtasks:          │ 
    │                                                             │ 
    │ Suggested breakdown:                                        │ 
    │ • T4a: OAuth2 flow for Google                               │ 
    │ • T4b: OAuth2 flow for GitHub                               │ 
    │ • T4c: Token refresh handling                               │ 
    │ • T4d: Error handling and edge cases                        │ 
    └─────────────────────────────────────────────────────────────┘ 
                                                                    
    
                                                                     
  [Send Refinement Request]  [Cancel]                                

```

**What happens next:**
- Customer receives request with provider's suggested breakdown
- Customer can accept suggestions, propose alternatives, or negotiate as-is
- Contract remains pending until refinement is resolved
- Provider can decline if customer refuses reasonable refinement

### Flow 8: Task Refinement Response (Customer)

Customer responds to provider's refinement request.

```

  TASK REFINEMENT RESPONSE                                           
                                                                     
  Contract: Authentication System Implementation                      
  From: DevAlice (Provider)                                          
                                                                     
    
    REFINEMENT REQUEST                                              
                                                                    
    Task: T4: OAuth2 integration (1,250 sats)                       
                                                                    
    Provider requested breakdown into:                               
    • T4a: OAuth2 flow for Google                                   
    • T4b: OAuth2 flow for GitHub                                   
    • T4c: Token refresh handling                                   
    • T4d: Error handling and edge cases                            
                                                                    
    
                                                                     
  YOUR RESPONSE                                                      
                                                                     
  ( ) Accept suggested breakdown                                     
      └─ Reallocate 1,250 sats across 4 subtasks                    
                                                                     
  ( ) Propose alternative breakdown                                  
      └─ [Define your own subtasks]                                 
                                                                     
  ( ) Keep task as-is                                                
      └─ Provider may decline the contract                          
                                                                     
  [Send Response]                                                    

```

**Outcomes:**
- **Accept**: Planning phase amended with subtasks; provider re-reviews
- **Alternative**: Counter-proposal sent to provider for review
- **Keep as-is**: Provider decides whether to accept the risk or decline

### Flow 9: Contract Amendment (Mid-Contract)

During implementation, discovered complexity requires adding new tasks.

```

  CONTRACT AMENDMENT                                                  
                                                                     
  Contract: Authentication System Implementation                      
  Status: In Progress (4/6 tasks complete)                           
                                                                     
    
    PROPOSED AMENDMENT                                              
                                                                    
    Reason: Discovered complexity                                    
    ┌─────────────────────────────────────────────────────────────┐ 
    │ SAML integration required for enterprise SSO support.       │ 
    │ This was not in original scope but is necessary for the     │ 
    │ authentication system to work with customer's IdP.          │ 
    └─────────────────────────────────────────────────────────────┘ 
                                                                    
    NEW TASKS                                                        
                                                                    
    Task                       Stake    Difficulty    Provider      
       
    T7: SAML assertion parsing   400        5         DevAlice     
    T8: IdP metadata handling    350        4         DevAlice     
                                                                    
    Additional stake required: 750 sats                              
    
                                                                     
  IMPACT                                                             
                                                                     
    Original phase difficulty: 5.05                                  
    Amended phase difficulty:  5.12                                  
                                                                     
    Planning accuracy impact: Will show 8 total tasks vs 6 planned   
      (Architect's planning accuracy affected)                       
                                                                     
  [Approve Amendment]  [Reject]  [Negotiate]                         

```

**Key properties:**
- New tasks are new sub-subcontracts with their own difficulty ratings
- Original tasks retain original ratings
- Phase aggregate recalculates to include new tasks
- Planning accuracy tracked (affects architect's trust if they did planning)
- Both parties must approve amendment

### Flow 10: Milestone Review (Customer)

Customer reviews completed tasks at milestone deadline. This is the payment gate.

```

  MILESTONE REVIEW                                                    
                                                                     
  Contract: Authentication System Implementation                      
  Milestone: M2 - Third-Party Integration                            
  Deadline: Feb 15, 2026 (today)                                     
  Milestone Stake: 2,750 sats                                        
                                                                     
    
    COMPLETED TASKS IN THIS MILESTONE                                
                                                                    
    Task                       Provider    Stake    Outcome          
       
    T4: OAuth2 integration     DevAlice   1,250    +0.92  ✓         
        Tests passing, demo accepted                                 
                                                                    
    T5: Rate limiting          DevAlice     750    +0.85  ✓         
        Implemented as specified                                     
                                                                    
    T6: Security audit         DevBob       750    +0.78  ?         
        Minor concerns about token handling                          
                                                                    
    
                                                                     
  YOUR OPTIONS                                                       
                                                                     
  (●) Accept all tasks                                               
      └─ Release 2,750 sats to providers                            
      └─ Trust updates: DevAlice +4.2, DevBob +1.8                  
                                                                     
  ( ) Dispute specific tasks                                         
      └─ Non-disputed tasks paid immediately                        
      └─ Disputed tasks enter arbitration                           
      └─ [Select tasks to dispute]                                  
                                                                     
  ( ) Let deadline pass (timeout)                                    
      └─ All tasks automatically accepted                           
      └─ Payment released, trust updated                            
                                                                     
  Time remaining: 23:45:12                                           
                                                                     
  [Accept All]  [Dispute Selected]                                   

```

**Key milestone review properties:**
- Customer reviews at milestone deadline, not per-task during work
- Three options: accept all, dispute specific tasks, or timeout
- Non-disputed tasks are paid immediately upon decision
- Disputed tasks enter Tier-1 arbitration (single arbitrator)
- Timeout = acceptance (customer forfeits review opportunity)
- Trust updates happen when payment is released
- See **ADR_Milestone_Payment_Gates.md** and **ADR_Dispute_Resolution.md**

### Flow 11: Task Dispute (Partial Milestone)

When customer disputes specific tasks within a milestone:

```

  TASK DISPUTE                                                        
                                                                     
  Contract: Authentication System Implementation                      
  Milestone: M2 - Third-Party Integration                            
                                                                     
    
    NON-DISPUTED TASKS (payment releasing)                           
                                                                    
    Task                       Provider    Stake    Status           
       
    T4: OAuth2 integration     DevAlice   1,250    Paid ✓           
    T5: Rate limiting          DevAlice     750    Paid ✓           
                                                                    
    Total released: 2,000 sats                                       
    
                                                                     
    DISPUTED TASK                                                    
                                                                    
    Task: T6 - Security audit                                        
    Provider: DevBob                                                 
    Stake: 750 sats (held in escrow)                                 
                                                                    
    Your evidence:                                                   
    ┌─────────────────────────────────────────────────────────────┐ 
    │ Token refresh implementation has race condition that could  │ 
    │ lead to session hijacking. Audit report did not identify    │ 
    │ this critical vulnerability. See attached PoC exploit.      │ 
    └─────────────────────────────────────────────────────────────┘ 
                                                                    
    Evidence hash: bafybeig...xyz                                    
    
                                                                     
  DISPUTE PROCESS                                                    
                                                                     
    1. Provider (DevBob) submits counter-evidence                    
    2. Both parties propose arbitrators                              
    3. First mutually-acceptable arbitrator selected                 
    4. Arbitrator reviews and sets payout percentage                 
    5. Loser pays 5% arbitration fee                                 
    6. Appeal available within 72 hours (15% fee, 3-arbitrator panel)
                                                                     
  [Submit Dispute]  [Cancel (Accept Task)]                           

```

### Flow 12: Provider Calibration View

Providers can view their difficulty estimation accuracy over time.

```

  PROVIDER CALIBRATION                                               
                                                                     
  DevAlice's Estimation History                                      
                                                                     
    
    CALIBRATION SCORE: 82/100                                       
                                                                    
    "Your difficulty estimates are generally accurate.              
     You slightly underestimate complex integration tasks."          
                                                                    
    
                                                                     
  ESTIMATION ACCURACY BY OUTCOME                                     
                                                                     
    On-time, good outcome     ████████████████████  67%  Accurate   
    Late, good outcome        ████████              27%  Underest.  
    Early, trivial effort     ██                     6%  Overest.   
    Failed despite trust      ░                      0%  Severe     
                                                                     
    
    ACCURACY BY TASK TYPE                                           
                                                                    
    Task Category          Est. Avg   Actual Avg   Calibration     
       
    Database work             4.2        4.0         +0.2  ●        
    API integration           5.5        6.8         -1.3  ◐        
    UI components             3.8        3.5         +0.3  ●        
    Security/auth             6.2        7.1         -0.9  ◐        
                                                                    
    ● Well-calibrated  ◐ Tends to underestimate  ○ Tends to overest.
    
                                                                     
  RECENT ESTIMATES                                                   
                                                                     
    Task                    Your Est.  Actual   Outcome              
       
    OAuth2 Google flow          6        6      On-time ✓           
    SAML parsing                5        7      Late (good) ◐       
    Session management          5        5      On-time ✓           
    Rate limiting               4        4      Early ●             
                                                                     
  Calibration affects how customers evaluate your estimates.         
  Well-calibrated providers are more attractive for complex work.    

```

**Why this matters:**
- Providers known for good calibration become more attractive to customers
- Creates incentive for honest, accurate estimation
- Tracks patterns (e.g., "tends to underestimate security work")
- Could become a tracked skill type: `V_calibration(provider)`

---

## Part Five: New Dimensions Uncovered

### 1. Planning Accuracy as Distinct Skill

The mutual sign-off on Planning phase commits a task baseline. Implementation then measures against it:

```
planning_accuracy(c) = tasks_completed_as_planned / planned_task_count
```

This creates a distinct trust dimension:

| Skill Domain | What It Measures |
|--------------|------------------|
| TypeScript Development | Can you write working code |
| System Architecture | Can you design sound systems |
| **Project Planning** | Can you accurately predict what's needed |

An Avatar might have:
- 85 cutes in TypeScript (excellent implementer)
- 32 cutes in Project Planning (poor estimator)

This is valuable signal. Some projects need predictability more than brilliance.

### 2. Phase-Specific Skill Types

Should each phase contribute to different skill types

**Option A: Single Skill per Contract**
- Contract is "TypeScript Development"
- All phases contribute to TypeScript trust
- Simple but loses granularity

**Option B: Skill per Phase** (Recommended)
- Specification -> Requirements Analysis trust
- Planning -> System Architecture trust
- Implementation -> TypeScript Development trust
- (Verification is customer acceptance, not a provider skill)
- Richer signal, more accurate trust attribution

**Option C: Hybrid**
- Primary skill (TypeScript) gets major contribution
- Phase skills get minor contributions
- Balances simplicity and granularity

### 3. Multi-Party Contracts

What about team projects

```
Team contract:
  providers: [Alice, Bob, Carol]
  consumer: ClientDAO
  phase_assignments: {
    Alice: {specification},
    Bob: {planning},
    Carol: {implementation}
  }
  verification: performed by ClientDAO (or delegate)
  outcome_attribution: each provider earns trust for their phase
```

Trust attribution:
- Each provider earns trust for phases they delivered
- Verification is customer's responsibility, not attributed to providers
- If multiple providers share a phase, outcome could be split or peer-assessed

### 4. Dispute Resolution

When consumer and provider disagree on outcome:

**Current model**: Single outcome value, unclear how disputes resolve.

**Enhanced model**:
```
dispute: {
  claimed_outcome_provider: +0.9,
  claimed_outcome_consumer: -0.3,
  mediator: ######## (DAO or trusted third party),
  mediated_outcome: +0.4,
  evidence: [
    {type: "artifact", hash: "bafybeig..."},
    {type: "communication_log", hash: "bafybeih..."},
    {type: "mediator_analysis", content: "..."}
  ]
}
```

Mediated disputes could:
- Use DAO governance for resolution
- Stake a portion to mediator as incentive
- Record dispute history as trust signal (too many disputes = red flag)

### 5. Recurring Relationships

Some professional relationships are ongoing:
- Retainer arrangements
- Subscription services
- Long-term collaborations

Could model as:
```
recurring_contract: {
  base_contract: {...},
  period: "monthly",
  renewal: "auto" | "approval_required",
  cumulative_outcomes: [...],
  relationship_trust_bonus: applied after N successful renewals
}
```

### 6. Publication Consent (Privacy Control)

Contract details are private by default. For any contract information to be publicly visible, **both parties must consent**:

```
contract_disclosure: {
  contract_hash: "...",
  disclosed_by: [a_provider, a_consumer],  // Both signatures required
  disclosure_scope: "full" | "outcome_only" | "existence_only",
  disclosed_at: timestamp
}
```

**Design decision**: Publication consent is recorded but does **not** affect trust weight.

| Considered | Rejected Because |
|------------|------------------|
| Public contracts weighted higher | Creates pressure/coercion to publish |
| Private-only contracts | Prevents legitimate portfolio building |

Publication consent follows the same mutual sign-off pattern as phase completion--it's verifiable that both parties agreed. But it's a *privacy control*, not a *trust input*.

**Disclosure scopes**:
- `existence_only`: "We worked together" (no details)
- `outcome_only`: Skill type + outcome value (no counterparty identity)
- `full`: Complete contract details visible

This enables selective portfolio building while protecting counterparties who prefer privacy.

### 7. Portfolio/Evidence (Optional Metadata)

Artifact hashes can be attached to phase sign-offs for external verification, but this is **metadata, not trust input**:

```
phase_signoff: {
  outcome: +0.92,
  mutually_signed: true,
  completed_at: timestamp,
  
  // Optional metadata (not used in trust calculation)
  artifact_references: [
    { type: "document_hash", value: "bafybeig..." },
    { type: "repo_commit", value: "a1b2c3d..." }
  ]
}
```

The trust system doesn't verify artifacts--it trusts the mutual sign-off. If both parties agreed the deliverable was +0.92 quality, that's the signal. Artifacts exist for human due diligence, not framework verification.

### 8. Skill Type Taxonomy

Who defines valid skill types

**Option A: Open taxonomy** (anyone can create)
- Flexible but fragmented
- "TypeScript Development" vs "typescript_dev" vs "TS coding"

**Option B: Curated taxonomy** (governance-controlled)
- Consistent but centralized
- NIP proposal for standard skill types

**Option C: Hierarchical with aliases**
```
skill_taxonomy: {
  "software_development": {
    "typescript": ["ts", "typescript_development", "typescript_dev"],
    "rust": ["rust_development", "rust_programming"],
    ...
  },
  "design": {
    "ui_ux": ["ux", "ui_design", "user_experience"],
    ...
  }
}
```

---

## Part Six: Implementation Priorities

### Phase 1: Multi-Phase Trust Attribution
1. Extend contract structure for phase-level outcomes
2. Implement phase skill type mapping
3. Add mutual sign-off verification to circuits
4. Compute planning accuracy from committed baseline
5. Update trust calculation: `V_t +=  (phase) . outcome(phase)`

### Phase 2: Privacy Controls
1. Publication consent event schema
2. Mutual sign-off for disclosure (same pattern as phase completion)
3. Disclosure scope levels (existence / outcome / full)
4. Private dashboard for Avatar owner

### Phase 3: Professional Network UX
1. Trust dashboard with per-skill visualization
2. Contract marketplace with eligibility filtering
3. Phase approval workflow (mutual sign-off UI)
4. ZK eligibility proof generation
5. Published Work section (mutually-consented contracts only)

### Phase 4: Organizational Trust
1. DAO profiles with aggregate trust
2. Multi-party contracts (team attribution)
3. Role-based outcome weighting

### Phase 5: Ecosystem Integration
1. Nostr event schemas for phase transitions
2. Nostr event schema for publication consent
3. NIP proposal for QoT contract events
4. Relay-side ZK verification (per existing spec)

---

## Part Seven: Nostria Integration Architecture

### Design Principle: Separate Pages

QoT functionality lives on **separate pages** from existing Nostria features, not as modifications to existing pages. This approach:

- **Preserves existing UX**: Users familiar with Nostria see no changes to their current experience
- **Enables independent evolution**: QoT workflows can evolve without affecting core social features
- **Maintains clear context**: Professional/contract context is distinct from social context
- **Simplifies development**: New pages can be built without modifying existing codebase

QoT pages may borrow UI conventions (styling, navigation patterns, component designs) to maintain visual continuity, but they are architecturally separate.

### What Nostria Already Has

Existing pages (unchanged):
- Profile with npub, NIP-05, Lightning
- Posts/notes feed
- Media sharing
- Follow graph
- Direct messages

### New QoT Pages

Separate pages to be added:

| Page | Purpose |
|------|---------|
| **Trust Dashboard** | View own trust scores per skill, contract history, verification status |
| **Avatar Public Profile** | Public view of skill-scoped trust (aggregate only, privacy-preserving) |
| **Contract Marketplace** | Browse available contracts, filter by eligibility |
| **Contract Detail** | View contract terms, accept/negotiate, track progress |
| **Provider Acceptance** | Review tasks, assess difficulty, request refinement, accept contract |
| **Task Refinement Request** | Provider requests customer break down vague/broad tasks |
| **Task Refinement Response** | Customer responds with refined breakdown or negotiates as-is |
| **Contract Amendment** | Mid-contract: add new tasks with difficulty ratings for discovered complexity |
| **Milestone Review** | Customer reviews completed tasks at milestone deadline: accept, dispute, or timeout |
| **Task Dispute** | Customer disputes specific task(s); submit evidence, select arbitrator |
| **Arbitration View** | Arbitrator reviews evidence, sets payout percentage |
| **Phase Approval** | Mutual sign-off workflow for phase completion |
| **Project View** | Multi-phase project coordination, milestone tracking, task assignments |
| **DAO Profile** | Organizational trust aggregation, member roster |
| **Eligibility Proof** | Generate and share ZK proofs for specific thresholds |
| **Customer Dashboard** | Customer-specific metrics (for providers evaluating customers) |
| **Provider Calibration View** | Track provider's estimation accuracy history |

### Integration Points

QoT pages connect to Nostria through:
- **Shared identity**: Same npub/keypair, NIP-05 verification
- **Lightning integration**: Zaps infrastructure for escrow funding
- **DMs (NIP-17)**: Private contract negotiation
- **Navigation**: QoT section in main navigation
- **Visual consistency**: Shared design language

---

## Conclusion

The vision of extending Nostria with QoT professional network pages is architecturally coherent. The key insight is that LinkedIn's value—professional reputation enabling opportunity access—can be delivered *better* with QoT:

| LinkedIn | QoT Professional Network |
|----------|---------------------------|
| Self-claimed work history | Verified contract outcomes |
| Peer endorsements (gameable) | No endorsements--contract outcomes are sufficient |
| Binary job applications | ZK eligibility proofs |
| Company pages | DAOs with aggregate trust |
| Identity-bound reputation | Avatar-bound, privacy-preserving |
| Public work history by default | Mutual consent for publication |

**The critical design principles**:

1. **Outcomes only**: QoT captures *trust-relevant outcomes*, not operational details. Phase outcomes and planning accuracy flow into trust. Task descriptions do not.

2. **Mutual sign-off prevents gaming**: The adversarial dynamic between parties provides validation--no external complexity metrics needed.

3. **Publication requires consent**: Contract details are private by default. Both parties must agree before any contract information becomes public. This is recorded but not trust-weighted (to avoid coercion incentives).

4. **Planning accuracy as distinct skill**: An Avatar who consistently delivers against mutually-committed plans has demonstrated predictability, valuable independent of implementation quality.

Next steps: formalize the phase-level trust attribution in the mathematical equations, extend the Noir circuits for mutual sign-off verification, design Nostr event schemas for phase transitions and publication consent.

---

## QoT Professional Network Differentiators

Key design decisions that distinguish QoT from traditional professional networks:

| Decision | Rationale |
|----------|-----------|
| **Multi-provider contracts** | Different Avatars can perform different phases. Each earns trust for their contribution independently. Enables specialist collaboration. |
| **Verification is acceptance, not work** | The customer (or delegate) performs verification. This determines implementation outcome. Verification is not a trust-earning phase like provider work, but customers earn trust from verification integrity (how well they rate). |
| **QA earns trust via separate contracts** | Quality Assurance is a valid skill type. QA professionals contract directly with customers to provide testing services. They earn QA trust from that contract, not as a phase within the build contract. |
| **Three trust-earning phases** | Specification, Planning, Implementation. Each has a provider who earns trust. Verification is the customer's acceptance mechanism. |
| **No endorsements** | Contract outcomes ARE attestations. Separate endorsements are either redundant (counterparty-only) or gameable (open to anyone). The mutual sign-off on phase completion is the verification. |
| **Publication requires mutual consent** | Contract details are private by default. Both parties must agree before any information becomes public. Recorded but not trust-weighted to avoid coercion. |
| **Outcomes only, not operational details** | QoT captures trust-relevant outcomes (phase quality, planning accuracy), not project management data (task descriptions, documents, timelines). |
| **Mutual sign-off prevents gaming** | The adversarial dynamic between provider and consumer provides validation. No external complexity metrics needed. |
| **Planning accuracy as distinct skill** | Task completion ratio against mutually-committed baseline creates a separate trust dimension: predictability independent of implementation quality. |
| **Aggregate trust public, contract details private** | Profile shows skill scores and contract counts. Individual contract details (counterparty, timing, stakes) require mutual publication consent. |
| **Counterparty trust already captured in weighting** | Contract weight includes (c) counterparty factor. No need for additional "who you worked with" reputation signal. |
| **Negative trust is permanent signal** | Failed contracts compute into trust and cannot be deleted. This is what makes QoT trust meaningful--you can't curate your history. |
| **Bidirectional trust** | Customers earn trust from behavior (commitment, escrow discipline, verification integrity, scope stability). Providers can evaluate customers before accepting contracts. |
| **Verification weight** | Customer credibility affects how much their ratings count. Rubber-stamping (low variance) or erratic (high variance) customers have reduced influence. |
| **Team-based implementation** | Tasks within milestones can have different providers. Each team member earns trust from their assigned tasks independently. Enables specialist collaboration at the task level. |
| **Customer trust dashboard** | Providers see customer metrics (commitment rate, escrow discipline, scope stability) before accepting contracts. Creates informed matching. |
| **Task-level difficulty assessment** | Difficulty is assessed at the task level by the provider at acceptance, not set by the customer. Phase difficulty aggregates from task difficulties. Both parties have incentive for accuracy—incorrect ratings lead to failed tasks affecting both. |
| **Milestone-based payment gates** | Tasks are grouped into milestones within Implementation. Customer reviews at milestone deadline (not per-task): accept all, dispute specific tasks, or timeout. Reduces provider risk via incremental payment. |
| **Deadline-based dispute resolution** | Customer disputes at milestone review trigger arbitration. Non-disputed tasks paid immediately. Tiered system: Tier-1 (single arbitrator, 5% fee), Tier-2 appeal (3-arbitrator panel, 15% fee). Deadlocked disputes refund all parties. |
| **Trust flows through tasks, not milestones** | Milestones are coordination containers for payment. Trust attribution happens at the task level. This keeps the math simple while enabling incremental payment. |

---

## Related Documents

- **ADR_Milestone_Payment_Gates.md** — Milestone-based payment model and customer review workflow
- **ADR_Dispute_Resolution.md** — Deadline-based dispute resolution with tiered arbitration
- **The_Difficulty_of_Assessing_Difficulty.md** — How difficulty ratings are determined at the task level
- **ADR_Subcontract_Architecture.md** — Multi-phase contract decomposition
- **ADR_No_Endorsements.md** — Why contract outcomes replace attestations
- **Quantum_of_Trust_Equations_in_CSharp.md** — Implementation with Task, MilestoneWithTasks, PhaseWithTasks, CustomerProfile
- **Quantum_of_Trust_Equations_in_Noir.md** — ZK circuit implementation
