# 06 — CI/CD & Deployment Pipeline

> From agent-generated code to production in minutes, with guardrails at every stage.

---

## Overview

The CI/CD pipeline is the factory's assembly line. Agents generate code continuously — the pipeline ensures only quality code reaches production. Every commit flows through a deterministic gauntlet: lint → type-check → test → scan → build → preview → E2E → deploy → monitor. No shortcuts, no bypasses.

**Design principles:**
1. **Agents trigger, pipelines decide** — agents push code; the pipeline determines if it ships
2. **Fast feedback, deep verification** — quick gates first (seconds), heavy analysis later (minutes)
3. **GitOps as the single source of truth** — deployment state lives in git, not in agent memory
4. **Progressive delivery** — preview → staging → canary → production
5. **Auto-rollback by default** — every deployment is reversible without human intervention

---

## Pipeline Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                     AGENT GENERATES CODE                         │
│              (Claude Code / Codex / Forge agents)                │
└────────────────────────────┬────────────────────────────────────┘
                             │ git push / PR created
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│  STAGE 1: FAST GATES  (~30s)                                     │
│                                                                   │
│  ┌────────────┐ ┌────────────┐ ┌────────────┐ ┌──────────────┐  │
│  │ Formatting │ │  Linting   │ │ Type Check │ │   Secrets    │  │
│  │ (Prettier/ │ │ (ESLint/   │ │ (tsc/mypy  │ │  Detection   │  │
│  │  Ruff fmt) │ │  Ruff)     │ │  --strict) │ │ (Gitleaks)   │  │
│  └────────────┘ └────────────┘ └────────────┘ └──────────────┘  │
│                                                                   │
│  Any failure → reject immediately, feed errors back to agent     │
└────────────────────────────────┬────────────────────────────────┘
                                 ▼
┌─────────────────────────────────────────────────────────────────┐
│  STAGE 2: TEST EXECUTION  (~2-10min)                             │
│                                                                   │
│  ┌────────────┐ ┌────────────┐ ┌────────────┐ ┌──────────────┐  │
│  │   Unit     │ │ Property-  │ │Integration │ │  Snapshot    │  │
│  │   Tests    │ │  Based     │ │   Tests    │ │   Tests     │  │
│  │ (Vitest/   │ │ (fast-     │ │ (API +DB   │ │ (Jest/      │  │
│  │  pytest)   │ │  check)    │ │  contracts)│ │  insta)     │  │
│  └────────────┘ └────────────┘ └────────────┘ └──────────────┘  │
│                                                                   │
│  Coverage gate: new code ≥90% line, ≥80% branch                 │
│  Coverage must not decrease overall                               │
└────────────────────────────────┬────────────────────────────────┘
                                 ▼
┌─────────────────────────────────────────────────────────────────┐
│  STAGE 3: SECURITY & DEEP ANALYSIS  (~5-15min)                   │
│                                                                   │
│  ┌────────────┐ ┌────────────┐ ┌────────────┐ ┌──────────────┐  │
│  │   SAST     │ │    SCA     │ │  Mutation  │ │Architecture │  │
│  │ (Semgrep/  │ │ (npm audit/│ │  Testing   │ │  Checks     │  │
│  │  CodeQL)   │ │  Trivy)    │ │ (Stryker/  │ │ (dep-       │  │
│  │            │ │            │ │  mutmut)   │ │  cruiser)   │  │
│  └────────────┘ └────────────┘ └────────────┘ └──────────────┘  │
│                                                                   │
│  + AI Code Review (Hawk QA agent or CodeRabbit)                  │
│  Mutation score on changed files: ≥75%                           │
└────────────────────────────────┬────────────────────────────────┘
                                 ▼
┌─────────────────────────────────────────────────────────────────┐
│  STAGE 4: BUILD & PREVIEW DEPLOY  (~2-5min)                      │
│                                                                   │
│  ┌────────────────────┐  ┌─────────────────────────────────┐    │
│  │ Build artifact     │  │ Preview deployment              │    │
│  │ (Docker image /    │  │ (Vercel preview / Cloudflare    │    │
│  │  static bundle)    │  │  Pages branch / Railway env)    │    │
│  └────────────────────┘  └─────────────────────────────────┘    │
│                                                                   │
│  Preview URL generated → fed to E2E stage                        │
└────────────────────────────────┬────────────────────────────────┘
                                 ▼
