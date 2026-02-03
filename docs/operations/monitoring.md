# Monitoring

**How to observe agent health, performance, and behavior.**

---

## Overview

Autonomous agents operate continuously and handle value. **Monitoring** provides visibility into agent operations, enabling:
- Early detection of failures
- Performance optimization
- Security incident identification
- Accountability and auditability

**If you can't monitor it, you can't trust it.**

---

## Core Principle

**Observability is mandatory for production agents.**

Agents MUST emit:
- **Logs** — What happened
- **Metrics** — How well it's working
- **Traces** — Where time is spent
- **Alerts** — When intervention needed

**Silent agents are dead agents.**

---

## Monitoring Categories

### 1. Health Monitoring

**Is the agent alive and functional?**

#### System Health

| Metric | Description | Alert Threshold |
|--------|-------------|-----------------|
| **Process uptime** | Time since last restart | Alert if <5 minutes |
| **Memory usage** | RAM consumption | Alert if >80% |
| **CPU usage** | Processor utilization | Alert if >90% for 5+ minutes |
| **Disk space** | Available storage | Alert if <10% free |
| **Network connectivity** | RPC reachability | Alert if down |

**Check frequency:** Every 1 minute

#### Component Health

| Component | Health Check | Failure Action |
|-----------|-------------|----------------|
| **Governor** | Can evaluate test action | Restart governor module |
| **BNKR integration** | Can query balance | Switch to backup RPC |
| **Database** | Can read/write | Restore from backup |
| **Logging system** | Can write log entry | Alert operator immediately |

**Check frequency:** Every 5 minutes

### 2. Financial Monitoring

**Is the agent's financial state healthy?**

#### Wallet Health

| Metric | Description | Alert Threshold |
|--------|-------------|-----------------|
| **Balance** | Available ETH/tokens | Alert if <0.01 ETH |
| **Balance trend** | Change over time | Alert if dropping >10%/day |
| **Pending txs** | Unconfirmed transactions | Alert if >3 pending |
| **Stuck txs** | Txs pending >10 minutes | Alert immediately |

**Check frequency:** Every 5 minutes

#### Transaction Monitoring

| Metric | Description | Alert Threshold |
|--------|-------------|-----------------|
| **Tx success rate** | % successful | Alert if <95% |
| **Tx volume** | Daily tx count | Alert if unusual spike/drop |
| **Gas spending** | ETH spent on gas | Alert if >daily budget |
| **Average gas price** | Gwei per tx | Alert if >2x normal |

**Check frequency:** Every 30 minutes

#### Escrow Monitoring

| Metric | Description | Alert Threshold |
|--------|-------------|-----------------|
| **Active escrows** | Open payment contracts | Monitor count |
| **Expiring escrows** | Deadline <24h | Alert at 6h remaining |
| **Disputed escrows** | Under dispute | Alert immediately |
| **Escrow value** | Total locked funds | Monitor trend |

**Check frequency:** Every 30 minutes

### 3. Performance Monitoring

**How efficiently is the agent operating?**

#### Response Times

| Metric | Description | Target |
|--------|-------------|--------|
| **Trigger latency** | Time to process trigger | <5 seconds |
| **Governor evaluation** | Decision time | <1 second |
| **Transaction submission** | Submit to confirmation | <30 seconds |
| **API response time** | External API calls | <2 seconds |

**Alert if p95 latency exceeds target by 2x.**

#### Throughput

| Metric | Description | Baseline |
|--------|-------------|----------|
| **Actions per minute** | Operations executed | Varies by agent |
| **Events processed** | Triggers handled | Varies by agent |
| **API requests** | External calls | Varies by agent |

**Alert if throughput drops >50% from baseline.**

#### Resource Efficiency

| Metric | Description | Optimization Target |
|--------|-------------|---------------------|
| **Gas efficiency** | Actual vs estimated gas | <10% variance |
| **RPC call rate** | Blockchain queries | Minimize redundant calls |
| **Cache hit rate** | % served from cache | >80% |

### 4. Security Monitoring

**Is the agent secure and behaving correctly?**

#### Authorization Monitoring

