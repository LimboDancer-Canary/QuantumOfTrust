# QoT Nostria Component Templates

*Angular implementation details for QoT integration with Nostria*

**Version:** 1.0  
**Date:** January 1, 2026

---

## Overview

This document contains Nostria-specific implementation details including Angular component templates, service implementations, and integration patterns. For architectural specifications, see **QoT_Nostria_Client_Specification.md**.

---

## 1. Nostria Stack Compatibility

### 1.1 Technology Stack

| Nostria Component | QoT Requirement | Compatibility |
|-------------------|-----------------|---------------|
| Angular 21 + Signals | Reactive trust score updates | ✅ Excellent |
| Angular Material | Professional UI components | ✅ Excellent |
| IndexedDB (via `idb`) | Local trust score caching | ✅ Excellent |
| nostr-tools | Nostr event creation/signing | ✅ Excellent |
| Service Worker/PWA | Offline trust verification | ✅ Excellent |
| Tauri (desktop) | Desktop ZK proving | ✅ Excellent |
| NIP-07/NIP-46 signing | Avatar keypair management | ✅ Excellent |

### 1.2 New Dependencies

| Package | Purpose | Size |
|---------|---------|------|
| `@aztec/aztec.js` | Blockchain RPC client | ~200KB |
| `@qot/circuits-wasm` | Trust computation (compiled from Noir) | ~500KB |
| `@aztec/bb.js` | Barretenberg proving backend | ~2MB WASM |
| `@noir-lang/noir_js` | Noir circuit execution | ~300KB |

### 1.3 Existing Service Integration Points

| Service | Purpose | QoT Integration Point |
|---------|---------|----------------------|
| `NostrService` | Core protocol operations | Event publishing for contracts, proofs |
| `AccountStateService` | User state management | Avatar identity, trust score signals |
| `DatabaseService` | IndexedDB wrapper | Contract/trust score caching |
| `RelayPoolService` | Connection management | QoT-specific relay connections |
| `LayoutService` | Responsive UI | QoT page layouts |
| `PublishService` | Event publishing | Contract/proof publication |

### 1.4 Routing Pattern

Nostria uses lazy-loaded routes:

```typescript
{
  path: 'feature',
  loadComponent: () => import('./pages/feature/feature.component').then(m => m.FeatureComponent)
}
```

QoT follows the same pattern under `/qot/*` namespace.

---

## 2. Component Templates

### 2.1 Trust Dashboard Component

```typescript
@Component({
  selector: 'app-qot-dashboard',
  standalone: true,
  imports: [CommonModule, MatCardModule, MatProgressBarModule, RouterLink],
  template: `
    <div class="dashboard-container">
      <section class="trust-scores">
        <h2 i18n>Verified Capabilities</h2>
        @for (score of trustScores(); track score.domain) {
          <app-trust-score-card 
            [score]="score"
            [showProveButton]="true"
            (prove)="navigateToProve(score.domain)" />
        }
      </section>
      
      <section class="contract-history">
        <h2 i18n>Contract History (Private)</h2>
        <app-contract-history [contracts]="contracts()" />
      </section>
      
      <section class="customer-metrics">
        <h2 i18n>My Customer Profile</h2>
        @if (customerProfile(); as profile) {
          <app-customer-metrics [profile]="profile" />
        }
      </section>
      
      <section class="pending-reviews">
        <h2 i18n>Pending Milestone Reviews</h2>
        @for (milestone of pendingMilestones(); track milestone.id) {
          <app-milestone-review-card
            [milestone]="milestone"
            (review)="navigateToMilestoneReview(milestone)" />
        }
        @empty {
          <p class="empty-state" i18n>No milestones awaiting review</p>
        }
      </section>
    </div>
  `
})
export class QotDashboardComponent {
  private qot = inject(QotService);
  private contractState = inject(ContractStateService);
  private router = inject(Router);
  
  trustScores = this.qot.myTrustScores;
  contracts = this.qot.myContractHistory;
  customerProfile = this.qot.customerProfile;
  pendingMilestones = this.contractState.pendingMilestoneReviews;
  
  navigateToProve(domain: string) {
    this.router.navigate(['/qot/prove', domain]);
  }
  
  navigateToMilestoneReview(milestone: Milestone) {
    this.router.navigate(['/qot/contract', milestone.contractId, 'milestone', milestone.id]);
  }
}
```

