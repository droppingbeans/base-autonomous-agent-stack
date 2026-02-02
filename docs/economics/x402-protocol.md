# x402 Protocol

**HTTP 402 Payment Required for agent monetization.**

---

## Overview

**x402** is a protocol for binding payments to specific work requests. It extends HTTP status code 402 (Payment Required) for agent-to-agent commerce.

**Core idea:** Before work begins, client must prove payment. After work completes, agent must prove fulfillment.

**No proof, no payment. No payment, no work.**

---

## Why 402?

HTTP 402 was reserved for "Payment Required" but never standardized. x402 gives it meaning for autonomous agents:

**Traditional 402:**
```
HTTP/1.1 402 Payment Required
```

**x402:**
```
HTTP/1.1 402 Payment Required
X-Payment-Address: 0xEscrowContract
X-Payment-Amount: 0.01 ETH
X-Request-ID: req_abc123
X-Proof-Deadline: 2026-02-09T18:00:00Z
```

---

## Protocol Flow

### 1. Work Request (Client → Agent)

Client sends request for paid work:

```http
POST /api/work HTTP/1.1
Host: agent.example.com
Content-Type: application/json

{
  "task": "analyze_token",
  "parameters": {
    "token_address": "0x123...",
    "metrics": ["price", "volume", "holders"]
  }
}
```

### 2. Payment Required (Agent → Client)

Agent responds with 402 and payment details:

```http
HTTP/1.1 402 Payment Required
X-Payment-Address: 0xEscrowContract...
X-Payment-Amount: 0.01
X-Payment-Token: ETH
X-Request-ID: req_abc123
X-Proof-Deadline: 2026-02-09T18:00:00Z
Content-Type: application/json

{
  "message": "Payment required before work begins",
  "escrow_address": "0xEscrowContract...",
  "amount": "0.01",
  "token": "ETH",
  "request_id": "req_abc123",
  "proof_deadline": "2026-02-09T18:00:00Z",
  "estimated_completion": "2026-02-02T19:00:00Z"
}
```

### 3. Payment Submission (Client → Escrow)

Client deposits payment into escrow contract:

```solidity
escrow.deposit{value: 0.01 ether}(
    requestId: "req_abc123",
    worker: agent_address,
    client: client_address,
    proofDeadline: unix_timestamp
)
```

Escrow emits event:

```solidity
event PaymentEscrowed(
    bytes32 indexed requestId,
    address indexed client,
    address indexed worker,
    uint256 amount,
    uint256 proofDeadline
);
```

### 4. Payment Confirmation (Client → Agent)

Client notifies agent that payment is escrowed:

```http
POST /api/work HTTP/1.1
Host: agent.example.com
X-Payment-Tx: 0xTransactionHash...
X-Request-ID: req_abc123
Content-Type: application/json

{
  "task": "analyze_token",
  "parameters": { ... },
  "payment_tx": "0xTransactionHash..."
}
```

### 5. Work Execution (Agent)

Agent verifies escrow onchain:
1. Check tx receipt for `PaymentEscrowed` event
2. Verify `requestId` matches
3. Verify `amount` matches expected payment
4. Verify `client` matches requester

If verified → Agent begins work.

### 6. Proof of Fulfillment (Agent → Escrow)

Agent completes work and submits proof:

```solidity
escrow.submitProof(
    requestId: "req_abc123",
    proof: proof_data,
    signature: agent_signature
)
```

**Proof formats (depending on work type):**
- **Data deliverable:** `keccak256(deliverable)` matches commitment
- **Transaction executed:** Tx hash + receipt with expected events
- **Report generated:** IPFS CID + signature
- **API response:** Signed response hash

### 7. Verification & Release (Client → Escrow)

Client verifies proof and releases payment:

```solidity
escrow.releasePayment(requestId: "req_abc123")
```

**Or client disputes:**

```solidity
escrow.disputeProof(
    requestId: "req_abc123",
    reason: "Proof does not match deliverable"
)
```

### 8. Payment Release (Escrow → Agent)

If proof is valid and client releases:

```solidity
event PaymentReleased(
    bytes32 indexed requestId,
    address indexed worker,
    uint256 amount
);
```

Agent receives payment.

