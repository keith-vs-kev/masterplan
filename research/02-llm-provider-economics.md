# LLM Provider Economics & Optimization

> **Research Date:** 2026-02-08
> **Status:** Living document — prices change frequently, verify before committing spend

---

## Executive Summary

The LLM landscape has matured into distinct tiers: **frontier proprietary** (Anthropic, OpenAI, Google), **speed-optimized inference** (Groq, Cerebras), **open-model hosts** (Together, Fireworks), **cloud hyperscalers** (AWS Bedrock, Azure), and **local inference**. The optimal strategy is **multi-provider routing** — matching task complexity to the cheapest capable model.

**Key insight:** For most tasks, you're overpaying by 10-50x if you default to frontier models. A well-designed routing layer can cut costs 80%+ while maintaining quality where it matters.

---

## Provider Deep Dive

### 1. Anthropic (Claude)

**API Direct: https://console.anthropic.com**

| Model | Input $/MTok | Output $/MTok | Context | Notes |
|-------|-------------|--------------|---------|-------|
| **Opus 4.6** | $5.00 | $25.00 | 200K | Best for agents/coding. >200K: $10/$37.50 |
| **Sonnet 4.5** | $3.00 | $15.00 | 200K | Best balance. >200K: $6/$22.50 |
| **Haiku 4.5** | $1.00 | $5.00 | 200K | Fast & cheap |
| Haiku 3 (legacy) | $0.25 | $1.25 | 200K | Cheapest Claude |

**Prompt Caching:** Huge savings — cached reads are 90% cheaper ($0.50/MTok for Opus input vs $5.00). 5-min TTL default.

**Batch API:** 50% off input and output.

**Rate Limits:** Tier-based. Free tier very limited. Tier 1 ($5 deposit): ~50K tokens/min. Scales up with spend.

**Free Tier:** Limited via claude.ai (web). No free API tier.

**Speed:** ~50-80 tok/s for Sonnet. Not the fastest.

**Strengths:** Best at nuanced reasoning, long-form writing, instruction following, coding. Extended thinking mode.

---

### 2. OpenAI

**API: https://platform.openai.com**

| Model | Input $/MTok | Output $/MTok | Context | Notes |
|-------|-------------|--------------|---------|-------|
| **GPT-5.2** | $1.75 | $14.00 | 128K? | Flagship. Cached: $0.175 input |
| **GPT-5.2 Pro** | $21.00 | $168.00 | 128K? | Maximum intelligence |
| **GPT-5 mini** | $0.25 | $2.00 | 128K? | Great value. Cached: $0.025 input |
| GPT-4.1 | ~$2.00 | ~$8.00 | 128K | Previous gen, fine-tunable |
| GPT-4.1 mini | ~$0.40 | ~$1.60 | 128K | Fine-tunable |
| GPT-4.1 nano | ~$0.10 | ~$0.40 | 128K | Cheapest OpenAI |
| **gpt-oss-120B** | $0.15 | $0.60 | 128K | Open-source, available on Groq/Together/Fireworks |
| **gpt-oss-20B** | $0.075 | $0.30 | 128K | Tiny, ultra-cheap |

**Batch API:** 50% off.

**Free Tier:** Free tier on ChatGPT. API has $5 free credits for new accounts (historically).

**Speed:** Moderate. ~60-100 tok/s depending on model.

**Strengths:** Broadest ecosystem, tool use, image generation (GPT-image), realtime API, web search built-in.

**Note:** gpt-oss models are open-weight and available much cheaper on inference providers.

---

### 3. Google (Gemini)

**API: https://ai.google.dev (free) / Vertex AI (paid)**

| Model | Input $/MTok | Output $/MTok | Context | Notes |
|-------|-------------|--------------|---------|-------|
| **Gemini 3 Pro** (preview) | $2.00 | $12.00 | 200K+ | Latest frontier |
| **Gemini 3 Flash** (preview) | $0.50 | $3.00 | 200K+ | Fast |
| **Gemini 2.5 Pro** | $1.25 | $10.00 | 1M | Excellent value |
| **Gemini 2.5 Flash** | $0.30 | $2.50 | 1M | Great balance |
| **Gemini 2.5 Flash Lite** | $0.10 | $0.40 | 1M | Ultra-cheap |

