# 19 — Self-Improvement & Learning System

> How the factory gets better over time — automatically.

**Date:** 2026-02-08
**Status:** Architecture Document
**Dependencies:** Memory & Knowledge (08), Factory Patterns (07), QA System, Agent Orchestration

---

## 1. Core Thesis

The factory's most important property isn't how good it is today — it's **how fast it improves**. Every failed build, every rejected PR, every customer complaint is a training signal. A factory that learns from 100 failures will outperform one that never fails but never learns.

The self-improvement system is the factory's immune system + nervous system: it detects problems, diagnoses root causes, and patches itself — ideally before a human notices.

---

## 2. The Improvement Loop

```
┌─────────────────────────────────────────────────────┐
│                                                     │
│   Observe → Measure → Diagnose → Patch → Verify    │
│       ▲                                    │        │
│       └────────────────────────────────────┘        │
│                                                     │
└─────────────────────────────────────────────────────┘
```

Every cycle tightens the loop. The system improves along five axes:

1. **Agent prompts & instructions** — what agents are told to do
2. **Agent configurations** — which models, temperature, context strategies
3. **Playbooks & templates** — reusable procedures for common tasks
4. **Tooling & infrastructure** — the scaffolding agents operate within
5. **Opportunity selection** — which businesses to build (portfolio intelligence)

---

## 3. The Signals Pattern (Friction Detection)

Adapted from Factory AI's architecture. This is the heart of self-improvement.

### 3.1 What Are Signals?

Signals are structured observations extracted from agent sessions by an LLM judge. Every agent session produces a stream of events — the Signals system watches that stream and tags moments of **friction** (something went wrong) or **delight** (something went unusually well).

### 3.2 Signal Detection Pipeline

```
Agent Session Events (logs, tool calls, outputs, errors, timing)
        │
        ▼
┌──────────────────────┐
│   Signal Extractor   │  ← LLM-as-judge, runs post-session
│   (cheap model)      │     or streaming for long sessions
└──────────┬───────────┘
           │
           ▼
┌──────────────────────┐
│   Signal Store       │  ← Structured DB: session_id, signal_type,
│                      │     severity, category, description, context
└──────────┬───────────┘
           │
           ▼
┌──────────────────────┐
│   Pattern Detector   │  ← Aggregates signals across sessions
│                      │     Finds recurring friction patterns
└──────────┬───────────┘
           │
           ▼
┌──────────────────────┐
│   Patch Generator    │  ← Proposes fixes (prompt edits, config
│                      │     changes, new playbook steps)
└──────────┬───────────┘
           │
           ▼
┌──────────────────────┐
│   Verification       │  ← A/B test or replay to confirm fix works
└──────────────────────┘
```

### 3.3 Signal Types

| Signal | Detection Method | Example |
|---|---|---|
| **Error loop** | Agent retries same action 3+ times | Build fails, agent keeps running `npm install` with same broken config |
| **Context thrashing** | Agent re-reads same files repeatedly | Can't find the right code, keeps opening/closing files |
| **Escalation** | Agent asks for human help | "I'm not sure how to proceed" |
| **Timeout** | Task exceeds expected duration by 2x+ | Simple bug fix taking 45 minutes |
| **Regression** | Previously passing test now fails | Deployment broke after "improvement" |
| **Quality rejection** | QA agent or human rejects output | PR review flags major issues |
| **Cost spike** | Token usage far exceeds baseline | Agent dumping entire codebase into context |
| **Delight** | Task completed faster/better than baseline | 5-minute fix for a type of bug that usually takes 30 |

### 3.4 Emergent Category Discovery

Fixed taxonomies go stale. Instead, the system discovers new friction categories:

1. **Embed** all signal descriptions into vector space
2. **Cluster** embeddings periodically (weekly)
3. **Label** new clusters with LLM: "What pattern do these 47 signals share?"
4. **Propose** new signal categories to the taxonomy
5. **Human review** (initially) → auto-accept (once trust is established)

Example emergent categories discovered over time:
- "branch pollution" — agents creating too many orphan branches
- "dependency guessing" — agents trying random package versions
- "test theatre" — agents writing tests that pass but don't test anything meaningful
- "context window stuffing" — agents loading everything hoping something is relevant

