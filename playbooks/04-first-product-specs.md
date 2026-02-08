# Playbook 04 — First Product Detailed Specs

> Three candidates for the factory's first product. One recommendation. Specific enough to start building tomorrow.

**Status:** Ready for Decision  
**Date:** 2026-02-08  
**Depends on:** Playbook 02, Research 11, Architecture 10

---

## Selection Criteria Recap

The first product must:
- Be buildable in 2-3 days by agents
- Have real revenue potential ($500-2K MRR at scale)
- Test the full pipeline (research → build → test → deploy → market)
- Be simple enough to be a good first test (not the hardest thing we could do)

**Philosophy from Architecture 10:** "Businesses are experiments, the factory is the asset." This product proves the factory. Revenue is the proof, not the goal.

---

## Candidate A: **StatusPing** — Uptime Monitoring for Indie Hackers

### 1. Product Brief

**What:** Dead-simple uptime monitoring. Enter a URL → get pinged every 60 seconds → get alerted via email/Slack/Discord when it goes down. Dashboard shows uptime %, response time history, and incident log.

**Who it's for:** Indie hackers, solo founders, and small dev teams who run 1-20 sites/APIs and find BetterStack/Datadog overkill. People who want monitoring set up in 30 seconds, not 30 minutes.

**Core insight:** Every product the factory builds will need uptime monitoring. This is infrastructure we need anyway, productized.

**Differentiator:** Stupidly simple. No agents, no integrations marketplace, no 47-tab dashboard. Add URL, pick alert channel, done. Pricing is per-monitor, not per-seat — solo devs with many projects love this.

### 2. Technical Spec

| Layer | Choice | Notes |
|-------|--------|-------|
| Frontend | Next.js 14 (App Router) | Standard factory stack |
| Backend | Next.js API routes + Supabase Edge Functions | Ping workers run on Cloudflare Workers (cron triggers) |
| Database | Supabase (Postgres) | Monitors table, checks table (time-series), incidents table |
| Auth | Supabase Auth (magic link + GitHub OAuth) | Two options, zero friction |
| Payments | Stripe (subscription) | 3 tiers |
| Hosting | Vercel (app) + Cloudflare Workers (ping infra) | Separation: app and checker are independent |
| Email alerts | Resend | Transactional: down/up/weekly report |
| Slack/Discord | Incoming webhooks | User pastes webhook URL, we POST to it |
| Analytics | PostHog | Signup, add-monitor, upgrade, churn events |
| Monitoring | Sentry + self-monitoring (eat own dogfood) | StatusPing monitors itself |

**Key APIs:**
- Cloudflare Workers Cron Triggers (free tier: 10M requests/month)
- Resend ($0/month for 3K emails, $20/month for 50K)
- Supabase (free tier covers early stage)
- Stripe Billing API

**Database schema (core):**
```sql
-- monitors
id, user_id, url, name, check_interval_seconds, 
alert_email, alert_webhook_url, alert_webhook_type,
is_paused, created_at

-- checks  
id, monitor_id, status_code, response_time_ms, 
is_up, checked_at

-- incidents
id, monitor_id, started_at, resolved_at, 
duration_seconds, cause
```

**Architecture:**
```
Cloudflare Worker (cron every 60s)
    → Fetch all active monitors from Supabase
    → HTTP HEAD each URL (timeout 10s)
    → Write result to checks table
    → If status changed (up→down or down→up):
        → Insert/update incident
        → Fire alert (email via Resend, webhook via fetch)

Next.js App (Vercel)
    → Dashboard: list monitors, uptime %, response time chart
    → Add/edit/delete monitors
    → Settings: alert preferences, billing
    → Public status page: yourname.statuspinghq.com
```

### 3. Revenue Model

**Pricing:**
| Plan | Price | Monitors | Check Interval | Features |
|------|-------|----------|----------------|----------|
| Starter | $9/mo | 10 monitors | 60s | Email alerts |
| Pro | $19/mo | 50 monitors | 30s | Email + Slack/Discord + public status page |
| Team | $39/mo | 200 monitors | 15s | All channels + API access + multiple users |

