# 13 — Hybrid Compute Layer

> **Status:** Architecture document
> **Date:** 2026-02-08
> **Hardware:** dreamteam — 1× RTX 3090 (24GB VRAM), Ryzen/Intel CPU, 64GB+ RAM

---

## Overview

The agentic factory runs a **hybrid compute layer**: local GPU inference on dreamteam for volume/privacy/latency, cloud APIs for frontier quality and burst capacity. LiteLLM acts as the unified proxy — every agent talks to one endpoint, routing happens transparently.

**Design principles:**
1. Local-first for high-volume, predictable workloads (embeddings, classification, simple generation)
2. Cloud for frontier quality (complex reasoning, novel code architecture)
3. Single API surface — agents don't know or care where inference runs
4. GPU is a shared resource — inference, TTS, and other workloads must coexist

---

## Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    Agent Layer                           │
│  (OpenClaw agents, subagents, cron tasks, pipelines)    │
└────────────────────────┬────────────────────────────────┘
                         │ OpenAI-compatible API
                         ▼
┌─────────────────────────────────────────────────────────┐
│                   LiteLLM Proxy                         │
│                                                         │
│  ┌─────────────┐  ┌──────────────┐  ┌───────────────┐  │
│  │ Router Rules │  │ Budget Mgmt  │  │ Rate Limiting │  │
│  └─────────────┘  └──────────────┘  └───────────────┘  │
│                                                         │
│  Model aliases:                                         │
│    local/fast     → llama.cpp (8B)                      │
│    local/quality  → llama.cpp (32B)                     │
│    cloud/cheap    → Groq Llama 8B / Gemini Flash Lite   │
│    cloud/mid      → Gemini 2.5 Flash / GPT-5 mini      │
│    cloud/frontier → Claude Sonnet 4.5 / Opus 4.6       │
│    embed/local    → nomic-embed-text (local)            │
└──────┬─────────────────┬────────────────┬───────────────┘
       │                 │                │
       ▼                 ▼                ▼
┌─────────────┐  ┌─────────────┐  ┌──────────────┐
│  llama.cpp  │  │  Cloud APIs │  │  Groq/etc    │
│  (dreamteam)│  │  (Anthropic │  │  (speed tier)│
│  :8080      │  │   OpenAI    │  │              │
│             │  │   Google)   │  │              │
└─────────────┘  └─────────────┘  └──────────────┘
```

---

## Routing Decision Matrix

### When to Route Locally

| Condition | Route | Why |
|-----------|-------|-----|
| Embeddings | **Always local** | Trivial compute, high volume, zero cost |
| Classification / extraction | **Local 8B** | Simple task, fast, free |
| Short summarization (<2K tokens) | **Local 8B** | Good enough quality |
| Code completion (routine) | **Local 32B** (Qwen Coder) | Near-frontier quality for standard patterns |
| RAG query synthesis | **Local 8B** | Formatting queries is simple |
| Batch text processing | **Local** | Volume play, no latency pressure |
| Privacy-sensitive data | **Local** | Data never leaves the box |
| Cloud budget exhausted | **Local fallback** | Graceful degradation |

### When to Route to Cloud

| Condition | Route | Why |
|-----------|-------|-----|
| Complex multi-step reasoning | **Cloud frontier** (Opus/Sonnet) | Local can't match quality |
| Novel architecture decisions | **Cloud frontier** | Needs frontier intelligence |
| Long context (>16K tokens) | **Cloud** (Gemini 2.5 Pro) | KV cache eats local VRAM |
| Creative writing / nuance | **Cloud frontier** | Better at subtlety |
| Multi-turn agent loops | **Cloud** | Reliability matters more than cost |
| GPU busy with other workload | **Cloud overflow** | Don't queue, route around |
| Speed-critical simple tasks | **Groq** | 500-1000 tok/s, dirt cheap |

### Routing Logic (Pseudocode)

```python
def route(request):
    # 1. Check GPU availability
    if gpu_busy(threshold=80%):
        return cloud_route(request)
    
    # 2. Task-type routing
    if request.task == "embedding":
        return "local/embed"
    
    if request.estimated_complexity == "simple":
        return "local/fast"  # 8B model
    
    if request.estimated_complexity == "medium":
        if request.context_length < 16_000:
            return "local/quality"  # 32B model
        else:
            return "cloud/mid"  # Gemini Flash
    
    # 3. Complex tasks → cloud
    if daily_cloud_spend < budget_limit:
        return "cloud/frontier"
    else:
        return "local/quality"  # budget fallback
