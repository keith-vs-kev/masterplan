# 16 — Marketing & Growth Automation Architecture

> How **Blaze** (marketing) and **Echo** (content) agents autonomously market factory-produced products — from strategy through distribution to ROI measurement.

---

## 1. Agent Roles & Responsibilities

### Blaze — Marketing Strategist & Distributor
- **Owns:** Channel strategy, campaign orchestration, paid ads, email campaigns, A/B testing, analytics, budget allocation
- **Decides:** Where to spend, what to test, when to scale/kill channels
- **Measures:** CAC, ROAS, conversion rates, channel ROI

### Echo — Content Creator & SEO Engine
- **Owns:** Content creation (blog, social, email copy, landing pages, video scripts), brand voice, SEO pipeline, content calendar
- **Decides:** What to write, tone/format per platform, keyword targeting
- **Measures:** Content quality scores, organic traffic, engagement rates, SEO rankings

### Coordination Model
```
┌─────────────────────────────────────────────────────────────┐
│                    Product Launch Event                       │
│            (from Factory Orchestrator / Forge)                │
└──────────────────────┬──────────────────────────────────────┘
                       │
                       ▼
              ┌────────────────┐
              │     Blaze      │
              │  (Strategist)  │
              └───────┬────────┘
                      │
         ┌────────────┼─────────────┐
         │            │             │
         ▼            ▼             ▼
   ┌──────────┐ ┌──────────┐ ┌──────────┐
   │  Echo    │ │  Blaze   │ │  Blaze   │
   │ (Content)│ │ (Ads)    │ │ (Email)  │
   └────┬─────┘ └────┬─────┘ └────┬─────┘
        │             │             │
        ▼             ▼             ▼
   ┌──────────────────────────────────────┐
   │         Distribution Layer           │
   │   (CMS, Social APIs, Ad Platforms,   │
   │    Email Providers, Landing Pages)   │
   └──────────────────┬──────────────────┘
                      │
                      ▼
              ┌────────────────┐
              │   Analytics    │
              │  Feedback Loop │
              └────────────────┘
```

**Blaze leads, Echo creates.** Blaze sets strategy (target audience, channels, budget, KPIs), then requests content from Echo. Echo produces assets matching brand voice and SEO requirements. Blaze distributes, measures, and feeds performance data back to both agents for iteration.

