# 18. Human-in-the-Loop & Approval Workflows

> When should agents stop and ask? How to balance autonomy with oversight.

## The Core Tension

Every agent system faces the same tradeoff: **autonomy vs. control**. Too many approval gates and agents become glorified form-fillers. Too few and you get runaway spending, embarrassing customer emails, or production outages. The goal isn't zero human involvement — it's **right-sized human involvement**.

The principle: **humans should review decisions that are irreversible, expensive, or reputationally risky. Agents should handle everything else.**

---

## When Agents Should Stop and Ask

### The Irreversibility Heuristic

The single best rule: **can this be undone?** Map every action to a reversibility score:

| Reversibility | Examples | Approval Needed? |
|---|---|---|
| **Fully reversible** | Draft a document, create a branch, query a database | No — auto-approve |
| **Awkwardly reversible** | Send a Slack message, create a calendar event, update a config | Low bar — confidence threshold |
| **Costly to reverse** | Deploy to production, send bulk emails, make purchases | Yes — human approval |
| **Irreversible** | Delete data, send legal docs, publish to public, financial transfers | Always — mandatory gate |

### The Confidence Threshold Model

Agents should self-assess confidence and escalate when uncertain:

```
if confidence >= 0.95 and action.reversibility == "full":
    execute()
elif confidence >= 0.80 and action.risk == "low":
    execute_with_logging()
elif confidence >= 0.60:
    execute_with_notification()  # async review
else:
    pause_and_ask()  # blocking review
```

**Important**: Confidence isn't just model logprobs. It's a composite of:
- How well the request matches known patterns
- Whether all required information is present
- Whether the action falls within established boundaries
- How many assumptions the agent had to make

### Decision Categories

**Auto-approve (agent acts freely):**
- Read-only operations (fetching data, searching, browsing)
- Internal drafts and working documents
- Calculations and analysis
- Idempotent operations
- Actions within pre-approved templates

**Notify-after (act, then inform):**
- Routine communications following established patterns
- Standard operational tasks within defined parameters
- Low-value purchases within budget
- Internal status updates

**Approve-before (pause, request sign-off):**
- Customer-facing communications (especially first contact)
- Financial transactions above threshold
- Production deployments
- Access permission changes
- Anything touching PII or sensitive data
- Actions outside the agent's established domain

**Never auto-approve:**
- Legal commitments or contracts
- Public statements on behalf of the org
- Deletion of production data
- Security-related changes
- Hiring/firing decisions
- Anything the human explicitly said to always check

---

## Approval Gate Architecture

### Gate Types

**1. Synchronous (Blocking) Gates**
Agent pauses execution and waits for human approval before continuing.

```
Agent → [Proposed Action + Context] → Approval Queue → Human Reviews → Approve/Reject/Modify → Agent Continues
```

Best for: high-stakes, time-insensitive decisions. Deployment gates, spend approvals, contract reviews.

**2. Asynchronous (Non-blocking) Gates**  
Agent proceeds but flags for review. Human can retroactively override.

```
Agent → [Executes Action] → Review Queue → Human Reviews → [Override if needed]
```

Best for: moderate-risk, time-sensitive decisions. Routine customer responses, standard operational tasks.

**3. Batched Review Gates**
Agent queues multiple decisions for periodic human review.

```
Agent → [Queue of 20 proposed actions] → Human reviews batch → Approve/reject individually → Agent processes approvals
```

Best for: high-volume, moderate-risk decisions. Content moderation, expense approvals, data classification.

**4. Escalation Gates**
Agent handles routine cases, escalates edge cases up a chain.

```
Agent → [Attempts resolution] → Confidence too low → Escalate to senior agent → Still uncertain → Escalate to human
```

Best for: tiered support systems, complex decision trees.

### Review Queue Design

A good review queue needs:

