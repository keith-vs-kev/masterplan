# 18 — Data & Analytics Platform

> How Dash (analytics agent) tracks everything the factory does — business metrics, agent performance, revenue, costs, ROI — and builds dashboards and reports autonomously.

---

## Executive Summary

The factory needs a central nervous system for numbers. Every agent action, every dollar earned, every token burned, every product shipped generates data. Dash ingests it all, models it, and surfaces insights — autonomously building dashboards, generating reports, and alerting when something's off. The stack: **TimescaleDB** (time-series + relational in one), **Redis** for real-time accumulators, **Grafana** for visualization, and Dash itself as the intelligence layer that interprets, reports, and recommends.

---

## 1. Design Principles

1. **Single source of truth** — All metrics flow through one pipeline. No shadow spreadsheets.
2. **Event-sourced** — Raw events are immutable and append-only. Aggregations are derived views.
3. **Agent-native** — Dash doesn't just display data; it reads it, reasons about it, and acts on it.
4. **Cost-aware by default** — Every action in the factory has a cost. Track it automatically.
5. **Revenue-attributed** — Every dollar of revenue traces back to the product, agent, and campaign that generated it.

---

## 2. Database Architecture

### Why TimescaleDB

The factory generates two kinds of data: **time-series** (token usage per minute, revenue per hour, error rates) and **relational** (products, agents, customers, invoices). Most stacks force you to pick one (Prometheus for metrics, Postgres for business data) and glue them together. TimescaleDB is PostgreSQL with time-series superpowers — one database, one query language, both workloads.

- **Hypertables** for time-series data with automatic partitioning and compression
- **Continuous aggregates** for pre-computed rollups (hourly → daily → monthly)
- **Standard SQL** — Dash can query it directly, no special APIs
- **Compression** — 90%+ compression on older data, keeps storage costs low
- **Retention policies** — Auto-drop raw data after N days, keep aggregates forever

### Supporting Stores

| Store | Purpose | Data |
|-------|---------|------|
| **TimescaleDB** | Primary analytical store | All metrics, events, business data |
| **Redis** | Real-time accumulators & kill switches | Live spend counters, rate limits, feature flags |
| **S3-compatible** (MinIO) | Raw event archive & report storage | Event logs, generated PDFs, CSV exports |

### Why NOT a Separate Time-Series DB?

Prometheus/InfluxDB are great for infrastructure metrics, but the factory's data is inherently mixed. "Revenue per agent per day" requires joining time-series (daily revenue) with relational (agent metadata, product catalog). Keeping it in one Postgres-based store avoids the ETL tax of syncing between systems.

---

## 3. Data Model

### 3.1 Core Entities (Relational)

```sql
-- The products the factory builds and sells
CREATE TABLE products (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            TEXT NOT NULL,
    type            TEXT NOT NULL,  -- 'micro-saas', 'api', 'seo-site', 'extension', 'digital-product'
    status          TEXT NOT NULL DEFAULT 'active',  -- 'idea', 'building', 'launched', 'active', 'sunset'
    launched_at     TIMESTAMPTZ,
    revenue_model   TEXT,          -- 'subscription', 'usage', 'one-time', 'ads', 'affiliate'
    monthly_cost    NUMERIC(10,2), -- infrastructure cost
    metadata        JSONB,         -- flexible product-specific data
    created_at      TIMESTAMPTZ DEFAULT now()
);

-- The agents in the factory
CREATE TABLE agents (
    id              TEXT PRIMARY KEY,  -- 'kev', 'dash', 'mae', etc.
    role            TEXT NOT NULL,     -- 'builder', 'marketer', 'ops', 'analytics'
    default_model   TEXT,              -- 'claude-opus-4-6', 'gpt-4o-mini', etc.
    daily_budget    NUMERIC(10,2),     -- daily spend limit in USD
    status          TEXT DEFAULT 'active',
    metadata        JSONB,
    created_at      TIMESTAMPTZ DEFAULT now()
);

-- Revenue channels (Stripe, Gumroad, AdSense, etc.)
CREATE TABLE revenue_channels (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    product_id      UUID REFERENCES products(id),
    channel_type    TEXT NOT NULL,     -- 'stripe', 'gumroad', 'adsense', 'affiliate', 'api-marketplace'
    channel_config  JSONB,            -- API keys, webhook URLs (encrypted at rest)
    created_at      TIMESTAMPTZ DEFAULT now()
);
```