┌─────────────────────────────────────────────────────────────────┐
│  STAGE 5: E2E & VISUAL REGRESSION  (~5-15min)                    │
│                                                                   │
│  ┌────────────┐ ┌────────────┐ ┌────────────┐ ┌──────────────┐  │
│  │ E2E Tests  │ │  Visual    │ │   DAST     │ │   Smoke     │  │
│  │(Playwright)│ │ Regression │ │ (ZAP/      │ │   Tests     │  │
│  │            │ │ (Chromatic/│ │  Nuclei)   │ │ (health +   │  │
│  │            │ │  Playwright│ │            │ │  critical   │  │
│  │            │ │  screenshots│            │ │  paths)     │  │
│  └────────────┘ └────────────┘ └────────────┘ └──────────────┘  │
│                                                                   │
│  Run against preview deployment                                  │
└────────────────────────────────┬────────────────────────────────┘
                                 ▼
┌─────────────────────────────────────────────────────────────────┐
│  STAGE 6: QUALITY GATE & APPROVAL                                │
│                                                                   │
│  All stages pass?                                                │
│  ├─ YES + low-risk  → auto-merge (graduated trust)              │
│  ├─ YES + high-risk → human approval required                   │
│  └─ NO              → reject, errors fed back to agent          │
│                                                                   │
│  Risk classification:                                            │
│    Low:  docs, tests, formatting, config                         │
│    Med:  feature code, UI changes                                │
│    High: auth, payments, DB schemas, infra, CI config            │
└────────────────────────────────┬────────────────────────────────┘
                                 ▼
┌─────────────────────────────────────────────────────────────────┐
│  STAGE 7: PRODUCTION DEPLOY                                      │
│                                                                   │
│  Progressive delivery:                                           │
│    1. Canary (5%) → monitor 10min                                │
│    2. Expand (25%) → monitor 10min                               │
│    3. Full rollout (100%)                                        │
│                                                                   │
│  Auto-rollback triggers:                                         │
│    • Error rate > 2x baseline                                    │
│    • p99 latency > 3x baseline                                   │
│    • Health check failures                                       │
│    • Memory/CPU anomalies                                        │
└────────────────────────────────┬────────────────────────────────┘
                                 ▼
┌─────────────────────────────────────────────────────────────────┐
│  STAGE 8: POST-DEPLOY MONITORING                                 │
│                                                                   │
│  ┌────────────┐ ┌────────────┐ ┌────────────┐ ┌──────────────┐  │
│  │  Health    │ │  Error     │ │   Perf     │ │   Cost      │  │
│  │  Checks   │ │  Tracking  │ │ Monitoring │ │  Tracking   │  │
│  │ (60s ping)│ │  (Sentry)  │ │ (latency/  │ │ (token +    │  │
│  │           │ │            │ │  throughput)│ │  infra $)   │  │
│  └────────────┘ └────────────┘ └────────────┘ └──────────────┘  │
│                                                                   │
│  Anomaly → alert agent → agent investigates → fix PR             │
│  Creates a self-healing feedback loop                            │
└─────────────────────────────────────────────────────────────────┘
```

---

## How Agents Trigger Deployments

Agents don't deploy directly. They operate through git, and the pipeline handles the rest.

### The GitOps Flow

```
Agent receives task (build feature / fix bug / respond to alert)
    ↓
Agent clones repo, creates branch, writes code
    ↓
Agent runs local pre-commit checks (lint, type-check, format)
    ↓
Agent pushes branch, creates PR with:
  • Description of changes
  • Link to originating task/issue
  • Agent identity tag (git trailer: Agent-By: forge/builder-01)
    ↓
GitHub Actions CI triggered automatically
    ↓
Pipeline runs stages 1-6
    ↓
On merge to main → stages 7-8 (deploy + monitor)
```

### Trigger Mechanisms

| Trigger | Mechanism | Example |
|---------|-----------|---------|
| **Agent PR** | `git push` + PR creation via GitHub API | Agent builds feature, opens PR |
| **Manual dispatch** | `workflow_dispatch` via GitHub API | Agent triggers specific workflow |
| **Scheduled** | Cron in GitHub Actions | Nightly full test suite, dependency updates |
| **Webhook** | External event triggers pipeline | Monitoring alert → agent fix → auto-PR |
| **ChatOps** | Message command triggers deploy | "deploy project-x to staging" |

### Agent Identity & Audit Trail

Every agent-generated commit includes:

```
feat: add user authentication endpoint

Implements JWT-based auth with refresh tokens.
Closes #142

Agent-By: forge/builder-01
Task-ID: task-2026-0208-0042
Model: claude-sonnet-4-20250514
Prompt-Hash: sha256:abc123...
```

This enables:
- Filtering agent vs human PRs in CI (apply stricter gates to agent PRs)
- Tracking agent quality metrics over time
- Auditing which model/prompt produced which code
- Graduated trust decisions based on agent track record

---

## Platform Strategy

### Decision Matrix

```
What are you deploying?
│
├── Static site / SPA / SSR frontend
│   └── Cloudflare Pages (primary) or Vercel (Next.js)
│
├── API / backend service
│   └── Railway (simple) or Fly.io (needs VMs/global)
│
├── Serverless function / webhook / edge logic
│   └── Cloudflare Workers
│
├── Complex infra (VPC, managed DB, queues)
│   └── Pulumi Automation API (TypeScript IaC)
│
└── Container orchestration at scale
    └── Managed Kubernetes + ArgoCD (GitOps)
```

### Platform Details

#### Cloudflare (Primary — Edge)
- **Pages**: Static/SSR sites, git-connected or `wrangler pages deploy`
- **Workers**: Serverless functions, `wrangler deploy`
- **D1/KV/R2**: Database, key-value, object storage — all CLI-provisionable
- **Why primary**: Fastest edge network, generous free tier, excellent CLI, agent-friendly API
- **Agent integration**: `wrangler` commands in GitHub Actions, REST API for management

#### Railway (Primary — Backend)
- **Deploy anything**: Auto-detects language, builds OCI image
- **Built-in databases**: Postgres, MySQL, Redis, MongoDB — one command
- **Environments**: Staging, production, ephemeral per-PR
- **Why primary**: Lowest friction for backend services, GraphQL API for automation
- **Agent integration**: `railway up` in CI, or GraphQL API for programmatic control

#### Vercel (Secondary — Frontend)
- **Best for**: Next.js specifically, AI SDK integration
- **Preview deploys**: Automatic per-PR preview URLs
- **Agent integration**: `vercel deploy --prod` in CI

#### Pulumi (Infrastructure)
- **Why over Terraform**: TypeScript (agents already know it), Automation API (no CLI shelling)
- **Use for**: Cloud resources beyond PaaS — VPCs, managed databases, DNS, IAM
- **Pattern**: `pulumi preview` → human review → `pulumi up`

### Multi-Platform Deploy Example (GitHub Actions)

```yaml
# .github/workflows/deploy.yml
name: Deploy Pipeline
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

concurrency:
  group: deploy-${{ github.ref }}
  cancel-in-progress: true

jobs:
  # ─── STAGE 1: Fast Gates ───
  fast-gates:
    runs-on: ubuntu-latest
    timeout-minutes: 5
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: 22 }
      - run: npm ci
      - run: npx prettier --check .
      - run: npx eslint --max-warnings 0 .
      - run: npx tsc --strict --noEmit
      - run: npx gitleaks detect --source .

  # ─── STAGE 2: Tests ───
  tests:
    needs: fast-gates
    runs-on: ubuntu-latest
    timeout-minutes: 15
    services:
      postgres:
        image: postgres:16
        env: { POSTGRES_PASSWORD: test }
        ports: ['5432:5432']
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: 22 }
      - run: npm ci
      - run: npx vitest run --coverage
      - run: npx vitest run --config vitest.integration.config.ts
      - name: Coverage gate
        run: |
          COVERAGE=$(cat coverage/coverage-summary.json | jq '.total.lines.pct')
          if (( $(echo "$COVERAGE < 80" | bc -l) )); then
            echo "Coverage $COVERAGE% below 80% threshold"
            exit 1
          fi

  # ─── STAGE 3: Security & Deep Analysis ───
  security:
    needs: fast-gates
    runs-on: ubuntu-latest
    timeout-minutes: 15
    steps:
      - uses: actions/checkout@v4
      - uses: returntocorp/semgrep-action@v1
        with: { config: 'p/default p/security-audit p/typescript' }
      - run: npm audit --audit-level=high
      - run: npx dependency-cruiser src --validate

  mutation:
    needs: tests
    runs-on: ubuntu-latest
    timeout-minutes: 30
    if: github.event_name == 'pull_request'
    steps:
      - uses: actions/checkout@v4
        with: { fetch-depth: 0 }
      - uses: actions/setup-node@v4
        with: { node-version: 22 }
      - run: npm ci
      - name: Mutation test changed files
        run: |
          CHANGED=$(git diff --name-only origin/main -- 'src/**/*.ts' | grep -v '.test.' || true)
          if [ -n "$CHANGED" ]; then
            npx stryker run --mutate "$CHANGED"
          fi

  # ─── STAGE 4: Build & Preview ───
  build-preview:
    needs: [tests, security]
    runs-on: ubuntu-latest
    timeout-minutes: 10
    outputs:
      preview-url: ${{ steps.deploy.outputs.url }}
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: 22 }
      - run: npm ci && npm run build
      - name: Deploy preview
        id: deploy
        if: github.event_name == 'pull_request'
        run: |
          # Platform-specific preview deploy
          # Cloudflare Pages:
          npx wrangler pages deploy dist --project-name=${{ github.repository }} \
            --branch=${{ github.head_ref }} | tee deploy.log
          URL=$(grep -oP 'https://[^\s]+' deploy.log | head -1)
          echo "url=$URL" >> $GITHUB_OUTPUT

  # ─── STAGE 5: E2E on Preview ───
  e2e:
    needs: build-preview
    if: github.event_name == 'pull_request'
    runs-on: ubuntu-latest
    timeout-minutes: 20
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: 22 }
      - run: npm ci && npx playwright install --with-deps
      - run: npx playwright test
        env:
          BASE_URL: ${{ needs.build-preview.outputs.preview-url }}

  # ─── STAGE 6: Quality Gate ───
  quality-gate:
    needs: [tests, security, mutation, e2e]
    if: always()
    runs-on: ubuntu-latest
    steps:
      - name: Check all gates
        run: |
          if [ "${{ needs.tests.result }}" != "success" ] || \
             [ "${{ needs.security.result }}" != "success" ]; then
            echo "Quality gates failed"
            exit 1
          fi

  # ─── STAGE 7: Production Deploy ───
  deploy-production:
    needs: quality-gate
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    environment: production  # requires approval for high-risk
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: 22 }
      - run: npm ci && npm run build
      - name: Deploy to production
        run: npx wrangler pages deploy dist --project-name=${{ github.repository }} --branch=main
      - name: Health check
        run: |
          for i in {1..30}; do
            STATUS=$(curl -s -o /dev/null -w '%{http_code}' $PROD_URL/health)
            if [ "$STATUS" = "200" ]; then
              echo "Health check passed"
              exit 0
            fi
            sleep 2
          done
          echo "Health check failed — triggering rollback"
          # Rollback: redeploy previous known-good commit
          git checkout HEAD~1
          npm ci && npm run build
          npx wrangler pages deploy dist --project-name=${{ github.repository }} --branch=main
          exit 1
