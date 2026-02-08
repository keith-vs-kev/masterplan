# 14 — Monitoring, Observability & Cost Control

> How to monitor an AI factory, prevent runaway costs, and know when things go wrong.

---

## The Problem

AI agents are non-deterministic, expensive, and opaque. A single runaway loop can burn hundreds of dollars in minutes. Traditional APM (Datadog, New Relic) doesn't understand tokens, traces with branching tool calls, or the concept of "the agent got stuck in a loop." You need purpose-built observability.

**Core challenges:**
- Token costs are variable and unpredictable (a summarization task might cost $0.01 or $5.00 depending on context length)
- Agents can enter infinite loops, retry storms, or hallucinate tool calls
- Quality is hard to measure — "it ran successfully" ≠ "it produced good output"
- Multi-agent systems create cascading failure modes
- You're often billed per-token with no natural ceiling

---

## What to Monitor

### 1. Token Usage & Cost

The #1 metric. Track per-request, per-agent, per-user, per-day.

| Metric | Why |
|--------|-----|
| Input tokens per call | Detect context bloat, unnecessary system prompts |
| Output tokens per call | Detect verbose or looping responses |
| Total tokens per task | End-to-end cost of a unit of work |
| Cost per agent/workflow | Which agents are expensive? |
| Cost velocity ($/hour) | Real-time burn rate — critical for alerts |
| Cache hit rate | Are you wasting money re-computing embeddings? |

**Key insight:** Track cost at the *task* level, not just the API call level. An agent that makes 47 API calls to answer one question has a very different cost profile than one that makes 2.

### 2. Error Rates & Failure Modes

| Metric | What it catches |
|--------|-----------------|
| API error rate (4xx, 5xx) | Provider outages, rate limits, auth issues |
| Rate limit hits | Need to add backoff, switch providers, or queue |
| Tool call failures | Broken integrations, schema drift |
| Timeout rate | Slow providers, complex queries hanging |
| Retry rate | Hidden cost multiplier |
| Parse/extraction failures | Model output not matching expected format |

### 3. Agent Performance Metrics

| Metric | Why |
|--------|-----|
| Task completion rate | Did the agent actually finish the job? |
| Steps per task | Efficiency — fewer steps = better |
| Latency (end-to-end) | User experience |
| Latency (per LLM call) | Provider performance |
| Tool usage patterns | Which tools get used? Which are dead weight? |
| Human escalation rate | How often does the agent give up? |
| Quality scores (eval) | Output correctness, relevance, safety |

### 4. Anomaly Detection

Look for:
- **Cost spikes** — sudden increase in $/hour (runaway loop)
- **Token explosions** — input context growing each iteration (agent feeding output back as input)
- **Repetitive tool calls** — same tool called >N times in a trace (stuck loop)
- **Latency degradation** — provider slowing down or model getting confused
- **Error cascades** — one failure triggering retries across multiple agents

---

## The Tool Landscape

### LangSmith (by LangChain)

**What:** Full-lifecycle LLM platform — tracing, evaluation, prompt management, monitoring.

**Strengths:**
- Deep LangChain/LangGraph integration (but framework-agnostic)
- Excellent trace visualization — see every step of an agent's reasoning
- Built-in evaluation framework with human annotation queues
- Dataset management for regression testing
- Production monitoring dashboards

**Weaknesses:**
- Strongest value prop if you're in the LangChain ecosystem
- Hosted SaaS (self-host option limited)
- Pricing can add up at scale

**Best for:** Teams using LangChain/LangGraph who want end-to-end observability + eval in one platform.

### Langfuse

**What:** Open-source LLM engineering platform. Tracing, analytics, evaluation, prompt management.

**Strengths:**
- **Open source & self-hostable** — huge advantage for data sovereignty and cost control
- OpenTelemetry-based — reduces vendor lock-in
- 50+ framework integrations (LangChain, LlamaIndex, OpenAI SDK, Vercel AI, etc.)
- Session tracking, user-level analytics
- Cost tracking built into traces
- Growing fast, strong community

**Weaknesses:**
- Younger than LangSmith — some features still maturing
- Self-hosting requires maintenance (Postgres + ClickHouse)

**Best for:** Teams that want open-source, self-hosted observability. Strong default choice.

### Helicone

**What:** LLM observability + AI gateway. Proxy-based approach — route API calls through Helicone.

