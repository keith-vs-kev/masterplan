# 10 — Revenue Engine & Business Generation Pipeline

> Architecture for the autonomous system that scans markets, builds products, generates revenue, and manages a portfolio of businesses — with humans as portfolio managers, not operators.

**Status:** Architecture Document
**Date:** 2026-02-08
**Depends on:** Research 11 (Revenue Generation), 16 (Marketing Automation), 20 (Endgame Vision)

---

## 1. Design Philosophy

Three principles govern the revenue engine:

1. **Businesses are experiments, the factory is the asset.** Any individual product may fail. The system that generates, tests, and scales them is what compounds.
2. **Kill fast, scale slow.** Every business gets a fixed budget and time window. No sentiment. If the numbers don't work by the gate, it dies. Winners earn the right to more resources gradually.
3. **Revenue before perfection.** Ship ugly, validate demand, then polish. A $500/mo product live today beats a perfect product in 3 months.

---

## 2. End-to-End Pipeline Overview

```
┌─────────────────────────────────────────────────────────────────────┐
│                        REVENUE ENGINE                               │
│                                                                     │
│  ┌──────────┐  ┌──────────┐  ┌──────┐  ┌───────┐  ┌─────────┐    │
│  │  SCAN    │→ │ VALIDATE │→ │ BUILD│→ │LAUNCH │→ │  GROW   │    │
│  │          │  │          │  │      │  │       │  │         │    │
│  │ Markets  │  │ Demand   │  │ Code │  │Deploy │  │Market   │    │
│  │ Signals  │  │ Test     │  │Design│  │Launch │  │Optimize │    │
│  │ Gaps     │  │ Score    │  │ Test │  │Seed   │  │Support  │    │
│  └──────────┘  └──────────┘  └──────┘  └───────┘  └─────────┘    │
│       ↑                                                  │         │
│       │            ┌──────────────────┐                  │         │
│       └────────────│ PORTFOLIO ENGINE │←─────────────────┘         │
│                    │                  │                             │
│                    │ Measure → Decide │                             │
│                    │ Kill ←→ Scale    │                             │
│                    │ Reinvest         │                             │
│                    └──────────────────┘                             │
└─────────────────────────────────────────────────────────────────────┘
```

The pipeline is a continuous loop. The portfolio engine feeds learnings back into scanning, improving opportunity selection over time.

---

## 3. Stage 1: Market Scanning & Opportunity Discovery

### Purpose
Continuously identify underserved markets, unmet needs, and exploitable gaps before they become crowded.

### Agent: Market Scanner

**Input signals (monitored continuously):**

| Source | Signal Type | What We Extract |
|--------|------------|-----------------|
| Reddit (r/SideProject, r/SaaS, niche subs) | Pain points, feature requests | "I wish X existed", "X is too expensive" |
| Twitter/X | Complaints, product gaps | Threads about broken workflows |
| Hacker News | Technical needs, tool gaps | "Ask HN: How do you handle X?" |
| Product Hunt | Launch gaps, feature requests | Comments on launches asking for missing features |
| G2 / Capterra reviews | Competitor weaknesses | 1-3 star reviews with specific complaints |
| Google Trends | Rising demand | Upward keyword trajectories |
| App Store reviews | Mobile opportunity gaps | Unmet needs in existing apps |
| Job postings | Emerging tool needs | New job titles = new tool categories |
| Freelance platforms (Upwork/Fiverr) | Recurring manual work | High-volume gig types ripe for automation |
| Expired domains / Flippa | Undervalued assets | Failed businesses with residual traffic or SEO value |

**Output:** Structured opportunity objects:

```typescript
interface Opportunity {
  id: string;
  title: string;
  problem: string;                    // The pain point in one sentence
  evidence: Signal[];                 // Links, quotes, data backing the opportunity
  category: BusinessCategory;         // micro-saas | api | content-site | digital-product | extension
  estimatedTAM: Range;               // Monthly revenue ceiling estimate
  competitorAnalysis: Competitor[];   // Existing solutions and their weaknesses
  buildComplexity: 1 | 2 | 3 | 4 | 5;
  timeToRevenue: Days;
  monetizationModel: string;
  confidenceScore: number;           // 0-100, weighted composite
  signals: {
    searchVolume: number;
    redditMentions: number;
    competitorWeakness: number;       // 0-100
    trendDirection: 'rising' | 'flat' | 'declining';
  };
}
```

