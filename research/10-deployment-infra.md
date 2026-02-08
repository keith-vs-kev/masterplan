# Deployment & Infrastructure Automation

> How AI agents deploy their own code, provision infrastructure, and monitor health autonomously.

## Executive Summary

Deployment automation is the capability that transforms an AI agent from a code generator into a full-stack autonomous developer. The modern platform landscape offers a spectrum from zero-config PaaS (Vercel, Railway) to full IaC (Terraform, Pulumi), each with different trade-offs for agent-driven automation. The key insight: **agents should start with high-level PaaS APIs for speed, and graduate to IaC for complex infrastructure** — mirroring how human developers work.

---

## 1. Platform Landscape

### 1.1 Serverless / Edge Platforms

#### Cloudflare Workers & Pages
- **What**: Serverless JS/TS/Python/Rust execution on Cloudflare's global edge network
- **Agent-friendliness**: ★★★★★
  - `wrangler` CLI is fully scriptable — `wrangler deploy` is a single command
  - Pages: git-connected or direct upload (`wrangler pages deploy ./dist`)
  - Workers: deploy functions with `wrangler deploy`
  - Bindings ecosystem (KV, D1, R2, Queues, Durable Objects) — all provisionable via CLI/API
  - REST API for everything: CRUD deployments, manage routes, DNS, secrets
- **Best for**: API endpoints, edge functions, static sites, lightweight full-stack apps
- **Limits**: 10ms CPU (free), 30s (paid) per request; 500 deploys/month free (Pages)
- **Agent strategy**: Use `wrangler` CLI directly. Workers are ideal for deploying agent-built APIs and webhooks. Durable Objects enable stateful agent services.

#### Vercel
- **What**: Frontend-first platform with serverless functions, now positioning as "AI Cloud"
- **Agent-friendliness**: ★★★★
  - `vercel` CLI: `vercel deploy --prod` — zero-config for Next.js, React, etc.
  - REST API for deployments, environment variables, domains
  - Git integration: push to deploy
  - AI SDK integration for agent workloads
  - Preview deployments per branch/PR
- **Best for**: Next.js apps, frontend-heavy projects, AI-powered web apps
- **Limits**: Serverless function timeout (10s free, 300s pro); bandwidth limits on free tier
- **Agent strategy**: Best when the agent is building web UIs. Use CLI for programmatic deploys. The preview URL pattern is excellent for agent self-testing.

### 1.2 Application Platforms (PaaS)

#### Railway
- **What**: All-in-one cloud platform — deploy anything from code repos or Docker images
- **Agent-friendliness**: ★★★★★
  - Detects language/framework automatically, builds OCI images
  - CLI: `railway up` deploys from current directory
  - REST API and GraphQL API for full programmatic control
  - Built-in Postgres, MySQL, Redis, MongoDB with one click
  - Environment management (staging, production, ephemeral)
  - Variables/secrets management via CLI
  - Observability built in (logs, metrics)
- **Best for**: Backend services, databases, full-stack apps, anything Docker-compatible
- **Pricing**: Usage-based ($5/month + compute/memory)
- **Agent strategy**: Excellent for agents deploying backend services. The "deploy anything" model means the agent doesn't need to think about infrastructure details. GraphQL API enables sophisticated automation.

#### Fly.io
- **What**: Deploy apps as micro-VMs (Firecracker) on a global network
- **Agent-friendliness**: ★★★★
  - `flyctl launch` + `flyctl deploy` — Dockerfile-based
  - Machines API: programmatically start/stop VMs via REST
  - Global distribution with specific region placement
  - Volumes for persistent storage
  - Built-in Postgres, Redis (Upstash)
  - Private networking between apps
- **Best for**: Stateful services, globally distributed apps, anything needing real VMs
- **Pricing**: Pay per VM-second; generous free tier
- **Agent strategy**: The Machines API is uniquely powerful — agents can spin up/down compute on demand. Great for deploying agent infrastructure that needs persistent processes (long-running workers, WebSocket servers).

### 1.3 Container Orchestration

#### Docker
- **What**: Container runtime — the universal packaging format
- **Agent-friendliness**: ★★★★
  - `docker build` + `docker push` — well-understood, scriptable
  - Dockerfile generation is a solved problem for LLMs
  - Docker Compose for multi-service local dev and simple deployments
  - Universal: every platform accepts Docker images
- **Agent strategy**: Agents should generate Dockerfiles as the universal deployment artifact. Even when targeting PaaS platforms, having a Dockerfile provides portability.

