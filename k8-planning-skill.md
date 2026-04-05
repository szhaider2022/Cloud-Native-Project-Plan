# K8 Planning Skill — Reusable Kubernetes Deployment Planner

## Skill Overview

This is a **Claude Code Custom Skill** that generates production-grade Kubernetes deployment plans for **any** application. Instead of manually designing K8s architecture from scratch every time, invoke this skill and it will analyze your project, ask the right questions, and produce a comprehensive deployment plan.

```
┌──────────────────────────────────────────────────────────────┐
│                                                               │
│   INPUT:  Any application description or codebase            │
│                                                               │
│   ┌──────────────────────────────────────────────────────┐   │
│   │              K8 PLANNING SKILL                        │   │
│   │                                                       │   │
│   │  Step 1: Identify microservices & dependencies       │   │
│   │  Step 2: Classify stateful vs stateless              │   │
│   │  Step 3: Design namespace isolation strategy         │   │
│   │  Step 4: Map workloads to K8s resources              │   │
│   │  Step 5: Plan networking & service types             │   │
│   │  Step 6: Define configs, secrets, RBAC               │   │
│   │  Step 7: Design scaling & health checks              │   │
│   │  Step 8: Harden security & network policies          │   │
│   │  Step 9: Plan deployment strategy & rollback         │   │
│   │  Step 10: Generate complete deployment plan          │   │
│   └──────────────────────────────────────────────────────┘   │
│                                                               │
│   OUTPUT: Complete Kubernetes deployment plan (Markdown)      │
│                                                               │
└──────────────────────────────────────────────────────────────┘
```

### How to Use

```bash
# Invoke in Claude Code
/k8-planner e-commerce app with React frontend, Node.js API, PostgreSQL, and Stripe payments

# Or for an existing codebase
/k8-planner

# Or with specific requirements
/k8-planner AI chatbot with GPU inference, vector database, and webhook integrations
```

---

## Skill Installation

### Directory Structure

```
~/.claude/skills/k8-planner/
├── SKILL.md              ← Main skill file (instructions for Claude)
├── reference.md          ← Common patterns, sizing guides, best practices
└── scripts/
    └── validate.sh       ← YAML validation helper
```

### Step 1: Create the skill directory

```bash
mkdir -p ~/.claude/skills/k8-planner/scripts
```

### Step 2: Copy SKILL.md (provided below)
### Step 3: Copy reference.md (provided below)
### Step 4: Copy validate.sh (provided below)
### Step 5: Invoke with `/k8-planner` — no registration needed

---

## SKILL.md — Main Skill File

```yaml
---
name: k8-planner
description: >
  Generate production-grade Kubernetes deployment plans for any application.
  Use when planning K8s deployments, designing cluster architecture, creating
  namespace strategies, RBAC policies, scaling plans, or security hardening.
  Trigger on: "plan kubernetes", "k8s deployment", "deploy to kubernetes",
  "cluster design", "kubernetes architecture".
argument-hint: "[app-description] or leave empty to analyze current project"
disable-model-invocation: false
user-invocable: true
allowed-tools: Read, Grep, Glob, Bash(kubectl *), Bash(docker inspect *), Bash(ls *), Bash(cat *)
effort: high
---

# Kubernetes Deployment Planning Skill

You are a **Senior DevOps Architect** with 20 years of experience designing
Kubernetes deployments for production systems at scale. You design systems that
are secure, reliable, cost-efficient, and operationally excellent.

Your task: Generate a complete Kubernetes deployment plan for the given application.

---

## Phase 1: Discovery & Analysis

If `$ARGUMENTS` is provided, use that as the application description.
If no arguments, analyze the current project by reading:

- Entry points: `package.json`, `go.mod`, `requirements.txt`, `pom.xml`, `Cargo.toml`
- Dockerfiles: `Dockerfile`, `docker-compose.yml`
- Existing K8s configs: `*.yaml`, `*.yml` in `k8s/`, `kubernetes/`, `deploy/`, `manifests/`
- Environment files: `.env.example`, `.env.sample`
- Architecture docs: `README.md`, `ARCHITECTURE.md`, `docs/`

### Questions to answer during discovery:

```
APPLICATION ANALYSIS CHECKLIST
──────────────────────────────
□ What are the distinct services/components?
□ For each service:
  □ Is it stateless or stateful?
  □ What language/runtime does it use?
  □ What port(s) does it expose?
  □ What are its dependencies (other services, databases, external APIs)?
  □ Does it need persistent storage?
  □ What environment variables does it require?
  □ Does it have any sensitive configuration (API keys, passwords)?
  □ What is its expected resource consumption (CPU, memory)?
  □ How does it handle graceful shutdown?
  □ Does it have health check endpoints?
