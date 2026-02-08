# Review 07 — Data Flow Trace Analysis

> **Reviewer:** Atlas  
> **Date:** 2026-02-08  
> **Scope:** End-to-end data flows through the full system architecture  
> **Method:** Three concrete scenarios traced step-by-step with latency estimates, bottleneck identification, and data transformation analysis

---

## Overview

This document traces data through the entire factory architecture for three representative scenarios. Each step identifies: what data moves, where it goes, what transforms it, estimated latency, and where bottlenecks lurk.

**Source documents:** All 20 architecture docs (01–20).

---

## Scenario 1: "Build Me a Price Comparison API"

**Trigger:** Adam sends a WhatsApp message: "Build me a price comparison API that scrapes competitor prices and exposes a REST endpoint"

### Full Data Flow

```
Step  Component                Data                              Latency    Cumulative
─────────────────────────────────────────────────────────────────────────────────────────
 1    WhatsApp → OpenClaw      Raw text message                  ~500ms     0.5s
 2    OpenClaw → Kev session   Session context + message         ~200ms     0.7s
 3    Kev → LLM (Opus)         System prompt + message + memory  ~3-8s      8.7s
      ↳ Kev reasons: what type of work, decompose into DAG
 4    Kev → Task Queue         Creates DAG: 6 tasks (JSON)       ~50ms      8.8s
      ↳ T1: Research competitors (Scout)
      ↳ T2: Design API spec (Atlas/Kev)        [depends: T1]
      ↳ T3: Build backend (Forge/Rex)           [depends: T2]
      ↳ T4: Build scraper module (Forge/Rex)    [depends: T2]
      ↳ T5: Test & QA (Hawk)                    [depends: T3, T4]
      ↳ T6: Deploy (Forge/Dash)                 [depends: T5]
 5    Kev → WhatsApp (via OC)  "On it. Breaking into 6 tasks..." ~300ms     9.1s
 6    Kev → OpenClaw subagent  Spawns Scout for T1               ~1-2s      11s
```

**Phase 1: Research (T1 — Scout)**

```
Step  Component                Data                              Latency    Cumulative
─────────────────────────────────────────────────────────────────────────────────────────
 7    Scout session starts     Task spec from queue               ~1s        12s
 8    Scout → Smart Router     Research request (quality: good)   ~10ms      12s
 9    Smart Router → Gemini    Route to Gemini 2.5 Flash          ~2-4s      16s
      ↳ Long context research, cost-optimized
      ↳ Data: "Find price comparison APIs, competitor pricing..."
10    Scout → Browser (fetch)  Scrape 3-5 competitor sites        ~5-15s     31s
      ↳ Stagehand/Playwright for dynamic sites
      ↳ Data flows: URL → Browserbase → HTML → extract() → JSON
11    Scout → Smart Router     Synthesis request                  ~3-5s      36s
      ↳ Input: scraped data + search results
      ↳ Output: structured research report (markdown)
12    Scout → Shared FS        Write /artifacts/{project}/T1/     ~50ms      36s
      ↳ research-report.md + competitor-data.json
13    Scout → Task Queue       Mark T1 COMPLETED                  ~50ms      36s
      ↳ Side effect: T2 dependency resolved → T2 moves to READY
14    Scout → Memory (Qdrant)  Store research findings            ~100ms     36s
```

**⏱ T1 total: ~25 seconds** (dominated by web scraping and LLM synthesis)

**Phase 2: API Design (T2 — Kev/Atlas)**

```
Step  Component                Data                              Latency    Cumulative
─────────────────────────────────────────────────────────────────────────────────────────
15    Kev claims T2            Reads T1 artifacts from shared FS  ~200ms     36.2s
16    Kev → Smart Router       Design request (quality: best)     ~10ms      
17    Smart Router → Opus      Generate API spec + OpenAPI schema ~5-10s     46s
      ↳ Input: research report + task spec + tech stack defaults
      ↳ Output: api-spec.md + openapi.yaml
18    Kev → Shared FS          Write artifacts                    ~50ms      46s
19    Kev → Task Queue         T2 COMPLETED → T3, T4 unblocked   ~50ms      46s
```

**⏱ T2 total: ~10 seconds**

**Phase 3: Build (T3 + T4 — parallel via Pi SDK)**

