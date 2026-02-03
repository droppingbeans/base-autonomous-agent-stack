# Guardrails

**Pre-execution checks, post-execution verification, and fallback behaviors.**

---

## Overview

Guardrails are **defensive checks** that run before and after actions. They:
- **Prevent** — Catch invalid actions before execution
- **Verify** — Confirm expected outcomes after execution
- **Recover** — Provide fallback when things go wrong

**Guardrails are the safety net between decision and action.**

---

## Why Guardrails Matter

Without guardrails, agents:
- Execute actions with invalid parameters
- Proceed without verifying success
- Fail silently and corrupt state
- Have no recovery path when failures occur

**Guardrails turn failures from catastrophes into recoverable events.**

---

## Guardrail Categories

### 1. Pre-Execution Checks

Run BEFORE action is executed. If check fails → action is DENIED.

#### Balance Verification

**Check:** Does wallet have sufficient balance for action + gas?

```
Required: action_value + (gas_estimate × gas_price)
Available: wallet_balance
Pass: available >= required
Fail: DENY action, log "insufficient balance"
```

**Example:**
```
Action: Send 0.05 ETH
Gas estimate: 21,000 gas @ 5 gwei = 0.000105 ETH
Required: 0.05 + 0.000105 = 0.050105 ETH
Balance: 0.045 ETH
Result: DENIED (insufficient balance)
```

#### Parameter Validation

**Check:** Are all parameters within valid ranges?

| Parameter | Validation | Action if Invalid |
|-----------|------------|-------------------|
| **Address** | Valid checksum, not zero address | DENY |
| **Amount** | Positive, within limits | DENY |
| **Gas** | Within min/max range | DENY |
| **Deadline** | Future timestamp | DENY |
| **Slippage** | 0-100% | DENY |

**Example:**
```
Action: Swap ETH for USDC
Slippage: 15%
Max slippage limit: 2%
Result: DENIED (slippage exceeds limit)
```

#### Authorization Check

**Check:** Is agent authorized to perform this action?

```
if action.type in authorized_actions:
    PASS
else:
    DENY, log "unauthorized action"
```

**Authorization levels:**
- **Routine** — Whitelisted, auto-approved (e.g., balance checks)
- **Standard** — Requires governor approval (e.g., token transfers)
- **Elevated** — Requires human approval (e.g., contract deployments)

#### State Validation

**Check:** Is current state compatible with action?

| State Check | Condition | Action if False |
|-------------|-----------|-----------------|
| **Nonce available** | No pending txs with same nonce | Wait or deny |
| **Contract exists** | Target address is contract | Deny if required |
| **Allowance set** | Token approval exists if needed | Set allowance first |
| **Market open** | Trading hours if applicable | Delay until open |

**Example:**
```
Action: Swap on DEX
State check: DEX contract exists at address
Result: Contract verified ✓
Proceed: Yes
```

### 2. Simulation Checks

Run action in simulation BEFORE executing onchain.

#### Gas Estimation

**Check:** Estimate gas required for transaction

```
estimated_gas = simulate_transaction(action)
if estimated_gas > max_gas_limit:
    DENY, log "gas estimate exceeds limit"
else:
    PASS
```

**Example:**
```
Action: Complex DeFi interaction
Estimated gas: 650,000
Max gas limit: 500,000
Result: DENIED (gas estimate too high)
```

#### Revert Prediction

**Check:** Will transaction revert?

```
try:
    simulate_transaction(action)
    PASS
except Revert(reason):
    DENY, log f"simulation reverted: {reason}"
```

**Common revert reasons:**
- Insufficient balance
- Slippage exceeded
- Deadline passed
- Allowance not set

#### Outcome Prediction

**Check:** Will action produce expected state change?

```
before_state = get_state()
simulate_transaction(action)
after_state = get_state()

if after_state matches expected_state:
    PASS
else:
    DENY, log "unexpected outcome in simulation"
```

**Example:**
```
Action: Swap 0.1 ETH for USDC
Expected: Receive ~300 USDC
Simulated: Receive 295 USDC
Result: PASS (within slippage tolerance)
```

