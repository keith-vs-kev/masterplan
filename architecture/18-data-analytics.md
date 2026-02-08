# 18 — Data & Analytics Platform

> The factory's central nervous system for numbers. Every token burned, dollar earned, task completed, and product shipped flows through here. Dash owns this domain — ingesting, modelling, interpreting, and reporting autonomously.

---

## Executive Summary

An AI factory without analytics is flying blind. This architecture defines how we track **business metrics** (revenue, profit, ROI per product), **agent performance** (cost per agent, task completion, efficiency), **revenue attribution** (which products/channels generate money), and **cost control** (per-agent budgets, velocity alerts, kill switches). The stack is deliberately minimal: **TimescaleDB** (time-series + relational in one Postgres-based store), **Redis** (real-time accumulators and budget enforcement), and **Grafana** (dashboards). Dash itself is the intelligence layer — querying, interpreting, reporting, and recommending.

---

## 1. Design Principles

1. **Single source of truth** — All metrics flow through one pipeline. No shadow spreadsheets, no duplicate counters.
2. **Event-sourced** — Raw events are immutable, append-only. Every aggregation is a derived view that can be recomputed.
3. **Agent-native** — Dash doesn't just display data; it reads, reasons about, and acts on it. Dashboards are generated programmatically.
4. **Cost-aware by default** — Every factory action has a cost. Track it automatically at the task level, not just the API call level.
5. **Revenue-attributed** — Every dollar traces back to the product, agent, and campaign that generated it.
6. **Minimum viable first** — Start with what matters (cost tracking + revenue tracking), add sophistication later.

---

## 2. Database Architecture

### Why TimescaleDB (Not Separate Systems)

The factory generates two kinds of data: **time-series** (token usage per minute, revenue per hour, error rates) and **relational** (products, agents, customers, invoices). Most stacks force a split — Prometheus for metrics, Postgres for business data, InfluxDB for time-series — then you spend forever gluing them together with ETL.

TimescaleDB is PostgreSQL with time-series superpowers. One database, one query language, both workloads.

- **Hypertables** — automatic partitioning and compression for time-series data
- **Continuous aggregates** — pre-computed rollups (hourly → daily → monthly) that refresh automatically
- **Standard SQL** — Dash can query it directly, no special APIs or query languages
- **Compression** — 90%+ on older data, storage stays cheap
- **Retention policies** — auto-drop raw data after N days, keep aggregates forever

### Supporting Stores

| Store | Purpose | Data |
|-------|---------|------|
| **TimescaleDB** | Primary analytical store | All metrics, events, business data |
| **Redis** | Real-time accumulators & budget enforcement | Live spend counters, rate limits, kill switches |
| **S3-compatible** (MinIO) | Archive & report storage | Raw event logs, generated reports, CSV exports |

---

## 3. Data Model

### 3.1 Core Entities (Relational)

```sql
-- Products the factory builds and sells
CREATE TABLE products (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            TEXT NOT NULL,
    type            TEXT NOT NULL,       -- 'micro-saas', 'api', 'seo-site', 'extension', 'digital-product'
    status          TEXT NOT NULL DEFAULT 'active',
    launched_at     TIMESTAMPTZ,
    revenue_model   TEXT,                -- 'subscription', 'usage', 'one-time', 'ads', 'affiliate'
    monthly_cost    NUMERIC(10,2),       -- estimated infra cost
    metadata        JSONB,
    created_at      TIMESTAMPTZ DEFAULT now()
);

-- Factory agents
CREATE TABLE agents (
    id              TEXT PRIMARY KEY,    -- 'kev', 'dash', 'mae', etc.
    role            TEXT NOT NULL,       -- 'builder', 'marketer', 'ops', 'analytics'
    default_model   TEXT,                -- 'claude-opus-4-6', 'gpt-4o-mini'
    daily_budget    NUMERIC(10,2),       -- daily spend limit USD
    status          TEXT DEFAULT 'active',
    metadata        JSONB,
    created_at      TIMESTAMPTZ DEFAULT now()
);

-- Revenue channels (Stripe, Gumroad, AdSense, etc.)
CREATE TABLE revenue_channels (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    product_id      UUID REFERENCES products(id),
    channel_type    TEXT NOT NULL,       -- 'stripe', 'gumroad', 'adsense', 'affiliate'
    channel_config  JSONB,              -- encrypted at rest
    created_at      TIMESTAMPTZ DEFAULT now()
);
```

