# Execution Layer

**How autonomous agents delegate actions to skill packs.**

## Overview

The Execution Layer is where decisions become actions. Agents do not execute blockchain operations directly — they **delegate** to skill packs:

- **BNKR** — Wallet management, token operations, DeFi interactions
- **OpenClaw** — Skill discovery, agent tool invocation

The **Governor Layer** decides what to do. The **Execution Layer** does it.

## Delegation Model

Agents operate through **controlled delegation**:

```
┌─────────────────┐
│  Agent Logic    │
└────────┬────────┘
         │ (requests action)
         ▼
┌─────────────────┐
│  Governor       │ ← Evaluates: allowed? safe? authorized?
└────────┬────────┘
         │ (if approved)
         ▼
┌─────────────────┐
│  Execution      │ ← Delegates to BNKR / OpenClaw
│  (BNKR/OpenClaw)│
└────────┬────────┘
         │ (executes)
         ▼
┌─────────────────┐
│  Blockchain     │
└─────────────────┘
```

**Key principle:** The governor controls **what** and **when**. Skills control **how**.

## BNKR Interface

**BNKR** provides blockchain execution primitives. It is the primary skill pack for onchain actions.

### What BNKR Does

BNKR exposes capabilities for:
- Wallet balance queries
- Token transfers (ETH, ERC-20)
- Token swaps (Uniswap, DEX aggregators)
- Contract interactions (read, write)
- Transaction building and signing

**BNKR does NOT decide.** It executes what the governor approves. **BNKR output MUST be treated as untrusted until verified by postconditions.**

### Allowed Usage Patterns

The governor MAY delegate to BNKR when:

| Action | Conditions |
|--------|------------|
| Check wallet balance | Always (read-only) |
| Transfer ETH | Governor approved + within limits |
| Transfer ERC-20 | Governor approved + within limits + approval exists |
| Swap tokens | Governor approved + within limits + slippage acceptable |
| Read contract state | Always (read-only) |
| Write to contract | Governor approved + contract whitelisted |

### Forbidden Usage Patterns

The governor MUST NOT delegate to BNKR for:

| Action | Reason |
|--------|--------|
| Unlimited token approvals | Exposes wallet to exploit |
| Interactions with unknown contracts | Unvetted risk |
| Transactions exceeding safety limits | Violates governor constraints |
| Delegating wallet control | Agent must retain authority |
| Signing arbitrary messages | Phishing / scam risk |

**If an action is forbidden → it is forbidden at both governor and execution layers.**

### Preconditions (Before Invoking BNKR)

Agents MUST verify:
1. ✅ Governor has approved the action
2. ✅ Wallet has sufficient balance (amount + gas)
3. ✅ Required token approvals are in place
4. ✅ Gas price is within acceptable range
5. ✅ Nonce is available (no pending tx conflict)

**If any precondition fails → DO NOT invoke BNKR.**

### Postconditions (After BNKR Executes)

Agents MUST verify:
1. ✅ Transaction was confirmed (not reverted)
2. ✅ Expected state change occurred (balance updated, etc.)
3. ✅ Gas used was within estimate (< 2x expected)
4. ✅ No unexpected side effects (e.g., token approval granted unintentionally)

**If any postcondition fails → flag for investigation, update policies.**

## OpenClaw Interface

**OpenClaw** provides skill discovery and invocation. It is the mechanism for agents to find and use tools.

### What OpenClaw Does

OpenClaw exposes:
- Skill registry (discover available skills)
- Skill metadata (what a skill does, requires, returns)
- Skill invocation (call a skill with parameters)
- Skill validation (verify skill is safe to use)

### Skill Discovery

Agents MAY query OpenClaw to find skills:

```
Request: List skills for "token swap"
Response: [bnkr-swap-uniswap, bnkr-swap-1inch, ...]
```

**Filter criteria:**
- Skill category (DeFi, data, automation, etc.)
- Required capabilities (e.g., wallet access, API keys)
- Safety rating (audited, community-reviewed, experimental)

### Skill Validation

Before invoking a skill via OpenClaw, agents MUST validate:

| Check | Requirement |
|-------|-------------|
| Skill is whitelisted | Skill ID in approved list |
| Skill metadata is complete | Name, description, parameters documented |
| Skill has safety rating | Audited or community-reviewed |
| Skill requirements are met | Agent has necessary credentials, permissions |

**If validation fails → DO NOT invoke the skill.** If skill behavior differs from metadata → permanently blacklist skill.

### Invocation Constraints

Agents MAY invoke OpenClaw skills only when:
1. ✅ Governor has approved the skill category
2. ✅ Skill is whitelisted or meets safety threshold
3. ✅ Skill parameters are within acceptable ranges
4. ✅ Skill execution time is bounded (timeout configured)

