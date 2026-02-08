# 08 — Monitoring, Observability & Cost Control

> Architecture for keeping the factory visible, accountable, and financially safe.

---

## Design Principles

1. **Every LLM call flows through a single control plane** — no direct provider calls, ever
2. **Cost is a first-class metric** — tracked per-token, per-task, per-agent, per-user, per-hour
3. **Hard limits before smart limits** — provider caps and per-key budgets exist from day zero
4. **Kill switches are infrastructure, not afterthoughts** — sub-second response time
5. **Observe cost velocity, not just totals** — $100/day is fine; $100 in 5 minutes is a fire

---

## Architecture Overview

```
┌──────────────────────────────────────────────────────────────────┐
│                         AGENT LAYER                              │
│   Agent 1 ─┐                                                     │
│   Agent 2 ─┤                                                     │
│   Agent N ─┘                                                     │
│       │                                                          │
│       ▼                                                          │
│  ┌─────────────────────────────────────────────────────────┐     │
│  │              LLM GATEWAY (LiteLLM)                      │     │
│  │  ┌──────────┬──────────┬───────────┬──────────────┐     │     │
│  │  │ Budget   │ Kill     │ Rate      │ Model        │     │     │
│  │  │ Enforcer │ Switches │ Limiter   │ Router       │     │     │
│  │  └──────────┴──────────┴───────────┴──────────────┘     │     │
│  │  ┌──────────┬──────────┬───────────┐                    │     │
│  │  │ Prompt   │ Semantic │ Provider  │                    │     │
│  │  │ Cache    │ Cache    │ Fallback  │                    │     │
│  │  └──────────┴──────────┴───────────┘                    │     │
│  └───────────────────┬─────────────────────────────────────┘     │
│                      │                                           │
│         ┌────────────┼────────────┐                              │
│         ▼            ▼            ▼                              │
│    Anthropic      OpenAI      Google/Groq/Together               │
└──────────────────────────────────────────────────────────────────┘
          │
          │ Every call emits structured events
          ▼
┌──────────────────────────────────────────────────────────────────┐
│                     OBSERVABILITY PLANE                           │
│                                                                   │
│  ┌────────────┐  ┌──────────────┐  ┌──────────────────────┐     │
│  │  Langfuse   │  │ Prometheus   │  │  Redis                │     │
│  │  (traces,   │  │ (time-series │  │  (real-time budgets,  │     │
│  │   evals,    │  │  metrics,    │  │   kill switch flags,  │     │
│  │   costs)    │  │  alerting)   │  │   cost accumulators)  │     │
│  └──────┬──────┘  └──────┬───────┘  └──────────┬───────────┘     │
│         │                │                      │                │
│         └────────────────┼──────────────────────┘                │
│                          ▼                                       │
│                 ┌────────────────┐                                │
│                 │    Grafana     │                                │
│                 │  (dashboards,  │                                │
│                 │   alerts →     │                                │
│                 │   PagerDuty/   │                                │
│                 │   Slack/ntfy)  │                                │
│                 └────────────────┘                                │
└──────────────────────────────────────────────────────────────────┘
```

---

## Stack Selection

### Decision: LiteLLM + Langfuse + Prometheus/Grafana + Custom Budget Layer

| Component | Tool | Why |
|-----------|------|-----|
| **LLM Gateway** | **LiteLLM** (self-hosted) | Open-source, 100+ providers, per-key budgets, model routing, OpenAI-compatible API. Every agent talks to LiteLLM, never directly to providers. |
| **Tracing & Cost Analytics** | **Langfuse** (self-hosted) | Open-source, OpenTelemetry-based, session/user/trace-level cost tracking, eval framework, prompt management. Self-hosted = no data leaves our infra. |
| **Real-time Metrics** | **Prometheus + Grafana** | Industry standard. Prometheus scrapes LiteLLM + custom exporters. Grafana for dashboards and alert rules. |
| **Real-time State** | **Redis** | Budget accumulators, kill switch flags, cost velocity windows. Sub-millisecond reads on every request. |
| **Alerting** | **Grafana Alerting → Slack/ntfy/PagerDuty** | Unified alert routing. Escalation chains for severity levels. |

### Why Not Helicone?

Helicone is excellent but overlaps with LiteLLM (both are gateways). LiteLLM gives us more control over routing logic and budget enforcement, and pairs natively with Langfuse for tracing. We'd be running two proxies for no gain.

### Why Not LangSmith?