### 3.2 Time-Series Tables (Hypertables)

```sql
-- Every LLM API call
CREATE TABLE llm_calls (
    time            TIMESTAMPTZ NOT NULL,
    agent_id        TEXT NOT NULL,
    task_id         UUID,
    model           TEXT NOT NULL,
    provider        TEXT NOT NULL,        -- 'anthropic', 'openai', 'groq'
    input_tokens    INTEGER NOT NULL,
    output_tokens   INTEGER NOT NULL,
    cached_tokens   INTEGER DEFAULT 0,
    cost_usd        NUMERIC(10,6) NOT NULL,
    latency_ms      INTEGER NOT NULL,
    status          TEXT NOT NULL,        -- 'success', 'error', 'timeout', 'rate_limited'
    error_type      TEXT,
    metadata        JSONB
);
SELECT create_hypertable('llm_calls', 'time');

-- Revenue events (every payment, ad impression, affiliate click)
CREATE TABLE revenue_events (
    time            TIMESTAMPTZ NOT NULL,
    product_id      UUID NOT NULL,
    channel_id      UUID,
    event_type      TEXT NOT NULL,    -- 'payment', 'subscription_start', 'subscription_cancel',
                                     -- 'refund', 'ad_impression', 'ad_click', 'affiliate_conversion'
    amount_usd      NUMERIC(10,2),
    currency        TEXT DEFAULT 'USD',
    customer_id     TEXT,            -- anonymized or external ID
    metadata        JSONB            -- plan name, invoice ID, etc.
);
SELECT create_hypertable('revenue_events', 'time');

-- Agent task executions
CREATE TABLE task_events (
    time            TIMESTAMPTZ NOT NULL,
    task_id         UUID NOT NULL,
    agent_id        TEXT NOT NULL,
    product_id      UUID,
    task_type       TEXT NOT NULL,    -- 'build', 'deploy', 'fix-bug', 'write-content', 'seo-audit',
                                     -- 'customer-support', 'marketing', 'research', 'report'
    status          TEXT NOT NULL,    -- 'started', 'completed', 'failed', 'escalated'
    duration_ms     INTEGER,
    total_cost_usd  NUMERIC(10,4),   -- total LLM + tool cost for this task
    steps_count     INTEGER,
    quality_score   NUMERIC(3,2),    -- 0.00 - 1.00, from eval/review
    metadata        JSONB
);
SELECT create_hypertable('task_events', 'time');

-- Infrastructure costs (hosting, domains, APIs, services)
CREATE TABLE infra_costs (
    time            TIMESTAMPTZ NOT NULL,
    product_id      UUID,            -- NULL for shared infra
    cost_type       TEXT NOT NULL,    -- 'hosting', 'domain', 'database', 'cdn', 'email', 'monitoring'
    provider        TEXT NOT NULL,    -- 'vercel', 'railway', 'cloudflare', 'supabase'
    amount_usd      NUMERIC(10,4) NOT NULL,
    billing_period  TEXT,            -- 'monthly', 'usage'
    metadata        JSONB
);
SELECT create_hypertable('infra_costs', 'time');

-- Marketing & growth metrics
CREATE TABLE growth_events (
    time            TIMESTAMPTZ NOT NULL,
    product_id      UUID NOT NULL,
    channel         TEXT NOT NULL,    -- 'organic', 'product-hunt', 'reddit', 'twitter',
                                     -- 'cold-email', 'referral', 'direct'
    metric          TEXT NOT NULL,    -- 'pageview', 'signup', 'trial_start', 'conversion',
                                     -- 'churn', 'referral_sent', 'backlink_acquired'
    value           NUMERIC(10,2) DEFAULT 1,
    metadata        JSONB
);
SELECT create_hypertable('growth_events', 'time');
```

### 3.3 Continuous Aggregates