#### Kubernetes
- **What**: Container orchestration at scale
- **Agent-friendliness**: ★★★ (complex but powerful)
  - `kubectl` CLI is fully scriptable
  - Declarative YAML manifests — agents can generate these
  - Helm charts for templated deployments
  - Massive ecosystem (Istio, ArgoCD, Cert-Manager, etc.)
  - Self-healing, auto-scaling, rolling updates
- **Complexity**: High. Networking, RBAC, storage classes, ingress controllers...
- **Agent strategy**: Use for production-grade deployments where scale matters. Agents should use Helm or Kustomize rather than raw YAML. Consider managed K8s (GKE, EKS, AKS) to reduce ops burden. ArgoCD for GitOps is very agent-friendly (push manifests → auto-deploy).

### 1.4 Infrastructure as Code (IaC)

#### Terraform
- **What**: Declarative IaC using HCL — the industry standard
- **Agent-friendliness**: ★★★★
  - `terraform plan` + `terraform apply` — predictable workflow
  - `plan` output is readable and auditable (great for human-in-the-loop)
  - Providers for everything: AWS, GCP, Azure, Cloudflare, Vercel, GitHub, etc.
  - State management (local or remote via Terraform Cloud, S3, etc.)
  - Import existing infrastructure
- **Limitations**: HCL is its own DSL (not a general-purpose language); state drift can be tricky
- **Agent strategy**: Ideal for provisioning cloud resources (databases, VPCs, DNS, etc.). The `plan` → human-review → `apply` workflow is a natural approval gate. Agents should always run `plan` first and present the diff.