**TAM estimate:**
- ~2M active developers/indie hackers globally running side projects
- ~15% need monitoring and don't have it or use free-tier-only tools
- Addressable: ~300K potential users
- At 0.05% capture = 150 paying customers × $15 avg = **$2,250 MRR achievable**

**Unit economics:**
- Infra cost per monitor: ~$0.02/month (Cloudflare Workers + Supabase row)
- 50-monitor Pro user costs us ~$1/month → 95% margin
- Break-even: 1 Starter customer covers infra for months

### 4. Build Plan (Task DAG)

```
DAY 1 (8 hours)
├── [Architect Agent] Setup repo, Supabase project, Stripe products, Vercel deploy
│   └── Duration: 1h
├── [Backend Agent] Database schema + migrations + Supabase RLS policies
│   └── Duration: 2h, depends on: Architect
├── [Backend Agent] Cloudflare Worker: cron ping logic + alert dispatch
│   └── Duration: 3h, depends on: Database schema
├── [Frontend Agent] Auth flow (magic link + GitHub) + dashboard shell
│   └── Duration: 2h, depends on: Architect
└── [Copy Agent] Landing page copy, email templates (down/up/welcome)
    └── Duration: 1h, parallel with all

DAY 2 (8 hours)  
├── [Frontend Agent] Dashboard: monitor list, add/edit modal, uptime chart
│   └── Duration: 4h, depends on: Day 1 backend
├── [Backend Agent] Stripe checkout + billing portal + plan enforcement
│   └── Duration: 3h, depends on: Day 1 backend
├── [Frontend Agent] Public status page (subdomain routing)
│   └── Duration: 2h, parallel
└── [Backend Agent] Weekly uptime report email (cron)
    └── Duration: 1h

DAY 3 (6 hours)
├── [QA Agent] E2E test: signup → add monitor → see check → get alert → pay → upgrade
│   └── Duration: 2h
├── [Frontend Agent] Polish: landing page, responsive, loading states, error handling
│   └── Duration: 2h
├── [Ops Agent] Sentry, PostHog, BetterStack, privacy policy, ToS
│   └── Duration: 1h
└── [Launch Agent] Product Hunt assets, Reddit drafts, Twitter thread, community post drafts
    └── Duration: 1h
```

**Total agent-hours:** ~22h across 3 days  
**Parallel agents needed:** 2-3 concurrent

### 5. Marketing Plan (First 100 Customers)

**Week 1 (Launch):**
- Product Hunt launch (Tuesday 12:01 AM PT) — target: 50+ upvotes, 30 signups
- Show HN post — target: 10 signups
- Reddit: r/SideProject, r/webdev, r/selfhosted — target: 20 signups
- Twitter launch thread with demo GIF — target: 10 signups
- Direct DM to 20 indie hackers who've tweeted about downtime/monitoring

**Week 2-4 (Organic + Content):**
- 5 SEO articles: "free uptime monitoring tools 2026", "how to monitor your side project", "BetterStack alternatives", "uptime monitoring for indie hackers", "how to set up a status page"
- Submit to 10 SaaS directories (SaaSHub, AlternativeTo, BetaList, etc.)
- Post in 5 relevant Discord/Slack communities (WIP, Indie Hackers, Dev communities)

**Month 2-3 (Compound):**
- Double content output (10 articles/month)
- Comparison pages: "StatusPing vs UptimeRobot", "StatusPing vs BetterStack"
- Small Reddit Ads test ($50, target r/webdev audience)
- Referral program: give 5 free monitors to referrer and referee

**Conversion funnel:** Visitor → free trial (no CC) → hits 10-monitor limit → upgrade  
**Target:** 500 signups month 1, 5% conversion = 25 paid = ~$400 MRR

### 6. Estimated Cost to Build

| Item | Cost |
|------|------|
| Agent compute (Claude API for coding, ~22h) | $50-80 |
| Domain (statuspinghq.com or similar) | $12 |
| Supabase (free tier) | $0 |
| Vercel (free tier) | $0 |
| Cloudflare Workers (free tier) | $0 |
| Resend (free tier: 3K emails/mo) | $0 |
| Stripe (2.9% + $0.30 per transaction) | Variable |
| Product Hunt launch assets | $0 (agent-generated) |
| Marketing (first month) | $50 |
| **Total to launch** | **~$120-150** |

