# 02 — Smart Router & Model Selection

> Architecture document for the AI Factory's LLM routing layer.
> Decides which provider/model handles each request based on task requirements, cost, speed, and quality.

---

## Overview

Every LLM call in the factory flows through a **Smart Router** — a proxy layer that inspects the request, classifies its requirements, selects the optimal provider/model, enforces budgets, and records costs. No agent calls providers directly.

```
┌──────────────────────────────────────────────────────────┐
│                      AGENTS / WORKERS                     │
│  (code-agent, research-agent, review-agent, etc.)        │
└────────────────────────┬─────────────────────────────────┘
                         │  OpenAI-compatible API
                         ▼
┌──────────────────────────────────────────────────────────┐
│                    SMART ROUTER                           │
│                                                          │
│  ┌────────────┐  ┌────────────┐  ┌───────────────────┐  │
│  │  Request    │  │  Budget    │  │  Provider Health  │  │
│  │  Classifier │  │  Enforcer  │  │  Monitor          │  │
│  └─────┬──────┘  └─────┬──────┘  └────────┬──────────┘  │
│        │               │                   │             │
│        └───────────┬───┘───────────────────┘             │
│                    ▼                                     │
│           ┌────────────────┐                             │
│           │ Routing Engine │                             │
│           └───────┬────────┘                             │
│                   │                                      │
│        ┌──────────┼──────────────┐                       │
│        ▼          ▼              ▼                       │
│   ┌─────────┐ ┌─────────┐ ┌──────────┐                  │
│   │ Local   │ │ Speed   │ │ Frontier │                   │
│   │ (Ollama)│ │ (Groq)  │ │ (Claude/ │                   │
│   │         │ │         │ │  OpenAI) │                   │
│   └─────────┘ └─────────┘ └──────────┘                  │
│                                                          │
│  ┌──────────────────────────────────────────────────┐    │
│  │  Telemetry: cost, tokens, latency → Langfuse     │    │
│  └──────────────────────────────────────────────────┘    │
└──────────────────────────────────────────────────────────┘
```

---

## Core Design Decisions

### LiteLLM as the Routing Foundation

We use **LiteLLM Proxy** as the base layer rather than building from scratch. Rationale:

| Consideration | LiteLLM | Custom Router | OpenRouter |
|---------------|---------|---------------|------------|
| Multi-provider support | 100+ providers | Build it | 100+ (hosted) |
| Per-key budgets | ✅ Built-in | Build it | ❌ |
| Cost tracking | ✅ Built-in | Build it | ✅ |
| Fallback chains | ✅ Built-in | Build it | ✅ |
| Local model support | ✅ Ollama/vLLM | Build it | ❌ |
| Self-hosted | ✅ | ✅ | ❌ |
| Latency overhead | <10ms | Minimal | Network hop to their proxy |
| Vendor lock-in | Low (OSS) | None | Medium |
| Customization | Config + callbacks | Unlimited | Limited |

**Decision:** LiteLLM Proxy with custom routing callbacks for our classification logic. We get 80% of the functionality for free and build the 20% that's unique to us.

### Why Not Pure OpenRouter

OpenRouter adds a margin and a network hop. Fine for prototyping, not for a cost-optimized factory running thousands of calls/day. We keep OpenRouter as a discovery/fallback mechanism only.

---

## Provider Registry

All providers are registered with capabilities, pricing, health status, and priority.

