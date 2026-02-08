# 12 — Human-in-the-Loop & Approval System

> Architecture for approval gates, notification routing, escalation paths, and graduated autonomy across the agentic factory.

---

## 1. Design Principles

1. **Humans approve _what_, agents decide _how_.** Adam sets strategy, constraints, and budgets. Agents handle execution.
2. **Gate on irreversibility and blast radius, not frequency.** High-volume low-risk actions flow freely; rare high-impact actions get gates.
3. **Earned autonomy.** Every agent starts supervised and graduates based on track record.
4. **3am-safe by default.** The system must handle decisions when Adam is asleep — safe defaults, pre-authorized playbooks, and cascading escalation.
5. **No rubber-stamping.** If approval rates exceed 95% on a gate, the gate is wrong — remove it or tighten the criteria.

---

## 2. Action Classification

Every agent action is classified into one of four tiers:

### Tier 0 — Auto-Execute (No Gate)

Agent acts freely. Logged but not reviewed.

- Read-only operations (search, fetch, browse, query)
- Internal drafts, working documents, scratchpad writes
- Calculations, analysis, data transformation
- Git commits to feature branches
- Running tests
- Internal agent-to-agent communication
- Idempotent infrastructure queries

### Tier 1 — Act & Notify (Async Review)

Agent executes immediately, Adam is notified for retroactive review.

- Routine internal comms following established patterns
- Spending under **$20 per transaction** / **$100 per day**
- Dev/staging deployments
- Standard operational tasks within defined playbooks
- Dependency updates passing all tests
- Scheduled content posts matching approved templates

### Tier 2 — Approve Before (Blocking Gate)

Agent pauses, sends approval request, waits for sign-off.

- **Spending $20–$500** per transaction
- Production deployments
- Customer-facing communications (first contact, complaints, anything mentioning refunds/legal)
- Access permission changes
- Any action touching PII or customer data
- New vendor/service onboarding
- DNS, domain, or certificate changes
- Publishing to public channels (Twitter, blog, Product Hunt)

### Tier 3 — Mandatory Multi-Step Approval

Always requires explicit human approval with full context review. No timeout auto-approve.

- **Spending above $500**
- Legal commitments, contracts, ToS changes
- Production database migrations or deletions
- Security-related changes (auth, encryption, access control)
- Hiring/contractor decisions
- Public statements on behalf of any brand
- Anything Adam has explicitly flagged as "always ask"

---

## 3. Notification & Approval Channels

### Primary Channel: WhatsApp

Adam's primary interface. All Tier 2+ approval requests route here by default.

**Format:**
```
🔒 APPROVAL REQUEST

What: Deploy pricing-service v2.3.1 to production
Why: Fixes billing rounding bug affecting 12 customers
Agent: builder-agent-03
Risk: Medium (touches payment flow)
Confidence: 0.82
Deadline: 4h

Reply: ✅ approve | ❌ reject | 📝 modify | ⬆️ escalate
```

**Interaction model:**
- Simple emoji or text replies for approve/reject
- "modify" opens a brief dialogue for changes
- Reminders at 50% and 90% of deadline

### Secondary Channel: Dashboard (Web UI)

For batched review, audit trail browsing, and detailed context.

- Review queue with filtering (by agent, tier, domain, status)
- Diff views for code/content changes
- Spending dashboards and budget envelope status
- Approval history and analytics
- Bulk approve/reject for batched items

### Escalation Channel: Phone Call

For Tier 3 items approaching deadline with no response, or system-detected emergencies.

- Automated voice call via Twilio
- Brief spoken summary of the situation
- Press 1 to approve, 2 to reject, 3 to extend deadline

### Notification Tiers

| Priority | Channel | Timing | Example |
|----------|---------|--------|---------|
| **Informational** | Dashboard digest | Daily batch | "Agents completed 47 tasks today" |
| **Awareness** | WhatsApp | Within minutes | "Budget 80% consumed for cloud infra" |
| **Action Required** | WhatsApp + reminder | Immediate + follow-up | Approval request with deadline |
| **Urgent** | WhatsApp + Phone | Immediate | Security incident, production down |

---

## 4. Approval Gate Implementation

### 4.1 Gate Types

