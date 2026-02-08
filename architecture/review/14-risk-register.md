# 14 — Risk Register

> Comprehensive risk analysis for the Agentic Factory masterplan.
> Covers technical, financial, operational, legal, and market risks.
>
> **Date:** 2026-02-08
> **Author:** Atlas
> **Scoring:** Likelihood (1-5) × Impact (1-5) = Risk Score (1-25)
> **Priority:** Critical (16-25), High (10-15), Medium (5-9), Low (1-4)

---

## Risk Scoring Key

| Score | Likelihood | Impact |
|-------|-----------|--------|
| 1 | Rare | Negligible |
| 2 | Unlikely | Minor |
| 3 | Possible | Moderate |
| 4 | Likely | Major |
| 5 | Almost Certain | Catastrophic |

---

## Summary: Top 10 Risks by Score

| # | Risk | Category | L | I | Score | Owner |
|---|------|----------|---|---|-------|-------|
| 1 | API cost blowout / runaway agent loops | Financial | 4 | 5 | **20** | Kev / Finn |
| 2 | Prompt injection compromises agent actions | Technical | 4 | 5 | **20** | Hawk / Law |
| 3 | Single point of failure (dreamteam server) | Operational | 4 | 4 | **16** | Forge |
| 4 | Platform account bans (Stripe, hosting) | Operational | 3 | 5 | **15** | Finn / Forge |
| 5 | Agent quality degradation / shipping bad code | Technical | 4 | 4 | **16** | Hawk |
| 6 | Revenue model failure (no product-market fit) | Market | 3 | 5 | **15** | Atlas / Chase |
| 7 | Model provider dependency / API changes | Technical | 3 | 4 | **12** | Kev / Forge |
| 8 | Secret/credential leakage | Technical | 3 | 5 | **15** | Hawk / Forge |
| 9 | Over-engineering before first revenue | Operational | 4 | 3 | **12** | Kev / Atlas |
| 10 | Regulatory crackdown on AI-operated businesses | Legal | 2 | 5 | **10** | Law |

---

## Technical Risks

### T1: API Cost Blowout / Runaway Agent Loops
- **Score:** L4 × I5 = **20 (Critical)**
- **Description:** An agent enters an infinite retry loop, context window stuffing, or cascading failures across agents, burning hundreds or thousands of dollars in minutes. Per research doc 14, this is the #1 operational risk for AI factories.
- **Trigger:** Bug in orchestration logic, failed tool call retries, agent feeding output back as input, model confusion creating unbounded tool call chains.
- **Mitigation:**
  - Per-task and per-agent budget caps enforced at LiteLLM proxy layer (arch 08, 13)
  - Kill switches at global, per-agent, and per-provider levels in Redis (arch 08)
  - Cost velocity monitoring: alert if $/min exceeds 3× rolling average (arch 08)
  - Max steps per agent run (hard cap of 20 tool calls per task) (arch 04)
  - Max tokens per request always set (never unbounded generation)
  - Provider-level hard spend caps in Anthropic/OpenAI dashboards
  - Daily spend alerts at 50%, 80%, 100% of budget via WhatsApp (arch 12)
- **Residual Risk:** Medium — automated controls catch most cases but novel failure modes can emerge.
- **Owner:** Kev (enforcement), Finn (budget monitoring)

### T2: Prompt Injection Compromising Agent Actions
- **Score:** L4 × I5 = **20 (Critical)**
- **Description:** Malicious text in emails, web pages, scraped documents, or user-uploaded content hijacks agent behaviour — causing data exfiltration, unauthorized actions, or privilege escalation. Per research 17, no fully reliable defence exists.
- **Trigger:** Agent processes untrusted external content (web scraping, email reading, user inputs) that contains adversarial instructions.
- **Mitigation:**
  - Dual LLM pattern: quarantined LLM processes untrusted content with no tool access; privileged LLM acts on sanitized summaries (research 17)
  - Input separation: external content tagged as `<<<UNTRUSTED_CONTENT>>>` in prompts
  - Tool allowlists per task: "summarize document" task cannot call email/deploy tools (arch 07)
  - Output filtering: scan agent outputs before execution for dangerous patterns
  - CaMeL-inspired capability-based mediation for high-stakes tool calls (research 17)
  - Regular red-team exercises against prompt injection
  - Human-in-the-loop gates for all external-facing actions (arch 12)
