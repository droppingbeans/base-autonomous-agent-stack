# Decision Framework

**When agents MUST, SHOULD, and MAY act.**

---

## Overview

The decision framework defines **when agents are allowed to take action**. It uses RFC 2119 keywords to prescribe behavior:

- **MUST** / **MUST NOT** — Absolute requirements (violations = failure)
- **SHOULD** / **SHOULD NOT** — Strong recommendations (violations = warnings)
- **MAY** — Optional behaviors (agent discretion)

**This is not guidance. This is policy.**

---

## Decision Hierarchy

Agents evaluate actions in this order:

```
1. Is this action PROHIBITED? → DENY
2. Is this action REQUIRED? → MUST DO
3. Is this action RECOMMENDED? → SHOULD DO (unless good reason not to)
4. Is this action OPTIONAL? → MAY DO (agent decides)
```

**Default:** If an action does not appear in any category → treat as PROHIBITED.

---

## MUST (Required Actions)

Agents MUST take these actions:

### Financial Operations

| Action | Requirement |
|--------|-------------|
| Check balance before sending funds | MUST verify sufficient balance + gas |
| Verify recipient address format | MUST validate checksum before sending |
| Confirm transaction after submission | MUST wait for confirmation, check receipt |
| Log all financial transactions | MUST record amount, recipient, tx hash |

### Safety Checks

| Action | Requirement |
|--------|-------------|
| Validate input parameters | MUST sanitize all user inputs |
| Check safety limits before execution | MUST not violate gas, value, exposure limits |
| Verify postconditions after execution | MUST confirm expected state change occurred |
| Report failures immediately | MUST log and alert on any failure |

### Authorization

| Action | Requirement |
|--------|-------------|
| Respect governor decisions | MUST NOT override governor denials |
| Stay within delegated scope | MUST NOT exceed authorized actions |
| Request approval for elevated actions | MUST get explicit permission for high-risk operations |

---

## MUST NOT (Prohibited Actions)

Agents MUST NOT take these actions:

### Financial Prohibitions

| Action | Reason |
|--------|--------|
| Send funds without governor approval | Unauthorized spending |
| Approve unlimited token spend | Exposes wallet to exploit |
| Interact with unknown contracts | Unvetted risk |
| Exceed transaction limits | Violates safety constraints |
| Retry failed transactions without analysis | May repeat failure or worsen state |

### Operational Prohibitions

| Action | Reason |
|--------|--------|
| Ignore safety limit violations | Defeats purpose of limits |
| Execute actions in parallel without coordination | Risk of nonce collision or double-spend |
| Silently swallow errors | Hides failures, prevents learning |
| Modify governor state directly | Bypasses decision engine |

### Security Prohibitions

| Action | Reason |
|--------|--------|
| Share private keys | Security violation |
| Sign arbitrary messages | Phishing risk |
| Delegate wallet control | Loss of autonomy |
| Disable security checks | Creates vulnerabilities |

---

## SHOULD (Recommended Actions)

Agents SHOULD take these actions unless there is a good reason not to:

### Best Practices

| Action | Rationale |
|--------|-----------|
| Estimate gas before submitting tx | Prevents gas-related failures |
| Use multicall for batch operations | Reduces gas costs and tx count |
| Cache frequently-accessed onchain data | Reduces RPC load and latency |
| Monitor wallet balance regularly | Early warning of depletion |
| Log decision rationale | Helps debug and learn |

### Risk Management

| Action | Rationale |
|--------|-----------|
| Set slippage limits on swaps | Protects against MEV/sandwich attacks |
| Use flashbots for sensitive txs | Prevents front-running |
| Test actions in sandbox first | Validates logic before production |
| Implement fallback behaviors | Graceful degradation on failure |

---

## SHOULD NOT (Discouraged Actions)

Agents SHOULD NOT take these actions unless necessary:

### Anti-Patterns

| Action | Reason |
|--------|--------|
| Retry immediately after failure | May worsen state, lacks investigation |
| Use maximum gas limit as default | Wastes funds |
| Interact with contracts lacking audits | Elevated risk |
| Batch unrelated actions | Makes debugging harder |

**Exception:** If there is a compelling reason (e.g., deadline, emergency), agent MAY override SHOULD NOT with explicit justification.

---

## MAY (Optional Actions)

Agents MAY take these actions at their discretion:

### Optimization

| Action | When to Use |
|--------|-------------|
| Batch low-priority actions | When gas prices are low |
| Use Layer 2 for small transactions | When cost savings matter |
| Cache aggressively | When data changes infrequently |
| Optimize for speed vs cost | Based on urgency |

### Operational Choices

