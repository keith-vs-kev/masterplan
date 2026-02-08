# Week 1 Sprint Plan — The Foundation

> **Sprint:** 2026-02-09 (Mon) → 2026-02-13 (Fri)
> **Goal:** Task queue + orchestration core + first automated pipeline running end-to-end
> **Friday Demo:** Adam triggers a product idea → Scout researches → Rex scaffolds → Hawk validates → output lands in review queue. All autonomous.

---

## Sprint Philosophy

Ship the critical path first. Per the Integration Blueprint (doc 20):

```
Task Queue → Kev Routing → Pi SDK → First Scout→Rex pipeline → First shipped product
```

Everything else is a force multiplier but not a blocker. No NATS, no ChromaDB, no dashboards this week. Those come later.

---

## Monday — Task Queue & State Machine

### Who Builds What

| Owner | Deliverable | Model |
|-------|-------------|-------|
| **Rex** | SQLite task store: schema, migrations, CRUD operations | Claude Code |
| **Rex** | Task state machine (CREATED → READY → CLAIMED → IN_PROGRESS → REVIEW → DONE / FAILED) | Claude Code |
| **Atlas** | Task JSON schema & API surface docs | Claude Opus |

### What Gets Built

**SQLite Task Store** (`/home/adam/agents/shared/queue/tasks.db`):

```sql
tasks (id, title, type, status, priority, agent_id, parent_id,
       dependencies[], budget_usd, tokens_used, cost_usd,
       input_json, output_json, artifacts[], error,
       created_at, claimed_at, completed_at)
```

**Task CLI** (`task-cli` or Node module):
- `task create` — create task with metadata
- `task list` — filter by status, agent, priority
- `task claim <id> <agent>` — atomic claim (SQLite transaction)
- `task update <id>` — progress, artifacts
- `task complete <id>` / `task fail <id>` — terminal states
- `task dag <parent>` — show dependency tree

### Dependencies
- None (day 1 = greenfield)