□ What databases/data stores are needed?
□ What external services does it depend on?
□ What is the expected traffic pattern?
□ What is the security sensitivity level?
□ Is there a CI/CD pipeline to integrate with?
```

---

## Phase 2: Architecture Decision Framework

For each service discovered, run through this decision tree:

### Workload Type Selection

```
Is the service stateful?
├── YES: Does it need stable network identity?
│   ├── YES → StatefulSet (databases, distributed systems)
│   └── NO: Does it run once and complete?
│       ├── YES → Job or CronJob
│       └── NO → Deployment with PVC
└── NO: Does it need to run on every node?
    ├── YES → DaemonSet (log collectors, monitoring agents)
    └── NO: Does it run on a schedule?
        ├── YES → CronJob
        └── NO → Deployment
```

### Service Type Selection

```
Who needs to access this service?
├── External users from internet
│   ├── Multiple services sharing one domain → Ingress + ClusterIP
│   ├── Single service, cloud environment → LoadBalancer
│   └── Single service, bare metal → NodePort
├── Other services inside the cluster only → ClusterIP (default)
└── No network access needed (worker/consumer) → Headless or None
```

### Namespace Strategy Selection

```
What is the security sensitivity?
├── HIGH (autonomous agents, financial, healthcare)
│   └── Multi-namespace: separate by trust boundary
│       (gateway, core, execution, data, audit)
├── MEDIUM (standard web apps, internal tools)
│   └── Single namespace per application
│       (isolate from other apps, not internally)
└── LOW (development, prototypes)
    └── Single namespace, minimal policies
```

### Replica Count Selection

```
Service criticality?
├── CRITICAL (auth, gateway, primary API)
│   └── Minimum 3 replicas + pod anti-affinity across nodes
├── IMPORTANT (business logic, workers)
│   └── Minimum 2 replicas
└── AUXILIARY (background jobs, monitoring)
    └── 1-2 replicas acceptable