### Scanning Cadence
- **Continuous:** Reddit, Twitter, HN monitored via streaming/polling
- **Daily:** Google Trends, competitor review aggregation
- **Weekly:** Deep competitor analysis, freelance platform mining, expired domain scan
- **Monthly:** Full market landscape review, category rotation

### Anti-Patterns to Avoid
- Don't chase trends that peaked (by the time it's on mainstream Twitter, it's too late)
- Don't target markets requiring regulatory compliance (healthcare, finance, legal — initially)
- Don't build for other AI builders (saturated, race to bottom)
- Don't compete with well-funded incumbents on features; compete on simplicity or niche focus

---

## 4. Stage 2: Idea Validation

### Purpose
Cheaply and quickly determine if an opportunity has real demand before investing build resources.

### Validation Pipeline

```
Opportunity
    │
    ▼
┌─────────────────────┐
│ AUTOMATED CHECKS     │  ← 2 hours, zero cost
│ • Search volume      │
│ • Competitor pricing │
│ • Keyword difficulty │
│ • Market size est.   │
└──────────┬──────────┘
           │ Score ≥ 60?
           ▼
┌─────────────────────┐
│ LANDING PAGE TEST    │  ← 4 hours, ~$50
│ • Generate LP        │
│ • Value prop + CTA   │
│ • Waitlist or        │
│   "Pre-order" button │
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│ TRAFFIC TEST         │  ← 48 hours, ~$100-200
│ • Post to 3 relevant │
│   communities        │
│ • Small paid test    │
│   ($50 Google/Reddit)│
│ • Direct outreach to │
│   10 potential users │
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│ GO / NO-GO GATE     │
│                     │
│ PASS if ANY of:     │
│ • 50+ waitlist      │
│ • >5% CTR on price  │
│ • 3+ "take my money"│
│   responses         │
│ • >2% conversion on │
│   paid traffic      │
│                     │
│ KILL otherwise.     │
└─────────────────────┘
```

### Validation Budget
- **Per opportunity:** Max $250 and 72 hours
- **Monthly budget:** Test 6-8 opportunities → ~$1,500-2,000/month on validation
- **Expected pass rate:** 25-35% proceed to build

### What the Landing Page Contains
The validation agent generates:
- Headline addressing the core pain point
- 3-5 bullet points of promised value
- Social proof (if applicable — "Join 50+ beta testers")
- Email capture or Stripe pre-order ($1 hold or full price with refund guarantee)
- Simple design via Framer AI, v0, or template

### Human Gate
- Opportunities scoring >80 with estimated build time <3 days: **auto-approved**
- Everything else: human reviews validation data and makes go/no-go call
- Goal: reduce human decisions to <5/month as confidence calibration improves

---

## 5. Stage 3: Product Requirements & Design

### Purpose
Translate a validated opportunity into a buildable specification with maximum constraint.

### Agent: Blueprint Generator

**Input:** Validated opportunity + landing page learnings
**Output:** Business Blueprint

```typescript
interface BusinessBlueprint {
  // Product
  name: string;
  tagline: string;
  coreFeature: string;              // ONE thing it does. Not three.
  mvpScope: Feature[];              // Max 5 features for v1
  explicitlyExcluded: string[];     // Features we are NOT building
  techStack: TechStack;
  
  // Business
  pricingModel: PricingModel;
  prices: PriceTier[];
  targetCustomer: string;
  acquisitionChannels: Channel[];
  
  // Operations
  estimatedBuildTime: Days;
  monthlyInfraCost: Dollars;
  breakEvenUsers: number;
  
  // Kill criteria
  day30Target: Metric;              // e.g., "$200 MRR" or "100 free users"
  day90Target: Metric;              // e.g., "$1,000 MRR"
  maxInvestment: Dollars;           // Total spend before forced kill
}
```

### Design Principles for Agent-Built Products

1. **One feature, done well.** The MVP does exactly one thing. Users should understand it in 5 seconds.
2. **Proven tech stack.** Don't innovate on infrastructure. Use Next.js/SvelteKit + Supabase + Stripe + Vercel. Every time.
3. **Pricing from day one.** No free tiers on v1. Charge $9-49/mo. Free plan only after product-market fit is proven.
4. **Self-serve only.** No sales calls, no demos, no onboarding meetings. If it can't sell itself, it's not simple enough.
5. **Built-in analytics.** Every product ships with event tracking (PostHog/Plausible) and revenue tracking from day one.