#### Pulumi
- **What**: IaC using real programming languages (TypeScript, Python, Go, C#, Java)
- **Agent-friendliness**: ★★★★★
  - Uses languages LLMs already know — no DSL to learn
  - Full programming constructs (loops, conditionals, functions, classes)
  - `pulumi preview` + `pulumi up` — same plan/apply pattern as Terraform
  - Automation API: run Pulumi programmatically from within code (no CLI needed)
  - Pulumi Cloud for state, secrets, and team management
  - Supports all major clouds via native providers
- **Advantages over Terraform for agents**: Real code means agents can use abstractions, compose components, and write tests. The Automation API is purpose-built for programmatic/agent use.
- **Agent strategy**: **Top pick for agent-driven IaC.** The Automation API means an agent can provision infrastructure directly from TypeScript without shelling out to a CLI. Write Pulumi programs in TypeScript alongside the application code.

### 1.5 CI/CD

#### GitHub Actions
- **What**: CI/CD built into GitHub — workflow automation via YAML
- **Agent-friendliness**: ★★★★
  - Workflows are YAML files in `.github/workflows/` — agents can generate and commit them
  - Trigger on push, PR, schedule, manual dispatch, webhook
  - Marketplace of pre-built actions
  - Matrix builds, caching, artifacts, environments with approval gates
  - Self-hosted runners for custom compute
  - REST API to trigger workflows, check status, download artifacts
- **Agent strategy**: The agent's primary CI/CD tool. Generate workflow files that handle build → test → deploy. Use `workflow_dispatch` for agent-triggered deployments. Use the API to monitor run status.

---

## 2. Architecture: The Forge Deployment Agent

### 2.1 Design Principles

1. **Declarative over imperative**: Prefer describing desired state (IaC, K8s manifests) over step-by-step scripts
2. **Preview before apply**: Always show what will change before changing it
3. **Rollback-first**: Every deployment must be reversible
4. **Progressive complexity**: Start with PaaS, graduate to IaC when needed
5. **Observe everything**: Deploy monitoring alongside the application

### 2.2 Deployment Decision Tree

```
Is it a static site or simple frontend?
  → Cloudflare Pages or Vercel

Is it a serverless API / webhook / edge function?
  → Cloudflare Workers

Is it a backend service (API, worker, bot)?
  → Railway (simple) or Fly.io (needs VMs/global)

Does it need complex infrastructure (VPCs, managed DBs, queues)?
  → Pulumi (preferred) or Terraform

Does it need container orchestration at scale?
  → Kubernetes (managed: GKE/EKS/AKS)

Does it just need CI/CD?
  → GitHub Actions
```

### 2.3 Deployment Pipeline Architecture

```
┌─────────────────────────────────────────────────┐
│                  Forge Agent                      │
│                                                   │
│  ┌───────────┐  ┌────────────┐  ┌─────────────┐ │
│  │ Code Gen   │→│ Build/Test  │→│ Deploy       │ │
│  │            │  │ (GH Actions)│  │ (Platform)   │ │
│  └───────────┘  └────────────┘  └─────────────┘ │
│       │              │               │            │
│       ▼              ▼               ▼            │
│  ┌───────────┐  ┌────────────┐  ┌─────────────┐ │
│  │ Dockerfile │  │ Test Report│  │ Health Check │ │
│  │ IaC Code   │  │ Artifacts  │  │ Monitoring   │ │
│  └───────────┘  └────────────┘  └─────────────┘ │
└─────────────────────────────────────────────────┘
```

### 2.4 Agent Capabilities Matrix

| Capability | Tools | Complexity |
|---|---|---|
| Deploy static site | Cloudflare Pages, Vercel CLI | Low |
| Deploy API/backend | Railway CLI, Fly.io CLI | Low |
| Deploy serverless function | Cloudflare Wrangler, Vercel CLI | Low |
| Provision database | Railway (built-in), Pulumi | Medium |
| Set up CI/CD pipeline | GitHub Actions YAML | Medium |
| Provision cloud infra | Pulumi Automation API, Terraform | High |
| Deploy to Kubernetes | kubectl, Helm, ArgoCD | High |
| Full environment (app + DB + CDN + DNS) | Pulumi + platform CLIs | High |

---

## 3. Implementation Patterns

### 3.1 The Simple Deploy (PaaS)

Agent deploys a Node.js API to Railway:

```bash
# Agent generates code, then:
cd /path/to/project
railway login --browserless  # use token
railway init
railway up                    # auto-detect, build, deploy
railway domain               # get public URL

# Verify
curl https://myapp.up.railway.app/health
```

**Total commands: 4.** This is the baseline agent experience.

### 3.2 The Full-Stack Deploy (Pulumi Automation API)

Agent provisions infrastructure programmatically:

```typescript
import { LocalWorkspace } from "@pulumi/pulumi/automation";

async function deploy() {
  const stack = await LocalWorkspace.createOrSelectStack({
    stackName: "production",
    projectName: "my-app",
    program: async () => {
      // Define infrastructure in code
      const db = new aws.rds.Instance("db", {
        engine: "postgres",
        instanceClass: "db.t3.micro",
        allocatedStorage: 20,
      });
      
      const app = new awsx.ecs.FargateService("app", {
        desiredCount: 2,
        taskDefinitionArgs: {
          container: {
            image: "myapp:latest",
            environment: [
              { name: "DATABASE_URL", value: db.endpoint },
            ],
          },
        },
      });
      
      return { url: app.loadBalancer.hostname, dbEndpoint: db.endpoint };
    },
  });

  // Preview changes (agent shows to human)
  const preview = await stack.preview();
  console.log(preview.changeSummary);
  
  // Apply after approval
  const result = await stack.up();
  console.log(`App URL: ${result.outputs.url.value}`);
}
```

**Key advantage**: No CLI shelling, no YAML — pure TypeScript the agent can generate and reason about.

### 3.3 GitOps Pattern (GitHub Actions + ArgoCD)

Agent commits code and workflow, platform handles deployment:

```yaml
# .github/workflows/deploy.yml (agent-generated)
name: Deploy
on:
  push:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: docker build -t ghcr.io/org/app:${{ github.sha }} .
      - run: docker push ghcr.io/org/app:${{ github.sha }}
      
  deploy:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - name: Update Kubernetes manifest
        run: |
          sed -i "s|image:.*|image: ghcr.io/org/app:${{ github.sha }}|" k8s/deployment.yaml
      - name: Commit and push
        run: |
          git add k8s/
          git commit -m "Deploy ${{ github.sha }}"
          git push
      # ArgoCD watches the repo and auto-syncs
```

### 3.4 Health Monitoring Pattern

Agent deploys monitoring alongside the application:

```typescript
// Agent adds health checking to every deployment
async function deployWithMonitoring(appUrl: string) {
  // 1. Deploy the app (any method above)
  
  // 2. Wait for healthy
  const healthy = await pollHealth(appUrl + "/health", {
    maxAttempts: 30,
    intervalMs: 2000,
    expectedStatus: 200,
  });
  
  if (!healthy) {
    // 3. Auto-rollback
    await rollback();
    throw new Error("Deployment failed health check — rolled back");
  }
  
  // 4. Set up ongoing monitoring
  // Deploy a Cloudflare Worker that pings every 60s
  // Alert via webhook if health check fails
}
```

---

## 4. Security Considerations

### 4.1 Credential Management
- **Never hardcode secrets** — use platform secret stores (Railway variables, Vercel env, GitHub Secrets)
- **Scoped tokens**: Create deployment-specific API tokens with minimal permissions
- **Short-lived credentials**: Use OIDC federation (GitHub Actions → AWS) instead of long-lived keys
- **Secret rotation**: Agent should be able to rotate credentials and update references

### 4.2 Approval Gates
- **Terraform/Pulumi plan review**: Always present diffs before applying
- **GitHub Environments**: Use required reviewers for production deployments
- **Cost estimation**: Show estimated cost impact before provisioning infrastructure
- **Blast radius**: Agent should assess and communicate what could break

### 4.3 Least Privilege
- Deploy tokens should only access the specific project/environment
- Kubernetes RBAC: agent service account limited to its namespace
- Cloud IAM: agent role limited to specific services (not admin)

---

## 5. Recommended Stack for Forge

### Tier 1: Quick Deploy (90% of cases)
- **Frontend**: Cloudflare Pages (zero-config, free, fast)
- **Backend API**: Railway (auto-detect, built-in databases)
- **Serverless**: Cloudflare Workers (edge, cheap, fast)
- **CI/CD**: GitHub Actions

### Tier 2: Production Grade
- **Infrastructure**: Pulumi (TypeScript, Automation API)
- **Containers**: Fly.io (Machines API for dynamic compute)
- **Orchestration**: Managed Kubernetes (when scale demands it)
- **Monitoring**: Cloudflare Worker health pings + webhook alerts

### Tier 3: Enterprise
- **Multi-cloud IaC**: Pulumi with multiple providers
- **GitOps**: ArgoCD + GitHub Actions
- **Service mesh**: Istio on Kubernetes
- **Observability**: Grafana Cloud / Datadog (Pulumi-provisioned)

---

## 6. Key Insights for Agent Design

1. **Pulumi Automation API is the killer feature**: It lets agents provision infrastructure without leaving TypeScript. No CLI parsing, no YAML generation — just code. This is the strongest argument for Pulumi over Terraform for agent-driven IaC.

2. **PaaS platforms are underrated**: Railway and Fly.io handle 90% of deployment needs with single commands. Agents shouldn't reach for Kubernetes until they actually need it.

3. **The preview/plan pattern is essential**: Every IaC tool offers "show me what will change before doing it." Agents must always use this and present it for approval on destructive/costly operations.

4. **Docker is the universal escape hatch**: If an agent can generate a Dockerfile, it can deploy anywhere. This should be the fallback packaging strategy.

5. **Health checks close the loop**: Deployment without verification is incomplete. Every deploy should include automated health checking and rollback capability.

6. **GitOps is naturally agent-friendly**: Agents already work with git. Making deployment a git operation (commit manifests → auto-deploy) fits the agent's existing workflow perfectly.

7. **Cost awareness matters**: Agents provisioning infrastructure should estimate and communicate costs. A `pulumi preview` that shows "creating 3 new RDS instances" needs a cost annotation.

---

## 7. Platform Comparison Summary

| Platform | Type | CLI | API | Complexity | Cost | Agent Pick |
|---|---|---|---|---|---|---|
| Cloudflare Workers | Serverless/Edge | wrangler | REST | Low | Free tier generous | ★★★★★ |
| Cloudflare Pages | Static/SSR | wrangler | REST | Low | Free tier generous | ★★★★★ |
| Vercel | Frontend PaaS | vercel | REST | Low | Free tier, then $20/mo | ★★★★ |
| Railway | Full PaaS | railway | GraphQL | Low | Usage-based (~$5+) | ★★★★★ |
| Fly.io | App Platform | flyctl | REST (Machines) | Medium | Usage-based | ★★★★ |
| Docker | Containers | docker | Engine API | Low-Med | Free (runtime) | ★★★★ |
| Kubernetes | Orchestration | kubectl | Full API | High | Varies (managed ~$70+/mo) | ★★★ |
| GitHub Actions | CI/CD | gh | REST | Medium | Free 2000 min/mo | ★★★★ |
| Terraform | IaC | terraform | Cloud API | Medium-High | Free (OSS) | ★★★★ |
| Pulumi | IaC | pulumi | Automation API | Medium | Free (OSS) | ★★★★★ |

---

*Research compiled 2026-02-08. Focus: enabling Forge agents to autonomously deploy and manage infrastructure with appropriate human oversight.*
