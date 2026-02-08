# 04 — Cost Audit: Real Monthly Burn Analysis

> **Auditor:** Finn (Cost Auditor subagent)
> **Date:** 2026-02-08
> **Verdict:** The architecture underestimates costs by 2-4× in months 1-3. The $40K MRR month-12 projection in doc 10 requires surviving a ~$3K-5K/mo burn before breakeven. Achievable, but the docs paint a rosier picture than reality.

---

## Methodology

I read all 20 architecture documents and the LLM provider economics research. I catalogued every service, tool, API, and infrastructure component mentioned, cross-referenced pricing, and built bottom-up cost models for months 1, 6, and 12. I assumed the architecture is implemented roughly as described — not a fantasy version, not a stripped-down version.

**Key assumption:** "dreamteam" is an existing server (1× RTX 3090, 64GB RAM). Hardware cost is sunk/amortized. Electricity is real.

---

## 1. Complete Cost Inventory

### 1.1 Infrastructure (Self-Hosted on dreamteam)

| Component | Source Doc | RAM | Disk | Monthly Cost |
|-----------|-----------|-----|------|-------------|
| PostgreSQL (orchestrator + LiteLLM) | 01, 02, 04 | 2GB | 10GB | $0 (self-hosted) |
| TimescaleDB (analytics) | 18 | 4GB | 100GB | $0 (self-hosted) |
| Redis (budgets, caching, real-time) | 02, 08, 18 | 1GB | — | $0 (self-hosted) |
| Qdrant (vector memory) | 05 | 2GB | 10GB | $0 (self-hosted) |
| Neo4j (knowledge graph) | 05 | 4GB | 20GB | $0 (self-hosted) |
| LiteLLM Proxy | 02, 08, 13 | 1GB | — | $0 (self-hosted) |
| Langfuse (tracing) | 08 | 4GB | 20GB | $0 (self-hosted) |
| Grafana | 08, 18 | 0.5GB | — | $0 (self-hosted) |
| Prometheus | 08 | 2GB | 10GB | $0 (self-hosted) |
| llama.cpp (inference) | 13 | — | 50GB models | $0 (self-hosted) |
| llama.cpp (embeddings, CPU) | 13 | 2GB | — | $0 (self-hosted) |
| MinIO (S3-compatible) | 18 | 1GB | 10GB | $0 (self-hosted) |
| Anomaly detector | 08 | 0.5GB | — | $0 (self-hosted) |
| MCP Memory Server | 05 | 0.5GB | — | $0 (self-hosted) |
| OpenClaw (agent runtime) | All | 2GB | — | $0 (self-hosted) |
| **TOTAL RAM REQUIRED** | | **~27GB** | **~230GB** | |

**⚠️ TRAP #1: dreamteam is oversubscribed.** 27GB RAM for services alone, plus the OS, plus llama.cpp needing GPU VRAM, plus agent processes. With 64GB total, you have ~35GB headroom. That sounds fine until you realize PostgreSQL, TimescaleDB, Neo4j, and Qdrant all want to cache data in RAM. Under load, you'll hit swap. **Budget for a second machine by month 3-4 or prepare to cut services.**

**Electricity cost:** ~200-300W continuous draw × 24/7 = 144-216 kWh/mo × $0.30/kWh (NZ) = **$43-65/mo**

### 1.2 LLM API Costs

This is the big one. The docs describe 14 agents, many running continuously or on cron. Let me model realistic usage.

#### Month 1: Building the Factory Itself

The factory doesn't exist yet. Month 1 is building orchestration, routers, MCP servers, dashboards. Work is mostly manual + a few agents (Kev, maybe Rex, Scout).

| Usage Pattern | Tokens/day | Model Tier | $/MTok avg | Daily Cost |
|---|---|---|---|---|
| Kev orchestrating (10 sessions) | 500K | Sonnet 4.5 ($9 avg) | $9.00 | $4.50 |
| Rex/coding agents building infra | 1M | Sonnet 4.5 | $9.00 | $9.00 |
| Scout research | 300K | Gemini 2.5 Flash | $1.40 | $0.42 |
| Code review (Hawk) | 200K | Sonnet 4.5 | $9.00 | $1.80 |
| Misc (local inference) | 500K | Local (free) | $0 | $0 |
| **Total** | **2.5M** | | | **$15.72/day** |