### 2.2 Eligibility Proof Component

```typescript
@Component({
  selector: 'app-eligibility-proof',
  standalone: true,
  imports: [CommonModule, MatCardModule, MatSliderModule, MatButtonModule],
  template: `
    <mat-card class="proof-card">
      <mat-card-header>
        <mat-card-title i18n>Generate Eligibility Proof</mat-card-title>
        <mat-card-subtitle>{{ domain() }}</mat-card-subtitle>
      </mat-card-header>
      
      <mat-card-content>
        <div class="threshold-selector">
          <label i18n>Minimum threshold to prove:</label>
          <mat-slider min="0" max="100" step="25" [value]="threshold()"
            (valueChange)="threshold.set($event)" />
          <span class="threshold-value">≥ {{ threshold() }} cutes</span>
        </div>
        
        @if (currentScore(); as score) {
          <div class="current-score">
            <span i18n>Your current score: {{ score.value | number:'1.1-1' }} cutes</span>
            @if (score.value >= threshold()) {
              <mat-icon color="primary">check_circle</mat-icon>
            } @else {
              <mat-icon color="warn">warning</mat-icon>
              <span class="warning" i18n>Score below threshold</span>
            }
          </div>
        }
        
        @if (isProving()) {
          <div class="proving-status">
            <mat-spinner diameter="24" />
            <span i18n>Generating proof... (~5 seconds)</span>
          </div>
        }
      </mat-card-content>
      
      <mat-card-actions>
        <button mat-raised-button color="primary"
          [disabled]="isProving() || !canProve()"
          (click)="generateProof()">
          <mat-icon>verified</mat-icon>
          <span i18n>Generate Proof</span>
        </button>
      </mat-card-actions>
    </mat-card>
    
    @if (generatedProof(); as proof) {
      <mat-card class="proof-result">
        <mat-card-header>
          <mat-card-title i18n>Proof Generated</mat-card-title>
        </mat-card-header>
        <mat-card-content>
          <p i18n>Your eligibility proof has been published. Share this link:</p>
          <code>nostr:{{ proof.nevent }}</code>
          <button mat-icon-button (click)="copyProofLink()">
            <mat-icon>content_copy</mat-icon>
          </button>
        </mat-card-content>
      </mat-card>
    }
  `
})
export class EligibilityProofComponent {
  private route = inject(ActivatedRoute);
  private qot = inject(QotService);
  private zkService = inject(ZkProvingService);
  private clipboard = inject(Clipboard);
  
  domain = signal('');
  threshold = signal(50);
  currentScore = signal<TrustScore | null>(null);
  isProving = this.zkService.isProving;
  generatedProof = signal<ProofResult | null>(null);
  
  constructor() {
    effect(() => {
      const params = this.route.snapshot.paramMap;
      this.domain.set(params.get('domain') ?? '');
    });
  }
  
  canProve = computed(() => {
    const score = this.currentScore();
    return score && score.value >= this.threshold();
  });
  
  async generateProof() {
    const proof = await this.qot.generateEligibilityProof(
      this.domain(),
      this.threshold()
    );
    this.generatedProof.set(proof);
  }
  
  copyProofLink() {
    const proof = this.generatedProof();
    if (proof) {
      this.clipboard.copy(`nostr:${proof.nevent}`);
    }
  }
}
```

### 2.3 Milestone Review Component