**Prompt Caching:** 90% discount on cached input.

**Batch/Flex:** 50% off standard pricing.

**Free Tier:** **EXTREMELY GENEROUS** via AI Studio — 1,500 RPD for Gemini 2.5 Flash, rate-limited but usable. Best free offering in the market.

**Speed:** Flash models are very fast, ~100+ tok/s.

**Context Window:** Up to **1M tokens** — largest in the industry.

**Strengths:** Multimodal (native audio, video, image), massive context, competitive pricing, grounding with Google Search.

---

### 4. Groq

**API: https://console.groq.com**

| Model | Input $/MTok | Output $/MTok | Speed (tok/s) | Context |
|-------|-------------|--------------|---------------|---------|
| **Llama 3.1 8B** | $0.05 | $0.08 | 840 | 128K |
| **gpt-oss-20B** | $0.075 | $0.30 | 1,000 | 128K |
| **Llama 4 Scout** | $0.11 | $0.34 | 594 | 128K |
| **Qwen3 32B** | $0.29 | $0.59 | 662 | 131K |
| **Llama 3.3 70B** | $0.59 | $0.79 | 394 | 128K |
| **Llama 4 Maverick** | $0.20 | $0.60 | 562 | 128K |
| **gpt-oss-120B** | $0.15 | $0.60 | 500 | 128K |
| **Kimi K2** | $1.00 | $3.00 | 200 | 256K |

**Free Tier:** Yes — generous free tier with rate limits. Developer tier from $10.

**Speed:** **FASTEST inference in the market.** Custom LPU hardware. 500-1000+ tok/s.

**Rate Limits:** Free tier limited. Developer tier: 10x higher.

**Strengths:** Speed. Period. If latency matters, Groq wins. Open-source models only.

---

### 5. Cerebras

**API: https://cloud.cerebras.ai**

| Model | Pricing | Speed | Notes |
|-------|---------|-------|-------|
| Various open models | Not publicly listed per-model | ~20x faster than OpenAI/Anthropic | Wafer-scale chips |

**Free Tier:** Yes — access to all models, community support.

**Developer Tier:** From $10. 10x higher rate limits.

**Cerebras Code:** $50/mo (24M tok/day) or $200/mo (120M tok/day) — great for coding.

**Strengths:** Extreme speed rivaling Groq. Custom silicon. Good for latency-sensitive apps.

---

### 6. Together AI

**API: https://api.together.xyz**

| Model | Input $/MTok | Output $/MTok | Notes |
|-------|-------------|--------------|-------|
| **Gemma 3n E4B** | $0.02 | $0.04 | Cheapest |
| **gpt-oss-20B** | $0.05 | $0.20 | |
| **Llama 3.1 8B** | $0.18 | $0.18 | |
| **Llama 4 Scout** | $0.18 | $0.59 | |
| **Llama 4 Maverick** | $0.27 | $0.85 | |
| **Qwen3 235B** | $0.20 | $0.60 | Great value for large model |
| **DeepSeek-V3.1** | $0.60 | $1.70 | |
| **DeepSeek-R1-0528** | $3.00 | $7.00 | Reasoning model |
| **Llama 3.1 405B** | $3.50 | $3.50 | Largest open model |

**Free Tier:** $5 free credits for new accounts.

**Batch API:** Significant discounts.

**Strengths:** Widest selection of open models. Fine-tuning support. Dedicated endpoints available. Image/video generation too.

---

### 7. Fireworks AI

**API: https://fireworks.ai**

| Model Category | Input $/MTok | Output $/MTok |
|---------------|-------------|--------------|
| **<4B params** | $0.10 | $0.10 |
| **4-16B params** | $0.20 | $0.20 |
| **>16B params** | $0.90 | $0.90 |
| **MoE 0-56B** | $0.50 | $0.50 |
| **DeepSeek V3** | $0.56 | $1.68 |
| **DeepSeek R1** | $1.35 | $5.40 |
| **Qwen3 235B** | $0.22 | $0.88 |
| **gpt-oss-120B** | $0.15 | $0.60 |
| **gpt-oss-20B** | $0.07 | $0.30 |
| **Kimi K2** | $0.60 | $2.50 |