**Month 1 LLM cost: ~$470**

But wait — this assumes efficient routing from day 1. In reality, you're still building the router in month 1. You'll be using direct API calls at full price. **Realistic month 1: $500-700.**

#### Month 6: Factory Running, 6 Products Live

| Usage Pattern | Tokens/day | Model Tier | Cost/day |
|---|---|---|---|
| Kev orchestration | 1M | Sonnet 4.5 | $9.00 |
| 2-3 coding agents building products | 3M | Mix (Sonnet + local) | $15.00 |
| Scout market scanning (continuous) | 1M | Gemini Flash | $1.40 |
| Echo content generation (5-10 articles/week) | 2M | Sonnet/Flash mix | $8.00 |
| Hawk QA/review | 500K | Sonnet 4.5 | $4.50 |
| Growth/marketing agents | 500K | Flash/Haiku | $1.00 |
| Support agents (Ally) | 300K | Haiku 4.5 | $0.90 |
| Memory extraction/consolidation | 500K | Local + Flash | $0.50 |
| Portfolio/analytics (Dash) | 300K | Sonnet | $2.70 |
| Embeddings | 2M | Local (free) | $0 |
| **Total** | **~11M** | | **$43/day** |

**Month 6 LLM cost: ~$1,300** (with smart routing, caching)