### 7. Kill Criteria

| Timeframe | Kill If |
|-----------|---------|
| Day 7 | <15 signups from launch |
| Day 30 | $0 MRR AND <50 total signups |
| Day 60 | <$100 MRR |
| Day 90 | <$500 MRR with no growth trend |
| Any time | Cloudflare/Supabase free tier exceeded before revenue covers upgrade |

---

## Candidate B: **WaitlistKit** — Viral Waitlist Pages with Built-in Referral

### 1. Product Brief

**What:** Create a waitlist landing page in 60 seconds. Paste your product name + one-liner → get a beautiful waitlist page with built-in referral system (share to move up the list), email capture, analytics, and embeddable widget. Think LaunchList meets Carrd, but faster.

**Who it's for:** Founders launching new products who want to validate demand and build hype before building. Also: the factory itself — every product we validate in Stage 2 needs a waitlist.

**Core insight:** This directly dogfoods the factory pipeline. Stage 2 (Validation) needs landing pages with email capture. Building this tool means future factory products validate faster.

**Differentiator:** Referral mechanics built in from the start. Most waitlist tools are just email capture. WaitlistKit gamifies waiting — share your link, move up the list, see your position. This creates viral loops that other landing page tools don't have.

### 2. Technical Spec

| Layer | Choice | Notes |
|-------|--------|-------|
| Frontend | Next.js 14 (App Router) | Standard stack |
| Backend | Next.js API routes | Lightweight, no separate server |
| Database | Supabase (Postgres) | Waitlists, subscribers, referrals |
| Auth | Supabase Auth (magic link) | One option, maximum simplicity |
| Payments | Stripe (subscription) | 2 tiers |
| Hosting | Vercel (wildcard subdomains for pages) | yourlist.waitlistkit.com |
| Email | Resend | Confirmation, position update, launch notification |
| Analytics | PostHog | Page views, signups, referral conversion |
| AI | Claude API | Generate page copy from product description |

**Key flow:**
```
Creator:
  1. Sign up → "Describe your product in 1-2 sentences"
  2. AI generates: headline, subheadline, 3 benefit bullets, CTA text, color scheme
  3. Preview → edit any text inline → publish
  4. Live at yourlist.waitlistkit.com
  5. Dashboard: signups over time, referral leaderboard, export CSV

Subscriber:
  1. Lands on waitlist page → enters email
  2. Confirmation page: "You're #47! Share to move up"
  3. Gets unique referral link
  4. Each referral moves them up 5 positions
  5. Gets email when they reach top 10, and on launch day
```

**Database schema:**
```sql
-- waitlists
id, user_id, name, slug, headline, subheadline, 
benefits_json, cta_text, color_scheme, 
is_published, subscriber_count, created_at

-- subscribers
id, waitlist_id, email, position, referral_code,
referred_by, referral_count, signed_up_at

-- events
id, waitlist_id, type, metadata_json, created_at
```

### 3. Revenue Model

**Pricing:**
| Plan | Price | Waitlists | Subscribers | Features |
|------|-------|-----------|-------------|----------|
| Starter | $12/mo | 3 | 500 each | Referral system, basic analytics |
| Pro | $29/mo | Unlimited | 5,000 each | Custom domain, remove branding, export, webhooks |

**TAM estimate:**
- ~500K new products/startups launch per year globally
- ~30% create some form of waitlist or pre-launch page
- Addressable: ~150K potential waitlist creators per year
- At 0.1% capture = 150 paying customers × $18 avg = **$2,700 MRR achievable**
- Recurring: creators often launch multiple products

**Unit economics:**
- Cost per waitlist: ~$0.05/month (Supabase rows + Vercel bandwidth)
- Pro user with 10 waitlists costs us ~$0.50/month → 98% margin

### 4. Build Plan (Task DAG)