```

---

## GitOps Model

### Repository Structure

```
project/
├── .github/
│   └── workflows/
│       ├── ci.yml              # PR pipeline (stages 1-5)
│       ├── deploy.yml          # Production deploy (stages 6-8)
│       └── rollback.yml        # Manual/auto rollback
├── infrastructure/
│   ├── pulumi/                 # IaC definitions
│   │   ├── index.ts
│   │   └── Pulumi.yaml
│   └── k8s/                   # K8s manifests (if applicable)
│       ├── deployment.yaml
│       └── service.yaml
├── src/                        # Application code
├── tests/
│   ├── unit/
│   ├── integration/
│   ├── e2e/
│   └── properties/             # Property-based tests
├── AGENTS.md                   # Agent instructions (includes test/quality rules)
├── .semgrep.yml                # Custom security rules
└── stryker.config.js           # Mutation testing config
```

### GitOps Principles

1. **Git is the source of truth** — deployment state, infra config, app config all in git
2. **PRs are the unit of change** — no `kubectl apply` from laptops or agent terminals
3. **Declarative, not imperative** — describe desired state, let the platform reconcile
4. **Environment promotion via branches/tags**:
   - `main` → production
   - `staging` → staging environment
   - PR branches → ephemeral preview environments

### ArgoCD for Kubernetes Workloads

For projects that graduate to Kubernetes:

```yaml
# argocd-application.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-app
spec:
  source:
    repoURL: https://github.com/org/project
    path: infrastructure/k8s
    targetRevision: main
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