```

---

## Local Inference Stack

### llama.cpp Server

**Why llama.cpp over Ollama/vLLM:**
- Single-user workload (agents, not multi-tenant)
- Lower VRAM overhead than vLLM
- GGUF quantization = best quality-per-bit on single GPU
- Direct control over model loading/unloading
- Built-in OpenAI-compatible API

**Recommended models for dreamteam (1× RTX 3090, 24GB):**

| Slot | Model | Quant | VRAM | Use Case |
|------|-------|-------|------|----------|
| Fast | Llama 3.1 8B | Q6_K | ~7GB | Classification, extraction, triage |
| Quality | Qwen 2.5 Coder 32B | Q4_K_M | ~20GB | Code gen, complex tasks |
| Quality-alt | DeepSeek-R1-Distill-Qwen-32B | Q4_K_M | ~20GB | Reasoning tasks |
| Embed | nomic-embed-text | F16 | ~0.5GB | Embeddings |

**Key constraint:** Only one large model fits in 24GB at a time. Model swapping is required.

### Model Swapping Strategy

The 3090 can't run an 8B and 32B simultaneously. Two approaches:

**Option A: Hot-swap on demand (recommended for low concurrency)**
```bash
# Model manager script
# Keeps 8B loaded by default (low VRAM, fast responses)
# Swaps to 32B when a quality request comes in
# Swaps back after idle timeout (e.g., 5 minutes)

# llama.cpp supports --model-url for lazy loading, but swap time is ~10-30s
# Mitigate: pre-download models to NVMe, swap is just VRAM load
```

- Default state: 8B model loaded (~7GB VRAM), leaves room for TTS/other
- On quality request: unload 8B, load 32B (~20GB), serve request
- After 5min idle: unload 32B, reload 8B
- Swap time: ~15-30s (NVMe → VRAM). Acceptable for async agent tasks.
- During swap: route to cloud (LiteLLM fallback)

**Option B: Dual-server with CPU offload**
```bash
# Run 8B fully on GPU + 32B with partial CPU offload
# 32B with ~16 layers on GPU + rest on CPU/RAM
# Slower for 32B (~5-8 tok/s) but always available
```

Not recommended — 5 tok/s is painfully slow for 32B. Better to swap cleanly.

**Option C: Embedding model always resident**
- nomic-embed-text is tiny (~0.5GB). Keep it loaded in a separate llama.cpp instance on CPU.
- GPU instance handles either 8B or 32B exclusively.

### llama.cpp Server Configuration

```bash
# Primary inference server (GPU)
llama-server \
  --model /models/llama-3.1-8b-instruct-q6_k.gguf \
  --host 0.0.0.0 \
  --port 8080 \
  --n-gpu-layers 99 \
  --ctx-size 8192 \
  --flash-attn \
  --threads 8 \
  --parallel 2 \
  --cont-batching

# Embedding server (CPU, always on)
llama-server \
  --model /models/nomic-embed-text-v1.5.Q8_0.gguf \
  --host 0.0.0.0 \
  --port 8081 \
  --embedding \
  --threads 4 \
  --ctx-size 2048
```

### Model Manager Service

A lightweight systemd service or daemon that:

1. **Monitors request queue** via LiteLLM webhook/middleware
2. **Decides which model should be loaded** based on pending request types
3. **Manages llama.cpp lifecycle** — stop server, swap model file, restart
4. **Reports status** to LiteLLM so it can route during swaps
5. **Tracks GPU state** — is something else using the GPU? (TTS, etc.)

```python
# model_manager.py (simplified)
class ModelManager:
    current_model = "8b"
    last_request_time = None
    IDLE_TIMEOUT = 300  # 5 min
    
    def request_model(self, model_class: str):
        if model_class == self.current_model:
            return True  # already loaded
        
        if gpu_in_use_by_other():  # TTS, etc.
            return False  # caller should use cloud
        
        self.swap_model(model_class)
        return True
    
    def swap_model(self, target):
        stop_llama_server()
        model_path = MODEL_PATHS[target]
        start_llama_server(model_path)
        self.current_model = target
    
    def idle_check(self):
        if (self.current_model != "8b" and 
            time.time() - self.last_request_time > self.IDLE_TIMEOUT):
            self.swap_model("8b")
```

---

## GPU Scheduling: Shared Resource Management

The RTX 3090 serves multiple workloads. They **cannot all run simultaneously at full capacity**.

### Workload Priority & VRAM Budget

| Priority | Workload | VRAM Needed | Frequency | Duration |
|----------|----------|-------------|-----------|----------|
| 1 (highest) | LLM inference (agents) | 7-20GB | Continuous | Seconds-minutes |
| 2 | TTS (ElevenLabs is cloud, but local TTS fallback) | 2-4GB | Intermittent | Seconds |
| 3 | Embedding generation | 0.5GB (CPU) | Burst | Milliseconds |
| 4 | Image generation (Stable Diffusion) | 6-10GB | Rare | Seconds |
| 5 (lowest) | Fine-tuning / training | 20-24GB | Rare | Hours |

### Scheduling Rules

1. **LLM inference is king.** Other workloads yield to it.
2. **TTS runs on cloud** (ElevenLabs) by default. Local TTS (Piper/XTTS) only as fallback, and only when GPU is idle.
3. **Embeddings run on CPU** — never compete for GPU.
4. **Image gen and fine-tuning are offline tasks** — schedule during low-activity windows (overnight NZ time, ~2am-8am).
5. **No concurrent large VRAM consumers** — mutex lock on GPU for model loads >10GB.

### Implementation: GPU Mutex

```bash
# /var/run/gpu-lock — simple flock-based mutex
# Any process needing >10GB VRAM must acquire this lock

