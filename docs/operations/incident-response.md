# Incident Response

**What to do when things go wrong.**

---

## Overview

Autonomous agents handle value and execute transactions. **Incidents** are inevitable:
- Bugs in logic
- Network failures
- Security compromises
- Operator errors
- External attacks

**Incident response defines how to detect, contain, resolve, and learn from failures.**

---

## Core Principle

**Speed and containment matter more than perfection.**

When an incident occurs:
1. **Detect** — Know something is wrong
2. **Triage** — Assess severity
3. **Contain** — Prevent further damage
4. **Investigate** — Understand root cause
5. **Resolve** — Fix the issue
6. **Post-mortem** — Learn and improve

**Act fast. Fix thoroughly.**

---

## Incident Severity Levels

### Severity 1: Critical

**Definition:** Agent security compromised or significant funds at risk

**Examples:**
- Private key compromised
- Wallet draining rapidly
- Smart contract exploit detected
- Unauthorized access gained

**Response time:** Immediate (within 5 minutes)

**Actions:**
1. **HALT ALL OPERATIONS**
2. Disconnect from network
3. Preserve logs and state
4. Assess damage
5. Notify all stakeholders
6. Execute recovery plan

### Severity 2: High

**Definition:** Agent unable to operate or degraded performance affecting critical functions

**Examples:**
- Process crashed and won't restart
- All transactions failing
- Governor logic error causing denials
- Balance depleted (cannot operate)

**Response time:** <30 minutes

**Actions:**
1. Alert operator
2. Attempt automated recovery
3. If recovery fails → manual intervention
4. Restore from backup if needed
5. Investigate cause

### Severity 3: Medium

**Definition:** Degraded performance or non-critical failures

**Examples:**
- High transaction failure rate (but some succeed)
- Slow response times
- Monitoring alerts firing
- Non-critical module failure

**Response time:** <2 hours

**Actions:**
1. Log incident details
2. Attempt fixes without downtime
3. Monitor for improvement
4. Schedule deeper investigation

### Severity 4: Low

**Definition:** Minor issues with minimal impact

**Examples:**
- Single transaction failure
- Temporary network blip
- Dashboard not updating
- Non-critical log errors

**Response time:** <24 hours

**Actions:**
1. Log for review
2. Fix during regular maintenance
3. No immediate action required

---

## Incident Detection

### Automated Detection

**Monitoring systems SHOULD automatically detect:**

| Incident Type | Detection Method |
|---------------|------------------|
| **Process crash** | Heartbeat timeout |
| **Wallet drainage** | Balance drop alert |
| **Transaction failures** | Failure rate threshold |
| **Security violation** | Unauthorized action attempt |
| **Performance degradation** | Latency threshold exceeded |
| **Network issues** | RPC connection failure |

**When detected → trigger incident response.**

### Manual Detection

**Operators may detect incidents via:**
- Dashboard anomalies
- User reports
- Failed operations
- Unexpected behavior

**Operators MUST have incident reporting channel (e.g., Telegram command: `/incident`).**

---

## Incident Response Playbooks

### Playbook 1: Wallet Compromise

**Scenario:** Private key may be compromised

**Steps:**

1. **Immediate (0-5 minutes):**
   - Execute kill switch: halt all operations
   - Disconnect agent from network
   - Transfer remaining funds to cold wallet (if possible)
   - Revoke all token approvals
   - Document time of compromise

2. **Short-term (5-30 minutes):**
   - Assess damage: what funds were taken?
   - Identify attack vector: how was key compromised?
   - Notify affected parties (escrow partners, etc.)
   - Generate new wallet
   - Update all systems with new address

3. **Long-term (30+ minutes):**
   - Investigate root cause
   - Implement additional security measures
   - Update incident log
   - Write post-mortem
   - Communicate resolution

**Prevention:**
- Store keys in secure vault
- Never log private keys
- Use hardware wallet when possible
- Rotate keys periodically

### Playbook 2: Stuck Transactions

**Scenario:** Multiple transactions stuck pending

**Steps:**

1. **Immediate (0-5 minutes):**
   - Identify nonce of first stuck transaction
   - Check gas price vs current network
   - Determine if RPC issue or gas issue

2. **Short-term (5-30 minutes):**
   - If low gas: submit replacement tx with higher gas
   - If nonce collision: cancel conflicting tx
   - If RPC issue: switch to backup RPC
   - Monitor for confirmation