```typescript
@Component({
  selector: 'app-milestone-review',
  standalone: true,
  imports: [CommonModule, MatCardModule, MatButtonModule, MatDialogModule],
  template: `
    <div class="milestone-review-container">
      <mat-card class="milestone-header">
        <mat-card-header>
          <mat-card-title i18n>Milestone Review</mat-card-title>
          <mat-card-subtitle>
            Deadline: {{ milestoneDeadline() | date:'medium' }}
            @if (isOverdue()) {
              <span class="overdue-warning" i18n>(Overdue - auto-accept in {{ hoursRemaining() }}h)</span>
            }
          </mat-card-subtitle>
        </mat-card-header>
      </mat-card>
      
      <section class="task-list">
        <h3 i18n>Tasks in this Milestone</h3>
        @for (task of tasks(); track task.id) {
          <mat-card class="task-card" [class.disputed]="task.status === 'disputed'">
            <mat-card-header>
              <mat-card-title>{{ task.description }}</mat-card-title>
              <mat-card-subtitle>
                Provider: {{ task.providerName }} | 
                Difficulty: {{ task.difficulty }}/10 |
                Weight: {{ task.weight }}%
              </mat-card-subtitle>
            </mat-card-header>
            
            <mat-card-content>
              @if (task.outcome !== null) {
                <div class="outcome-display">
                  <span i18n>Outcome: {{ task.outcome | number:'1.0-0' }}%</span>
                  <mat-progress-bar 
                    mode="determinate" 
                    [value]="(task.outcome + 100) / 2" />
                </div>
              }
            </mat-card-content>
            
            <mat-card-actions>
              @if (task.status === 'pending_review') {
                <button mat-button color="primary" (click)="acceptTask(task)">
                  <mat-icon>check</mat-icon>
                  <span i18n>Accept</span>
                </button>
                <button mat-button color="warn" (click)="openDisputeDialog(task)">
                  <mat-icon>report_problem</mat-icon>
                  <span i18n>Dispute</span>
                </button>
              } @else if (task.status === 'accepted') {
                <span class="status-badge accepted" i18n>Accepted</span>
              } @else if (task.status === 'disputed') {
                <span class="status-badge disputed" i18n>Disputed</span>
              }
            </mat-card-actions>
          </mat-card>
        }
      </section>
      
      <div class="review-actions">
        <button mat-raised-button color="primary"
          [disabled]="!canAcceptAll()"
          (click)="acceptAllTasks()">
          <mat-icon>done_all</mat-icon>
          <span i18n>Accept All Tasks</span>
        </button>
        
        <button mat-raised-button
          [disabled]="!hasReviewedAll()"
          (click)="submitReview()">
          <mat-icon>send</mat-icon>
          <span i18n>Submit Review</span>
        </button>
      </div>
    </div>
  `
})
export class MilestoneReviewComponent {
  private route = inject(ActivatedRoute);
  private qot = inject(QotService);
  private dialog = inject(MatDialog);
  
  contractId = signal('');
  milestoneId = signal('');
  tasks = signal<Task[]>([]);
  milestoneDeadline = signal<Date | null>(null);
  
  isOverdue = computed(() => {
    const deadline = this.milestoneDeadline();
    return deadline && deadline < new Date();
  });
  
  hoursRemaining = computed(() => {
    const deadline = this.milestoneDeadline();
    if (!deadline) return 0;
    const autoAcceptDeadline = new Date(deadline.getTime() + 7 * 24 * 60 * 60 * 1000);
    return Math.max(0, Math.floor((autoAcceptDeadline.getTime() - Date.now()) / (60 * 60 * 1000)));
  });
  
  canAcceptAll = computed(() => {
    return this.tasks().every(t => t.status === 'pending_review');
  });
  
  hasReviewedAll = computed(() => {
    return this.tasks().every(t => t.status !== 'pending_review');
  });
  
  async acceptTask(task: Task) {
    await this.qot.acceptTask(this.contractId(), task.id);
    this.refreshTasks();
  }
  
  async acceptAllTasks() {
    await this.qot.submitMilestoneReview(
      this.contractId(), 
      this.milestoneId(), 
      { acceptAll: true }
    );
    this.refreshTasks();
  }
  
  openDisputeDialog(task: Task) {
    const dialogRef = this.dialog.open(DisputeDialogComponent, {
      data: { task, contractId: this.contractId() }
    });
    
    dialogRef.afterClosed().subscribe(result => {
      if (result) {
        this.refreshTasks();
      }
    });
  }
  
  async submitReview() {
    // Final submission triggers payment release for accepted tasks
    await this.qot.finalizeMilestoneReview(this.contractId(), this.milestoneId());
  }
  
  private async refreshTasks() {
    const state = await this.qot.getMilestoneState(this.contractId(), this.milestoneId());
    this.tasks.set(state.tasks);
  }
}
```

### 2.4 Dispute Dialog Component