- **Residual Risk:** High — this is fundamentally unsolved. Defence is layered mitigation, not prevention.
- **Owner:** Hawk (testing), Law (policy)

### T3: Agent Quality Degradation / Shipping Bad Code
- **Score:** L4 × I4 = **16 (Critical)**
- **Description:** AI-generated code contains security vulnerabilities, logic errors, architectural violations, or hallucinated APIs that pass automated checks and reach production.
- **Trigger:** Model quality regression from provider updates, insufficient test coverage, rubber-stamped PR reviews, test theatre (meaningless tests that pass).
- **Mitigation:**
  - 5-layer guardrail stack: prompt rules → inline linting → pre-commit → CI/CD → post-deploy (research 13, arch 09)
  - Cross-model review: different model reviews code than wrote it (arch 15)
  - SAST via Semgrep with custom rules, secrets detection via Gitleaks (research 13)
  - Mutation testing (Stryker) to verify test quality (research 13)
  - Hawk QA agent is adversarial to Rex build agent by design (arch 09)
  - Mandatory `typecheck + lint + test + build` gate — no exceptions (arch 15)
  - Architectural consistency enforcement via dependency-cruiser (research 13)
  - Human review required for: auth, payments, data models, infra changes (arch 12)
- **Residual Risk:** Medium — layered gates catch most issues, but novel bugs can slip through.
- **Owner:** Hawk

### T4: Model Provider Dependency / API Breaking Changes
- **Score:** L3 × I4 = **12 (High)**
- **Description:** Anthropic, OpenAI, or Google change pricing, deprecate models, impose rate limits, or degrade quality in model updates — disrupting factory operations.
- **Trigger:** Provider pricing increase, model deprecation, quality regression in model update, rate limit tightening, ToS changes restricting automated use.
- **Mitigation:**
  - Multi-model routing built into architecture from day 1 (arch 02, 13)
  - LiteLLM proxy with automatic fallbacks: `local/fast → cloud/fast`, `local/quality → cloud/mid` (arch 13)
  - Local inference on RTX 3090 for 60%+ of token volume (arch 13)
  - Free tier maximization: Cerebras for research/triage tasks (README)
  - Model-agnostic orchestration: agents don't know/care which model serves them
  - Evaluation suite to detect quality regression on model updates (arch 19)
- **Residual Risk:** Low-Medium — multi-model routing and local inference provide strong fallbacks.
- **Owner:** Kev (routing), Forge (infrastructure)

### T5: Secret / Credential Leakage
- **Score:** L3 × I5 = **15 (High)**
- **Description:** API keys, tokens, or private data leak through agent outputs, logs, prompt context, or compromised tool calls.
- **Trigger:** Agent hallucinating credentials into output, secrets appearing in git commits, logging sensitive data, prompt injection causing credential exfiltration.
- **Mitigation:**
  - Secrets injected by runtime tool executor, never visible to agents (arch 07, research 17)
  - Output scanning with regex + ML-based secret detection (research 17)
  - Gitleaks in pre-commit hooks and CI (research 13)
  - Log redaction for API key patterns
  - Short-lived dynamic secrets from Vault where possible (research 17)
  - Per-agent scoped credentials — compromise of one doesn't expose all (arch 07)
  - Agents never construct API calls with tokens directly (arch 14)
- **Residual Risk:** Medium — layered controls effective but prompt injection could bypass.
- **Owner:** Hawk (detection), Forge (infrastructure)