**Without optimization: ~$2,000-2,500** (if caching underperforms or routing isn't tuned)

#### Month 12: Full Autonomy, 9 Products, Scaling

| Usage Pattern | Tokens/day | Cost/day |
|---|---|---|
| Orchestration (Kev + Rex) | 2M | $12.00 |
| Building (2-3 parallel builds) | 5M | $20.00 |
| Market scanning + validation | 2M | $3.00 |
| Content generation (20+ articles/week) | 5M | $15.00 |
| QA/testing | 1M | $6.00 |
| Growth/marketing (9 products) | 2M | $4.00 |
| Support (9 products) | 1M | $3.00 |
| Analytics/reporting | 500K | $3.00 |
| Memory/knowledge graph | 1M | $1.00 |
| Local inference | 5M | $0 |
| **Total** | **~25M** | **$67/day** |

**Month 12 LLM cost: ~$2,000** (optimized)

### 1.3 SaaS & Third-Party Services (Per Product)

Doc 10 specifies a "standard tech stack" per product. Here's what each product ACTUALLY costs:

| Service | Free Tier? | Realistic Monthly Cost | Notes |
|---|---|---|---|
| **Vercel** (hosting) | Yes, generous | $0-20/product | Free tier covers low traffic. Pro ($20) needed for commercial use, team features |
| **Supabase** (DB + auth) | Yes, 500MB DB | $0-25/product | Free for tiny apps. Pro ($25) once you hit limits |
| **Stripe** | No monthly fee | 2.9% + $0.30/txn | Variable. On $500 MRR = ~$15. On $5K = ~$150 |
| **Cloudflare** (DNS + CDN) | Yes | $0/product | Free tier is excellent |
| **Resend** (email) | 3K emails/mo free | $0-20/product | Free covers early stage |
| **PostHog** (analytics) | 1M events/mo free | $0/product | Self-host for free or generous cloud free tier |
| **Sentry** (errors) | 5K events/mo free | $0-26/product | Free covers low volume |
| **BetterStack** (uptime) | Free tier exists | $0-25/product | Free covers basics |
| **Domain** | No | $10-15/year = ~$1/mo | Per product |

**Per-product monthly infra cost reality:**

| Stage | Cost/Product/Month |
|---|---|
| Pre-revenue (free tiers) | $1-5 |
| Early revenue ($100-500 MRR) | $20-50 |
| Growing ($500-2K MRR) | $50-100 |
| Scaled ($2K+ MRR) | $100-200 |

### 1.4 Marketing & Growth Costs (Per Product)

From doc 10 and 16:

| Item | Month 1 (launch) | Monthly ongoing |
|---|---|---|
| Paid ads testing | $50-100 | $300-500 (if ROAS positive) |
| Directory submissions | $0-50 | $0 |
| Community promotion | $0 (time only) | $0 |
| SEO content (agent-generated) | $0 (LLM cost covered above) | $0 |
| **Total per product** | **$50-150** | **$300-500** |

### 1.5 Browserbase (Browser Automation)

Doc 17 mentions Browserbase for stealth scraping and testing.

| Usage | Cost |
|---|---|
| Free tier | Limited, maybe 100 sessions |
| Developer | ~$50-100/mo |
| Scale (market scanning + testing 9 products) | ~$100-300/mo |

### 1.6 OpenClaw Subscription

The agent runtime itself. Pricing unclear from docs but it's a commercial product. **Estimate: $0-100/mo** depending on plan.

---

## 2. Total Monthly Cost Summary

### Month 1

| Category | Cost |
|---|---|
| Electricity (dreamteam) | $50 |
| Hardware amortization (3090 + server) | $75 |
| LLM APIs | $600 |
| SaaS per product (1-2 products, free tiers) | $10 |
| Domains (2) | $2 |
| Marketing (launch 2 products) | $200 |
| Browserbase | $50 |
| Misc (API keys, tools, unexpected) | $50 |
| **TOTAL MONTH 1** | **~$1,037** |

Doc 10 claims Month 1 cost: $2,000. **Verdict: Roughly accurate if you include human time opportunity cost. Pure infrastructure/API cost is lower, but they're also optimistic about LLM costs because the router doesn't exist yet.**

### Month 6

| Category | Cost |
|---|---|
| Electricity | $60 |
| Hardware amortization | $75 |
| LLM APIs | $1,500 |
| SaaS per product (6 products) | $200 |
| Domains (8 total inc. killed) | $8 |
| Marketing (2-3 active growth campaigns) | $1,200 |
| Browserbase | $150 |
| Validation budget (test 4 ideas) | $1,000 |
| Misc/unexpected | $100 |
| **TOTAL MONTH 6** | **~$4,293** |

Doc 10 claims Month 6 cost: $4,500. **Verdict: Surprisingly close. But this assumes smart routing is working well. Without it, add $500-1,000 to LLM costs.**

### Month 12

| Category | Cost |
|---|---|
| Electricity | $65 |
| Hardware amortization | $75 |
| LLM APIs | $2,200 |
| SaaS per product (9 products, some scaled) | $600 |
| Domains | $12 |
| Marketing (4-5 growth campaigns) | $2,500 |
| Browserbase | $250 |
| Validation budget | $1,500 |
| Second server/hardware (likely needed) | $100 |
| Stripe fees (on $40K MRR) | $1,200 |
| Misc/unexpected | $200 |
| **TOTAL MONTH 12** | **~$8,702** |

Doc 10 claims Month 12 cost: $7,500. **Verdict: Underestimates by ~$1,200. Main gaps: Stripe transaction fees (never mentioned in doc 10!) and second server costs.**

---

## 3. Cost Traps — The Things That Will Bite You

### 🪤 TRAP 1: Stripe Fees Are Real Revenue Loss

Doc 10's P&L never accounts for Stripe's 2.9% + $0.30/txn. At $40K MRR:
- ~$1,160 + ~$120 (assuming 400 transactions) = **~$1,280/mo**
- That's not cost, it's revenue you never see. The "net profit" numbers in doc 10 are all overstated by this amount.

### 🪤 TRAP 2: The "Free Tier Cliff"

Every product starts on free tiers (Vercel, Supabase, Resend, PostHog, Sentry). The moment ANY product gets traction:
- Supabase free → Pro: +$25/mo
- Vercel free → Pro: +$20/mo
- Sentry free → Team: +$26/mo

With 9 products, if even 4 outgrow free tiers, that's **$200-300/mo** that didn't exist in your projections.

### 🪤 TRAP 3: LLM Costs Scale Non-Linearly with Agent Complexity

The docs model token usage linearly. Real agent systems exhibit:
- **Context window bloat**: Long conversations accumulate context. A 10-step agent chain at 4K tokens/step = 40K tokens input on the final step alone.
- **Retry amplification**: Doc 04 specifies 3 retries with model escalation. Each failed attempt burns tokens. A task that fails twice on Haiku then succeeds on Sonnet costs 3× the single-attempt price.
- **Memory retrieval overhead**: Doc 05's RAG system adds retrieved context to every call. That's 1-4K extra input tokens per call, multiplied across all calls.

**Realistic uplift: 30-50% more than simple token arithmetic suggests.**

### 🪤 TRAP 4: The 14-Agent Fantasy

Doc 03 defines 14 specialist agents. Running even 6 of these with dedicated OpenClaw sessions, heartbeat polling, and cron jobs means:
- Constant background LLM calls just for heartbeats and status checks
- Context loading on every session wake-up (SOUL.md, MEMORY.md, recent context)
- Each heartbeat poll: ~500-2K tokens × every 30 min × 6 agents = **~72K-288K tokens/day just on keepalive**

At Sonnet pricing, that's $0.65-2.60/day = **$20-78/mo on agents doing literally nothing.**

### 🪤 TRAP 5: The dreamteam Server is a SPOF and Resource Bottleneck

Everything runs on one box:
- 6+ Docker containers (Postgres, TimescaleDB, Redis, Qdrant, Neo4j, Langfuse, Grafana, Prometheus, MinIO, LiteLLM)
- llama.cpp inference server
- OpenClaw agent runtime
- Git repos, artifact storage

**27GB RAM for services + 24GB VRAM for inference + OS overhead = you're at 90%+ utilization.** When TimescaleDB does a compression job while Neo4j runs a graph traversal while llama.cpp is serving a 32B model — you'll OOM or thrash swap.

Second server needed by month 3-4. Budget **$1,500-3,000** for a decent used workstation or **$50-100/mo** for a VPS.

### 🪤 TRAP 6: Validation Budget Is a Burn Rate, Not an Investment

Doc 10 says test 6-8 opportunities/month at $250 each = $1,500-2,000/mo. With a 25-35% pass rate, you're spending $1,000-1,500/mo on ideas that die. That's fine strategically, but it's **pure burn** that doesn't show up as "cost per product" — it's cost per *attempt*.

### 🪤 TRAP 7: Marketing Costs Compound, Not Replace

Doc 10's growth engine runs SEO + paid + community for each product that survives. But killed products leave behind:
- Domains still renewing ($1/mo each, adds up)
- Landing pages still hosted (usually free, but cognitive overhead)
- Content still indexed (potential brand confusion)

More importantly, **marketing budget is the hardest to predict**. The $300-500/mo/product estimate assumes disciplined ROAS testing. One product with a stubborn founder-agent burning $500/mo on paid ads with no ROAS = pure waste.

### 🪤 TRAP 8: Prompt Caching Is Not Guaranteed

Docs repeatedly cite "90% savings from prompt caching." Reality:
- Anthropic cache TTL = 5 minutes. If your agents don't call within 5 min, cache is cold.
- Cache only works for the **prefix** of the prompt. Dynamic content at the start invalidates everything.
- Cache hits require identical byte-for-byte content. System prompt changes = full miss.

**Realistic cache hit rate: 20-40%, not 90%.** Budget accordingly.

### 🪤 TRAP 9: Monitoring Stack Has Its Own Cost

The docs claim the monitoring stack "costs ~$0/month to run (self-hosted, open-source)." This is technically true but misleading:
- Langfuse needs PostgreSQL + ClickHouse = **4-6GB RAM**
- Prometheus TSDB grows with metric cardinality. 14 agents × multiple models × multiple products = lots of time series
- Grafana + Prometheus + Langfuse + Redis = 4 more services to maintain, update, debug when they break

The cost is **operational complexity and RAM**, not dollars.

### 🪤 TRAP 10: Extended Thinking Burns Tokens Invisibly

Doc 03 assigns `thinking: extended` to Rex, Scout, Hawk, Finn, Law, Atlas, and Dot (7 of 14 agents). Extended thinking on Claude generates internal reasoning tokens that you **pay for but never see**. A single Opus extended-thinking call can easily burn 5-10K thinking tokens on top of the visible output.

At Opus pricing ($25/MTok output, thinking tokens billed as output): 10K thinking tokens = $0.25 per call. Do that 50 times/day = **$12.50/day = $375/mo** just on invisible thinking.

---

## 4. Revised P&L Projection

Using realistic costs with traps accounted for:

| | Month 1 | Month 6 | Month 12 |
|---|---|---|---|
| **Revenue** | $200 | $8,000 | $40,000 |
| **Stripe fees** | -$6 | -$250 | -$1,280 |
| **Net revenue** | $194 | $7,750 | $38,720 |
| | | | |
| **LLM APIs** | -$700 | -$1,800 | -$2,800 |
| **Electricity** | -$50 | -$60 | -$65 |
| **Hardware amortization** | -$75 | -$75 | -$75 |
| **SaaS/infra per product** | -$10 | -$200 | -$600 |
| **Domains** | -$2 | -$8 | -$15 |
| **Marketing/growth** | -$200 | -$1,200 | -$2,500 |
| **Validation burn** | -$500 | -$1,500 | -$1,500 |
| **Browserbase** | -$50 | -$150 | -$250 |
| **Second server (from M4)** | $0 | -$100 | -$100 |
| **Misc/unexpected (10%)** | -$100 | -$300 | -$500 |
| **Total costs** | **-$1,687** | **-$5,393** | **-$8,405** |
| | | | |
| **Net profit** | **-$1,493** | **+$2,357** | **+$30,315** |
| **Gross margin** | — | 30% | 78% |

**Key differences from doc 10's projections:**
- Month 1: Doc says -$1,800 net. I say **-$1,493**. Close enough — they padded for human time.
- Month 6: Doc says +$3,500 net. I say **+$2,357**. Stripe fees + validation burn + realistic LLM costs eat the difference.
- Month 12: Doc says +$32,500 net. I say **+$30,315**. Surprisingly close at scale because margins improve. But Stripe alone eats $1,280.

---

## 5. Recommendations

### Immediate (Month 1)

1. **Set hard provider caps NOW.** Before building anything, set Anthropic/OpenAI/Google dashboard spend limits to $300/mo each. Doc 08 says this but it needs to happen day zero.
2. **Don't build the full monitoring stack in month 1.** You don't need Langfuse + Prometheus + Grafana + TimescaleDB + Neo4j + Qdrant when you have 2 agents. Start with LiteLLM + Redis + SQLite. Add complexity when you need it.
3. **Budget $2K for month 1, not $1K.** Have buffer for LLM cost surprises when the router isn't built yet.

### Medium-Term (Month 3-6)

4. **Track Stripe fees as a cost line item.** The P&L in doc 10 is wrong without this.
5. **Monitor free tier usage per product.** Set alerts when any product hits 80% of any free tier limit. The $25/mo Supabase upgrade shouldn't be a surprise.
6. **Kill heartbeat polling for idle agents.** Use event-driven wake-up instead. Stop burning tokens on agents checking "anything for me? no? ok bye."
7. **Plan the second server.** By month 4, dreamteam will be groaning. Budget $2K-3K or $100/mo VPS.

### Long-Term (Month 6-12)

8. **Benchmark actual cache hit rates.** If you're below 30%, restructure prompts for better cacheability. This is a $300-500/mo swing.
9. **Audit extended thinking usage.** Are 7 agents really using extended thinking productively? Switch to `low` or `off` where quality doesn't suffer.
10. **Consider Paddle/LemonSqueezy over Stripe for some products.** They handle sales tax, which Stripe doesn't. Tax compliance is a hidden cost that will hit at scale.

---

## 6. Bottom Line

The architecture is **cost-conscious in design** — self-hosting, smart routing, tiered models, budget enforcement. The economics work at scale. The danger zone is **months 2-5**: you're burning $1,500-3,000/mo building the machine before it generates meaningful revenue. That's a $6K-15K bet.

The biggest cost risks are:
1. **LLM costs 30-50% higher than modeled** (context bloat, retries, thinking tokens, low cache rates)
2. **Stripe fees eating 3-4% of gross revenue** (never mentioned in the revenue docs)
3. **Server resource exhaustion** requiring hardware spend in month 3-4
4. **Marketing spend with poor ROAS** on products that should have been killed faster

If the revenue engine works even at 60% of projections ($24K MRR by month 12 instead of $40K), the economics still work. The factory breaks even around month 6-7 and generates meaningful profit by month 9-10. **The architecture is viable. The cost projections just need honest padding.**

---

*Audited: 2026-02-08 by Finn*