**Strengths:**
- **Gateway model** — acts as a proxy, so integration is just changing the base URL
- 100+ model support through unified API
- 0% markup on API credits (they manage provider keys for you)
- Built-in caching, rate limiting, retries, and fallbacks
- Cost tracking is automatic (every request logged with cost)
- Edge-deployed, <50ms overhead
- **Provider fallbacks** — if OpenAI goes down, auto-route to Anthropic

**Weaknesses:**
- Gateway = single point of routing (though they handle this well)
- Less deep on evaluation/experimentation compared to LangSmith/Langfuse

**Best for:** Teams wanting cost tracking + reliability (fallbacks, caching) with minimal integration effort. Excellent as a cost control layer.

### Arize Phoenix

**What:** Open-source AI observability focused on tracing + evaluation. Built on OpenTelemetry/OpenInference.

**Strengths:**
- **Fully open source** (Apache 2.0)
- Strong tracing with auto-instrumentation (LangChain, LlamaIndex, OpenAI, Anthropic, DSPy, Vercel AI SDK)
- Prompt playground with span replay — debug by replaying LLM calls
- Dataset & experiment framework for systematic testing
- Works across Python, TypeScript, Java
- Good visualization of agent traces as graphs

**Weaknesses:**
- Lighter on production monitoring/alerting compared to Helicone
- Less cost-tracking focus

**Best for:** Teams wanting open-source tracing + evaluation, especially for debugging and experimentation during development.

### Comparison Matrix

| Feature | LangSmith | Langfuse | Helicone | Phoenix |
|---------|-----------|----------|----------|---------|
| Open source | Partial | ✅ Yes | ✅ Yes | ✅ Yes |
| Self-hostable | Limited | ✅ Yes | ✅ Yes | ✅ Yes |
| Tracing | ✅ Excellent | ✅ Good | ✅ Good | ✅ Excellent |
| Cost tracking | ✅ Yes | ✅ Yes | ✅ Best | ⚠️ Basic |
| Evaluation | ✅ Best | ✅ Good | ⚠️ Basic | ✅ Good |
| Prompt mgmt | ✅ Yes | ✅ Yes | ❌ No | ✅ Yes |
| Gateway/proxy | ❌ No | ❌ No | ✅ Yes | ❌ No |
| Provider fallback | ❌ No | ❌ No | ✅ Yes | ❌ No |
| Alerting | ✅ Yes | ⚠️ Basic | ✅ Yes | ⚠️ Basic |
| OTel-based | ❌ No | ✅ Yes | ❌ No | ✅ Yes |
| Pricing | Paid tiers | Free/paid | Free/paid | Free |

### Other Notable Tools

- **LiteLLM** — Open-source LLM proxy/gateway. Route to 100+ providers, track spend, set budgets. Pairs well with Langfuse/Phoenix for observability. Key feature: **budget limits per API key**.
- **Portkey** — AI gateway with observability, caching, fallbacks. Similar to Helicone.
- **Weights & Biases (Weave)** — Tracing and evaluation from the W&B team. Good if you're already in W&B ecosystem.
- **Braintrust** — Eval-focused platform with tracing. Strong on systematic testing.
- **OpenLIT** — Open-source, OpenTelemetry-native. GPU monitoring included.

---

## Custom Solutions: Building Your Own

For an "AI factory" running many agents, you'll likely need a custom layer on top of (or instead of) these tools.

### Architecture

```
┌─────────────────────────────────────────────────┐
│                  AI Factory                       │
│                                                   │
│  Agent 1 ─┐                                      │
│  Agent 2 ─┤──▶ LLM Proxy Layer ──▶ Providers    │
│  Agent N ─┘    (LiteLLM / custom)                │
│                     │                             │
│                     ▼                             │
│              Token/Cost Logger                    │
│                     │                             │
│          ┌──────────┼──────────┐                  │
│          ▼          ▼          ▼                  │
│     Time-series   Traces    Alerts               │
│     (Prometheus)  (Langfuse) (PagerDuty)         │
│          │          │          │                  │
│          └──────────┼──────────┘                  │
│                     ▼                             │
│              Cost Dashboard                       │
│         (Grafana / custom UI)                     │
└─────────────────────────────────────────────────┘
```

### Key Components to Build

**1. LLM Proxy/Gateway**
- Intercept all LLM calls in one place
- Log tokens, latency, model, cost per call
- LiteLLM is the best open-source option here
- Add rate limiting, caching, and fallback logic