### 3.2 Time-Series Tables (Hypertables)

```sql
-- Every LLM API call (the most important table)
CREATE TABLE llm_calls (
    time            TIMESTAMPTZ NOT NULL,
    agent_id        TEXT NOT NULL,
    task_id         UUID,
    model           TEXT NOT NULL,
    provider        TEXT NOT NULL,
    input_tokens    INTEGER NOT NULL,
    output_tokens   INTEGER NOT NULL,
    cached_tokens   INTEGER DEFAULT 0,
    cost_usd        NUMERIC(10,6) NOT NULL,
    latency_ms      INTEGER NOT NULL,
    status          TEXT NOT NULL,       -- 'success', 'error', 'timeout', 'rate_limited'
    error_type      TEXT,
    metadata        JSONB
);
SELECT create_hypertable('llm_calls', 'time');

-- Revenue events (payments, subscriptions, refunds, ad revenue)
CREATE TABLE revenue_events (
    time            TIMESTAMPTZ NOT NULL,
    product_id      UUID NOT NULL,
    channel_id      UUID,
    event_type      TEXT NOT NULL,       -- 'payment', 'subscription_start', 'subscription_cancel',
                                         -- 'refund', 'ad_impression', 'affiliate_conversion'
    amount_usd      NUMERIC(10,2),
    currency        TEXT DEFAULT 'USD',
    customer_id     TEXT,
    metadata        JSONB
);
SELECT create_hypertable('revenue_events', 'time');

-- Agent task executions (cost-at-task-level)
CREATE TABLE task_events (
    time            TIMESTAMPTZ NOT NULL,
    task_id         UUID NOT NULL,
    agent_id        TEXT NOT NULL,
    product_id      UUID,
    task_type       TEXT NOT NULL,       -- 'build', 'deploy', 'fix-bug', 'write-content', 'research'
    status          TEXT NOT NULL,       -- 'started', 'completed', 'failed', 'escalated'
    duration_ms     INTEGER,
    total_cost_usd  NUMERIC(10,4),      -- total LLM + tool cost for this task
    steps_count     INTEGER,
    quality_score   NUMERIC(3,2),       -- 0.00–1.00 from eval
    metadata        JSONB
);
SELECT create_hypertable('task_events', 'time');

-- Infrastructure costs (hosting, domains, services)
CREATE TABLE infra_costs (
    time            TIMESTAMPTZ NOT NULL,
    product_id      UUID,               -- NULL for shared infra
    cost_type       TEXT NOT NULL,       -- 'hosting', 'domain', 'database', 'cdn', 'email'
    provider        TEXT NOT NULL,
    amount_usd      NUMERIC(10,4) NOT NULL,
    billing_period  TEXT,
    metadata        JSONB
);
SELECT create_hypertable('infra_costs', 'time');

-- Marketing & growth metrics
CREATE TABLE growth_events (
    time            TIMESTAMPTZ NOT NULL,
    product_id      UUID NOT NULL,
    channel         TEXT NOT NULL,       -- 'organic', 'reddit', 'twitter', 'referral', 'direct'
    metric          TEXT NOT NULL,       -- 'pageview', 'signup', 'trial_start', 'conversion', 'churn'
    value           NUMERIC(10,2) DEFAULT 1,
    metadata        JSONB
);
SELECT create_hypertable('growth_events', 'time');
```

### 3.3 Continuous Aggregates