| Action | When to Use |
|--------|-------------|
| Choose between execution paths | When multiple valid options exist |
| Prioritize certain actions over others | When resources are constrained |
| Adjust retry logic | Based on error type and context |

**MAY means:** The agent has discretion. No approval needed, but decision should be logged.

---

## Decision Matrix

### Read vs Write Operations

| Operation Type | Governor Check | Action |
|----------------|----------------|--------|
| **Read-only** (e.g., check balance) | Optional | MAY execute freely |
| **Write (low value)** (e.g., send $1) | Required | MUST get governor approval |
| **Write (high value)** (e.g., send $100) | Required + limits | MUST get approval + stay within limits |
| **Write (critical)** (e.g., deploy contract) | Required + elevated | MUST get explicit human approval |

### Known vs Unknown Entities

| Entity | Action |
|--------|--------|
| **Whitelisted contract** | MAY interact (with governor approval) |
| **Community-vetted contract** | SHOULD verify before interacting |
| **Unknown contract** | MUST NOT interact |
| **Blocklisted contract** | MUST NOT interact under any circumstances |

### Time-Sensitive vs Routine Actions

| Urgency | Approach |
|---------|----------|
| **Emergency** (funds at risk) | MAY bypass some checks with logging |
| **Time-sensitive** (opportunity closing) | SHOULD prioritize speed over perfection |
| **Routine** | MUST follow full decision flow |

**Emergency exceptions MUST be logged and reviewed afterward.**

---

## Example Decisions

### Example 1: Send ETH to Known Address

**Request:** Send 0.01 ETH to 0xRecipient (whitelisted)

**Evaluation:**
1. ✅ MUST check balance → Balance sufficient
2. ✅ MUST validate address → Valid Ethereum address
3. ✅ SHOULD estimate gas → 21,000 gas estimated
4. ✅ MUST get governor approval → Approved (within limits)
5. ✅ MAY proceed

**Result:** ALLOWED

### Example 2: Approve Unlimited Token Spend

**Request:** Approve Uniswap router for unlimited USDC spend

**Evaluation:**
1. ❌ MUST NOT approve unlimited spend → Violates security policy

**Result:** DENIED (no further evaluation needed)

### Example 3: Interact with New DeFi Protocol

**Request:** Deposit ETH into new yield aggregator

**Evaluation:**
1. SHOULD verify contract before interacting → Not audited
2. MUST NOT interact with unknown contracts → Contract not whitelisted
3. Governor denies → Not in authorized scope

**Result:** DENIED

**Exception:** If operator explicitly whitelists the contract → re-evaluate → MAY proceed

---

## Conflict Resolution

### What if MUST conflicts with SHOULD?

**MUST always wins.**

Example:
- MUST NOT exceed tx limit (0.1 ETH)
- SHOULD complete user request (send 0.2 ETH)
- Conflict: Cannot satisfy both

**Resolution:** DENY the action. Explain why to operator.

### What if two SHOULD recommendations conflict?

**Agent uses discretion (MAY decide).**

Example:
- SHOULD optimize for speed (user wants fast execution)
- SHOULD optimize for cost (low gas priority)
- Conflict: Cannot optimize for both simultaneously

**Resolution:** Agent weighs context (e.g., is deadline urgent?) and chooses. Log decision rationale.

---

## Decision Logging

All decisions MUST be logged with:

```json
{
  "timestamp": "2026-02-02T18:30:00Z",
  "action": "send_eth",
  "parameters": {
    "to": "0xRecipient",
    "amount": "0.01 ETH"
  },
  "decision": "ALLOWED",
  "rationale": "Within limits, whitelisted recipient, sufficient balance",
  "checks": {
    "balance_check": "PASS",
    "address_validation": "PASS",
    "safety_limits": "PASS",
    "authorization": "PASS"
  }
}
```

**Logs enable:**
- Debugging failed decisions
- Auditing agent behavior
- Learning from past decisions
- Compliance verification

---

## Updating the Framework

Decision policies MAY be updated when:
1. New failure modes are discovered
2. New protocols/contracts are vetted
3. Risk models are refined
4. Operator explicitly approves changes

**Policy updates MUST:**
- Document rationale
- Version the policy
- Notify affected agents
- Provide migration path

**See:** `governance/change-process.md` (🚧 Planned)

---

## Summary

**The decision framework prescribes agent behavior.**

- ✅ MUST = absolute requirements (no exceptions)
- ✅ MUST NOT = absolute prohibitions (no exceptions)
- ✅ SHOULD = strong recommendations (override with justification)
- ✅ SHOULD NOT = discouraged (avoid unless necessary)
- ✅ MAY = optional (agent discretion)

**Default: Deny unless explicitly allowed.**

**All decisions MUST be logged for auditability.**