- **Context**: What is the agent proposing? Why? What alternatives were considered?
- **Diff view**: What will change? Before/after state.
- **Risk indicators**: Cost, blast radius, reversibility, confidence score
- **One-click approve/reject**: Minimize friction or humans will rubber-stamp everything
- **Modification ability**: Human should be able to edit the proposed action, not just yes/no
- **Timeout handling**: What happens if no one reviews? Auto-reject? Escalate? Default safe action?
- **Audit trail**: Every approval/rejection logged with timestamp and rationale

### The Rubber-Stamp Problem

If approval rates exceed 95%, one of these is true:
1. The agent is very good → consider removing the gate
2. Humans are rubber-stamping → the gate is security theatre
3. Selection bias → agent is only surfacing easy cases

**Countermeasure**: Periodically inject "test" decisions that should be rejected. Track rejection rates. If reviewers never reject, the gate isn't working.

---

## Spending Limits & Financial Controls

### Tiered Spending Authority

```yaml
spending_tiers:
  auto_approve:
    max_per_transaction: $50
    max_per_day: $200
    max_per_month: $1000
    
  notify_after:
    max_per_transaction: $500
    max_per_day: $1000
    requires: transaction_logging
    
  require_approval:
    max_per_transaction: $5000
    approver: team_lead
    timeout: 4h
    
  executive_approval:
    above: $5000
    approver: finance_director
    timeout: 24h
    requires: [budget_code, justification, vendor_verification]
```

### Budget Envelope Model

Give agents a **budget envelope** — a pre-approved spending pool for a defined scope:

- "You have $500/month for cloud infrastructure scaling decisions"
- "You have $100/day for customer retention offers"
- "You have $0 for anything not in your approved vendor list"

The envelope depletes as the agent spends. When it's low, the agent notifies. When it's empty, all spending requires approval. Refills on a schedule or by request.

### Rate Limiting

Beyond dollar amounts, limit:
- Number of transactions per time window
- Cumulative exposure across correlated decisions  
- Velocity (sudden spike in spending = pause and flag)

---

## Deployment Gates

### Progressive Deployment with Human Checkpoints

```
Code Change → Auto-tests → [Gate: Staging Deploy] → Canary (1%) → [Gate: Human Review of Metrics] → Gradual Rollout (10%, 50%, 100%)
```

**Automated gates** (no human needed):
- Unit tests pass
- Integration tests pass
- Security scan clean
- Performance benchmarks within tolerance
- No new dependency vulnerabilities

**Human gates** (always require sign-off):
- First deployment of a new service
- Database migration in production
- Changes to authentication/authorization
- Changes affecting billing or payments
- Breaking API changes

**Conditional gates** (human review if criteria met):
- Canary metrics deviate >2σ from baseline
- Error rate increases >0.1%
- Latency p99 increases >20%
- New code paths touch sensitive modules

### Rollback Authority

Agents should **always** have authority to rollback without approval. The cost of a false rollback is much lower than the cost of a bad deployment continuing.

---

## Customer-Facing Output Review

### The Highest-Stakes Category

Customer-facing content is where HITL matters most. A wrong API call is a bug. A wrong email to a customer is a reputation event.

### Graduated Trust Model

**Phase 1: Full review** — Every customer-facing output is reviewed before sending. Agent drafts, human sends. Build a dataset of what "good" looks like.

**Phase 2: Template-based auto-send** — Agent can auto-send responses that match pre-approved templates exactly. Novel responses still require review.

**Phase 3: Confidence-based** — Agent auto-sends when confidence is high and the response matches established patterns. Edge cases routed for review.

**Phase 4: Audit-based** — Agent sends freely. Random sample (e.g., 10%) reviewed retroactively. Anomalies flagged automatically.

**Phase 5: Exception-based** — Only flagged outputs reviewed. Agent has earned trust through track record.

### Content Review Signals

Auto-flag for human review if the output:
- Contains apologies or admissions of fault (legal implications)
- Mentions pricing, discounts, or refund amounts
- References competitors
- Contains technical claims about the product
- Deviates significantly from tone/style guidelines
- Is responding to an angry or escalated customer
- Includes any form of commitment or promise
- Is the first response to a new customer