---

## 4. Feedback Loops

### 4.1 QA Failure → Prompt Improvement

The tightest, most valuable loop.

```
Build Agent produces code
        │
        ▼
QA Agent runs tests, linting, security scan
        │
        ├── PASS → record success pattern
        │
        └── FAIL → Failure Analysis Agent
                        │
                        ▼
                Classify failure:
                  ├── Code bug → log, no prompt change needed
                  ├── Misunderstood requirements → improve task decomposition prompt
                  ├── Wrong architecture choice → add to architectural playbook
                  ├── Missing context → improve context retrieval strategy
                  ├── Known anti-pattern → add to "don't do this" examples
                  └── Novel failure → flag for human review, add to knowledge base
```

**Implementation:**

```typescript
interface QAFailure {
  task_id: string;
  agent_id: string;
  failure_type: "test" | "lint" | "security" | "performance" | "review";
  description: string;
  code_diff: string;
  agent_prompt_version: string;
  root_cause?: string;  // filled by analysis agent
  prompt_patch?: string; // proposed prompt improvement
}

// After N failures of the same root_cause:
// 1. Generate prompt patch
// 2. A/B test patch vs current prompt
// 3. If patch wins, promote to production
```

### 4.2 Customer Feedback → Product Improvement

```
Support Agent receives complaint/request
        │
        ▼
Categorize: bug | feature request | confusion | churn risk
        │
        ▼
Route:
  ├── Bug → create ticket → Build Agent fixes → QA validates
  ├── Feature request → score against roadmap → queue or reject
  ├── Confusion → improve docs/onboarding → Copy Agent updates
  └── Churn risk → alert human portfolio manager
```

### 4.3 Business Metrics → Strategy Improvement

```
Revenue, churn, acquisition cost, time-to-first-user
        │
        ▼
Portfolio Intelligence Agent
        │
        ├── Winning patterns → weight opportunity scorer toward similar niches
        ├── Losing patterns → add to anti-patterns (avoid these markets)
        ├── Launch channel effectiveness → adjust launch playbook weights
        └── Pricing signals → update pricing strategy templates
```

### 4.4 Build Metrics → Process Improvement

Track per-task and per-project:

| Metric | What It Tells You |
|---|---|
| Time to completion | Are agents getting faster? |
| Token cost per task | Are agents getting more efficient? |
| First-pass success rate | Are agents getting it right the first time? |
| QA rejection rate | Is code quality improving? |
| Human intervention rate | Is autonomy increasing? |
| Lines of code per task | Are agents over/under-engineering? |
| Test coverage of output | Are agents writing meaningful tests? |

**Trend analysis:** If any metric regresses for 3+ consecutive measurement periods, trigger investigation signal.

---

## 5. A/B Testing Agent Configurations

### 5.1 What Can Be A/B Tested

| Parameter | Example Variants |
|---|---|
| System prompt | v1 (concise) vs v2 (detailed with examples) |
| Model | claude-sonnet vs gpt-4o vs deepseek-r1 |
| Temperature | 0.0 vs 0.3 vs 0.7 |
| Context strategy | full-file vs chunk-relevant vs summary-then-detail |
| Tool availability | all tools vs restricted toolset |
| Planning approach | plan-then-execute vs iterative |
| Review mode | self-review vs separate review agent |

### 5.2 A/B Testing Framework

```
┌─────────────────────────────────────┐
│         Experiment Registry         │
│  ┌───────────────────────────────┐  │
│  │ experiment_id: "prompt-v12"   │  │
│  │ parameter: "system_prompt"    │  │
│  │ variants: [A, B]             │  │
│  │ traffic_split: 50/50         │  │
│  │ success_metric: "qa_pass_rate"│  │
│  │ min_samples: 30              │  │
│  │ status: "running"            │  │
│  └───────────────────────────────┘  │
└─────────────────────────────────────┘
          │
          ▼
    Task Router
    ├── Task 1 → Variant A
    ├── Task 2 → Variant B
    ├── Task 3 → Variant A
    └── ...
          │
          ▼
    Results Collector
    ├── Quality score (QA pass rate, review score)
    ├── Speed (time to completion)
    ├── Cost (tokens used)
    └── Human satisfaction (if applicable)
          │
          ▼
    Statistical Analysis
    ├── Sufficient samples? → Continue
    ├── Clear winner (p < 0.05)? → Promote winner
    └── No difference? → Keep cheaper/faster variant
```