gpu_acquire() {
    exec 200>/var/run/gpu.lock
    flock -n 200 || return 1  # non-blocking, fail if busy
}

gpu_release() {
    flock -u 200
}
```

More sophisticated: a small GPU scheduler daemon that tracks VRAM usage via `nvidia-smi` and queues/preempts workloads.

---

## LiteLLM Configuration

### Setup

```yaml
# litellm_config.yaml
model_list:
  # Local models
  - model_name: local/fast
    litellm_params:
      model: openai/llama-3.1-8b
      api_base: http://localhost:8080/v1
      api_key: none
    model_info:
      max_tokens: 8192
      input_cost_per_token: 0
      output_cost_per_token: 0

  - model_name: local/quality
    litellm_params:
      model: openai/qwen-2.5-coder-32b
      api_base: http://localhost:8080/v1
      api_key: none
    model_info:
      max_tokens: 8192
      input_cost_per_token: 0
      output_cost_per_token: 0

  - model_name: local/embed
    litellm_params:
      model: openai/nomic-embed-text
      api_base: http://localhost:8081/v1
      api_key: none

  # Cloud speed tier
  - model_name: cloud/fast
    litellm_params:
      model: groq/llama-3.1-8b-instant
      api_key: os.environ/GROQ_API_KEY

  # Cloud mid tier
  - model_name: cloud/mid
    litellm_params:
      model: gemini/gemini-2.5-flash
      api_key: os.environ/GEMINI_API_KEY

  # Cloud frontier
  - model_name: cloud/frontier
    litellm_params:
      model: anthropic/claude-sonnet-4-5-20250514
      api_key: os.environ/ANTHROPIC_API_KEY

  # Fallback frontier
  - model_name: cloud/frontier
    litellm_params:
      model: anthropic/claude-opus-4-6-20250610
      api_key: os.environ/ANTHROPIC_API_KEY

router_settings:
  routing_strategy: simple-shuffle  # for models with same name, load-balance
  num_retries: 2
  timeout: 120
  fallbacks:
    - local/fast: [cloud/fast]
    - local/quality: [cloud/mid]

general_settings:
  master_key: os.environ/LITELLM_MASTER_KEY
  database_url: os.environ/LITELLM_DB_URL  # PostgreSQL for usage tracking
  max_budget: 100  # monthly cloud spend cap in USD
  budget_duration: 1mo
```

### Running LiteLLM

```bash
# As a systemd service
litellm --config litellm_config.yaml --port 4000

# All agents use: http://localhost:4000/v1 as their OpenAI base URL
```

### Agent Integration

Agents don't need to know about routing. They request a model alias:
- `local/fast` for quick tasks
- `local/quality` for important local tasks  
- `cloud/frontier` for complex reasoning
- Or use a meta-model name and let the router decide

---

## Cost Optimization Math

### Current Baseline (dreamteam context)

**Assumptions:**
- Factory runs ~50 agent tasks/day average
- ~2M tokens/day total (input + output)
- Mix: 60% simple, 30% medium, 10% complex

**Strategy A: All cloud frontier (Claude Sonnet 4.5)**
```
2M tok/day × $9/MTok avg × 30 days = $540/month
```

**Strategy B: All cloud cheap (Gemini 2.5 Flash)**
```
2M tok/day × $1.40/MTok avg × 30 days = $84/month
```

**Strategy C: Hybrid (recommended)**
```
Simple (60% = 1.2M tok/day):  Local         → $0 (electricity only)
Medium (30% = 0.6M tok/day):  Local 32B     → $0 (electricity only)  
Complex (10% = 0.2M tok/day): Cloud frontier → 0.2M × $9 × 30 = $54/month

Electricity: ~200W avg × 24/7 = 144 kWh/mo × $0.30 = $43/month
Hardware amortization: $1,810 / 24 months = $75/month

Total: $54 + $43 + $75 = $172/month (first 2 years)
Total after hardware paid off: $54 + $43 = $97/month
```

**Strategy D: Hybrid + aggressive cost optimization**
```
Same split, but:
- Use prompt caching on cloud calls (90% input savings)
- Use batch API for non-urgent complex tasks (50% off)
- Use Groq for speed-critical simple tasks that happen during GPU swap