### T6: Single GPU Constraint / Model Swapping Latency
- **Score:** L4 × I2 = **8 (Medium)**
- **Description:** The RTX 3090 can only run one large model at a time. Model swaps take 15-30 seconds during which local inference is unavailable. GPU contention between inference, TTS, and other workloads.
- **Trigger:** High-volume concurrent requests needing different model sizes, TTS job blocking inference, swap happening during time-sensitive task.
- **Mitigation:**
  - Cloud fallback routing during GPU swaps (arch 13)
  - Embeddings always on CPU (separate llama.cpp instance) (arch 13)
  - GPU mutex lock preventing concurrent large VRAM consumers (arch 13)
  - Model manager daemon with idle timeout swap-back to 8B default (arch 13)
  - TTS defaults to ElevenLabs cloud; local TTS only when GPU idle (arch 13)
  - Future: second RTX 3090 for ~$800 eliminates swapping entirely (arch 13)
- **Residual Risk:** Low — cloud fallback makes this a latency issue, not a failure.
- **Owner:** Forge / Glim

### T7: Memory System Fragmentation / Context Loss
- **Score:** L3 × I3 = **9 (Medium)**
- **Description:** Agent memory becomes fragmented, stale, or contradictory across filesystem markdown files, ChromaDB embeddings, and per-session context. Agents lose important context or act on outdated information.
- **Trigger:** Memory files not updated, embedding drift, conflicting information across memory layers, session context window limits.
- **Mitigation:**
  - Three-tier memory architecture: short-term (session), medium-term (daily files), long-term (MEMORY.md + ChromaDB) (arch 05, 20)
  - Periodic memory consolidation during heartbeats (AGENTS.md)
  - MCP memory server for structured store/recall with semantic search (arch 14)
  - Self-improvement system detects "context thrashing" signals (arch 19)
- **Residual Risk:** Medium — memory management is an ongoing challenge for all agent systems.
- **Owner:** Atlas / Kev

### T8: Communication Bus Failure Modes (NATS)
- **Score:** L2 × I4 = **8 (Medium)**
- **Description:** Message storms, deadlocks, poison messages, or lost messages in the NATS-based communication bus disrupt agent coordination.
- **Trigger:** Event-triggered events causing exponential fan-out, circular task dependencies, malformed messages crashing consumers.
- **Mitigation:**
  - Rate limiting per agent (100 msg/s default) with circuit breaker (arch 11)
  - Architectural rule: events → tasks only, never events → events (arch 11)
  - Mandatory TTL on all TASK_REQUEST messages (max 1 hour) (arch 11)
  - DAG enforcement: supervisor rejects circular task chains (arch 11)
  - Deadlock detector monitoring wait-for graph (arch 11)
  - Poison message routing to DLQ after 3 nack/redeliveries (arch 11)
  - JetStream persistence for message survival across restarts (arch 11)
- **Residual Risk:** Low — comprehensive failure mode coverage in architecture.
- **Owner:** Forge

### T9: Browser Automation / Scraping Fragility
- **Score:** L3 × I2 = **6 (Medium)**
- **Description:** Web scraping and browser automation break due to anti-bot measures, site redesigns, or CAPTCHA challenges, disrupting market research and competitive intelligence.
- **Trigger:** Target sites update anti-bot measures, Cloudflare/reCAPTCHA blocks, DOM changes invalidating selectors, Browserbase service issues.
- **Mitigation:**
  - Stagehand self-healing selectors with caching (arch 17)
  - Browserbase for stealth (fingerprint rotation, proxy rotation, CAPTCHA solving) (arch 17)
  - Per-domain rate limits and robots.txt respect (arch 17)
  - Fallback to API-based data sources when scraping fails (arch 17)
  - Cache aggressively to reduce scraping frequency (arch 17)
- **Residual Risk:** Low — self-healing selectors and Browserbase handle most cases.
- **Owner:** Scout / Hawk

### T10: Stagehand Cache Staleness
- **Score:** L3 × I2 = **6 (Medium)**
- **Description:** Stagehand's cached AI selectors become stale, causing tests to pass on cached selectors while missing real UI bugs.
- **Trigger:** UI redesign not detected by cache invalidation, gradual DOM drift.
- **Mitigation:**
  - Periodic cache invalidation schedule (arch 17)
  - Weekly cache-cold test runs (arch 17)
  - Pixel-diff as blocking gate (not Stagehand alone) (arch 17)
  - LLM visual audit as advisory layer for semantic issues (arch 17)