```
Step  Component                Data                              Latency    Cumulative
─────────────────────────────────────────────────────────────────────────────────────────
20    Kev → Pi SDK             Spawn 2 parallel sessions          ~2s        48s
      ↳ Session A: Claude Code for REST API (T3)
      ↳ Session B: Claude Code for scraper module (T4)
21    Pi SDK → Smart Router    Both route to Sonnet 4.5           ~10ms
22    Smart Router checks      Budget remaining? ✓                ~1ms
      ↳ Redis: GET budget:agent:forge → $45.00 remaining
      ↳ Redis: INCRBYFLOAT velocity:forge 0.15
23    Claude Code (A) works    Scaffolds from ts-api template     ~3-8min    
      ↳ Reads: api-spec.md, openapi.yaml, AGENTS.md
      ↳ Writes: src/routes/, src/services/, package.json
      ↳ Git: creates branch agent/price-api
      ↳ Runs: npm install, tsc --strict, vitest
      ↳ Multiple LLM round-trips (5-15 calls)
      ↳ Each call: Smart Router → Sonnet → ~2-5s
      ↳ Heartbeat every 30s → Task Queue
24    Claude Code (B) works    Builds scraper module               ~3-8min
      ↳ Parallel in git worktree
      ↳ Reads: competitor-data.json, research-report.md
      ↳ Writes: src/scrapers/, src/scheduler/
      ↳ Uses Playwright for scraping logic
25    Pi SDK → Shared FS       Both sessions write results         ~100ms
26    Pi SDK → Task Queue      T3, T4 COMPLETED → T5 unblocked    ~50ms
```

**⏱ T3+T4 total: ~5-8 minutes** (parallel, so wall-clock = max of the two)

**🔴 BOTTLENECK 1: Build phase dominates total time.** Each Claude Code session makes 5-15 LLM calls, each 2-5s. With tool execution (npm install, git, test runs) adding 30-60s total, the build phase is 80%+ of total wall-clock time.

**Phase 4: Test & QA (T5 — Hawk)**

```
Step  Component                Data                              Latency    Cumulative
─────────────────────────────────────────────────────────────────────────────────────────
27    Hawk session starts      Claims T5 from queue               ~1s
28    Hawk → git               Checks out merged branch           ~2s
29    Hawk → Smart Router      Test generation (quality: best)    ~10ms
30    Smart Router → Sonnet    Different model/persona than Rex   ~5-10s
      ↳ Input: PR diff + api-spec.md + existing tests
      ↳ Output: adversarial test suite
31    Hawk → shell             Run full test suite                ~30-60s
      ↳ vitest (unit) + playwright (E2E if UI)
      ↳ tsc --strict, eslint, semgrep, gitleaks
32    Hawk → Smart Router      Mutation testing on changed files  ~2-5min
      ↳ Stryker runs → mutation score
33    Hawk → Task Queue        T5 COMPLETED or FAILED             ~50ms
      ↳ If FAILED: error report → T3/T4 re-opened (retry)
      ↳ Retry cascade: same model → adjusted prompt → fallback model
```

**⏱ T5 total: ~3-6 minutes**

**Phase 5: Deploy (T6 — Forge/Dash)**

```
Step  Component                Data                              Latency    Cumulative
─────────────────────────────────────────────────────────────────────────────────────────
34    Forge claims T6          Reads deploy config                ~200ms
35    Forge → GitHub Actions   Push to main triggers CI           ~30-60s
      ↳ Fast gates: lint, type-check, secrets scan (~30s)
      ↳ Tests: full suite re-run (~2-3min)
      ↳ Security: semgrep, npm audit (~1-2min)
36    CI → Cloudflare/Railway  Deploy to preview                  ~30-60s
37    Forge → browser          Post-deploy smoke test             ~30s
      ↳ Health check + critical path verification
38    Human approval gate      Tier 2: production deploy          VARIABLE
      ↳ WhatsApp notification to Adam
      ↳ Timeout: 4 hours (deny if no response)
39    Deploy → production      Progressive: 5% → 25% → 100%     ~20min
40    Forge → Monitoring       Set up Sentry + BetterStack        ~30s
41    Forge → Task Queue       T6 COMPLETED                       ~50ms
42    Kev → WhatsApp           "Price comparison API is live 🚀"  ~300ms
```

**⏱ T6 total: ~5-10 minutes** (excluding human approval wait)

### End-to-End Summary

```
Phase           Time           Data Volume        Cost Estimate
──────────────────────────────────────────────────────────────────
Research        ~25s           ~50KB artifacts     ~$0.05 (Gemini)
Design          ~10s           ~20KB spec          ~$0.15 (Opus)
Build           ~5-8min        ~200KB code         ~$1.50 (Sonnet × 2)
Test/QA         ~3-6min        ~50KB test results  ~$0.30 (Sonnet)
Deploy          ~5-10min       ~10KB deploy logs   ~$0.05 (minimal LLM)
──────────────────────────────────────────────────────────────────
TOTAL           ~15-25min      ~330KB              ~$2.05
(excl. human approval)
```

### Data Transformation Chain