```

---

## Phase 3: Resource Sizing Guide

### CPU & Memory Estimation

```
┌─────────────────────────────────────────────────────────────────┐
│              RESOURCE SIZING REFERENCE TABLE                     │
│                                                                  │
│  Service Type          CPU Req   CPU Lim   Mem Req   Mem Lim   │
│  ─────────────         ───────   ───────   ───────   ───────   │
│  Static Frontend       50m       200m      64Mi      128Mi     │
│  (Nginx/Caddy)                                                  │
│                                                                  │
│  Light API             100m      500m      128Mi     256Mi     │
│  (Node.js, Go, Rust)                                           │
│                                                                  │
│  Medium API            250m      1000m     256Mi     512Mi     │
│  (Python/Django,                                                │
│   Java/Spring Boot)                                             │
│                                                                  │
│  Heavy API             500m      2000m     512Mi     1Gi       │
│  (AI inference proxy,                                           │
│   data processing)                                              │
│                                                                  │
│  Background Worker     250m      1000m     256Mi     512Mi     │
│  (Queue consumer,                                               │
│   event processor)                                              │
│                                                                  │
│  AI/ML Worker          1000m     4000m     1Gi       4Gi       │
│  (Model inference,                                              │
│   embedding generation)                                         │
│                                                                  │
│  PostgreSQL            500m      2000m     1Gi       2Gi       │
│  Redis                 250m      1000m     512Mi     1Gi       │
│  MongoDB               500m      2000m     1Gi       2Gi       │
│  Elasticsearch         1000m     2000m     2Gi       4Gi       │
│  Vector DB (Qdrant,    500m      2000m     2Gi       4Gi       │
│   Pinecone, Weaviate)                                           │
│  Kafka                 500m      2000m     1Gi       2Gi       │
│  RabbitMQ              250m      1000m     512Mi     1Gi       │
│                                                                  │
│  NOTES:                                                         │
│  • 1000m = 1 CPU core                                          │
│  • Requests = guaranteed minimum (scheduler uses this)         │
│  • Limits = maximum allowed (exceeded CPU → throttle,          │
│    exceeded memory → OOM kill)                                  │
│  • Start conservative, scale based on monitoring data          │
│  • Java apps: set -Xmx to 75% of memory limit                │
│  • Node.js: set --max-old-space-size to 75% of memory limit   │
└─────────────────────────────────────────────────────────────────┘
```

### Storage Sizing

```
Data Type                    Recommended Size    Storage Class
──────────                   ────────────────    ─────────────
Relational DB (small)        10-20Gi             ssd-retain
Relational DB (medium)       50-100Gi            ssd-retain
Relational DB (large)        200Gi+              ssd-retain
Redis/Cache                  5-10Gi              ssd-retain
Vector DB                    20-50Gi             ssd-retain
File uploads                 50-200Gi            standard
Log storage                  100Gi+              standard
ML models                    10-50Gi             ssd-retain
```

---

## Phase 4: Security Planning Matrix

### Secrets Classification

```
For each piece of configuration, classify:

PUBLIC (ConfigMap):
  ✓ Service URLs, ports, hostnames
  ✓ Feature flags, log levels
  ✓ Non-sensitive environment settings
  ✓ Application mode (production/staging)

SENSITIVE (Secret):
  ✓ Database passwords
  ✓ API keys (LLM, payment, email providers)
  ✓ JWT signing keys
  ✓ OAuth client secrets
  ✓ Encryption keys
  ✓ TLS certificates
  ✓ Service account credentials
```

### Secret Rotation Schedule

```
Rotation Period    Secret Types
───────────────    ────────────
30 days            JWT signing keys (support dual-key validation)
60 days            Database credentials (use Vault dynamic secrets)
90 days            Redis passwords, internal service tokens
90 days            LLM/AI API keys
180 days           SMTP credentials, third-party API keys
365 days           Google/AWS service account keys
Never (versioned)  Encryption master keys (version, never delete)
```

### RBAC Design Template

```
For every service, create:
  1. ServiceAccount (never use 'default')
  2. Role with MINIMUM required permissions
  3. RoleBinding connecting ServiceAccount → Role

Permission Levels:
  NONE:       Service doesn't need K8s API access
              → automountServiceAccountToken: false
  READ-ONLY:  Service reads its own ConfigMap/Secret
              → get, list, watch on configmaps/secrets
  STORAGE:    StatefulSet manages its PVCs
              → get, list, watch, create, update on PVCs
  OPERATOR:   Service manages other K8s resources
              → create, update, patch, delete (rare, use carefully)
```

### Network Policy Template

```
Security Level: STANDARD
  → Default deny ingress per namespace
  → Explicit allow rules for each service pair
  → Databases only accessible from application Pods
  → No direct internet access to data stores

Security Level: HIGH (autonomous agents, financial)
  → Default deny BOTH ingress AND egress
  → Explicit allow rules for every connection
  → Egress whitelist for external domains
  → Cross-namespace policies with explicit allows
  → Audit namespace (append-only)
```

---

## Phase 5: Health Check Templates

```
Service Type              Liveness                Readiness                  Startup
──────────────           ──────────              ──────────                ──────────
REST API                 GET /healthz            GET /ready                GET /healthz
                         every 15s               every 10s                 every 5s
                         failureThreshold: 3     failureThreshold: 3       failureThreshold: 30

gRPC Service             grpc health check       grpc health check         grpc health check
                         every 15s               every 10s                 every 5s

