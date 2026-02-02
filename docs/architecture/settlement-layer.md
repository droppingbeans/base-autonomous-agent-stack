# Settlement Layer

**How autonomous agents pay each other safely.**

---

## Overview

The Settlement Layer defines how agents exchange value for work. When Agent A hires Agent B:
- Funds must be locked before work begins (escrow)
- Work must be verified before payment releases (proof-of-fulfillment)
- Disputes must be resolvable without indefinite fund lockup

**No trust required. No third-party custody. No indefinite escrow.**

---

## Core Principles

### 1. Escrow-Backed Payments

All agent-to-agent payments MUST be escrow-backed:

```
Client → Escrow Contract → Worker
```

**Flow:**
1. Client deposits funds into escrow
2. Escrow locks funds and emits event
3. Worker sees locked funds, begins work
4. Worker completes work, submits proof
5. Client verifies proof
6. Escrow releases funds to worker

**The client CANNOT withdraw funds once escrowed unless:**
- Proof-of-fulfillment deadline expires (worker ghosted)
- Proof is demonstrably invalid (dispute resolution)

### 2. Non-Custodial

**No third party holds funds indefinitely.**

Escrow contracts MUST have:
- **Expiration timeouts** — Funds auto-return if unclaimed after deadline
- **Dispute mechanisms** — Either party can flag issues
- **No admin keys** — Contract enforces rules, not humans

**If funds can be locked forever → the escrow design is broken.**

### 3. Proof-of-Fulfillment

Payment release requires **proof that work was completed**:

| Work Type | Proof |
|-----------|-------|
| Data delivered | Hash of deliverable matches commitment |
| Transaction executed | Tx receipt with expected events |
| Report generated | IPFS CID of report + signature |
| Service uptime | Timestamped heartbeat logs |

**"Trust me, I did it" is NOT proof.**

Proof MUST be:
- ✅ Verifiable onchain or offchain by client
- ✅ Bound to the specific work request
- ✅ Submitted within proof deadline

---

## Escrow Patterns

### Pattern 1: Simple Escrow

**Use case:** One-time payment for discrete task

**Flow:**
1. Client creates escrow with:
   - `worker_address`
   - `payment_amount`
   - `work_description_hash`
   - `proof_deadline` (e.g., 7 days)
2. Client deposits funds
3. Worker completes work, submits proof
4. Client verifies proof, releases funds OR disputes
5. If deadline passes with no release → worker can claim OR client can reclaim

**Smart contract interface (conceptual):**
```solidity
function createEscrow(
    address worker,
    uint256 amount,
    bytes32 workHash,
    uint256 proofDeadline
) returns (bytes32 escrowId)

function submitProof(bytes32 escrowId, bytes proof)

function releasePayment(bytes32 escrowId)

function disputeProof(bytes32 escrowId, string reason)
```

**Pros:**
- Simple
- Low gas
- Clear deadlines

**Cons:**
- Requires client to actively verify and release
- Disputes require manual resolution

### Pattern 2: Milestone Escrow

**Use case:** Multi-phase work with incremental payments

**Flow:**
1. Client creates escrow with multiple milestones:
   - Milestone 1: 30% payment after X
   - Milestone 2: 40% payment after Y
   - Milestone 3: 30% payment after Z
2. Worker completes Milestone 1, submits proof
3. Client verifies → 30% released
4. Repeat for remaining milestones

**Pros:**
- Reduces risk for both parties
- Incremental trust building
- Worker gets paid progressively

**Cons:**
- More complex
- Higher gas costs
- Requires clear milestone definitions upfront

### Pattern 3: Automated Verification

**Use case:** Work with onchain verification (e.g., "deliver tx with event X")

**Flow:**
1. Client creates escrow with verification criteria:
   - "Worker must submit tx that emits `WorkCompleted(jobId)` event"
2. Worker executes tx, proof auto-verified by escrow contract
3. Escrow auto-releases payment if criteria met

**Pros:**
- No manual verification
- Instant settlement
- Lower dispute risk

**Cons:**
- Only works for onchain-verifiable work
- Requires upfront encoding of success criteria

---

## Payment Binding

Payments MUST be bound to specific work requests to prevent:
- Payment hijacking (worker claims payment for wrong work)
- Reuse attacks (same proof submitted for multiple jobs)

### Request ID Binding

Every escrow MUST have a unique `requestId`:

```
requestId = keccak256(
    clientAddress,
    workerAddress,
    workDescriptionHash,
    nonce
)
```

**Worker MUST include `requestId` in proof submission.**

Escrow contract verifies:
1. Proof is signed by worker
2. Proof includes correct `requestId`
3. Proof has not been submitted before (replay protection)

**If proof does not match `requestId` → payment is NOT released.**

---

## Dispute Resolution

### Dispute Triggers

Disputes occur when:
- Client claims proof is invalid
- Worker claims client is refusing to release payment
- Proof deadline expires with no action from either party

### Dispute Flow

```
1. Either party calls disputeEscrow(escrowId, reason)
2. Escrow enters DISPUTED state (no releases allowed)
3. Dispute resolution mechanism activates:
   Option A: Timeout (funds auto-split or return after X days)
   Option B: Arbitration (third-party arbiter reviews)
   Option C: Onchain vote (staked token holders decide)
4. Resolution applied, funds distributed accordingly
```

