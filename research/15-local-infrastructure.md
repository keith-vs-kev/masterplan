# 15 — Local & Self-Hosted AI Infrastructure

> Research completed: 2026-02-08
> Status: Deep research document

---

## Executive Summary

Local AI inference has matured dramatically. Consumer GPUs (RTX 3090/4090) can now run 70B+ parameter models at usable speeds. The key question isn't "can you run locally?" — it's "when should you?" This document covers the inference engines, hardware considerations, model recommendations, cost analysis, and hybrid routing strategies for maximizing local compute.

---

## 1. Inference Engines Compared

### llama.cpp
- **What:** C/C++ inference engine, GGUF format, CPU+GPU hybrid
- **Best for:** Single-GPU setups, CPU offloading, edge deployment, maximum hardware compatibility
- **Key features:**
  - Quantization: Q2_K through Q8_0, IQ (importance-matrix) quants for better quality at low bits
  - Metal (Apple Silicon), CUDA, Vulkan, ROCm, SYCL support
  - Flash attention, KV cache quantization
  - Built-in HTTP server with OpenAI-compatible API
  - Speculative decoding support
  - Multimodal support (vision models)
- **Performance:** ~40-80 tok/s for 7B Q4 on RTX 4090, ~15-25 tok/s for 70B Q4 (with partial offload)
- **Strengths:** Runs anywhere, incredible quantization options, active development, low memory overhead
- **Weaknesses:** Single-request optimized (less throughput than batched engines), no tensor parallelism across GPUs natively (split layers instead)

### vLLM
- **What:** High-throughput Python serving engine with PagedAttention
- **Best for:** Multi-user serving, high concurrency, production API endpoints
- **Key features:**
  - PagedAttention: Near-zero memory waste for KV cache
  - Continuous batching: Dynamically batches incoming requests
  - Tensor parallelism across GPUs
  - GPTQ, AWQ, INT4/8, FP8 quantization
  - OpenAI-compatible API
  - Speculative decoding, chunked prefill
  - Prefix caching (great for shared system prompts)
  - Multi-LoRA serving (serve multiple fine-tunes from one base model)
- **Performance:** Highest throughput under concurrent load. 2-24x throughput vs naive HF serving
- **Strengths:** Battle-tested in production, excellent multi-user throughput, broad model support
- **Weaknesses:** Higher VRAM overhead than llama.cpp, primarily NVIDIA-focused (AMD/Intel improving), heavier setup

### Ollama
- **What:** User-friendly wrapper around llama.cpp with model management
- **Best for:** Getting started, development, single-user local use
- **Key features:**
  - One-line model downloads (`ollama pull llama3.1:70b`)
  - Automatic GPU detection and offloading
  - OpenAI-compatible API
  - Modelfile system for customization
  - Built-in model library