Worker/Consumer          GET /healthz            GET /ready                GET /healthz
                         every 20s               every 15s                 every 5s
                         (checks process alive)  (checks queue connected)  failureThreshold: 20

PostgreSQL               exec: pg_isready        exec: pg_isready          exec: pg_isready
                         every 15s               every 10s                 every 5s
                                                                           failureThreshold: 60

Redis                    exec: redis-cli ping    exec: redis-cli ping      exec: redis-cli ping
                         every 15s               every 10s                 every 5s

MongoDB                  exec: mongosh --eval    exec: mongosh --eval      exec: mongosh --eval
                         "db.runCommand('ping')" "db.runCommand('ping')"   "db.runCommand('ping')"
                         every 15s               every 10s                 every 5s

Readiness checks should verify DEPENDENCIES:
  API readiness → check DB connection + cache connection
  Worker readiness → check queue connection
  Frontend readiness → check static files served correctly
```

---

## Phase 6: Scaling Strategy Templates

```
Service Type         Min  Max   Scale Trigger              Scale-Down Cooldown
──────────────       ───  ───   ─────────────              ────────────────────
Public API/Gateway   3    15    CPU > 70% OR RPS > 500     5 min
Business Logic API   2    10    CPU > 70%                  5 min
Background Worker    2    10    Queue depth > 50           10 min
AI/ML Worker         1    8     Queue depth > 10           10 min (slow to start)
Event Consumer       2    8     Consumer lag > 1000        5 min
Frontend/UI          2    6     CPU > 70%                  5 min

NEVER auto-scale:
  • Databases (PostgreSQL, MongoDB) — scaling requires replication setup
  • Message brokers (Kafka, RabbitMQ) — scaling requires partition rebalancing
  • StatefulSets in general — scale manually with careful planning
```

---

## Phase 7: Deployment Strategy Templates

### Rollout Order

```
Always deploy in this order:

Phase 1: Data Layer (databases, caches, queues)
  → Wait for health checks to pass
  → Run migrations if needed

Phase 2: Core Services (auth, business logic, internal APIs)
  → Auth/security services FIRST
  → Business logic services NEXT
  → Wait for readiness probes

Phase 3: Workers (background processors, consumers)
  → Connect to queues and start processing

Phase 4: Frontend/Gateway (user-facing components)
  → Only after all backends are healthy

Phase 5: Ingress/Routing (expose to traffic)
  → Apply Ingress rules last

Phase 6: Network Policies (lock down)
  → Apply after everything is running
```

### Update Strategy by Service Type

```
Stateless Services (APIs, frontends, workers):
  strategy: RollingUpdate
  maxSurge: 1
  maxUnavailable: 0    (for critical services)
  maxUnavailable: 1    (for non-critical services)

Databases (StatefulSets):
  updateStrategy: OnDelete    (manual, controlled updates)
  NEVER auto-restart databases

Rollback:
  kubectl rollout undo deployment/<name> -n <namespace>
  progressDeadlineSeconds: 300 (mark failed after 5 min)
```

---

## Phase 8: Output Format

Generate the deployment plan in this exact structure:

```markdown
# [Application Name] — Kubernetes Deployment Plan

## 1. Application Overview
  - Service table (name, role, nature)
  - Supporting infrastructure table

## 2. Architecture Diagram
  - ASCII diagram showing all components
  - Data flow description

## 3. Namespace Design
  - Namespace strategy with reasoning
  - Resource quotas per namespace

## 4. Pods, Deployments & StatefulSets
  - For EACH service: Kind, Replicas, Image, Port, Strategy with reasoning

## 5. Services & Networking
  - Service map diagram
  - Service table (name, type, port, reasoning)

## 6. ConfigMaps
  - For EACH service: all key-value pairs with purpose

## 7. Secrets Management & Expiry Handling
  - Secret definitions with sensitivity levels
  - Rotation schedule and method for each secret
  - Secret injection method (env vars vs volume mounts)

## 8. Resource Requests & Limits
  - Per-service resource table with reasoning
  - Total cluster footprint calculation