### 3. Post-Execution Verification

Run AFTER action is executed. If verification fails → alert and potentially rollback.

#### Receipt Verification

**Check:** Was transaction confirmed onchain?

```
receipt = wait_for_receipt(tx_hash, timeout=60s)
if receipt.status == 1:
    PASS
else:
    FAIL, log "transaction reverted", attempt recovery
```

**Verification steps:**
1. Wait for tx confirmation (up to timeout)
2. Check receipt status (1 = success, 0 = revert)
3. Verify gas used is within expected range
4. Check for expected events emitted

#### State Verification

**Check:** Did action produce expected state change?

```
expected_balance_change = -0.1 ETH
actual_balance_change = new_balance - old_balance

if abs(actual_balance_change - expected_balance_change) < tolerance:
    PASS
else:
    FAIL, log "unexpected state change", investigate
```

**Example:**
```
Action: Send 0.05 ETH to recipient
Expected: Balance decreases by ~0.050105 ETH (including gas)
Actual: Balance decreased by 0.050112 ETH
Difference: 0.000007 ETH (acceptable)
Result: PASS
```

#### Event Verification

**Check:** Were expected events emitted?

```
expected_events = ["Transfer", "Swap"]
emitted_events = receipt.logs

for event in expected_events:
    if event not in emitted_events:
        FAIL, log f"missing expected event: {event}"
```

**Example:**
```
Action: Token swap on Uniswap
Expected events: ["Swap", "Sync"]
Emitted events: ["Swap", "Sync"]
Result: PASS (all expected events present)
```

#### Postcondition Checks

**Check:** Are postconditions satisfied?

| Postcondition | Check | Action if False |
|---------------|-------|-----------------|
| **Balance updated** | New balance matches expected | Alert, investigate |
| **Allowance consumed** | Allowance decreased if used | Log warning |
| **Nonce incremented** | Nonce increased by 1 | Alert (stuck nonce) |
| **Position opened** | New position exists in records | Alert, sync state |

### 4. Fallback Behaviors

What to do when checks fail or actions don't execute as expected.

#### Retry with Backoff

**Trigger:** Transaction fails due to transient issue (network error, low gas price)

**Fallback:**
```
for attempt in 1..max_retries:
    result = execute_action()
    if result.success:
        return result
    else:
        wait(exponential_backoff(attempt))
```

**Backoff schedule:**
- Attempt 1: Immediate
- Attempt 2: Wait 1s
- Attempt 3: Wait 2s
- Attempt 4: Wait 4s (if max_retries = 3, stop here)

**Do NOT retry if:**
- Governor denied action
- Safety limit violated
- Authorization failed

#### Graceful Degradation

**Trigger:** Primary execution path fails

**Fallback:** Use simpler/safer alternative

**Example 1: Gas price spike**
```
Primary: Execute swap immediately
Fallback: Wait for gas price to drop below threshold
```

**Example 2: DEX unavailable**
```
Primary: Swap on Uniswap
Fallback: Swap on alternative DEX (if whitelisted)
```

**Example 3: RPC failure**
```
Primary: Use Alchemy RPC
Fallback: Use public Base RPC
```

#### State Rollback

**Trigger:** Action executed but postconditions failed

**Fallback:** Reverse action if possible

**Example:**
```
Action: Approve token spend
Executed: Approval set to 1000 USDC
Postcondition check: FAILED (unexpected event)
Rollback: Set approval back to 0
```

**Rollback is NOT always possible:**
- Funds already sent (irreversible)
- Contract state changed externally
- Rollback would exceed gas limits

**If rollback not possible → alert operator, log full context.**

#### Alert and Halt

**Trigger:** Unrecoverable failure or unexpected state

**Fallback:** Stop all operations, alert operator

**Example:**
```
Action: Complex multi-step operation
Step 3: FAILED with unknown error
Rollback: NOT possible (prior steps succeeded)
Fallback: HALT operations, alert operator, log full state
```

**Halt conditions:**
- Multiple consecutive failures
- State corruption detected
- Security violation suspected
- Wallet balance drain detected

---

## Guardrail Implementation Pattern