```
DAY 1 (8 hours)
├── [Architect Agent] Repo setup, Supabase, Stripe, Vercel, domain
│   └── 1h
├── [Backend Agent] Database schema + RLS + API routes (CRUD waitlists, subscribe endpoint)
│   └── 3h, depends on: Architect
├── [Backend Agent] Referral system logic (unique codes, position recalculation)
│   └── 2h, depends on: Schema
├── [Frontend Agent] Auth + dashboard shell + create waitlist form
│   └── 2h, depends on: Architect
└── [Copy Agent] Landing page for WaitlistKit itself + email templates
    └── 1h, parallel

DAY 2 (8 hours)
├── [Frontend Agent] Waitlist page renderer (public-facing, themed, responsive)
│   └── 3h
├── [Frontend Agent] AI copy generation (product description → page content)
│   └── 2h, depends on: page renderer
├── [Frontend Agent] Dashboard: subscriber list, referral leaderboard, analytics charts
│   └── 3h
└── [Backend Agent] Stripe checkout + plan enforcement (waitlist limits, subscriber limits)
    └── 2h

DAY 3 (5 hours)
├── [Frontend Agent] Inline editing of waitlist page + publish flow
│   └── 2h
├── [QA Agent] E2E: create waitlist → publish → subscribe → refer → see position change → pay
│   └── 1.5h
├── [Ops Agent] Monitoring, analytics, legal pages
│   └── 1h
└── [Launch Agent] PH assets, Reddit/Twitter drafts
    └── 0.5h
```

**Total agent-hours:** ~21h across 3 days

### 5. Marketing Plan (First 100 Customers)

**Week 1 (Launch):**
- Product Hunt launch — target: 80+ upvotes (waitlist tools do well on PH)
- Show HN — "I built a viral waitlist tool in 3 days"
- Reddit: r/SideProject, r/startups, r/EntrepreneurRideAlong
- Twitter launch thread + demo video
- DM 30 indie hackers who are "building in public" pre-launch

**Week 2-4:**
- Partnership angle: post in Indie Hackers, WIP, and startup communities offering free Pro for first 50 users
- 5 SEO articles: "how to build a viral waitlist", "best waitlist tools 2026", "pre-launch marketing strategy", "waitlist referral system", "validate your startup idea"
- Create a free "Waitlist Playbook" PDF (email gated) — drives signups
- Embed widget integration guide for existing sites

**Month 2-3:**
- Content doubles: comparison pages, case studies from early users
- Reddit Ads ($50 test, target r/startups)
- Integrate with popular tools (Zapier webhook) to increase stickiness
- Build in public: share metrics weekly on Twitter

**Viral mechanic:** Every waitlist page has a small "Powered by WaitlistKit" link. Each successful waitlist becomes marketing for us. Network effect.

### 6. Estimated Cost to Build

| Item | Cost |
|------|------|
| Agent compute (~21h) | $45-70 |
| Domain (waitlistkit.com or similar) | $12-30 |
| Claude API (copy generation) | $5-10/month |
| Supabase, Vercel, Resend (free tiers) | $0 |
| Marketing (month 1) | $50 |
| **Total to launch** | **~$115-165** |

### 7. Kill Criteria

| Timeframe | Kill If |
|-----------|---------|
| Day 7 | <20 signups from launch |
| Day 30 | <5 waitlists created by non-us users AND $0 MRR |
| Day 60 | <$100 MRR |
| Day 90 | <$500 MRR with no growth trend |
| Any time | No viral loop evidence (referral links not being shared) |

---

## Candidate C: **CronPilot** — Cron Job Monitoring as a Service

### 1. Product Brief

**What:** Monitor your cron jobs and scheduled tasks. Register a job → get a unique ping URL → add `curl https://cronpilot.com/ping/abc123` to the end of your cron script → get alerted if the ping doesn't arrive on schedule. Plus: see run history, duration tracking, and a timeline view of all your jobs.

**Who it's for:** Developers and DevOps people running scheduled tasks (backups, data syncs, report generation, cleanup scripts). Anyone who's been burned by a cron job silently failing.

**Core insight:** This is a known pain point with proven willingness to pay (Cronitor, Healthchecks.io are $20M+ businesses). The market is validated — we just need a simpler, cheaper entry point for solo devs and small teams.