**2. Cost Accumulator**
- Aggregate costs by: agent, task, user, time window
- Maintain running totals with Redis or similar
- Calculate rolling averages for anomaly detection

**3. Budget Enforcement**
```python
# Pseudocode for budget enforcement
class BudgetEnforcer:
    def check(self, agent_id, estimated_cost):
        spent = self.get_spent_today(agent_id)
        limit = self.get_daily_limit(agent_id)
        if spent + estimated_cost > limit:
            raise BudgetExceededError(f"{agent_id} would exceed ${limit}/day")
        if self.get_cost_velocity(agent_id) > self.velocity_threshold:
            raise RunawayDetectedError(f"{agent_id} burning too fast")
```

**4. Trace Storage**
- Every agent run = a trace with spans
- Store in Langfuse, Phoenix, or custom (Postgres + time-series)
- Link traces to business outcomes (did the task succeed?)

**5. Alerting Pipeline**
- Cost velocity alerts ($/min exceeds threshold)
- Error rate alerts (>X% failures in Y minutes)
- Loop detection (same tool called >N times)
- Latency alerts (P95 > threshold)
- Daily spend approaching budget

---

## Preventing Runaway Costs

This is the most critical section. A single bug can cost thousands.

### Layer 1: Hard Limits (Non-negotiable)

| Control | Implementation |
|---------|---------------|
| **Provider spend limits** | Set hard caps in OpenAI/Anthropic dashboards. Last line of defense. |
| **Per-API-key budgets** | LiteLLM supports this natively. Each agent gets its own key with a daily cap. |
| **Max tokens per request** | Always set `max_tokens`. Never let a model generate unbounded output. |
| **Max steps per agent run** | Hard cap on tool call iterations (e.g., max 20 steps). |
| **Request timeout** | Kill any LLM call that takes >60s (or whatever makes sense). |

### Layer 2: Smart Controls

| Control | Implementation |
|---------|---------------|
| **Cost velocity monitoring** | If an agent spends >$X in Y minutes, pause it. |
| **Loop detection** | Track tool call sequences. If the same pattern repeats 3+ times, kill the run. |
| **Context window guards** | If input tokens approach model limit, truncate or summarize instead of paying for max context. |
| **Caching** | Cache identical or semantically similar requests. Huge savings for repetitive queries. |
| **Model tiering** | Use cheap models (GPT-4o-mini, Haiku) for simple tasks. Only escalate to expensive models when needed. |
| **Batch processing** | Batch API calls where possible (OpenAI Batch API is 50% cheaper). |

### Layer 3: Kill Switches

Every AI factory needs kill switches at multiple levels:

```
Global kill switch     → Stop ALL LLM calls (nuclear option)
Per-agent kill switch  → Stop one misbehaving agent
Per-provider switch    → Stop calls to one provider (outage)
Per-user switch        → Stop one user's agents (abuse)
Gradual throttle       → Reduce call rate instead of full stop
```

**Implementation:** The LLM proxy is the enforcement point. A simple Redis flag check before each API call:

```python
async def call_llm(request):
    if await redis.get("kill:global"):
        raise KillSwitchError("Global kill switch active")
    if await redis.get(f"kill:agent:{request.agent_id}"):
        raise KillSwitchError(f"Agent {request.agent_id} killed")
    # ... proceed with call
```

### Layer 4: Financial Controls

- **Daily budget alerts** at 50%, 80%, 100% of budget → Slack/email/PagerDuty
- **Weekly cost reviews** — is spend trending up? Why?
- **Per-task cost tracking** — know exactly what each unit of work costs
- **Cost attribution** — if agents serve users, know cost per user
- **Prepaid credits model** — don't use post-paid billing if you can avoid it

---

## Recommended Stack for an AI Factory

### Minimum Viable Monitoring

For getting started:

1. **LiteLLM** as the proxy layer (budget limits, logging, model routing)
2. **Langfuse** (self-hosted) for tracing and cost analytics
3. **Prometheus + Grafana** for real-time metrics and alerting
4. **Custom budget enforcer** (simple Redis-based, ~100 lines of code)

### Production Grade

1. **LiteLLM or Helicone** as the gateway
2. **Langfuse** for deep tracing and evaluation
3. **Custom cost accumulator** with per-agent budgets
4. **Prometheus + Grafana** for dashboards
5. **PagerDuty/OpsGenie** for on-call alerts
6. **Kill switch system** in the proxy layer
7. **Weekly automated cost reports**