```sql
-- Hourly agent cost rollup (the most-queried aggregate)
CREATE MATERIALIZED VIEW agent_costs_hourly
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 hour', time) AS bucket,
    agent_id,
    model,
    provider,
    COUNT(*) AS call_count,
    SUM(input_tokens) AS total_input_tokens,
    SUM(output_tokens) AS total_output_tokens,
    SUM(cost_usd) AS total_cost,
    AVG(latency_ms) AS avg_latency,
    COUNT(*) FILTER (WHERE status != 'success') AS error_count
FROM llm_calls
GROUP BY bucket, agent_id, model, provider;

-- Daily revenue per product
CREATE MATERIALIZED VIEW revenue_daily
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 day', time) AS bucket,
    product_id,
    event_type,
    SUM(amount_usd) AS total_revenue,
    COUNT(*) AS event_count,
    COUNT(DISTINCT customer_id) AS unique_customers
FROM revenue_events
WHERE amount_usd > 0
GROUP BY bucket, product_id, event_type;

-- Daily product P&L (the money view)
CREATE VIEW product_pnl_daily AS
SELECT
    COALESCE(r.bucket, t.bucket, i.bucket) AS day,
    p.name AS product_name,
    p.type AS product_type,
    COALESCE(r.total_revenue, 0) AS revenue,
    COALESCE(t.total_agent_cost, 0) AS agent_cost,
    COALESCE(i.total_infra_cost, 0) AS infra_cost,
    COALESCE(r.total_revenue, 0)
        - COALESCE(t.total_agent_cost, 0)
        - COALESCE(i.total_infra_cost, 0) AS net_profit
FROM (
    SELECT bucket, product_id, SUM(total_revenue) AS total_revenue
    FROM revenue_daily GROUP BY 1, 2
) r
FULL JOIN (
    SELECT time_bucket('1 day', time) AS bucket, product_id,
           SUM(total_cost_usd) AS total_agent_cost
    FROM task_events WHERE status = 'completed' GROUP BY 1, 2
) t ON r.bucket = t.bucket AND r.product_id = t.product_id
FULL JOIN (
    SELECT time_bucket('1 day', time) AS bucket, product_id,
           SUM(amount_usd) AS total_infra_cost
    FROM infra_costs GROUP BY 1, 2
) i ON COALESCE(r.bucket, t.bucket) = i.bucket
       AND COALESCE(r.product_id, t.product_id) = i.product_id
JOIN products p ON p.id = COALESCE(r.product_id, t.product_id, i.product_id);
```

---

## 4. Ingestion Pipeline

### Architecture

```
┌─────────────────────────────────────────────────────┐
│                    Data Sources                      │
├──────────┬──────────┬──────────┬──────────┬─────────┤
│ LLM Proxy│ Stripe   │ Agent    │ Infra    │ Web     │
│ (LiteLLM)│ Webhooks │ Events   │ Billing  │Analytics│
└────┬─────┴────┬─────┴────┬─────┴────┬─────┴────┬────┘
     │          │          │          │           │
     ▼          ▼          ▼          ▼           ▼
┌─────────────────────────────────────────────────────┐
│            Ingestion Layer (Node.js/Bun)             │
│                                                      │
│  Webhook Receiver  │  Polling Collectors  │  Events  │
│                    ▼                                 │
│           Normalizer & Router                        │
│      (validate, enrich, tag product/agent)           │
│                    │                                 │
│        ┌───────────┼───────────┐                     │
│        ▼           ▼           ▼                     │
│   TimescaleDB    Redis      S3 Archive               │
│   (analytics)   (real-time) (raw backup)             │
└─────────────────────────────────────────────────────┘
```

### Ingestion Sources

