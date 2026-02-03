# OpenClaw Interface

**How agents discover, validate, and invoke skills via the OpenClaw ecosystem.**

---

## Overview

**OpenClaw** is a collection of open-source agent skills for blockchain operations, token deployment, identity management, and protocol interactions. Agents use OpenClaw to extend capabilities without reinventing primitives.

**This document defines interface expectations, not OpenClaw's implementation.**

---

## What is OpenClaw?

OpenClaw provides modular skills for:
- Token deployment (Clanker SDK)
- ENS name management
- NFT operations (Yoink, QRCoin)
- Privacy tools (Veil Cash)
- Identity (ERC-8004, ENS Primary Name)
- DeFi protocols (Zapper, Base-native tools)

**Skills are discovered dynamically, validated before use, and invoked through standardized interfaces.**

---

## Core Principle

**Agents MUST validate skills before invocation.**

OpenClaw skills are:
- Open-source (auditable)
- Modular (single-purpose)
- Composable (work together)

But agents MUST NOT assume skills are safe. Every skill requires validation.

**Skill discovery ≠ skill trust.**

---

## Skill Discovery

### ClawdHub Registry

Skills are published to ClawdHub (clawdhub.com):

```
1. Agent queries ClawdHub for available skills
2. ClawdHub returns skill metadata
3. Agent evaluates skill suitability
4. Agent installs skill if approved
```

**Skill metadata includes:**
- Name and description
- Capabilities (what it does)
- Dependencies (what it requires)
- Version and author
- Audit status

### Skill Installation

```bash
# Discovery
clawdhub search "token deployment"

# Installation
clawdhub install clanker

# Verification
clawdhub verify clanker
```

**Agents SHOULD verify skill integrity before use:**
- Check signature
- Review source code
- Test in sandbox environment

---

## Skill Validation

### Pre-Use Validation

Before invoking any OpenClaw skill, agent MUST verify:

| Check | Purpose |
|-------|---------|
| **Skill signature** | Authenticity verification |
| **Source code review** | No malicious behavior |
| **Dependency audit** | No vulnerable dependencies |
| **Version pinning** | Consistent behavior |
| **Sandbox testing** | No unexpected side effects |

**If validation fails → do NOT use skill.**

### Skill Whitelist

Agents SHOULD maintain a skill whitelist:

```json
{
  "approved_skills": {
    "clanker": {
      "version": "1.0.0",
      "approved_date": "2026-02-01",
      "approved_by": "operator",
      "capabilities": ["token_deployment"]
    },
    "ens-primary-name": {
      "version": "1.0.0",
      "approved_date": "2026-02-01",
      "approved_by": "operator",
      "capabilities": ["ens_management"]
    }
  }
}
```

**Skills not in whitelist → automatically denied.**

### Version Pinning

**Agents MUST pin skill versions:**

```bash
# ❌ Dangerous (auto-update)
clawdhub install clanker

# ✅ Safe (pinned version)
clawdhub install clanker@1.0.0
```

**Automatic updates can introduce breaking changes or vulnerabilities.**

---

## Skill Invocation

### Request Pattern

When invoking OpenClaw skill:

```json
{
  "skill": "clanker",
  "operation": "deploy_token",
  "parameters": {
    "name": "MyToken",
    "symbol": "MTK",
    "supply": "1000000"
  },
  "request_id": "req_deploy_001",
  "agent_id": "agent_xyz"
}
```

**Governor MUST approve before invocation.**

### Response Pattern

OpenClaw skills SHOULD return:

```json
{
  "status": "success",
  "result": {
    "contract_address": "0xToken...",
    "transaction_hash": "0xabc...",
    "deployment_cost": "0.005 ETH"
  },
  "request_id": "req_deploy_001"
}
```

**Agent MUST verify response matches expected outcome.**

---

## Governor Integration

### Skill Authorization Levels

| Skill Type | Authorization Level | Governor Check |
|------------|---------------------|----------------|
| **Read-only** (e.g., check ENS name) | Routine | Optional |
| **Low-risk write** (e.g., set primary name) | Standard | Required |
| **High-risk write** (e.g., deploy contract) | Elevated | Required + human approval |

**Default: Deny unless explicitly authorized.**

### Preconditions

Before invoking skill, governor MUST verify:

1. **Skill is whitelisted** → In approved skills list
2. **Operation is allowed** → Within agent's authorized scope
3. **Parameters are valid** → Meet skill's requirements
4. **Cost is acceptable** → Within budget limits
5. **No conflicts** → No pending operations that conflict

### Postconditions

After skill execution, governor MUST verify:

1. **Expected result** → Outcome matches specification
2. **Cost within bounds** → Gas/fees within estimate
3. **No side effects** → No unexpected state changes
4. **Events emitted** → Expected onchain events present

---

## OpenClaw Skill Catalog

### Token Deployment

**Skill:** `clanker`  
**Purpose:** Deploy ERC20 tokens on Base/Ethereum

**Capabilities:**
- Deploy standard tokens
- Set vesting schedules
- Configure airdrops
- Manage LP fees

**Governor requirements:**
- Elevated approval (contract deployment)
- Deployment cost within budget
- Token parameters validated

### ENS Management

**Skill:** `ens-primary-name`  
**Purpose:** Set ENS primary name on Base/L2s

**Capabilities:**
- Set primary name
- Configure reverse resolution
- Multi-chain support

**Governor requirements:**
- Standard approval
- ENS name owned by agent
- Gas cost within limits

### NFT Operations

**Skill:** `yoink`  
**Purpose:** Play Yoink capture-the-flag game

**Capabilities:**
- Yoink flag from current holder
- Check game stats
- View leaderboard

**Governor requirements:**
- Standard approval
- Transaction cost acceptable
- Game contract whitelisted

### Privacy Tools

**Skill:** `veil`  
**Purpose:** Shielded transactions on Base

**Capabilities:**
- Deposit to private pool
- Private transfers
- ZK proof generation

**Governor requirements:**
- Standard approval
- Privacy pool contract whitelisted
- Deposit amount within limits

### Identity

**Skill:** `bankr` (ERC-8004 component)  
**Purpose:** Agent registration on Ethereum

**Capabilities:**
- Register agent identity
- Transfer identity
- Query registrations

**Governor requirements:**
- Standard approval
- Registration fee acceptable
- Identity parameters validated

---

## Security Considerations

### Skill Sandboxing

**Skills SHOULD be sandboxed during testing:**

```
1. Install skill in isolated environment
2. Test with dummy data
3. Verify no unintended network calls
4. Check resource usage (CPU, memory)
5. Approve if tests pass
```

**Never test skills with production credentials.**

### Dependency Management

**Skills have dependencies. Agents MUST audit:**

```
Skill: clanker
Dependencies:
  - ethers.js (audit required)
  - viem (audit required)
  - OpenZeppelin contracts (audit required)
```

**Vulnerable dependency → entire skill is unsafe.**

### API Key Security

**Some skills require API keys. Agents MUST:**

- Store keys in secure vault (not plaintext)
- Use least-privilege keys (read-only when possible)
- Rotate keys periodically
- Revoke keys if skill removed

**Never commit API keys to code or logs.**

### Rate Limiting

**OpenClaw skills may hit external APIs. Agents MUST:**

- Respect rate limits
- Implement backoff strategies
- Cache responses when appropriate
- Monitor API usage

**Excessive API calls → skill may be disabled by provider.**

---

## Error Handling

### Skill-Specific Errors

| Error Type | Cause | Recovery |
|------------|-------|----------|
| **SKILL_NOT_FOUND** | Skill not installed | Install skill |
| **INVALID_PARAMETERS** | Bad input | Fix parameters |
| **EXECUTION_FAILED** | Skill logic error | Investigate skill code |
| **RATE_LIMITED** | Too many requests | Wait and retry |
| **NETWORK_ERROR** | RPC/API unavailable | Retry with backoff |

### Fallback Patterns

**If skill fails:**
1. Log error with full context
2. Check if alternative skill exists
3. If no alternative → deny action
4. Alert operator if critical

**Example:**
```
Primary skill: clanker (deploy token)
Fallback: None (contract deployment is atomic)
Action on failure: Deny, alert operator
```

---

## Skill Lifecycle

### Installation

```bash
1. Discover skill via ClawdHub
2. Review skill documentation
3. Audit source code and dependencies
4. Test in sandbox environment
5. Add to whitelist if approved
6. Install with pinned version
```

### Updates