**Cached Input:** 50% off for all models.

**Batch:** 50% off.

**On-Demand GPUs:** H100 @ $4/hr, H200 @ $6/hr, B200 @ $9/hr.

**Free Tier:** Free tier available with rate limits.

**Strengths:** Simple tiered pricing, good speed, fine-tuning, on-demand GPU deployment.

---

### 8. AWS Bedrock

**Models Available:** Claude (Anthropic), Llama, Mistral, Cohere, Amazon Titan, Stability AI

**Pricing:** Generally same as provider direct pricing + small AWS margin (0-20% markup).

**Strengths:**
- Single billing through AWS
- VPC/PrivateLink for security
- Provisioned throughput for predictable performance
- Guardrails and governance built-in
- Cross-region inference

**Weaknesses:** Slightly higher prices, slower model availability (weeks behind direct API).

**Best For:** Enterprise with existing AWS footprint, compliance requirements.

---

### 9. Azure OpenAI

**Models Available:** OpenAI models (GPT-5.x, GPT-4.1, o-series), Mistral, Llama, Phi

**Pricing:** Same as OpenAI direct for most models. Provisioned Throughput Units (PTUs) for committed use.

**Strengths:**
- Enterprise security (VNET, managed identity)
- Content filtering built-in
- Global deployment options
- PTU-based pricing for predictable costs

**Weaknesses:** Model availability lags OpenAI direct. More complex setup.

**Best For:** Enterprise with Azure footprint, Microsoft 365 integration needs.

---

### 10. Mistral

**API: https://console.mistral.ai**

| Model | Approx Input $/MTok | Approx Output $/MTok | Notes |
|-------|---------------------|---------------------|-------|
| Magistral Medium | ~$2.00 | ~$6.00 | Reasoning model |
| Magistral Small | ~$0.50 | ~$1.50 | Efficient reasoning |
| Mistral Small 3.1 | ~$0.10 | ~$0.30 | Fast, cheap |
| Codestral | ~$0.30 | ~$0.90 | Code-focused |
| Ministral 3B | ~$0.04 | ~$0.04 | Tiny/edge |
| Ministral 8B | ~$0.10 | ~$0.10 | Small/edge |

**Free Tier:** Free Le Chat access. API has free tier with limits.

**Strengths:** Strong European option (EU data residency), good code models, efficient small models, open-weight options available on other providers.

---

### 11. Cohere

**Pricing:** Enterprise-only model now. No public per-token pricing.

**Products:** North (AI platform), Compass (search), Command R+ (model)

**Strengths:** Enterprise search/RAG, embeddings, reranking. Strong at retrieval-augmented generation.

**Best For:** Enterprise customers needing managed AI platform with search capabilities.

**Note:** Less relevant for individual/startup use due to enterprise-only pricing.

---

### 12. Local Inference

**Options:**
- **Ollama** — easiest setup, Mac/Linux/Windows
- **llama.cpp** — raw performance, C++ inference
- **vLLM** — production serving, high throughput
- **TGI (HuggingFace)** — Docker-based serving
- **LocalAI** — OpenAI-compatible local server

**Cost Model:** Hardware only (electricity + amortized GPU cost)

| GPU | ~Cost | Models it can run | Throughput |
|-----|-------|-------------------|-----------|
| RTX 4090 (24GB) | $1,600 | Up to 13B (full), 70B (quantized) | 30-80 tok/s |
| RTX 3090 (24GB) | $800 used | Same as 4090, slower | 20-50 tok/s |
| Mac M4 Max (128GB) | $4,000 | Up to 70B+ (full), 405B (quantized) | 15-40 tok/s |
| 2x RTX 4090 | $3,200 | Up to 70B (full) | 50-100 tok/s |
| H100 (80GB) | $25K-$30K | Everything | 100-300 tok/s |

**Break-even Analysis:**
- At $0.50/MTok average API spend and 100M tok/month = $50/mo
- RTX 4090 pays for itself at ~300M+ tok/month sustained
- Only worth it for **very high volume** or **privacy requirements**