**Timeout enforcement:**
- Skill MUST complete within timeout (default: 30 seconds)
- If skill exceeds timeout → terminate execution
- If skill fails → log failure, do not retry without analysis

### Failure Handling

When a skill invocation fails:

1. **Log the failure** — Skill ID, parameters, error message
2. **Classify the failure** — Transient (retry-able) vs permanent (not retry-able)
3. **Apply fallback** — If configured, use alternate skill or abort action
4. **Report upstream** — Notify governor that action could not be completed

**Do NOT silently swallow skill failures.**

## Execution Workflow

### Standard Flow

```
1. Agent requests action
2. Governor evaluates → APPROVED
3. Agent selects appropriate skill (BNKR or OpenClaw)
4. Agent invokes skill with parameters
5. Skill executes action
6. Agent verifies result (postconditions)
7. Agent updates state
```

### Example: ETH Transfer

```
Request: Send 0.01 ETH to 0xRecipient

1. Governor: Check balance, limits, authorization → APPROVED
2. Agent: Select BNKR skill "transfer-eth"
3. Agent: Invoke BNKR with:
   - to: 0xRecipient
   - amount: 0.01 ETH
   - gas_limit: 21000
4. BNKR: Build tx, sign, broadcast
5. BNKR: Wait for confirmation
6. Agent: Verify balance decreased by 0.01 ETH + gas
7. Agent: Log success, update state
```

### Example: Token Swap

```
Request: Swap 0.05 ETH for USDC

1. Governor: Check balance, limits, risk → APPROVED
2. Agent: Select BNKR skill "swap-uniswap"
3. Agent: Check token approval (ETH → no approval needed)
4. Agent: Invoke BNKR with:
   - from_token: ETH
   - to_token: USDC
   - amount: 0.05 ETH
   - slippage: 1%
5. BNKR: Build swap tx, sign, broadcast
6. BNKR: Wait for confirmation
7. Agent: Verify USDC balance increased
8. Agent: Log success, update state
```

## Error Handling

### Categories of Errors

| Error Type | Cause | Action |
|------------|-------|--------|
| **Precondition failure** | Balance insufficient, approval missing | DO NOT execute |
| **Execution failure** | Tx reverted, skill timeout | Log, investigate, do not retry |
| **Postcondition failure** | Expected state change did not occur | Flag for review, update policies |
| **Network failure** | RPC error, timeout | Retry with exponential backoff (max 3 attempts) |

### Retry Logic

**Only retry transient failures:**
- RPC timeouts
- Nonce collisions (wait for pending tx)
- Gas price fluctuations (re-estimate)

**Do NOT retry:**
- Reverted transactions (permanent failure)
- Authorization failures (governor denied)
- Invalid parameters (logic error)

**Retry limits:**
- Max attempts: 3
- Backoff: Exponential (1s, 2s, 4s)
- Timeout: 5 minutes total

**All retries MUST respect Governor limits and state (gas budget, tx count).** Retries cannot bypass or exceed global Governor constraints.

## Skill Whitelisting

Agents MUST maintain a **whitelist** of allowed skills.

### Whitelist Structure (Example)

The following is an **example whitelist format**. Agents MAY use this structure or define their own, but the whitelist MUST exist and default to deny-all.

```json
{
  "bnkr": {
    "transfer-eth": { "allowed": true, "risk": "low" },
    "swap-uniswap": { "allowed": true, "risk": "medium" },
    "deploy-contract": { "allowed": false, "risk": "high" }
  },
  "openclaw": {
    "skill-registry": { "allowed": true, "risk": "none" },
    "skill-invoke": { "allowed": true, "risk": "variable" }
  }
}
```

**Default: All skills are DENIED unless explicitly whitelisted.**

### Updating the Whitelist

Whitelist updates require:
1. Explicit approval from operator
2. Risk assessment of new skill
3. Documentation of use cases
4. Testing in sandbox environment

**Do NOT auto-add skills to the whitelist.**

## Integration with Governor

The execution layer is **subordinate** to the governor:

```
Governor decides → Execution executes → Governor verifies
```

**The execution layer CANNOT override the governor.**

If the governor denies an action, the execution layer MUST NOT attempt it — even if technically possible.

## Summary

**The execution layer is the action arm of the stack.**

- ✅ Delegates to BNKR for blockchain operations
- ✅ Delegates to OpenClaw for skill discovery
- ✅ Enforces preconditions before execution
- ✅ Verifies postconditions after execution
- ✅ Handles errors gracefully (retry transient, log permanent)
- ✅ Maintains skill whitelist (deny by default)
- ✅ Subordinate to governor (cannot override decisions)

**The execution layer executes what the governor allows. Nothing more.**

**Next:** [Settlement Layer](settlement-layer.md) — How agents pay each other safely