### Standard Tech Stack (Default)

| Layer | Choice | Why |
|-------|--------|-----|
| Frontend | Next.js 14+ (App Router) | Agents know it well, massive ecosystem |
| Backend | Next.js API routes + Supabase | Minimal surface area |
| Database | Supabase (Postgres) | Auth + DB + storage in one |
| Auth | Supabase Auth or Clerk | Zero-config, social logins |
| Payments | Stripe | Industry standard, agent-friendly API |
| Hosting | Vercel | Git-push deploys, edge network |
| Email | Resend | Simple API, transactional + marketing |
| Analytics | PostHog | Self-serve, event tracking, funnels |
| Monitoring | Sentry + BetterStack | Error tracking + uptime |
| Domain | Cloudflare | DNS + CDN + DDoS protection |

Deviations require justification. The whole point is that agents build faster on a familiar stack.

---

## 6. Stage 4: Build

### Purpose
Go from blueprint to deployed, functional product in 2-7 days.

### Build Agent Swarm

```
Blueprint
    │
    ▼
┌──────────────────┐
│ ARCHITECT AGENT   │  Decomposes into tasks, sets up repo, CI/CD
└────────┬─────────┘
         │
    ┌────┴────┬──────────┬──────────┐
    ▼         ▼          ▼          ▼
┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐
│FRONTEND│ │BACKEND │ │ COPY   │ │DESIGN  │
│ AGENT  │ │ AGENT  │ │ AGENT  │ │ AGENT  │
│        │ │        │ │        │ │        │
│UI/UX   │ │API     │ │Landing │ │Brand   │
│Pages   │ │DB      │ │Docs    │ │Assets  │
│Forms   │ │Auth    │ │Emails  │ │OG imgs │
│        │ │Billing │ │Help    │ │Favicon │
└────┬───┘ └────┬───┘ └───┬────┘ └───┬────┘
     │          │         │          │
     ▼          ▼         ▼          ▼
┌──────────────────────────────────────────┐
│            INTEGRATION AGENT              │
│  Merges work, resolves conflicts,        │
│  runs full test suite                    │
└────────────────┬─────────────────────────┘
                 │
                 ▼
┌──────────────────────────────────────────┐
│              QA AGENT                     │
│  • E2E tests (critical paths)            │
│  • Auth flow works                       │
│  • Payment flow works                    │
│  • Core feature works                    │
│  • No console errors                     │
│  • Mobile responsive                     │
│  • Security basics (no exposed keys,     │
│    HTTPS, CORS, rate limiting)           │
└────────────────┬─────────────────────────┘
                 │ All green?
                 ▼
┌──────────────────────────────────────────┐
│            DEPLOY AGENT                   │
│  • Push to production                    │
│  • Custom domain + SSL                   │
│  • Monitoring + alerting                 │
│  • Seed data if needed                   │
│  • Smoke test live environment           │
└──────────────────────────────────────────┘
```

### Build Budget & Timelines

| Business Type | Target Build Time | Compute Budget |
|--------------|-------------------|----------------|
| Browser extension | 1-2 days | $50-100 |
| API service | 2-3 days | $75-150 |
| Micro-SaaS | 3-5 days | $150-300 |
| Content/SEO site | 2-3 days | $100-200 |
| Digital product | 1-2 days | $50-100 |

### Quality Gates (Non-Negotiable)

Before any product goes live:
- [ ] Core user flow works end-to-end (signup → use → pay)
- [ ] Stripe integration tested with test mode
- [ ] No hardcoded secrets in codebase
- [ ] Error monitoring active (Sentry)
- [ ] Uptime monitoring active (BetterStack)
- [ ] Basic rate limiting on API endpoints
- [ ] Privacy policy and ToS pages exist
- [ ] Transactional emails work (welcome, receipt)
- [ ] Analytics tracking fires on key events

### What We Don't Do in V1
- No admin dashboard (check Stripe + Supabase directly)
- No user settings page (unless core to product)
- No blog (comes in growth phase)
- No multi-language support
- No mobile app
- No fancy animations or transitions