- **Residual Risk:** Low.
- **Owner:** Hawk

---

## Financial Risks

### F1: Insufficient Revenue / Portfolio Hit Rate Too Low
- **Score:** L3 × I5 = **15 (High)**
- **Description:** Factory-built products fail to achieve product-market fit. Hit rate below 30% threshold makes the portfolio model unsustainable. Revenue doesn't cover LLM costs.
- **Trigger:** Poor market selection, weak product quality, inadequate marketing, wrong pricing, too much competition.
- **Mitigation:**
  - Rigorous opportunity scoring: TAM × buildability × speed-to-revenue × competition (arch 10, research 11)
  - Validation before building: 50+ waitlist signups or >5% pricing CTR as go/no-go (research 11)
  - Fast kill decisions: sunset products below threshold after defined period (research 20)
  - Portfolio diversification: don't build 10 of the same type (research 20)
  - Start with Tier 1 businesses (micro-SaaS, API services) with highest autonomy potential (research 11)
  - Self-improvement system learns from failures to improve opportunity selection (arch 19)
- **Residual Risk:** High — market risk is inherently uncertain. Portfolio approach mitigates but doesn't eliminate.
- **Owner:** Atlas (strategy), Chase (sales)

### F2: Cloud Spend Exceeds Revenue
- **Score:** L3 × I4 = **12 (High)**
- **Description:** Monthly LLM API + infrastructure costs exceed revenue, creating negative cash flow that burns through capital.
- **Trigger:** Aggressive scaling before revenue, model price increases, inefficient routing wasting tokens on expensive models.
- **Mitigation:**
  - Cost pyramid enforcement: 60% free/cheap (Cerebras/local), 30% mid-tier, 10% frontier (README)
  - Daily spend cap of $30/day initially, alerts at threshold (arch 20)
  - Hybrid compute: local inference for 90% of token volume at near-zero marginal cost (arch 13)
  - Prompt caching (90% input savings) and batch API (50% off) on cloud calls (arch 13)
  - Per-task cost tracking with Dash analytics (arch 18)
  - Budget envelope system per agent team (arch 12)
  - Target: revenue ≥ LLM costs by month 3 (arch 20)
- **Residual Risk:** Medium — cost controls are strong but early revenue is uncertain.
- **Owner:** Finn (finance), Dash (monitoring)

### F3: Payment Processor Bans
- **Score:** L3 × I5 = **15 (High)**
- **Description:** Stripe or other payment processors flag and ban accounts due to perceived mass-automated business activity, chargeback rates, or policy violations.
- **Trigger:** Multiple businesses on one Stripe account, high chargeback rate, automated account activity patterns, ToS violations.
- **Mitigation:**
  - Each product operates as a legitimate business with genuine value (research 20)
  - Keep chargeback rates well below processor thresholds
  - Diversify payment processors (Stripe + Lemon Squeezy + Paddle) (research 11)
  - Transparent pricing, clear refund policies
  - Monitor Stripe account health metrics proactively
  - Legal entity structure: umbrella holding with separate business units (research 20)
- **Residual Risk:** Medium — legitimate business operations reduce risk but don't eliminate it.
- **Owner:** Finn / Law

### F4: Hardware Failure / Data Loss
- **Score:** L2 × I4 = **8 (Medium)**
- **Description:** RTX 3090 failure, SSD failure, or data corruption on dreamteam causes loss of agent state, task queue, or project code.
- **Trigger:** Hardware failure (GPU, SSD, PSU), power surge, filesystem corruption.
- **Mitigation:**
  - All project code in git repos (distributed by nature)
  - UPS for power protection (arch 20)
  - Auto-restart services via systemd (arch 13)
  - Cloud fallback for critical agents during hardware downtime (arch 20)
  - Regular backups of task queue (SQLite) and memory files
  - NVMe SSD with SMART monitoring for early failure detection
- **Residual Risk:** Low — git + cloud fallback limits blast radius.
- **Owner:** Forge

