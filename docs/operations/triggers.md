# Triggers

**When autonomous agents should act.**

---

## Overview

Agents are long-running processes that respond to external and internal stimuli. **Triggers** define when an agent should wake up and evaluate whether to take action.

This document defines trigger patterns for autonomous agent operations.

---

## Core Principle

**Agents are reactive, not proactive.**

Agents do NOT:
- Poll constantly (wastes resources)
- Act without stimulus (no random actions)
- Guess when to act (wait for clear signals)

**Agents wait for triggers, then decide whether to act.**

---

## Trigger Categories

### 1. Event-Based Triggers

**Trigger:** Onchain or offchain event occurs

#### Onchain Events

| Event Type | Example | Agent Response |
|------------|---------|----------------|
| **Payment received** | ETH sent to wallet | Log receipt, update balance |
| **Token transfer** | ERC-20 received | Update portfolio |
| **Contract call** | Function invoked | Execute callback logic |
| **Block mined** | New block on chain | Update state, check conditions |

**Implementation:**
- Subscribe to contract events via WebSocket
- Filter for relevant addresses and topics
- Process events as they arrive

**Example:**
```
Event: Transfer(from, to, amount)
Filter: to == agent_address
Action: Update balance, log transaction
```

#### Offchain Events

| Event Type | Example | Agent Response |
|------------|---------|----------------|
| **HTTP request** | API call received | Process request, return response |
| **Message received** | Telegram message | Parse message, execute command |
| **File change** | Config file updated | Reload configuration |
| **Schedule reached** | Cron job fires | Execute scheduled task |

**Implementation:**
- HTTP server for API requests
- Webhook endpoints for external services
- File watchers for config changes
- Cron scheduler for time-based triggers

### 2. Condition-Based Triggers

**Trigger:** Monitored condition becomes true

#### Price Conditions

| Condition | Example | Agent Response |
|-----------|---------|----------------|
| **Price above threshold** | ETH > $3000 | Consider selling |
| **Price below threshold** | ETH < $2500 | Consider buying |
| **Price change** | ETH moved >5% in 1h | Evaluate market |
| **Volatility spike** | Price variance high | Increase caution |

**Implementation:**
- Poll price feeds periodically (every 60s)
- Calculate thresholds and changes
- Trigger action if condition met

#### Balance Conditions

| Condition | Example | Agent Response |
|-----------|---------|----------------|
| **Balance below minimum** | ETH < 0.01 | Alert to fund wallet |
| **Balance above maximum** | ETH > 10 | Consider withdrawal |
| **Token balance changed** | USDC increased | Update portfolio |

**Implementation:**
- Check balance periodically (every 5 minutes)
- Compare to thresholds
- Trigger alert or action

#### Time Conditions

| Condition | Example | Agent Response |
|-----------|---------|----------------|
| **Deadline approaching** | Proof due in 1 hour | Prioritize submission |
| **Escrow expiring** | Payment expires in 6h | Alert or claim |
| **Scheduled task** | Daily report at 8am | Generate and send |

**Implementation:**
- Track deadlines in database
- Check proximity every minute
- Trigger action when threshold crossed

### 3. Operator-Initiated Triggers

**Trigger:** Human operator sends command

#### Direct Commands

| Command | Example | Agent Response |
|---------|---------|----------------|
| **Execute action** | "Send 0.1 ETH to 0xRecipient" | Evaluate and execute |
| **Query state** | "What's my balance?" | Return current balance |
| **Update config** | "Set max tx to 0.2 ETH" | Update and confirm |
| **Emergency stop** | "Halt all operations" | Stop and await instructions |

**Implementation:**
- CLI interface for local commands
- Telegram/messaging for remote commands
- API endpoints for programmatic control

#### Configuration Changes

| Change | Example | Agent Response |
|--------|---------|----------------|
| **Limits updated** | Max tx value increased | Reload governor limits |
| **Skill added** | New skill installed | Load and validate skill |
| **Whitelist modified** | Contract added | Update authorization list |

**Implementation:**
- File watcher on config files
- Reload configuration on change
- Validate before applying

### 4. Heartbeat Triggers

**Trigger:** Regular intervals for health checks and maintenance

#### Periodic Health Checks

| Frequency | Action | Purpose |
|-----------|--------|---------|
| **Every minute** | Check critical systems | Detect failures quickly |
| **Every 5 minutes** | Check balances | Monitor wallet health |
| **Every 30 minutes** | Check deadlines | Avoid missing proofs |
| **Every hour** | Log metrics | Track performance |
| **Every day** | Generate report | Daily summary |

**Implementation:**
- Cron scheduler for periodic tasks
- Health check functions
- Metric collection and logging

**Example heartbeat:**
```
Every 5 minutes:
  1. Check ETH balance
  2. Check pending transactions
  3. Check escrow deadlines
  4. Log status to dashboard
  5. Return to sleep
```

---

## Trigger Patterns

### Pattern 1: Poll-Then-Decide

**Use case:** Monitoring conditions that change slowly

```
1. Set polling interval (e.g., every 60 seconds)
2. Wake up at interval
3. Query current state
4. Evaluate conditions
5. If condition met → take action
6. Sleep until next interval
```

**Example:** Price monitoring

**Pros:** Simple, predictable
**Cons:** Wastes resources if nothing changes

### Pattern 2: Event-Driven

**Use case:** Responding to specific onchain/offchain events

```
1. Subscribe to event stream
2. Wait for event (blocking)
3. When event arrives → process immediately
4. Return to waiting
```

**Example:** Payment received

**Pros:** Efficient, immediate response
**Cons:** Requires persistent connection

### Pattern 3: Hybrid

**Use case:** Combine polling and events