```yaml
# provider-registry.yaml
providers:
  # ── Frontier (quality-critical tasks) ──────────────────
  anthropic:
    models:
      opus-4.6:
        id: "claude-opus-4-6-20250514"
        tier: frontier
        input_cost_mtok: 5.00
        output_cost_mtok: 25.00
        context_window: 200000
        speed_tok_s: 60
        capabilities: [reasoning, coding, agents, creative, vision]
        cache_discount: 0.90  # 90% off cached input
        batch_discount: 0.50
      sonnet-4.5:
        id: "claude-sonnet-4-5-20250514"
        tier: premium
        input_cost_mtok: 3.00
        output_cost_mtok: 15.00
        context_window: 200000
        speed_tok_s: 70
        capabilities: [reasoning, coding, agents, creative, vision]
        cache_discount: 0.90
        batch_discount: 0.50
      haiku-4.5:
        id: "claude-haiku-4-5-20250514"
        tier: mid
        input_cost_mtok: 1.00
        output_cost_mtok: 5.00
        context_window: 200000
        speed_tok_s: 100
        capabilities: [general, extraction, classification, vision]

  openai:
    models:
      gpt-5-mini:
        id: "gpt-5-mini"
        tier: mid
        input_cost_mtok: 0.25
        output_cost_mtok: 2.00
        context_window: 128000
        speed_tok_s: 80
        capabilities: [general, coding, tool_use]
        cache_discount: 0.90
      gpt-5.2:
        id: "gpt-5.2"
        tier: frontier
        input_cost_mtok: 1.75
        output_cost_mtok: 14.00
        context_window: 128000
        speed_tok_s: 70
        capabilities: [reasoning, coding, creative, vision]

  # ── Value (cost-optimized) ─────────────────────────────
  google:
    models:
      gemini-2.5-flash:
        id: "gemini-2.5-flash"
        tier: mid
        input_cost_mtok: 0.30
        output_cost_mtok: 2.50
        context_window: 1000000
        speed_tok_s: 120
        capabilities: [general, coding, vision, long_context]
        cache_discount: 0.90
        batch_discount: 0.50
      gemini-2.5-flash-lite:
        id: "gemini-2.5-flash-lite"
        tier: budget
        input_cost_mtok: 0.10
        output_cost_mtok: 0.40
        context_window: 1000000
        speed_tok_s: 150
        capabilities: [general, extraction, classification]
      gemini-2.5-pro:
        id: "gemini-2.5-pro"
        tier: premium
        input_cost_mtok: 1.25
        output_cost_mtok: 10.00
        context_window: 1000000
        speed_tok_s: 80
        capabilities: [reasoning, coding, long_context, vision]

  # ── Speed (latency-critical) ───────────────────────────
  groq:
    models:
      llama-3.1-8b:
        id: "llama-3.1-8b-instant"
        tier: budget
        input_cost_mtok: 0.05
        output_cost_mtok: 0.08
        context_window: 128000
        speed_tok_s: 840
        capabilities: [general, extraction, classification]
      gpt-oss-120b:
        id: "gpt-oss-120b"
        tier: mid
        input_cost_mtok: 0.15
        output_cost_mtok: 0.60
        context_window: 128000
        speed_tok_s: 500
        capabilities: [general, coding, reasoning]
      llama-4-maverick:
        id: "meta-llama/llama-4-maverick-17b-128e-instruct"
        tier: mid
        input_cost_mtok: 0.20
        output_cost_mtok: 0.60
        context_window: 128000
        speed_tok_s: 562
        capabilities: [general, coding]

  # ── Open-source hosts ──────────────────────────────────
  together:
    models:
      deepseek-r1:
        id: "deepseek-ai/DeepSeek-R1-0528"
        tier: premium
        input_cost_mtok: 3.00
        output_cost_mtok: 7.00
        context_window: 128000
        speed_tok_s: 100
        capabilities: [reasoning]
      qwen3-235b:
        id: "Qwen/Qwen3-235B"
        tier: mid
        input_cost_mtok: 0.20
        output_cost_mtok: 0.60
        context_window: 131000
        speed_tok_s: 120
        capabilities: [general, coding, reasoning]

  fireworks:
    models:
      gpt-oss-120b:
        id: "accounts/fireworks/models/gpt-oss-120b"
        tier: mid
        input_cost_mtok: 0.15
        output_cost_mtok: 0.60
        context_window: 128000
        speed_tok_s: 150
        capabilities: [general, coding]

  # ── Local (privacy, free after hardware) ───────────────
  local-ollama:
    base_url: "http://localhost:11434"
    models:
      llama-3.1-8b:
        id: "ollama/llama3.1:8b"
        tier: budget
        input_cost_mtok: 0.00
        output_cost_mtok: 0.00
        context_window: 128000
        speed_tok_s: 50
        capabilities: [general, extraction, classification]
      qwen-coder-32b:
        id: "ollama/qwen2.5-coder:32b"
        tier: mid
        input_cost_mtok: 0.00
        output_cost_mtok: 0.00
        context_window: 32000
        speed_tok_s: 20
        capabilities: [coding]
```

---

## Routing Rules

### Request Classification

Every request carries (or is inferred to have) a **routing intent**:

```typescript
interface RoutingIntent {
  task_type: "classification" | "extraction" | "summarization" | "coding"
               | "reasoning" | "creative" | "general" | "embedding";
  quality: "best" | "good" | "acceptable";   // quality threshold
  speed: "instant" | "fast" | "normal" | "batch";  // latency requirement
  privacy: "local_only" | "any";             // data sensitivity
  max_cost_usd?: number;                     // per-call budget cap
  context_size?: number;                     // estimated input tokens
  agent_id: string;                          // for budget tracking
}
```

