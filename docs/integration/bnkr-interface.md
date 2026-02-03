# BNKR Interface

**How agents delegate wallet operations and onchain execution to BNKR.**

---

## Overview

**BNKR** (Bankr) is an AI-powered crypto trading agent that provides wallet operations, token trading, DeFi interactions, and portfolio management. Agents use BNKR to execute onchain actions without implementing blockchain logic directly.

**This document defines interface expectations, not BNKR's implementation.**

---

## Canonical BNKR Skill Registry

**This repository defines governance and interface expectations only.**

BNKR implementations live externally. The canonical BNKR and OpenClaw skills repository is maintained at:

**https://github.com/BankrBot/openclaw-skills**

That repository contains:
- BNKR skill implementations
- OpenClaw skill ecosystem
- Integration examples and tooling

**Availability ≠ authorization. The Governor Layer still decides.**

BNKR skills being available does not mean agents are authorized to use them. Governor evaluation, safety limits, and authorization checks remain mandatory.

---

## Delegation Pattern

Agents do NOT execute onchain actions directly. Actions are **delegated** to BNKR:

```
Agent Logic
    ↓ (decides: "send 0.01 ETH")
Governor
    ↓ (approves: "allowed, within limits")
BNKR Skill
    ↓ (executes: transaction submission)
Blockchain
```

**BNKR is a tool, not a decision maker.**

---

## Core Principle

**Agents MUST treat BNKR as an untrusted execution module.**

BNKR provides capabilities, but:
- BNKR does NOT make decisions → Governor decides
- BNKR does NOT override limits → Governor enforces
- BNKR does NOT handle authorization → Governor authorizes

**The governor defines WHEN and WHY BNKR may be invoked. BNKR defines HOW.**

---

## BNKR Capabilities

### 1. Wallet Operations

| Operation | Description | Governor Check Required |
|-----------|-------------|-------------------------|
| **Check balance** | Query wallet balance | Optional (read-only) |
| **Send ETH** | Transfer native token | Required |
| **Send ERC-20** | Transfer token | Required |
| **Approve token** | Set allowance | Required |
| **Check allowance** | Query token approval | Optional (read-only) |

**Read operations MAY bypass governor. Write operations MUST be approved.**

### 2. Token Trading

| Operation | Description | Governor Check Required |
|-----------|-------------|-------------------------|
| **Swap tokens** | Trade on DEX | Required |
| **Buy token** | Market buy | Required |
| **Sell token** | Market sell | Required |
| **Check price** | Query token price | Optional (read-only) |
| **Get quote** | Estimate swap output | Optional (read-only) |

**All trades MUST pass through governor evaluation.**

### 3. DeFi Interactions

| Operation | Description | Governor Check Required |
|-----------|-------------|-------------------------|
| **Provide liquidity** | Add to LP | Required + elevated |
| **Remove liquidity** | Withdraw from LP | Required |
| **Stake tokens** | Deposit to staking | Required |
| **Unstake tokens** | Withdraw from staking | Required |
| **Claim rewards** | Harvest yields | Required |

**DeFi operations are elevated risk → stricter governor checks.**

### 4. Portfolio Management

| Operation | Description | Governor Check Required |
|-----------|-------------|-------------------------|
| **List positions** | View all holdings | Optional (read-only) |
| **Get portfolio value** | Total value calculation | Optional (read-only) |
| **Transaction history** | View past transactions | Optional (read-only) |
| **Performance metrics** | Calculate gains/losses | Optional (read-only) |

**Portfolio queries are informational → no governor approval needed.**

---

## Interface Expectations

### Request Format

When invoking BNKR, agents MUST provide:

**Minimum required fields:**
```json
{
  "operation": "send_eth",
  "parameters": {
    "to": "0xRecipient",
    "amount": "0.01"
  },
  "request_id": "req_abc123",
  "agent_id": "agent_xyz"
}
```

**Optional fields:**
```json
{
  "gas_limit": 21000,
  "gas_price": "5 gwei",
  "deadline": "2026-02-03T01:00:00Z",
  "slippage": 0.02
}
```

### Response Format

**Agents SHOULD normalize BNKR responses into this canonical shape for logging and audit.**

BNKR implementations may vary in their native response format. Agents are responsible for adapting responses to this standard structure:

**Success response (canonical):**
```json
{
  "status": "success",
  "transaction": {
    "hash": "0xabc...",
    "from": "0xAgent",
    "to": "0xRecipient",
    "value": "0.01 ETH",
    "gas_used": 21000,
    "block": 12345678,
    "confirmed": true
  },
  "request_id": "req_abc123"
}
```