```
Natural language request
  → Structured task DAG (JSON)
    → Research query + web scraping instructions
      → Raw HTML + search results
        → Structured research report (markdown + JSON)
          → API spec + OpenAPI schema (YAML)
            → TypeScript source code + tests
              → Git commits + PR
                → CI artifacts (test results, coverage, security scan)
                  → Docker image / deployed service
                    → Live API endpoint + monitoring dashboard
```

### Bottlenecks & Risks

| Bottleneck | Location | Impact | Mitigation |
|---|---|---|---|
| **🔴 Build phase LLM latency** | Steps 23-24 | 80% of wall-clock | Parallel sessions help; Groq for simple sub-tasks |
| **🔴 Human approval gate** | Step 38 | Unbounded wait (0-4h) | Pre-authorized playbooks for low-risk deploys |
| **🟡 Web scraping variability** | Step 10 | 5-30s depending on sites | Browserbase cloud for reliability; timeout + fallback |
| **🟡 CI pipeline duration** | Step 35 | 3-5min fixed overhead | Parallel CI jobs; cache node_modules |
| **🟡 Model swap on dreamteam** | If local inference needed | 15-30s GPU swap | Cloud fallback during swap |
| **🟢 Task Queue operations** | Steps 4,13,19,26,33 | <100ms each | SQLite is fast enough |
| **🟢 Filesystem I/O** | Artifact reads/writes | <100ms each | SSD, small files |

---

## Scenario 2: Cron Triggers Revenue Engine

**Trigger:** Kev's 30-minute heartbeat cron fires. Revenue engine pipeline kicks off: scan markets → validate opportunity → build → launch.

### Full Data Flow

```
Step  Component                Data                              Latency    Cumulative
─────────────────────────────────────────────────────────────────────────────────────────
 1    OpenClaw cron            Heartbeat fires for Kev            ~100ms     0.1s
 2    Kev reads HEARTBEAT.md   Check: any revenue engine tasks?   ~50ms      0.15s
 3    Kev → Task Queue         Check portfolio state              ~50ms      0.2s
      ↳ Read: products.json, pipeline-state.json
      ↳ Decision: time for next market scan cycle
```

**Phase 1: Market Scanning (Market Scanner Agent)**

```
Step  Component                Data                              Latency    Cumulative
─────────────────────────────────────────────────────────────────────────────────────────
 4    Kev → subagent           Spawn Scout as Market Scanner      ~1s        1.2s
 5    Scout → Smart Router     Research queries (quality: good)   ~10ms
 6    Smart Router → Cerebras  Route to free tier (bulk research) ~1-2s
      ↳ Cerebras: gpt-oss-120b, 1500 tok/s, $0.00
 7    Scout → Browser          Scrape signal sources              ~30-120s   ~2min
      ├── Reddit API: r/SaaS, r/SideProject (3-5 subs)
      │   ↳ Data: JSON → pain points extraction
      ├── HN API: front page + "Ask HN" posts
      │   ↳ Data: JSON → unmet needs extraction
      ├── G2/Capterra: competitor reviews (via Browserbase)
      │   ↳ Data: HTML → structured review extraction
      ├── Google Trends API: rising keywords
      │   ↳ Data: JSON → trend signals
      └── Product Hunt: recent launches + comments
          ↳ Data: GraphQL → gap analysis
 8    Scout → Smart Router     Synthesize opportunities           ~3-5s
      ↳ Input: ~100KB of scraped signals
      ↳ Output: 5-10 Opportunity objects (structured JSON)
 9    Scout → Shared FS        Write opportunities to pipeline/   ~50ms
10    Scout → Memory (Qdrant)  Store market intelligence          ~100ms
      ↳ Vector embed opportunity descriptions
      ↳ Store for dedup against past scans
11    Scout → Task Queue       Scanning complete                  ~50ms
```

**⏱ Scanning total: ~2-3 minutes**

**Data volume:** ~100KB scraped → ~15KB structured opportunities

**Phase 2: Opportunity Scoring & Validation**

```
Step  Component                Data                              Latency    Cumulative
─────────────────────────────────────────────────────────────────────────────────────────
12    Kev reads opportunities  Load 5-10 candidates               ~50ms      ~3min
13    Kev → Smart Router       Score each opportunity             ~3-5s
      ↳ Model: Sonnet 4.5 (needs reasoning for scoring)
      ↳ Input: opportunity + historical outcomes from Memory
      ↳ Output: confidenceScore, buildComplexity, timeToRevenue
      ↳ Query Memory: "similar opportunities we tried before?"
14    Memory (Qdrant) → Kev    Similar past opportunities         ~50ms
      ↳ Vector search: top-5 similar previous attempts
      ↳ Includes outcome data (killed, scaled, revenue)
15    Kev filters              Keep opportunities score ≥ 60      ~10ms
      ↳ Typically 2-4 pass the filter
16    Kev → Task Queue         Create validation tasks            ~50ms
```