```sql
-- Hourly agent cost rollup
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

-- Daily product P&L
CREATE VIEW product_pnl_daily AS
SELECT
    r.bucket AS day,
    p.name AS product_name,
    p.type AS product_type,
    COALESCE(r.total_revenue, 0) AS revenue,
    COALESCE(t.total_agent_cost, 0) AS agent_cost,
    COALESCE(i.total_infra_cost, 0) AS infra_cost,
    COALESCE(r.total_revenue, 0)
        - COALESCE(t.total_agent_cost, 0)
        - COALESCE(i.total_infra_cost, 0) AS net_profit
FROM revenue_daily r
FULL JOIN (
    SELECT time_bucket('1 day', time) AS bucket, product_id, SUM(total_cost_usd) AS total_agent_cost
    FROM task_events WHERE status = 'completed' GROUP BY 1, 2
) t ON r.bucket = t.bucket AND r.product_id = t.product_id
FULL JOIN (
    SELECT time_bucket('1 day', time) AS bucket, product_id, SUM(amount_usd) AS total_infra_cost
    FROM infra_costs GROUP BY 1, 2
) i ON r.bucket = i.bucket AND r.product_id = i.product_id
JOIN products p ON p.id = COALESCE(r.product_id, t.product_id, i.product_id);
```

---

## 4. Ingestion Pipeline

### Architecture

```
┌─────────────────────────────────────────────────────────┐
│                     Data Sources                         │
├──────────┬──────────┬──────────┬──────────┬─────────────┤
│ LLM Proxy│ Stripe   │ Agent    │ Infra    │ Analytics   │
│ (LiteLLM)│ Webhooks │ Events   │ Billing  │ (PostHog)   │
└────┬─────┴────┬─────┴────┬─────┴────┬─────┴──────┬──────┘
     │          │          │          │            │
     ▼          ▼          ▼          ▼            ▼
┌─────────────────────────────────────────────────────────┐
│              Ingestion Layer (Node.js/Bun)               │
│                                                          │
│  ┌──────────┐  ┌───────────┐  ┌───────────────────────┐ │
│  │ Webhook  │  │ Polling   │  │ Event Bus (Redis      │ │
│  │ Receiver │  │ Collectors│  │ Streams / NATS)       │ │
│  └────┬─────┘  └─────┬─────┘  └──────────┬────────────┘ │
│       │              │                    │              │
│       └──────────────┼────────────────────┘              │
│                      ▼                                   │
│            ┌─────────────────┐                           │
│            │   Normalizer    │  Validate, enrich,        │
│            │   & Router      │  tag with product/agent   │
│            └────────┬────────┘                           │
│                     │                                    │
│         ┌───────────┼───────────┐                        │
│         ▼           ▼           ▼                        │
│    TimescaleDB    Redis       S3 Archive                 │
│    (analytics)   (real-time)  (raw events)               │
└─────────────────────────────────────────────────────────┘
```

### Ingestion Sources

| Source | Method | Frequency | Data |
|--------|--------|-----------|------|
| **LLM Proxy (LiteLLM)** | Callback hook / log tail | Real-time | Every LLM call: tokens, cost, latency, model |
| **Stripe** | Webhooks | Real-time | Payments, subscriptions, refunds, disputes |
| **Gumroad / Lemon Squeezy** | Webhooks | Real-time | Digital product sales |
| **Google AdSense** | API poll | Hourly | Ad revenue, impressions, CPM |
| **Affiliate networks** | API poll | Hourly | Clicks, conversions, commissions |
| **Agent event bus** | Redis Streams subscription | Real-time | Task start/complete/fail, agent state changes |
| **Infrastructure billing** | API poll / webhook | Daily | Vercel, Railway, Cloudflare, Supabase costs |
| **PostHog / Plausible** | API poll | Hourly | Pageviews, signups, feature usage per product |
| **Google Search Console** | API poll | Daily | Search impressions, clicks, ranking positions |
| **GitHub** | Webhooks | Real-time | Commits, PRs, deployments, CI results |

### Real-Time Accumulators (Redis)

For metrics that need sub-second reads (kill switches, budget enforcement, live dashboards):

```
dash:spend:daily:{agent_id}     → Running total USD today
dash:spend:hourly:{agent_id}    → Rolling 1-hour spend
dash:revenue:daily:{product_id} → Revenue today
dash:revenue:mtd                → Month-to-date total revenue
dash:errors:5min:{agent_id}     → Error count in sliding 5-min window
dash:velocity:{agent_id}        → $/min rolling average
```

These are write-through: every event updates both Redis (for speed) and TimescaleDB (for history).

---

## 5. KPIs & Metrics Framework

### 5.1 Factory-Level KPIs (The Numbers That Matter)