```typescript
@Component({
  selector: 'app-dispute-dialog',
  standalone: true,
  imports: [CommonModule, MatDialogModule, MatButtonModule, MatInputModule, FormsModule],
  template: `
    <h2 mat-dialog-title i18n>Dispute Task</h2>
    
    <mat-dialog-content>
      <p i18n>
        You are disputing: <strong>{{ data.task.description }}</strong>
      </p>
      
      <p class="warning" i18n>
        Disputing pauses payment for this task until resolution. 
        Non-disputed tasks in this milestone will be paid immediately.
      </p>
      
      <mat-form-field appearance="outline" class="full-width">
        <mat-label i18n>Reason for dispute</mat-label>
        <textarea matInput 
          [(ngModel)]="disputeReason" 
          rows="4"
          required></textarea>
      </mat-form-field>
      
      <div class="evidence-upload">
        <label i18n>Evidence (optional)</label>
        <input type="file" (change)="onFileSelected($event)" multiple />
        @for (file of selectedFiles(); track file.name) {
          <div class="file-item">
            <mat-icon>attachment</mat-icon>
            <span>{{ file.name }}</span>
          </div>
        }
      </div>
    </mat-dialog-content>
    
    <mat-dialog-actions align="end">
      <button mat-button mat-dialog-close i18n>Cancel</button>
      <button mat-raised-button color="warn"
        [disabled]="!disputeReason"
        (click)="submitDispute()">
        <mat-icon>gavel</mat-icon>
        <span i18n>Submit Dispute</span>
      </button>
    </mat-dialog-actions>
  `
})
export class DisputeDialogComponent {
  data = inject(MAT_DIALOG_DATA);
  private dialogRef = inject(MatDialogRef<DisputeDialogComponent>);
  private qot = inject(QotService);
  
  disputeReason = '';
  selectedFiles = signal<File[]>([]);
  
  onFileSelected(event: Event) {
    const input = event.target as HTMLInputElement;
    if (input.files) {
      this.selectedFiles.set(Array.from(input.files));
    }
  }
  
  async submitDispute() {
    const evidenceHash = await this.uploadEvidence();
    
    await this.qot.disputeTask(
      this.data.contractId,
      this.data.task.milestoneId,
      this.data.task.id,
      {
        reason: this.disputeReason,
        evidenceHash
      }
    );
    
    this.dialogRef.close(true);
  }
  
  private async uploadEvidence(): Promise<string | null> {
    const files = this.selectedFiles();
    if (files.length === 0) return null;
    
    // Upload to Blossom and return hash
    // Implementation depends on Blossom service
    return null;
  }
}
```

---

## 3. Service Implementations

### 3.1 ZK Proving Service (Web Worker)

```typescript
@Injectable({ providedIn: 'root' })
export class ZkProvingService {
  private worker: Worker | null = null;
  isProving = signal(false);
  
  private async ensureWorker(): Promise<Worker> {
    if (!this.worker) {
      this.worker = new Worker(
        new URL('./zk-proving.worker', import.meta.url),
        { type: 'module' }
      );
    }
    return this.worker;
  }
  
  async generateProof(circuitId: string, inputs: unknown): Promise<Uint8Array> {
    this.isProving.set(true);
    
    try {
      const worker = await this.ensureWorker();
      
      return new Promise((resolve, reject) => {
        worker.onmessage = (e) => {
          if (e.data.type === 'proof') {
            resolve(e.data.proof);
          } else if (e.data.type === 'error') {
            reject(new Error(e.data.message));
          }
        };
        worker.onerror = (e) => reject(e);
        worker.postMessage({ type: 'prove', circuitId, inputs });
      });
    } finally {
      this.isProving.set(false);
    }
  }
}
```

**Web Worker (`zk-proving.worker.ts`):**

```typescript
/// <reference lib="webworker" />

import { UltraHonkBackend } from '@aztec/bb.js';
import { Noir } from '@noir-lang/noir_js';

let backend: UltraHonkBackend | null = null;
let circuits: Map<string, Noir> = new Map();

async function initBackend() {
  if (!backend) {
    backend = new UltraHonkBackend();
    await backend.init();
  }
  return backend;
}

async function getCircuit(circuitId: string): Promise<Noir> {
  if (!circuits.has(circuitId)) {
    const module = await import(`@qot/circuits-wasm/${circuitId}`);
    const noir = new Noir(module.circuit);
    circuits.set(circuitId, noir);
  }
  return circuits.get(circuitId)!;
}

self.onmessage = async (e) => {
  const { type, circuitId, inputs } = e.data;
  
  if (type === 'prove') {
    try {
      const backend = await initBackend();
      const circuit = await getCircuit(circuitId);
      
      const { witness } = await circuit.execute(inputs);
      const proof = await backend.generateProof(witness);
      
      self.postMessage({ type: 'proof', proof });
    } catch (error) {
      self.postMessage({ type: 'error', message: (error as Error).message });
    }
  }
};
```