---

## 7. Stage 5: Launch & Initial Distribution

### Purpose
Get the product in front of its target audience and acquire first users within 7 days of deploy.

### Launch Playbook (Templated, Agent-Executed)

**Day 0 (Launch Day):**
```
├── Product Hunt submission (if applicable)
│   ├── Maker comment with story
│   ├── 5 screenshots + demo GIF
│   └── First-day-only discount
├── Hacker News "Show HN" post
├── Reddit posts in 2-3 relevant subreddits
│   └── Focus on the problem, not the product
├── Twitter/X launch thread
│   ├── Problem → solution → demo → link
│   └── Tag relevant accounts
└── Direct outreach
    ├── 20 emails to people who expressed the pain point
    └── 10 DMs to relevant community members
```

**Days 1-7:**
```
├── Respond to ALL comments and feedback
├── Post in 5 directories (BetaList, SaaSHub, AlternativeTo, etc.)
├── Submit to 3 newsletter features (relevant niche newsletters)
├── Write 2 "building in public" posts
└── Engage in 5 relevant community threads (provide value, soft mention)
```

**Days 7-14:**
```
├── Analyze which channels drove signups
├── Double down on top 2 channels
├── Start SEO content (3 articles targeting long-tail keywords)
├── Set up Google Search Console
└── First small paid experiment ($50) on best-performing channel
```

### Launch Budget
- **Per product:** $100-300 for launch phase
- **Breakdown:** $50-100 community promotion, $50-100 paid test, $50-100 directory/tool costs

### What "Good" Looks Like After Launch

| Metric | Day 7 Target | Day 30 Target |
|--------|-------------|---------------|
| Signups | 50+ | 200+ |
| Paid users | 3+ | 15+ |
| MRR | $50+ | $300+ |
| Churn (monthly) | N/A | <15% |
| Support tickets | Manageable by agent | Manageable by agent |

These are minimums for continued investment. Missing all of them by day 30 = kill candidate.

---

## 8. Stage 6: Growth & Marketing Engine

### Purpose
Scale products that survived launch into sustainable revenue streams.

### The Autonomous Marketing Stack

Products that pass the day-30 gate get a dedicated growth agent with three engines:

#### Engine 1: SEO & Content (Long-term compounding)
```
Keyword Research Agent
    │
    ├── Identify 50-100 long-tail keywords (low difficulty, commercial intent)
    ├── Cluster into topic groups
    └── Prioritize by traffic potential × conversion likelihood
         │
         ▼
Content Production Agent
    │
    ├── Generate 5-10 articles/week per product
    ├── Each article: 1,500-2,500 words, naturally optimized
    ├── Include product mentions where relevant (not forced)
    ├── Publish to product blog
    └── Syndicate to Medium, Dev.to, relevant platforms
         │
         ▼
Link Building Agent
    │
    ├── HARO responses (3-5/week)
    ├── Guest post pitching (2-3/week)
    ├── Directory submissions (ongoing)
    └── Broken link reclamation
```

**Timeline to results:** 2-4 months for meaningful organic traffic. This is a compounding asset.

#### Engine 2: Paid Acquisition (Fast feedback)
```
Experiment Agent
    │
    ├── Create 10-20 ad variants (copy + creative)
    ├── Deploy on Google Ads + Meta Ads (small budget: $10-20/day)
    ├── Run for 7 days minimum
    ├── Kill ads below ROAS threshold
    ├── Scale ads above ROAS threshold
    └── Feed learnings back into landing page optimization
         │
         ▼
Landing Page Optimizer
    │
    ├── A/B test headlines (biggest lever)
    ├── A/B test CTAs
    ├── A/B test pricing display
    └── Implement winner, start next test
```

**Budget:** $300-500/month per product initially. Scale to $2,000+/month only with proven ROAS > 3x.

#### Engine 3: Community & Organic (Relationship capital)
```
Community Agent
    │
    ├── Monitor relevant subreddits, forums, Slack/Discord groups
    ├── Provide genuine help (answer questions, share expertise)
    ├── Soft-mention product ONLY when directly relevant
    ├── Share user success stories
    └── Build reputation in 3-5 key communities
```

**This engine is the hardest to automate well.** Communities detect and punish inauthentic participation. The agent must provide genuine value 90% of the time and promote 10%.