| KPI | Formula | Target | Cadence |
|-----|---------|--------|---------|
| **Total Revenue** | Sum of all revenue events | Growing MoM | Daily |
| **Total Cost** | LLM + infra + services | < 30% of revenue | Daily |
| **Net Profit** | Revenue - Cost | Positive, growing | Daily |
| **Gross Margin** | (Revenue - Cost) / Revenue | > 70% | Weekly |
| **Revenue per Product** | Revenue / active products | > $500/mo avg | Weekly |
| **Cost per Agent** | Total LLM cost / agent | Optimize outliers | Daily |
| **ROI per Product** | (Revenue - Cost) / Cost × 100 | > 200% | Weekly |
| **Products Shipped** | Count of products launched | 2-4/month | Monthly |
| **Time to Revenue** | Days from idea → first dollar | < 14 days | Per product |
| **Portfolio Hit Rate** | Products profitable / products launched | > 30% | Quarterly |

### 5.2 Agent Performance KPIs

| KPI | Formula | Target |
|-----|---------|--------|
| **Task Completion Rate** | Completed / (Completed + Failed + Escalated) | > 90% |
| **Avg Cost per Task** | Total LLM cost / completed tasks | Decreasing trend |
| **Avg Steps per Task** | Mean steps for completed tasks | Lower = better |
| **Error Rate** | Failed calls / total calls | < 5% |
| **Escalation Rate** | Human-escalated / total tasks | < 10% |
| **Quality Score** | Mean quality_score from evals | > 0.8 |
| **Cache Hit Rate** | Cached tokens / total input tokens | > 30% |
| **Model Efficiency** | % tasks using cheapest viable model | Increasing |

### 5.3 Product Health KPIs

| KPI | Formula | Target |
|-----|---------|--------|
| **MRR** | Monthly recurring revenue | Growing |
| **Churn Rate** | Cancellations / total subscribers | < 5%/mo |
| **CAC** | Marketing cost / new customers | < 3 months LTV |
| **LTV** | Avg revenue per customer × avg lifespan | > 3× CAC |
| **Conversion Rate** | Signups → paid | > 5% |
| **DAU/MAU** | Daily active / monthly active | > 20% |
| **NPS / Satisfaction** | Survey or review scores | > 40 NPS |
| **Uptime** | Available minutes / total minutes | > 99.5% |

### 5.4 Growth KPIs

| KPI | Formula | Target |
|-----|---------|--------|
| **Organic Traffic** | Pageviews from search | Growing 10%+ MoM |
| **Keyword Rankings** | Keywords in top 10 | Increasing |
| **Backlinks Acquired** | New referring domains | 10+/month |
| **Email List Size** | Total subscribers | Growing |
| **Referral Rate** | Users who refer / total users | > 5% |

---

## 6. Dashboard Architecture

### Autonomous Dashboard Building

Dash doesn't just read dashboards — it **creates and maintains them**. The approach:

1. **Grafana as the rendering engine** — Dash programmatically creates/updates dashboards via the Grafana HTTP API
2. **Dashboard-as-code** — Dashboard JSON definitions stored in git, versioned, reviewable
3. **Template library** — Pre-built dashboard templates for common patterns (product P&L, agent performance, growth funnel)
4. **Auto-generation** — When a new product launches, Dash automatically creates its dashboard from template
5. **Anomaly annotations** — Dash adds annotations to graphs when it detects something worth noting

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
├── Product Dashboards (one per product, auto-generated)
│   ├── Revenue & growth metrics
│   ├── User funnel (visit → signup → paid → retain)
│   ├── Feature usage
│   └── Support tickets & satisfaction
├── Growth Dashboard
│   ├── SEO performance (rankings, traffic, backlinks)
│   ├── Marketing channel attribution
│   ├── Conversion funnels
│   └── Content performance
└── Operational Health
    ├── Uptime & latency per product
    ├── Deployment frequency
    ├── Error budgets
    └── Infrastructure costs
```

### Grafana Configuration

```yaml
# Provisioned via docker-compose or ansible
grafana:
  datasources:
    - name: TimescaleDB
      type: postgres
      url: timescaledb:5432
      database: factory_analytics
    - name: Redis
      type: redis-datasource
      url: redis:6379

  dashboards:
    provider:
      folder: /var/lib/grafana/dashboards
      # Dash writes dashboard JSON here via API or file sync