### 3.2 Bundle Size Management

```typescript
// Lazy load bb.js only when needed
@Injectable({ providedIn: 'root' })
export class NoirBackendService {
  private backend: UltraHonkBackend | null = null;
  
  async loadBackend(): Promise<UltraHonkBackend> {
    if (!this.backend) {
      const { UltraHonkBackend } = await import('@aztec/bb.js');
      this.backend = new UltraHonkBackend();
      await this.backend.init();
    }
    return this.backend;
  }
}
```

### 3.3 Relay Publishing

```typescript
@Injectable({ providedIn: 'root' })
export class QotRelayService {
  private accountRelayService = inject(AccountRelayService);
  private relayPool = inject(RelayPoolService);
  private config = inject(ConfigService);
  
  async publishQotEvent(event: NostrEvent): Promise<void> {
    const userRelays = this.accountRelayService.getWriteRelays();
    const qotRelays = this.config.get<string[]>('qot.verifierRelays') ?? [];
    
    // Publish to both user relays and QoT verifier relays
    await Promise.all([
      ...userRelays.map(r => this.relayPool.publish(r, event)),
      ...qotRelays.map(r => this.relayPool.publish(r, event))
    ]);
  }
  
  // Privacy: delayed publishing
  async publishWithDelay(event: NostrEvent, maxDelayHours = 24): Promise<void> {
    const delay = Math.random() * maxDelayHours * 60 * 60 * 1000;
    setTimeout(() => this.publishQotEvent(event), delay);
  }
}
```

### 3.4 Escrow Funding

Stakes are held in the QoTEscrow contract on Aztec. The escrow token is Aztec-native (future Bitcoin integration via trustless bridges is planned but not in initial release).

```typescript
@Injectable({ providedIn: 'root' })
export class EscrowService {
  private aztec = inject(AztecService);
  
  async fundEscrow(listingId: string, amount: number, difficulty: number): Promise<void> {
    // Approve token transfer to escrow contract
    await this.aztec.approveEscrow(amount);
    
    // Accept listing locks stake in QoTEscrow
    await this.aztec.acceptListing(listingId, amount, difficulty);
  }
  
  async getEscrowBalance(contractId: string): Promise<number> {
    return this.aztec.getEscrowBalance(contractId);
  }
}
```

Existing `@getalby/sdk` integration remains available for Lightning-based tipping or off-chain payments, but contract escrow is on-chain.

---

## 4. CSS Patterns

### 4.1 Dashboard Layout

```scss
.dashboard-container {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(300px, 1fr));
  gap: 24px;
  padding: 24px;
  
  @media (max-width: 768px) {
    grid-template-columns: 1fr;
    padding: 16px;
  }
}

section {
  display: flex;
  flex-direction: column;
  gap: 16px;
  
  h2 {
    margin: 0;
    font-size: 1.25rem;
    font-weight: 500;
  }
}
```

### 4.2 Status Badges

```scss
.status-badge {
  display: inline-flex;
  align-items: center;
  padding: 4px 12px;
  border-radius: 16px;
  font-size: 0.875rem;
  font-weight: 500;
  
  &.accepted {
    background-color: var(--success-light);
    color: var(--success-dark);
  }
  
  &.disputed {
    background-color: var(--warn-light);
    color: var(--warn-dark);
  }
  
  &.pending {
    background-color: var(--neutral-light);
    color: var(--neutral-dark);
  }
}
```

### 4.3 Milestone Review Cards

```scss
.task-card {
  transition: border-color 0.2s;
  
  &.disputed {
    border-left: 4px solid var(--warn-color);
  }
  
  .outcome-display {
    display: flex;
    align-items: center;
    gap: 12px;
    
    mat-progress-bar {
      flex: 1;
    }
  }
}

.review-actions {
  display: flex;
  justify-content: flex-end;
  gap: 16px;
  padding: 24px;
  border-top: 1px solid var(--divider-color);
}
```

---

## Related Documents

- **QoT_Nostria_Client_Specification.md** — Technical architecture
- **QoT_Nostria_Implementation_Plan.md** — Timeline and phases
- **QoT_Professional_Network_UX_Analysis.md** — UX flows and wireframes