| Event | Description | Alert Condition |
|-------|-------------|-----------------|
| **Failed authorization** | Governor denied action | Alert if >10/hour |
| **Unauthorized access** | Invalid operator command | Alert immediately |
| **Privilege escalation** | Elevated approval requested | Log and review |

#### Anomaly Detection

| Anomaly | Description | Alert Condition |
|---------|-------------|-----------------|
| **Unusual tx volume** | Spike in transactions | >3x daily average |
| **Unusual gas spending** | Excessive gas usage | >2x daily budget |
| **Unusual counterparties** | New addresses interacted with | >5 new addresses/hour |
| **Off-hours activity** | Actions during quiet periods | Depends on schedule |

#### Safety Limit Monitoring

| Limit | Current | Alert Threshold |
|-------|---------|-----------------|
| **Daily tx volume** | % of limit used | Alert at 80% |
| **Max tx value** | Approaching limit | Alert if attempted |
| **Gas budget** | % consumed | Alert at 80% |

---

## Monitoring Tools

### Logging

**Structured logging is mandatory:**

```json
{
  "timestamp": "2026-02-03T00:45:00Z",
  "level": "info",
  "component": "governor",
  "action": "send_eth",
  "status": "approved",
  "details": {
    "to": "0xRecipient",
    "amount": "0.01 ETH",
    "safety_checks": "passed"
  },
  "request_id": "req_abc123"
}
```

**Log levels:**
- **DEBUG** — Verbose details (dev only)
- **INFO** — Normal operations
- **WARN** — Concerning but not critical
- **ERROR** — Failures requiring attention
- **CRITICAL** — System-threatening issues

### Metrics

**Time-series metrics for trending:**

```
agent_balance_eth{wallet="main"} 0.0542
agent_tx_count{status="success"} 145
agent_tx_count{status="failed"} 3
agent_governor_decisions{result="approved"} 120
agent_governor_decisions{result="denied"} 8
agent_response_time_seconds{operation="send_eth"} 0.234
```

**Tools:** Prometheus, Grafana, CloudWatch

### Dashboards

**Visual dashboards for real-time monitoring:**

**Critical Metrics Dashboard:**
- Wallet balance (ETH, USDC)
- Transaction success rate (24h)
- Pending transactions
- Active escrows
- System health (CPU, memory, disk)

**Performance Dashboard:**
- Trigger latency (p50, p95, p99)
- Governor evaluation time
- Transaction confirmation time
- API response times

**Security Dashboard:**
- Failed authorizations
- Unusual activity alerts
- Safety limit utilization
- Blocklist hits

### Alerts

**Alerting channels:**

| Severity | Channel | Response Time |
|----------|---------|---------------|
| **Critical** | SMS + Telegram + PagerDuty | Immediate |
| **High** | Telegram + Email | <15 minutes |
| **Medium** | Email | <1 hour |
| **Low** | Dashboard only | Review daily |

**Alert examples:**

**Critical:**
- Wallet balance <0.001 ETH
- Process crashed
- Stuck transactions >5
- Security violation detected

**High:**
- Balance <0.01 ETH
- Transaction failure rate >10%
- Escrow deadline <1 hour

**Medium:**
- Balance <0.05 ETH
- Gas spending >80% of budget
- Performance degradation

---

## Monitoring Configuration

### What to Monitor

**Minimum monitoring for production:**

```json
{
  "monitoring": {
    "health": {
      "enabled": true,
      "check_interval": "1m",
      "metrics": ["uptime", "memory", "cpu", "disk"]
    },
    "financial": {
      "enabled": true,
      "check_interval": "5m",
      "metrics": ["balance", "tx_success_rate", "escrows"]
    },
    "performance": {
      "enabled": true,
      "metrics": ["latency", "throughput", "gas_efficiency"]
    },
    "security": {
      "enabled": true,
      "metrics": ["auth_failures", "anomalies", "limit_usage"]
    }
  }
}
```

### Alert Thresholds

**Configurable per agent:**

```json
{
  "alerts": {
    "balance_low": {
      "threshold": "0.01 ETH",
      "severity": "high",
      "channels": ["telegram", "email"]
    },
    "tx_failure_rate": {
      "threshold": 0.1,
      "window": "1h",
      "severity": "high",
      "channels": ["telegram"]
    },
    "memory_high": {
      "threshold": 0.8,
      "duration": "5m",
      "severity": "medium",
      "channels": ["email"]
    }
  }
}
```

