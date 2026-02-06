# Detailed Learning Guide - Hyperscale-rs (v2)

## Module 2: BFT Consensus & Safety

### 2.3 Vote Locking & HotStuff-2 Unlock (CRITICAL CORRECTION)

**What is Vote Locking?**

Vote locking is a safety mechanism that prevents a validator from voting for conflicting blocks at the same height **within the same round**.

**How it works**:
1. When a validator votes for a block at height H, round R, it creates a vote lock: `voted_heights[H] = (block_hash, R)`
2. If another block arrives at height H in the same round R, the validator checks its vote lock and **cannot vote again**.

**The Critical Misunderstanding (v1)**:

In v1 of this guide, we incorrectly stated that vote locks persist across rounds. This is **not true**.

**The Correct Behavior (v2)**:

Vote locks are **cleared** when a round advances without a QC forming at the current height. This is the **HotStuff-2 unlock mechanism**.

**HotStuff-2 Unlock Mechanism**:

When a round advances (e.g., from R to R+1):
1. The system checks if a QC was formed at the current height H.
2. **If NO QC formed**: The vote lock for height H is **cleared**.
3. **If YES QC formed**: The vote lock is **kept**.

**Why is this safe?**
- If no QC formed, the previous block didn_t achieve consensus.
- It is safe to vote for a different block in the new round.
- This is a key feature of HotStuff-2 that ensures **liveness** (the system doesn_t get stuck).

**Code Reference**:
- `state.rs`, line 4331: `self.voted_heights.remove(&height)` - This is where the vote lock is cleared.

**Safety vs Liveness**:
- **Safety**: Vote lock prevents equivocation within a round.
- **Liveness**: Clearing vote locks on round advance prevents the system from getting stuck.

### 2.4 Round Advancement & Vote Lock Clearing (CRITICAL CORRECTION)

**What is Round Advancement?**

Round advancement is the process of moving to the next round at the same height when a block fails to achieve consensus.

**How it works**:
1. A timeout occurs (e.g., `on_proposal_timer`).
2. The round number is incremented (e.g., `self.view += 1`).
3. The **HotStuff-2 unlock mechanism** is triggered.

**Vote Lock Clearing during Round Advancement**:

When `advance_round()` is called:
1. It checks if a QC was formed at the current height.
2. If no QC was formed, it **clears the vote lock** for that height.
3. This allows the validator to vote for a new block in the new round.

**Code Reference**:
- `state.rs`, line 4304: `advance_round()` - The main function for round advancement.
- `state.rs`, line 4331: `self.voted_heights.remove(&height)` - The vote lock clearing.

**What happens next?**
- **If vote lock cleared**: The new leader can propose a new block, and validators can vote for it.
- **If vote lock kept**: The new leader must re-propose the same block that the validator is locked to.

### 2.5 View Synchronization & Vote Locks (Updated)

**What is View Synchronization?**

View synchronization is how validators stay in sync with the current round number.

**How it works**:
- When a validator receives a QC with a higher round number, it updates its local round number (`self.view`).

**Vote Locks during View Synchronization**:
- When a validator syncs to a new view, it **does not automatically clear its vote locks**.
- Vote locks are only cleared during **round advancement** (when a timeout occurs).

**Key Point**:
- View synchronization and vote lock clearing are **separate mechanisms**.
- View sync keeps validators in the same round.
- Round advance (with vote lock clearing) allows progress when a round fails.