### F5: Electricity Cost Escalation
- **Score:** L2 × I2 = **4 (Low)**
- **Description:** NZ electricity prices increase significantly, raising the ~$76/month baseline for 24/7 GPU operation.
- **Trigger:** Energy market changes, rate increases.
- **Mitigation:**
  - Currently ~$76/month — manageable even with significant increases
  - GPU scheduling: run intensive workloads during off-peak if time-of-use pricing applies
  - Second GPU only when justified by volume (arch 13)
- **Residual Risk:** Low.
- **Owner:** Finn

---

## Operational Risks

### O1: Single Point of Failure (dreamteam Server)
- **Score:** L4 × I4 = **16 (Critical)**
- **Description:** All factory operations depend on one physical server. Any outage (hardware failure, power loss, network issue, OS crash) halts the entire factory.
- **Trigger:** Hardware failure, power outage, network outage, kernel panic, accidental misconfiguration.
- **Mitigation:**
  - UPS for power continuity (arch 20)
  - Auto-restart all services via systemd on-failure (arch 13)
  - Cloud fallback agents for critical functions (Kev can operate via cloud-only models)
  - Git repos provide code resilience
  - Task queue state recoverable from filesystem
  - Future: dedicated inference node separate from dev workstation (arch 13)
  - Long-term: redundant server or cloud VM for critical orchestration
- **Residual Risk:** High — single server is a real vulnerability until redundancy is added.
- **Owner:** Forge

### O2: Over-Engineering Before First Revenue
- **Score:** L4 × I3 = **12 (High)**
- **Description:** Team spends months perfecting infrastructure (NATS bus, MCP gateway, ChromaDB, REEF dashboard, etc.) before shipping a single revenue-generating product.
- **Trigger:** Architecture perfectionism, shiny-object syndrome, building Phase 4+ before Phase 1-2 generate revenue.
- **Mitigation:**
  - Build order explicitly defined: ship first product before Phase 4+ (arch 20)
  - Critical path identified: Task Queue → Kev Routing → Pi SDK → First pipeline → First product → Revenue (arch 20)
  - Phase 1-2 is MVP; everything else is a force multiplier (arch 20)
  - Week 4 milestone: first product deployed to production (arch 20)
  - Month 3 milestone: first revenue > $0 MRR (arch 20)
- **Residual Risk:** Medium — requires discipline to resist infrastructure temptation.
- **Owner:** Kev (enforcement), Atlas (strategy)

### O3: Agent Coordination Failures at Scale
- **Score:** L3 × I3 = **9 (Medium)**
- **Description:** As the number of agents and concurrent tasks grows, coordination problems emerge: race conditions on shared files, git conflicts between parallel agents, task queue contention, deadlocked workflows.
- **Trigger:** Multiple agents working on overlapping files, git merge conflicts, stale filesystem state, complex multi-step workflows.
- **Mitigation:**
  - Git worktrees for parallel agent execution on non-overlapping file sets (arch 15)
  - Single-writer principle: each resource owned by one agent (research 12)
  - Idempotent task processing with unique task IDs (research 12)
  - Timeouts on every inter-agent call (research 12)
  - DAG enforcement preventing circular dependencies (arch 11)
  - Start with sequential execution, add parallelism incrementally
- **Residual Risk:** Medium — coordination at scale is inherently hard.
- **Owner:** Kev

### O4: Maintenance Burden of Portfolio Products
- **Score:** L3 × I3 = **9 (Medium)**
- **Description:** As the portfolio grows to 15-30+ live products, maintenance overhead (dependency updates, bug fixes, security patches, uptime monitoring) consumes all agent capacity, leaving no bandwidth for new products.
- **Trigger:** Dependency vulnerabilities, customer bug reports, infrastructure updates, platform breaking changes.
- **Mitigation:**
  - Automated dependency updates via Renovate/Dependabot (research 13)
  - Automated security patching playbooks (arch 15)
  - Monitoring agent (Dash) for proactive issue detection (arch 18)
  - Support agent (Dot) for automated customer support
  - Strict kill criteria: sunset unprofitable products quickly (research 20)
  - Shared infrastructure reduces per-product overhead (arch 20)