3. **Long-term (30+ minutes):**
   - If still stuck: submit cancel transaction (same nonce, 0 value, high gas)
   - Wait for one to confirm
   - Resume normal operations
   - Investigate why txs got stuck

**Prevention:**
- Use dynamic gas pricing (based on network)
- Set reasonable gas limits
- Monitor pending txs actively
- Implement nonce management

### Playbook 3: Process Crash

**Scenario:** Agent process terminated unexpectedly

**Steps:**

1. **Immediate (0-5 minutes):**
   - Check if crash loop (restart failing repeatedly)
   - Review crash logs for error
   - Assess if safe to restart

2. **Short-term (5-30 minutes):**
   - If safe: restart with monitoring
   - If unsafe: investigate logs first
   - Verify wallet state is consistent
   - Check for pending operations

3. **Long-term (30+ minutes):**
   - If crash persists: rollback to previous version
   - Fix bug causing crash
   - Test fix in staging
   - Deploy corrected version

**Prevention:**
- Comprehensive error handling
- Process monitoring with auto-restart
- State persistence (survive crashes)
- Regular testing and QA

### Playbook 4: High Transaction Failure Rate

**Scenario:** >20% of transactions failing

**Steps:**

1. **Immediate (0-5 minutes):**
   - Check network status (is Base down?)
   - Review failure reasons (gas, revert, etc.)
   - Determine if issue is agent or external

2. **Short-term (5-30 minutes):**
   - If network congestion: increase gas prices
   - If contract issue: blocklist failing contract
   - If agent bug: fix parameter validation
   - Pause non-critical operations

3. **Long-term (30+ minutes):**
   - Monitor for improvement
   - Resume normal operations when stable
   - Analyze failed txs for patterns
   - Update logic to prevent recurrence

**Prevention:**
- Simulate transactions before submission
- Validate parameters thoroughly
- Monitor contract availability
- Use fallback strategies

### Playbook 5: Balance Depleted

**Scenario:** Wallet balance too low to operate

**Steps:**

1. **Immediate (0-5 minutes):**
   - Alert operator to fund wallet
   - Pause all non-essential operations
   - Prioritize critical actions only

2. **Short-term (5-30 minutes):**
   - Transfer funds from backup wallet
   - Verify balance updated
   - Resume normal operations

3. **Long-term (30+ minutes):**
   - Investigate why balance depleted faster than expected
   - Adjust budget or funding schedule
   - Implement better balance monitoring

**Prevention:**
- Set balance alert thresholds
- Auto-fund from reserve wallet
- Monitor spending trends
- Set daily spending limits

### Playbook 6: Security Alert (Failed Authorization)

**Scenario:** Repeated failed authorization attempts

**Steps:**

1. **Immediate (0-5 minutes):**
   - Log all details of failed attempts
   - Check if legitimate (operator mistyped) or attack
   - If attack: blocklist source IP/address

2. **Short-term (5-30 minutes):**
   - Review recent successful authorizations
   - Check for unauthorized access
   - Verify agent configuration intact

3. **Long-term (30+ minutes):**
   - Strengthen authentication if needed
   - Update security policies
   - Monitor for continued attempts

**Prevention:**
- Strong authentication
- Rate limiting on auth attempts
- IP whitelisting for operators
- Multi-factor authentication

---

## Incident Communication

### Internal Communication

**Incident status updates SHOULD include:**

```
Incident ID: INC-2026-02-03-001
Severity: SEV-2 (High)
Status: Investigating
Started: 2026-02-03 01:00:00 UTC
Summary: All transactions failing due to RPC timeout
Impact: Cannot execute operations, escrows at risk
Actions taken:
  - Switched to backup RPC
  - Testing transaction submission
Next steps:
  - Monitor success rate
  - Escalate to SEV-1 if not resolved in 15min
ETA: 01:30 UTC
```

### External Communication

**If incident affects partners (escrow counterparties, etc.):**

```
To: Escrow Partners
Subject: Temporary Service Disruption

We are experiencing technical issues affecting transaction execution.

Impact: Proof submissions may be delayed
Status: Investigating, working on resolution
ETA: Service restored by 02:00 UTC

We will update within 30 minutes.

No funds are at risk. All escrows remain secure.
```

---

## Incident Recovery

### State Recovery

**After incident resolution, verify:**