---

## Request ID Binding

Every x402 payment MUST be bound to a unique `requestId`:

```
requestId = keccak256(
    clientAddress,
    agentAddress,
    taskDescriptionHash,
    nonce,
    timestamp
)
```

**Purpose:**
- Prevents payment hijacking (agent claims wrong payment)
- Prevents replay attacks (same proof for multiple payments)
- Binds payment to specific work

**Agent MUST verify:**
1. `requestId` in payment matches `requestId` in escrow event
2. `requestId` has not been used before
3. `requestId` is included in proof submission

---

## Proof Requirements

### Proof MUST Include

1. **Request ID** — Binds proof to specific payment
2. **Deliverable Hash** — Hash of completed work
3. **Timestamp** — When work was completed
4. **Signature** — Agent's signature over all above

```json
{
  "request_id": "req_abc123",
  "deliverable_hash": "0xabc...",
  "timestamp": "2026-02-02T18:45:00Z",
  "signature": "0x..."
}
```

### Proof MUST Be Verifiable

Client MUST be able to verify proof without trusting agent:

**For data deliverables:**
```
claimed_hash = keccak256(deliverable)
proof_hash == claimed_hash ? VALID : INVALID
```

**For transactions:**
```
receipt = eth_getTransactionReceipt(proof_tx_hash)
receipt.status == 1 AND receipt.logs contains expected event ? VALID : INVALID
```

**For reports/documents:**
```
ipfs_content = fetch(proof_ipfs_cid)
keccak256(ipfs_content) == commitment_hash ? VALID : INVALID
```

### Proof Deadline

All proofs MUST be submitted before deadline:

```
if current_time > proof_deadline:
    REJECT proof submission
    client MAY reclaim escrowed funds
```

**Typical deadline:** 7 days from escrow creation

**If agent misses deadline → client gets refund.**

---

## Payment-Before-Work Guarantee

x402 enforces **payment-before-work**:

```
Client MUST escrow funds → Agent begins work → Agent completes → Client verifies → Funds release
```

**Agent MUST NOT begin work until:**
1. Payment is escrowed onchain
2. Escrow event is verified
3. Amount matches expected payment

**Exception:** Agent MAY begin work before escrow if client is trusted/whitelisted, but this is OPTIONAL.

---

## Refund Rules

### Client Refund Scenarios

Client MAY reclaim funds if:

| Scenario | Action |
|----------|--------|
| Agent never submits proof | Reclaim after `proof_deadline` expires |
| Agent submits invalid proof | Dispute, then reclaim if dispute wins |
| Agent ghosts | Reclaim after `proof_deadline + dispute_window` |

### Agent Protection

Agent is protected if:
- Proof is submitted before deadline
- Proof is verifiable
- Client fails to verify/release within `verification_deadline`

**Auto-release:** If client does not respond within `verification_deadline`, escrow SHOULD auto-release payment to agent.

**Typical verification deadline:** 3 days after proof submission

---

## Dispute Handling

### Dispute Flow

```
1. Agent submits proof
2. Client disputes: "Proof invalid because X"
3. Escrow enters DISPUTED state
4. Dispute resolution mechanism activates:
   - Timeout (auto-split after 7 days)
   - Arbitration (third-party review)
   - Onchain verification (if work is onchain-verifiable)
5. Funds distributed according to resolution
```

### Dispute Reasons

Valid dispute reasons:

| Reason | Example |
|--------|---------|
| **Proof hash mismatch** | `keccak256(deliverable) != proof.deliverable_hash` |
| **Work incomplete** | Deliverable missing required components |
| **Work incorrect** | Deliverable does not meet specifications |
| **Deadline missed** | Proof submitted after `proof_deadline` |

**Frivolous disputes (no valid reason) MAY result in client losing escrowed funds.**

---

## Fee Structures

x402 does NOT dictate pricing. Agents set their own rates.

### Pricing Models

| Model | Description | Example |
|-------|-------------|---------|
| **Fixed price** | Flat fee per task | $10 for token analysis |
| **Hourly rate** | Time-based | $50/hour for research |
| **Usage-based** | Per unit consumed | $0.01 per API call |
| **Value-based** | % of value delivered | 5% of profits generated |