### 5.3 Multi-Armed Bandit for Model Selection

Rather than fixed A/B splits, use Thompson Sampling to dynamically allocate traffic:

- New task arrives → sample from each model's Beta distribution → pick highest
- Observe result → update that model's success/failure counts
- Over time, traffic naturally flows to the best model per task type

**Key insight:** Different models may be best for different task types. The bandit should be **per-task-category**, not global:
- Bug fixes → Model A might win
- Greenfield features → Model B might win
- Test writing → Model C might win

---

## 6. Knowledge Distillation

### 6.1 Why Distill?

Expensive models (Claude Opus, GPT-4) produce better results but cost 10-30x more than cheap models (Claude Haiku, GPT-4o-mini). If you can teach the cheap model to perform like the expensive one on specific tasks, you get the quality at a fraction of the cost.

### 6.2 Distillation Pipeline

```
Phase 1: Collect Training Data
  ├── Run expensive model on N tasks
  ├── Filter for high-quality outputs (QA pass, human approved)
  └── Store as (input, output) pairs with full context

Phase 2: Fine-Tune or Prompt-Engineer Cheap Model
  ├── Option A: Fine-tune (if provider supports it)
  │   └── Upload pairs → fine-tune API → custom model
  ├── Option B: Few-shot prompt engineering
  │   └── Select best examples → embed in system prompt
  └── Option C: Distilled playbooks
      └── Extract patterns from expensive model's outputs
          → codify as explicit instructions for cheap model

Phase 3: Validate
  ├── Run cheap model on held-out test tasks
  ├── Compare quality metrics vs expensive model
  ├── If quality gap < threshold → deploy cheap model
  └── If quality gap > threshold → iterate or keep expensive model
```

### 6.3 Progressive Delegation

```
Task difficulty estimator
        │
        ├── Simple (boilerplate, formatting, standard patterns)
        │   └── Cheap model (Haiku/4o-mini)
        │
        ├── Medium (feature implementation, bug fixes)
        │   └── Mid-tier model (Sonnet/4o)
        │
        └── Complex (architecture, novel problems, multi-file refactors)
            └── Expensive model (Opus/o1-pro)
```

Over time, the difficulty boundaries shift downward as cheap models get fine-tuned on accumulated examples from expensive models:

```
Month 1:  Simple ──────|── Medium ──────|── Complex
          [Haiku 40%]    [Sonnet 40%]     [Opus 20%]

Month 6:  Simple ─────────────|── Medium ────|── Complex  
          [Haiku 60%]           [Sonnet 30%]   [Opus 10%]

Month 12: Simple ──────────────────|── Medium |── Complex
          [Haiku 75%]                [Sonnet 20%] [Opus 5%]
```

**Cost impact:** If average task cost starts at $0.50 and shifts to $0.15 through distillation, a factory running 1000 tasks/day saves $10K/month.

### 6.4 Playbook Distillation

The most practical near-term approach. Instead of fine-tuning models:

1. Expensive model solves a class of problem successfully 10+ times
2. Analysis agent extracts the common approach, decision points, pitfalls
3. Codify as a **playbook** — structured instructions any model can follow
4. Test playbook with cheap model
5. Iterate until cheap model + playbook ≈ expensive model freeform

Example: "Expensive model keeps choosing to split the React component into a container + presentational pattern when components exceed 200 lines. Codify this as a playbook rule."

---

## 7. Learning from Mistakes

### 7.1 The Post-Mortem Agent

Every significant failure triggers an automated post-mortem:

```
Failure detected (deploy failure, customer data loss, revenue drop, etc.)
        │
        ▼
Post-Mortem Agent:
  1. Reconstruct timeline from logs and signals
  2. Identify root cause (or top 3 candidates)
  3. Classify: preventable? systemic? one-off?
  4. Generate remediation:
     ├── Immediate: rollback, hotfix
     ├── Short-term: add test, update prompt, add guard
     └── Long-term: architectural change, new playbook
  5. Store in Knowledge Graph:
     failure → caused_by → root_cause
     root_cause → prevented_by → remediation
     remediation → applied_to → [agents/prompts/configs]
```

