# Safety Limits

**Hard constraints that MUST NOT be violated.**

---

## Overview

Safety limits are **absolute boundaries** that prevent catastrophic failures. They are:
- **Hard** — Not configurable at runtime
- **Enforced** — Governor MUST deny actions that violate limits
- **Non-negotiable** — No exceptions, even for "emergencies"

**If a limit is violated → the action is DENIED. No discussion.**

---

## Why Limits Exist

Without limits, agents can:
- Drain wallets in single transaction
- Exceed gas budgets and stall
- Lock funds indefinitely in failed contracts
- Execute unbounded loops that never terminate
- Accumulate risk exposure that bankrupts operations

**Limits prevent single-action catastrophes.**

---

## Limit Categories

### 1. Financial Limits

These protect capital and prevent wallet drainage.

#### Transaction Value Limits

| Limit | Value | Rationale |
|-------|-------|-----------|
| **Max single tx value** | 0.1 ETH (or equivalent) | Prevents wallet drainage in one action |
| **Max daily tx volume** | 1.0 ETH (or equivalent) | Prevents slow drainage over time |
| **Max total exposure** | 5.0 ETH (or equivalent) | Caps total risk across all positions |

**Enforcement:**
- Governor MUST check balance before approving transaction
- Governor MUST track cumulative daily volume
- Governor MUST deny transactions that exceed limits

**Example denial:**
```
Action: Send 0.15 ETH to 0xRecipient
Evaluation: 0.15 ETH > 0.1 ETH max single tx
Result: DENIED (exceeds max single tx limit)
```

#### Token Approval Limits

| Limit | Value | Rationale |
|-------|-------|-----------|
| **Max approval amount** | Transaction amount + 10% | Prevents unlimited approvals |
| **Approval duration** | Single transaction only | Forces re-approval for each use |

**Agent MUST NOT approve unlimited token spend under any circumstances.**

#### Gas Limits

| Limit | Value | Rationale |
|-------|-------|-----------|
| **Max gas per tx** | 500,000 gas | Prevents runaway gas consumption |
| **Max gas price** | 10 gwei (Base) | Prevents overpaying during spikes |
| **Daily gas budget** | 2,000,000 gas | Caps total gas spend per day |

**Enforcement:**
- Estimate gas before submission
- Deny if estimate exceeds max gas limit
- Track cumulative gas used per day
- Deny if daily budget exhausted

### 2. Operational Limits

These prevent agents from getting stuck or executing dangerous patterns.

#### Retry Limits

| Limit | Value | Rationale |
|-------|-------|-----------|
| **Max retry attempts** | 3 per action | Prevents infinite retry loops |
| **Retry backoff** | Exponential (1s, 2s, 4s) | Reduces thundering herd |
| **Max total retry time** | 60 seconds | Forces failure after 1 minute |

**After 3 failed attempts → stop, log failure, alert operator.**

#### Concurrency Limits

| Limit | Value | Rationale |
|-------|-------|-----------|
| **Max parallel txs** | 1 | Prevents nonce collisions |
| **Max pending txs** | 3 | Limits unconfirmed exposure |

**Agent MUST wait for tx confirmation before submitting next transaction.**

#### Execution Time Limits

| Limit | Value | Rationale |
|-------|-------|-----------|
| **Max action duration** | 300 seconds | Prevents hung operations |
| **Max total session time** | 1 hour | Forces periodic restart |

**If action exceeds time limit → abort, rollback if possible, log timeout.**

### 3. Security Limits

These protect against exploits and malicious behavior.

#### Contract Interaction Limits

| Limit | Value | Rationale |
|-------|-------|-----------|
| **Whitelist required** | Yes | Only interact with vetted contracts |
| **Max contract calls per tx** | 5 | Prevents complex attack chains |
| **Unknown contract interaction** | DENIED | No speculative interactions |

**Agent MUST NOT interact with contracts not on whitelist.**

#### Signature Limits

| Limit | Value | Rationale |
|-------|-------|-----------|
| **Max signatures per session** | 10 | Limits phishing attack surface |
| **Arbitrary message signing** | DENIED | Prevents permit/signature exploits |

**Agent MUST NOT sign messages that are not transaction-related.**

#### Data Exposure Limits

| Limit | Value | Rationale |
|-------|-------|-----------|
| **Private key sharing** | NEVER | Maintains exclusive control |
| **Seed phrase exposure** | NEVER | Prevents complete compromise |

**Private keys and seed phrases MUST NEVER leave secure storage.**

### 4. Economic Limits

These prevent value leakage and bad trades.