**Differentiator:** Cheapest option for small-scale users. Healthchecks.io is open source (free self-hosted) but their hosted plan starts at $20/mo. Cronitor starts at $20/mo. We undercut at $7/mo and win the long tail of solo devs with 5-20 cron jobs.

### 2. Technical Spec

| Layer | Choice | Notes |
|-------|--------|-------|
| Frontend | Next.js 14 (App Router) | Standard stack |
| Backend | Next.js API routes + Cloudflare Workers (ping endpoint) | Ping endpoint must be fast + globally distributed |
| Database | Supabase (Postgres) | Jobs, pings, alerts |
| Auth | Supabase Auth (magic link + GitHub) | Developer audience, GitHub is expected |
| Payments | Stripe (subscription) | 3 tiers |
| Hosting | Vercel (app) + Cloudflare Workers (ping receiver) | Ping endpoint on edge for <50ms response |
| Alerts | Resend (email) + webhooks (Slack/Discord/PagerDuty) | |
| Analytics | PostHog | |

**Key flow:**
```
1. Sign up → Create a "check" (name it, set expected schedule: "every 1h", "every day at 3am")
2. Get unique URL: https://cronpilot.com/ping/abc123
3. Add to your cron: `0 * * * * /path/to/script.sh && curl -fsS https://cronpilot.com/ping/abc123`
4. CronPilot expects a ping every hour. If one is late → alert fires
5. Dashboard shows: all checks, last ping time, status (✅/⚠️/🔴), run duration
```

**Database schema:**
```sql
-- checks
id, user_id, name, slug, ping_url, 
expected_period_seconds, grace_period_seconds,
alert_email, alert_webhook_url,
last_ping_at, status, created_at

-- pings
id, check_id, pinged_at, duration_ms, 
source_ip, user_agent

-- alerts
id, check_id, type, fired_at, resolved_at
```

**Ping endpoint (Cloudflare Worker):**
```
GET /ping/:slug
  → Look up check by slug
  → Record ping (timestamp, duration if provided via ?duration=3200)
  → Update check.last_ping_at and check.status
  → Return 200 OK
  
Total latency: <100ms from anywhere in the world
```

**Miss detection (Cloudflare Worker cron, every 60s):**
```
  → Query all active checks where:
      NOW() - last_ping_at > expected_period + grace_period
      AND status != 'down'
  → For each: set status='down', create alert, fire notification
```

### 3. Revenue Model

**Pricing:**
| Plan | Price | Checks | Features |
|------|-------|--------|----------|
| Hobby | $7/mo | 20 checks | Email alerts, 7-day history |
| Pro | $19/mo | 100 checks | All alert channels, 90-day history, API |
| Team | $49/mo | 500 checks | Multi-user, audit log, 1-year history |

**TAM estimate:**
- ~25M developers globally, ~40% run some form of scheduled tasks
- ~10M potential users of cron monitoring
- ~2% would pay for hosted monitoring at indie price points = 200K
- At 0.05% capture = 100 customers × $15 avg = **$1,500 MRR achievable**

**Unit economics:**
- Cost per check: ~$0.01/month (one Supabase row per ping, Cloudflare Workers free tier)
- 20-check Hobby user costs us ~$0.20/month → 97% margin
- Cloudflare Workers free tier: 100K requests/day = supports ~2,000 checks at 1-minute intervals

### 4. Build Plan (Task DAG)

```
DAY 1 (7 hours)
├── [Architect Agent] Repo, Supabase, Stripe, Vercel, Cloudflare Worker project, domain
│   └── 1h
├── [Backend Agent] Database schema + Cloudflare Worker ping endpoint
│   └── 2h, depends on: Architect
├── [Backend Agent] Miss detection worker (cron) + alert dispatch (email + webhook)
│   └── 2h, depends on: Schema
├── [Frontend Agent] Auth + dashboard shell + create check form
│   └── 2h, depends on: Architect
└── [Copy Agent] Landing page copy + email templates + docs (integration guide)
    └── 1h, parallel

DAY 2 (7 hours)
├── [Frontend Agent] Dashboard: check list with status indicators, ping timeline, detail view
│   └── 3h
├── [Frontend Agent] Settings: alert preferences, integrations (webhook config)
│   └── 1.5h
├── [Backend Agent] Stripe checkout + plan enforcement + billing portal
│   └── 2h
└── [Backend Agent] Simple REST API for programmatic check management
    └── 1.5h