**Synchronous (Blocking)**
Agent pauses execution, queues the request, waits for response.
→ Used for: Tier 2 and Tier 3 actions.

**Asynchronous (Non-blocking)**
Agent executes, logs the action, queues for retroactive review.
→ Used for: Tier 1 actions.

**Batched Review**
Agent queues multiple low-urgency items for periodic human review.
→ Used for: Content review, expense categorization, weekly report sign-off.

**Escalation Gate**
Agent attempts resolution → low confidence → escalates to senior agent → still uncertain → escalates to human.
→ Used for: Customer support tiers, complex decision trees.

### 4.2 Approval Request Schema

```yaml
approval_request:
  id: "apr-20260208-0042"
  agent_id: "builder-03"
  task_id: "task-abc123"
  tier: 2
  action_type: "production_deployment"
  summary: "Deploy pricing-service v2.3.1"
  detail: "Fixes float rounding in invoice calculation. 3 customers reported $0.01 discrepancies."
  confidence: 0.82
  risk_assessment:
    financial: low
    reputational: medium
    reversibility: "rollback available, <5min"
  alternatives_considered:
    - "Hotfix only the rounding function (rejected: broader refactor needed)"
  deadline: "2026-02-08T13:00:00Z"
  timeout_action: "deny"
  diff_url: "https://dashboard.factory/diffs/apr-20260208-0042"
```

### 4.3 Timeout Behaviours

| Tier | Default Timeout | Timeout Action | Rationale |
|------|----------------|----------------|-----------|
| Tier 1 | N/A | Auto-executed | Non-blocking |
| Tier 2 | 4 hours | **Deny** | Safe default; agent retries or escalates |
| Tier 2 (time-sensitive) | 2 hours | **Escalate** to phone | Customer SLA at risk |
| Tier 3 | 24 hours | **Deny** (never auto-approve) | High-stakes must wait |

**Reminder cadence:** 50% of deadline → first reminder. 90% → urgent reminder. 100% → timeout action.

---

## 5. Spending Controls

### 5.1 Budget Envelopes

Each agent (or agent team) receives pre-approved spending pools:

```yaml
budgets:
  cloud_infrastructure:
    owner: infra-agent
    monthly_limit: $500
    per_transaction_auto: $20
    alert_at: [50%, 80%, 95%]
    refill: monthly_auto

  marketing_spend:
    owner: marketing-agent
    monthly_limit: $300
    per_transaction_auto: $10
    alert_at: [50%, 80%]
    refill: on_approval

  customer_retention:
    owner: support-agent
    monthly_limit: $200
    per_transaction_auto: $15
    refill: monthly_auto

  tools_and_apis:
    owner: orchestrator
    monthly_limit: $400
    per_transaction_auto: $50
    alert_at: [80%]
    refill: monthly_auto
```

### 5.2 Velocity Controls

Beyond per-transaction limits:
- **Daily caps** across all agents: $200/day auto-approved
- **Spike detection**: >3x normal daily spend → pause and notify
- **New vendor lockout**: first transaction to any new recipient requires Tier 2 approval regardless of amount
- **Cumulative exposure tracking**: correlated spending across agents is monitored as aggregate risk

---

## 6. Deployment Gates

### 6.1 Pipeline with Human Checkpoints

```
Code Change
  → Automated tests (unit, integration, security scan)
  → Auto-deploy to staging
  → [Tier 1: Notify Adam of staging deploy]
  → Canary deploy (1% traffic)
  → Automated metric validation (error rate, latency, p99)
  → [Tier 2: Approve production rollout]
  → Gradual rollout (10% → 50% → 100%)
```

### 6.2 Auto-Rollback Authority

Agents **always** have authority to rollback without approval. False rollback cost ≪ bad deployment cost.

Triggers:
- Error rate increase >0.5% over baseline
- Latency p99 increase >25%
- Any 5xx spike >2σ from normal
- Health check failures

### 6.3 Always-Gate Deployments

These always require Tier 2+ approval regardless of agent trust level:
- First deploy of a new service
- Database migrations in production
- Auth/authz changes
- Billing/payment flow changes
- Breaking API changes
- Infrastructure provider changes

---

## 7. Customer-Facing Content

### 7.1 Graduated Trust Model