**Agent MUST communicate pricing before work begins (in 402 response).**

### Platform Fees

Escrow contracts MAY charge platform fees:

```
Total escrowed: $100
Platform fee (2%): $2
Agent receives: $98
```

**Agent MUST account for fees when quoting prices.**

---

## Abuse Prevention

### Double-Spending Prevention

Escrow contract MUST prevent:
- Client from withdrawing funds before proof deadline
- Agent from claiming payment without valid proof
- Either party from reusing same `requestId`

### Proof Replay Prevention

Escrow contract MUST track used `requestId`s:

```solidity
mapping(bytes32 => bool) public usedRequestIds;

function submitProof(bytes32 requestId, ...) {
    require(!usedRequestIds[requestId], "Request ID already used");
    usedRequestIds[requestId] = true;
    ...
}
```

### Client Ghosting Prevention

If client does not verify proof within `verification_deadline`:

```
agent MAY auto-claim payment after deadline
```

**Default verification deadline:** 3 days

---

## Multi-Agent Payments

### Sequential Work

Agent A completes work → Agent B continues:

```
Client → Escrow (100 tokens)
  → Agent A submits proof → 60 tokens released to A
  → Agent B submits proof → 40 tokens released to B
```

**Milestone-based escrow with sequential releases.**

### Parallel Work

Agent A and Agent B work simultaneously:

```
Client → Escrow A (60 tokens) → Agent A
Client → Escrow B (40 tokens) → Agent B
```

**Separate escrows for independent agents.**

---

## Integration with Settlement Layer

x402 sits on top of the Settlement Layer:

```
x402 (Application Protocol)
    ↓
Settlement Layer (Escrow Contracts)
    ↓
Base L2 (Blockchain)
```

**x402 defines:**
- HTTP headers and response codes
- Request ID binding
- Proof formats

**Settlement Layer defines:**
- Escrow smart contracts
- Payment release logic
- Dispute resolution

**See:** [`architecture/settlement-layer.md`](../architecture/settlement-layer.md)

---

## Example: Full x402 Flow

**Scenario:** Client wants token analysis, agent charges 0.01 ETH

### Step 1: Request

```http
POST /api/analyze_token HTTP/1.1
Content-Type: application/json

{
  "token": "0x123...",
  "metrics": ["price", "volume"]
}
```

### Step 2: 402 Response

```http
HTTP/1.1 402 Payment Required
X-Payment-Address: 0xEscrow...
X-Payment-Amount: 0.01
X-Request-ID: req_abc123

{"message": "Payment required"}
```

### Step 3: Client Escrows

```solidity
escrow.deposit{value: 0.01 ether}("req_abc123", agent, client, deadline)
```

### Step 4: Agent Verifies Escrow

```javascript
const event = await escrow.queryFilter("PaymentEscrowed", {requestId: "req_abc123"})
// Verify amount, client, deadline
```

### Step 5: Agent Completes Work

```json
{
  "token": "0x123...",
  "price": "$1.23",
  "volume": "$4.56M",
  "analysis": "..."
}
```

### Step 6: Agent Submits Proof

```solidity
proof = {
    requestId: "req_abc123",
    deliverableHash: keccak256(analysis),
    signature: sign(deliverableHash)
}
escrow.submitProof(proof)
```

### Step 7: Client Verifies & Releases

```javascript
const deliveredAnalysis = fetch("/api/analysis/req_abc123")
const hash = keccak256(deliveredAnalysis)
if (hash == proof.deliverableHash) {
    escrow.releasePayment("req_abc123")
}
```

### Step 8: Agent Receives Payment

```solidity
event PaymentReleased("req_abc123", agent, 0.01 ether)
```

**Done. Agent got paid, client got work.**

---

## Summary

**x402 makes agent labor monetizable.**

- ✅ Payment-before-work (escrow enforced)
- ✅ Proof-of-fulfillment required (verifiable deliverables)
- ✅ Request ID binding (prevents payment hijacking)
- ✅ Dispute resolution (bounded timeouts)
- ✅ Refund protection (for both client and agent)
- ✅ Multi-agent support (sequential, parallel)

**x402 is the protocol. Escrow contracts are the enforcement.**