### Retention & Expansion

Once a product has 50+ paying users:
```
Churn Analysis Agent
    │
    ├── Track who churns and why (cancellation survey, usage patterns)
    ├── Identify at-risk users (declining usage, support tickets)
    ├── Trigger win-back sequences (email with discount or new features)
    └── Feed churn reasons into feature prioritization
         │
         ▼
Feature Expansion Agent
    │
    ├── Analyze feature requests (support tickets, reviews, social mentions)
    ├── Score by: request frequency × implementation effort × revenue impact
    ├── Build top-scored features (max 1 new feature per 2-week cycle)
    └── Announce via email + changelog
         │
         ▼
Upsell Agent
    │
    ├── Identify power users on free/low tier
    ├── Create premium features that serve power user needs
    ├── Test pricing tiers
    └── Implement usage-based triggers for upgrade prompts
```

---

## 9. Stage 7: Measure & Portfolio Management

### Purpose
The portfolio engine is the brain of the factory. It decides what lives, what dies, and where resources go.

### Business Lifecycle States

```
VALIDATING → BUILDING → LAUNCHED → GROWING → MATURE → DECLINING
                                      ↓                    ↓
                                   SCALING              SUNSETTING
                                      ↓
                                   CASH COW
```

Every business has a state. Transitions are triggered by metrics hitting thresholds.

### Kill Criteria (Automatic)

A business is flagged for termination if:

| Timeframe | Kill If |
|-----------|---------|
| Day 30 | $0 revenue AND <50 signups |
| Day 60 | <$100 MRR AND declining signups |
| Day 90 | <$500 MRR AND no growth trajectory |
| Day 180 | <$1,000 MRR |
| Any time | Monthly burn > 2x monthly revenue for 3 consecutive months |
| Any time | Zero active users for 14 consecutive days |

**Kill process:**
1. Agent flags the business with data
2. If below auto-kill threshold: immediately sunset (shut down infra, redirect domain to portfolio page)
3. If borderline: human gets a 1-page brief with recommendation. 24-hour response window. No response = auto-kill.
4. Killed businesses are archived: codebase saved, learnings documented, domain retained if valuable

### Scale Criteria

A business earns more resources if:

| Signal | Action |
|--------|--------|
| >$1,000 MRR with >20% month-over-month growth | Increase marketing budget 2x |
| >$3,000 MRR with <10% churn | Add paid acquisition channel |
| >$5,000 MRR | Assign dedicated growth agent |
| >$10,000 MRR | Consider human attention for strategic decisions |
| >$20,000 MRR | Evaluate if worth spinning into standalone entity |

### Portfolio Health Dashboard

The portfolio engine maintains a real-time dashboard:

```
┌─────────────────────────────────────────────────────┐
│  PORTFOLIO DASHBOARD                                 │
├─────────────────────────────────────────────────────┤
│                                                      │
│  Total MRR:     $34,200     (+12% MoM)              │
│  Products:      14 live │ 3 building │ 2 validating │
│  Margin:        82%                                  │
│  Runway:        ∞ (profitable)                       │
│                                                      │
│  TOP PERFORMERS                                      │
│  ┌──────────────────┬────────┬───────┬────────┐     │
│  │ Product          │ MRR    │ Trend │ Status │     │
│  ├──────────────────┼────────┼───────┼────────┤     │
│  │ NicheAPI Pro     │ $8,200 │  ↑18% │ SCALE  │     │
│  │ SEODash          │ $6,100 │  ↑12% │ GROW   │     │
│  │ FormBot          │ $4,800 │  ↑ 8% │ GROW   │     │
│  │ DataPipe         │ $3,500 │  → 2% │ MATURE │     │
│  └──────────────────┴────────┴───────┴────────┘     │
│                                                      │
│  KILL CANDIDATES                                     │
│  ┌──────────────────┬────────┬───────┬────────┐     │
│  │ QuickInvoice     │   $120 │  ↓15% │ DAY 87 │     │
│  │ PodClip          │    $45 │  ↓30% │ DAY 62 │     │
│  └──────────────────┴────────┴───────┴────────┘     │
│                                                      │
│  MONTHLY COSTS                                       │
│  Agent compute:   $1,200                             │
│  Infrastructure:  $840                               │
│  Marketing:       $3,400                             │
│  Total:           $5,440                             │
│                                                      │
│  NET MARGIN:      $28,760/mo                         │
└─────────────────────────────────────────────────────┘
```