DAY 3 (4 hours)
├── [QA Agent] E2E: signup → create check → send ping → miss detection → alert fires → pay
│   └── 1.5h
├── [Frontend Agent] Landing page + responsive polish + integration docs page
│   └── 1.5h
├── [Ops Agent] Sentry, PostHog, self-monitoring, legal pages
│   └── 0.5h
└── [Launch Agent] PH assets, community post drafts, comparison page drafts
    └── 0.5h
```

**Total agent-hours:** ~18h across 3 days (lightest build of the three)

### 5. Marketing Plan (First 100 Customers)

**Week 1 (Launch):**
- Product Hunt launch — "Dead simple cron monitoring for $7/mo"
- Show HN — developers are the exact HN audience
- Reddit: r/selfhosted, r/devops, r/sysadmin, r/webdev
- Twitter thread: "Your cron jobs are silently failing. Here's how to know."
- Direct outreach: reply to tweets/posts about cron failures, backup failures, monitoring

**Week 2-4:**
- 5 SEO articles: "cron job monitoring tools 2026", "how to monitor cron jobs", "Cronitor alternatives", "Healthchecks.io vs CronPilot", "never miss a failed backup again"
- Integration guides: "Monitor Laravel scheduled tasks", "Monitor GitHub Actions", "Monitor Docker cron"
- Submit to DevOps tool directories and awesome-lists on GitHub
- Post integration snippets as GitHub Gists (backlinks + discovery)

**Month 2-3:**
- Framework-specific SDKs (Python, Node, Go, PHP — agent-generated)
- "Works with" integration pages (each one is an SEO landing page)
- Dev community content: write for Dev.to, Hashnode
- Small Google Ads test ($50, target "cron monitoring" keywords)

**Developer trust mechanic:** Open-source the client libraries. Developers trust what they can read.

### 6. Estimated Cost to Build

| Item | Cost |
|------|------|
| Agent compute (~18h) | $40-60 |
| Domain (cronpilot.com or similar) | $12-20 |
| Supabase, Vercel, Cloudflare (free tiers) | $0 |
| Resend (free tier) | $0 |
| Marketing (month 1) | $50 |
| **Total to launch** | **~$105-135** |

### 7. Kill Criteria

| Timeframe | Kill If |
|-----------|---------|
| Day 7 | <20 signups (developer audience should respond fast) |
| Day 30 | <10 active checks being pinged AND $0 MRR |
| Day 60 | <$70 MRR |
| Day 90 | <$500 MRR with no growth trend |
| Any time | Cloudflare free tier exceeded before revenue covers Pro plan |

---

## Comparison Matrix

| Criteria | StatusPing | WaitlistKit | CronPilot |
|----------|-----------|-------------|-----------|
| **Build complexity** | Medium | Medium | **Low** ★ |
| **Build time (days)** | 3 | 3 | **2.5** ★ |
| **Agent-hours** | 22 | 21 | **18** ★ |
| **Cost to launch** | $120-150 | $115-165 | **$105-135** ★ |
| **Tests full pipeline** | ✅ | ✅ | ✅ |
| **Tests standard stack** | ✅ (+ CF Workers) | ✅ | ✅ (+ CF Workers) |
| **Revenue ceiling (MRR)** | $2,250 | **$2,700** ★ | $1,500 |
| **Market validation** | Proven (many players) | Proven (LaunchList etc) | **Proven + underserved** ★ |
| **Dogfoods factory** | ✅ Monitor our products | **✅✅ Validate future products** ★ | ✅ Monitor our cron jobs |
| **Viral/organic growth** | Low | **High** ★ (powered-by links) | Medium (dev word-of-mouth) |
| **Audience reachability** | Good (devs) | **Great** (indie hackers = our community) | Good (devs) |
| **Competitive moat** | Low (crowded) | Medium (referral mechanics) | **Medium** (price + simplicity) |
| **Time to first revenue** | 7-14 days | 7-14 days | **5-10 days** ★ |
| **Maintenance burden** | Medium (global pings) | Low | **Low** ★ |

Stars (★) = wins that category.

---

## ⭐ Recommendation: Candidate C — CronPilot

### Why CronPilot Over the Others

**1. Lightest build = highest chance of shipping on time.**  
At 18 agent-hours and the simplest architecture, CronPilot has the lowest risk of the factory's first test failing due to build complexity. The first product's job is to prove the pipeline, not to be ambitious. We save ambitious for Product #2.

**2. Proven market with underserved segment.**  
Cronitor ($20M+ business) and Healthchecks.io prove people pay for this. But both start at $20/mo. There's a clear gap at $7/mo for the solo dev with 5-20 cron jobs. We're not inventing a category — we're undercutting an established one.

**3. Developer audience converts fast.**  
Developers decide in minutes, not weeks. They add one `curl` to a script, see it work, and upgrade when they hit the free limit. No onboarding friction, no "let me think about it." This means faster time-to-revenue, which is what we're testing.

**4. The architecture is beautifully simple.**  
One Cloudflare Worker receives pings. One cron checks for misses. One Next.js app shows the dashboard. Three components. Each independently testable. If something breaks, it's obvious where.

**5. Self-dogfooding from day one.**  
Every product the factory builds after this will have scheduled tasks (report generation, data cleanup, email campaigns). CronPilot monitors them. We're building infrastructure we'll use on every subsequent product.

### Why Not the Others?

**StatusPing** is good but more competitive (UptimeRobot free tier is very generous) and the global ping infrastructure is slightly more complex to get right. Save for Product #3.

**WaitlistKit** has the highest ceiling and best viral mechanics, but it's the most complex build with the most edge cases (inline editing, referral position recalculation, themed page rendering). It's Product #2 — after the factory is proven. The dogfooding argument is strong, but we can validate future products with a simple Supabase form + landing page until WaitlistKit exists.

### Adjusted Build Timeline for CronPilot

```
DAY 1 — FOUNDATION + CORE
  Morning:  Repo, infra, DB schema, auth flow
  Afternoon: Cloudflare Worker ping endpoint + miss detection
  Evening:  Dashboard shell with check CRUD