Cloud cost: ~$30/month (with caching + batch)
Electricity: $43/month
Hardware: $75/month (amortized)

Total: $148/month → $73/month after hardware paid off
```

### Savings Summary

| Strategy | Monthly Cost | vs All-Frontier |
|----------|-------------|-----------------|
| All Sonnet 4.5 | $540 | baseline |
| All Gemini Flash | $84 | -84% |
| Hybrid (basic) | $172→$97 | -68%→-82% |
| Hybrid (optimized) | $148→$73 | -73%→-86% |

### Scaling Inflection Points

- **At 5M tok/day:** Local savings double. Add second 3090 consideration.
- **At 10M tok/day:** Local handles $300+/mo worth of cloud compute. Second GPU pays for itself in 3 months.
- **At 20M+ tok/day:** Consider dedicated inference box or cloud GPU rental (RunPod H100 @ $2.50/hr).

---

## Operations & Management

### Systemd Services

```ini
# /etc/systemd/system/llama-inference.service
[Unit]
Description=llama.cpp Inference Server
After=network.target

[Service]
Type=simple
User=adam
ExecStart=/usr/local/bin/llama-server \
  --model /models/current-model.gguf \
  --host 0.0.0.0 --port 8080 \
  --n-gpu-layers 99 --ctx-size 8192 \
  --flash-attn --parallel 2
Restart=on-failure
RestartSec=5
Environment=CUDA_VISIBLE_DEVICES=0

[Install]
WantedBy=multi-user.target
```

```ini
# /etc/systemd/system/llama-embed.service
[Unit]
Description=llama.cpp Embedding Server (CPU)
After=network.target

[Service]
Type=simple
User=adam
ExecStart=/usr/local/bin/llama-server \
  --model /models/nomic-embed-text.gguf \
  --host 0.0.0.0 --port 8081 \
  --embedding --threads 4 --ctx-size 2048
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

```ini
# /etc/systemd/system/litellm.service
[Unit]
Description=LiteLLM Proxy
After=llama-inference.service

[Service]
Type=simple
User=adam
ExecStart=/usr/local/bin/litellm \
  --config /etc/litellm/config.yaml --port 4000
Restart=on-failure
EnvironmentFile=/etc/litellm/env

[Install]
WantedBy=multi-user.target
```

### Monitoring

**Key metrics to track:**
- GPU VRAM usage and utilization (`nvidia-smi` → Prometheus exporter)
- llama.cpp throughput (tokens/sec), queue depth, latency (p50/p95)
- LiteLLM request counts by model, costs, error rates
- Model swap frequency and duration
- Cloud spend (daily/weekly/monthly)

**Health checks:**
```bash
# llama.cpp health
curl -s http://localhost:8080/health | jq .status

# LiteLLM health  
curl -s http://localhost:4000/health
```

### Model Updates

New models drop frequently. Process:
1. Download GGUF to `/models/` (use `huggingface-cli download`)
2. Test quality with eval suite (small benchmark of representative tasks)
3. Update symlink `/models/current-model.gguf`
4. Restart llama-inference service
5. Update LiteLLM config if model name changed

---

## Future Scaling Path

| Milestone | Action | Cost |
|-----------|--------|------|
| **Now** | 1× RTX 3090, llama.cpp + LiteLLM | $0 (already have it) |
| **10M tok/day** | Add 2nd RTX 3090 (NVLink for 48GB) | ~$800 |
| **20M tok/day** | Dedicated inference node, separate from dev workstation | ~$2,000 |
| **50M+ tok/day** | Cloud GPU rental (RunPod/vast.ai) or enterprise inference | Variable |
| **Revenue > $5K/mo** | H100 cloud instance for production workloads | ~$2.50/hr |

### Second GPU Benefits
- Run 70B models fully in VRAM (Q4_K_M) — massive quality jump
- Run 8B + 32B simultaneously — no more swapping
- RTX 3090 supports NVLink (unlike 4090!) — 48GB unified VRAM
- Cost: ~$800 used. Pays for itself in 2-3 months at scale.

---

## Summary

The hybrid compute layer is built on three components:

1. **llama.cpp on dreamteam** — local inference for 90% of token volume at near-zero marginal cost
2. **LiteLLM proxy** — unified API surface with intelligent routing, budget controls, and fallbacks  
3. **Cloud APIs** — frontier quality for the 10% of tasks that need it, plus overflow capacity

This architecture saves 70-86% vs all-cloud while maintaining frontier quality where it matters. The GPU is treated as a shared resource with clear scheduling priorities. Model swapping handles the single-GPU constraint gracefully with cloud fallback during transitions.

**Start with:** 8B model as default, cloud frontier for complex tasks, embeddings on CPU. Add model swapping and 32B local inference as agent volume grows.