- **Residual Risk:** Medium — portfolio maintenance is a known scaling challenge.
- **Owner:** Dot (ops), Forge (infra)

### O5: Human Bottleneck on Approval Gates
- **Score:** L3 × I3 = **9 (Medium)**
- **Description:** Adam becomes the bottleneck on Tier 2/3 approval gates, causing work to queue up. Approval fatigue leads to rubber-stamping, defeating the purpose of human oversight.
- **Trigger:** High volume of approval requests, Adam asleep/busy, notification fatigue, overly broad gate criteria.
- **Mitigation:**
  - Graduated autonomy: agents earn trust and gates are relaxed (arch 12)
  - Remove gates with >95% approval rate (arch 12)
  - 3am protocol: pre-authorized playbooks for known scenarios (arch 12)
  - Batch review for low-urgency items (arch 12)
  - Timeout behaviours: deny by default for safe operations (arch 12)
  - Reminder cadence at 50% and 90% of deadline (arch 12)
  - Dashboard for batched review when convenient (arch 12)
- **Residual Risk:** Low-Medium — graduated autonomy addresses this over time.
- **Owner:** Kev / Adam

### O6: Hosting Provider Lock-In or Outage
- **Score:** L2 × I3 = **6 (Medium)**
- **Description:** Railway, Vercel, or Cloudflare experience outages or change pricing/terms, affecting deployed products.
- **Trigger:** Provider outage, pricing increase, ToS change, account suspension.
- **Mitigation:**
  - Multi-platform deployment strategy: Cloudflare (edge), Railway (backend), Vercel (frontend) (arch 20)
  - Docker as universal escape hatch — any product can be redeployed elsewhere (research 10)
  - Pulumi IaC for infrastructure portability (research 10)
  - No single provider hosts all products
- **Residual Risk:** Low.
- **Owner:** Forge

---

## Legal Risks

### L1: AI-Generated Code IP / Copyright Uncertainty
- **Score:** L3 × I3 = **9 (Medium)**
- **Description:** Legal status of AI-generated code ownership is unsettled globally. Risk that factory output is not protectable by copyright, or infringes training data copyrights.
- **Trigger:** Legal challenge to AI-generated code ownership, regulatory ruling, competitor claiming infringement.
- **Mitigation:**
  - NZ Copyright Act s5(2)(a) protects computer-generated works — better position than US (research 20)
  - Products differentiated by execution, service, and data — not just code
  - Monitor legal developments via Law agent
  - Standard open-source licenses for dependencies properly attributed
  - Don't claim hand-written authorship for AI-generated code
- **Residual Risk:** Medium — legal landscape is evolving.
- **Owner:** Law

### L2: Regulatory Crackdown on AI-Operated Businesses
- **Score:** L2 × I5 = **10 (High)**
- **Description:** EU AI Act or similar regulation requires disclosure, human oversight, or restricts AI-operated commercial entities. Could require restructuring of factory operations.
- **Trigger:** EU AI Act enforcement, NZ following EU's lead, FTC enforcement actions, industry-specific regulation.
- **Mitigation:**
  - Human-in-the-loop maintained for key decisions (arch 12)
  - Document oversight processes and approval audit trails (arch 12)
  - Policy-as-code for all approval policies (arch 12)
  - Immutable audit trail for every agent action (arch 12)
  - NZ relatively light-touch, following EU with delay (research 20)
  - Adapt quickly: factory architecture supports adding gates/oversight
- **Residual Risk:** Medium — regulation is coming but timeline uncertain.
- **Owner:** Law

### L3: GDPR / Privacy Compliance
- **Score:** L3 × I4 = **12 (High)**
- **Description:** Products serving EU users must comply with GDPR. Agent-processed customer data could violate data minimization, purpose limitation, or cross-border transfer rules.
- **Trigger:** EU customer data processed by agents, data stored on NZ server, LLM API calls sending customer data to US providers.
- **Mitigation:**
  - Privacy-by-design: agents process minimum necessary data
  - Local inference for privacy-sensitive data (never leaves the box) (arch 13)
  - Standard privacy policies and ToS templates per product (research 20)
  - Cookie consent and data processing disclosures
  - No PII in agent prompts or logs where avoidable
  - Per-product data retention policies
