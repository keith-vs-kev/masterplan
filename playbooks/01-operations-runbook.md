# Operations Runbook — Agentic Factory

> Day-1 operations guide for Adam. How to start the factory, keep it running, fix things when they break, and interact with it daily.
>
> **Last updated:** 2026-02-08

---

## Table of Contents

1. [Quick Reference Card](#1-quick-reference-card)
2. [Starting the Factory](#2-starting-the-factory)
3. [Daily Operations](#3-daily-operations)
4. [Health Checks](#4-health-checks)
5. [Monitoring & Dashboards](#5-monitoring--dashboards)
6. [Common Failure Modes & Fixes](#6-common-failure-modes--fixes)
7. [Backup & Recovery](#7-backup--recovery)
8. [Cost Management](#8-cost-management)
9. [Adam's Daily Interaction Guide](#9-adams-daily-interaction-guide)
10. [Emergency Procedures](#10-emergency-procedures)
11. [Maintenance Windows](#11-maintenance-windows)
12. [Runbook Changelog](#12-runbook-changelog)

---

## 1. Quick Reference Card

### Key Paths

| What | Path |
|------|------|
| Agent workspaces | `/home/adam/.openclaw/workspace-*` |
| Shared state / task queue | `/home/adam/agents/shared/queue/` |
| Architecture docs | `/home/adam/projects/masterplan/architecture/` |
| Agent memory (daily) | `/home/adam/.openclaw/workspace-*/memory/` |
| Project repos | `/home/adam/projects/` |
| Orchestrator DB | `/home/adam/agents/shared/orchestrator.db` |
| LiteLLM config | `/etc/litellm/config.yaml` (or local equivalent) |
| Local models | `/models/` or Ollama default |

### Key Services

| Service | Port | Check |
|---------|------|-------|
| OpenClaw Gateway | 3000 | `openclaw gateway status` |
| LiteLLM Proxy | 4000 | `curl -s http://localhost:4000/health` |
| llama.cpp (inference) | 8080 | `curl -s http://localhost:8080/health` |
| llama.cpp (embeddings) | 8081 | `curl -s http://localhost:8081/health` |
| Ollama | 11434 | `curl -s http://localhost:11434/api/tags` |
| Qdrant | 6333 | `curl -s http://localhost:6333/collections` |
| Neo4j | 7474 | `curl -s http://localhost:7474` |
| TimescaleDB | 5432 | `pg_isready -h localhost -p 5432` |
| Grafana | 3001 | `curl -s http://localhost:3001/api/health` |
| Redis | 6379 | `redis-cli ping` |
| NATS | 4222 | `nats server check connection` |

### Key Commands

```bash
# Factory status at a glance
openclaw gateway status
systemctl status llama-inference llama-embed litellm
nvidia-smi                          # GPU status
docker ps                           # All containerised services

# Quick health check (run all at once)
curl -sf http://localhost:4000/health && echo "LiteLLM ✅" || echo "LiteLLM ❌"
curl -sf http://localhost:8080/health && echo "Inference ✅" || echo "Inference ❌"
redis-cli ping | grep -q PONG && echo "Redis ✅" || echo "Redis ❌"
```

---

## 2. Starting the Factory

### 2.1 Cold Start (after reboot or first-time setup)

Run in this order — later services depend on earlier ones.

```bash
# ── Step 1: Infrastructure Services ──────────────────────
# Start databases and message broker
docker compose -f /home/adam/factory/docker-compose.yml up -d \
  redis nats timescaledb qdrant neo4j grafana

# Wait for DBs to be ready
until pg_isready -h localhost -p 5432 2>/dev/null; do sleep 1; done
until redis-cli ping 2>/dev/null | grep -q PONG; do sleep 1; done
echo "Infrastructure ready ✅"

# ── Step 2: Local Inference ──────────────────────────────
sudo systemctl start llama-embed     # CPU embeddings (fast, always-on)
sudo systemctl start llama-inference # GPU inference (loads 8B model by default)

# Verify GPU is being used
nvidia-smi | grep -q llama && echo "GPU inference ✅" || echo "GPU inference ❌"

# ── Step 3: LiteLLM Proxy ───────────────────────────────
sudo systemctl start litellm
sleep 3
curl -sf http://localhost:4000/health && echo "LiteLLM ✅"

# ── Step 4: OpenClaw Gateway ────────────────────────────
openclaw gateway start
sleep 5
openclaw gateway status

# ── Step 5: Verify Agent Sessions ────────────────────────
# Kev (orchestrator) and Forge should come online via heartbeat
# Check in WhatsApp or:
openclaw gateway status | grep -i "agent"
```

### 2.2 Warm Start (services already running, just gateway restart)

```bash
openclaw gateway restart
```

### 2.3 Start Verification Checklist

After starting, confirm each layer:

- [ ] `nvidia-smi` shows GPU with llama process
- [ ] `curl localhost:4000/health` returns OK
- [ ] `curl localhost:8080/health` returns OK  
- [ ] `openclaw gateway status` shows running
- [ ] Send "status" to Kev via WhatsApp — should respond within 60s
- [ ] Redis: `redis-cli info keyspace` shows databases
- [ ] Docker: `docker ps` shows all expected containers

---

## 3. Daily Operations

### 3.1 Morning Routine (automated via Kev's heartbeat)

Kev performs these checks automatically. Adam receives a morning digest via WhatsApp by ~8am NZDT:

1. **Overnight task summary** — what completed, what failed, what's stuck
2. **Cost report** — yesterday's LLM spend + rolling 7-day average
3. **Dead letter queue** — any tasks that need human attention
4. **Agent health** — all agents responding to heartbeats
5. **Infrastructure** — disk space, GPU status, service health

### 3.2 What Adam Should Do Daily

| Time | Action | How |
|------|--------|-----|
| **Morning** | Read Kev's digest | WhatsApp — react or respond |
| **Morning** | Review dead-letter queue | Reply to Kev's summary or check dashboard |
| **Anytime** | Approve/reject Tier 2 requests | Reply ✅ or ❌ to WhatsApp approval requests |
| **Anytime** | Give new work to Kev | WhatsApp: "Build X" / "Research Y" |
| **Evening** | Glance at cost dashboard | Grafana or ask Kev "what did we spend today?" |

### 3.3 Task Queue Operations

```bash
# View current queue state
ls /home/adam/agents/shared/queue/
# Subdirs: backlog/ ready/ in-progress/ review/ done/ failed/

# Count tasks by status
for dir in backlog ready in-progress review done failed; do
  echo "$dir: $(ls /home/adam/agents/shared/queue/$dir/ 2>/dev/null | wc -l)"
done

# View a specific task
cat /home/adam/agents/shared/queue/in-progress/<task-id>.json | jq .

# Move a stuck task back to ready
mv /home/adam/agents/shared/queue/in-progress/<task-id>.json \
   /home/adam/agents/shared/queue/ready/

# Kill a bad task
mv /home/adam/agents/shared/queue/in-progress/<task-id>.json \
   /home/adam/agents/shared/queue/failed/
```

### 3.4 Agent Management

```bash
# Check which agents are active
openclaw gateway status

# Restart a specific agent (if stuck)
# Via WhatsApp to Kev: "restart forge" or manually:
openclaw gateway restart

# Check agent memory/logs
ls /home/adam/.openclaw/workspace-forge/memory/
cat /home/adam/.openclaw/workspace-forge/memory/$(date +%Y-%m-%d).md
```

---

## 4. Health Checks

### 4.1 Automated Health Check Script

Save as `/home/adam/factory/scripts/health-check.sh`:

```bash
#!/bin/bash
# Factory health check — run manually or via cron

RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
NC='\033[0m'

FAIL=0

check() {
  local name="$1"
  local cmd="$2"
  if eval "$cmd" > /dev/null 2>&1; then
    echo -e "${GREEN}✅ $name${NC}"
  else
    echo -e "${RED}❌ $name${NC}"
    FAIL=$((FAIL + 1))
  fi
}

echo "=== Factory Health Check $(date) ==="
echo ""

echo "── Infrastructure ──"
check "Redis"           "redis-cli ping | grep -q PONG"
check "TimescaleDB"     "pg_isready -h localhost -p 5432"
check "NATS"            "curl -sf http://localhost:8222/healthz"
check "Qdrant"          "curl -sf http://localhost:6333/collections"
check "Grafana"         "curl -sf http://localhost:3001/api/health"

echo ""
echo "── Compute ──"
check "GPU available"   "nvidia-smi > /dev/null"
check "llama inference" "curl -sf http://localhost:8080/health"
check "llama embed"     "curl -sf http://localhost:8081/health"
check "LiteLLM proxy"   "curl -sf http://localhost:4000/health"
check "Ollama"          "curl -sf http://localhost:11434/api/tags"

echo ""
echo "── Agents ──"
check "OpenClaw Gateway" "openclaw gateway status 2>&1 | grep -qi running"

echo ""
echo "── Disk ──"
DISK_PCT=$(df /home/adam --output=pcent | tail -1 | tr -d ' %')
if [ "$DISK_PCT" -lt 85 ]; then
  echo -e "${GREEN}✅ Disk: ${DISK_PCT}% used${NC}"
else
  echo -e "${RED}❌ Disk: ${DISK_PCT}% used (>85% warning)${NC}"
  FAIL=$((FAIL + 1))
fi

echo ""
echo "── GPU Memory ──"
GPU_MEM=$(nvidia-smi --query-gpu=memory.used --format=csv,noheader,nounits 2>/dev/null || echo "N/A")
GPU_TOT=$(nvidia-smi --query-gpu=memory.total --format=csv,noheader,nounits 2>/dev/null || echo "N/A")
echo "GPU VRAM: ${GPU_MEM}MB / ${GPU_TOT}MB"

echo ""
if [ $FAIL -eq 0 ]; then
  echo -e "${GREEN}All checks passed ✅${NC}"
else
  echo -e "${RED}${FAIL} check(s) failed ❌${NC}"
fi

exit $FAIL
```

```bash
chmod +x /home/adam/factory/scripts/health-check.sh
```

### 4.2 Cron-Based Health Monitoring

Add to crontab (`crontab -e`):

```cron
# Health check every 15 minutes, alert on failure
*/15 * * * * /home/adam/factory/scripts/health-check.sh >> /home/adam/factory/logs/health.log 2>&1 || /home/adam/factory/scripts/alert.sh "Health check failed"

# Disk space check daily
0 8 * * * df -h /home/adam | mail -s "Disk report" adam@local 2>/dev/null
```

### 4.3 Manual Deep Health Check

When something feels off:

```bash
# 1. System resources
htop                              # CPU/RAM overview
nvidia-smi                        # GPU utilisation
df -h                             # Disk space
iostat -x 1 3                     # Disk I/O

# 2. Service logs
journalctl -u llama-inference --since "1 hour ago" --no-pager | tail -50
journalctl -u litellm --since "1 hour ago" --no-pager | tail -50
docker logs --tail 100 redis
docker logs --tail 100 timescaledb

# 3. Network (outbound to LLM APIs)
curl -sf https://api.anthropic.com/ && echo "Anthropic reachable ✅"
curl -sf https://api.openai.com/ && echo "OpenAI reachable ✅"

# 4. Queue health
echo "Dead-lettered tasks:"
ls /home/adam/agents/shared/queue/failed/ 2>/dev/null | wc -l

echo "Stuck in-progress (>2h old):"
find /home/adam/agents/shared/queue/in-progress/ -mmin +120 -type f 2>/dev/null
```

---

## 5. Monitoring & Dashboards

### 5.1 Key Metrics to Watch

| Metric | Where | Alert If |
|--------|-------|----------|
| Daily LLM spend | Grafana / LiteLLM dashboard | > $50/day |
| GPU VRAM usage | `nvidia-smi` | Sustained > 23GB (of 24GB) |
| Task queue depth (ready) | Queue directory | > 50 tasks waiting |
| Failed tasks (24h) | Queue failed/ directory | > 10 |
| Dead letter queue | Queue failed/ | Any items |
| Disk usage | `df -h` | > 85% |
| Agent heartbeat misses | OpenClaw logs | Any agent offline > 5min |
| LiteLLM error rate | LiteLLM /metrics endpoint | > 5% |
| Model swap frequency | llama.cpp logs | > 20/hour (thrashing) |

### 5.2 Grafana Dashboards

Access Grafana at `http://localhost:3001` (or configured port).

**Pre-built dashboards:**
- **Factory Overview** — revenue, cost, task throughput, agent status
- **LLM Costs** — per-model, per-agent, daily/weekly/monthly trends
- **Agent Performance** — task completion rates, avg duration, error rates
- **Infrastructure** — GPU, disk, memory, network

### 5.3 LiteLLM Dashboard

LiteLLM provides a built-in UI at `http://localhost:4000/ui`:
- Model usage breakdown
- Cost per model/key
- Request latency
- Error rates by provider

### 5.4 Quick Status via WhatsApp

Ask Kev anytime:
- "status" → factory overview
- "what did we spend today?" → cost report
- "any failures?" → dead letter + error summary
- "how's the queue?" → task counts by status

---

## 6. Common Failure Modes & Fixes

### 6.1 GPU Out of Memory (OOM)

**Symptoms:** llama.cpp crashes, `nvidia-smi` shows 24GB/24GB, CUDA OOM in logs.

**Fix:**
```bash
# Check what's using the GPU
nvidia-smi

# Kill competing GPU processes
# (identify PID from nvidia-smi, kill if it's not llama)
kill <PID>

# If model swap got stuck, restart inference
sudo systemctl restart llama-inference

# If chronic: switch to smaller model
# Edit /etc/systemd/system/llama-inference.service
# Change model path to the 8B variant
sudo systemctl daemon-reload
sudo systemctl restart llama-inference
```

### 6.2 LLM API Rate Limits / Outage

**Symptoms:** Tasks failing with 429/503 errors, LiteLLM logs show provider errors.

**Fix:**
- LiteLLM handles this automatically via fallback chains
- If all providers are down:
  ```bash
  # Check provider status pages
  # Anthropic: status.anthropic.com
  # OpenAI: status.openai.com
  
  # Force route to local models temporarily
  # Tasks will be slower but won't fail
  ```
- For persistent rate limits: check API key quotas, upgrade tier, or reduce concurrency

### 6.3 Agent Stuck / Not Responding

**Symptoms:** Agent doesn't respond to WhatsApp messages, tasks stuck in `in-progress`.

**Fix:**
```bash
# Check if gateway is running
openclaw gateway status

# If gateway is down
openclaw gateway restart

# If gateway is up but agent is stuck
# Check agent's workspace for errors
cat /home/adam/.openclaw/workspace-<agent>/memory/$(date +%Y-%m-%d).md

# Nuclear option: restart gateway (restarts all agent sessions)
openclaw gateway restart
```

### 6.4 Task Stuck in Queue

**Symptoms:** Tasks sitting in `ready/` or `in-progress/` for hours.

**Fix:**
```bash
# Check how long tasks have been waiting
find /home/adam/agents/shared/queue/ready/ -mmin +60 -type f

# For stuck in-progress tasks (agent may have crashed)
find /home/adam/agents/shared/queue/in-progress/ -mmin +120 -type f

# Move stuck tasks back to ready for re-claiming
for f in $(find /home/adam/agents/shared/queue/in-progress/ -mmin +120 -type f); do
  mv "$f" /home/adam/agents/shared/queue/ready/
  echo "Reset: $(basename $f)"
done
```

### 6.5 Disk Space Running Low

**Symptoms:** Builds fail, databases refuse writes, health check fires.

**Fix:**
```bash
# Find biggest offenders
du -sh /home/adam/projects/*/node_modules/ 2>/dev/null | sort -rh | head -10
du -sh /home/adam/.openclaw/workspace-*/memory/ 2>/dev/null | sort -rh | head -10

# Clean docker
docker system prune -f
docker volume prune -f

# Clean old agent memory (keep last 30 days)
find /home/adam/.openclaw/workspace-*/memory/ -name "*.md" -mtime +30 -delete

# Clean old build artifacts
find /tmp/worktrees/ -maxdepth 1 -mtime +7 -exec rm -rf {} +

# Clean npm caches
npm cache clean --force
pnpm store prune
```

### 6.6 Model Swap Thrashing

**Symptoms:** Constant model loading/unloading, slow inference, high latency.

**Fix:**
```bash
# Check swap frequency
journalctl -u llama-inference --since "1 hour ago" | grep -c "loading model"

# If > 10 swaps/hour, the workload is mixed and needs a decision:
# Option A: Lock to 8B model (fast, always available)
# Option B: Lock to 32B model (quality, but slower for simple tasks)
# Option C: Route more to cloud during high-swap periods

# To lock model, edit the model manager config:
# Set IDLE_TIMEOUT to a higher value (e.g., 30 min instead of 5)
```

### 6.7 Database Issues

**TimescaleDB:**
```bash
# Check if running
docker logs timescaledb --tail 20

# Restart
docker restart timescaledb

# Check disk usage
docker exec timescaledb psql -U factory -c "SELECT pg_size_pretty(pg_database_size('factory_analytics'));"
```

**Redis:**
```bash
# Check memory
redis-cli info memory | grep used_memory_human

# If Redis is full, flush non-critical caches
redis-cli FLUSHDB  # WARNING: clears current DB

# Check for runaway keys
redis-cli --bigkeys
```

### 6.8 Git / PR Issues

**Symptoms:** Agents can't push, PRs fail to create, merge conflicts.

**Fix:**
```bash
# Check SSH key is loaded
ssh -T git@github.com

# Clean up orphan branches from agents
git branch -r --merged main | grep -v main | xargs -I {} git push origin --delete {}

# Fix detached HEAD in agent worktree
cd /path/to/worktree
git checkout main
git pull
```

---

## 7. Backup & Recovery

### 7.1 What to Back Up

| Data | Method | Frequency | Retention |
|------|--------|-----------|-----------|
| **Orchestrator DB** (SQLite) | `sqlite3 .backup` | Every 6 hours | 30 days |
| **TimescaleDB** | `pg_dump` | Daily | 90 days |
| **Agent memory files** | rsync to backup dir | Daily | 90 days |
| **Git repos** | Already on GitHub | Real-time (push) | Forever |
| **Qdrant collections** | Qdrant snapshot API | Weekly | 4 snapshots |
| **Neo4j** | `neo4j-admin dump` | Weekly | 4 dumps |
| **Redis** | RDB snapshot | Daily | 7 days |
| **LiteLLM config** | Git-tracked | On change | Forever |
| **Docker volumes** | `docker run --volumes-from` | Weekly | 4 copies |
| **API keys / secrets** | 1Password / Vault | On change | N/A |

### 7.2 Automated Backup Script

Save as `/home/adam/factory/scripts/backup.sh`:

```bash
#!/bin/bash
set -euo pipefail

BACKUP_DIR="/home/adam/backups/factory/$(date +%Y-%m-%d)"
mkdir -p "$BACKUP_DIR"

echo "=== Factory Backup $(date) ==="

# SQLite orchestrator DB
if [ -f /home/adam/agents/shared/orchestrator.db ]; then
  sqlite3 /home/adam/agents/shared/orchestrator.db ".backup '$BACKUP_DIR/orchestrator.db'"
  echo "✅ Orchestrator DB"
fi

# TimescaleDB
docker exec timescaledb pg_dump -U factory factory_analytics | gzip > "$BACKUP_DIR/timescaledb.sql.gz"
echo "✅ TimescaleDB"

# Agent memory
rsync -a --delete /home/adam/.openclaw/workspace-*/memory/ "$BACKUP_DIR/agent-memory/"
echo "✅ Agent memory"

# Qdrant snapshot
curl -sf -X POST "http://localhost:6333/collections/team_memory/snapshots" > /dev/null && echo "✅ Qdrant snapshot"

# Redis
redis-cli BGSAVE
cp /var/lib/redis/dump.rdb "$BACKUP_DIR/redis-dump.rdb" 2>/dev/null && echo "✅ Redis"

# Task queue state
cp -r /home/adam/agents/shared/queue/ "$BACKUP_DIR/queue/"
echo "✅ Task queue"

# Cleanup old backups (keep 30 days)
find /home/adam/backups/factory/ -maxdepth 1 -type d -mtime +30 -exec rm -rf {} +

echo "=== Backup complete: $BACKUP_DIR ==="
ls -lh "$BACKUP_DIR/"
```

Add to crontab:
```cron
# Daily backup at 3am NZDT
0 3 * * * /home/adam/factory/scripts/backup.sh >> /home/adam/factory/logs/backup.log 2>&1
```

### 7.3 Recovery Procedures

**Orchestrator DB:**
```bash
# Stop gateway first
openclaw gateway stop
cp /home/adam/backups/factory/YYYY-MM-DD/orchestrator.db /home/adam/agents/shared/orchestrator.db
openclaw gateway start
```

**TimescaleDB:**
```bash
docker stop timescaledb
docker run --rm -v timescaledb_data:/var/lib/postgresql/data \
  -v /home/adam/backups/factory/YYYY-MM-DD:/backup \
  timescale/timescaledb:latest \
  bash -c "gunzip < /backup/timescaledb.sql.gz | psql -U factory factory_analytics"
docker start timescaledb
```

**Full disaster recovery (dreamteam dies):**
1. Set up new machine with same OS
2. Restore from GitHub (all code repos)
3. Restore from backup dir (databases, agent memory)
4. Re-run cold start procedure (Section 2.1)
5. Verify with health check (Section 4.1)

---

## 8. Cost Management

### 8.1 Daily Cost Targets

| Category | Target | Alert Threshold |
|----------|--------|-----------------|
| Total LLM spend | < $30/day | > $50/day |
| Cloud frontier (Opus/Sonnet) | < $15/day | > $25/day |
| Cloud mid (Gemini/Groq) | < $10/day | > $15/day |
| Local inference (electricity) | ~$1.50/day | N/A |
| Infrastructure (hosting) | < $5/day | > $10/day |

### 8.2 Cost Monitoring Commands

```bash
# LiteLLM spend today (if LiteLLM tracks in DB)
curl -s http://localhost:4000/spend/logs | jq '.[] | select(.startTime > "'$(date +%Y-%m-%d)'") | .spend' | paste -sd+ | bc

# Redis real-time spend counters
redis-cli GET "dash:spend:daily:total"
redis-cli GET "dash:spend:daily:kev"
redis-cli GET "dash:spend:daily:forge"

# Ask Kev via WhatsApp
# "What's our spend today?"
# "Cost breakdown by agent"
```

### 8.3 Cost Reduction Playbook

If spend is too high:

1. **Check for runaway agents** — any single agent burning > $10/day?
   ```bash
   redis-cli KEYS "dash:spend:daily:*" | xargs -I {} redis-cli GET {}
   ```

2. **Check for retry loops** — tasks failing and retrying endlessly
   ```bash
   grep -r "attempt" /home/adam/agents/shared/queue/in-progress/ | grep -v '"attempt":1'
   ```

3. **Force cheaper models** — temporarily route everything through local/budget
   ```bash
   # Edit LiteLLM config to disable frontier tier
   # Or set daily budget cap lower
   ```

4. **Pause non-critical work** — stop background tasks
   ```bash
   # Move backlog tasks out temporarily
   mkdir -p /home/adam/agents/shared/queue/paused/
   mv /home/adam/agents/shared/queue/ready/*.json /home/adam/agents/shared/queue/paused/
   ```

5. **Check prompt caching** — are we getting cache hits?
   ```bash
   # Check LiteLLM logs for cache_hit metrics
   journalctl -u litellm --since "1 hour ago" | grep -c "cache_hit"
   ```

---

## 9. Adam's Daily Interaction Guide

### 9.1 How to Talk to the Factory

**Primary interface: WhatsApp (via Kev)**

| You Say | What Happens |
|---------|-------------|
| "Build a URL shortener SaaS" | Kev decomposes → Scout researches → Atlas writes PRD → Rex builds → Hawk tests → Forge deploys |
| "Status" | Kev reports: active tasks, queue depth, spend, agent health |
| "What's in the queue?" | Kev lists ready + in-progress tasks |
| "Kill task X" | Kev moves task to failed/ |
| "Deploy project-x to production" | Kev triggers Forge deployment pipeline |
| "What did we spend yesterday?" | Kev queries Dash for cost report |
| "Prioritise task X" | Kev bumps task priority |
| ✅ (reply to approval request) | Kev approves the pending action |
| ❌ (reply to approval request) | Kev rejects, agent gets feedback |

### 9.2 Approval Workflow

When you receive an approval request via WhatsApp:

```
🔒 APPROVAL REQUEST

What: Deploy pricing-service v2.3.1 to production
Why: Fixes billing rounding bug
Agent: forge
Risk: Medium
Deadline: 4h

Reply: ✅ approve | ❌ reject | 📝 modify
```

- Reply **✅** to approve
- Reply **❌** to reject (agent will seek alternative)
- Reply **📝** with instructions to modify the approach
- If you don't respond within the deadline, the safe default applies (usually: deny)

### 9.3 What Needs Your Attention

**Always review (Tier 2+):**
- Production deployments
- Customer-facing communications (first contact)
- Spending > $20 per transaction
- DNS/domain/certificate changes
- Public posts (Twitter, blog, Product Hunt)

**Auto-approved (Tier 0-1) — just get notified:**
- Staging deployments
- Internal agent comms
- Research tasks
- Code commits to feature branches
- Spending < $20 per transaction

### 9.4 Weekly Review Checklist

Every Monday, spend 15-30 minutes:

- [ ] Review weekly cost dashboard in Grafana
- [ ] Check dead letter queue — any recurring failures?
- [ ] Review shipped products — any customer feedback?
- [ ] Glance at agent performance metrics — anyone degrading?
- [ ] Approve/reject any stale Tier 2 requests
- [ ] Give Kev new strategic direction if needed

---

## 10. Emergency Procedures

### 10.1 Production Service Down

```bash
# 1. Check what's down
/home/adam/factory/scripts/health-check.sh

# 2. If it's a deployed product:
# Check Railway/Vercel/Cloudflare dashboards
# Rollback if recent deploy caused it:
#   railway up --rollback (Railway)
#   vercel rollback (Vercel)
#   wrangler rollback (Cloudflare)

# 3. If it's the factory itself:
openclaw gateway restart
sudo systemctl restart llama-inference litellm
docker restart redis timescaledb
```

### 10.2 Cost Runaway (Spend > $100/day)

```bash
# 1. IMMEDIATELY: Pause all agent work
openclaw gateway stop

# 2. Check which agent/model is burning money
redis-cli KEYS "dash:spend:daily:*" | xargs -I {} sh -c 'echo {} $(redis-cli GET {})'

# 3. Check for infinite loops
grep -r "attempt" /home/adam/agents/shared/queue/in-progress/ | sort -t: -k2 -n

# 4. Kill runaway tasks
# Move everything to paused
mv /home/adam/agents/shared/queue/ready/*.json /home/adam/agents/shared/queue/paused/ 2>/dev/null
mv /home/adam/agents/shared/queue/in-progress/*.json /home/adam/agents/shared/queue/failed/ 2>/dev/null

# 5. Restart with lower budget
# Edit LiteLLM config: set max_budget lower
# Restart: openclaw gateway start
```

### 10.3 Security Incident

```bash
# 1. Isolate: Stop all external-facing services
openclaw gateway stop
# Stop any deployed products if compromised

# 2. Investigate
# Check git log for unauthorized changes
git log --all --oneline --since="24 hours ago" --author-date-order

# Check for leaked secrets
gitleaks detect --source=/home/adam/projects/

# 3. Rotate credentials
# API keys, SSH keys, tokens — all of them
# Update in 1Password/Vault, then in service configs

# 4. Resume after confirming containment
```

### 10.4 Database Corruption

```bash
# SQLite
sqlite3 /home/adam/agents/shared/orchestrator.db "PRAGMA integrity_check;"
# If corrupt: restore from backup (Section 7.3)

# TimescaleDB
docker exec timescaledb psql -U factory -c "SELECT 1;"
# If not responding: docker restart timescaledb
# If data corrupt: restore from pg_dump backup

# Redis
redis-cli PING
# If corrupt: redis-cli FLUSHALL (lose cache, not critical data)
# Critical data is in SQLite/TimescaleDB, not Redis
```

---

## 11. Maintenance Windows

### 11.1 Safe Maintenance Times

- **Best:** 2am–6am NZDT (overnight, low activity)
- **OK:** Weekends
- **Avoid:** Business hours (8am–6pm NZDT) unless emergency

### 11.2 Planned Maintenance Procedure

```bash
# 1. Notify (via Kev/WhatsApp or just note it)
echo "Maintenance window: $(date) - Reason: [description]"

# 2. Drain the queue
# Let in-progress tasks finish (wait up to 30 min)
# Move ready tasks to paused
mkdir -p /home/adam/agents/shared/queue/paused/
mv /home/adam/agents/shared/queue/ready/*.json /home/adam/agents/shared/queue/paused/ 2>/dev/null

# 3. Stop services (reverse of start order)
openclaw gateway stop
sudo systemctl stop litellm llama-inference llama-embed
docker compose -f /home/adam/factory/docker-compose.yml down

# 4. Do maintenance work
# ... (updates, disk cleanup, hardware changes, etc.)

# 5. Restart (cold start procedure, Section 2.1)

# 6. Restore queue
mv /home/adam/agents/shared/queue/paused/*.json /home/adam/agents/shared/queue/ready/ 2>/dev/null

# 7. Verify
/home/adam/factory/scripts/health-check.sh
```

### 11.3 Routine Maintenance Tasks

| Task | Frequency | Procedure |
|------|-----------|-----------|
| OS updates | Weekly | `sudo apt update && sudo apt upgrade -y` (during maintenance window) |
| Docker image updates | Monthly | `docker compose pull && docker compose up -d` |
| Model updates | As needed | Download GGUF → test → swap symlink → restart llama-inference |
| Log rotation | Automatic | Configured via logrotate (verify: `logrotate --debug /etc/logrotate.conf`) |
| Database vacuum | Monthly | `sqlite3 orchestrator.db "VACUUM;"` / `docker exec timescaledb psql -c "VACUUM ANALYZE;"` |
| Backup verification | Monthly | Restore a backup to temp location, verify data integrity |
| API key rotation | Quarterly | Generate new keys, update configs, restart services |

---

## 12. Runbook Changelog

| Date | Change | Author |
|------|--------|--------|
| 2026-02-08 | Initial version | Forge |

---

*This runbook is a living document. Update it when you discover new failure modes, change infrastructure, or improve procedures.*