Vendor lock-in, not self-hostable, strongest in LangChain ecosystem. Langfuse gives us equivalent tracing with full data sovereignty.

---

## Component Design

### 1. LLM Gateway (LiteLLM)

Single entry point for all LLM traffic. Runs as a service with a Postgres-backed config.

**Responsibilities:**
- **Model routing** — task complexity → model tier (see §2 of research 02)
- **Per-key budgets** — each agent/team gets an API key with daily/monthly caps
- **Provider fallback** — if Anthropic is down, route to Google; if rate-limited, queue or degrade
- **Prompt caching** — leverage provider-native caching (Anthropic 90% savings, OpenAI 90%, Google 90%)
- **Request/response logging** → forwarded to Langfuse via callback
- **Rate limiting** — per-agent, per-model, global
- **max_tokens enforcement** — never allow unbounded generation

**Configuration (key fields):**
```yaml
model_list:
  - model_name: "quick"          # Simple tasks
    litellm_params:
      model: groq/llama-3.1-8b-instant
      max_budget: 5.0            # $/day
  - model_name: "balanced"       # Medium tasks  
    litellm_params:
      model: google/gemini-2.5-flash
      max_budget: 20.0
  - model_name: "frontier"       # Complex tasks
    litellm_params:
      model: anthropic/claude-sonnet-4-5-20250514
      max_budget: 50.0
  - model_name: "max"            # When nothing else will do
    litellm_params:
      model: anthropic/claude-opus-4-6-20250801
      max_budget: 30.0

litellm_settings:
  max_budget: 200.0              # Global daily cap
  budget_duration: "1d"
  callbacks: ["langfuse"]
  cache: true
  cache_params:
    type: "redis"
```

### 2. Budget Enforcement Layer

Sits inside the gateway's request path. Redis-backed for speed.

```
Request arrives
  → Check global kill switch         (Redis GET kill:global)
  → Check agent kill switch          (Redis GET kill:agent:{id})  
  → Check provider kill switch       (Redis GET kill:provider:{name})
  → Load agent's budget              (Redis GET budget:{agent_id})
  → Check cost velocity              (Redis sliding window)
  → If all pass → forward to LLM
  → Log cost on response             (Redis INCRBYFLOAT spent:{agent_id}:{date})
```

**Budget tiers:**
| Level | Limit | Reset | Enforcement |
|-------|-------|-------|-------------|
| Per-request | max_tokens set on every call | N/A | Hard — request rejected if missing |
| Per-agent per-day | $1–$100 depending on agent | Daily midnight UTC | Hard — agent paused until reset |
| Per-agent per-task | Configurable per workflow | Per task execution | Hard — task fails gracefully |
| Per-model per-day | Prevents overuse of expensive models | Daily | Soft — degrades to cheaper model |
| Global daily | $200 (adjustable) | Daily | Hard — nuclear option |
| Global monthly | $3,000 (adjustable) | Monthly | Hard — everything stops |

**Cost velocity detection:**
```python
# Sliding window: if agent spends > $X in Y minutes, trigger alert/pause
VELOCITY_WINDOW = 300  # 5 minutes
VELOCITY_THRESHOLD = {
    "warning": 5.0,    # $5 in 5 min → alert
    "critical": 15.0,  # $15 in 5 min → pause agent
    "emergency": 50.0, # $50 in 5 min → global pause + page on-call
}
```

### 3. Kill Switch System

Multiple granularity levels, all enforced at the gateway:

| Switch | Scope | Trigger | Recovery |
|--------|-------|---------|----------|
| `kill:global` | All LLM calls | Manual or emergency auto-trigger | Manual only |
| `kill:agent:{id}` | Single agent | Velocity breach, error cascade, manual | Manual or auto after cooldown |
| `kill:provider:{name}` | One provider | Provider outage detected, manual | Auto when health check passes |
| `kill:model:{name}` | One model | Model-specific errors spike | Auto after cooldown |
| `throttle:agent:{id}` | Reduced rate | Approaching budget, velocity warning | Auto when velocity normalizes |

**Implementation:** Redis keys with optional TTL for auto-recovery.

```python
# In gateway middleware (pseudocode)
async def pre_request(request):
    checks = [
        redis.get("kill:global"),
        redis.get(f"kill:agent:{request.agent_id}"),
        redis.get(f"kill:provider:{request.provider}"),
    ]
    if any(await asyncio.gather(*checks)):
        raise KillSwitchActive(...)
    
    velocity = await get_sliding_window_cost(request.agent_id, window=300)
    if velocity > VELOCITY_THRESHOLD["critical"]:
        await redis.set(f"kill:agent:{request.agent_id}", "velocity_breach", ex=600)
        await alert("critical", f"Agent {request.agent_id} killed: ${velocity:.2f} in 5min")
        raise KillSwitchActive(...)
```