## 9. RBAC — Roles & RoleBindings
  - ServiceAccount per service
  - Role definitions
  - RoleBinding table
  - Access matrix (visual grid)
  - Human access roles

## 10. Inter-Service Communication
  - Communication flow diagram
  - Protocol/DNS/pattern table for every service pair
  - Sync vs Async reasoning

## 11. Scaling Strategy
  - HPA configuration per service
  - Scaling scenario example

## 12. Health Checks
  - Liveness, Readiness, Startup probes per service

## 13. Storage Design
  - PVC table with sizes and storage classes
  - Backup strategy

## 14. Security Hardening
  - Network policies (diagram)
  - Pod security standards
  - "What can reach what" matrix

## 15. Deployment Strategy
  - Rollout order (phased)
  - Update strategy per service
  - Rollback plan
```

---

## Phase 9: Quality Checklist

Before finalizing the plan, verify:

```
COMPLETENESS
□ Every service has a Deployment or StatefulSet defined
□ Every service has a Service resource
□ Every service has resource requests AND limits
□ Every service has health checks (liveness + readiness)
□ Every service has its own ServiceAccount
□ Every sensitive value is in a Secret (not ConfigMap)
□ Every non-sensitive config is in a ConfigMap (not hardcoded)
□ Inter-service communication is fully documented
□ Architecture diagram matches the service definitions

SECURITY
□ No service uses the 'default' ServiceAccount
□ All Pods run as non-root
□ Read-only root filesystem where possible
□ All capabilities dropped
□ Network policies restrict traffic to minimum needed
□ Databases are NOT accessible from internet-facing services directly
□ Secrets have rotation strategy defined
□ RBAC follows least-privilege principle

RELIABILITY
□ Critical services have 3+ replicas
□ Pod anti-affinity for critical services (spread across nodes)
□ StatefulSets used for stateful services (not Deployments with PVCs)
□ Graceful shutdown configured (terminationGracePeriodSeconds)
□ PodDisruptionBudget for critical services
□ Rollback plan documented

SCALABILITY
□ HPA configured for all stateless services
□ Databases are NOT auto-scaled
□ Resource limits allow burst capacity (limits > requests)
□ Cluster sizing accounts for scaling headroom (50%+ over baseline)

OPERATIONS
□ Deployment order specified
□ Update strategy defined per service type
□ Backup strategy for persistent data
□ Monitoring/observability considered
```
```

---

## reference.md — Supporting Reference File

```markdown
# K8s Planning Reference Guide

## Common Architecture Patterns

### Pattern 1: Simple Web Application
```
Internet → Ingress → Frontend (Deployment) → API (Deployment) → Database (StatefulSet)
```
- 1 namespace
- 3-4 services
- Standard security

### Pattern 2: Microservices with Message Queue
```
Internet → Ingress → Gateway → Services (multiple) → Queue → Workers → Database
```
- 1-2 namespaces
- 5-10 services
- Async communication via queue

### Pattern 3: AI/Agent System with Tool Execution
```
Internet → Gateway → Orchestrator → LLM Service
                                   → Auth Service → Tool Executor (sandboxed)
                                   → Memory Service → Vector DB