DAY 2 — FEATURES + BILLING  
  Morning:  Dashboard detail view (ping timeline, status history)
  Afternoon: Stripe integration + alert system (email + webhooks)
  Evening:  Landing page + docs/integration guide

DAY 3 — POLISH + LAUNCH
  Morning:  QA, E2E tests, fix bugs
  Midday:   Monitoring, analytics, legal pages
  Afternoon: Launch prep (PH, Reddit, Twitter assets)
  Evening:   GO LIVE → first community posts
```

### First Week Post-Launch

| Day | Action |
|-----|--------|
| Day 3 (launch) | Community posts, respond to all comments |
| Day 4 | Product Hunt submission |
| Day 5 | Show HN post, more Reddit/community posts |
| Day 6 | Direct outreach (20 DMs to devs who've complained about cron) |
| Day 7 | First metrics review, write 2 SEO articles |
| Day 10 | Post-mortem: what worked, what broke, factory learnings |

### Decision Gate

After CronPilot launches:
- **If Day 30 metrics pass:** Start WaitlistKit (Product #2) using factory learnings
- **If Day 30 metrics fail but factory pipeline worked:** Kill CronPilot, start WaitlistKit anyway — the pipeline is the win
- **If factory pipeline broke:** Fix pipeline issues first, then retry with CronPilot (simplest build)

---

## What This Proves About the Factory

Regardless of CronPilot's revenue outcome, a successful build proves:

1. ✅ Agents can ship production SaaS in 2-3 days
2. ✅ The standard stack (Next.js + Supabase + Stripe + Vercel) works
3. ✅ Cloudflare Workers can serve as our edge compute layer
4. ✅ The launch playbook produces real users
5. ✅ Real Stripe payments can be processed
6. ✅ Monitoring and analytics fire correctly
7. ✅ The build DAG / agent coordination model works

The factory is the product. CronPilot is just the test.

---

*Playbook 04 — First Product Detailed Specs*  
*Date: 2026-02-08*  
*Recommendation: CronPilot — simplest build, proven market, fastest to revenue*