**Manual kill via CLI:**
```bash
# Kill a specific agent immediately
redis-cli SET kill:agent:research-bot "manual:adam:runaway_loop"

# Kill all traffic (emergency)
redis-cli SET kill:global "manual:adam:investigating_cost_spike"

# Resume
redis-cli DEL kill:agent:research-bot
```

### 4. Tracing (Langfuse)

Every LLM call, tool call, and agent step is a **span** within a **trace**.

**Trace hierarchy:**
```
Trace: "research-task-4821"
  ├── Generation: plan (claude-sonnet-4-5, 1,200 tokens, $0.021)
  ├── Span: web-search
  │   ├── Generation: query-formulation (gemini-2.5-flash, 200 tokens, $0.001)
  │   └── Span: fetch-results (tool call, 340ms)
  ├── Generation: synthesize (claude-sonnet-4-5, 3,400 tokens, $0.058)
  └── Score: quality=0.87, cost=$0.08, steps=4
```

**What Langfuse tracks:**
- Token counts (input/output/total) per generation
- Cost per generation (auto-calculated from model pricing)
- Latency per span
- Model used, temperature, max_tokens
- User/session/agent attribution
- Custom scores (quality evals, human ratings)
- Prompt versions (for A/B testing)

**Langfuse self-hosted deployment:**
- Postgres (trace storage) + ClickHouse (analytics) + Langfuse server
- Deployed alongside the factory infrastructure
- ~2GB RAM, minimal CPU for our scale

### 5. Metrics & Dashboards (Prometheus + Grafana)

**Prometheus metrics exported by the gateway:**

```
# Token usage
llm_tokens_total{agent, model, direction="input|output"} counter
llm_cost_dollars_total{agent, model} counter

# Request metrics
llm_requests_total{agent, model, status="success|error|killed"} counter
llm_request_duration_seconds{agent, model} histogram

# Budget
llm_budget_remaining_dollars{agent} gauge
llm_budget_utilization_ratio{agent} gauge

# Velocity
llm_cost_velocity_dollars_per_minute{agent} gauge

# Kill switches
llm_kill_switch_active{scope, target} gauge

# Cache
llm_cache_hits_total{agent, model} counter
llm_cache_hit_ratio{agent} gauge

# Agent performance
agent_task_completion_total{agent, status="completed|failed|killed"} counter
agent_steps_per_task{agent} histogram
agent_human_escalation_total{agent} counter
```

**Grafana Dashboard Panels:**

```
┌─────────────────────────────────────────────────────────────┐
│  🏭 AI FACTORY — COST & HEALTH                              │
├────────────────┬────────────────┬────────────────────────────┤
│  TODAY: $47.23 │  MTD: $892     │  Budget used: 44% ████░░  │
├────────────────┴────────────────┴────────────────────────────┤
│  Cost velocity ($/hr, 24h)         [time-series graph]       │
│  Active agents: 12/15              [status indicators]       │
│  Error rate (1h): 1.8%             [gauge + trend]           │
│  Cache hit rate: 38%               [gauge + trend]           │
│  P95 latency: 2.1s                 [gauge + trend]           │
├──────────────────────────────────────────────────────────────┤
│  Cost by Agent (today)              Cost by Model (today)    │
│  research-agent  $12.40 ████████   opus-4.6    $18.20       │
│  code-review     $8.20  █████      sonnet-4.5  $14.10       │
│  email-bot       $6.10  ████       flash-2.5   $8.40        │
│  task-planner    $4.50  ███        llama-8b    $2.30        │
├──────────────────────────────────────────────────────────────┤
│  Provider Health          │  Kill Switches                   │
│  Anthropic  ✅ 52ms       │  Global:     🟢 OFF              │
│  OpenAI     ✅ 84ms       │  Agents:     🟢 All clear        │
│  Google     ✅ 61ms       │  Providers:  🟢 All clear        │
│  Groq       ✅ 12ms       │  Throttled:  ⚠️ research-agent   │
└──────────────────────────────────────────────────────────────┘
```

### 6. Alerting Rules