### 7.2 Anti-Pattern Library

A living, growing collection of "don't do this" patterns:

```yaml
anti_patterns:
  - id: "ap-001"
    name: "Infinite retry loop"
    description: "Agent retries failed operation without changing approach"
    detection: "Same tool call with same args > 3 times in 60 seconds"
    remedy: "After 2 failures, agent must try alternative approach or escalate"
    added_date: "2026-02-15"
    source: "Signal cluster analysis, 47 occurrences in week 2"
    
  - id: "ap-002"
    name: "Context window stuffing"
    description: "Agent loads entire files when only a function is needed"
    detection: "File read operations > 10 in single planning step"
    remedy: "Use targeted search (grep/AST) before full file reads"
    added_date: "2026-02-22"
    source: "Cost spike analysis, avg 3x token overspend"
```

Anti-patterns are injected into agent system prompts as negative examples. The library grows automatically from Signal cluster analysis.

### 7.3 Success Pattern Library

Mirror of anti-patterns — things that work well:

```yaml
success_patterns:
  - id: "sp-001"
    name: "Test-first for bug fixes"
    description: "Write failing test reproducing bug before attempting fix"
    evidence: "72% higher first-pass success rate vs fix-then-test"
    applies_to: ["bug_fix"]
    discovered: "A/B experiment prompt-v8, 200 samples"
    
  - id: "sp-002"
    name: "Architecture sketch before code"
    description: "Generate file/function structure outline before writing code"
    evidence: "40% fewer refactor cycles on medium+ complexity tasks"
    applies_to: ["feature", "greenfield"]
    discovered: "Signal analysis, delight signals correlated with planning step"
```

---

## 8. The Meta-Learning Loop

The self-improvement system must also improve itself.

### 8.1 Improvement Metrics

| Metric | Target Trend | Measures |
|---|---|---|
| Mean time to detect friction | ↓ Decreasing | How fast do we spot problems? |
| Mean time to patch | ↓ Decreasing | How fast do we fix them? |
| Patch success rate | ↑ Increasing | Do our fixes actually work? |
| False positive signal rate | ↓ Decreasing | Are we flagging real problems? |
| Agent improvement velocity | ↑ Increasing | How much better per week? |
| Cost per improvement cycle | ↓ Decreasing | Is learning getting cheaper? |

### 8.2 Guardrails

Self-improvement without guardrails is dangerous. The system could optimize for metrics while degrading actual quality.

**Hard limits:**
- No prompt change can be auto-deployed without A/B validation on ≥30 samples
- No model downgrade without quality gate verification
- Human review required for: architectural changes, security-related patches, changes to the self-improvement system itself
- Rollback any change that degrades primary metrics by >5% within 24 hours
- Maximum 3 concurrent experiments per agent role (avoid confounding)

**Goodhart's Law protection:**
- Multiple metrics, not single optimization target
- Periodic human review of "improved" outputs (do they actually feel better?)
- Track proxy metrics AND downstream outcomes (QA pass rate is a proxy; customer satisfaction is the outcome)
- Adversarial review: periodically challenge the system to find cases where metrics improved but quality didn't

---

## 9. Implementation Phases

### Phase 1: Instrument (Week 1-2)
- Add structured logging to all agent sessions (tool calls, timing, token counts, outcomes)
- Define the core metrics table (Section 4.4)
- Build the metrics dashboard
- Store all session data for retrospective analysis

### Phase 2: Detect (Week 3-4)
- Build the Signal Extractor (LLM-as-judge over session logs)
- Define initial signal taxonomy (Section 3.3)
- Set up the Signal Store (structured DB)
- Build basic alerting (signal rate exceeds threshold → notify human)

### Phase 3: Diagnose (Week 5-6)
- Build the Pattern Detector (cluster signals, find recurring patterns)
- Implement the Post-Mortem Agent for significant failures
- Start the Anti-Pattern Library with manually identified patterns
- Connect QA failures to the diagnosis pipeline