Agent pushes updated manifests → ArgoCD detects drift → auto-syncs to cluster.

---

## Feedback Loops

The pipeline isn't just a gate — it's a teacher. When things fail, the agent learns.

### Error-to-Agent Feedback

```
CI fails
    ↓
Parse failure output (structured JSON from CI)
    ↓
Classify: lint error | type error | test failure | security issue
    ↓
Route back to originating agent with:
  • Exact error message
  • File + line number
  • Suggested fix category
  • Link to CI run for full context
    ↓
Agent creates fix commit, pushes to same PR
    ↓
CI re-runs (max 3 auto-fix attempts, then escalate to human)
```

### Production-to-Agent Feedback

```
Monitoring detects anomaly post-deploy
    ↓
Auto-rollback triggers
    ↓
Alert agent with:
  • Error logs / stack traces
  • Metrics diff (before/after deploy)
  • The specific commit that caused the issue
    ↓
Agent creates hotfix PR
    ↓
Fast-track pipeline (still full gates, but prioritized)
```

---

## Health Monitoring & Observability

### Deploy-Time Health Checks

Every deployment includes:

```typescript
// Deployed as a Cloudflare Worker alongside the app
export default {
  async scheduled(event, env) {
    const endpoints = [
      { url: `${env.APP_URL}/health`, name: 'health' },
      { url: `${env.APP_URL}/api/status`, name: 'api' },
    ];

    for (const ep of endpoints) {
      const start = Date.now();
      const res = await fetch(ep.url);
      const latency = Date.now() - start;

      if (res.status !== 200 || latency > 5000) {
        await fetch(env.ALERT_WEBHOOK, {
          method: 'POST',
          body: JSON.stringify({
            alert: `${ep.name} degraded`,
            status: res.status,
            latency,
            timestamp: new Date().toISOString(),
          }),
        });
      }
    }
  },
};
```