| Source | Method | Frequency | Data |
|--------|--------|-----------|------|
| **LiteLLM** | Callback hook | Real-time | Every LLM call: tokens, cost, latency, model |
| **Stripe** | Webhooks | Real-time | Payments, subscriptions, refunds |
| **Gumroad / Lemon Squeezy** | Webhooks | Real-time | Digital product sales |
| **Google AdSense** | API poll | Hourly | Ad revenue, impressions |
| **Agent event bus** | Redis Streams | Real-time | Task start/complete/fail |
| **Infra billing** | API poll | Daily | Vercel, Railway, Cloudflare costs |
| **PostHog / Plausible** | API poll | Hourly | Pageviews, signups, feature usage |
| **Google Search Console** | API poll | Daily | Search impressions, clicks, rankings |
| **GitHub** | Webhooks | Real-time | Commits, PRs, deployments |

### Real-Time Accumulators (Redis)

For metrics that need sub-second reads — budget enforcement, live dashboards, kill switches:

```
dash:spend:daily:{agent_id}     → Running total USD today
dash:spend:hourly:{agent_id}    → Rolling 1-hour spend
dash:revenue:daily:{product_id} → Revenue today
dash:revenue:mtd                → Month-to-date total revenue
dash:errors:5min:{agent_id}     → Error count in sliding 5-min window
dash:velocity:{agent_id}        → $/min rolling average
```

Write-through pattern: every event updates both Redis (speed) and TimescaleDB (history).

---

## 5. KPIs & Metrics Framework

### 5.1 Factory-Level KPIs (The Numbers That Matter)

| KPI | Formula | Target | Cadence |
|-----|---------|--------|---------|
| **Total Revenue** | Sum all revenue events | Growing MoM | Daily |
| **Total Cost** | LLM + infra + services | < 30% of revenue | Daily |
| **Net Profit** | Revenue − Cost | Positive, growing | Daily |
| **Gross Margin** | (Revenue − Cost) / Revenue | > 70% | Weekly |
| **Revenue per Product** | Revenue / active products | > $500/mo avg | Weekly |
| **Cost per Agent** | Total LLM cost / agent | Optimize outliers | Daily |
| **ROI per Product** | (Revenue − Cost) / Cost × 100 | > 200% | Weekly |
| **Products Shipped** | Products launched this period | 2–4/month | Monthly |
| **Time to Revenue** | Days from idea → first dollar | < 14 days | Per product |
| **Portfolio Hit Rate** | Profitable products / launched | > 30% | Quarterly |

### 5.2 Agent Performance KPIs

| KPI | Formula | Target |
|-----|---------|--------|
| **Task Completion Rate** | Completed / (Completed + Failed + Escalated) | > 90% |
| **Avg Cost per Task** | Total LLM cost / completed tasks | Decreasing trend |
| **Avg Steps per Task** | Mean steps for completed tasks | Lower = more efficient |
| **Error Rate** | Failed calls / total calls | < 5% |
| **Escalation Rate** | Human-escalated / total tasks | < 10% |
| **Quality Score** | Mean quality_score from evals | > 0.8 |
| **Cache Hit Rate** | Cached tokens / total input tokens | > 30% |
| **Model Efficiency** | % tasks using cheapest viable model | Increasing |
| **Cost Velocity** | $/minute rolling average | Stable, no spikes |

### 5.3 Product Health KPIs

| KPI | Formula | Target |
|-----|---------|--------|
| **MRR** | Monthly recurring revenue | Growing |
| **Churn Rate** | Cancellations / total subscribers | < 5%/mo |
| **CAC** | Marketing cost / new customers | < 3 months LTV |
| **LTV** | Avg revenue × avg lifespan | > 3× CAC |
| **Conversion Rate** | Signups → paid | > 5% |
| **DAU/MAU** | Daily active / monthly active | > 20% |
| **Uptime** | Available / total minutes | > 99.5% |

### 5.4 Growth KPIs

| KPI | Formula | Target |
|-----|---------|--------|
| **Organic Traffic** | Search pageviews | Growing 10%+ MoM |
| **Keyword Rankings** | Keywords in top 10 | Increasing |
| **Backlinks** | New referring domains | 10+/month |
| **Email List** | Total subscribers | Growing |
| **Referral Rate** | Users who refer / total | > 5% |