```

Dash creates dashboards by generating Grafana JSON models:

```python
# Pseudocode — Dash creating a product dashboard
async def create_product_dashboard(product):
    template = load_template("product-pnl")
    dashboard = template.render(
        product_id=product.id,
        product_name=product.name,
        revenue_model=product.revenue_model,
    )
    await grafana_api.create_or_update_dashboard(dashboard)
    log(f"Created dashboard for {product.name}")
```

---

## 7. Automated Reporting

### Report Types

| Report | Cadence | Audience | Content |
|--------|---------|----------|---------|
| **Daily Digest** | Daily 08:00 | Human (Adam) | Yesterday's revenue, costs, alerts, notable events |
| **Weekly P&L** | Monday 09:00 | Human | Per-product P&L, trends, agent efficiency, recommendations |
| **Monthly Business Review** | 1st of month | Human | Full financials, product portfolio review, growth metrics, strategic recommendations |
| **Agent Performance Report** | Weekly | Factory agents | Per-agent metrics, optimization suggestions |
| **Product Health Check** | Weekly per product | Relevant agents | Traffic, revenue, churn, action items |
| **Anomaly Alert** | Real-time | Human + agents | Cost spikes, revenue drops, error surges |

### Report Generation Pipeline

```
1. Scheduled trigger (cron) or event trigger (anomaly detected)
2. Dash queries TimescaleDB for relevant metrics
3. Dash compares against targets, previous periods, and trends
4. Dash generates narrative analysis (using LLM on the numbers)
5. Dash formats report (Markdown for chat, PDF for archive)
6. Dash delivers via appropriate channel (WhatsApp, email, file)
7. Report archived to S3
```

### Example: Daily Digest Template

```markdown
# 📊 Factory Daily Digest — {date}

## Revenue
- **Yesterday:** ${revenue_yesterday} ({revenue_delta}% vs prior day)
- **MTD:** ${revenue_mtd} (tracking {revenue_projection} for the month)
- **Top product:** {top_product} at ${top_product_revenue}

## Costs
- **LLM spend:** ${llm_cost} across {llm_call_count} calls
- **Infra:** ${infra_cost}
- **Total:** ${total_cost} | **Net:** ${net_profit}
- **Burn rate:** ${burn_rate}/hour (avg)

## Agent Performance
- Most active: {busiest_agent} ({task_count} tasks, ${agent_cost})
- Best efficiency: {most_efficient_agent} (${cost_per_task}/task)
- Errors: {error_count} ({error_rate}% rate)

## Alerts
{alerts_list}

## Notable
{notable_events}

## Dash's Take
{ai_analysis}  <!-- Dash's interpretation of the numbers -->
```

### Anomaly Detection & Alerting

Dash monitors continuously and alerts on:

| Condition | Threshold | Action |
|-----------|-----------|--------|
| Cost velocity spike | > 3× rolling average $/min | Alert + auto-pause agent |
| Revenue drop | > 30% day-over-day | Alert human |
| Error rate surge | > 10% in 5-min window | Alert + investigate |
| Product downtime | > 5 min unresponsive | Alert + trigger ops agent |
| Churn spike | > 2× average daily cancellations | Alert + flag for analysis |
| Budget threshold | Agent at 80% daily budget | Warn agent, alert at 100% |
| Zero revenue | Product earning $0 for 48h+ (if normally earning) | Flag for review |

---

## 8. ROI Calculation Engine

### Per-Product ROI

```sql
-- Product ROI for any time period
SELECT
    p.name,
    p.type,
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
LEFT JOIN revenue_events r ON r.product_id = p.id AND r.time >= {start} AND r.time < {end}
LEFT JOIN task_events t ON t.product_id = p.id AND t.time >= {start} AND t.time < {end}
LEFT JOIN infra_costs i ON i.product_id = p.id AND i.time >= {start} AND i.time < {end}
GROUP BY p.id, p.name, p.type
ORDER BY net_profit DESC;
```

### Per-Agent ROI

Harder — agents contribute to multiple products. Attribution model:

```
Agent ROI = (Revenue of products agent worked on × attribution weight) - Agent's total cost
                                        Agent's total cost