```
1. Subscribe to high-priority events (payments)
2. Poll low-priority conditions (balance checks)
3. Respond to whichever triggers first
```

**Example:** Monitor payments via events, check deadlines via polling

**Pros:** Balanced efficiency and completeness
**Cons:** More complex to implement

### Pattern 4: Backpressure

**Use case:** Handle burst of triggers

```
1. Triggers arrive faster than agent can process
2. Queue triggers with priority
3. Process queue in order
4. Drop low-priority triggers if queue full
```

**Example:** High-volume trading bot during volatility spike

**Pros:** Prevents overload
**Cons:** May miss some triggers

---

## Trigger Priority

**Not all triggers are equal. Agents SHOULD prioritize:**

| Priority | Trigger Type | Response Time |
|----------|--------------|---------------|
| **Critical** | Emergency stop, wallet compromise | Immediate |
| **High** | Payment received, deadline approaching | <5 seconds |
| **Medium** | Price condition met, balance low | <30 seconds |
| **Low** | Heartbeat check, metric logging | <5 minutes |

**If multiple triggers fire simultaneously → process by priority.**

---

## Trigger Evaluation Flow

**When trigger fires:**

```
1. TRIGGER RECEIVED
   ↓
2. VALIDATE TRIGGER
   - Is trigger authentic?
   - Is trigger duplicate?
   - Is trigger relevant?
   ↓
3. GOVERNOR EVALUATION
   - Should we act on this trigger?
   - Are preconditions met?
   - Are we within limits?
   ↓
4. EXECUTE ACTION (if approved)
   ↓
5. LOG TRIGGER & OUTCOME
```

**Every trigger passes through governor, even if obvious.**

---

## Trigger Configuration

**Agents SHOULD configure triggers:**

```json
{
  "triggers": {
    "events": {
      "payment_received": {
        "enabled": true,
        "contract": "0xEscrow...",
        "event": "PaymentEscrowed",
        "action": "process_payment"
      }
    },
    "conditions": {
      "balance_low": {
        "enabled": true,
        "threshold": "0.01 ETH",
        "check_interval": "5m",
        "action": "alert_operator"
      }
    },
    "heartbeat": {
      "health_check": {
        "enabled": true,
        "interval": "1m",
        "action": "check_systems"
      }
    }
  }
}
```

**Disabled triggers → never fire.**

---

## Example Triggers

### Example 1: Payment Received

**Trigger:** Payment escrowed onchain

```
Event: PaymentEscrowed(requestId, client, worker, amount)
Filter: worker == agent_address
Action:
  1. Verify escrow amount matches expected
  2. Verify client address is known
  3. Update internal accounting
  4. Begin work on request
```

### Example 2: Balance Low

**Trigger:** Wallet balance below threshold

```
Condition: balance < 0.01 ETH
Check interval: Every 5 minutes
Action:
  1. Log warning
  2. Alert operator via Telegram
  3. Pause non-critical operations
  4. Wait for funding
```

### Example 3: Deadline Approaching

**Trigger:** Proof deadline in 1 hour

```
Condition: proof_deadline - current_time < 3600s
Check interval: Every minute
Action:
  1. Prioritize proof submission
  2. Complete work if not done
  3. Submit proof immediately
  4. Log submission
```

### Example 4: Operator Command

**Trigger:** Human sends command via Telegram

```
Message: "/send 0.05 ETH to 0xRecipient"
Validation: Verify sender is authorized operator
Action:
  1. Parse command
  2. Governor evaluates (within limits?)
  3. If approved → execute send
  4. Return confirmation to operator
```

---

## Trigger Logging

**Every trigger MUST be logged:**

```json
{
  "timestamp": "2026-02-03T00:45:00Z",
  "trigger_type": "event",
  "trigger_name": "payment_received",
  "trigger_data": {
    "request_id": "req_abc123",
    "amount": "0.01 ETH",
    "client": "0xClient..."
  },
  "governor_decision": "APPROVED",
  "action_taken": "begin_work",
  "execution_time_ms": 145
}
```

**Logs enable debugging and audit trails.**

---

## Trigger Failure Handling

**What if trigger processing fails?**

### Retry Logic

```
if trigger.type == "transient_failure":
    retry_with_backoff(trigger, max_retries=3)
    
if trigger.type == "permanent_failure":
    log_error(trigger)
    alert_operator()
    do_not_retry()
```

### Dead Letter Queue

**If trigger fails repeatedly:**

```
1. Move to dead letter queue
2. Alert operator
3. Investigate manually
4. Fix issue
5. Replay trigger if needed
```

---

## Trigger Security

### Trigger Authentication

**Agents MUST verify trigger source:**

| Trigger Source | Authentication Method |
|----------------|----------------------|
| **Onchain events** | Verify contract address, signature |
| **API requests** | Verify API key or signature |
| **Operator commands** | Verify operator identity |
| **File changes** | Verify file integrity (checksum) |

**Unauthenticated triggers → ignored.**

### Trigger Rate Limiting

**Prevent trigger spam:**

```
Max triggers per source:
  - Onchain events: 100/minute
  - API requests: 10/minute
  - Operator commands: 5/minute
  
If exceeded → temporarily ignore source, alert operator
```

---

## Summary

**Triggers define when agents act.**

- ✅ Event-based (onchain/offchain events)
- ✅ Condition-based (prices, balances, time)
- ✅ Operator-initiated (commands, config changes)
- ✅ Heartbeat (periodic health checks)
- ✅ Prioritized by criticality
- ✅ Logged for auditability
- ✅ Authenticated for security

**Agents wait for triggers, then let governor decide.**

---

**See also:**
- `monitoring.md` — How to monitor agent health
- `incident-response.md` — What to do when things go wrong
- `governor-layer.md` — Decision engine that evaluates triggers