```bash
1. Check for skill updates
2. Review changelog
3. Audit changes
4. Test updated skill
5. Update version in whitelist
6. Install new version
```

**Never auto-update skills in production.**

### Removal

```bash
1. Determine skill is no longer needed
2. Check for dependencies (other skills using this one)
3. Remove from whitelist
4. Uninstall skill
5. Revoke associated API keys
```

---

## Example: Token Deployment via Clanker

**Scenario:** Agent wants to deploy a token

### Step 1: Skill Validation

```
Skill: clanker@1.0.0
Validation:
  ✓ In whitelist
  ✓ Version pinned
  ✓ Source code audited (2026-02-01)
  ✓ No vulnerable dependencies
Result: APPROVED for use
```

### Step 2: Governor Pre-Check

```
Operation: Deploy ERC20 token
Checks:
  ✓ Operation allowed (elevated approval granted)
  ✓ Parameters valid (name, symbol, supply)
  ✓ Cost acceptable (~0.005 ETH deployment)
  ✓ No conflicts (no pending deployments)
Result: APPROVED
```

### Step 3: Skill Invocation

```json
{
  "skill": "clanker",
  "operation": "deploy_token",
  "parameters": {
    "name": "AgentToken",
    "symbol": "AGTK",
    "initial_supply": "1000000",
    "network": "base"
  },
  "request_id": "req_deploy_001"
}
```

### Step 4: Skill Execution

```
Clanker:
1. Validates parameters
2. Estimates deployment cost
3. Submits deployment transaction
4. Waits for confirmation
5. Returns contract address
```

### Step 5: Governor Post-Check

```
Result: 0xToken123...
Checks:
  ✓ Contract deployed at expected address
  ✓ Token parameters match specification
  ✓ Deployment cost within estimate
  ✓ Events: "TokenDeployed" emitted
Result: VERIFIED
```

### Step 6: Agent Receives Result

```json
{
  "status": "success",
  "result": {
    "contract_address": "0xToken123...",
    "transaction_hash": "0xabc...",
    "token": {
      "name": "AgentToken",
      "symbol": "AGTK",
      "total_supply": "1000000"
    },
    "deployment_cost": "0.0048 ETH"
  },
  "request_id": "req_deploy_001"
}
```

**Token deployed successfully.**

---

## OpenClaw vs BNKR

| Aspect | BNKR | OpenClaw |
|--------|------|----------|
| **Purpose** | Wallet operations, trading | Protocol interactions, identity, tooling |
| **Scope** | Financial operations | Broader agent capabilities |
| **Wallet** | Dedicated BNKR wallet | Agent's main wallet |
| **Risk level** | Medium (financial) | Variable (depends on skill) |
| **Authorization** | Governor approval | Governor + skill validation |

**Use BNKR for trading. Use OpenClaw for everything else.**

---

## Skill Development Guidelines

If building new OpenClaw skills:

### Requirements

1. **Single responsibility** — One skill, one purpose
2. **Composable** — Works with other skills
3. **Documented** — Clear usage instructions
4. **Tested** — Unit tests and integration tests
5. **Versioned** — Semantic versioning
6. **Auditable** — Open source, readable code

### Interface Contract

Every skill MUST provide:

```json
{
  "name": "my-skill",
  "version": "1.0.0",
  "description": "Brief description",
  "capabilities": ["capability1", "capability2"],
  "operations": {
    "operation1": {
      "parameters": {...},
      "returns": {...}
    }
  },
  "dependencies": ["ethers", "viem"],
  "author": "agent|human",
  "license": "MIT"
}
```

### Safety Guidelines

Skills MUST NOT:
- Access private keys directly
- Make unauthorized network calls
- Exceed resource limits
- Hide functionality in obfuscated code

---

## Summary

**OpenClaw extends agent capabilities through validated, composable skills.**

- ✅ Skills are discovered via ClawdHub
- ✅ Skills are validated before use
- ✅ Skills are whitelisted and version-pinned
- ✅ Governor oversees all skill invocations
- ✅ Errors are handled gracefully
- ✅ Skills are sandboxed during testing

**OpenClaw provides the tools. Governor provides the oversight.**

---

**See also:**
- `bnkr-interface.md` — BNKR wallet operations
- `governor-layer.md` — Decision engine architecture
- `guardrails.md` — Pre/post-execution checks