**How intent is determined (priority order):**
1. **Explicit** — agent passes `model` or routing hints in metadata/headers
2. **Agent profile** — each agent has a default routing profile (see below)
3. **Heuristic** — classify based on system prompt content, message length, presence of code blocks

### Agent Routing Profiles

Each agent is assigned a profile that sets defaults:

```yaml
agent_profiles:
  # Heavy coding agent — needs quality, less price-sensitive
  code-agent:
    default_quality: best
    default_speed: normal
    preferred_models: [sonnet-4.5, gpt-oss-120b]
    fallback_models: [gpt-5-mini, gemini-2.5-flash]
    daily_budget_usd: 20.00
    max_cost_per_call_usd: 0.50

  # Research agent — long context, moderate quality
  research-agent:
    default_quality: good
    default_speed: normal
    preferred_models: [gemini-2.5-pro, gemini-2.5-flash]
    fallback_models: [sonnet-4.5]
    daily_budget_usd: 15.00
    max_cost_per_call_usd: 1.00

  # Quick-task agent — speed and cost matter most
  task-runner:
    default_quality: acceptable
    default_speed: fast
    preferred_models: [groq/llama-3.1-8b, gemini-2.5-flash-lite]
    fallback_models: [haiku-4.5]
    daily_budget_usd: 5.00
    max_cost_per_call_usd: 0.05

  # Review/QA agent — needs frontier quality
  qa-agent:
    default_quality: best
    default_speed: normal
    preferred_models: [opus-4.6, gpt-5.2]
    fallback_models: [sonnet-4.5]
    daily_budget_usd: 30.00
    max_cost_per_call_usd: 2.00
```

### The Routing Decision Tree

```
REQUEST ARRIVES
     │
     ▼
[1] Budget check ──── OVER BUDGET? ──→ Reject / queue for tomorrow
     │ OK
     ▼
[2] Privacy check ─── LOCAL_ONLY? ──→ Route to local-ollama
     │ ANY
     ▼
[3] Explicit model? ── YES ──→ Use it (if healthy + within budget)
     │ NO
     ▼
[4] Speed requirement?
     ├── INSTANT (<200ms TTFT) ──→ Groq (fastest available)
     ├── FAST (<1s TTFT) ──→ Groq or Gemini Flash
     ├── BATCH (async OK) ──→ Use batch API (50% off)
     │── NORMAL ──→ continue
     ▼
[5] Context size?
     ├── >200K tokens ──→ Gemini 2.5 Pro (1M context, avoid 2x pricing elsewhere)
     │── ≤200K ──→ continue
     ▼
[6] Quality × Cost optimization
     ├── BEST quality ──→ Pick from frontier tier
     │   └── Select cheapest frontier model with required capabilities
     ├── GOOD quality ──→ Pick from premium/mid tier
     │   └── Prefer: gpt-oss-120b > gemini-2.5-flash > gpt-5-mini
     ├── ACCEPTABLE ──→ Pick from budget tier
     │   └── Prefer: groq/llama-8b > gemini-flash-lite > local
     ▼
[7] Provider health check
     ├── Primary healthy? ──→ Use it
     ├── Primary degraded? ──→ Use fallback
     └── All degraded? ──→ Queue + alert
```

### Capability Matching

The router also ensures the selected model has the right capabilities:

- **Vision tasks** → must have `vision` capability
- **Long context** → must have context_window ≥ input size
- **Structured output / JSON mode** → most models support this, but verify
- **Tool use** → ensure model supports function calling

---

## Fallback Chains

Every model selection includes an ordered fallback chain. If the primary fails (rate limit, outage, timeout), the router tries the next option automatically.

```yaml
fallback_chains:
  frontier:
    - anthropic/opus-4.6
    - openai/gpt-5.2
    - google/gemini-2.5-pro
    - together/deepseek-r1       # reasoning fallback

  premium:
    - anthropic/sonnet-4.5
    - google/gemini-2.5-pro
    - openai/gpt-5.2

  mid:
    - groq/gpt-oss-120b          # fast + cheap
    - fireworks/gpt-oss-120b     # redundancy
    - google/gemini-2.5-flash
    - anthropic/haiku-4.5
    - openai/gpt-5-mini

  budget:
    - groq/llama-3.1-8b
    - google/gemini-2.5-flash-lite
    - local-ollama/llama-3.1-8b  # free fallback

  local:
    - local-ollama/qwen-coder-32b
    - local-ollama/llama-3.1-8b
    - groq/llama-3.1-8b          # escape to cloud if local is down
```