---

## 6. Cost Per Agent — Deep Dive

Cost per agent is the most actionable metric in the factory. It answers: "Is this agent earning its keep?"

### Tracking Layers

```
Per-call cost    →  What each LLM API call costs (from LiteLLM)
Per-task cost    →  Sum of all calls in a task (the useful unit)
Per-agent cost   →  Sum of all tasks by agent (daily/weekly/monthly)
Per-product cost →  Sum of all agent costs attributed to a product
```

### Agent Cost Attribution

When an agent works on a specific product, the cost is straightforward. When an agent does cross-cutting work (Dash running analytics, Mae coordinating), use proportional attribution:

```sql
-- Agent cost breakdown: direct vs shared
SELECT
    agent_id,
    SUM(total_cost_usd) FILTER (WHERE product_id IS NOT NULL) AS direct_cost,
    SUM(total_cost_usd) FILTER (WHERE product_id IS NULL) AS shared_cost,
    SUM(total_cost_usd) AS total_cost,
    COUNT(*) AS task_count,
    AVG(total_cost_usd) AS avg_cost_per_task
FROM task_events
WHERE time >= now() - interval '7 days'
  AND status = 'completed'
GROUP BY agent_id
ORDER BY total_cost DESC;
```

### Budget Enforcement

Each agent gets a daily budget. Enforced at the LLM proxy layer via Redis:

```python
class BudgetEnforcer:
    async def check(self, agent_id, estimated_cost):
        spent = float(await redis.get(f"dash:spend:daily:{agent_id}") or 0)
        limit = await self.get_daily_limit(agent_id)

        if spent + estimated_cost > limit:
            raise BudgetExceededError(f"{agent_id} at ${spent:.2f}/{limit:.2f}")

        velocity = float(await redis.get(f"dash:velocity:{agent_id}") or 0)
        if velocity > self.velocity_threshold:
            raise RunawayDetectedError(f"{agent_id} burning ${velocity:.2f}/min")
```

### Cost Optimization Signals

Dash monitors for optimization opportunities:

- **Model downgrade candidates** — Tasks completing successfully on expensive models that could use cheaper ones
- **Cache misses** — Repeated similar queries that should be cached
- **Long context bloat** — Input tokens growing across iterations (agent feeding output back as input)
- **Retry waste** — High retry rates multiplying costs silently
- **Idle agents** — Agents consuming budget but completing few tasks

---

## 7. Revenue Tracking

### Revenue Flow

```
Customer pays → Stripe/Gumroad webhook → Ingestion layer → revenue_events table
                                              ↓
                                         Redis accumulator (dash:revenue:daily:{product_id})
                                              ↓
                                         Continuous aggregate (revenue_daily)
                                              ↓
                                         P&L view (product_pnl_daily)
```

### Revenue Attribution Chain

Every dollar maps to: **Channel** → **Product** → **Revenue Model** → **Customer**

```sql
-- Revenue breakdown: what's making money and how
SELECT
    p.name AS product,
    p.revenue_model,
    rc.channel_type,
    SUM(re.amount_usd) AS revenue,
    COUNT(DISTINCT re.customer_id) AS customers,
    SUM(re.amount_usd) / NULLIF(COUNT(DISTINCT re.customer_id), 0) AS revenue_per_customer
FROM revenue_events re
JOIN products p ON p.id = re.product_id
LEFT JOIN revenue_channels rc ON rc.id = re.channel_id
WHERE re.time >= now() - interval '30 days'
  AND re.amount_usd > 0
GROUP BY p.name, p.revenue_model, rc.channel_type
ORDER BY revenue DESC;
```

### Per-Product P&L

The core business question — is each product profitable?

```
Revenue (from revenue_events)
− Agent costs (from task_events, attributed to product)
− Infrastructure costs (from infra_costs, attributed to product)
= Net profit per product per day
```

### Payback Period Tracking