### Key Metrics to Track

| Metric | Source | Alert Threshold |
|--------|--------|----------------|
| Error rate | Sentry / app logs | >2x baseline post-deploy |
| p50/p95/p99 latency | App metrics | >3x baseline |
| Health endpoint | Cloudflare Worker cron | Any non-200 |
| Memory / CPU | Platform metrics | >80% sustained |
| Deploy frequency | GitHub API | Track only (no alert) |
| MTTR (mean time to recovery) | Deploy + rollback logs | Track only |
| Agent fix success rate | CI re-run data | <50% → reduce agent trust |

---

## Graduated Trust Model

Agents earn deployment autonomy over time.

| Level | Auto-Merge Scope | Human Review Required | Criteria to Advance |
|-------|-------------------|----------------------|---------------------|
| **L0** (New) | Nothing | All PRs | N/A — starting point |
| **L1** (Probation) | Docs, tests, formatting | Feature code, config | 20 successful PRs, 0 incidents |
| **L2** (Trusted) | Low + medium risk code | Auth, payments, infra, DB schemas | 100 PRs, <2% rollback rate |
| **L3** (Veteran) | Most code | Critical security paths only | 500 PRs, <1% rollback rate, 3+ months |

Trust level is per-agent, per-repository. A new agent in a new repo starts at L0 regardless of history elsewhere.

---

## Cost Controls

### Pipeline Cost Optimization

- **Parallel stages**: Stages 2 and 3 run concurrently (tests + security in parallel)
- **Conditional expensive jobs**: Mutation testing only on PRs, not on every push
- **Caching**: Node modules, Docker layers, Playwright browsers
- **Skip on docs-only**: `paths-ignore: ['**.md', 'docs/**']` for full pipeline
- **Self-hosted runners**: For heavy workloads (mutation testing), use dedicated compute

### Deployment Cost

- **Cloudflare Workers/Pages**: Free tier covers most projects (100K requests/day, 500 deploys/month)
- **Railway**: Usage-based, ~$5-20/month for typical services
- **Preview environments**: Auto-delete after PR merge (avoid zombie deployments)
- **Cost alerts**: Budget limits on all platforms, notify agent + human if exceeded

---

## Summary

The pipeline transforms autonomous code generation into reliable production deployments:

1. **Agents commit code** — tagged with identity and task metadata
2. **Fast gates reject garbage** — lint, type-check, secrets scan in 30 seconds
3. **Tests verify correctness** — unit, integration, property-based, mutation
4. **Security scans catch vulnerabilities** — SAST, SCA, dependency audit
5. **Preview deploys enable E2E** — test against real infrastructure
6. **Quality gates enforce standards** — auto-merge or human review based on risk
7. **Progressive delivery limits blast radius** — canary → expand → full rollout
8. **Monitoring closes the loop** — anomalies trigger rollback and agent investigation

The factory never sleeps. Neither does the pipeline.

---

*Architecture document 06. Part of the Forge Masterplan.*