**Fallback triggers:**
- HTTP 429 (rate limited) → immediate fallback
- HTTP 5xx → retry once, then fallback
- Timeout (>30s no response) → fallback
- HTTP 402/403 (billing/auth) → fallback + alert

**Retry policy:**
- Max 1 retry on same provider (with exponential backoff)
- Max 3 total attempts across fallback chain
- If all fail → queue the request + alert ops

---

## Cost Tracking & Budget Enforcement

### Per-Call Cost Calculation

```python
def calculate_cost(response, model_config):
    input_cost = (response.usage.prompt_tokens / 1_000_000) * model_config.input_cost_mtok
    output_cost = (response.usage.completion_tokens / 1_000_000) * model_config.output_cost_mtok

    # Apply cache discount if applicable
    if response.usage.prompt_tokens_cached:
        cached_savings = (response.usage.prompt_tokens_cached / 1_000_000) \
                         * model_config.input_cost_mtok * model_config.cache_discount
        input_cost -= cached_savings

    return input_cost + output_cost
```

### Budget Layers

```
Layer 1: Provider hard caps     → Set in provider dashboards (last resort)
Layer 2: Global daily cap       → $100/day factory-wide (configurable)
Layer 3: Per-agent daily cap    → From agent_profiles (e.g., $20/day)
Layer 4: Per-call cap           → Reject calls estimated to exceed threshold
Layer 5: Velocity alert         → If any agent burns >$5 in 5 minutes → pause + alert
```

### Cost Tracking Storage

```
Redis (real-time):
  cost:global:2026-02-08         → running daily total
  cost:agent:code-agent:2026-02-08 → per-agent daily total
  cost:velocity:code-agent       → sliding window (last 5 min spend)

Postgres (historical):
  llm_calls table:
    id, timestamp, agent_id, task_id, provider, model,
    input_tokens, output_tokens, cached_tokens,
    cost_usd, latency_ms, status, error
```

### Cost Dashboard Metrics

| Metric | Granularity | Alert Threshold |
|--------|-------------|-----------------|
| Total spend today | Global | >80% of daily cap |
| Spend velocity ($/min) | Per-agent | >$1/min |
| Avg cost per task | Per-agent | >2x 7-day average |
| Cache hit rate | Global | <20% (are we wasting money?) |
| Batch vs realtime ratio | Global | Informational |

---

## Prompt Caching Strategy

Caching is the single biggest cost lever (up to 90% savings on input tokens).

### What to Cache
- **System prompts** — identical across calls for the same agent → always cache
- **Tool/function definitions** — same per agent session → always cache
- **Document context in RAG** — pin retrieved docs in cache for multi-turn → cache aggressively
- **Conversation history prefix** — grows incrementally → natural cache hits

### Implementation
LiteLLM handles cache headers automatically for supported providers. We ensure:
1. System prompts are placed first (cache-friendly ordering)
2. Large static context blocks use explicit cache breakpoints (Anthropic `cache_control`)
3. Monitor cache hit rate — if <20%, investigate prompt structure

---

## LiteLLM Proxy Configuration

```yaml
# litellm_config.yaml
model_list:
  # Frontier
  - model_name: "frontier"
    litellm_params:
      model: "claude-opus-4-6-20250514"
      api_key: "os.environ/ANTHROPIC_API_KEY"
    model_info:
      max_tokens: 200000

  - model_name: "frontier"
    litellm_params:
      model: "gpt-5.2"
      api_key: "os.environ/OPENAI_API_KEY"

  # Premium
  - model_name: "premium"
    litellm_params:
      model: "claude-sonnet-4-5-20250514"
      api_key: "os.environ/ANTHROPIC_API_KEY"

  - model_name: "premium"
    litellm_params:
      model: "gemini-2.5-pro"
      api_key: "os.environ/GOOGLE_API_KEY"

  # Mid-tier (cost-optimized workhorse)
  - model_name: "mid"
    litellm_params:
      model: "groq/gpt-oss-120b"
      api_key: "os.environ/GROQ_API_KEY"

  - model_name: "mid"
    litellm_params:
      model: "gemini/gemini-2.5-flash"
      api_key: "os.environ/GOOGLE_API_KEY"

  # Budget
  - model_name: "budget"
    litellm_params:
      model: "groq/llama-3.1-8b-instant"
      api_key: "os.environ/GROQ_API_KEY"

  - model_name: "budget"
    litellm_params:
      model: "gemini/gemini-2.5-flash-lite"
      api_key: "os.environ/GOOGLE_API_KEY"

  # Local
  - model_name: "local"
    litellm_params:
      model: "ollama/llama3.1:8b"
      api_base: "http://localhost:11434"

router_settings:
  routing_strategy: "latency-based-routing"  # picks fastest healthy provider
  num_retries: 2
  retry_after: 5
  timeout: 60
  fallbacks:
    - frontier: ["premium"]
    - premium: ["mid"]
    - mid: ["budget"]
    - budget: ["local"]

general_settings:
  master_key: "os.environ/LITELLM_MASTER_KEY"
  database_url: "os.environ/DATABASE_URL"
  alerting: ["slack"]

litellm_settings:
  success_callback: ["langfuse"]
  failure_callback: ["langfuse"]
  cache: true
  cache_params:
    type: "redis"
    host: "localhost"
    port: 6379
```

