# Governor Layer

**The decision engine for autonomous agents on Base.**

---

## Overview

The Governor Layer is the **core of the Base Autonomous Agent Stack**. It is the decision engine that sits between your operational logic and the execution layer.

**Before any action, the governor asks:**
1. Is this action allowed?
2. Does this action violate safety limits?
3. What could go wrong?
4. Do I have authorization?

**If the governor does not explicitly allow an action, the action MUST NOT be taken.**

---

## Deny by Default

The governor operates on a **deny-by-default** posture:

- ❌ If an action is not explicitly allowed → **DENIED**
- ❌ If preconditions are not met → **DENIED**
- ❌ If safety limits are violated → **DENIED**
- ✅ If all checks pass → **ALLOWED**

**There is no "gray area."** The governor either allows or denies. Ambiguity = denial.

---

## Decision Flow

Every action follows this flow:

```
┌─────────────────────┐
│  Action Requested   │
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│  1. Preconditions   │──❌──> DENIED
│  Are they met?      │
└──────────┬──────────┘
           │ ✅
           ▼
┌─────────────────────┐
│  2. Safety Limits   │──❌──> DENIED
│  Are they violated? │
└──────────┬──────────┘
           │ ✅
           ▼
┌─────────────────────┐
│  3. Risk Assessment │──❌──> DENIED
│  Is risk acceptable?│
└──────────┬──────────┘
           │ ✅
           ▼
┌─────────────────────┐
│  4. Authorization   │──❌──> DENIED
│  Do I have approval?│
└──────────┬──────────┘
           │ ✅
           ▼
┌─────────────────────┐
│  ACTION ALLOWED     │
└─────────────────────┘
```

**Each step is a gate.** Fail any gate → action is denied.

---

## 1. Preconditions

**Preconditions** are requirements that must be true before an action can be considered.

### Examples

| Action | Preconditions |
|--------|---------------|
| Send ETH | Wallet balance > amount + gas |
| Swap tokens | Token approval granted |
| Deploy contract | Deployment bytecode validated |
| Pay another agent | Escrow contract exists |

**Rule:** If any precondition is false → **DENY**

### Implementation

Agents MUST check preconditions before invoking the governor:

```
if balance < (amount + estimatedGas):
    DENY("Insufficient balance")
```

**Do not attempt the action hoping it will fail gracefully.** Check first, act second.

---

## 2. Safety Limits

**Safety limits** are hard constraints that MUST NOT be violated.

### Categories

#### A. Transaction Limits

| Limit | Value | Rationale |
|-------|-------|-----------|
| Max gas per tx | 1,000,000 | Prevent runaway execution |
| Max ETH per tx | 0.1 ETH | Prevent catastrophic loss |
| Max token value per tx | $500 USD | Prevent overexposure |
| Max tx per block | 1 | Prevent spam / nonce collision |

#### B. Exposure Limits

| Limit | Value | Rationale |
|-------|-------|-----------|
| Max wallet exposure | 10% of holdings | Prevent single-tx wipeout |
| Max contract interaction | Whitelist only | Prevent malicious contract calls |
| Max approval amount | Exact swap amount | Prevent unlimited approval exploits |

#### C. Temporal Limits

| Limit | Value | Rationale |
|-------|-------|-----------|
| Min time between txs | 10 seconds | Prevent accidental double-spend |
| Max pending txs | 3 | Prevent nonce queue overflow |
| Tx timeout | 5 minutes | Prevent stuck transactions |

**Rule:** If any safety limit is violated → **DENY**

### Hard vs Soft Limits

**All limits are HARD by default.** Violations result in instant denial (MUST NOT).

**Soft limits** (violations = warning + require explicit override) MUST be explicitly configured and documented. If a limit is not documented as soft, it is hard.

**Default:** All limits are hard. Soft limits are the exception, not the rule.

---

## 3. Risk Assessment

**Risk assessment** evaluates what could go wrong and whether it's acceptable.

### Risk Factors

| Factor | Risk Level | Action |
|--------|------------|--------|
| Interacting with unknown contract | HIGH | Deny unless whitelisted |
| Swapping illiquid token | MEDIUM | Require slippage check |
| Sending to new address | LOW | Verify address format |
| Reading onchain data | NONE | Allow |

### Risk Matrix

```
Impact vs Probability:

         LOW     MEDIUM    HIGH
HIGH     ⚠️      ❌        ❌
MEDIUM   ✅      ⚠️        ❌
LOW      ✅      ✅        ⚠️
```

- ✅ Allow
- ⚠️ Warn + require confirmation
- ❌ Deny

**Confirmation defined:** For ⚠️ (warn) cases, "confirmation" means explicit approval from a human operator OR an explicit policy flag configured by the operator. Agents MUST NOT invent their own confirmation mechanisms. If no confirmation mechanism is configured, treat ⚠️ as ❌ (deny).

**Example:**
- **Swapping $10 of ETH for USDC on Uniswap** → Low impact, low probability → ✅ Allow
- **Swapping $1000 of ETH for unknown token** → High impact, medium probability → ❌ Deny
- **Approving unlimited token spend** → High impact, low probability → ❌ Deny

### Mitigation Requirements

For actions in the **⚠️ warn** category, agents MUST:
1. Log the risk explicitly
2. Implement fallback behavior
3. Monitor post-execution state