- **Residual Risk:** Medium — GDPR compliance requires ongoing vigilance.
- **Owner:** Law / Dot

### L4: Contract / ToS Liability for AI Errors
- **Score:** L2 × I4 = **8 (Medium)**
- **Description:** AI-built software causes harm (data breach, incorrect outputs, financial loss) and the factory is held liable.
- **Trigger:** Security vulnerability in shipped product, incorrect AI output causing user harm, data breach.
- **Mitigation:**
  - Automated QA gates including security scanning (arch 09, research 13)
  - Standard limitation of liability in product ToS
  - E&O / professional liability insurance (research 20)
  - Products designed as tools (user responsible for decisions), not advice
  - Clear disclaimers on AI-generated content
  - Auto-rollback on deployment verification failure (arch 17)
- **Residual Risk:** Low-Medium — proper ToS and insurance limit exposure.
- **Owner:** Law

### L5: Platform ToS Violations
- **Score:** L3 × I3 = **9 (Medium)**
- **Description:** Automated activity violates terms of service of platforms used (Twitter API, Reddit, Google, Stripe, app stores), leading to account bans.
- **Trigger:** Automated posting detected and flagged, scraping beyond ToS limits, multiple automated accounts, app store policy violation.
- **Mitigation:**
  - Respect rate limits and platform policies by default (arch 17)
  - Per-domain rate limits on scraping (arch 17)
  - Legitimate business operations on each platform
  - Diversify: don't depend on any single platform for distribution
  - Monitor platform policy changes
- **Residual Risk:** Medium — platforms increasingly hostile to automation.
- **Owner:** Blaze / Law

---

## Market Risks

### M1: Market Saturation from Other AI Factories
- **Score:** L3 × I4 = **12 (High)**
- **Description:** Competitors (Devin, Factory AI, other indie builders) flood the same micro-SaaS and API niches with AI-built products, compressing margins and making customer acquisition harder.
- **Trigger:** AI tools becoming accessible to everyone, competitors targeting same niches, race to the bottom on pricing.
- **Mitigation:**
  - Speed advantage: ship in days, not months (research 20)
  - Portfolio diversification across verticals (research 20)
  - Focus on underserved niches, not tech-forward markets (research 19)
  - Self-improving factory gets better at picking winners (arch 19)
  - Build defensible moats: data advantages, network effects, brand (research 20)
  - First-mover advantage in specific niches
- **Residual Risk:** High — this is the biggest long-term market risk.
- **Owner:** Atlas / Scout

### M2: LLM Quality Plateau or Regression
- **Score:** L2 × I4 = **8 (Medium)**
- **Description:** LLM capabilities plateau or regress, limiting the quality of agent-generated code, content, and decisions. Factory output quality hits a ceiling.
- **Trigger:** Diminishing returns from model scaling, provider cost-cutting reducing quality, training data exhaustion.
- **Mitigation:**
  - Self-improvement system: factory gets better through playbooks, not just model quality (arch 19)
  - Knowledge distillation: teach cheap models to perform like expensive ones (arch 19)
  - Template-based scaffolding: deterministic skeleton + agent domain logic (arch 15)
  - Cross-model review catches model-specific blind spots (arch 15)
  - Local fine-tuning capability for domain-specific improvements
- **Residual Risk:** Medium — but the trajectory is still improving.
- **Owner:** Atlas / Rex

### M3: Google Algorithm Updates Killing SEO Revenue
- **Score:** L4 × I3 = **12 (High)**
- **Description:** Google penalizes AI-generated content, reducing organic traffic to content sites and SEO-driven products.
- **Trigger:** Google AI content detection improvements, algorithm update targeting programmatic SEO, E-E-A-T requirements strengthening.
- **Mitigation:**
  - Add unique data, tools, and user-generated content to content sites (research 11)
  - Don't rely solely on generated text — build genuinely useful tools (research 11)
  - Diversify traffic sources: don't depend only on Google (email, social, direct) (arch 16)
  - Revenue diversification: SEO is one channel, not the only one (arch 16)
  - Monitor algorithm changes and adapt quickly