### Portfolio Diversification Rules

Don't put all eggs in one basket:

- **Max 3 products in the same category** (e.g., max 3 SEO tools)
- **Max 40% of revenue from one product** (if exceeded, accelerate other products)
- **Max 50% of revenue from one acquisition channel** (if all revenue is from Google, diversify)
- **Mix business models:** some subscription, some usage-based, some one-time purchase
- **Mix build complexity:** always have some quick-ship products alongside ambitious ones

### Reinvestment Logic

```
Monthly Profit
    │
    ├── 40% → Growth budget for existing winners
    │         (marketing, paid ads, feature development)
    │
    ├── 30% → New business experiments
    │         (validation + build budget for new products)
    │
    ├── 20% → Infrastructure & compute
    │         (better models, more agent capacity, tooling)
    │
    └── 10% → Reserve
              (emergency fund, opportunistic acquisitions)
```

Reinvestment ratios shift as the portfolio matures. Early stage: heavier on new experiments. Mature: heavier on scaling winners.

---

## 10. The Agent Roster

### Core Agents (Always Running)

| Agent | Role | Runs |
|-------|------|------|
| **Market Scanner** | Monitors signals, generates opportunities | Continuous |
| **Portfolio Manager** | Tracks all businesses, triggers kill/scale | Daily |
| **Finance Agent** | Revenue tracking, cost monitoring, P&L | Daily |
| **Alert Agent** | Downtime, errors, anomalies across all products | Continuous |

### Per-Business Agents (Spawned as Needed)

| Agent | Role | Active During |
|-------|------|--------------|
| **Validation Agent** | Landing page, traffic tests | Validation stage |
| **Blueprint Agent** | PRD, tech spec, architecture | Design stage |
| **Build Swarm** (4-5 agents) | Frontend, backend, copy, design, QA | Build stage |
| **Launch Agent** | Distribution, community posts, outreach | Launch stage |
| **Growth Agent** | SEO, paid, community, retention | Growth stage |
| **Support Agent** | Customer tickets, FAQ, onboarding | Post-launch ongoing |
| **Maintenance Agent** | Bug fixes, dependency updates, security patches | Post-launch ongoing |

### Agent Coordination

Agents communicate through:
1. **Shared state store** — each business has a state object with current metrics, status, and pending actions
2. **Event bus** — agents emit events (e.g., "build complete", "MRR crossed $1K") that trigger downstream agents
3. **Human escalation queue** — issues that exceed agent authority get queued for human review

---

## 11. Financial Model

### Unit Economics Per Business

| Phase | Duration | Cost | Expected Outcome |
|-------|----------|------|-----------------|
| Validation | 3 days | $250 | 25-35% pass rate |
| Build | 2-7 days | $100-300 | 90% deploy successfully |
| Launch | 14 days | $100-300 | 60% get first users |
| Growth (month 1-3) | 90 days | $500-1,500 | 30% reach $500+ MRR |

**Cost to produce one $500+ MRR business:**
- Test 4 ideas: 4 × $250 = $1,000
- Build 1.5 (some fail in build): 1.5 × $200 = $300
- Launch & grow: $1,000
- **Total: ~$2,300 to get one product to $500/mo**
- **Payback: ~5 months**

### 12-Month Projection (Conservative)

| Month | Products Launched | Products Live | Total MRR | Monthly Cost | Net |
|-------|------------------|---------------|-----------|-------------|-----|
| 1 | 2 | 1 | $200 | $2,000 | -$1,800 |
| 2 | 2 | 2 | $600 | $2,500 | -$1,900 |
| 3 | 2 | 3 | $1,500 | $3,000 | -$1,500 |
| 4 | 2 | 4 | $3,000 | $3,500 | -$500 |
| 5 | 2 | 5 | $5,500 | $4,000 | $1,500 |
| 6 | 2 | 6 | $8,000 | $4,500 | $3,500 |
| 7 | 2 | 6 | $12,000 | $5,000 | $7,000 |
| 8 | 2 | 7 | $16,000 | $5,500 | $10,500 |
| 9 | 2 | 8 | $21,000 | $6,000 | $15,000 |
| 10 | 2 | 8 | $27,000 | $6,500 | $20,500 |
| 11 | 2 | 9 | $33,000 | $7,000 | $26,000 |
| 12 | 2 | 9 | $40,000 | $7,500 | $32,500 |