### Phase 4: Patch (Week 7-8)
- Build the A/B testing framework (Section 5.2)
- Implement the Patch Generator (propose prompt/config changes from signals)
- Run first experiments: prompt variants on real tasks
- Build the rollback mechanism

### Phase 5: Learn (Week 9-12)
- Implement emergent category discovery (Section 3.4)
- Build the knowledge distillation pipeline (Section 6)
- Deploy progressive delegation (cheap model for easy tasks)
- Close the full loop: detect → diagnose → patch → verify → promote

### Phase 6: Meta-Learn (Ongoing)
- Track improvement metrics (Section 8.1)
- Tune the Signal Extractor's own prompts based on false positive rates
- Expand A/B testing to more parameters
- Human review cadence: weekly initially → monthly as trust builds

---

## 10. Data Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    Data Stores                          │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  Session Logs (append-only)                             │
│  ├── tool_calls, timing, tokens, errors, outcomes       │
│  └── Storage: structured logs (JSON) → object store     │
│                                                         │
│  Signal Store (structured DB)                           │
│  ├── signal_type, severity, category, session_id        │
│  ├── description, context, proposed_action              │
│  └── Storage: PostgreSQL                                │
│                                                         │
│  Experiment Registry (structured DB)                    │
│  ├── experiment_id, variants, metrics, status           │
│  ├── results per variant per sample                     │
│  └── Storage: PostgreSQL                                │
│                                                         │
│  Pattern Library (knowledge graph + files)              │
│  ├── anti-patterns, success patterns, playbooks         │
│  ├── relationships: pattern → agents → outcomes         │
│  └── Storage: Neo4j + markdown files (human-readable)   │
│                                                         │
│  Distillation Store (vector DB + files)                 │
│  ├── (input, output) training pairs from expensive models│
│  ├── playbook extractions                               │
│  └── Storage: Qdrant + markdown playbooks               │
│                                                         │
│  Metrics Store (time-series)                            │
│  ├── per-agent, per-task-type, per-model performance    │
│  └── Storage: TimescaleDB or Prometheus                 │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

---

## 11. Key Design Decisions

### Why LLM-as-Judge (Not Rules)?

Rules catch known patterns. LLM judges catch novel friction. The signal extractor sees things like "the agent apologized for confusion 4 times in this session" — you'd never write a rule for that, but an LLM recognizes it instantly.

**Cost:** ~$0.01-0.05 per session analysis using a cheap model (Haiku/4o-mini). At 1000 sessions/day = $10-50/day. Trivial compared to the value of improvement.

### Why A/B Test (Not Just Deploy)?

Intuition about what makes a better prompt is unreliable. "More detailed instructions" sometimes helps, sometimes hurts. "Add examples" sometimes helps, sometimes causes the model to over-index on the examples. Only measurement tells the truth.

### Why Playbook Distillation (Not Just Fine-Tuning)?

Fine-tuning requires:
- API support (not all providers offer it)
- Enough training data (hundreds of examples minimum)
- Risk of catastrophic forgetting
- Retraining when the base model updates

Playbook distillation works with any model, any provider, immediately, and is human-readable/auditable.

### Why Emergent Categories (Not Fixed Taxonomy)?

A fixed taxonomy assumes you know all the failure modes upfront. You don't. The most interesting signals are the ones you didn't expect — "test theatre" (agents writing meaningless tests) was not in anyone's initial taxonomy, but it's a real pattern the system should discover on its own.

---

## 12. Success Criteria

The self-improvement system is working when:

1. **Agent first-pass success rate increases month-over-month** — fewer QA rejections, fewer human interventions
2. **Cost per task decreases** — through distillation and progressive delegation
3. **New failure modes are detected within days, not weeks** — the signal system surfaces problems early
4. **Patches are validated and deployed without human involvement** (for low-risk changes) — the loop closes automatically
5. **The factory's hit rate on business selection improves** — portfolio intelligence learns from outcomes
6. **Human time spent on factory operations decreases** — from hours/day to hours/week to hours/month

The ultimate measure: **the factory 6 months from now should be embarrassingly better than the factory today, and nobody had to manually make most of those improvements.**

---

*Last updated: 2026-02-08*