**If no mitigation exists → treat as ❌ deny.**

---

## 4. Authorization

**Authorization** determines if the agent has permission to take this action.

### Authorization Levels

| Level | Description | Example Actions |
|-------|-------------|-----------------|
| **UNRESTRICTED** | No approval needed | Read blockchain data, check balances |
| **STANDARD** | Agent has standing approval | Swap tokens within limits, send ETH within limits |
| **ELEVATED** | Requires explicit human approval | Deploy contracts, interact with new protocols |
| **PROHIBITED** | Never allowed | Delegate wallet control, approve unlimited spend |

**Note on STANDARD:** Standing approval is bounded by Governor limits and cannot override them. If an action violates a Governor limit (gas, value, exposure), it is denied even if the agent has standing approval.

**Default:** All actions start as **PROHIBITED** until explicitly moved to a higher level.

### Delegation Model

Agents operate under **delegated authority**:
- Human grants agent a scope of allowed actions
- Agent MUST NOT exceed that scope
- Agent MAY sub-delegate to skills (BNKR, OpenClaw) within scope

**If action exceeds scope → DENY and request approval.**

---

## Governor State

The governor maintains state to enforce limits:

### Tracked Metrics

| Metric | Purpose |
|--------|---------|
| `tx_count_last_minute` | Enforce rate limits |
| `pending_tx_nonces` | Prevent nonce collisions |
| `cumulative_gas_used_today` | Enforce daily gas budgets |
| `wallet_exposure_current` | Track % of holdings at risk |
| `last_tx_timestamp` | Enforce minimum time between txs |

### State Reset

State resets on:
- Daily rollover (00:00 UTC) → gas budgets, exposure limits
- Block confirmation → pending tx count
- Manual reset → all state (emergency only)

**State MUST be persisted** across agent restarts.

---

## Guardrails

Guardrails are **automatic safety mechanisms** that activate when things go wrong.

### Pre-Execution Guardrails

| Guardrail | Trigger | Action |
|-----------|---------|--------|
| Balance check | Before any tx | Deny if insufficient |
| Gas estimation | Before tx submission | Deny if > max gas limit |
| Nonce check | Before tx submission | Wait if nonce collision |
| Approval check | Before token operations | Request approval if missing |

### Post-Execution Guardrails

| Guardrail | Trigger | Action |
|-----------|---------|--------|
| Receipt verification | After tx confirmation | Revert if failed |
| Balance reconciliation | After tx confirmation | Alert if unexpected change |
| Gas analysis | After tx confirmation | Flag if > 2x estimate |
| Event log check | After contract interaction | Verify expected events emitted |

### Fallback Behaviors

If a guardrail triggers:
1. **Log the failure** (what, when, why)
2. **Halt execution** (do not proceed)
3. **Alert operator** (if human oversight exists)
4. **Document for learning** (update policies if needed)

**Do NOT retry failed actions without understanding why they failed.**

---

## Integration with Execution Layer

The governor sits **above** the execution layer:

```
Governor (Decision) → Execution (BNKR/OpenClaw)
```

**Flow:**
1. Agent requests action from governor
2. Governor evaluates (preconditions, limits, risk, authorization)
3. If allowed → Governor delegates to execution layer (BNKR skill)
4. Execution layer performs action
5. Governor verifies post-execution state

**The governor NEVER executes directly.** It only decides and delegates.

---

## Example: Token Swap Decision

**Request:** Swap 0.05 ETH for USDC on Uniswap

**Governor evaluation:**

### 1. Preconditions
- ✅ Wallet balance: 0.1 ETH (sufficient)
- ✅ Gas estimate: 150,000 (< 1M limit)
- ✅ Uniswap router whitelisted

### 2. Safety Limits
- ✅ Tx value: 0.05 ETH (< 0.1 ETH limit)
- ✅ Gas: 150,000 (< 1M limit)
- ✅ Exposure: 50% of holdings (< 10% limit? ❌)

**RESULT: DENIED** — Exposure limit violated (swapping 50% of holdings exceeds 10% limit)

**Mitigation:** Reduce swap to 0.01 ETH (10% of holdings) → Re-evaluate → ✅ ALLOWED

---

## Edge Cases

### What if the governor itself fails?

**Fail-safe:** If the governor cannot evaluate → **DENY ALL ACTIONS**

No governor = no actions. This prevents agents from operating without oversight.

### What if limits need emergency adjustment?

**Emergency override:** Human operator can temporarily adjust limits via config update.

**Requirements:**
- Must be logged with reason
- Must have expiration time
- Must revert to defaults after expiration

### What if two actions conflict?

**Conflict resolution:**
1. Evaluate both actions independently
2. If both allowed individually but conflict (e.g., insufficient balance for both) → DENY both
3. Require explicit prioritization from operator

---

## Summary

**The governor is your safety layer.**

- ✅ Deny by default
- ✅ Four-gate evaluation (preconditions, limits, risk, authorization)
- ✅ Hard constraints that MUST NOT be violated
- ✅ Guardrails that activate automatically
- ✅ State tracking for enforcement
- ✅ Integration with execution layer (decide, delegate, verify)

**If you ignore the governor, you will fail.**

---

**Next:** [Execution Layer](execution-layer.md) — How the governor delegates actions to BNKR and OpenClaw