---

## Escalation Paths

### Escalation Chain Design

```
L0: Agent handles autonomously
    ↓ (low confidence, edge case, policy exception)
L1: Senior agent / specialized agent reviews
    ↓ (still uncertain, high stakes, novel situation)
L2: Human operator reviews
    ↓ (policy decision, significant risk, complaint escalation)
L3: Team lead / manager
    ↓ (legal, PR risk, large financial impact)
L4: Executive / legal team
```

### Escalation Triggers

- **Explicit request**: Customer asks to speak to a human
- **Sentiment detection**: Anger, frustration, threat of legal action
- **Loop detection**: Agent has attempted resolution 3+ times without success
- **Scope creep**: Request falls outside agent's defined domain
- **Precedent-setting**: Decision would establish a new pattern
- **Conflict**: Agent's instructions conflict with each other
- **Time pressure**: Urgent matter that can't wait for normal processing

### Anti-Patterns

- **Escalation ping-pong**: Agent escalates to human, human sends back to agent, repeat. Fix: clear ownership rules.
- **Silent escalation**: Agent escalates but nobody notices. Fix: escalation SLAs with alerting.
- **Over-escalation**: Agent escalates everything. Fix: track escalation rates, retrain on resolved cases.
- **Dead-end escalation**: Escalated to a queue nobody monitors. Fix: escalation health checks.

---

## Notification Systems

### Notification Tiers

**1. Informational** (async, batched)
- Daily/weekly summary of agent actions
- Metrics and trends
- Channel: Email digest, dashboard

**2. Awareness** (async, timely)
- Agent completed a significant task
- Budget threshold approaching
- Unusual pattern detected
- Channel: Slack/Teams notification

**3. Action Required** (sync-ish, time-bound)
- Approval request with deadline
- Review queue item
- Channel: Push notification + Slack with reminder

**4. Urgent** (sync, immediate)
- Agent detected potential incident
- Security-related event
- Channel: PagerDuty, SMS, phone call

### Notification Fatigue

The biggest risk with HITL is **notification fatigue** leading to rubber-stamping. Mitigate by:

- Ruthlessly pruning informational notifications
- Batching where possible
- Making approval UIs fast and friction-free
- Tracking response times — if they're dropping, reduce volume
- Ensuring the signal-to-noise ratio stays high
- Rotating review duty across team members

---

## Balancing Autonomy with Oversight

### The Trust Ladder

Agents should **earn** autonomy over time:

```
New Agent → Supervised → Assisted → Autonomous → Trusted
   100% review   80% review   20% review   Audit-only   Exception-only
```

Movement up requires:
- Track record of correct decisions
- Low error/override rate over sustained period
- Passing adversarial testing
- Domain-specific certification (e.g., handles edge cases correctly)

Movement down happens when:
- Error rate spikes
- New domain or capability added
- External conditions change (new regulations, policy shifts)
- After any significant incident

### The "Newspaper Test"

Before auto-approving any agent action class, ask: **"If the agent got this wrong and it made the news, how bad would it be?"**

- Not newsworthy → safe to auto-approve
- Mildly embarrassing → notify-after with audit
- Industry news → require approval  
- Front page → mandatory multi-person approval

### Context-Dependent Autonomy

The same action might need different approval levels depending on context:

- Sending $100 to a known vendor → auto-approve
- Sending $100 to a new recipient → require approval
- Posting to internal Slack → auto-approve
- Posting to public Twitter → require approval
- Restarting a dev server → auto-approve
- Restarting a production server → require approval (unless incident response)

### Designing for the 3am Test

Your approval system must handle: **"It's 3am, the agent needs a decision, and no one is awake."**

Options:
1. **Default deny**: Agent waits. Safe but may miss SLAs.
2. **Default allow with limits**: Agent proceeds within conservative bounds, flags for morning review.
3. **Escalation cascade**: Try primary approver → backup → on-call → auto-approve with logging.
4. **Pre-authorized playbooks**: For known scenarios, pre-approve the response. Agent follows playbook without waiting.