**Every action follows this flow:**

```
1. PRE-EXECUTION CHECKS
   ├─ Balance verification
   ├─ Parameter validation
   ├─ Authorization check
   ├─ State validation
   └─ Simulation (gas estimation, revert prediction)
   
   If any check fails → DENY action

2. EXECUTE ACTION
   └─ Submit transaction onchain

3. POST-EXECUTION VERIFICATION
   ├─ Receipt verification
   ├─ State verification
   ├─ Event verification
   └─ Postcondition checks
   
   If any verification fails → ALERT & INVESTIGATE

4. FALLBACK (if needed)
   ├─ Retry with backoff
   ├─ Graceful degradation
   ├─ State rollback
   └─ Alert and halt
```

---

## Guardrail Logging

All guardrail checks MUST be logged:

```json
{
  "timestamp": "2026-02-03T00:00:00Z",
  "action": "send_eth",
  "phase": "pre-execution",
  "checks": {
    "balance_verification": "PASS",
    "parameter_validation": "PASS",
    "authorization_check": "PASS",
    "state_validation": "PASS",
    "simulation": {
      "gas_estimate": 21000,
      "revert_prediction": "PASS",
      "outcome_prediction": "PASS"
    }
  },
  "decision": "APPROVED"
}
```

**Post-execution log:**
```json
{
  "timestamp": "2026-02-03T00:00:05Z",
  "action": "send_eth",
  "phase": "post-execution",
  "tx_hash": "0xabc...",
  "verification": {
    "receipt_verified": true,
    "state_verified": true,
    "events_verified": true,
    "postconditions": "PASS"
  },
  "result": "SUCCESS"
}
```

---

## Common Guardrail Failures

### Failure 1: Insufficient Balance

**Pre-check:** Balance verification  
**Cause:** Wallet balance too low  
**Fallback:** Deny action, alert operator to fund wallet

### Failure 2: Transaction Revert

**Simulation:** Revert prediction  
**Cause:** Slippage exceeded in simulation  
**Fallback:** Adjust slippage or wait for better price

### Failure 3: Unexpected State Change

**Post-check:** State verification  
**Cause:** Received less tokens than expected  
**Fallback:** Log discrepancy, investigate contract behavior

### Failure 4: Missing Event

**Post-check:** Event verification  
**Cause:** Expected "Transfer" event not emitted  
**Fallback:** Alert operator, manual verification required

---

## Guardrail Configuration

Guardrails are configured per agent:

```json
{
  "guardrails": {
    "pre_execution": {
      "balance_check": true,
      "parameter_validation": true,
      "authorization_check": true,
      "simulation_required": true
    },
    "post_execution": {
      "receipt_verification": true,
      "state_verification": true,
      "event_verification": true,
      "verification_timeout_seconds": 60
    },
    "fallback": {
      "retry_enabled": true,
      "max_retry_attempts": 3,
      "backoff_strategy": "exponential",
      "graceful_degradation": true,
      "rollback_enabled": true,
      "halt_on_critical_failure": true
    }
  }
}
```

---

## Guardrails vs Safety Limits

**Safety Limits:** Hard boundaries (MUST NOT exceed)  
**Guardrails:** Defensive checks (SHOULD verify)

| Concept | Type | Example |
|---------|------|---------|
| **Safety Limit** | Absolute constraint | Max tx value: 0.1 ETH |
| **Guardrail** | Verification step | Check balance before send |

**Both work together:**
1. Guardrail checks balance (pre-execution)
2. Safety limit enforces max tx value (governor)
3. Guardrail verifies expected balance change (post-execution)

---

## Summary

**Guardrails prevent, verify, and recover.**

- ✅ Pre-execution checks catch invalid actions before they execute
- ✅ Simulation predicts failures before spending gas
- ✅ Post-execution verification confirms expected outcomes
- ✅ Fallback behaviors provide recovery paths

**Every action MUST pass guardrails. No exceptions.**

---

**See also:**
- `safety-limits.md` — Hard constraints
- `decision-framework.md` — MUST/SHOULD/MAY policies
- `governor-layer.md` — Decision engine that enforces guardrails