```
Phase 1: FULL REVIEW          — Agent drafts, Adam sends. Build training data.
Phase 2: TEMPLATE AUTO-SEND   — Pre-approved templates auto-send. Novel content reviewed.
Phase 3: CONFIDENCE-BASED     — High-confidence + pattern-match → auto-send. Edge cases reviewed.
Phase 4: AUDIT SAMPLING        — Agent sends freely. 10% random sample reviewed retroactively.
Phase 5: EXCEPTION-ONLY       — Only flagged anomalies reviewed.
```

Each agent starts at Phase 1. Advancement criteria:
- **Phase 1→2**: 50+ approved outputs with <5% modification rate
- **Phase 2→3**: 200+ outputs, <2% override rate, passes adversarial test suite
- **Phase 3→4**: 500+ outputs, <1% override rate, 30-day clean streak
- **Phase 4→5**: 2000+ outputs, <0.5% issue rate, 90-day clean streak

### 7.2 Auto-Flag Triggers

Regardless of phase, always flag for human review if content:
- Contains apologies or admissions of fault
- Mentions pricing, discounts, refunds
- References competitors by name
- Responds to angry/escalated customers
- Makes commitments or promises
- Is the first message to a new customer
- Contains legal or compliance-adjacent language

---

## 8. Escalation Paths

### 8.1 Escalation Chain

```
L0: Agent handles autonomously
    ↓ (low confidence, edge case, policy gap)
L1: Specialist agent reviews (e.g., security-agent reviews auth change)
    ↓ (still uncertain, novel situation)
L2: Adam via WhatsApp (standard approval flow)
    ↓ (legal risk, large financial, PR exposure)
L3: Adam + external advisor (lawyer, accountant — Adam loops them in)
```

### 8.2 Escalation Triggers

- **Explicit**: customer asks for a human
- **Sentiment**: anger, threats, legal language detected
- **Loop**: agent attempted resolution 3+ times without progress
- **Scope**: request falls outside agent's defined domain
- **Precedent**: decision would establish a new pattern with no prior example
- **Conflict**: agent's instructions contradict each other
- **Time**: urgent matter exceeding agent's authority

### 8.3 Anti-Patterns to Monitor

| Anti-Pattern | Detection | Fix |
|---|---|---|
| Escalation ping-pong | Same item bounces agent↔human 3+ times | Clear ownership rules, better context in handoff |
| Silent escalation | Items sit in queue >2h with no acknowledgement | Reminder cadence + phone escalation |
| Over-escalation | Agent escalation rate >30% | Retrain on resolved cases, expand playbooks |
| Dead-end queue | Queue items >24h old with no reviewer | Health check alerts, auto-reassign |

---

## 9. Graduated Autonomy System

### 9.1 Trust Levels

Each agent has a trust level that governs its approval thresholds:

```
SUPERVISED   → All outputs reviewed (Tier 2 gate on everything)
ASSISTED     → 80% reviewed, template actions auto-approved
AUTONOMOUS   → Only flagged items reviewed, broad Tier 1 authority
TRUSTED      → Exception-only review, elevated spending limits
```

### 9.2 Trust Score Calculation

```yaml
trust_score:
  inputs:
    - task_completion_rate        # weight: 0.2
    - human_override_rate         # weight: 0.3 (inverse)
    - error_rate                  # weight: 0.3 (inverse)
    - adversarial_test_pass_rate  # weight: 0.1
    - time_in_service             # weight: 0.1
  
  thresholds:
    supervised:  0.0 - 0.39
    assisted:    0.4 - 0.69
    autonomous:  0.7 - 0.89
    trusted:     0.9+
  
  promotion_requirements:
    min_decisions: 100
    min_days_at_level: 14
    max_override_rate: 5%
  
  demotion_triggers:
    - single_critical_error      # → drop to SUPERVISED
    - override_rate > 10%        # → drop one level
    - new_capability_added       # → drop to ASSISTED for that domain
    - 30_day_inactivity          # → drop one level
```

### 9.3 Domain-Specific Trust

An agent can be TRUSTED for code deployment but SUPERVISED for customer communications. Trust is tracked per-domain:

```yaml
agent: builder-03
trust:
  code_deployment: AUTONOMOUS
  customer_email: ASSISTED
  spending: ASSISTED
  infrastructure: SUPERVISED
```