```
- 3-5 namespaces (security boundaries)
- 6-10 services
- Zero-trust security model
- Sandboxed tool execution

### Pattern 4: Data Pipeline
```
Ingestion → Queue → Processors (parallel) → Data Store → API → Dashboard
```
- 2 namespaces (processing, serving)
- CronJobs for scheduled tasks
- High storage requirements

## Service Type Quick Reference

| Cloud Provider | LoadBalancer | Ingress Controller | Storage Class |
|---------------|-------------|-------------------|---------------|
| AWS EKS | AWS NLB/ALB | AWS ALB Ingress Controller | gp3 (SSD), io2 (high IOPS) |
| GCP GKE | Google Cloud LB | GKE Ingress | pd-ssd, pd-standard |
| Azure AKS | Azure LB | Azure App Gateway Ingress | managed-premium (SSD) |
| Bare Metal | MetalLB | Nginx Ingress | local-storage, NFS |
| Minikube | minikube tunnel | Nginx Ingress (addon) | standard (hostpath) |

## Common Mistakes to Avoid

1. **Using `latest` image tags** → Always use immutable version tags
2. **No resource limits** → Pods can consume entire node resources
3. **Secrets in ConfigMaps** → Always use K8s Secrets for sensitive data
4. **Single replica for critical services** → One Pod failure = downtime
5. **No health checks** → K8s cannot detect or recover from failures
6. **LoadBalancer per service** → Expensive; use Ingress instead
7. **Default ServiceAccount** → Shares permissions across all Pods
8. **No network policies** → Any Pod can reach any other Pod
9. **Hardcoded config in images** → Use ConfigMaps for environment-specific config
10. **No graceful shutdown** → Requests dropped during deployments
```

---

## scripts/validate.sh — Validation Script

```bash
#!/bin/bash
# ─────────────────────────────────────────────────
# K8s Manifest Validator
# Validates generated Kubernetes YAML files
# Usage: ./validate.sh <manifest-directory>
# ─────────────────────────────────────────────────

set -euo pipefail

RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
NC='\033[0m' # No Color

MANIFEST_DIR="${1:-.}"
ERRORS=0
WARNINGS=0

echo "========================================="
echo "  K8s Manifest Validator"
echo "========================================="
echo ""
echo "Scanning: $MANIFEST_DIR"
echo ""

# Check if kubectl is available
if ! command -v kubectl &> /dev/null; then
    echo -e "${RED}ERROR: kubectl not found. Install kubectl to validate manifests.${NC}"
    exit 1
fi

# Find all YAML files
YAML_FILES=$(find "$MANIFEST_DIR" -name "*.yaml" -o -name "*.yml" 2>/dev/null)

if [ -z "$YAML_FILES" ]; then
    echo -e "${YELLOW}WARNING: No YAML files found in $MANIFEST_DIR${NC}"
    exit 0
fi

# Validate each file
for file in $YAML_FILES; do
    echo -n "Checking $(basename "$file")... "

    # Dry-run validation
    if kubectl apply --dry-run=client -f "$file" > /dev/null 2>&1; then
        echo -e "${GREEN}VALID${NC}"
    else
        echo -e "${RED}INVALID${NC}"
        kubectl apply --dry-run=client -f "$file" 2>&1 | head -5
        ERRORS=$((ERRORS + 1))
    fi
done

echo ""

# Check for common issues
echo "Running best-practice checks..."
echo ""

for file in $YAML_FILES; do
    # Check for 'latest' tag
    if grep -q "image:.*:latest" "$file" 2>/dev/null; then
        echo -e "${YELLOW}WARNING: $file uses ':latest' tag — use versioned tags${NC}"
        WARNINGS=$((WARNINGS + 1))
    fi

    # Check for missing resource limits
    if grep -q "kind: Deployment\|kind: StatefulSet" "$file" 2>/dev/null; then
        if ! grep -q "resources:" "$file" 2>/dev/null; then
            echo -e "${YELLOW}WARNING: $file has no resource requests/limits${NC}"
            WARNINGS=$((WARNINGS + 1))
        fi
    fi

    # Check for missing health checks
    if grep -q "kind: Deployment" "$file" 2>/dev/null; then
        if ! grep -q "livenessProbe\|readinessProbe" "$file" 2>/dev/null; then
            echo -e "${YELLOW}WARNING: $file has no health probes${NC}"
            WARNINGS=$((WARNINGS + 1))
        fi
    fi

    # Check for default service account
    if grep -q "serviceAccountName: default" "$file" 2>/dev/null; then
        echo -e "${YELLOW}WARNING: $file uses 'default' ServiceAccount — create a dedicated one${NC}"
        WARNINGS=$((WARNINGS + 1))
    fi
done

echo ""
echo "========================================="
echo "  Results"
echo "========================================="
echo -e "  Errors:   ${RED}$ERRORS${NC}"
echo -e "  Warnings: ${YELLOW}$WARNINGS${NC}"
echo ""

if [ $ERRORS -gt 0 ]; then
    echo -e "${RED}FAILED: $ERRORS invalid manifest(s)${NC}"
    exit 1
else
    echo -e "${GREEN}PASSED: All manifests are valid${NC}"
    exit 0
fi
```