1. **Wallet state** — Balance matches expected
2. **Transaction history** — All txs accounted for
3. **Escrow status** — No deadlines missed
4. **Configuration** — Settings correct
5. **Monitoring** — Alerts functioning

**If state corrupted → restore from backup.**

### Backup and Restore

**Agents SHOULD maintain backups:**

| Data | Backup Frequency | Retention |
|------|-----------------|-----------|
| **Configuration** | On change | Indefinite |
| **Database** | Every hour | 7 days |
| **Logs** | Continuous | 30 days |
| **State snapshots** | Every 6 hours | 3 days |

**Restore procedure:**
1. Stop agent
2. Restore configuration
3. Restore database from latest backup
4. Verify state integrity
5. Replay missed events (if any)
6. Restart agent
7. Verify operations resume normally

### Gradual Resumption

**After recovery, do NOT immediately resume full operations:**

```
1. Start with read-only operations (balance checks)
2. Test single low-value transaction
3. Verify success and state consistency
4. Gradually increase operation volume
5. Monitor closely for 1 hour
6. If stable → full operations resumed
```

**Rushing back to full operations risks repeating the incident.**

---

## Post-Mortem

**After every SEV-1 or SEV-2 incident, write post-mortem:**

### Post-Mortem Template

```markdown
# Incident Post-Mortem: [Title]

**Date:** 2026-02-03
**Severity:** SEV-2
**Duration:** 45 minutes
**Author:** Operator

## Summary
Brief description of what happened.

## Timeline
- 01:00 UTC: Incident detected
- 01:05 UTC: Root cause identified
- 01:15 UTC: Fix deployed
- 01:30 UTC: Operations resumed
- 01:45 UTC: Incident closed

## Root Cause
Technical explanation of what went wrong.

## Impact
- Transactions affected: 12 failed
- Funds at risk: None
- Partners impacted: 2 escrow counterparties
- Reputation impact: Minor (communicated proactively)

## What Went Well
- Detected within 5 minutes
- Backup RPC available
- Communication timely

## What Went Wrong
- Should have detected RPC degradation earlier
- No automated failover to backup
- Alert threshold too lenient

## Action Items
- [ ] Implement automatic RPC failover
- [ ] Lower RPC latency alert threshold
- [ ] Add RPC health checks every 30s
- [ ] Document RPC failover procedure

## Lessons Learned
Always have backup RPC configured. Single point of failure is unacceptable.
```

**Post-mortems are NOT about blame. They're about learning.**

---

## Incident Prevention

### Reduce Incident Probability

**Strategies:**

1. **Comprehensive testing** — Test edge cases thoroughly
2. **Code review** — Second pair of eyes on changes
3. **Gradual rollouts** — Test in staging first
4. **Monitoring** — Detect issues early
5. **Redundancy** — Backup systems for critical components
6. **Rate limiting** — Prevent runaway operations
7. **Safety limits** — Hard constraints on risk

### Reduce Incident Impact

**Even with best prevention, incidents happen. Minimize damage:**

1. **Circuit breakers** — Auto-halt on anomalies
2. **Compartmentalization** — Isolate blast radius
3. **Rapid detection** — Know immediately when things break
4. **Automated recovery** — Self-healing where possible
5. **Clear procedures** — Playbooks for common incidents
6. **Regular drills** — Practice incident response

---

## Incident Metrics

**Track incident history to improve:**

| Metric | Target |
|--------|--------|
| **Mean time to detect (MTTD)** | <5 minutes |
| **Mean time to respond (MTTR)** | <30 minutes |
| **Incident frequency** | <1 SEV-1/month, <4 SEV-2/month |
| **Repeat incidents** | 0 (same root cause) |
| **Post-mortem completion** | 100% for SEV-1/SEV-2 |

**Improving metrics = improving reliability.**

---

## Summary

**Incidents are inevitable. Response is controllable.**

- ✅ Severity levels (Critical → Low)
- ✅ Automated detection (monitoring alerts)
- ✅ Response playbooks (common scenarios)
- ✅ Communication (internal, external)
- ✅ Recovery (restore state, verify, resume)
- ✅ Post-mortems (learn, improve, prevent)
- ✅ Prevention (testing, monitoring, redundancy)

**Fast detection + Clear procedures + Learning mindset = Resilient agents.**

---

**See also:**
- `monitoring.md` — How to detect incidents early
- `triggers.md` — What causes agents to act
- `guardrails.md` — Pre/post-execution checks that prevent incidents