- **Residual Risk:** Medium — Google risk is real but diversification helps.
- **Owner:** Echo / Blaze

### M4: Customer Trust Issues with AI-Operated Products
- **Score:** L2 × I3 = **6 (Medium)**
- **Description:** Customers discover products are AI-built/operated and lose trust, leading to churn or negative publicity.
- **Trigger:** Public disclosure, AI error causes visible quality issue, social media backlash.
- **Mitigation:**
  - Focus on value delivered, not how it's delivered (research 20)
  - Don't hide AI involvement but don't lead with it
  - Maintain high quality standards — AI-operated shouldn't mean low quality
  - Responsive support (even if AI-powered) builds trust
  - Genuine product quality is the best defence
- **Residual Risk:** Low — market is increasingly accepting of AI involvement.
- **Owner:** Chase / Echo

### M5: Pricing Race to Bottom
- **Score:** L3 × I3 = **9 (Medium)**
- **Description:** AI enables competitors to undercut on price, compressing margins across the portfolio.
- **Trigger:** Multiple AI-built competitors in same niche, customers choosing cheapest option.
- **Mitigation:**
  - Compete on quality and UX, not just price
  - Local inference keeps costs low (70-86% savings vs all-cloud) (arch 13)
  - Focus on niches where price is secondary to value (B2B, professional tools)
  - Build switching costs (data lock-in, integrations, workflow dependence)
  - A/B test pricing — most products are under-priced (research 11)
- **Residual Risk:** Medium.
- **Owner:** Chase / Finn

---

## Risk Heat Map

```
Impact →      1         2         3         4         5
            Negligible  Minor     Moderate  Major     Catastrophic
Likelihood
    5       │         │         │         │         │
  Almost    │         │         │         │         │
  Certain   │         │         │         │         │
            │─────────│─────────│─────────│─────────│─────────│
    4       │         │ T6      │ O2      │ T3, O1  │ T1, T2  │
  Likely    │         │         │         │         │         │
            │─────────│─────────│─────────│─────────│─────────│
    3       │         │ T9,T10  │ O3-O5   │ T4,M1   │ F1,T5   │
  Possible  │         │         │ L1,L5   │ L3,M3   │ F3      │
            │         │         │ M4,M5   │ F2      │         │
            │─────────│─────────│─────────│─────────│─────────│
    2       │         │ F5      │ O6      │ F4,L4   │ L2      │
  Unlikely  │         │         │ M4      │ M2      │         │
            │─────────│─────────│─────────│─────────│─────────│
    1       │         │         │         │         │         │
  Rare      │         │         │         │         │         │
            │─────────│─────────│─────────│─────────│─────────│
```

---

## Risk Response Summary

### Accept (Monitor Only)
- F5: Electricity cost escalation (Low, manageable)
- M4: Customer trust issues (Low, market trend is favorable)

### Mitigate (Active Controls)
- T1, T2: Cost blowout & prompt injection (Critical — multi-layer defences)
- T3, O1: Code quality & SPOF (Critical — guardrails + redundancy planning)
- F1, F3: Revenue & payment risks (High — diversification + portfolio approach)
- T5: Secret leakage (High — vault + output scanning)
- O2: Over-engineering (High — discipline + milestones)
- L2, L3: Regulatory & GDPR (High — compliance-first design)

### Transfer
- L4: Contract liability → insurance (E&O / professional liability)
- F4: Hardware failure → cloud fallback + git distribution

### Avoid
- No risks currently warrant full avoidance — all are manageable with mitigation.

---

## Review Schedule

| Review | Frequency | Owner |
|--------|-----------|-------|
| Risk register update | Monthly | Atlas |
| Critical risk review | Weekly (first 3 months) | Kev + Atlas |
| Financial risk check | Weekly | Finn / Dash |
| Security risk assessment | Monthly | Hawk |
| Legal/regulatory scan | Monthly | Law |
| Post-incident risk update | Per incident | Responsible owner |

---

*Risk Register v1.0 — 2026-02-08*
*Next review: 2026-03-08*