**Assumptions:** 2 launches/month, 35% survival past month 3, survivors grow 15-25% MoM, 1-2 breakout products. Products killed at month 3 if below threshold (hence "products live" doesn't always increase).

**Break-even: Month 5. $40K MRR by month 12.**

---

## 12. Business Type Playbooks

### Playbook A: Micro-SaaS
- **Build time:** 3-5 days
- **Target MRR:** $1K-$10K
- **Pricing:** $9-49/month
- **Distribution:** SEO + communities + small paid
- **Example:** Niche dashboard, workflow tool, single-purpose utility
- **Kill signal:** <$300 MRR at day 90

### Playbook B: API Service
- **Build time:** 2-3 days
- **Target MRR:** $1K-$15K
- **Pricing:** Usage-based ($0.001-$1.00/call) or tiers
- **Distribution:** RapidAPI marketplace + developer communities + docs SEO
- **Example:** Data enrichment, text processing, specialized AI wrapper
- **Kill signal:** <100 API calls/day at day 60

### Playbook C: Content/SEO Site
- **Build time:** 2-3 days (site) + ongoing content
- **Target MRR:** $500-$5K
- **Pricing:** Ads (AdSense/Mediavine) + affiliate
- **Distribution:** Pure SEO, programmatic content at scale
- **Example:** Comparison site, niche directory, calculator tools
- **Kill signal:** <1,000 organic sessions/month at day 120

### Playbook D: Digital Product
- **Build time:** 1-2 days
- **Revenue:** One-time purchases, $19-99
- **Distribution:** Gumroad/LemonSqueezy + SEO + communities
- **Example:** Templates, e-books, prompt packs, datasets
- **Kill signal:** <10 sales in first 30 days

### Playbook E: Browser Extension
- **Build time:** 1-3 days
- **Target MRR:** $500-$5K
- **Pricing:** Freemium, $5-15/month premium
- **Distribution:** Chrome Web Store SEO + communities
- **Example:** Productivity tools, page enhancers, data extractors
- **Kill signal:** <100 installs at day 30

---

## 13. The Learning Loop

The factory gets smarter over time. Every business — success or failure — generates data.

### What We Track Per Business

```typescript
interface BusinessOutcome {
  id: string;
  category: BusinessCategory;
  niche: string;
  
  // Discovery
  sourceSignals: Signal[];           // What triggered the idea
  validationScore: number;           // Pre-build confidence
  validationMethod: string;          // How we tested demand
  
  // Build
  buildDays: number;
  buildCost: number;
  techStack: string[];
  buildIssues: string[];             // What went wrong
  
  // Launch
  launchChannels: Channel[];
  day7Signups: number;
  day7Revenue: number;
  bestChannel: Channel;
  
  // Growth
  day30MRR: number;
  day90MRR: number;
  day180MRR: number;
  peakMRR: number;
  churnRate: number;
  
  // Outcome
  status: 'killed' | 'active' | 'scaled' | 'sold';
  totalRevenue: number;
  totalCost: number;
  roi: number;
  
  // Learnings
  whatWorked: string[];
  whatFailed: string[];
  wouldDoDifferently: string[];
}
```

### Feedback Loops

1. **Opportunity scoring improves** — historical success rates per category, niche, and signal type calibrate the confidence scorer
2. **Build estimates improve** — actual build times feed back into estimation
3. **Launch playbooks improve** — which channels work for which product types
4. **Kill criteria calibrate** — are we killing too early? Too late? Adjust thresholds based on outcomes
5. **Pricing intelligence** — what price points work in which markets

### Monthly Learning Review

The portfolio manager agent produces a monthly report:
- Businesses launched, killed, and scaled
- Win rate (% of launches reaching $500 MRR)
- Average time to $500 MRR
- Best and worst performing categories
- Channel effectiveness by product type
- Updated playbook recommendations

---

## 14. Human Touchpoints

Even at maximum autonomy, humans remain in the loop for:

| Decision | Why Human | Frequency |
|----------|-----------|-----------|
| Portfolio strategy | High-level direction, risk appetite | Monthly |
| Kill/scale borderline cases | Judgment calls on ambiguous data | Weekly |
| Brand/voice calibration | Authenticity matters for community trust | Per product launch |
| Legal/compliance review | Agents can't assess regulatory risk well | Per product launch |
| Financial review | Verify P&L, tax implications | Monthly |
| Crisis response | Data breach, PR issue, platform ban | As needed |

**Target:** Human spends <5 hours/week on the portfolio as it matures.

### Escalation Protocol

```
Agent encounters issue
    │
    ├── Severity LOW (minor bug, routine question)
    │   └── Agent handles autonomously, logs for review
    │
    ├── Severity MEDIUM (feature decision, borderline kill/scale)
    │   └── Agent queues for human review with recommendation
    │       Human has 24h to respond. No response = agent proceeds with recommendation.
    │
    └── Severity HIGH (security issue, legal concern, platform ban)
        └── Immediate notification to human. Agent pauses affected operations.
```

---

## 15. Implementation Phases

### Phase 1: Manual Factory (Months 1-2)
- Build market scanner agent
- Human manually orchestrates first 2-3 products using the pipeline
- Document every decision for automation
- Establish infrastructure (Stripe, hosting, monitoring)
- **Deliverable:** 2-3 live products, validated pipeline

### Phase 2: Semi-Automated Factory (Months 3-4)
- Automate validation pipeline (landing page → traffic test → scoring)
- Automate build pipeline (blueprint → code → deploy)
- Automate launch playbook execution
- Human reviews at gates only
- **Deliverable:** Launch 2 products/month with <10 hours human time each

### Phase 3: Autonomous Factory (Months 5-8)
- Portfolio engine runs autonomously
- Kill/scale decisions automated (with human override)
- Growth agents operate independently
- Human role shifts to weekly portfolio review
- **Deliverable:** 8-12 live products, $10K+ MRR, <5 hours/week human time

### Phase 4: Scaling Factory (Months 9-12)
- Parallel build capacity (2-3 products building simultaneously)
- Cross-product synergies (shared audiences, cross-promotion)
- Factory learning loop producing measurably better outcomes
- Consider Factory-as-a-Service for meta-revenue
- **Deliverable:** $30K-50K MRR, self-sustaining growth

---

## 16. Risks & Mitigations

| Risk | Impact | Mitigation |
|------|--------|-----------|
| Stripe account termination | Critical — kills billing | Multiple Stripe accounts, backup processors (Paddle, LemonSqueezy). Each product uses own Stripe account. |
| Agent quality plateau | High — products don't meet minimum quality | Hybrid approach: agents do 80%, human reviews critical paths. Invest in better QA gates. |
| Platform bans (Google, hosting) | High — kills distribution | Multi-cloud (Vercel + Railway + Fly.io). Diversified traffic sources. No single dependency. |
| Market saturation | Medium — AI products flooding niches | Focus on underserved non-tech niches. Compete on simplicity, not features. |
| Cost overruns | Medium — compute costs exceed revenue | Strict per-business budgets. Kill fast. Use cheapest model that works per task. |
| Legal/regulatory | Medium — new AI disclosure laws | Maintain human-in-the-loop documentation. Comply proactively. |
| Reputation risk | Medium — low-quality products damage brand | Each product has independent branding. Quality gates before launch. Kill bad products quickly. |

---

## 17. Success Metrics

### Factory-Level KPIs

| Metric | Month 3 Target | Month 6 Target | Month 12 Target |
|--------|----------------|----------------|-----------------|
| Products live | 3-5 | 6-8 | 8-12 |
| Total MRR | $1,500 | $8,000 | $40,000 |
| Win rate (reach $500 MRR) | 20% | 30% | 40% |
| Avg time to first revenue | 14 days | 10 days | 7 days |
| Human hours/week | 20+ | 10 | <5 |
| Cost per successful product | $3,000 | $2,500 | $2,000 |
| Portfolio gross margin | 50% | 75% | 85% |

### The North Star

> A system that identifies a market opportunity on Monday, has a validated landing page by Tuesday, a deployed product by Friday, first paying users by next Monday, and a kill-or-scale decision within 30 days — with a human spending less than 1 hour per business per week.

---

*Architecture document: Revenue Engine & Business Generation Pipeline*
*Date: 2026-02-08*