| Alert | Condition | Severity | Action |
|-------|-----------|----------|--------|
| **Cost velocity spike** | Agent >$5 in 5min | Warning | Slack notification |
| **Cost velocity critical** | Agent >$15 in 5min | Critical | Auto-kill agent + page on-call |
| **Daily budget 80%** | Agent at 80% of daily limit | Warning | Slack notification |
| **Daily budget exceeded** | Agent at 100% | Critical | Auto-kill agent |
| **Global budget 80%** | Factory at 80% of daily limit | Warning | Slack + email |
| **Error rate spike** | >10% errors in 5min window | Warning | Slack notification |
| **Error cascade** | >3 agents erroring simultaneously | Critical | Page on-call |
| **Provider down** | >50% errors to one provider in 2min | Warning | Auto-failover + Slack |
| **Loop detected** | Same tool called >5x in one trace | Warning | Kill trace, alert |
| **Context explosion** | Input tokens >100K and growing per step | Warning | Kill trace, alert |
| **Model degradation** | P95 latency >10s for 5min | Warning | Consider provider switch |

### 7. Anomaly Detection

Beyond static thresholds, detect anomalies in:

**Cost patterns:**
- Rolling 7-day average cost per agent. Alert if today exceeds 2σ above mean.
- Hour-over-hour cost comparison. Alert if current hour is 3x previous hour.

**Behavioral patterns:**
- **Token growth per step** — if each successive step uses more tokens than the last (context stuffing loop), kill after 3 consecutive increases >20%.
- **Repetitive tool calls** — hash the (tool_name, args_subset) tuple. If the same hash appears 3+ times in a trace, flag as stuck loop.
- **Output similarity** — if consecutive outputs have >90% cosine similarity, the agent is repeating itself. Kill.

**Implementation:** Custom Python service consuming Langfuse traces via webhook, running simple statistical checks. Not ML — just sensible heuristics. Graduate to ML anomaly detection only if needed.

---

## Preventing Runaway Costs — Defense in Depth

```
Layer 0: PROVIDER CAPS
  └─ Hard spend limits set in Anthropic/OpenAI/Google dashboards
     Ultimate safety net. Set slightly above expected monthly max.

Layer 1: GATEWAY HARD LIMITS  
  └─ LiteLLM per-key daily budgets
  └─ max_tokens on every request (never unbounded)
  └─ Max steps per agent run (e.g., 25 tool calls)
  └─ Request timeout (60s default)

Layer 2: SMART CONTROLS (Redis-backed)
  └─ Cost velocity monitoring (sliding 5-min window)
  └─ Loop detection (repetitive tool calls)
  └─ Context growth detection
  └─ Auto-kill on breach

Layer 3: OBSERVABILITY
  └─ Langfuse traces — full audit trail
  └─ Grafana dashboards — human-readable cost visibility
  └─ Prometheus alerts — automated notification

Layer 4: MODEL ECONOMICS
  └─ Task-complexity routing (cheap models for cheap tasks)
  └─ Prompt caching (90% savings on repeated context)
  └─ Batch API for async work (50% savings)
  └─ Semantic caching for similar queries (20-40% savings)

Layer 5: ORGANIZATIONAL
  └─ Weekly cost review cadence
  └─ Shadow mode for new agents (run without side effects, measure cost)
  └─ Cost per task tracked against value generated
  └─ Agent-level P&L (does this agent generate more value than it costs?)
```

---

## Model Routing Strategy

Integrated into the gateway. Agents request a capability tier, not a specific model:

| Tier | Use Case | Primary Model | Fallback | ~Cost/MTok Out |
|------|----------|---------------|----------|----------------|
| `quick` | Classification, extraction, yes/no | Groq/Llama-3.1-8B | Gemini 2.5 Flash Lite | $0.08–$0.40 |
| `balanced` | General tasks, summarization, chat | Gemini 2.5 Flash | GPT-5 mini | $2.00–$2.50 |
| `frontier` | Code gen, complex reasoning, writing | Claude Sonnet 4.5 | Gemini 2.5 Pro | $10.00–$15.00 |
| `max` | Hardest problems, architecture, novel code | Claude Opus 4.6 | GPT-5.2 | $25.00 |

**Auto-downgrade:** When an agent hits 80% of its daily budget, `frontier` requests route to `balanced`, `max` routes to `frontier`. Graceful degradation instead of hard stops where possible.

---

## Agent-Level Cost Awareness

Agents are designed to be cost-aware:

```python
class CostAwareAgent:
    def __init__(self, daily_budget: float):
        self.daily_budget = daily_budget
    
    async def select_tier(self, task_complexity: str) -> str:
        spent = await self.get_spent_today()
        remaining_ratio = 1 - (spent / self.daily_budget)
        
        if remaining_ratio < 0.2:
            return "quick"        # Conserve budget
        elif remaining_ratio < 0.5:
            return min(task_complexity, "balanced")  # Cap at balanced
        else:
            return task_complexity  # Use requested tier
```

---

## Data Retention & Storage

| Data | Store | Retention | Purpose |
|------|-------|-----------|---------|
| Traces (full) | Langfuse (Postgres) | 90 days | Debugging, eval |
| Traces (summary) | Langfuse (ClickHouse) | 1 year | Cost analytics |
| Metrics | Prometheus | 30 days (full), 1yr (downsampled) | Dashboards, alerts |
| Budget state | Redis | Real-time, daily reset | Enforcement |
| Cost reports | Postgres | Indefinite | Financial tracking |
| Kill switch events | Postgres + Prometheus | 1 year | Audit trail |

---

## Deployment

All components self-hosted on factory infrastructure:

| Service | Resources | Notes |
|---------|-----------|-------|
| LiteLLM | 1 CPU, 1GB RAM | Stateless, scale horizontally |
| Langfuse | 2 CPU, 4GB RAM | Postgres + ClickHouse sidecars |
| Prometheus | 1 CPU, 2GB RAM | Local SSD for TSDB |
| Grafana | 0.5 CPU, 512MB RAM | Dashboards + alerting |
| Redis | 0.5 CPU, 512MB RAM | Budget state, kill switches, cache |
| Anomaly detector | 0.5 CPU, 512MB RAM | Custom Python service |

**Total overhead:** ~6 CPU, 9GB RAM. Trivial compared to the value of not having a $5,000 surprise bill.

---

## Implementation Phases

### Phase 1: Foundation (Week 1)
- Deploy LiteLLM as the gateway — all agents route through it
- Set per-key daily budgets for every agent
- Set provider-level hard caps in dashboards
- Set max_tokens on every request
- Deploy Redis for budget tracking + kill switches
- Manual kill switch commands documented and tested

### Phase 2: Visibility (Week 2)
- Deploy Langfuse (self-hosted) — enable LiteLLM → Langfuse callback
- Deploy Prometheus — scrape LiteLLM metrics
- Deploy Grafana — build cost dashboard
- Set up basic alerts (daily budget 80%, error rate >10%)

### Phase 3: Smart Controls (Week 3)
- Implement cost velocity monitoring (sliding window in Redis)
- Implement loop detection (repetitive tool call hashing)
- Auto-kill on velocity breach
- Context growth detection
- Model tier routing (quick/balanced/frontier/max)

### Phase 4: Optimization (Week 4+)
- Prompt caching tuning (measure cache hit rates, optimize system prompts for cacheability)
- Semantic caching for common query patterns
- Batch API integration for async workflows
- Auto-downgrade on budget pressure
- Weekly automated cost reports
- Shadow mode for new agent deployments
- Anomaly detection service

---

## Key Decisions & Rationale

| Decision | Rationale |
|----------|-----------|
| LiteLLM over Helicone | More control over routing logic; native budget enforcement; pairs with Langfuse; avoids double-proxy |
| Langfuse over LangSmith | Self-hostable, open-source, OTel-based, no vendor lock-in, data sovereignty |
| Redis for real-time state | Sub-ms latency for kill switch checks on every request; simple key-value fits the model perfectly |
| Prometheus over custom TSDB | Industry standard, Grafana integration, massive ecosystem of exporters and dashboards |
| Heuristic anomaly detection over ML | Simpler, more predictable, easier to debug. ML anomaly detection adds complexity without clear ROI at our scale |
| Self-hosted everything | Data sovereignty, cost control, no per-trace SaaS billing that scales with usage |

---

## Summary

The monitoring stack serves one purpose: **maximize value per dollar while making catastrophic spend impossible.**

The gateway (LiteLLM) is the single enforcement point. Nothing reaches a provider without passing through budget checks, kill switch checks, and rate limits. Langfuse provides the audit trail and cost analytics. Prometheus/Grafana give real-time visibility and automated alerting. Redis holds the real-time state that makes sub-millisecond enforcement possible.

Five layers of defense ensure that a runaway agent cannot cause financial damage: provider caps → gateway limits → velocity detection → observability alerts → organizational review. Each layer is independent — any single layer failing still leaves four others protecting the budget.

The system costs ~$0/month to run (self-hosted, open-source) while protecting against potentially thousands in runaway spend. That's the best ROI in the entire factory.