**Communication protocol:**
- Blaze → Echo: `ContentRequest` (brief, audience, channel, keywords, deadline)
- Echo → Blaze: `ContentDelivery` (assets, metadata, variant suggestions)
- Blaze → Echo: `PerformanceFeedback` (what worked, what didn't, iterate)

---

## 2. SEO Content Pipeline

Echo runs a continuous SEO engine targeting organic growth for every factory product.

### Pipeline Stages

```
1. Keyword Discovery     →  Ahrefs/SEMrush API + Google Search Console
2. Opportunity Scoring   →  Volume × (1/Difficulty) × Commercial Intent
3. Content Brief         →  Target keyword, outline, internal links, word count
4. Draft Generation      →  LLM (Claude/GPT-4) with brand voice context
5. SEO Optimization      →  Keyword density, readability, meta tags, schema
6. Review Gate           →  Automated quality check (grammar, factual, plagiarism)
7. Publish               →  Push to CMS (WordPress/Webflow API)
8. Index & Monitor       →  Google Indexing API, rank tracking
9. Refresh Cycle         →  Re-optimize underperforming content monthly
```

### Programmatic SEO (Scale Play)

For products with broad applicability, Echo generates templated pages:
- `[Product] vs [Competitor]` comparison pages
- `[Product] for [Industry/Use-Case]` landing pages
- `Best [Category] tools [Year]` listicles featuring our products
- `How to [solve problem]` tutorials with product integration

**Target:** 50-200 pages per product launch, long-tail keywords with <30 difficulty.

### Tools & APIs
| Function | Tool | Integration |
|----------|------|-------------|
| Keyword research | SEMrush API / DataForSEO | REST API |
| SERP analysis | Ahrefs API | REST API |
| Content optimization | Surfer SEO API | REST API |
| CMS publishing | WordPress REST API / Webflow API | REST API |
| Indexing | Google Indexing API | REST API |
| Rank tracking | SEMrush Position Tracking API | REST API |
| Plagiarism check | Copyscape API | REST API |

---

## 3. Social Media Automation

### Platform Strategy

| Platform | Content Type | Frequency | Agent |
|----------|-------------|-----------|-------|
| X/Twitter | Thread launches, tips, engagement | 2-3x/day | Echo creates, Blaze schedules |
| LinkedIn | Thought leadership, case studies | 1x/day | Echo creates, Blaze schedules |
| Reddit | Community participation, launches | As relevant | Blaze monitors, Echo drafts |
| YouTube | Tutorials, demos | 1-2x/week | Echo scripts, external render |
| TikTok/Shorts | Quick demos, tips | 3-5x/week | Echo scripts, AI video gen |

### Automation Flow

```
Blaze: Trend Monitor
  │
  ├── Scans trending topics (Twitter API, Google Trends API)
  ├── Identifies relevant conversations
  └── Triggers Echo content requests
         │
         ▼
Echo: Content Factory
  │
  ├── Generates platform-native content (thread vs carousel vs short-form)
  ├── Creates accompanying visuals (Midjourney/DALL-E API)
  ├── Adapts tone per platform
  └── Returns variants for A/B testing
         │
         ▼
Blaze: Distribution
  │
  ├── Schedules via Buffer API / native platform APIs
  ├── Posts at optimal times (learned from historical engagement data)
  ├── Monitors engagement, responds to comments (with guardrails)
  └── Feeds performance back to Echo
```

### APIs
- **X/Twitter:** X API v2 (post, engage, analytics)
- **LinkedIn:** LinkedIn Marketing API (organic + paid)
- **Reddit:** Reddit API (monitor, post with rate limits)
- **Buffer/Hootsuite:** Scheduling layer API
- **Google Trends:** Trends API for topic discovery

---

## 4. Email Campaign System

### Campaign Types (Automated)

| Type | Trigger | Owner |
|------|---------|-------|
| Welcome sequence | New signup | Blaze orchestrates, Echo writes |
| Product launch | New product deployed | Blaze orchestrates, Echo writes |
| Nurture drip | Time-based after signup | Blaze orchestrates, Echo writes |
| Re-engagement | 30-day inactivity | Blaze triggers, Echo writes |
| Weekly digest | Cron (weekly) | Echo auto-generates |
| Transactional | User action (purchase, etc.) | Automated templates |

### Pipeline

```
1. Segmentation    →  Blaze queries CRM/database for audience segments
2. Brief           →  Blaze creates campaign brief (goal, segment, CTA)
3. Copy Generation →  Echo writes subject lines (5 variants), body, CTA
4. Personalization →  Dynamic merge fields from CRM data
5. A/B Setup       →  Blaze selects test variables (subject, CTA, send time)
6. Send            →  SendGrid/Resend API with optimized send times
7. Monitor         →  Track opens, clicks, conversions, unsubscribes
8. Iterate         →  Winner deployed to remaining segment; learnings stored
```

### Tools
- **Email delivery:** SendGrid API / Resend API
- **CRM:** HubSpot API / internal customer database
- **Personalization:** Dynamic content engine (internal)
- **Analytics:** PostHog / Mixpanel for conversion tracking

---

## 5. Landing Page Generation

When a new product launches, Echo auto-generates landing pages.

### Generation Pipeline

```
1. Product Brief     →  From factory: name, features, value prop, target audience
2. Copy Generation   →  Echo writes: headline, subheadline, benefits, social proof, CTA
3. Layout Selection  →  Template library (hero + features + testimonials + CTA)
4. Page Build        →  v0/Vercel AI or Webflow API generates functional page
5. Variant Creation  →  3 headline variants, 2 CTA variants = 6 combinations
6. Deploy            →  Push to hosting (Vercel/Cloudflare Pages)
7. Connect Tracking  →  PostHog/GA4 events on all CTAs
8. A/B Test          →  Traffic split across variants
9. Converge          →  Winner selected after statistical significance
```

### Tech Stack
- **Page generation:** v0 by Vercel (React) / Framer AI / custom templates
- **Hosting:** Vercel / Cloudflare Pages
- **A/B testing:** PostHog feature flags / custom split logic
- **Analytics:** PostHog / GA4

---

## 6. A/B Testing Framework

Blaze runs continuous experimentation across all channels.

### What Gets Tested

| Channel | Variables | Method |
|---------|-----------|--------|
| Landing pages | Headlines, CTAs, layouts, hero images | Multi-armed bandit |
| Email | Subject lines, send times, CTA text | Traditional A/B |
| Ads | Copy, creative, audience, bidding | Platform-native (Meta/Google AI) |
| Social posts | Format, time, hashtags, tone | Sequential testing |
| Pricing | Price points, plan names, feature bundling | Bayesian A/B |

### Statistical Engine

```python
# Blaze's testing logic (simplified)
class ExperimentEngine:
    def should_conclude(self, experiment):
        """Bayesian approach — faster than frequentist"""
        posterior_a = beta(successes_a + 1, failures_a + 1)
        posterior_b = beta(successes_b + 1, failures_b + 1)
        prob_b_better = P(posterior_b > posterior_a)
        
        if prob_b_better > 0.95 or prob_b_better < 0.05:
            return True  # Confident enough to declare winner
        if total_samples > max_samples:
            return True  # Budget exhausted
        return False

    def pick_winner(self, experiment):
        """Deploy winner, log learnings, suggest next experiment"""
        winner = max(variants, key=lambda v: v.conversion_rate)
        self.deploy(winner)
        self.log_learning(experiment)
        self.suggest_next_experiment(experiment.learnings)
```

### Multi-Armed Bandit for Landing Pages

Instead of 50/50 splits, Blaze uses Thompson Sampling to allocate more traffic to winning variants during the test — maximizing conversions even during experimentation.

---

## 7. Content Calendar (Automated)

Echo maintains a rolling 90-day content calendar, auto-populated and adjusted based on performance.

### Calendar Structure

```yaml
content_calendar:
  generation: "Rolling 90-day, refreshed weekly"
  sources:
    - product_launches: "From factory pipeline — auto-creates launch content"
    - seo_opportunities: "From keyword research — fills gaps"
    - trending_topics: "From trend monitoring — opportunistic"
    - evergreen_refresh: "From rank tracking — update declining content"
    - campaign_themes: "Monthly themes set by Blaze strategy"
  
  daily_output:
    blog_posts: 1-2 (SEO-driven)
    social_posts: 5-8 (across platforms)
    email: 0-1 (campaign-dependent)
    
  weekly_output:
    landing_pages: 1-2 (for new products/campaigns)
    video_scripts: 2-3
    case_studies: 1
    
  monthly_output:
    strategy_review: "Blaze analyzes all channels, reallocates budget"
    content_audit: "Echo reviews all content performance, prunes/refreshes"
    calendar_regeneration: "Next 90 days re-planned based on learnings"
```

### Auto-Population Logic

```
On new product launch:
  → Echo queues: launch blog post, 5 social posts, email blast,
    landing page, 3 comparison pages, 1 tutorial

On keyword opportunity detected (volume >500, difficulty <30):
  → Echo queues: SEO article within 7 days

On content ranking drop (position 1-10 → 11-20):
  → Echo queues: content refresh within 14 days

On trending topic match:
  → Echo queues: reactive social post within 2 hours
```

---

## 8. ROI Measurement Per Channel

### Attribution Model

Blaze uses **multi-touch attribution** (position-based) with automated budget reallocation.

```
┌─────────────────────────────────────────────────┐
│           Channel Performance Dashboard          │
├─────────────┬──────┬───────┬──────┬─────────────┤
│ Channel     │ CAC  │ ROAS  │ LTV/ │ Budget      │
│             │      │       │ CAC  │ Action       │
├─────────────┼──────┼───────┼──────┼─────────────┤
│ Organic SEO │ $2   │ 50x   │ 25:1 │ ↑ Scale     │
│ Email       │ $0.5 │ 40x   │ 30:1 │ ↑ Scale     │
│ Social Org. │ $5   │ 10x   │ 8:1  │ → Maintain  │
│ Google Ads  │ $15  │ 4x    │ 3:1  │ → Optimize  │
│ Meta Ads    │ $20  │ 2x    │ 2:1  │ ↓ Reduce    │
│ Reddit      │ $8   │ 6x    │ 5:1  │ → Test more │
└─────────────┴──────┴───────┴──────┴─────────────┘
```

### Metrics Tracked Per Channel

| Metric | How Measured | Tool |
|--------|-------------|------|
| **CAC** (Customer Acquisition Cost) | Total channel spend / new customers attributed | PostHog + Stripe |
| **ROAS** (Return on Ad Spend) | Revenue from channel / spend on channel | PostHog + Stripe |
| **LTV:CAC** | Lifetime value / acquisition cost | Internal calculation |
| **Conversion Rate** | Signups or purchases / visitors from channel | PostHog |
| **Time to Convert** | First touch → purchase elapsed time | PostHog |
| **Content ROI** | Revenue attributed / content production cost | Internal |
| **Organic Traffic Value** | Equivalent CPC × organic clicks | SEMrush |

### Automated Budget Reallocation

```python
# Blaze rebalances weekly
def rebalance_budget(channels, total_budget):
    """Allocate more to high-ROI channels, less to low-ROI"""
    for channel in channels:
        channel.score = channel.roas * channel.growth_trend * channel.confidence
    
    # 70% to proven winners, 20% to promising channels, 10% to experiments
    proven = [c for c in channels if c.confidence > 0.8 and c.roas > 3]
    promising = [c for c in channels if c.confidence > 0.5 and c.roas > 1]
    experimental = [c for c in channels if c.confidence < 0.5]
    
    allocate(proven, total_budget * 0.70)
    allocate(promising, total_budget * 0.20)
    allocate(experimental, total_budget * 0.10)
```

---

## 9. Full Tool & API Inventory

### Content Creation
| Tool | Purpose | Cost |
|------|---------|------|
| Claude API / GPT-4 API | Core content generation | ~$0.01-0.10/article |
| Midjourney / DALL-E API | Visual asset generation | ~$0.02-0.10/image |
| ElevenLabs API | Voiceover for video | ~$0.01/minute |
| Runway API | AI video generation | ~$0.10-0.50/clip |

### SEO & Research
| Tool | Purpose | Cost |
|------|---------|------|
| SEMrush API | Keyword research, rank tracking | $120-450/mo |
| Ahrefs API | Backlink analysis, content gaps | $99-999/mo |
| Google Search Console API | Index status, performance | Free |
| DataForSEO API | Bulk SERP data | Pay-per-call |

### Distribution
| Tool | Purpose | Cost |
|------|---------|------|
| X API v2 | Twitter posting & analytics | Free-$5k/mo |
| LinkedIn Marketing API | Organic + paid LinkedIn | Free (organic) |
| SendGrid / Resend API | Email delivery | ~$20-100/mo |
| Google Ads API | Paid search campaigns | Ad spend variable |
| Meta Marketing API | Facebook/Instagram ads | Ad spend variable |
| Buffer API | Social scheduling | $6-120/mo |

### Analytics & Testing
| Tool | Purpose | Cost |
|------|---------|------|
| PostHog | Product analytics, A/B testing | Free-$450/mo |
| Google Analytics 4 | Web analytics | Free |
| Stripe API | Revenue tracking | Transaction-based |
| Plausible | Privacy-friendly analytics | $9-99/mo |

### Infrastructure
| Tool | Purpose | Cost |
|------|---------|------|
| Vercel | Landing page hosting | Free-$20/mo |
| Cloudflare Pages | Static site hosting | Free |
| WordPress API | Blog CMS | $4-45/mo |
| Webflow API | Marketing site CMS | $14-39/mo |

---

## 10. Agent Coordination Protocol

### Event-Driven Architecture

Blaze and Echo communicate via the factory's event bus:

```
Events:
  product.launched       → Blaze creates marketing plan, requests content from Echo
  content.ready          → Blaze distributes across channels
  campaign.performance   → Blaze analyzes, sends feedback to Echo
  keyword.opportunity    → Echo creates SEO content
  content.underperforming→ Echo refreshes, Blaze adjusts distribution
  budget.rebalanced      → Blaze shifts spend across channels
  experiment.concluded   → Blaze deploys winner, logs learning
```

### Decision Authority

| Decision | Agent | Escalation Threshold |
|----------|-------|---------------------|
| What content to create | Echo (guided by Blaze brief) | Never — autonomous |
| Where to distribute | Blaze | Never — autonomous |
| Budget <$100/day | Blaze | Never — autonomous |
| Budget >$100/day | Blaze | Requires human approval |
| Kill underperforming channel | Blaze | If spend >$500 total |
| Brand voice changes | Echo | Always requires human approval |
| Respond to PR crisis | Neither | Always escalate to human |
| New channel expansion | Blaze | Requires human approval |

---

## 11. Growth Flywheel

The system creates a compounding growth loop:

```
Products launch
  → Echo creates content (SEO, social, email, landing pages)
    → Blaze distributes across channels
      → Users discover and convert
        → Revenue funds more ad spend
          → Analytics identify what works
            → Echo creates more of what works
              → Blaze scales winning channels
                → More products launch (funded by revenue)
                  → Cycle accelerates
```

**Key compounding effects:**
- SEO content accumulates — each article is a permanent asset
- Email list grows — each subscriber is a free distribution channel
- Brand authority builds — higher conversion rates over time
- Data accumulates — better targeting, better content, lower CAC

---

## 12. Phase 1 Implementation (First 30 Days)

| Week | Blaze | Echo |
|------|-------|------|
| 1 | Set up analytics (PostHog + GA4), connect Stripe | Set up CMS, configure brand voice, connect SEO tools |
| 2 | Create email infrastructure (SendGrid), build first segments | Generate first 20 SEO articles, create social templates |
| 3 | Launch first email campaign, set up social scheduling | Build first 3 landing pages, start content calendar |
| 4 | Run first A/B tests, establish baseline metrics | Publish daily, begin trend monitoring |

**Success metrics at 30 days:**
- 20+ SEO articles published and indexed
- 3+ landing pages live with A/B tests running
- Email list started with welcome sequence active
- Social presence established on 2+ platforms
- Baseline CAC and conversion rates measured per channel
- Content calendar populated for next 90 days

---

*Architecture designed: 2026-02-08*
*Agents: Blaze (marketing strategy & distribution), Echo (content creation & SEO)*
*Dependencies: Factory Orchestrator (product launches), Stripe (revenue tracking), PostHog (analytics)*