Most production systems use a mix: pre-authorized playbooks for common 3am scenarios, default deny for everything else, with an on-call escalation path for true emergencies.

---

## Implementation Patterns

### Policy-as-Code

Define approval policies declaratively:

```yaml
policies:
  - name: customer_email_send
    action: send_email
    conditions:
      - target: external_customer
    approval:
      default: require_human
      auto_approve_when:
        - template: pre_approved
        - sentiment: positive
        - confidence: ">0.95"
        - customer_tier: ["standard"]
      always_require_when:
        - customer_tier: ["enterprise", "strategic"]
        - contains: ["refund", "legal", "escalat"]
        - first_contact: true
    timeout: 2h
    timeout_action: escalate
    
  - name: infrastructure_change
    action: modify_infrastructure
    approval:
      default: require_human
      auto_approve_when:
        - change_type: scale_up
        - within_budget: true
        - environment: ["dev", "staging"]
      never_auto_approve:
        - environment: production
        - change_type: [delete, migrate]
    timeout: 4h
    timeout_action: deny
```

### Approval Request Format

When an agent requests approval, it should provide:

```markdown
## Approval Request: [Action Type]

**What**: Send refund of $150 to customer #4521
**Why**: Customer reported defective product (order #8834). Matches refund policy criteria.
**Confidence**: 0.87 (below auto-approve threshold of 0.95)
**Risk**: Low financial ($150), Medium reputational (repeat customer)
**Alternatives considered**: 
  - Replacement shipment (customer preferred refund)
  - Store credit (policy allows full refund for defective items)
**Deadline**: 24h (customer SLA)

[Approve] [Reject] [Modify] [Escalate]
```

### Audit Trail Schema

```json
{
  "decision_id": "uuid",
  "timestamp": "2026-02-08T09:00:00Z",
  "agent_id": "agent-42",
  "action": "send_customer_email",
  "target": "customer#4521",
  "confidence": 0.87,
  "approval_type": "human_approved",
  "approver": "user@company.com",
  "approval_time_ms": 34000,
  "modifications": "Changed tone of second paragraph",
  "outcome": "approved_with_changes",
  "downstream_result": "email_sent_successfully"
}
```

---

## Key Recommendations

1. **Start restrictive, loosen over time.** It's easier to grant autonomy than to recover from a bad autonomous decision.

2. **Make approval fast and easy.** If reviewing takes more than 30 seconds, humans will either rubber-stamp or avoid the queue. One-click approve with full context visible.

3. **Measure your gates.** Track: approval rate, time-to-approve, override rate, false positive rate. Remove gates that never reject. Add gates where errors cluster.

4. **Design for failure modes.** What happens when the approver is unavailable? When the queue backs up? When the agent's confidence calibration drifts? Plan for it.

5. **Separate "what" from "how".** Humans should approve *what* to do (strategy, goals, constraints). Agents should figure out *how* (execution, optimization, routine decisions).

6. **Budget envelopes > per-transaction approvals.** Give agents pre-approved resource budgets. Faster, less fatiguing, and teaches agents to optimize within constraints.

7. **The goal is earned trust, not permanent oversight.** Every approval gate should have criteria for when it can be relaxed. If an agent has 1000 correct decisions with zero overrides, it's earned more autonomy.

8. **Keep humans in the loop, not in the way.** The best HITL systems are invisible when things go right and immediately present when things go wrong.

---

## References & Further Reading

- Anthropic, "Building Effective Agents" (2025) — patterns for agent autonomy and orchestration
- Humanloop, "What is Human-in-the-Loop AI?" — HITL training and deployment approaches  
- Google DeepMind, "Scalable Oversight" research program — theoretical foundations for human oversight of AI systems
- Temporal.io — durable workflow patterns applicable to approval queue implementation
- AWS Step Functions — state machine patterns for human approval gates in cloud workflows
- NIST AI Risk Management Framework — governance frameworks for AI decision-making authority levels