For each product, track cumulative revenue vs cumulative cost from launch:

```sql
-- Payback analysis: when does a product break even?
SELECT
    day,
    product_name,
    SUM(revenue) OVER w AS cumulative_revenue,
    SUM(agent_cost + infra_cost) OVER w AS cumulative_cost,
    SUM(net_profit) OVER w AS cumulative_profit
FROM product_pnl_daily
WHERE product_name = 'ProductX'
WINDOW w AS (ORDER BY day ROWS UNBOUNDED PRECEDING)
ORDER BY day;
```

---

## 8. Dashboard Architecture

### Autonomous Dashboard Management

Dash creates and maintains dashboards programmatically via the Grafana HTTP API. Dashboard definitions are stored as JSON in git — versioned and reviewable.

When a new product launches, Dash auto-generates its dashboard from a template. When anomalies occur, Dash adds annotations to the relevant graphs.

### Dashboard Hierarchy

```
Factory Overview (single pane of glass)
├── Financial Dashboard
│   ├── Revenue (total, by product, by channel)
│   ├── Costs (LLM, infra, by agent, by product)
│   ├── P&L per product
│   └── Cash flow & runway
├── Agent Performance
│   ├── Per-agent cost & efficiency
│   ├── Task completion rates
│   ├── Model usage breakdown
│   └── Error rates & anomalies
├── Product Dashboards (auto-generated per product)
│   ├── Revenue & growth
│   ├── User funnel (visit → signup → paid → retain)
│   ├── Feature usage
│   └── Support & satisfaction
├── Growth Dashboard
│   ├── SEO (rankings, traffic, backlinks)
│   ├── Channel attribution
│   ├── Conversion funnels
│   └── Content performance
└── Ops Health
    ├── Uptime & latency
    ├── Deploy frequency
    ├── Error budgets
    └── Infra costs
```

### Key Dashboard: Factory Overview

What Grafana shows at a glance:

```
┌──────────────────────────────────────────────┐
│  AI Factory Dashboard                         │
├──────────────┬──────────────┬────────────────┤
│ TODAY: $47.23│ MTD: $892.41 │ Budget: 78%    │
├──────────────┴──────────────┴────────────────┤
│  Cost/hour (24h)        [sparkline]          │
│  Active agents: 12/15   [status dots]        │
│  Error rate: 2.1%       [↓ improving]        │
│  Avg latency: 1.2s      [→ stable]          │
│  Cache hit rate: 34%    [↑ improving]        │
├──────────────────────────────────────────────┤
│  Top Cost Agents (today)                      │
│  1. research-agent   $12.40  ████████░░      │
│  2. code-review      $8.20   █████░░░░░      │
│  3. email-assistant  $6.10   ████░░░░░░      │
├──────────────────────────────────────────────┤
│  Top Revenue Products (MTD)                   │
│  1. SEO tool         $340    ██████████       │
│  2. API service      $280    ████████░░       │
│  3. Chrome ext       $190    ██████░░░░       │
├──────────────────────────────────────────────┤
│  Alerts (24h)                                 │
│  ⚠️  research-agent exceeded velocity limit   │
│  ✅  Resolved: API rate limit on OpenAI       │
└──────────────────────────────────────────────┘
```

---

## 9. Automated Reporting

### Report Types

| Report | Cadence | Content |
|--------|---------|---------|
| **Daily Digest** | Daily 08:00 | Yesterday's revenue, costs, alerts, notable events |
| **Weekly P&L** | Monday 09:00 | Per-product P&L, trends, agent efficiency, recommendations |
| **Monthly Review** | 1st of month | Full financials, portfolio review, growth, strategy |
| **Agent Report** | Weekly | Per-agent metrics, optimization suggestions |
| **Anomaly Alert** | Real-time | Cost spikes, revenue drops, error surges |

### Report Generation

```
Trigger (cron or anomaly) → Query TimescaleDB → Compare vs targets/trends
    → Generate narrative (LLM on the numbers) → Format (Markdown/PDF)
    → Deliver (WhatsApp/email) → Archive to S3
```