Attribution weight = Agent's task hours on product / Total task hours on product
```

### Build vs Buy Decision Support

For each product, Dash tracks:
- **Time to build** (total agent hours from idea → launch)
- **Build cost** (LLM + human time during build phase)
- **Ongoing maintenance cost** (agent time post-launch)
- **Revenue trajectory** (daily revenue curve)
- **Payback period** (days until cumulative revenue > cumulative cost)
- **Break-even projection** (when will this product be profitable?)

---

## 9. Implementation Plan

### Phase 1: Foundation (Week 1-2)

- [ ] Deploy TimescaleDB (Docker on factory server)
- [ ] Create schema (tables, hypertables, continuous aggregates)
- [ ] Set up Redis for real-time accumulators
- [ ] Build ingestion service: LiteLLM callback → TimescaleDB
- [ ] Wire Stripe webhooks → revenue_events
- [ ] Deploy Grafana, connect to TimescaleDB
- [ ] Create Factory Overview dashboard (manual first pass)

### Phase 2: Intelligence (Week 3-4)

- [ ] Build Dash's query interface (SQL tool for reading analytics)
- [ ] Implement daily digest report generation
- [ ] Implement anomaly detection (cost velocity, error rate)
- [ ] Set up alerting pipeline (Dash → WhatsApp/email)
- [ ] Create dashboard templates for auto-generation
- [ ] Wire agent event bus → task_events ingestion

### Phase 3: Full Automation (Week 5-8)

- [ ] Auto-create product dashboards on product launch
- [ ] Weekly P&L report generation
- [ ] Monthly business review generation
- [ ] ROI calculation engine
- [ ] Growth metrics ingestion (Search Console, PostHog)
- [ ] Infra cost polling (Vercel, Railway, Cloudflare APIs)
- [ ] Historical backfill from existing data sources

### Phase 4: Advanced (Month 3+)

- [ ] Predictive analytics (revenue forecasting, cost projection)
- [ ] Automated recommendations ("sunset product X, double down on Y")
- [ ] A/B test analysis automation
- [ ] Competitor monitoring dashboards
- [ ] Self-improving queries (Dash learns which metrics matter most)

---

## 10. Technical Specifications

### Infrastructure Requirements

| Component | Spec | Notes |
|-----------|------|-------|
| TimescaleDB | 2 CPU, 4GB RAM, 100GB SSD | Grows with data; compression helps |
| Redis | 1 CPU, 1GB RAM | Lightweight, just accumulators |
| Grafana | 1 CPU, 1GB RAM | Low resource, serves dashboards |
| Ingestion service | Node.js/Bun process | Runs alongside factory services |
| S3/MinIO | 10GB initially | Report archive, raw event backup |

### Data Retention Policy

| Data | Raw Retention | Aggregate Retention |
|------|---------------|---------------------|
| LLM calls | 90 days | Hourly: 1 year, Daily: forever |
| Revenue events | Forever | Daily: forever |
| Task events | 180 days | Daily: forever |
| Infra costs | Forever | Monthly: forever |
| Growth events | 90 days | Daily: forever |

### Security

- TimescaleDB encrypted at rest, TLS in transit
- Revenue data access restricted to Dash + human admin
- API keys for external services stored in encrypted config (not in DB)
- Grafana behind auth (admin only initially)
- Redis not exposed externally

---

## 11. Dash's Analytical Capabilities

Dash isn't just a pipeline — it's the analyst. Key capabilities:

1. **Natural language queries** — "What was our best product last month?" → Dash writes SQL, runs it, interprets results
2. **Trend detection** — "Revenue is up 15% but costs are up 40% — margin compression on product X"
3. **Anomaly explanation** — "Cost spike at 14:32 was agent Kev stuck in a retry loop on the SEO crawler"
4. **Recommendations** — "Product Y has negative ROI for 3 weeks. Recommend sunset or pivot."
5. **Forecasting** — "At current growth rate, product X will hit $1K MRR by March 15"
6. **Cross-metric correlation** — "Traffic from Reddit post correlated with 3× signup rate for 48h"

### Dash's Data Access Pattern

```
Dash receives question/trigger
    → Determines what data is needed
    → Writes and executes SQL against TimescaleDB
    → Checks Redis for real-time values if needed
    → Interprets results using LLM reasoning
    → Generates narrative + visualization (Grafana link or inline chart)
    → Delivers to appropriate channel
```

---

*Architecture designed: 2026-02-08*
*Companion research: 14-monitoring-observability, 11-revenue-generation, 07-ai-factory-patterns*