---

## 10. Policy-as-Code

All approval policies are defined declaratively and version-controlled:

```yaml
# policies/customer-comms.yaml
policies:
  - name: customer_email_send
    action: send_email
    conditions:
      - target_type: external_customer
    approval:
      default: tier_2
      auto_approve_when:
        - agent_trust_level: ["trusted"]
          template: pre_approved
          sentiment: positive
        - agent_phase: ">= 3"
          confidence: ">= 0.95"
      always_require_when:
        - customer_tier: ["enterprise", "strategic"]
        - contains_any: ["refund", "legal", "sorry", "our fault"]
        - first_contact: true
    timeout: 2h
    timeout_action: escalate

  - name: production_deploy
    action: deploy
    conditions:
      - environment: production
    approval:
      default: tier_2
      auto_approve_when:
        - change_type: rollback  # rollbacks always auto-approved
      never_auto_approve:
        - change_type: [migration, new_service, auth_change]
    timeout: 4h
    timeout_action: deny

  - name: spending
    action: financial_transaction
    approval:
      default: tier_1
      tier_2_when:
        - amount: "> $20"
        - vendor: new
      tier_3_when:
        - amount: "> $500"
        - category: legal
    timeout: 4h
    timeout_action: deny
```

---

## 11. Audit Trail

Every approval decision is logged immutably:

```json
{
  "decision_id": "apr-20260208-0042",
  "timestamp": "2026-02-08T09:15:00Z",
  "agent_id": "builder-03",
  "action": "production_deploy",
  "tier": 2,
  "confidence": 0.82,
  "approval_type": "human_approved",
  "approver": "adam",
  "channel": "whatsapp",
  "response_time_ms": 185000,
  "modifications": null,
  "outcome": "approved",
  "downstream_result": "deploy_success"
}
```

Audit data feeds into:
- Trust score calculation
- Gate effectiveness analysis (approval rates, response times)
- Anomaly detection (spending patterns, escalation frequency)
- Monthly review dashboards

---

## 12. The 3am Protocol

When Adam is unavailable (overnight, AFK, DND):

1. **Pre-authorized playbooks execute normally.** Known scenarios with documented responses proceed at Tier 1.
2. **Tier 2 items queue.** Reminders resume when Adam comes online.
3. **Time-sensitive Tier 2** (customer SLA at risk): agent takes the conservative action within its authority, flags for morning review.
4. **True emergencies** (production down, security breach): auto-rollback/lockdown first, then phone call escalation.
5. **Everything else waits.** The cost of delay is almost always lower than the cost of a bad autonomous decision.

```yaml
overnight_policy:
  hours: "23:00-07:00 NZDT"
  tier_1: execute_normally
  tier_2: queue_with_conservative_default
  tier_2_urgent: execute_safe_action + flag_for_review
  tier_3: queue_always
  emergency: auto_mitigate + phone_call
```

---

## 13. Implementation Phases

### Phase 1: Foundation (Week 1-2)
- Implement 4-tier action classification in orchestrator
- WhatsApp approval flow (request → emoji reply → confirm)
- Basic spending envelope system
- Structured audit logging

### Phase 2: Intelligence (Week 3-4)
- Confidence scoring for agent actions
- Auto-flag triggers for customer content
- Dashboard for review queue and audit trail
- Timeout and reminder system

### Phase 3: Graduation (Month 2-3)
- Trust score tracking per agent per domain
- Automated promotion/demotion based on track record
- Policy-as-code engine for declarative gate definitions
- Batched review for high-volume decisions

### Phase 4: Optimization (Ongoing)
- Gate effectiveness analytics (remove useless gates, add where errors cluster)
- Adversarial testing injection for anti-rubber-stamp
- Phone escalation for urgent items
- Cross-agent spending correlation and velocity detection

---

## References

- Research: [18-human-in-loop.md](../research/18-human-in-loop.md) — core HITL patterns and research
- Research: [17-security-access.md](../research/17-security-access.md) — security context for agent permissions
- Anthropic, "Building Effective Agents" (2025)
- NIST AI Risk Management Framework
- Temporal.io — durable workflow patterns for approval queues