**Error response:**
```json
{
  "status": "error",
  "error": {
    "code": "INSUFFICIENT_BALANCE",
    "message": "Balance: 0.005 ETH, Required: 0.01105 ETH",
    "details": {
      "available": "0.005",
      "required": "0.01105",
      "shortfall": "0.00605"
    }
  },
  "request_id": "req_abc123"
}
```

---

## Governor Integration

### Preconditions (Before BNKR Invocation)

Governor MUST verify:

1. **Action is allowed** → Operation is in authorized scope
2. **Parameters are valid** → Address format, amount range, gas limits
3. **Safety limits respected** → Within max tx value, daily volume
4. **Balance sufficient** → Wallet can cover amount + gas
5. **No pending conflicts** → No nonce collision risk

**If any precondition fails → DENY invocation.**

### Invocation Pattern

```
1. Agent requests action
2. Governor evaluates preconditions
3. If approved → Delegate to BNKR
4. BNKR executes and returns result
5. Governor verifies postconditions
6. Agent receives final result
```

**Governor is in control, not BNKR.**

### Postconditions (After BNKR Execution)

Governor MUST verify:

1. **Transaction succeeded** → Receipt status is success
2. **Expected state change** → Balance changed as expected
3. **Gas within bounds** → Gas used is reasonable
4. **Events emitted** → Expected events present in logs
5. **No side effects** → No unexpected state changes

**If postconditions fail → Alert and investigate.**

---

## Error Handling

### BNKR Error Categories

| Error Type | Cause | Governor Action |
|------------|-------|-----------------|
| **INSUFFICIENT_BALANCE** | Wallet balance too low | Deny, alert to fund wallet |
| **INVALID_PARAMETER** | Bad address or amount | Deny, fix parameter |
| **TRANSACTION_FAILED** | Reverted onchain | Log failure, investigate |
| **NETWORK_ERROR** | RPC unavailable | Retry with backoff |
| **TIMEOUT** | Execution took too long | Cancel, log timeout |
| **RATE_LIMIT** | Too many requests | Wait and retry |

### Recovery Patterns

**For transient errors (network, rate limits):**
- Retry with exponential backoff
- Max 3 attempts
- If still failing → halt and alert

**For permanent errors (invalid params, insufficient balance):**
- Do NOT retry
- Log error details
- Alert operator

**For transaction reverts:**
- Analyze revert reason
- Determine if fixable (adjust gas, slippage)
- If not fixable → deny action permanently

---

## Security Considerations

### Execution Wallet Isolation

**Agents SHOULD isolate execution funds in a dedicated execution wallet (often BNKR-controlled), separate from a main/cold wallet.**

This pattern provides:
- Blast radius containment (compromised execution wallet does not expose main holdings)
- Different risk profiles (execution wallet has lower limits)
- Simplified accounting and monitoring

**Recommended setup:**
- Main wallet: Long-term holdings, cold storage
- Execution wallet: Day-to-day operations, BNKR has access
- Fund execution wallet periodically from main wallet
- Never share private keys between wallets

### Spending Limits

**Agents SHOULD configure execution wallet limits to be stricter than governor limits:**

| Limit | Recommendation | Configuration Location |
|-------|----------------|------------------------|
| **Max tx value** | Lower than governor max | Agent config / governor |
| **Daily spend cap** | Lower than governor cap | Agent config / governor |
| **Allowed contracts** | Whitelist only | Agent config / governor |

**Defense-in-depth: Multiple layers of limits provide redundancy.**

If both governor and execution wallet have limits, the stricter limit applies. This protects against misconfigurations or bugs in either layer.

### Signature Verification

**All BNKR responses SHOULD be signed:**

```json
{
  "status": "success",
  "transaction": {...},
  "signature": "0xabc...",
  "signer": "0xBNKR..."
}
```

Agent SHOULD verify:
- Signature is valid
- Signer is expected BNKR address
- Response has not been tampered with

**Do NOT trust unsigned BNKR responses in production.**

---

## Allowed Operations

### Default Whitelist

Operations agents MAY invoke via BNKR:

**Always allowed (with governor approval):**
- Check balance
- Send ETH (within limits)
- Send ERC-20 (within limits)
- Approve token (limited amount only)
- Swap tokens (whitelisted DEXs only)

**Conditionally allowed (elevated approval):**
- Provide liquidity (requires human approval)
- Deploy contracts (requires human approval)
- Sign arbitrary messages (DENIED by default)

**Never allowed:**
- Unlimited token approvals
- Interactions with unknown contracts
- Private key export
- Seed phrase exposure

### Operation Scope Configuration

Agents SHOULD configure operation scopes:

```json
{
  "bnkr_allowed_operations": {
    "send_eth": { "allowed": true, "max_value": "0.1 ETH" },
    "send_erc20": { "allowed": true, "max_value": "100 USDC" },
    "swap": { "allowed": true, "max_slippage": 0.02 },
    "approve": { "allowed": true, "max_multiplier": 1.1 },
    "provide_liquidity": { "allowed": false },
    "deploy_contract": { "allowed": false }
  }
}
```

**Operations not in whitelist → automatically denied.**

---

## BNKR vs Main Wallet

**Agents SHOULD maintain two wallets:**

| Wallet | Purpose | Private Key Location | Limits |
|--------|---------|---------------------|--------|
| **Main wallet** | Long-term holdings, elevated actions | Secure vault | High |
| **BNKR wallet** | Day-to-day operations, trading | BNKR has access | Lower |

**Benefits:**
- Limits blast radius if BNKR wallet compromised
- Allows different risk profiles
- Simplifies accounting and monitoring

**Fund flow:**
```
Main Wallet → BNKR Wallet (periodic funding)
BNKR Wallet → Operations (daily trading, swaps)
BNKR Wallet → Main Wallet (profits, withdrawals)
```

---

## Monitoring and Logging

### Required Logs

Every BNKR invocation MUST be logged:

```json
{
  "timestamp": "2026-02-03T00:30:00Z",
  "request_id": "req_abc123",
  "operation": "send_eth",
  "parameters": {
    "to": "0xRecipient",
    "amount": "0.01 ETH"
  },
  "result": {
    "status": "success",
    "tx_hash": "0xabc...",
    "gas_used": 21000
  },
  "governor_decision": "APPROVED",
  "execution_time_ms": 1234
}
```

### Monitoring Metrics

Track BNKR usage:

| Metric | Purpose |
|--------|---------|
| **Success rate** | % of successful operations |
| **Average execution time** | Latency monitoring |
| **Gas efficiency** | Actual gas vs estimates |
| **Error frequency** | Failure pattern detection |
| **Daily spend** | Budget tracking |

**Alert if:**
- Success rate drops below 95%
- Execution time exceeds 30s
- Gas usage exceeds estimates by >20%
- Error rate spikes above 10%

---

## Example Invocation Flow

**Scenario:** Agent wants to swap 0.1 ETH for USDC

### Step 1: Governor Pre-Check

```
Action: Swap 0.1 ETH for USDC
Checks:
  ✓ Operation allowed (swap in whitelist)
  ✓ Parameters valid (amount within limits)
  ✓ Balance sufficient (0.15 ETH available)
  ✓ Slippage acceptable (1% within 2% max)
Result: APPROVED
```

### Step 2: BNKR Invocation

```json
{
  "operation": "swap",
  "parameters": {
    "from_token": "ETH",
    "to_token": "USDC",
    "amount": "0.1",
    "slippage": 0.01,
    "dex": "uniswap"
  },
  "request_id": "req_swap_001"
}
```

### Step 3: BNKR Execution

```
BNKR:
1. Estimates gas: 150,000
2. Simulates swap: expected output ~300 USDC
3. Submits transaction to Uniswap
4. Waits for confirmation
5. Returns result
```

### Step 4: Governor Post-Check

```
Transaction: 0xabc...
Checks:
  ✓ Receipt status: success
  ✓ Balance change: -0.1015 ETH (including gas)
  ✓ USDC received: 298 USDC (within slippage)
  ✓ Events: "Swap" event present
Result: VERIFIED
```

### Step 5: Agent Receives Result

```json
{
  "status": "success",
  "transaction": {
    "hash": "0xabc...",
    "from_token": "ETH",
    "to_token": "USDC",
    "amount_in": "0.1 ETH",
    "amount_out": "298 USDC",
    "gas_used": 142000
  },
  "request_id": "req_swap_001"
}
```

**Swap complete. Agent now has 298 USDC.**

---

## Summary

**BNKR is a delegated execution module, not a decision engine.**

- ✅ Governor decides when to invoke BNKR
- ✅ Governor enforces limits and safety checks
- ✅ BNKR executes but does not authorize
- ✅ All operations logged and monitored
- ✅ Separate wallet for BNKR operations
- ✅ Strict operation whitelisting

**BNKR capabilities are powerful. Governor oversight is mandatory.**

---

---

**See also:**
- [`../architecture/governor-layer.md`](../architecture/governor-layer.md) — Decision engine architecture
- [`../architecture/execution-layer.md`](../architecture/execution-layer.md) — Delegation patterns
- [`../policies/safety-limits.md`](../policies/safety-limits.md) — Hard constraints on operations
- [`../policies/guardrails.md`](../policies/guardrails.md) — Pre/post-execution checks