- **Performance:** Same as llama.cpp (it's the backend)
- **Strengths:** Incredibly easy to use, great DX, solid model library
- **Weaknesses:** Less control than raw llama.cpp, primarily single-user, limited batching. Overhead from abstraction layer.

### Text Generation Inference (TGI) — Hugging Face
- **What:** Rust-based serving engine by Hugging Face
- **Best for:** HuggingFace ecosystem integration, production serving
- **Key features:**
  - Continuous batching, tensor parallelism
  - Flash Attention 2
  - GPTQ, AWQ, EETQ, EXL2 quantization
  - Watermarking, grammar/guided generation
  - Optimized for A100/H100 but works on consumer GPUs
- **Performance:** Comparable to vLLM for throughput
- **Strengths:** Well-integrated with HF ecosystem, robust production features
- **Weaknesses:** More enterprise-focused, slightly less community momentum than vLLM

### SGLang
- **What:** High-performance serving framework from LMSYS (UC Berkeley)
- **Best for:** Complex multi-turn/branching inference, structured generation, highest raw performance
- **Key features:**
  - RadixAttention: Automatic KV cache reuse across requests with shared prefixes
  - Compressed finite state machine for structured output (faster constrained decoding)
  - Supports tensor/data/expert parallelism
  - OpenAI-compatible API
  - Native TPU support via JAX backend
  - Day-0 support for latest models (DeepSeek, Llama, etc.)
  - Diffusion model support (video/image generation)
- **Performance:** Often fastest engine in benchmarks, especially for structured output and multi-turn
- **Strengths:** Cutting-edge performance, excellent for agentic workloads, great structured generation
- **Weaknesses:** Newer/less mature ecosystem, documentation still catching up

### Quick Decision Matrix

| Scenario | Best Engine |
|----------|-------------|
| Single user, easy setup | **Ollama** |
| Single GPU, max compatibility | **llama.cpp** |
| Multi-user API server | **vLLM** or **SGLang** |
| Agentic/multi-turn workloads | **SGLang** |
| Structured output (JSON, etc.) | **SGLang** |
| HuggingFace ecosystem | **TGI** |
| Apple Silicon | **llama.cpp** / **Ollama** |
| Multi-GPU cluster | **vLLM** or **SGLang** |

---

## 2. Consumer GPU Hardware Guide

### RTX 3090 (24GB VRAM, ~$700-900 used)
- **Architecture:** Ampere, 10496 CUDA cores, 936 GB/s memory bandwidth
- **Sweet spot models:**
  - 7-13B at FP16 or 8-bit
  - 30-34B at Q4/Q5 quantization
  - 70B at Q3/Q4 with partial CPU offload (slow for prefill)
- **Power:** 350W TDP
- **Value proposition:** Best price/VRAM ratio on the used market. Workhorse for local inference.

### RTX 4090 (24GB VRAM, ~$1600-2000)
- **Architecture:** Ada Lovelace, 16384 CUDA cores, 1008 GB/s memory bandwidth
- **Sweet spot models:**
  - Same VRAM as 3090 so same model sizes
  - ~1.5-2x faster inference than 3090
  - FP8 support for better quality at same speed
- **Power:** 450W TDP
- **Value proposition:** Fastest consumer card. Worth it if you need speed. Not worth it purely for VRAM (same as 3090).

### RTX 3090 vs 4090 for Local AI

| Metric | RTX 3090 | RTX 4090 |
|--------|----------|----------|
| VRAM | 24GB | 24GB |
| Price (used) | ~$800 | ~$1800 |
| tok/s (7B Q4) | ~45 | ~80 |
| tok/s (70B Q4) | ~12 | ~22 |
| Power draw | 350W | 450W |
| $/VRAM GB | $33 | $75 |
| Best for | Budget builds | Speed-critical |

### Multi-GPU Considerations
- **2x RTX 3090:** 48GB total VRAM for ~$1600. Can run 70B at Q4-Q5 fully in VRAM. Better value than single 4090 for large models.
- **NVLink:** RTX 3090 supports NVLink (2-way). 4090 does NOT. For tensor parallelism, 3090 pairs are actually better.
- **PCIe bandwidth:** Without NVLink, multi-GPU communication goes through PCIe. x16 Gen4 = 32 GB/s. Adequate for pipeline parallelism, bottleneck for tensor parallelism.
- **Power/cooling:** 2x 3090 = 700W. Need adequate PSU (1000W+) and case airflow.
- **Motherboard:** Need x16/x16 PCIe slots (not x16/x4). Check motherboard PCIe lane allocation.

### Other Options
- **RTX 4060 Ti 16GB (~$400):** Budget option, fine for 7-13B models
- **AMD RX 7900 XTX (24GB, ~$800):** ROCm support improving but still behind CUDA
- **Apple M-series (unified memory):** M2/M3/M4 Ultra with 128-192GB unified memory can run massive models. Slower per-token but can fit models nothing else can. Great for development.
- **Used Tesla P40 (24GB, ~$150-200):** Extremely cheap VRAM. No FP16 tensor cores (slow). Good for experimentation on a budget.

---

## 3. What Models Run Well Locally

### Tier 1: Runs Great on Single 24GB GPU (Daily Driver)
| Model | Size | Quant | VRAM | Quality | Use Case |
|-------|------|-------|------|---------|----------|
| Qwen 2.5 72B | 72B | Q4_K_M | 42GB (2 GPU) | Excellent | General purpose, coding |
| Llama 3.3 70B | 70B | Q4_K_M | 42GB (2 GPU) | Excellent | General purpose |
| DeepSeek-R1-Distill-Qwen-32B | 32B | Q5_K_M | ~22GB | Very Good | Reasoning |
| Qwen 2.5 Coder 32B | 32B | Q5_K_M | ~22GB | Excellent | Code generation |
| Mistral Small 24B | 24B | Q5_K_M | ~18GB | Very Good | General, multilingual |
| Llama 3.1 8B | 8B | FP16 | ~16GB | Good | Fast general purpose |
| Phi-4 14B | 14B | Q6_K | ~12GB | Very Good | Reasoning, compact |
| Gemma 2 27B | 27B | Q4_K_M | ~17GB | Very Good | General purpose |

### Tier 2: Needs Multi-GPU or Partial Offload
| Model | Size | Quant | VRAM | Notes |
|-------|------|-------|------|-------|
| DeepSeek-V3 | 671B MoE | Q4 | ~200GB+ | Needs serious hardware or cloud |
| Llama 3.1 405B | 405B | Q2/Q3 | ~150GB+ | 6-8x 3090s or cloud |
| Mixtral 8x22B | 141B MoE | Q4 | ~80GB | 3-4x 24GB GPUs |

### Tier 3: Best Bang-for-Buck (Single GPU Sweet Spots)
- **For coding:** Qwen 2.5 Coder 32B Q4 — rivals GPT-4 on code tasks
- **For general chat:** Llama 3.3 70B Q3 (with some CPU offload) or Qwen 2.5 32B Q5
- **For reasoning:** DeepSeek-R1-Distill-Qwen-32B — chain-of-thought reasoning at 32B
- **For speed:** Llama 3.1 8B or Phi-4 14B — fastest with good quality
- **For RAG/embedding:** nomic-embed-text or BGE-large — tiny, fast, runs on anything

### Model Selection Rules of Thumb
1. **Quantization quality:** Q4_K_M is the sweet spot — minimal quality loss, ~4x size reduction
2. **IQ quants (llama.cpp):** IQ4_XS, IQ3_M give better quality than standard Q3/Q4 at same size
3. **MoE models:** Only activate a fraction of parameters per token. Mixtral 8x7B activates ~13B per token despite being 47B total. Great for VRAM-constrained setups but need full model in memory.
4. **Bigger model + lower quant > Smaller model + higher quant** (generally true down to ~Q3)

---

## 4. Local vs Cloud: When to Use Each

### Use Local When:
- **Privacy/compliance:** Data cannot leave your network (medical, legal, financial)
- **High volume, predictable load:** Amortized GPU cost beats per-token API pricing
- **Low latency needed:** No network round-trip (local = instant start)
- **Offline capability required:** Internet outages can't stop you
- **Development/testing:** Rapid iteration without API costs
- **Fine-tuning:** Need to run custom/fine-tuned models
- **Batch processing:** Large volumes of tokens to process (embeddings, classification)
- **You already own the hardware**

### Use Cloud/API When:
- **Frontier quality needed:** GPT-4o, Claude Opus, Gemini Ultra — no local equivalent
- **Spiky/unpredictable load:** Don't want idle hardware 95% of the time
- **Multimodal (advanced):** Video, complex image understanding still cloud-dominant
- **Speed to market:** Don't want to manage infrastructure
- **Scaling beyond your hardware:** Burst capacity
- **Context window:** 128K-1M+ context windows are hard to serve locally (massive KV cache VRAM)

### The Hybrid Sweet Spot (Most Cost-Effective)
Route based on task complexity:
- **Simple tasks (classification, extraction, summarization of short text):** Local 7-14B model
- **Medium tasks (coding, general Q&A, RAG):** Local 32-70B model
- **Hard tasks (complex reasoning, novel problems, creative writing):** Cloud frontier model
- **Embeddings:** Always local (trivial to run, high volume)
- **Image generation:** Local (SD/Flux) for volume, cloud for highest quality

---

## 5. Cost Comparison: Local vs API

### Local Infrastructure Cost (Single Workstation)

**Budget Build (1x RTX 3090):**
| Component | Cost |
|-----------|------|
| RTX 3090 (used) | $800 |
| CPU (Ryzen 7 / i7) | $300 |
| RAM (64GB DDR4) | $120 |
| Motherboard | $200 |
| PSU (850W) | $120 |
| NVMe SSD (2TB) | $120 |
| Case + cooling | $150 |
| **Total** | **~$1,810** |

**Power cost:** ~350W avg × 24/7 = 252 kWh/month × $0.30/kWh (NZ) = **~$76/month**

**Performance Build (2x RTX 3090):**
| Component | Cost |
|-----------|------|
| 2x RTX 3090 (used) | $1,600 |
| CPU (Ryzen 9 / i9) | $450 |
| RAM (128GB DDR4) | $250 |
| Motherboard (x16/x16) | $350 |
| PSU (1200W) | $200 |
| NVMe SSD (4TB) | $250 |
| Case + cooling | $200 |
| **Total** | **~$3,300** |

**Power cost:** ~700W avg × 24/7 = 504 kWh/month × $0.30/kWh = **~$151/month**

### API Cost Comparison

Assuming **1M tokens/day** (input+output combined, moderate usage):

| Provider | Model | Cost/1M tokens | Monthly (30M tok) |
|----------|-------|-----------------|-------------------|
| OpenAI | GPT-4o | ~$7.50 avg | ~$225 |
| OpenAI | GPT-4o-mini | ~$0.30 avg | ~$9 |
| Anthropic | Claude Sonnet 4 | ~$9 avg | ~$270 |
| Anthropic | Claude Haiku | ~$1.25 avg | ~$37.50 |
| Google | Gemini 2.0 Flash | ~$0.30 avg | ~$9 |
| DeepSeek | DeepSeek-V3 | ~$0.55 avg | ~$16.50 |
| Local 3090 | 32B Q5 | Electricity only | ~$76 |

### Break-Even Analysis

**Budget local build ($1,810 + $76/mo) vs GPT-4o ($225/mo):**
- Break-even: ~12 months (hardware paid off in savings)
- After that: saving ~$150/month
- **But:** Local 32B ≠ GPT-4o quality. Fairer comparison is vs GPT-4o-mini or Haiku

**Budget local build vs GPT-4o-mini ($9/mo) at 1M tok/day:**
- Local costs more due to electricity alone ($76 > $9)
- **Local only wins on cost at very high volume** (>5M tok/day vs cheap models)

**The real math:** Local wins when:
1. Volume exceeds ~3-5M tokens/day against cheap API models
2. You're replacing frontier model calls (GPT-4o/Claude) and quality is acceptable
3. Privacy requirements would otherwise force expensive enterprise API tiers
4. You need the hardware for other purposes too (gaming, rendering, training)

### Cost Summary
- **Low volume (<1M tok/day):** Cloud APIs are cheaper
- **Medium volume (1-5M tok/day):** Hybrid approach optimal
- **High volume (>5M tok/day):** Local wins handily, especially vs frontier models
- **Privacy-constrained:** Local wins regardless of volume (enterprise API pricing is brutal)

---

## 6. Hybrid Routing Strategies

### Architecture: Smart Router

```
                    ┌─────────────┐
  Requests ──────►  │ Smart Router │
                    └──────┬──────┘
                           │
              ┌────────────┼────────────┐
              ▼            ▼            ▼
        ┌──────────┐ ┌──────────┐ ┌──────────┐
        │ Local GPU │ │ Local GPU │ │ Cloud API │
        │  (fast)   │ │ (quality) │ │ (frontier)│
        │  8B model │ │ 32-70B   │ │ GPT-4o etc│
        └──────────┘ └──────────┘ └──────────┘
```

### Routing Strategies

**1. Complexity-Based Routing**
- Classify incoming request difficulty (can use the local 8B model to triage)
- Simple → local small model
- Medium → local large model  
- Hard → cloud frontier

**2. Latency-Based Routing**
- If local queue depth < threshold → route local
- If local is overloaded → overflow to cloud
- Guarantees SLA response times

**3. Cost-Budget Routing**
- Set daily/monthly cloud spend cap
- Use cloud for highest-value requests first
- Fall back to local when budget exhausted

**4. Task-Type Routing**
- Embeddings → always local
- Classification/extraction → local small
- Code generation → local 32B (Qwen Coder)
- Complex reasoning → cloud frontier
- Creative/long-form → cloud (better at nuance)

### Implementation Approaches

**LiteLLM (recommended for most):**
```python
# litellm supports routing across local + cloud with OpenAI-compatible API
import litellm

# Configure models
litellm.model_list = [
    {"model_name": "fast", "litellm_params": {"model": "ollama/llama3.1:8b", "api_base": "http://localhost:11434"}},
    {"model_name": "quality", "litellm_params": {"model": "ollama/qwen2.5:32b", "api_base": "http://localhost:11434"}},
    {"model_name": "frontier", "litellm_params": {"model": "gpt-4o"}},
]
```

**OpenRouter:** Can mix local endpoints with cloud APIs via single API.

**Custom Router:** Use an OpenAI-compatible proxy that inspects requests and routes based on rules. Many frameworks support this (LangChain, Semantic Kernel, custom FastAPI).

### Recommended Hybrid Setup
1. **Ollama or vLLM** running locally with 2-3 models (8B fast, 32B quality)
2. **LiteLLM proxy** or custom router in front
3. **Cloud API keys** as fallback (OpenAI, Anthropic, DeepSeek)
4. Route based on: task type > cost budget > latency requirements
5. Log everything — optimize routing rules based on actual usage patterns

---

## 7. Building a Local Inference Cluster

### Single Node (Start Here)
1. Build workstation with 1-2x RTX 3090/4090
2. Install Ubuntu 22.04/24.04 LTS + NVIDIA drivers + CUDA
3. Install your inference engine (Ollama for easy, vLLM for production)
4. Expose OpenAI-compatible API on local network
5. Done. This handles most individual/small team needs.

### Multi-Node Cluster

**When you need it:**
- Serving >10 concurrent users
- Running models too large for one machine
- Need redundancy/high availability

**Architecture:**
```
                    ┌──────────────┐
                    │ Load Balancer │  (nginx/HAProxy/Traefik)
                    └──────┬───────┘
                           │
            ┌──────────────┼──────────────┐
            ▼              ▼              ▼
      ┌───────────┐  ┌───────────┐  ┌───────────┐
      │  Node 1   │  │  Node 2   │  │  Node 3   │
      │ 2x 3090   │  │ 2x 3090   │  │ 2x 3090   │
      │ vLLM/32B  │  │ vLLM/32B  │  │ vLLM/8B   │
      └───────────┘  └───────────┘  └───────────┘
```

**Key components:**
1. **Nodes:** Standard workstations or rack servers with GPUs
2. **Networking:** 10GbE minimum between nodes. For tensor parallelism across nodes, InfiniBand is ideal but impractical for consumer setups.
3. **Load balancer:** Route requests based on model availability, queue depth
4. **Shared storage:** NFS or object storage for model files (avoid downloading models to every node)
5. **Monitoring:** Prometheus + Grafana for GPU utilization, queue depth, latency, throughput
6. **Orchestration:** Docker Compose for simple, Kubernetes for complex

**Software Stack:**
```bash
# Per node
docker run --gpus all -v /models:/models \
  vllm/vllm-openai:latest \
  --model /models/Qwen2.5-32B-Instruct-GPTQ-Int4 \
  --tensor-parallel-size 2 \
  --port 8000

# Load balancer (nginx example)
upstream vllm_cluster {
    least_conn;
    server node1:8000;
    server node2:8000;
    server node3:8000;
}
```

### Scaling Considerations
- **Start with one beefy node** — don't over-engineer early
- **Scale horizontally** by adding identical nodes behind load balancer
- **Model sharding across nodes** (tensor parallelism over network) is mostly impractical on consumer hardware — stick to fitting models within a single node's VRAM
- **Pipeline parallelism** (split model layers across nodes) works but adds latency from network hops
- **Ray Serve + vLLM** for more sophisticated multi-node orchestration
- **Consider used enterprise gear:** Dell R740 + 4x Tesla T4 (used, ~$2000) for cheaper per-GPU scaling

### Monitoring & Operations
- **GPU metrics:** `nvidia-smi`, DCGM exporter → Prometheus
- **Key metrics:** GPU utilization, VRAM usage, request queue depth, p50/p95 latency, tokens/sec
- **Auto-scaling:** Not really needed at small scale. Monitor utilization and add nodes manually when consistently >80%
- **Model management:** Keep a central model repository. Use symlinks or NFS mounts.

---

## 8. Practical Recommendations

### For a Solo Developer / Power User
1. **Hardware:** Single RTX 3090 ($800) or M-series Mac with 32GB+ unified memory
2. **Software:** Ollama for daily use, llama.cpp for experimentation
3. **Models:** Qwen 2.5 32B (general), Qwen Coder 32B (code), Llama 3.1 8B (fast)
4. **Cloud fallback:** DeepSeek API (cheapest frontier-ish quality) + Claude for hard problems
5. **Total cost:** ~$1,800 hardware + ~$50-100/mo (power + occasional cloud)

### For a Small Team (5-10 people)
1. **Hardware:** 2x RTX 3090 workstation
2. **Software:** vLLM or SGLang with OpenAI-compatible API
3. **Models:** 32B quality model + 8B fast model
4. **Router:** LiteLLM proxy with cloud fallback
5. **Total cost:** ~$3,300 hardware + ~$150-250/mo (power + cloud overflow)

### For Production / Startup
1. **Hardware:** 2-3 nodes with 2x 3090 each, or rent cloud GPUs (Lambda, RunPod, vast.ai)
2. **Software:** vLLM or SGLang + Kubernetes + monitoring stack
3. **Models:** Depends on use case — likely fine-tuned models
4. **Hybrid:** Local for base load, cloud burst for peaks
5. **Consider:** At scale, renting A100/H100 cloud GPUs may be more cost-effective than consumer hardware (better throughput per dollar at high utilization)

### Key Takeaways
1. **llama.cpp/Ollama for personal use, vLLM/SGLang for serving** — don't overthink engine choice
2. **RTX 3090 is the value king** — 24GB VRAM at half the price of 4090
3. **32B models are the sweet spot** for single-GPU quality-to-speed ratio
4. **Hybrid routing is almost always optimal** — use local for volume, cloud for quality
5. **Start simple, scale incrementally** — one GPU → two GPUs → second node → cluster
6. **The gap between local and cloud is narrowing fast** — today's "good enough" local models will be tomorrow's embarrassingly good local models

---

## 9. Resources & Links

- [llama.cpp](https://github.com/ggml-org/llama.cpp) — C/C++ inference, GGUF format
- [vLLM](https://docs.vllm.ai/) — High-throughput serving with PagedAttention
- [SGLang](https://github.com/sgl-project/sglang) — High-performance serving with RadixAttention
- [Ollama](https://ollama.ai/) — User-friendly local model runner
- [TGI](https://huggingface.co/docs/text-generation-inference) — HuggingFace serving engine
- [LiteLLM](https://github.com/BerriAI/litellm) — Unified API proxy for routing
- [r/LocalLLaMA](https://reddit.com/r/LocalLLaMA) — Community knowledge base
- [Open LLM Leaderboard](https://huggingface.co/spaces/open-llm-leaderboard/open_llm_leaderboard) — Model benchmarks
- [vast.ai](https://vast.ai/) / [RunPod](https://runpod.io/) — Cheap cloud GPU rental
