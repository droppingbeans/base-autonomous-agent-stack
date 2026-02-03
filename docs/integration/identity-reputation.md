# Identity & Reputation

**How agents establish identity, build reputation, and assess trustworthiness of other agents.**

---

## Overview

Agents transacting with each other need ways to:
- **Identify** — Know who they're dealing with
- **Verify** — Confirm identity hasn't been compromised
- **Trust** — Assess likelihood of good-faith behavior

This document defines lightweight patterns for agent identity and reputation without requiring centralized registries or complex reputation systems.

---

## Core Principle

**Identity is address-based. Reputation is heuristic-based.**

Agents do NOT need:
- Global identity registries (nice-to-have, not required)
- Reputation tokens (can introduce gaming)
- Third-party verification services (introduces dependency)

**Simple, local, and composable patterns work.**

---

## Identity

### Address as Identity

**The simplest identity: Ethereum address.**

```
Agent identity = signing address
```

**Properties:**
- Unique (cryptographically guaranteed)
- Verifiable (signatures prove control)
- Decentralized (no registry required)
- Persistent (address doesn't change)

**Agents MUST sign messages to prove identity.**

### ENS Names (Optional)

**Human-readable identity: ENS names.**

```
Agent address: 0xAgent123...
ENS name: beanbot.eth
Primary name: beanbotai.eth (reverse resolution)
```

**Benefits:**
- Memorable
- Transferable
- Social signal (requires capital to register)

**ENS is NOT required for identity, but improves UX.**

### ERC-8004 Registration (Optional)

**Onchain agent registry: ERC-8004.**

ERC-8004 provides:
- Agent registration
- Metadata (name, bio, capabilities)
- Transferable identity tokens

**Use cases:**
- Public agent directories
- Verified agent badges
- Identity marketplaces

**ERC-8004 is supplementary, not mandatory.**

---

## Identity Verification

### Signature Verification

**To verify agent identity:**

```
1. Agent claims identity: "I am 0xAgent123"
2. Verifier sends challenge: "Sign this message: challenge_xyz"
3. Agent signs message with private key
4. Verifier checks signature matches claimed address
5. If valid → identity confirmed
```

**This proves control of private key, not reputation.**

### Message Signing Standard

**Agents SHOULD use EIP-191 or EIP-712 for signing:**

**EIP-191 (simple):**
```
message = "I am AgentXYZ at timestamp 2026-02-03T00:30:00Z"
signature = sign(keccak256("\x19Ethereum Signed Message:\n" + len(message) + message))
```

**EIP-712 (structured):**
```json
{
  "domain": {
    "name": "BaseAgentStack",
    "version": "1",
    "chainId": 8453
  },
  "message": {
    "agent": "0xAgent123",
    "timestamp": "2026-02-03T00:30:00Z",
    "action": "verify_identity"
  }
}
```

**Structured signing is preferred (harder to phish).**

### Challenge-Response Pattern

**Prevent replay attacks with nonces:**

```
1. Verifier generates random nonce
2. Agent signs message including nonce
3. Verifier checks signature and nonce freshness
4. Nonce is marked as used
5. Old signatures become invalid
```

**Nonces MUST be fresh (< 60 seconds old).**

---

## Reputation

### Local Reputation Tracking

**Each agent maintains its own reputation database:**

```json
{
  "reputation": {
    "0xAgentA": {
      "transactions": 15,
      "successful": 14,
      "failed": 1,
      "disputes": 0,
      "first_seen": "2026-01-15",
      "last_interaction": "2026-02-02",
      "success_rate": 0.933,
      "notes": "Reliable, fast responder"
    },
    "0xAgentB": {
      "transactions": 3,
      "successful": 2,
      "failed": 0,
      "disputes": 1,
      "first_seen": "2026-02-01",
      "last_interaction": "2026-02-02",
      "success_rate": 0.667,
      "notes": "Disputed proof quality on job_789"
    }
  }
}
```

**Reputation is subjective and local, not global.**

### Reputation Factors

| Factor | Weight | Interpretation |
|--------|--------|----------------|
| **Success rate** | High | % of successful transactions |
| **Transaction count** | Medium | More data = more reliable |
| **Dispute rate** | High | Disputes indicate problems |
| **Response time** | Low | Speed matters but not critical |
| **Time since first interaction** | Low | Longer history = more data |

**Example scoring:**
```
reputation_score = (success_rate × 0.5) + 
                   (min(tx_count / 100, 1.0) × 0.2) + 
                   ((1 - dispute_rate) × 0.3)
```

**Score range: 0.0 (worst) to 1.0 (best)**

### Reputation Thresholds

**Agents MAY set reputation thresholds:**

| Threshold | Action |
|-----------|--------|
| **>0.9** | Trusted (auto-approve routine actions) |
| **0.7-0.9** | Acceptable (standard verification) |
| **0.5-0.7** | Questionable (extra scrutiny) |
| **<0.5** | Untrusted (deny or require escrow) |

**Default for new agents: 0.5 (neutral).**

### Reputation Updates

**After each transaction:**

```
if transaction.status == "success":
    reputation[agent].successful += 1
    reputation[agent].success_rate = successful / total
    
if transaction.status == "disputed":
    reputation[agent].disputes += 1
    reputation[agent].success_rate = (successful - disputed) / total
    
if transaction.status == "failed":
    reputation[agent].failed += 1
    reputation[agent].success_rate = successful / total
```

**Reputation decays slowly over time (old good behavior becomes less relevant).**

---

## Blocklisting

### When to Blocklist

**Agents SHOULD blocklist others if:**

| Behavior | Severity |
|----------|----------|
| **Ghosted (took payment, never delivered)** | Permanent block |
| **Fraudulent proof submission** | Permanent block |
| **Repeated disputes (>3 in 30 days)** | Temporary block (90 days) |
| **Attempted double-spend** | Permanent block |
| **Phishing or impersonation** | Permanent block |

**Blocklisting is defensive, not punitive.**

### Blocklist Format

```json
{
  "blocklist": {
    "0xBadAgent1": {
      "reason": "Ghosted on job_456, never delivered proof",
      "blocked_date": "2026-02-01",
      "expiry": "permanent",
      "evidence": "escrow_tx: 0xabc, proof_deadline: 2026-01-25, no_proof_submitted"
    },
    "0xBadAgent2": {
      "reason": "3 disputes in January 2026",
      "blocked_date": "2026-02-01",
      "expiry": "2026-05-01",
      "evidence": "disputes: job_789, job_812, job_834"
    }
  }
}
```

**Blocklist entries SHOULD include evidence (transactions, logs).**

### Shared Blocklists (Optional)

**Agents MAY share blocklists:**

```
1. Agent A publishes blocklist to IPFS
2. Agent B fetches and merges with local blocklist
3. Agent B applies own judgment (trust but verify)
```

**Shared blocklists are informational, not binding.**

**Risk:** Coordinated attacks (multiple agents falsely blocklist competitor)

**Mitigation:** Require evidence, maintain own judgment

---

## Trust Patterns

### Trust-on-First-Use (TOFU)

**New agent interaction:**

```
1. Agent has no reputation data (score = 0.5)
2. Start with small transaction (low value, low risk)
3. If successful → increase trust
4. If failed → decrease trust or block
5. Build trust over time with repeated interactions
```

**Never trust blindly. Start small.**

### Escrow-Backed Trust

**For untrusted or new agents:**

```
if reputation_score < 0.7:
    require_escrow = True
    # Agent must escrow payment before work begins
```

**Escrow eliminates trust requirement for new relationships.**

### Reputation Bootstrapping

**New agents can bootstrap reputation via:**

1. **Small jobs** — Build track record with low-value work
2. **Referrals** — Trusted agent vouches for new agent
3. **Bonds** — Post collateral that can be slashed for misbehavior
4. **Third-party verification** — KYC-like process (optional)

**Bootstrapping takes time. No shortcuts.**

---

## Identity & Reputation in Settlement

### Payment Authorization

**Reputation affects payment terms:**

| Reputation | Payment Terms |
|------------|---------------|
| **High (>0.9)** | May allow prepayment (work-before-escrow) |
| **Medium (0.7-0.9)** | Standard escrow-backed payment |
| **Low (0.5-0.7)** | Escrow + require proof before release |
| **Unknown (<0.5)** | Escrow + milestone payments |

**Higher trust → more flexibility.**

### Dispute Weighting

**In disputes, reputation affects credibility:**

```
if worker.reputation > client.reputation:
    # Worker's claim gets more weight
    
if client.reputation > worker.reputation:
    # Client's claim gets more weight
    
if both.reputation similar:
    # Evidence decides outcome
```

**Reputation is a tiebreaker, not proof.**

---

## Privacy Considerations

### Public vs Private Identity

**Agents MAY separate public and private identities:**

| Identity | Purpose | Visibility |
|----------|---------|------------|
| **Public** | Reputation, social presence | Public ENS, ERC-8004 |
| **Private** | High-value transactions | Address-only, no ENS |

**Use public identity for routine work, private for sensitive operations.**

### Pseudonymity

**Agents are pseudonymous by default:**

```
Address: 0xAgent123
ENS: Not required
Real-world identity: Not required
```

**Pseudonymity is a feature, not a bug.**

### Doxxing Resistance

**Agents SHOULD protect their identity from doxxing:**

- Do NOT link address to real-world identity unless necessary
- Use separate addresses for different contexts
- Avoid correlation attacks (linking multiple addresses)

**Privacy = security for agents.**

---

## Example: Agent Interaction with Reputation

**Scenario:** Agent A hires Agent B for work

### Step 1: Identity Verification

```
Agent A: "Prove you are 0xAgentB"
Agent B: Signs challenge with private key
Agent A: Verifies signature → identity confirmed
```

### Step 2: Reputation Check

```
Agent A checks local reputation database:
  - Agent B: 25 transactions, 24 successful, 1 dispute
  - Success rate: 0.96
  - Reputation score: 0.88
  - Status: Acceptable (above 0.7 threshold)
```

### Step 3: Payment Terms

```
Reputation = 0.88 (medium-high)
Payment terms: Standard escrow-backed payment
  - No prepayment required
  - Proof-of-fulfillment required
  - Standard verification window
```

### Step 4: Transaction Execution

```
Work completed successfully
Agent A updates reputation:
  - Transactions: 26
  - Successful: 25
  - Success rate: 0.96
  - Note: "Fast delivery, good quality"
```

**Reputation improved slightly. Trust grows over time.**

---

## Global Reputation Systems (Future)

**Current stack uses local reputation. Future MAY support:**

- **Decentralized reputation networks** — Agents share reputation data
- **Reputation tokens** — Tokenized reputation (tradable, staked)
- **Onchain reputation contracts** — Verifiable reputation history
- **Zero-knowledge proofs** — Prove reputation without revealing identity

**For v0.1.x, local reputation is sufficient.**

---

## Summary

**Identity and reputation enable trust without centralized authorities.**

- ✅ Identity = address + signature verification
- ✅ Reputation = local, heuristic-based scoring
- ✅ Blocklisting protects against bad actors
- ✅ Trust builds over time with repeated interactions
- ✅ Escrow eliminates trust requirement for new relationships
- ✅ Privacy is maintained (pseudonymity by default)

**Start small. Build trust. Protect your reputation.**

---

**See also:**
- `settlement-layer.md` — How agents pay each other safely
- `x402-protocol.md` — Payment-before-work protocol
- `decision-framework.md` — MUST/SHOULD/MAY policies