### Definition of Done
- [ ] SQLite schema created and migrated
- [ ] All state transitions enforced (no illegal jumps)
- [ ] Atomic task claiming (two agents can't claim same task)
- [ ] CLI or module can create, list, claim, complete, fail tasks
- [ ] 5+ unit tests passing for state machine logic
- [ ] Committed to `masterplan` repo or new `factory-core` repo

---

## Tuesday — Kev Orchestration Logic & Cost Pyramid

### Who Builds What

| Owner | Deliverable | Model |
|-------|-------------|-------|
| **Rex** | Kev's task routing engine — reads queue, matches agent+model, dispatches | Claude Code |
| **Rex** | Cost pyramid router (Cerebras free → local → cloud mid → cloud frontier) | Claude Code |
| **Atlas** | Agent capability registry (YAML config: agent → skills, models, budget) | Claude Opus |

### What Gets Built

**Agent Registry** (`/home/adam/agents/shared/config/agents.yaml`):
```yaml
agents:
  scout:
    capabilities: [research, competitor-analysis, market-sizing]
    default_model: cerebras/gpt-oss-120b  # free tier
    fallback_model: gemini/gemini-2.5-flash
    max_concurrent: 2
    daily_budget_usd: 5.00

  rex:
    capabilities: [code-generation, scaffolding, refactoring, testing]
    default_model: claude-code  # via Pi SDK
    fallback_model: codex
    max_concurrent: 1
    daily_budget_usd: 20.00

  hawk:
    capabilities: [code-review, testing, security-scan, qa]
    default_model: claude/claude-sonnet-4-5
    fallback_model: gemini/gemini-2.5-flash
    max_concurrent: 2
    daily_budget_usd: 10.00
```

**Cost Pyramid Logic** (in Kev's routing):
```
1. Can a free model do this? (Cerebras, Gemini free tier) → use it
2. Can local inference do this? (Ollama) → use it
3. Is this medium complexity? → cloud/mid (Gemini Flash, GPT-4o-mini)
4. Complex/critical? → cloud/frontier (Claude Sonnet/Opus)
```

**Task Routing** (Kev reads `tasks` where status=READY):
1. Check task type → match to agent capabilities
2. Check agent availability (not at max_concurrent)
3. Select model via cost pyramid based on task complexity
4. Dispatch: OpenClaw subagent spawn OR Pi SDK session

### Dependencies
- Monday's task queue (reads from it)

### Definition of Done
- [ ] Agent registry YAML loaded and parsed
- [ ] Kev can read READY tasks and match to agents
- [ ] Cost pyramid selects cheapest viable model per task type
- [ ] Dispatch creates OpenClaw subagent session with correct context
- [ ] Budget tracking: tokens and cost recorded per task
- [ ] Manual test: create task → Kev picks it up → dispatches to agent

---

## Wednesday — Scout + Rex Pipeline (First Automated Chain)

### Who Builds What

| Owner | Deliverable | Model |
|-------|-------------|-------|
| **Rex** | Scout agent prompt & research workflow (takes idea → outputs research brief) | Claude Code |
| **Rex** | Rex scaffolding workflow (takes research brief → outputs project skeleton) | Claude Code |
| **Kev** | DAG wiring: idea task → scout subtask → rex subtask (auto-triggered) | Kev orchestration |

### What Gets Built

**Scout Research Task Flow:**
1. Receives task: `{type: "research", input: {idea: "...", market: "..."}}`
2. Uses Cerebras (free) for initial research fan-out
3. Outputs: `research-brief.md` with market size, competitors, feasibility, recommended approach
4. Marks task DONE with artifact path

**Rex Scaffold Task Flow:**
1. Triggered when scout's research task completes (DAG dependency)
2. Reads research brief as context
3. Scaffolds project using template system (doc 15):
   - `ts-api` or `ts-nextjs` template selected based on brief
   - Generates domain-specific code on top of scaffold
4. Runs quality gates: `typecheck + lint + test + build`
5. Commits to git, creates branch
6. Marks task DONE with repo path + branch

**DAG Definition:**
```
product-idea (parent)
├── research (scout) — no deps
└── scaffold (rex) — depends on: research
    └── qa-review (hawk) — depends on: scaffold [Thursday]
```

### Dependencies
- Monday's task queue (task creation + state machine)
- Tuesday's routing (Kev dispatches scout, then rex on completion)

### Definition of Done
- [ ] Scout produces a research brief from a product idea (Cerebras)
- [ ] Rex reads scout's output and scaffolds a real project
- [ ] Project passes `typecheck + lint + build` (test optional if no domain logic yet)
- [ ] DAG auto-triggers: research.DONE → scaffold.READY → scaffold.CLAIMED
- [ ] Full chain runs with one trigger (Adam drops an idea, pipeline executes)
- [ ] Output committed to a git repo with clean branch

---

## Thursday — Hawk QA Gate & Review Queue

### Who Builds What

| Owner | Deliverable | Model |
|-------|-------------|-------|
| **Rex** | Hawk QA agent workflow (reviews Rex's output, runs tests, reports) | Claude Code |
| **Rex** | Review queue view (tasks in REVIEW status, with pass/fail/notes) | Claude Code |
| **Atlas** | QA checklist & quality criteria doc | Claude Opus |

### What Gets Built

**Hawk QA Task Flow:**
1. Triggered when Rex's scaffold task completes (DAG dependency)
2. Clones/reads Rex's output branch
3. Runs automated checks:
   - `typecheck` (strict mode)
   - `lint` (zero warnings)
   - `test` (if tests exist)
   - `build` (must succeed)
   - Basic security scan (no secrets in code, no `eval()`)
4. Uses a *different model* than Rex (Gemini or Sonnet if Rex used Opus) for cross-model review
5. Writes QA report: pass/fail per check, issues found, confidence score
6. If PASS → task moves to REVIEW (for Adam's final look) or auto-DONE
7. If FAIL → task moves to FAILED with feedback, re-queued for Rex

**Review Queue:**
- Simple CLI or markdown view: `task list --status=review`
- Shows: task title, agent who built it, Hawk's QA verdict, link to code
- Adam can approve (→ DONE) or reject (→ back to READY with notes)

**Full DAG now:**
```
product-idea
├── research (scout)
└── scaffold (rex) ← depends: research
    └── qa-review (hawk) ← depends: scaffold
        └── [human review if needed]
```

### Dependencies
- Wednesday's Scout→Rex pipeline (Hawk reviews Rex's output)
- Tuesday's routing (dispatches Hawk)

### Definition of Done
- [ ] Hawk receives Rex's output and runs all quality checks
- [ ] Cross-model review: Hawk uses different LLM than Rex used
- [ ] QA report generated with per-check pass/fail
- [ ] Failed tasks re-queued with actionable feedback
- [ ] Passed tasks visible in review queue for Adam
- [ ] Full pipeline: idea → scout → rex → hawk → review queue

---

## Friday — Integration, Polish & Demo

### Who Builds What

| Owner | Deliverable | Model |
|-------|-------------|-------|
| **Rex** | Bug fixes, edge cases, retry logic for the full pipeline | Claude Code |
| **Atlas** | Sprint retrospective doc + Week 2 sprint plan outline | Claude Opus |
| **Kev** | Demo run: live pipeline execution with a real product idea | Kev orchestration |

### What Gets Built

**Morning: Integration hardening**
- Error handling: what happens when Scout fails? When Rex's build breaks? When Cerebras is down?
- Retry logic: failed tasks retry once with same model, then escalate to better model
- Timeout handling: tasks stuck >30 min get killed and re-queued
- Cost tracking summary: "Today's pipeline cost $X across Y tasks"

**Afternoon: Demo prep & execution**
- Pick 2-3 real micro-SaaS ideas from the research backlog
- Run them through the full pipeline live
- Document what worked, what broke, what's slow

### Demo Script (Friday afternoon)

Adam does this:
1. **Triggers idea:** "Build a Markdown-to-PDF API" (or similar micro-SaaS)
2. **Watches pipeline:**
   - Scout researches: competitors, market, feasibility (~2-5 min)
   - Rex scaffolds: project structure, API routes, basic logic (~5-10 min)
   - Hawk reviews: type-check, lint, build, cross-model review (~3-5 min)
3. **Reviews output:** Opens review queue, sees QA report, browses generated code
4. **Verdict:** Approve, reject with notes, or flag for iteration

**Expected demo duration:** 15-20 minutes end-to-end

### Definition of Done
- [ ] Full pipeline runs 2+ times without manual intervention
- [ ] Error recovery works (at least one graceful failure + retry)
- [ ] Cost for full pipeline run logged and visible
- [ ] Adam can trigger a pipeline from WhatsApp and see results
- [ ] Sprint retro doc written with wins, issues, Week 2 priorities

---

## Sprint-Wide Constraints

### Budget
| Resource | Daily Cap | Weekly Cap |
|----------|-----------|------------|
| Cloud LLM (Anthropic/OpenAI) | $30 | $150 |
| Free tier (Cerebras/Gemini) | Unlimited | Unlimited |
| Local inference (Ollama) | $0 | $0 |

### Tech Stack (This Week Only)
- **Language:** TypeScript (strict)
- **DB:** SQLite (via better-sqlite3 or drizzle)
- **Runtime:** Node.js on dreamteam
- **Models:** Cerebras (free), Claude Code (builds), Claude Sonnet/Gemini (reviews)
- **Deployment:** None this week — everything runs locally

### What We're NOT Building This Week
- ❌ NATS message bus (doc 11) — filesystem + SQLite is enough for now
- ❌ ChromaDB / vector memory (doc 05) — agents use markdown files
- ❌ REEF dashboard (doc 20) — CLI views are fine
- ❌ MCP gateway (doc 14) — agents use tools directly
- ❌ Deployment pipeline (doc 06) — no products ship to production yet
- ❌ Marketing pipeline (doc 16) — nothing to market yet
- ❌ LiteLLM proxy (doc 13) — direct API calls, add proxy later
- ❌ Human approval tiers (doc 12) — simple approve/reject for now

### What We ARE Deferring Intentionally
These are important but Week 2+:
- Git worktree parallel builds (doc 15) — sequential is fine for 2-3 tasks/day
- A/B testing agent configs (doc 19) — need baseline data first
- Budget envelopes per agent (doc 12) — daily cap is enough for now
- Continuous aggregates (doc 18) — just log costs to SQLite

---

## Risk Register

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| Cerebras free tier hits rate limits | Medium | Scout slows down | Fallback to Gemini free tier |
| Rex scaffold fails typecheck | High | Pipeline stalls at Rex | Rex retries with error context (self-heal loop, max 3) |
| SQLite concurrency issues | Low | Task claiming race conditions | WAL mode + immediate transactions |
| Scope creep (building infra before pipeline) | High | Nothing demos Friday | Hard rule: if it doesn't help the demo, defer it |
| Pi SDK/Claude Code session instability | Medium | Builds fail randomly | Retry with fresh session, log for diagnosis |

---

## Success Criteria

By Friday 5pm:

1. ✅ **Pipeline works:** Idea → Scout → Rex → Hawk → Review queue, fully automated
2. ✅ **It's repeatable:** Run it 2+ times with different ideas, same pipeline
3. ✅ **Costs are tracked:** Know exactly what the pipeline costs per run
4. ✅ **Adam can use it:** Trigger from WhatsApp, review output, approve/reject
5. ✅ **Code quality:** Generated projects pass typecheck + lint + build

If all five are green, Week 1 is a success. Everything else is gravy.

---

*Sprint plan authored: 2026-02-08*
*Next: Week 2 focuses on deployment pipeline (Forge), first production ship, and cost optimization.*