---

## Monitoring Best Practices

### 1. Monitor the Monitor

**Ensure monitoring system itself is healthy:**

- Heartbeat checks (monitoring system sends "alive" signal)
- Dead man's switch (alert if no signals for 10 minutes)
- Redundant alerting (multiple channels)

**If monitoring fails, you're flying blind.**

### 2. Baseline Normal Behavior

**Establish what "normal" looks like:**

```
Week 1: Collect baseline metrics
Week 2: Set initial alert thresholds
Week 3: Tune thresholds based on false positives
Week 4: Production monitoring live
```

**Alert thresholds should reflect actual behavior, not guesses.**

### 3. Reduce Alert Fatigue

**Too many alerts → alerts get ignored.**

**Strategies:**
- Aggregate related alerts (5 balance checks → 1 "balance critical")
- Use appropriate severities (not everything is critical)
- Set reasonable thresholds (avoid 1% variance alerts)
- Provide context in alerts (include values, not just "low balance")

### 4. Correlation, Not Just Events

**Single metrics can lie. Correlate multiple signals:**

```
High CPU + Low throughput = Performance bottleneck
High tx volume + Low balance = Wallet drainage risk
Failed txs + High gas = Network congestion
```

**Look for patterns, not isolated events.**

### 5. Runbook for Common Issues

**Every alert should have a runbook:**

```
Alert: Balance low (<0.01 ETH)
Runbook:
  1. Check recent transactions (unexpected large send?)
  2. Check gas spending (unusual spike?)
  3. Fund wallet from main wallet
  4. Verify balance updated
  5. Investigate cause if drainage unexpected
```

---

## Example Monitoring Setup

### Scenario: Production Trading Agent

**Monitoring stack:**
- **Logs:** JSON to file + Elasticsearch
- **Metrics:** Prometheus + Grafana
- **Alerts:** Telegram bot + Email
- **Dashboard:** Grafana (public read-only link)

**Critical metrics tracked:**
1. Wallet balance (ETH, USDC)
2. Transaction success rate (last 1h, 24h)
3. Pending transactions
4. Active escrows
5. Gas spending (today)
6. Governor approval rate
7. System uptime
8. Response time (p95)

**Alerts configured:**
- Balance <0.01 ETH → Telegram (high severity)
- Tx failure rate >10% → Telegram (high severity)
- Stuck tx >10 minutes → Telegram (critical)
- Escrow deadline <1 hour → Telegram (high severity)
- Memory >80% → Email (medium severity)

**Dashboard widgets:**
- Balance chart (24h trend)
- Transaction timeline (success/fail)
- Governor decisions (approve/deny pie chart)
- Gas spending (daily budget bar)
- System health (CPU, memory, disk gauges)

---

## Monitoring Checklist

**Before launching agent in production:**

- [ ] Structured logging configured
- [ ] Metrics collection enabled
- [ ] Dashboards created and accessible
- [ ] Alert channels configured (Telegram, email)
- [ ] Alert thresholds set based on testing
- [ ] Runbooks written for common alerts
- [ ] Dead man's switch configured
- [ ] Monitoring system health checks enabled
- [ ] Backup alerting configured (redundancy)
- [ ] Access controls set (who can see dashboards)

**If any item unchecked → not ready for production.**

---

## Summary

**Monitoring makes agents observable and accountable.**

- ✅ Health monitoring (system, components)
- ✅ Financial monitoring (balance, transactions, escrows)
- ✅ Performance monitoring (latency, throughput, efficiency)
- ✅ Security monitoring (authorization, anomalies, limits)
- ✅ Structured logging (JSON, searchable)
- ✅ Time-series metrics (trends, alerting)
- ✅ Visual dashboards (real-time visibility)
- ✅ Multi-channel alerting (Telegram, email, SMS)

**Monitor everything. Alert on what matters. Review daily.**

---

**See also:**
- `triggers.md` — What causes agents to act
- `incident-response.md` — What to do when alerts fire
- `guardrails.md` — Pre/post-execution verification