**Best Models for Local:**
- **Llama 3.1 8B** — great quality/size ratio
- **Qwen3 32B** — excellent reasoning for size
- **Mistral Small 3.1** — efficient, good coding
- **DeepSeek V3** — needs beefy hardware but exceptional
- **Phi-4** (14B) — Microsoft's efficient model

---

## Decision Matrix: Task → Provider Routing

### By Task Type

| Task | Best Provider | Model | Cost ($/MTok out) | Why |
|------|--------------|-------|-------------------|-----|
| **Quick classification/extraction** | Groq | Llama 3.1 8B | $0.08 | Speed + cost |
| **Simple Q&A / chat** | Google | Gemini 2.5 Flash Lite | $0.40 | Cheap, free tier |
| **Code generation** | Anthropic | Sonnet 4.5 | $15.00 | Best quality |
| **Code generation (budget)** | Fireworks | gpt-oss-120B | $0.60 | 25x cheaper |
| **Long document analysis** | Google | Gemini 2.5 Pro | $10.00 | 1M context |
| **Complex reasoning** | Anthropic | Opus 4.6 | $25.00 | Best reasoning |
| **Complex reasoning (budget)** | Together | DeepSeek-R1 | $7.00 | 3.5x cheaper |
| **Bulk processing** | Any + Batch | Haiku 4.5 batch | $2.50 | 50% off |
| **Real-time / latency-critical** | Groq | Any | Varies | 500-1000 tok/s |
| **Embeddings** | Fireworks | Small embed | $0.008 | Cheapest |
| **Image understanding** | Google | Gemini 2.5 Flash | $2.50 | Native multimodal |
| **Privacy-sensitive** | Local | Llama 3.1 8B | $0 (hw cost) | No data leaves |
| **Free/hobby projects** | Google | AI Studio free | $0 | 1500 RPD free |

### By Budget Tier

#### 🆓 Free Tier ($0/month)
- **Google AI Studio:** Gemini 2.5 Flash — 1,500 req/day, rate-limited
- **Groq Free:** Various open models, rate-limited
- **Cerebras Free:** All models, rate-limited
- **Mistral Free:** Le Chat access

#### 💰 Low Budget ($10-50/month)
- **Groq Developer** ($10): Fast open-model inference
- **Cerebras Developer** ($10): Ultra-fast inference
- **Together AI**: Pay-per-token with $5 free credits
- Route: Gemini Flash for general, Groq for speed, Together for variety

#### 💎 Production ($100-1000/month)
- **Anthropic API**: Sonnet 4.5 for quality tasks
- **OpenAI API**: GPT-5 mini for general tasks
- **Google Vertex**: Gemini 2.5 Pro for long-context
- Route: Tier tasks — Haiku/Flash for simple, Sonnet/Pro for complex

#### 🏢 Enterprise ($1000+/month)
- **AWS Bedrock** or **Azure OpenAI** for governance
- **Anthropic/OpenAI direct** for latest models
- **Dedicated endpoints** (Together/Fireworks) for throughput
- Batch processing for 50% savings on async workloads

---

## Cost Optimization Strategies

### 1. Prompt Caching (saves 50-90%)
All major providers support it. For repeated system prompts or context:
- Anthropic: 90% savings on cached reads
- OpenAI: 90% savings
- Google: 90% savings

### 2. Batch Processing (saves 50%)
Available on Anthropic, OpenAI, Google, Fireworks, Together. Use for any non-real-time workload.

### 3. Model Routing (saves 60-90%)
Don't use Opus for "what's the weather?" Use a tiered approach:
```
Simple tasks → Haiku 4.5 / Gemini Flash Lite / Llama 8B ($0.08-0.40/MTok)
Medium tasks → Sonnet 4.5 / Gemini 2.5 Flash / GPT-5 mini ($2-5/MTok)
Complex tasks → Opus 4.6 / Gemini 2.5 Pro / GPT-5.2 ($10-25/MTok)
```