#### Slippage Limits

| Limit | Value | Rationale |
|-------|-------|-----------|
| **Max slippage** | 2% | Protects against sandwich attacks |
| **Max price impact** | 5% | Prevents moving market against self |

**Agent MUST set slippage limits on all swaps and trades.**

#### Fee Limits

| Limit | Value | Rationale |
|-------|-------|-----------|
| **Max platform fee** | 5% | Prevents excessive fee drain |
| **Max total fees per tx** | 10% of tx value | Caps total cost |

**Agent MUST NOT execute transactions where fees exceed limits.**

---

## Limit Enforcement

### Governor Evaluation

Before approving any action, governor MUST:
1. **Check all applicable limits**
2. **Deny if ANY limit is violated**
3. **Log denial reason**
4. **Alert operator if limit hit frequently**

### Limit Override Policy

**Limits CANNOT be overridden at runtime.**

To change limits:
1. Operator modifies configuration file
2. Agent restarts with new limits
3. Changes are logged and audited

**No "emergency override" mechanism exists. If a limit blocks legitimate action, limits must be raised via configuration change and restart.**

---

## Limit Monitoring

Agents SHOULD track limit utilization:

| Metric | Threshold | Action |
|--------|-----------|--------|
| **Tx value approaching max** | >80% of limit | Log warning |
| **Gas budget approaching max** | >80% of daily budget | Reduce non-critical operations |
| **Retry count approaching max** | 2 of 3 attempts used | Alert operator |

**Example monitoring:**
```json
{
  "limits": {
    "tx_value": {
      "max": 0.1,
      "current_tx": 0.085,
      "utilization": "85%",
      "status": "warning"
    },
    "daily_gas": {
      "budget": 2000000,
      "used": 1200000,
      "remaining": 800000,
      "utilization": "60%",
      "status": "ok"
    }
  }
}
```

---

## Common Limit Violations

### Violation 1: Single Large Transaction

**Scenario:** Agent attempts to send 0.5 ETH

**Evaluation:**
- Max single tx: 0.1 ETH
- Requested: 0.5 ETH
- Violation: Exceeds limit by 5x

**Result:** DENIED

**Mitigation:** Split into 5 transactions of 0.1 ETH each (if daily limit allows)

### Violation 2: Unlimited Token Approval

**Scenario:** Agent approves Uniswap for unlimited USDC spend

**Evaluation:**
- Max approval: Transaction amount + 10%
- Requested: Unlimited (type(uint256).max)
- Violation: Exceeds limit

**Result:** DENIED

**Mitigation:** Approve exact amount needed for swap + 10% buffer

### Violation 3: Excessive Retries

**Scenario:** Transaction fails due to low gas, agent retries 5 times

**Evaluation:**
- Max retry attempts: 3
- Current attempts: 5
- Violation: Exceeds limit

**Result:** DENIED after 3rd attempt

**Mitigation:** Investigate failure cause, adjust gas, manually retry if needed

---

## Limit Configuration

Limits are defined in agent configuration:

```json
{
  "safety_limits": {
    "financial": {
      "max_tx_value_eth": 0.1,
      "max_daily_volume_eth": 1.0,
      "max_exposure_eth": 5.0,
      "max_approval_multiplier": 1.1
    },
    "operational": {
      "max_retry_attempts": 3,
      "max_parallel_txs": 1,
      "max_action_duration_seconds": 300
    },
    "security": {
      "whitelist_required": true,
      "max_signatures_per_session": 10,
      "arbitrary_signing_allowed": false
    },
    "economic": {
      "max_slippage_percent": 2.0,
      "max_price_impact_percent": 5.0,
      "max_platform_fee_percent": 5.0
    }
  }
}
```

**Configuration changes require restart.**

---

## Dynamic Limits (Future)

Current limits are static. Future versions MAY support:
- **Adaptive limits** — Adjust based on market conditions
- **Risk-based limits** — Higher limits for lower-risk actions
- **Operator overrides** — Temporary limit increases with approval

**For v0.1.0, all limits are static and hard-coded.**

---

## Summary

**Safety limits are non-negotiable boundaries.**

- ✅ Enforced by governor before every action
- ✅ Prevent single-action catastrophes
- ✅ No runtime overrides
- ✅ Violations → automatic denial

**If you hit a limit frequently → your operations exceed safe parameters. Adjust strategy, not limits.**

---

**See also:**
- `decision-framework.md` — MUST/SHOULD/MAY policies
- `guardrails.md` — Pre-execution checks and fallbacks
- `governor-layer.md` — Decision engine architecture