---

## Skill Evaluation (Optional)

To evaluate this skill's accuracy and reliability, test it against these scenarios:

### Eval Test Cases

| # | Input | Expected Output Should Include |
|---|-------|-------------------------------|
| 1 | "Simple blog with Next.js and PostgreSQL" | 1 namespace, Deployment for frontend, Deployment for API, StatefulSet for PostgreSQL, ClusterIP + Ingress, basic RBAC |
| 2 | "E-commerce with React, Node.js API, PostgreSQL, Redis, Stripe" | Separate ConfigMaps for each service, Stripe API key in Secret, Redis as cache (StatefulSet), HPA for API |
| 3 | "Real-time chat with WebSocket, Node.js, MongoDB, Redis pub/sub" | WebSocket support in Ingress annotations, Redis for pub/sub, MongoDB StatefulSet, sticky sessions consideration |
| 4 | "AI agent that can execute code and send emails" | Multi-namespace (security boundary), sandboxed execution, network policy egress whitelist, audit logging, RBAC strict isolation |
| 5 | "Cron job that processes CSV uploads daily" | CronJob resource, PVC for file storage, no Service needed, resource limits to prevent runaway processing |
| 6 | "ML model serving with GPU inference" | GPU resource requests (`nvidia.com/gpu`), node affinity for GPU nodes, larger memory limits, HPA on custom metric (inference latency) |
| 7 | "Internal dashboard — only company employees" | No LoadBalancer (internal only), VPN/SSO integration notes, ClusterIP + internal Ingress, corporate namespace |

### How to Run Evaluation

```bash
# Test each scenario and verify the output includes expected elements
# The skill should produce a complete plan for each, not partial output

# Example evaluation prompt:
/k8-planner Simple blog with Next.js and PostgreSQL

# Then verify:
# ✓ Does the output have all 15 sections?
# ✓ Are resource values reasonable for the service type?
# ✓ Is every service covered (no missing components)?
# ✓ Are security best practices followed?
# ✓ Is the architecture diagram consistent with the service definitions?
# ✓ Would this plan work in a real production environment?
```

### Evaluation Scoring Rubric

| Category | Weight | Criteria |
|----------|--------|----------|
| **Completeness** | 25% | All 15 sections present, every service covered, no missing resources |
| **Accuracy** | 25% | Correct resource types (Deployment vs StatefulSet), appropriate Service types, valid resource values |
| **Security** | 20% | RBAC defined, Secrets not in ConfigMaps, network policies present, non-root containers |
| **Reasoning** | 15% | Every decision has a "why" — not just what to deploy, but why this way |
| **Practicality** | 15% | Resource values are realistic, scaling strategy makes sense, rollback plan is actionable |

---

## Complete Installation Script

```bash
#!/bin/bash
# ─────────────────────────────────────────────────
# K8 Planning Skill Installer
# Installs the skill to ~/.claude/skills/k8-planner/
# ─────────────────────────────────────────────────

set -euo pipefail

SKILL_DIR="$HOME/.claude/skills/k8-planner"

echo "Installing K8 Planning Skill..."

# Create directory structure
mkdir -p "$SKILL_DIR/scripts"

echo "Created: $SKILL_DIR"
echo ""
echo "Now copy the following files from this document:"
echo "  1. SKILL.md content      → $SKILL_DIR/SKILL.md"
echo "  2. reference.md content  → $SKILL_DIR/reference.md"
echo "  3. validate.sh content   → $SKILL_DIR/scripts/validate.sh"
echo ""
echo "Then make the script executable:"
echo "  chmod +x $SKILL_DIR/scripts/validate.sh"
echo ""
echo "Done! Use /k8-planner in Claude Code to invoke the skill."
```