### 4. Output Token Optimization
Output tokens cost 3-5x more than input. Reduce output verbosity:
- Use structured output (JSON) — less fluff
- Set max_tokens appropriately
- Ask for concise responses

### 5. Context Window Management
- Summarize conversation history instead of passing full transcripts
- Use RAG instead of stuffing everything in context
- >200K tokens triggers 2x pricing on Anthropic and Google

### 6. Provider Arbitrage
Same model, different prices:
| Model | Provider A | Provider B | Savings |
|-------|-----------|-----------|---------|
| gpt-oss-120B | Groq $0.15/$0.60 | Fireworks $0.15/$0.60 | Same — compare speed |
| Llama 3.3 70B | Groq $0.59/$0.79 | Together $0.88/$0.88 | Groq 25% cheaper |
| DeepSeek V3 | Fireworks $0.56/$1.68 | Together $0.60/$1.70 | Similar |

### 7. OpenRouter as a Meta-Router
[OpenRouter](https://openrouter.ai) provides a single API across 100+ models/providers. Adds a small margin but simplifies multi-provider routing. Good for prototyping, less ideal for high-volume production.

---

## Speed Comparison (approx. output tok/s)

| Provider | Typical Speed | Notes |
|----------|--------------|-------|
| **Groq** | 500-1,000 | Fastest. Custom LPU silicon |
| **Cerebras** | 400-800 | ~20x faster than GPU inference |
| **Fireworks** | 100-200 | Good, GPU-based |
| **Together** | 80-150 | Good, GPU-based |
| **Google** | 80-150 | Flash models faster |
| **OpenAI** | 60-100 | Moderate |
| **Anthropic** | 50-80 | Moderate |
| **Local (4090)** | 30-80 | Depends on model size |

---

## Recommended Architecture: Smart Router

```
┌─────────────────────────────────────────┐
│              Request Classifier          │
│  (tiny local model or rules-based)       │
└──────────┬──────────┬──────────┬────────┘
           │          │          │
      Simple     Medium      Complex
           │          │          │
    ┌──────▼──┐ ┌────▼────┐ ┌──▼──────┐
    │ Groq    │ │ Gemini  │ │Anthropic│
    │ Llama8B │ │ 2.5Flash│ │Opus 4.6 │
    │$0.08/MT │ │$2.50/MT │ │$25/MT   │
    └─────────┘ └─────────┘ └─────────┘
```

**Classification heuristic:**
- **Simple** (<100 tokens expected output, factual, classification, extraction): Cheapest/fastest
- **Medium** (general chat, summarization, moderate code): Mid-tier
- **Complex** (multi-step reasoning, novel code architecture, creative writing): Frontier

**Estimated savings vs. always using frontier:** 70-85%

---

## Key Takeaways

1. **Google's free tier is unbeatable** for hobby/dev — use Gemini 2.5 Flash via AI Studio
2. **Groq/Cerebras win on speed** — use for latency-sensitive or high-volume simple tasks
3. **Anthropic/OpenAI win on quality** — use for tasks where accuracy matters most
4. **Open-source models via Together/Fireworks** offer 10-50x cost savings over proprietary
5. **Prompt caching + batch processing** are the two easiest cost wins (50-90% savings)
6. **gpt-oss-120B** is the best value open model right now — $0.15/$0.60 across multiple providers
7. **Always check if the task actually needs a frontier model** — most don't

---

## Appendix: Quick Reference Card

### Cheapest per tier (output $/MTok)

| Tier | Model | Provider | Output $/MTok |
|------|-------|----------|---------------|
| Ultra-cheap | Gemma 3n E4B | Together | $0.04 |
| Budget | Llama 3.1 8B | Groq | $0.08 |
| Value | Gemini 2.5 Flash Lite | Google | $0.40 |
| Mid-range | gpt-oss-120B | Groq/Fireworks | $0.60 |
| Quality | GPT-5 mini | OpenAI | $2.00 |
| Premium | Gemini 2.5 Flash | Google | $2.50 |
| Frontier | Sonnet 4.5 | Anthropic | $15.00 |
| Maximum | Opus 4.6 | Anthropic | $25.00 |
| Overkill | GPT-5.2 Pro | OpenAI | $168.00 |