### Enterprise

All of the above, plus:
- SOC2-compliant trace storage
- Role-based access to observability data
- Audit logs for who changed what
- Multi-region failover
- Custom eval pipelines for quality monitoring
- SLA monitoring per agent/workflow

---

## Key Metrics Dashboard

What your Grafana/dashboard should show at a glance:

```
┌─────────────────────────────────────────────┐
│  AI Factory Dashboard                        │
├──────────────┬──────────────┬───────────────┤
│ TODAY: $47.23│ MTD: $892.41 │ Budget: 78%   │
├──────────────┴──────────────┴───────────────┤
│                                              │
│  Cost/hour (last 24h)     [sparkline graph] │
│  Active agents: 12/15     [status dots]     │
│  Error rate: 2.1%         [trend arrow ↓]   │
│  Avg latency: 1.2s        [trend arrow →]   │
│  Cache hit rate: 34%      [trend arrow ↑]   │
│                                              │
├──────────────────────────────────────────────┤
│  Top Cost Agents (today)                     │
│  1. research-agent    $12.40  ████████░░     │
│  2. code-review       $8.20   █████░░░░░     │
│  3. email-assistant   $6.10   ████░░░░░░     │
├──────────────────────────────────────────────┤
│  Alerts (last 24h)                           │
│  ⚠️  research-agent exceeded velocity limit  │
│  ✅  Resolved: API rate limit on OpenAI      │
└──────────────────────────────────────────────┘
```

---

## Practical Patterns

### Pattern: The Cost-Aware Agent

Agents should know their own budget:

```python
class CostAwareAgent:
    def __init__(self, budget_limit=1.0):
        self.budget_limit = budget_limit
        self.spent = 0.0
    
    async def step(self):
        if self.spent > self.budget_limit * 0.8:
            # Switch to cheaper model
            self.model = "gpt-4o-mini"
        if self.spent > self.budget_limit:
            return self.graceful_shutdown()
```

### Pattern: Exponential Backoff on Cost

If an agent's cost is accelerating, slow it down:

```python
if cost_velocity > threshold:
    delay = min(base_delay * (2 ** consecutive_fast_calls), max_delay)
    await sleep(delay)
```

### Pattern: Shadow Mode

Before deploying a new agent to production, run it in shadow mode:
- Execute the full workflow but don't apply side effects
- Log all costs and outputs
- Compare against baseline
- Only promote to production after cost and quality validation

### Pattern: Token Budget Allocation

For complex multi-step tasks, pre-allocate a token budget:

```
Total budget: 10,000 tokens
- Planning: 2,000 tokens
- Research (3 calls): 5,000 tokens  
- Synthesis: 2,000 tokens
- Buffer: 1,000 tokens
```

If any step exceeds its allocation, truncate or abort before cascading.

---

## Anti-Patterns to Avoid

1. **No limits anywhere** — "We'll add monitoring later" → $5,000 surprise bill
2. **Logging everything to stdout** — Useless at scale. Use structured tracing.
3. **Alerting on every error** — Alert fatigue. Alert on *rates* and *trends*, not individual errors.
4. **Cost tracking only at the API level** — You need task-level cost attribution to make business decisions.
5. **Manual kill switches only** — If you have to SSH in to stop a runaway agent, it's too slow. Automate it.
6. **Ignoring cache opportunities** — Many agent workflows repeat similar queries. Semantic caching can cut costs 20-40%.
7. **Same model for everything** — Using GPT-4 for "yes/no" classification is burning money.

---

## Bottom Line

1. **Start with hard limits** — Provider caps, per-key budgets, max tokens. Do this before writing a single agent.
2. **Add a proxy layer** — LiteLLM or Helicone. This is your control plane.
3. **Instrument everything** — Langfuse or Phoenix for traces. Every LLM call, every tool call, every cost.
4. **Build kill switches early** — They're trivial to implement and save you from disasters.
5. **Monitor cost velocity, not just total cost** — A slow $100/day spend is fine. $100 in 5 minutes is a fire.
6. **Tier your models** — Cheap models for cheap tasks. Expensive models only when justified.
7. **Review costs weekly** — Trends matter more than absolute numbers.

The goal isn't to minimize cost — it's to maximize value per dollar while having guardrails that prevent catastrophic spend. An agent that costs $50/day but generates $500 in value is great. An agent that costs $50/day because it's stuck in a loop is a fire.