### Anomaly Detection & Alerting

| Condition | Threshold | Action |
|-----------|-----------|--------|
| Cost velocity spike | > 3× rolling avg $/min | Alert + auto-pause agent |
| Revenue drop | > 30% day-over-day | Alert human |
| Error rate surge | > 10% in 5-min window | Alert + investigate |
| Product downtime | > 5 min unresponsive | Alert + trigger ops |
| Churn spike | > 2× avg daily cancellations | Alert + flag |
| Budget threshold | Agent at 80% daily budget | Warn; pause at 100% |
| Zero revenue | Product earning $0 for 48h+ | Flag for review |

---

## 10. ROI Calculation Engine

### Per-Product ROI

```sql
SELECT
    p.name,
    SUM(r.amount_usd) AS total_revenue,
    SUM(t.total_cost_usd) AS agent_costs,
    SUM(i.amount_usd) AS infra_costs,
    SUM(r.amount_usd) - SUM(t.total_cost_usd) - SUM(i.amount_usd) AS net_profit,
    CASE
        WHEN (SUM(t.total_cost_usd) + SUM(i.amount_usd)) > 0
        THEN ((SUM(r.amount_usd) - SUM(t.total_cost_usd) - SUM(i.amount_usd))
              / (SUM(t.total_cost_usd) + SUM(i.amount_usd))) * 100
        ELSE NULL
    END AS roi_percent
FROM products p
LEFT JOIN revenue_events r ON r.product_id = p.id
LEFT JOIN task_events t ON t.product_id = p.id AND t.status = 'completed'
LEFT JOIN infra_costs i ON i.product_id = p.id
GROUP BY p.id, p.name
ORDER BY net_profit DESC;
```

### Per-Agent ROI

Agents contribute to multiple products. Attribution model:

```
Agent ROI = (Revenue of attributed products × attribution weight) − Agent cost
            ─────────────────────────────────────────────────────────────────
                                    Agent cost

Attribution weight = Agent's task hours on product / Total task hours on product
```

### Build vs Buy Signals

For each product, Dash tracks:
- **Build cost** — total agent + human time from idea → launch
- **Ongoing cost** — monthly agent + infra maintenance
- **Revenue trajectory** — daily revenue curve since launch
- **Payback period** — days until cumulative revenue > cumulative cost
- **Break-even projection** — extrapolated from current trajectory

---

## 11. Minimum Viable Analytics Stack

### What to Deploy First

The full architecture above is the target. Here's what to deploy on day one:

#### Tier 1: Non-Negotiable (Week 1)

1. **LiteLLM as proxy** — route all LLM calls through it. Gives you per-call cost logging for free.
2. **TimescaleDB** — single Docker container. Create `llm_calls` and `revenue_events` tables.
3. **Redis** — for `dash:spend:daily:{agent_id}` accumulators and budget enforcement.
4. **Budget enforcer** — ~100 lines of code in the proxy layer. Hard daily caps per agent.
5. **Stripe webhook receiver** — pipe payments into `revenue_events`.

This alone gives you: cost tracking, revenue tracking, per-agent budgets, and runaway protection.

#### Tier 2: Visibility (Week 2–3)

6. **Grafana** — connect to TimescaleDB. Build the Factory Overview dashboard.
7. **Continuous aggregates** — `agent_costs_hourly` and `revenue_daily`.
8. **Task events ingestion** — wire agent event bus into `task_events`.
9. **Daily digest** — Dash generates and sends a morning summary.

Now you can see what's happening and get alerted.

#### Tier 3: Intelligence (Week 4–6)

10. **Anomaly detection** — cost velocity, error rate, revenue drop alerts.
11. **Product P&L view** — the `product_pnl_daily` aggregate.
12. **Weekly P&L report** — Dash generates per-product profitability analysis.
13. **Infra cost polling** — pull costs from Vercel, Railway, Cloudflare APIs.