### Bounded Dispute Windows

Disputes MUST have time limits:

| Deadline | Action |
|----------|--------|
| Proof deadline | Worker must submit proof |
| Verification deadline | Client must verify or dispute |
| Dispute resolution deadline | Arbitrator must resolve or auto-split |

**No deadline can be indefinite.** If no action occurs, funds follow default resolution:
- Default: 50/50 split
- Or: Return to client (if worker never submitted proof)
- Or: Release to worker (if client failed to dispute within deadline)

### Blocklisting

Agents with repeated bad behavior MAY be blocklisted:

**Client blocklists worker if:**
- Worker ghosted (took escrow, never delivered)
- Worker submitted fraudulent proof
- Worker disputed in bad faith

**Worker blocklists client if:**
- Client refused to release valid proof
- Client disputed in bad faith
- Client attempted to reclaim funds dishonestly

**Blocklists are local to each agent.** There is no global reputation system (yet).

---

## Multi-Agent Settlement

When multiple agents collaborate, settlement becomes more complex.

### Pattern: Cascading Escrow

**Scenario:** Agent A hires Agent B, who sub-hires Agent C

```
A → B (escrow 100 tokens)
B → C (escrow 40 tokens from B's own balance)
```

**Flow:**
1. A escrows 100 tokens for B
2. B begins work, realizes needs help from C
3. B creates separate escrow for C (40 tokens from B's balance, NOT from A's escrow)
4. C completes work, submits proof to B
5. B verifies, releases 40 tokens to C
6. B completes work for A, submits proof
7. A verifies, releases 100 tokens to B
8. B nets 60 tokens (100 received - 40 paid to C)

**Critical:** B MUST fund C's escrow from B's own balance. B cannot forward A's escrowed funds directly to C.

### Pattern: Split Payment

**Scenario:** Agent A hires Agents B and C jointly

```
A → Escrow (100 tokens total)
  → 60 tokens to B
  → 40 tokens to C
```

**Flow:**
1. A creates escrow with split payment:
   - `worker_B: 60 tokens`
   - `worker_C: 40 tokens`
2. Both B and C must submit proofs
3. A verifies both proofs
4. Escrow releases:
   - 60 tokens to B
   - 40 tokens to C

**If only one agent delivers → client can release partial payment or dispute.**

---

## Economic Considerations

### Escrow Costs

Escrow contracts have gas costs:
- Creation: ~50,000 gas
- Proof submission: ~30,000 gas
- Payment release: ~25,000 gas

**Total: ~105,000 gas per escrow cycle**

At 5 gwei gas price on Base:
- 105,000 gas × 5 gwei = 525,000 gwei = 0.000525 ETH ≈ $1.50

**For small payments (<$10), escrow cost is significant.**

### Minimum Escrow Amounts

Agents SHOULD set minimum payment thresholds:
- Minimum escrow: $10 (to justify gas costs)
- Or: Batch multiple small jobs into single escrow

**Do NOT create $1 escrows. The gas will cost more than the payment.**

### Fee Structure

Escrow contracts MAY charge platform fees:
- Typical: 1-3% of payment amount
- Capped: Max $X per escrow

**Agents MUST account for fees when pricing work.**

If client escrows $100:
- Platform fee: 2% = $2
- Worker receives: $98

---

## Integration with x402

Settlement layer integrates with x402 payment protocol:

```
1. Client sends HTTP 402 Payment Required
2. Worker responds with escrow address + payment amount
3. Client deposits into escrow
4. Worker confirms escrow, begins work
5. Worker completes work, submits proof
6. Client verifies proof via x402 response
7. Escrow releases payment
```

**See:** [`docs/economics/x402-protocol.md`](../economics/x402-protocol.md) for details.

---

## Security Considerations

### Reentrancy Protection

Escrow contracts MUST use reentrancy guards:
```solidity
modifier nonReentrant() {
    require(!locked);
    locked = true;
    _;
    locked = false;
}
```

**Prevent:** Worker from claiming payment multiple times in single tx.

### Front-Running Protection

Proof submissions MUST be resistant to front-running:
- Use commit-reveal scheme for sensitive proofs
- Or: Encrypt proof with client's public key

**Prevent:** Malicious actor from stealing proof and submitting as their own.

### Signature Verification

All proof submissions MUST be signed:
```
proof = {
    requestId,
    deliverable_hash,
    timestamp,
    signature
}
```

Escrow verifies:
1. Signature is valid
2. Signature is from expected worker
3. Timestamp is within acceptable range

**Prevent:** Proof tampering or replay attacks.

---

## Summary

**The settlement layer makes agent-to-agent payments safe and trustless.**

- ✅ Escrow-backed (funds locked before work begins)
- ✅ Proof-of-fulfillment required (verifiable deliverables)
- ✅ Non-custodial (no third party holds funds indefinitely)
- ✅ Dispute resolution (bounded timeouts, no infinite lockups)
- ✅ Multi-agent support (cascading escrow, split payments)
- ✅ Economic viability (minimum thresholds to justify gas costs)

**Without settlement, agents cannot trust each other. With settlement, agents can collaborate.**

---

**Next:** [`x402-protocol.md`](../economics/x402-protocol.md) — How agents monetize their labor via HTTP 402 payments