For each passing opportunity (let's trace one):

```
Step  Component                Data                              Latency    Cumulative
─────────────────────────────────────────────────────────────────────────────────────────
17    Validation Agent starts  Claims validation task             ~1s
18    Agent → Smart Router     Generate landing page copy         ~3-5s
      ↳ Sonnet 4.5: headline, value prop, CTA, pricing
19    Agent → v0/Vercel AI     Generate landing page              ~30-60s
      ↳ Input: copy + template selection
      ↳ Output: React component / static HTML
20    Agent → Cloudflare Pages Deploy landing page                ~30s
      ↳ wrangler pages deploy
21    Agent → Smart Router     Generate social posts              ~2-3s
      ↳ 3 Reddit posts, 1 HN post, 2 tweets
22    Agent → Social APIs      Post to communities                ~5-10s
      ├── Reddit API: POST to 3 subreddits
      ├── Twitter/X API: POST thread
      └── Rate limited: 1 post per 2 seconds
23    Agent → Monitoring       Set up PostHog tracking on LP      ~10s
      ↳ Events: pageview, cta_click, signup
24    Agent → Task Queue       Validation task RUNNING            ~50ms
      ↳ Status: waiting for 48h traffic data
```

**⏱ Validation setup: ~3-5 minutes per opportunity**

**48 hours later (next cron cycle picks up):**

```
Step  Component                Data                              Latency    Cumulative
─────────────────────────────────────────────────────────────────────────────────────────
25    Kev cron fires           Check validation tasks in progress ~200ms
26    Kev → PostHog API        Pull analytics for landing pages   ~2s
      ↳ Data: pageviews, CTR, signups, time-on-page
27    Kev → Smart Router       Evaluate validation results        ~3-5s
      ↳ Input: analytics data + kill criteria
      ↳ Output: GO / NO-GO decision with rationale
28a   IF GO:
      Kev → Task Queue         Create build DAG (like Scenario 1) ~50ms
      ↳ Triggers Phase 3-6 from revenue engine doc [10]
28b   IF NO-GO:
      Kev → Shared FS          Archive opportunity + learnings    ~50ms
      Kev → Memory             Store outcome for future scoring   ~100ms
      Kev → Cloudflare         Tear down landing page             ~10s
```

**Phase 3: Build (if GO) — follows Scenario 1 pattern**

```
Step  Component                Data                              Latency
─────────────────────────────────────────────────────────────────────────
29    Blueprint Agent          Generate BusinessBlueprint         ~5-10s
      ↳ Tech stack defaults from [15], pricing from [10]
30    Build Swarm (4 agents)   Parallel build via Pi SDK          ~15-30min
      ↳ Frontend agent → Next.js scaffold
      ↳ Backend agent → Hono API + Drizzle
      ↳ Copy agent → Docs, emails, help pages
      ↳ Design agent → OG images, favicon, brand
31    Integration Agent        Merge worktrees, resolve conflicts ~2-5min
32    QA Agent (Hawk)          Full test suite + mutation testing ~5-10min
33    Deploy Agent             Cloudflare + Railway + Stripe      ~5-10min
```

**Phase 4: Launch Playbook (automated)**

```
Step  Component                Data                              Latency
─────────────────────────────────────────────────────────────────────────
34    Echo (Content)           Generate launch content            ~3-5min
      ↳ Product Hunt submission
      ↳ 5 social media posts
      ↳ Launch email to waitlist
      ↳ 3 SEO articles
35    Blaze (Marketing)        Execute launch playbook            ~10-30min
      ↳ Post to directories (BetaList, SaaSHub, etc.)
      ↳ Reddit/HN "Show" posts
      ↳ Direct outreach (20 emails)
36    Blaze → Analytics        Set up tracking                    ~5min
      ↳ PostHog events, Stripe webhooks
      ↳ Google Search Console verification
37    Dash (Analytics)         Create product dashboard           ~2min
      ↳ Grafana API: generate dashboard from template
      ↳ TimescaleDB: create product-specific views
```

### End-to-End Revenue Engine Cycle

```
Phase                   Time              Cost           Data Flow
───────────────────────────────────────────────────────────────────────
Market Scan             ~2-3min           ~$0.02         Signals → Opportunities
Scoring                 ~30s              ~$0.10         Opportunities → Scored list
Validation (setup)      ~5min/opp         ~$0.50/opp     Opp → Landing page + posts
Validation (wait)       48 hours          $50-100 ads    Traffic data → GO/NO-GO
Build (if GO)           ~15-30min         ~$3-5          Blueprint → Deployed product
Launch                  ~30min            ~$0.50 LLM     Product → Distribution
Post-launch monitor     Ongoing           ~$0.10/day     Metrics → Kill/Scale decision
───────────────────────────────────────────────────────────────────────
TOTAL (to launch)       ~50min + 48h      ~$55-107       Idea → Live product
```

### Critical Data Paths

```
Market signals (100KB raw)
  → Opportunity objects (15KB structured JSON)
    → Validation landing page (50KB HTML/React)
      → Traffic analytics (2KB metrics JSON)
        → GO/NO-GO decision (100B boolean + rationale)
          → BusinessBlueprint (10KB structured spec)
            → Source code (200-500KB)
              → Deployed product + monitoring
                → Revenue events → TimescaleDB
                  → Portfolio P&L dashboard
```

### Bottlenecks & Risks

| Bottleneck | Location | Impact | Mitigation |
|---|---|---|---|
| **🔴 48h validation wait** | Steps 24-25 | Dominates total cycle time | Run multiple validations in parallel; reduce to 24h for high-confidence |
| **🔴 Build swarm concurrency** | Step 30 | 4 parallel LLM sessions = cost spike | Budget caps per-product; use Cerebras for copy/design agents |
| **🟡 Web scraping reliability** | Step 7 | Anti-bot blocks, rate limits | Browserbase stealth; fallback to API-based sources |
| **🟡 Social posting rate limits** | Step 22 | Reddit/HN throttling | Queue posts over hours, not seconds |
| **🟡 Memory query for similar opps** | Step 14 | Cold-start: no historical data | Bootstrap with manual entries; improves over time |
| **🟡 Stripe account risk** | Step 33 | Account termination kills billing | Multiple accounts; backup processors (Paddle, Lemon Squeezy) |
| **🟢 Cron granularity** | Step 1 | 30min heartbeat = max 30min delay | Acceptable; event-driven triggers for urgent items |

---

## Scenario 3: Customer Complaint

**Trigger:** A customer emails support@product.com: "Your API has been returning 500 errors for the past hour. I'm losing money. Fix this NOW or I'm canceling."

### Full Data Flow

```
Step  Component                Data                              Latency    Cumulative
─────────────────────────────────────────────────────────────────────────────────────────
 1    Email → Webhook          Inbound email via SendGrid/Resend  ~1-5s      5s
      ↳ Data: from, subject, body, headers
 2    Webhook → OpenClaw       Routes to Ally (Support Agent)     ~500ms     5.5s
 3    Ally session activates   Receives email context             ~1s        6.5s
 4    Ally → Smart Router      Classify & triage                  ~1-2s      8.5s
      ↳ Model: Haiku (fast, cheap — triage is simple)
      ↳ Input: email text
      ↳ Output: {
          severity: "high",
          category: "service_outage",
          sentiment: "angry",
          churn_risk: "high",
          action: "escalate_immediately"
        }
```

**🔴 CRITICAL PATH: Escalation fires immediately**

```
Step  Component                Data                              Latency    Cumulative
─────────────────────────────────────────────────────────────────────────────────────────
 5    Ally → Event Bus (NATS)  Emit: escalation.high              ~10ms      8.5s
      ↳ Message envelope: {
          type: "ESCALATION",
          source: "agent:ally",
          priority: 3 (critical),
          correlationId: "complaint-xyz",
          payload: { customer_id, product_id, complaint_summary }
        }
 6    Event Bus → Kev          Kev receives escalation            ~50ms      8.6s
 7    Event Bus → Dash         Dash receives (parallel)           ~50ms      8.6s
```

**Branch A: Kev coordinates response**

```
Step  Component                Data                              Latency    Cumulative
─────────────────────────────────────────────────────────────────────────────────────────
 8    Kev → Smart Router       Assess situation                   ~3-5s      13s
      ↳ Model: Sonnet (needs reasoning about blast radius)
      ↳ Input: complaint + product status + recent deploy history
 9    Kev → Monitoring         Check product health               ~2s        15s
      ├── Sentry API: recent errors for product
      │   ↳ Data: error count spike, stack traces
      ├── BetterStack: uptime status
      │   ↳ Data: 3 health check failures in past hour
      └── PostHog: user activity drop
          ↳ Data: 60% fewer API calls vs baseline
10    Kev → Git (GitHub API)   Recent deploys to this product     ~1s        16s
      ↳ Data: last deploy was 2 hours ago, commit: "optimize query cache"
11    Kev decides              Root cause hypothesis              ~0ms       16s
      ↳ "Deploy 2h ago likely caused 500 errors. Rollback first, investigate second."
```

**Branch B: Dash tracks the incident (parallel with Kev)**

```
Step  Component                Data                              Latency    Cumulative
─────────────────────────────────────────────────────────────────────────────────────────
 8b   Dash → TimescaleDB      Query error rate timeline           ~500ms     9s
      ↳ SQL: SELECT time_bucket('5 min', time), count(*)
              FROM task_events WHERE product_id = X
              AND status = 'failed' AND time > now() - interval '4 hours'
 9b   Dash → Redis            Check current error velocity        ~1ms       9s
      ↳ GET dash:errors:5min:{product_id}
10b   Dash → Grafana API      Create incident annotation          ~500ms     9.5s
      ↳ Marks the timeline with "Customer complaint received"
11b   Dash → TimescaleDB      Log incident event                  ~50ms      9.5s
```

**Branch C: Immediate customer response (parallel)**

```
Step  Component                Data                              Latency    Cumulative
─────────────────────────────────────────────────────────────────────────────────────────
 8c   Ally → Smart Router      Draft acknowledgment email         ~2-3s      11s
      ↳ Model: Sonnet (customer-facing, needs quality)
      ↳ Input: complaint + severity + customer history
      ↳ Output: empathetic acknowledgment, NO promises yet
      ↳ AUTO-FLAG: angry customer + mentions "canceling"
          → Tier 2 approval required (ref: doc [12])
 9c   Ally → Approval Queue    Request approval to send           ~50ms      11s
      ↳ WhatsApp notification to Adam:
        "🔒 APPROVAL: Reply to angry customer about API outage.
         Draft: [acknowledgment text]. ✅/❌?"
```

**🔴 HUMAN GATE: Adam must approve customer response**
- If Adam responds quickly (~1-5min): flow continues
- If 3am / Adam unavailable: conservative auto-response after 2h timeout
- Emergency protocol: Ally sends a minimal "We're aware and investigating" (pre-approved template)

**Rollback & Fix (Kev orchestrates)**

```
Step  Component                Data                              Latency    Cumulative
─────────────────────────────────────────────────────────────────────────────────────────
12    Kev → Forge/Dash         Auto-rollback (always authorized)  ~30-60s    ~1.5min
      ↳ Forge: git revert HEAD, push to main
      ↳ CI: abbreviated pipeline (skip mutation testing)
      ↳ Deploy: rollback to previous known-good version
      ↳ Data flow: git revert → CI → Cloudflare/Railway deploy
13    Forge → Monitoring       Verify rollback health             ~30s       ~2min
      ├── Health endpoint: 200 OK ✓
      ├── Sentry: error rate dropping ✓
      └── BetterStack: uptime restored ✓
14    Kev → Event Bus          Emit: incident.mitigated           ~10ms
15    Ally receives            Update for customer                ~50ms
16    Ally → Smart Router      Draft resolution email             ~3-5s      ~2.5min
      ↳ "Issue identified and resolved. Service restored."
      ↳ AUTO-FLAG: mentions resolution → Tier 2 approval
17    Ally → Approval Queue    Request approval                   ~50ms
```

**Post-Incident Investigation (async)**

```
Step  Component                Data                              Latency    Cumulative
─────────────────────────────────────────────────────────────────────────────────────────
18    Kev → subagent           Spawn Forge for root cause         ~1s
19    Forge → git              Analyze reverted commit            ~5s
      ↳ Read diff, understand what "optimize query cache" changed
20    Forge → Smart Router     Root cause analysis                ~5-10s
      ↳ Model: Opus (complex reasoning about code + infra interaction)
      ↳ Output: "Query cache optimization used stale connection pool
                 under concurrent load. Fix: use connection-per-request
                 pattern instead of shared pool."
21    Forge → git              Create fix PR (not just revert)    ~5-15min
22    Hawk → QA                Review fix with adversarial tests  ~3-5min
      ↳ Specifically: concurrent load tests
      ↳ Specifically: cache invalidation edge cases
23    Forge → Task Queue       Fix ready for deploy               ~50ms
24    Human approval           Tier 2: production deploy          VARIABLE
25    Post-mortem Agent        Generate incident report           ~2min
      ↳ Timeline, root cause, fix, prevention measures
      ↳ Stored in Memory + Shared FS
26    Self-Improvement         Signal extraction from incident    ~30s
      ↳ Signal: "deploy caused outage"
      ↳ Anti-pattern logged: "shared connection pool under load"
      ↳ Playbook updated: "always load-test cache changes"
27    Dash → TimescaleDB       Log full incident cost             ~50ms
      ↳ LLM cost, human time, customer impact, revenue impact
```

### End-to-End Customer Complaint Timeline

```
Time     Event                                    Data State
────────────────────────────────────────────────────────────────
T+0s     Email received                           Raw email text
T+8s     Triage complete, escalation fired         Classified complaint JSON
T+11s    Customer ack drafted (pending approval)   Draft email
T+15s    Root cause hypothesized                   Deploy correlation
T+60s    Rollback deployed                         Previous version live
T+120s   Health verified, service restored         Monitoring data confirms
T+150s   Customer notified of resolution           Approved email sent
T+5min   Root cause analysis started               Code analysis
T+20min  Fix PR created and tested                 New code + tests
T+30min  Post-mortem complete                      Incident report
T+varies Fix deployed to production                Permanent fix live
```

### Data Transformation Chain

```
Raw email text
  → Classified complaint (JSON: severity, category, sentiment)
    → Escalation event (NATS message envelope)
      → Monitoring queries (Sentry errors, uptime, analytics)
        → Incident correlation (deploy history + error timeline)
          → Rollback command (git revert + deploy pipeline)
            → Health verification (HTTP checks + metric comparison)
              → Customer communication (drafted + approved email)
                → Root cause analysis (code diff → LLM reasoning)
                  → Fix PR (code + tests)
                    → Post-mortem report (markdown)
                      → Self-improvement signal (anti-pattern + playbook update)
                        → Incident cost record (TimescaleDB)
```

### Bottlenecks & Risks

| Bottleneck | Location | Impact | Mitigation |
|---|---|---|---|
| **🔴 Human approval for customer email** | Steps 9c, 17 | Customer waits for response | Pre-approved templates for common scenarios; graduated trust |
| **🔴 3am incident handling** | All steps | No human available | Auto-rollback is always authorized; minimal template responses; phone escalation for revenue-impacting |
| **🟡 Monitoring API latency** | Step 9 | 2-3s to gather health data | Cache recent metrics in Redis; parallel API calls |
| **🟡 CI pipeline for rollback** | Step 12 | 30-60s even abbreviated | Fast-track rollback skips mutation testing, runs only smoke tests |
| **🟡 Multiple approval gates** | Steps 9c, 17, 24 | Each can stall the flow | Batch related approvals; emergency override for active incidents |
| **🟢 Event bus delivery** | Steps 5-7 | <100ms with NATS | At-least-once delivery with JetStream |
| **🟢 Triage classification** | Step 4 | <2s with Haiku | Cheap model is fine for classification |

---

## Cross-Scenario Analysis

### Latency Budget Breakdown (Typical)

```
Component               Scenario 1    Scenario 2    Scenario 3
                        (Build API)   (Rev Engine)  (Complaint)
──────────────────────────────────────────────────────────────
LLM inference           ~70%          ~30%          ~40%
Tool execution          ~15%          ~10%          ~15%
Web scraping/APIs       ~5%           ~40%          ~10%
CI/CD pipeline          ~8%           ~15%          ~20%
Human approval          ~0-80%*       ~0%**         ~30-80%*
Network/routing         ~2%           ~5%           ~5%
──────────────────────────────────────────────────────────────
* Highly variable — can dominate when human is unavailable
** Revenue engine is designed for autonomous operation
```

### System-Wide Bottlenecks

| # | Bottleneck | Affects | Severity | Root Cause | Fix |
|---|---|---|---|---|---|
| 1 | **LLM round-trip latency** | All scenarios | High | 2-8s per call × 5-20 calls per task | Parallel sessions; cheaper models for simple sub-tasks; prompt caching (90% input savings) |
| 2 | **Human approval gates** | Scenarios 1, 3 | High | Blocking wait for WhatsApp response | Graduated trust; pre-authorized playbooks; batch approvals; timeout-with-safe-default |
| 3 | **GPU model swapping** | All (if local) | Medium | 15-30s swap, can't run 8B+32B simultaneously | Cloud fallback during swap; always-loaded 8B; second GPU ($800) eliminates swapping |
| 4 | **Build phase duration** | Scenario 1, 2 | Medium | Complex code gen = many LLM round-trips | Git worktrees for parallelism; template scaffolding reduces generation; playbooks for common patterns |
| 5 | **Web scraping reliability** | Scenario 1, 2 | Medium | Anti-bot, rate limits, site changes | Browserbase stealth; fallback to APIs; cached results for recent scrapes |
| 6 | **CI pipeline overhead** | Scenario 1, 2, 3 | Low-Med | 3-5min minimum even for trivial changes | Parallel jobs; dependency caching; skip non-essential gates for rollbacks |

### Data Volume Estimates (Daily Steady-State)

```
Source                          Volume/Day      Storage
────────────────────────────────────────────────────────
LLM call logs                   ~5,000 calls    ~50MB (TimescaleDB)
Task events                     ~200 tasks      ~5MB
Revenue events                  ~100-500        ~1MB
Git commits                     ~50-100         ~10MB
Browser scraping artifacts      ~500 pages      ~100MB raw → ~10MB extracted
Monitoring metrics              ~100K points    ~20MB
Agent memory writes             ~200 entries    ~2MB
Audit trail                     ~10K events     ~10MB
────────────────────────────────────────────────────────
TOTAL                                           ~210MB/day → ~6.3GB/month
```

### Cost Flow Map

```
                    ┌──────────────────────┐
                    │   Revenue Sources     │
                    │   (Stripe, ads, etc.) │
                    └──────────┬───────────┘
                               │ $$$
                               ▼
┌──────────────────────────────────────────────────────────┐
│                    PORTFOLIO ENGINE                       │
│              Tracks per-product P&L                       │
└──────┬───────────────┬───────────────┬───────────────────┘
       │               │               │
       ▼               ▼               ▼
  ┌─────────┐   ┌───────────┐   ┌──────────────┐
  │ LLM API │   │   Infra   │   │  Marketing   │
  │  Costs  │   │   Costs   │   │    Spend     │
  │ $1-5/day│   │ $5-20/day │   │ $10-50/day   │
  └────┬────┘   └─────┬─────┘   └──────┬───────┘
       │              │                │
       ▼              ▼                ▼
  Smart Router   Railway/CF/       Social APIs
  → Provider     Vercel billing    Ad platforms
  billing APIs   APIs              Resend email
       │              │                │
       ▼              ▼                ▼
  ┌────────────────────────────────────────────┐
  │            TimescaleDB                      │
  │   llm_calls + infra_costs + growth_events  │
  │            ↓                                │
  │   Continuous aggregates → Grafana           │
  │   Daily P&L views → Dash reports            │
  └────────────────────────────────────────────┘
```

---

## Architectural Gaps Identified

### 1. No Event-Driven Customer Complaint Path (Currently)

The architecture docs describe email webhooks → Ally routing, but there's no explicit webhook receiver service documented. The NATS event bus ([11]) is designed but the ingestion from external email → bus is hand-wavy.

**Recommendation:** Build a thin webhook receiver service (Cloudflare Worker) that normalizes inbound emails/support tickets → NATS events. This is a critical gap for Scenario 3.

### 2. Approval Batching Not Implemented

Doc [12] mentions batched review but no mechanism exists to group related approvals. In Scenario 3, Adam gets 3 separate WhatsApp messages (ack email, resolution email, fix deploy) instead of one batched approval.

**Recommendation:** Implement approval bundling — group approvals by correlationId within a 60-second window before sending to human.

### 3. Memory Cold-Start Problem

Scenario 2's opportunity scoring (Step 14) relies on historical outcome data in Qdrant. On day 1, this is empty. The scoring model will have no calibration data.

**Recommendation:** Bootstrap Memory with manual entries of 10-20 known SaaS outcomes (from Adam's experience or public case studies) before the first revenue engine cycle.

### 4. No Circuit Breaker Between Build Agents and Smart Router

If the Smart Router's primary model (Sonnet) goes down during a build phase (Scenario 1, Step 23), each of the 5-15 LLM calls will independently fail and retry. The circuit breaker ([02]) is per-model, but the agent doesn't know to pause its work during an outage.

**Recommendation:** Expose circuit breaker state as an MCP resource. Agents query `router://health/{model}` before starting multi-call workflows. If degraded, agent waits or switches strategy proactively.

### 5. Post-Mortem → Self-Improvement Gap

Doc [19] describes the self-improvement signal pipeline, but doc [12] (human approval) and doc [11] (communication bus) don't reference it. Post-mortem outputs (Scenario 3, Step 26) need an explicit path into the Signal Store.

**Recommendation:** Add a NATS topic `signal.incident.postmortem` that the Post-Mortem Agent publishes to, consumed by the Signal Extractor in [19].

---

## Summary

The factory's data flows are coherent but reveal predictable bottlenecks:

1. **LLM latency is the universal tax** — every decision, every transform, every generation adds 2-8s. Parallelism and caching are the primary mitigations.
2. **Human gates are the wildcard** — they can be 0s (pre-approved) or 4h (sleeping). Graduated trust is the long-term fix; pre-authorized playbooks are the short-term fix.
3. **The system is write-heavy, read-light** — most data flows are append-only (events, logs, artifacts). Reads are concentrated in Kev's routing decisions and Dash's analytics. This is a good fit for TimescaleDB + Redis.
4. **Cross-component coordination happens via filesystem + task queue** — this works at current scale (<50 tasks/day) but will need NATS or PostgreSQL NOTIFY at scale.
5. **Cost tracking is well-instrumented** — every LLM call flows through the Smart Router with budget enforcement. The weakest link is infrastructure cost tracking (polling billing APIs daily vs real-time).

The architecture is sound for Phase 1-2 operation. The gaps identified above should be addressed before Phase 3 (autonomous revenue engine at scale).

---

*Review document — Data Flow Trace Analysis*  
*Reviewer: Atlas | Date: 2026-02-08*