---

## Observability Integration

All calls flow through LiteLLM → Langfuse for tracing:

```
Agent call → LiteLLM Proxy → Provider
                  │
                  ├── Langfuse trace (tokens, cost, latency, model)
                  ├── Redis (real-time cost counters)
                  └── Prometheus metrics (request rate, error rate, latency histograms)
```

**Key Langfuse traces include:**
- `agent_id` — which agent made the call
- `task_id` — what task this is part of
- `routing_decision` — why this model was chosen
- `fallback_used` — did we fall back? From what?
- `cost_usd` — calculated cost

**Prometheus metrics exported:**
- `llm_requests_total{provider, model, agent, status}`
- `llm_tokens_total{provider, model, direction}` (input/output)
- `llm_cost_usd_total{provider, model, agent}`
- `llm_latency_seconds{provider, model, quantile}`
- `llm_fallbacks_total{from_provider, to_provider}`
- `llm_budget_remaining_usd{agent}`

---

## Optimization Roadmap

### Phase 1: Static Routing (MVP)
- LiteLLM proxy with fixed model lists per tier
- Agents request a tier (`budget`, `mid`, `premium`, `frontier`)
- Fallback chains configured statically
- Redis-based budget enforcement
- Langfuse for tracing
- **Target: Week 1-2**

### Phase 2: Smart Classification
- Add request classifier (lightweight local model or rule-based)
- Agents send raw requests; router picks the tier automatically
- Prompt caching optimization
- Batch API integration for async workloads
- **Target: Week 3-4**

### Phase 3: Adaptive Routing
- Track quality scores per model per task type
- Route based on empirical performance, not just assumptions
- A/B test models: send 10% of traffic to challenger model, compare quality
- Dynamic provider health scoring (latency percentiles, error rates)
- Auto-adjust routing weights based on cost/quality ratio
- **Target: Month 2-3**

### Phase 4: Predictive Optimization
- Predict call cost before execution (estimate tokens from prompt length)
- Pre-budget allocation for multi-step agent plans
- Semantic caching (cache semantically similar queries, not just identical)
- Fine-tuned small models to replace expensive frontier calls for specific tasks
- **Target: Month 4+**

---

## Key Design Principles

1. **Every call goes through the router** — no direct provider calls, ever
2. **Default cheap, escalate when needed** — start with the cheapest model that might work; only use frontier when justified
3. **Fail open to fallbacks, not to errors** — if a provider is down, route elsewhere silently
4. **Budget enforcement is non-negotiable** — hard limits at every layer, no exceptions
5. **Measure everything** — you can't optimize what you don't track
6. **Agents don't choose models** — they declare intent (quality, speed, capabilities); the router decides the model
7. **Cache aggressively** — 90% savings on repeated context is too good to ignore
8. **Batch when you can** — 50% off for anything that doesn't need real-time response

---

## Estimated Cost Impact

Assuming a factory running ~50M tokens/day across all agents:

| Strategy | Without Router | With Router | Savings |
|----------|---------------|-------------|---------|
| All frontier (Opus) | ~$750/day | — | — |
| Smart tiering | — | ~$75-150/day | **80-90%** |
| + Prompt caching | — | ~$50-100/day | +30% on cached |
| + Batch API | — | ~$40-80/day | +50% on async |
| + Local for embeddings/simple | — | ~$30-60/day | +free for local |

**Bottom line:** The router turns a potential $22K/month bill into $1-2K/month for the same effective output quality.