#### Tier 4: Advanced (Month 2+)

14. **Auto-generated product dashboards**
15. **Growth metrics ingestion** (Search Console, PostHog)
16. **Predictive analytics** (revenue forecasting, cost projection)
17. **Automated recommendations** ("sunset X, double down on Y")

### Infrastructure Requirements (Minimum)

| Component | Spec | Notes |
|-----------|------|-------|
| TimescaleDB | 2 CPU, 4GB RAM, 100GB SSD | Grows with data; compression helps |
| Redis | 1 CPU, 512MB RAM | Just accumulators + flags |
| Grafana | 1 CPU, 512MB RAM | Lightweight dashboard server |
| Ingestion service | Bun/Node process | Runs alongside factory services |

Total: fits on a single $20/mo VPS alongside other factory services.

---

## 12. Data Retention Policy

| Data | Raw Retention | Aggregate Retention |
|------|---------------|---------------------|
| LLM calls | 90 days | Hourly: 1 year, Daily: forever |
| Revenue events | Forever | Daily: forever |
| Task events | 180 days | Daily: forever |
| Infra costs | Forever | Monthly: forever |
| Growth events | 90 days | Daily: forever |

---

## 13. Security

- TimescaleDB: encrypted at rest, TLS in transit
- Revenue data: access restricted to Dash + human admin
- API keys: stored in encrypted config, never in analytics DB
- Grafana: behind auth, admin-only initially
- Redis: not exposed externally, bound to localhost/Docker network

---

## 14. Dash's Analytical Capabilities

Dash isn't just a pipeline — it's the analyst. Key capabilities:

1. **Natural language queries** — "What was our best product last month?" → Dash writes SQL, runs it, interprets
2. **Trend detection** — "Revenue up 15% but costs up 40% — margin compression on product X"
3. **Anomaly explanation** — "Cost spike at 14:32 was Kev stuck in a retry loop on the SEO crawler"
4. **Recommendations** — "Product Y negative ROI for 3 weeks. Recommend sunset or pivot."
5. **Forecasting** — "At current growth, product X hits $1K MRR by March 15"
6. **Cross-metric correlation** — "Reddit post → 3× signup rate for 48h"

### Data Access Pattern

```
Dash receives question/trigger
    → Determines what data is needed
    → Writes SQL against TimescaleDB
    → Checks Redis for real-time values if needed
    → Interprets results with LLM reasoning
    → Generates narrative + Grafana link
    → Delivers to appropriate channel
```

---

## 15. Implementation Plan

### Phase 1: Foundation (Week 1–2)
- [ ] Deploy TimescaleDB + Redis (Docker)
- [ ] Create schema (tables, hypertables)
- [ ] Build ingestion: LiteLLM callback → TimescaleDB
- [ ] Wire Stripe webhooks → revenue_events
- [ ] Implement budget enforcer in proxy layer
- [ ] Deploy Grafana, create Factory Overview dashboard

### Phase 2: Intelligence (Week 3–4)
- [ ] Build Dash's SQL query interface
- [ ] Implement daily digest report
- [ ] Implement anomaly detection (cost velocity, error rate)
- [ ] Set up alerting (Dash → WhatsApp)
- [ ] Create continuous aggregates
- [ ] Wire agent event bus → task_events

### Phase 3: Full Analytics (Week 5–8)
- [ ] Auto-create product dashboards on launch
- [ ] Weekly P&L + monthly review reports
- [ ] ROI calculation engine
- [ ] Growth metrics ingestion
- [ ] Infra cost polling
- [ ] Historical backfill

### Phase 4: Advanced (Month 3+)
- [ ] Predictive analytics
- [ ] Automated recommendations
- [ ] A/B test analysis
- [ ] Self-improving metric selection

---

*Architecture designed: 2026-02-08*
*Companion research: 14-monitoring-observability, 11-revenue-generation, 07-ai-factory-patterns*
