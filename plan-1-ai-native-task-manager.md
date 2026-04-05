# Plan 1: AI Native Task Manager — Kubernetes Deployment Plan

## Table of Contents

1. [Application Overview](#1-application-overview)
2. [Architecture Diagram](#2-architecture-diagram)
3. [Namespace Design](#3-namespace-design)
4. [Pods, Deployments & StatefulSets](#4-pods-deployments--statefulsets)
5. [Services & Networking](#5-services--networking)
6. [ConfigMaps](#6-configmaps)
7. [Secrets Management & Expiry Handling](#7-secrets-management--expiry-handling)
8. [Resource Requests & Limits](#8-resource-requests--limits)
9. [RBAC — Roles & RoleBindings](#9-rbac--roles--rolebindings)
10. [Inter-Service Communication](#10-inter-service-communication)
11. [Scaling Strategy](#11-scaling-strategy)
12. [Health Checks](#12-health-checks)
13. [Storage Design](#13-storage-design)
14. [Security Hardening](#14-security-hardening)
15. [Deployment Strategy](#15-deployment-strategy)

---

## 1. Application Overview

The AI Native Task Manager is a microservices-based application consisting of four core services:

| Service | Role | Nature |
|---------|------|--------|
| **UI Interface** | Web-based frontend for user interaction — task creation, dashboards, real-time updates | Stateless |
| **Backend APIs** | Central REST/gRPC API server — authentication, task CRUD, business logic, data validation | Stateless |
| **Task Agent** | AI-powered worker — processes tasks using LLM inference, generates suggestions, auto-categorizes, prioritizes | Stateless (compute-heavy) |
| **Notification Service** | Event-driven service — sends email, push, and in-app notifications based on task events | Stateless |

**Supporting Infrastructure (required by the core services):**

| Component | Role | Nature |
|-----------|------|--------|
| **PostgreSQL** | Primary relational database — stores users, tasks, projects, audit logs | Stateful |
| **Redis** | In-memory cache + message broker — session cache, task queue, pub/sub for real-time events | Stateful |

---

## 2. Architecture Diagram

```
                          ┌──────────────────────────────┐
                          │          INTERNET             │
                          └──────────────┬───────────────┘
                                         │
                                         ▼
                          ┌──────────────────────────────┐
                          │     Ingress Controller        │
                          │  (NGINX / Cloud ALB)          │
                          │                               │
                          │  app.example.com/*  ──► UI    │
                          │  api.example.com/*  ──► API   │
                          └──────────┬──────────┬────────┘
                                     │          │
                    ┌────────────────┘          └────────────────┐
                    ▼                                            ▼
         ┌──────────────────┐                        ┌──────────────────┐
         │   UI Interface    │                        │   Backend APIs    │
         │   (Deployment)    │                        │   (Deployment)    │
         │   Replicas: 2     │                        │   Replicas: 3     │
         │   Port: 3000      │                        │   Port: 8080      │
         └──────────────────┘                        └────────┬─────────┘
                                                              │
                                          ┌───────────────────┼───────────────────┐
                                          │                   │                   │
                                          ▼                   ▼                   ▼
                               ┌──────────────────┐ ┌─────────────────┐ ┌─────────────────┐
                               │   Task Agent      │ │   PostgreSQL     │ │     Redis        │
                               │   (Deployment)    │ │   (StatefulSet)  │ │  (StatefulSet)   │
                               │   Replicas: 2     │ │   Replicas: 2    │ │  Replicas: 3     │
                               │   Port: 9090      │ │   Port: 5432     │ │  Port: 6379      │
                               └────────┬─────────┘ └─────────────────┘ └────────┬────────┘
                                        │                                         │
                                        │          ┌──────────────────┐           │
                                        └─────────►│ Notification Svc  │◄──────────┘
                                         via Redis │ (Deployment)      │  subscribes to
                                         queue     │ Replicas: 2       │  event channel
                                                   │ Port: 8081        │
                                                   └──────────────────┘
```

**Data Flow:**
```
User ──► UI ──► Backend API ──► PostgreSQL (read/write tasks)
                    │
                    ├──► Task Agent (via Redis queue: "process this task with AI")
                    │         │
                    │         └──► Backend API (returns AI result)
                    │
                    └──► Redis pub/sub (publish event: "task updated")
                              │
                              └──► Notification Service (subscribes, sends alert)
```

---

## 3. Namespace Design

All resources for this application are isolated within a dedicated namespace, separated from other workloads and system services.

```
┌──────────────────────────────────────────────────────────────┐
│                     CLUSTER                                   │
│                                                               │
│  ┌────────────────────────────────────────┐                  │
│  │  Namespace: ai-task-manager             │                  │
│  │                                         │                  │
│  │  All application Deployments,           │                  │
│  │  StatefulSets, Services, ConfigMaps,    │                  │
│  │  Secrets, ServiceAccounts, Roles        │                  │
│  └────────────────────────────────────────┘                  │
│                                                               │
│  ┌────────────────────────────────────────┐                  │
│  │  Namespace: ingress-nginx  (system)     │  ← Managed      │
│  └────────────────────────────────────────┘    separately    │
│                                                               │
│  ┌────────────────────────────────────────┐                  │
│  │  Namespace: monitoring     (system)     │  ← Prometheus,  │
│  └────────────────────────────────────────┘    Grafana       │
└──────────────────────────────────────────────────────────────┘
```

**Why a single application namespace?**
- All four services are tightly coupled and belong to one product
- Simplifies inter-service DNS (no cross-namespace references needed)
- RBAC is scoped to this namespace — other teams/apps cannot interfere
- Resource quotas can be applied at the namespace level to cap total consumption

**Namespace resource quota (applied to `ai-task-manager`):**

| Resource | Quota |
|----------|-------|
| Total CPU requests | 8 cores |
| Total memory requests | 16Gi |
| Total CPU limits | 16 cores |
| Total memory limits | 32Gi |
| Max Pods | 50 |
| Max Services | 15 |
| Max ConfigMaps | 20 |
| Max Secrets | 20 |
| Max PersistentVolumeClaims | 10 |

---

## 4. Pods, Deployments & StatefulSets

### 4.1 UI Interface — Deployment

| Property | Value | Reasoning |
|----------|-------|-----------|
| **Kind** | Deployment | Stateless — serves static assets and client-side app, any replica can handle any request |
| **Replicas** | 2 | High availability — if one Pod fails, the other serves traffic with zero downtime |
| **Image** | `ai-task-manager/ui:1.0.0` | Immutable tags for reproducible deployments (never use `latest`) |
| **Container Port** | 3000 | Standard port for Node.js/Next.js serving |
| **Strategy** | RollingUpdate (maxSurge: 1, maxUnavailable: 0) | Zero-downtime deploys — new Pod must be healthy before old one is terminated |

### 4.2 Backend APIs — Deployment

| Property | Value | Reasoning |
|----------|-------|-----------|
| **Kind** | Deployment | Stateless — all state is in PostgreSQL/Redis, API server holds no local data |
| **Replicas** | 3 | Handles the highest traffic volume; 3 replicas distribute load and survive node failures |
| **Image** | `ai-task-manager/backend-api:1.0.0` | |
| **Container Port** | 8080 | Standard HTTP API port |
| **Strategy** | RollingUpdate (maxSurge: 1, maxUnavailable: 1) | One Pod can be down during update since 2 remaining replicas handle traffic |

### 4.3 Task Agent — Deployment

| Property | Value | Reasoning |
|----------|-------|-----------|
| **Kind** | Deployment | Stateless — pulls tasks from Redis queue, processes via LLM API, writes result back. No local state. |
| **Replicas** | 2 | AI inference is slow; 2 workers process tasks in parallel. Scale with HPA when queue depth grows. |
| **Image** | `ai-task-manager/task-agent:1.0.0` | |
| **Container Port** | 9090 | gRPC or health check endpoint |
| **Strategy** | RollingUpdate (maxSurge: 1, maxUnavailable: 0) | Never lose an in-progress task during deployment |

### 4.4 Notification Service — Deployment

| Property | Value | Reasoning |
|----------|-------|-----------|
| **Kind** | Deployment | Stateless — subscribes to Redis pub/sub, sends notifications via external providers (SMTP, FCM) |
| **Replicas** | 2 | Ensures notifications are delivered even if one Pod fails. Duplicate delivery is prevented via idempotency keys in Redis. |
| **Image** | `ai-task-manager/notification-svc:1.0.0` | |
| **Container Port** | 8081 | Health check and metrics endpoint |
| **Strategy** | RollingUpdate (maxSurge: 1, maxUnavailable: 1) | Brief gap in notifications is acceptable; they are retried from the queue |

### 4.5 PostgreSQL — StatefulSet

| Property | Value | Reasoning |
|----------|-------|-----------|
| **Kind** | StatefulSet | Stateful — data must survive Pod restarts. Each replica has a stable network identity and its own persistent volume. |
| **Replicas** | 2 | 1 primary (read/write) + 1 read replica (failover + read scaling). NOT a Deployment because Pods need stable names (`postgres-0`, `postgres-1`) for replication. |
| **Image** | `postgres:16-alpine` | Alpine for smaller image; PostgreSQL 16 for latest features |
| **Container Port** | 5432 | Standard PostgreSQL port |
| **Volume** | 20Gi SSD PVC per replica | Persistent storage that outlives Pod restarts |
| **Pod Management Policy** | OrderedReady | Primary (`postgres-0`) must be running before replica (`postgres-1`) starts |

### 4.6 Redis — StatefulSet

| Property | Value | Reasoning |
|----------|-------|-----------|
| **Kind** | StatefulSet | Stateful — Redis Sentinel mode requires stable Pod identities for leader election and replication |
| **Replicas** | 3 | 1 master + 2 replicas with Sentinel for automatic failover (requires odd number for quorum) |
| **Image** | `redis:7-alpine` | |
| **Container Port** | 6379 (data), 26379 (sentinel) | |
| **Volume** | 5Gi SSD PVC per replica | Persistence for queue data — prevents task loss on restart |

---

## 5. Services & Networking

### Service Map

```
┌───────────────────────────────────────────────────────────────────────────┐
│                                                                           │
│  EXTERNAL ACCESS (via Ingress)                                           │
│  ┌─────────────────────────────────────────────────────────────────┐     │
│  │  Ingress: ai-task-manager-ingress                                │     │
│  │                                                                   │     │
│  │  Rule 1: app.example.com  ──► ui-svc (ClusterIP:3000)           │     │
│  │  Rule 2: api.example.com  ──► backend-api-svc (ClusterIP:8080)  │     │
│  │                                                                   │     │
│  │  TLS: Terminated at Ingress (cert-manager + Let's Encrypt)       │     │
│  └─────────────────────────────────────────────────────────────────┘     │
│                                                                           │
│  INTERNAL SERVICES (ClusterIP — not exposed outside)                     │
│  ┌──────────────────────┐  ┌──────────────────────┐                     │
│  │ task-agent-svc        │  │ notification-svc      │                     │
│  │ ClusterIP:9090        │  │ ClusterIP:8081        │                     │
│  └──────────────────────┘  └──────────────────────┘                     │
│  ┌──────────────────────┐  ┌──────────────────────┐                     │
│  │ postgres-svc          │  │ redis-svc             │                     │
│  │ ClusterIP:5432        │  │ ClusterIP:6379        │                     │
│  └──────────────────────┘  └──────────────────────┘                     │
│  ┌──────────────────────┐                                                │
│  │ postgres-svc-readonly │  Headless service for StatefulSet             │
│  │ ClusterIP:5432        │  (routes to read replica only)                │
│  └──────────────────────┘                                                │
│                                                                           │
└───────────────────────────────────────────────────────────────────────────┘
```

### Service Definitions

| Service Name | Type | Port | Target | Reason for Type |
|-------------|------|------|--------|-----------------|
| `ui-svc` | ClusterIP | 3000 | UI Deployment | Accessed via Ingress (not directly from internet). ClusterIP is sufficient — Ingress handles external routing. |
| `backend-api-svc` | ClusterIP | 8080 | Backend API Deployment | Same as UI — Ingress terminates TLS and routes `api.example.com` traffic here. |
| `task-agent-svc` | ClusterIP | 9090 | Task Agent Deployment | Internal only — Backend API calls Task Agent directly or via Redis queue. No external access needed. |
| `notification-svc` | ClusterIP | 8081 | Notification Deployment | Internal only — triggered by Redis pub/sub events, never called by users directly. |
| `postgres-svc` | ClusterIP | 5432 | PostgreSQL StatefulSet (primary) | Internal only — only Backend API needs database access. Exposing a database externally is a critical security risk. |
| `postgres-readonly-svc` | ClusterIP | 5432 | PostgreSQL StatefulSet (replica) | Separate service for read-heavy queries — distributes load off the primary. |
| `redis-svc` | ClusterIP | 6379 | Redis StatefulSet (master) | Internal only — used as cache and message broker by Backend API, Task Agent, and Notification Service. |

**Why NOT LoadBalancer for UI and API?**

Using Ingress + ClusterIP instead of individual LoadBalancers because:
- LoadBalancer per service = 1 public IP per service = higher cloud cost
- Ingress consolidates all external traffic through a single entry point
- Ingress provides TLS termination, path-based routing, rate limiting, and CORS — all in one place
- In production, the Ingress Controller itself is backed by one cloud LoadBalancer

---

## 6. ConfigMaps

### 6.1 `ui-config`

| Key | Value | Purpose |
|-----|-------|---------|
| `API_BASE_URL` | `https://api.example.com` | Tells the frontend where to send API requests |
| `APP_ENV` | `production` | Controls feature flags and logging behavior |
| `ENABLE_ANALYTICS` | `true` | Toggle analytics collection |
| `WS_ENDPOINT` | `wss://api.example.com/ws` | WebSocket endpoint for real-time task updates |

### 6.2 `backend-api-config`

| Key | Value | Purpose |
|-----|-------|---------|
| `DATABASE_HOST` | `postgres-svc` | K8s DNS name — resolves to PostgreSQL ClusterIP |
| `DATABASE_PORT` | `5432` | Standard PostgreSQL port |
| `DATABASE_NAME` | `taskmanager` | Application database |
| `DATABASE_READ_HOST` | `postgres-readonly-svc` | Read replica for heavy queries |
| `REDIS_HOST` | `redis-svc` | K8s DNS name — resolves to Redis ClusterIP |
| `REDIS_PORT` | `6379` | Standard Redis port |
| `TASK_AGENT_URL` | `http://task-agent-svc:9090` | Internal service URL for direct agent calls |
| `LOG_LEVEL` | `info` | Production logging level |
| `CORS_ORIGINS` | `https://app.example.com` | Allowed origins for CORS |
| `RATE_LIMIT_RPM` | `100` | Rate limit: 100 requests per minute per user |
| `MAX_UPLOAD_SIZE_MB` | `10` | File attachment size limit |

### 6.3 `task-agent-config`

| Key | Value | Purpose |
|-----|-------|---------|
| `REDIS_HOST` | `redis-svc` | Queue connection for consuming task jobs |
| `REDIS_PORT` | `6379` | |
| `TASK_QUEUE_NAME` | `ai-task-processing` | Redis queue name the agent listens on |
| `LLM_MODEL` | `gpt-4o` | Which LLM model to use for task processing |
| `LLM_TIMEOUT_MS` | `30000` | 30-second timeout for LLM API calls |
| `MAX_RETRIES` | `3` | Retry failed AI processing up to 3 times |
| `CALLBACK_URL` | `http://backend-api-svc:8080/internal/task-result` | Where to POST AI results back |
| `CONCURRENCY` | `5` | Process 5 tasks simultaneously per Pod |

### 6.4 `notification-config`

| Key | Value | Purpose |
|-----|-------|---------|
| `REDIS_HOST` | `redis-svc` | Subscribes to Redis pub/sub channels |
| `REDIS_PORT` | `6379` | |
| `EVENT_CHANNEL` | `task-events` | Redis pub/sub channel name |
| `SMTP_HOST` | `smtp.sendgrid.net` | Email provider host |
| `SMTP_PORT` | `587` | TLS SMTP port |
| `FROM_EMAIL` | `noreply@example.com` | Sender email address |
| `FCM_PROJECT_ID` | `task-manager-prod` | Firebase project for push notifications |
| `RETRY_DELAY_MS` | `5000` | Wait 5 seconds before retrying failed notifications |

---

## 7. Secrets Management & Expiry Handling

### 7.1 Secret Definitions

| Secret Name | Keys | Used By | Sensitivity |
|------------|------|---------|-------------|
| `db-credentials` | `DATABASE_USER`, `DATABASE_PASSWORD` | Backend API | CRITICAL — full database access |
| `redis-credentials` | `REDIS_PASSWORD` | Backend API, Task Agent, Notification Service | HIGH — access to cache and queues |
| `llm-api-key` | `LLM_API_KEY` | Task Agent | CRITICAL — LLM provider API key (costly if leaked) |
| `jwt-secret` | `JWT_SIGNING_KEY` | Backend API | CRITICAL — if leaked, attackers can forge auth tokens |
| `smtp-credentials` | `SMTP_USERNAME`, `SMTP_PASSWORD` | Notification Service | MEDIUM — email sending capability |
| `fcm-service-account` | `FCM_KEY_JSON` | Notification Service | MEDIUM — push notification capability |
| `encryption-key` | `DATA_ENCRYPTION_KEY` | Backend API | CRITICAL — encrypts sensitive task data at rest |

### 7.2 Secret Injection Method

Secrets are mounted as **environment variables** (not files) for simple key-value secrets, and as **volume mounts** for multi-line secrets (like `FCM_KEY_JSON`):

```
Pod Spec (Backend API):
  env:
    - name: DATABASE_PASSWORD
      valueFrom:
        secretKeyRef:
          name: db-credentials
          key: DATABASE_PASSWORD

Pod Spec (Notification Service):
  volumes:
    - name: fcm-key
      secret:
        secretName: fcm-service-account
  containers:
    - volumeMounts:
        - name: fcm-key
          mountPath: /etc/secrets/fcm
          readOnly: true
```

### 7.3 Expiry & Rotation Strategy

```
┌────────────────────────────────────────────────────────────┐
│               SECRET ROTATION LIFECYCLE                     │
│                                                             │
│   ┌──────────┐    ┌──────────┐    ┌──────────┐            │
│   │  Create   │──►│  Monitor  │──►│  Rotate   │──┐        │
│   │  Secret   │   │  Expiry   │   │  Secret   │  │        │
│   └──────────┘    └──────────┘    └──────────┘  │        │
│        ▲                                          │        │
│        └──────────────────────────────────────────┘        │
│                                                             │
│   Tool: External Secrets Operator (ESO)                    │
│   Source: AWS Secrets Manager / HashiCorp Vault             │
│   Auto-sync: Every 1 hour                                  │
└────────────────────────────────────────────────────────────┘
```

| Secret | Rotation Period | Method |
|--------|----------------|--------|
| `db-credentials` | 90 days | Vault dynamic secrets — generates temporary DB credentials with TTL. Old credentials remain valid for 24h overlap. |
| `redis-credentials` | 90 days | Manual rotation with ESO sync. Update in Vault → ESO syncs to K8s Secret → Pods restart via rolling update. |
| `llm-api-key` | 180 days | Rotate in LLM provider dashboard → update in Vault → ESO auto-syncs. |
| `jwt-secret` | 180 days | Support dual-key validation during rotation: accept tokens signed by old OR new key for 24 hours, then drop old key. |
| `smtp-credentials` | 365 days | Low risk, annual rotation via provider portal. |
| `encryption-key` | Never rotate (versioned) | Use key versioning — new data encrypted with v2, old data decrypted with v1. Never delete old key versions. |

**Why External Secrets Operator (ESO)?**
- Native K8s Secrets are base64-encoded (not encrypted) — anyone with namespace access can decode them
- ESO syncs from a secure vault (AWS Secrets Manager, HashiCorp Vault) → K8s Secret
- Central audit log of who accessed which secret and when
- Automatic rotation without manual `kubectl` commands

---

## 8. Resource Requests & Limits

### Resource Allocation Table

| Service | CPU Request | CPU Limit | Memory Request | Memory Limit | Reasoning |
|---------|------------|-----------|---------------|-------------|-----------|
| **UI Interface** | 100m | 300m | 128Mi | 256Mi | Serves pre-built static assets. Very lightweight. Nginx/Node serve is not CPU intensive. |
| **Backend API** | 250m | 1000m | 256Mi | 512Mi | Handles HTTP request processing, JSON serialization, DB queries. Moderate and bursty. |
| **Task Agent** | 500m | 2000m | 512Mi | 1Gi | CPU-intensive prompt construction and response parsing. LLM call itself is external (API), but pre/post-processing is heavy. |
| **Notification Service** | 100m | 500m | 128Mi | 256Mi | Lightweight — formats and sends HTTP requests to SMTP/FCM. Low resource footprint. |
| **PostgreSQL** | 500m | 2000m | 1Gi | 2Gi | Database operations are memory-intensive (buffer pool, query cache). CPU spikes during complex queries. |
| **Redis** | 250m | 1000m | 512Mi | 1Gi | In-memory data store — memory is the critical resource. CPU is low except during persistence (RDB/AOF). |

### Total Resource Footprint (all replicas combined)

```
Service              Replicas    Total CPU Req    Total Mem Req
─────────            ────────    ─────────────    ─────────────
UI Interface         2           200m             256Mi
Backend API          3           750m             768Mi
Task Agent           2           1000m            1Gi
Notification Svc     2           200m             256Mi
PostgreSQL           2           1000m            2Gi
Redis                3           750m             1.5Gi
                                 ──────           ──────
TOTAL                            3900m            5.8Gi
                                 (~4 CPU cores)   (~6 GB RAM)

Minimum cluster size: 3 worker nodes × 2 CPU × 4Gi RAM = 6 CPU, 12Gi RAM
(headroom for system pods, kubelet, and burst capacity)
```

---

## 9. RBAC — Roles & RoleBindings

### 9.1 Service Accounts

Every service runs under its own ServiceAccount — never the `default` one. This enforces the **principle of least privilege**: each service can only access what it explicitly needs.

| ServiceAccount | Used By | Purpose |
|---------------|---------|---------|
| `sa-ui` | UI Interface | Minimal permissions — frontend needs no K8s API access |
| `sa-backend-api` | Backend API | Can read ConfigMaps and Secrets in its namespace |
| `sa-task-agent` | Task Agent | Can read its own ConfigMap and Secret. Can read Pod status for health reporting. |
| `sa-notification` | Notification Service | Can read its own ConfigMap and Secret only |
| `sa-postgres` | PostgreSQL | Can manage its own PersistentVolumeClaims |
| `sa-redis` | Redis | Can manage its own PersistentVolumeClaims |

### 9.2 Roles

**Role: `config-reader`** — Read access to ConfigMaps and Secrets
```
Namespace: ai-task-manager
Rules:
  - apiGroups: [""]
    resources: ["configmaps", "secrets"]
    verbs: ["get", "list", "watch"]
```

**Role: `pvc-manager`** — Manage persistent storage (for database StatefulSets)
```
Namespace: ai-task-manager
Rules:
  - apiGroups: [""]
    resources: ["persistentvolumeclaims"]
    verbs: ["get", "list", "watch", "create", "update", "patch"]
```

**Role: `pod-reader`** — Read-only access to Pod status (for health monitoring)
```
Namespace: ai-task-manager
Rules:
  - apiGroups: [""]
    resources: ["pods", "pods/log"]
    verbs: ["get", "list", "watch"]
```

**Role: `minimal-access`** — No K8s API access (for services that don't need it)
```
Namespace: ai-task-manager
Rules: []    ← Empty. No permissions at all.
```

### 9.3 RoleBindings

| RoleBinding | ServiceAccount | Role | What It Allows |
|------------|----------------|------|---------------|
| `backend-api-config-binding` | `sa-backend-api` | `config-reader` | Backend can read ConfigMaps and Secrets to load its configuration |
| `task-agent-config-binding` | `sa-task-agent` | `config-reader` | Task Agent can read its own config and LLM API key |
| `task-agent-pod-binding` | `sa-task-agent` | `pod-reader` | Task Agent can report its own Pod status for monitoring |
| `notification-config-binding` | `sa-notification` | `config-reader` | Notification Service can read SMTP credentials |
| `postgres-storage-binding` | `sa-postgres` | `pvc-manager` | PostgreSQL can create and manage its persistent volumes |
| `redis-storage-binding` | `sa-redis` | `pvc-manager` | Redis can create and manage its persistent volumes |
| `ui-minimal-binding` | `sa-ui` | `minimal-access` | UI has NO K8s API access — it only serves static files |

### 9.4 Human Access Roles (for DevOps team)

| ClusterRole | Who | Permissions |
|------------|-----|-------------|
| `task-manager-admin` | DevOps Lead | Full access to `ai-task-manager` namespace — deploy, scale, manage secrets |
| `task-manager-developer` | Developers | Read Pods, logs, ConfigMaps. No access to Secrets. Can port-forward for debugging. |
| `task-manager-viewer` | Stakeholders | Read-only access to Pods, Services, Deployments — can view status but change nothing |

```
Access Matrix:

                    Admin    Developer    Viewer
                    ─────    ─────────    ──────
View Pods           ✓        ✓            ✓
View Logs           ✓        ✓            ✗
Read ConfigMaps     ✓        ✓            ✗
Read Secrets        ✓        ✗            ✗
Edit Deployments    ✓        ✗            ✗
Scale Replicas      ✓        ✗            ✗
Delete Resources    ✓        ✗            ✗
Port Forward        ✓        ✓            ✗
Exec into Pod       ✓        ✗            ✗
```

---

## 10. Inter-Service Communication

### Communication Map

```
┌─────────────────────────────────────────────────────────────────────┐
│                   COMMUNICATION PATTERNS                             │
│                                                                      │
│   ┌──────┐  HTTPS (external)    ┌──────────────┐                   │
│   │ User │ ────────────────────► │   Ingress     │                   │
│   └──────┘                       └──────┬───────┘                   │
│                                         │ HTTP (internal, plain)     │
│                              ┌──────────┴──────────┐                │
│                              ▼                     ▼                │
│                        ┌──────────┐          ┌──────────┐          │
│                        │    UI    │          │ Backend  │           │
│                        │         │          │   API    │           │
│                        └──────────┘          └────┬─────┘          │
│                                                   │                 │
│                    ┌──────────────┬────────────────┼──────────┐     │
│                    │              │                │          │     │
│                    ▼              ▼                ▼          ▼     │
│              ┌──────────┐  ┌──────────┐    ┌─────────┐ ┌────────┐ │
│              │   Task   │  │  Redis   │    │Postgres │ │Postgres│ │
│              │  Agent   │  │ (Queue + │    │(Primary)│ │(Read)  │ │
│              └─────┬────┘  │  PubSub) │    └─────────┘ └────────┘ │
│                    │       └────┬─────┘                            │
│                    │            │                                   │
│                    │            ▼                                   │
│                    │      ┌──────────────┐                         │
│                    │      │ Notification │                          │
│                    │      │   Service    │                          │
│                    │      └──────┬───────┘                         │
│                    │             │                                  │
│                    │             ▼                                  │
│                    │      External Services                        │
│                    │      (SMTP, FCM)                              │
│                    │                                               │
│                    └──► LLM API (external)                         │
└─────────────────────────────────────────────────────────────────────┘
```

### Communication Protocols & DNS Resolution

| From | To | Protocol | DNS Name | Pattern |
|------|-----|----------|----------|---------|
| Ingress → UI | HTTP | `ui-svc.ai-task-manager.svc.cluster.local:3000` | Synchronous request-response |
| Ingress → Backend API | HTTP | `backend-api-svc.ai-task-manager.svc.cluster.local:8080` | Synchronous request-response |
| Backend API → PostgreSQL (write) | TCP/PostgreSQL | `postgres-svc:5432` | Synchronous query |
| Backend API → PostgreSQL (read) | TCP/PostgreSQL | `postgres-readonly-svc:5432` | Synchronous query (read replica) |
| Backend API → Redis | TCP/Redis | `redis-svc:6379` | Synchronous (cache) + async (queue publish) |
| Backend API → Task Agent | Redis Queue | `redis-svc:6379` queue: `ai-task-processing` | **Asynchronous** — Backend pushes job to queue, Agent pulls and processes |
| Task Agent → Backend API | HTTP callback | `backend-api-svc:8080/internal/task-result` | Async — Agent POSTs result when AI processing is done |
| Task Agent → LLM Provider | HTTPS | External API (e.g., `api.openai.com`) | Synchronous with 30s timeout |
| Backend API → Redis (events) | Redis Pub/Sub | `redis-svc:6379` channel: `task-events` | Publish event: "task.created", "task.updated" |
| Redis → Notification Svc | Redis Pub/Sub | Subscription on `task-events` | **Asynchronous** — Notification Service reacts to events |
| Notification Svc → SMTP | SMTP/TLS | External: `smtp.sendgrid.net:587` | Synchronous send with retry |
| Notification Svc → FCM | HTTPS | External: `fcm.googleapis.com` | Synchronous push with retry |

### Why This Communication Design?

```
Synchronous (HTTP/gRPC):
  ✓ User-facing requests that need immediate response
  ✓ Database queries
  ✗ NOT for long-running AI processing (would timeout)

Asynchronous (Redis Queue):
  ✓ AI task processing (takes 5-30 seconds)
  ✓ Decouples Backend from Task Agent — if Agent is slow, Backend still responds fast
  ✓ Built-in retry — failed jobs stay in queue

Pub/Sub (Redis Channels):
  ✓ Event broadcasting — one event, multiple consumers
  ✓ Notification Service doesn't need to poll Backend
  ✓ Easily add new subscribers later (e.g., analytics service)
```

---

## 11. Scaling Strategy

### Horizontal Pod Autoscaler (HPA) Configuration

| Service | Min Replicas | Max Replicas | Scale-Up Trigger | Scale-Down Trigger |
|---------|-------------|-------------|-----------------|-------------------|
| UI Interface | 2 | 6 | CPU > 70% | CPU < 30% for 5 min |
| Backend API | 3 | 10 | CPU > 70% OR requests/sec > 500 | CPU < 30% for 5 min |
| Task Agent | 2 | 8 | Redis queue depth > 50 pending tasks | Queue depth < 5 for 10 min |
| Notification Svc | 2 | 5 | CPU > 70% | CPU < 30% for 5 min |

**PostgreSQL and Redis are NOT auto-scaled** — database scaling requires careful planning (adding replicas involves data replication setup, not just launching Pods).

```
Traffic Pattern Example:

Normal:    [API×3]  [Agent×2]  [Notif×2]
              │
   9 AM: Users flood in
              │
Peak:      [API×7]  [Agent×5]  [Notif×3]    ← HPA scaled up
              │
   11 PM: Traffic drops
              │
Night:     [API×3]  [Agent×2]  [Notif×2]    ← HPA scaled down (saves cost)
```

---

## 12. Health Checks

| Service | Liveness Probe | Readiness Probe | Startup Probe |
|---------|---------------|-----------------|---------------|
| **UI** | `GET /healthz` every 15s, fail after 3 | `GET /` every 10s, fail after 3 | `GET /` every 5s, fail after 30 (allow build time) |
| **Backend API** | `GET /health/live` every 15s, fail after 3 | `GET /health/ready` every 10s, fail after 3 (checks DB + Redis connectivity) | `GET /health/live` every 5s, fail after 60 (allow DB migration) |
| **Task Agent** | `GET /healthz` every 20s, fail after 3 | `GET /readyz` every 15s, fail after 3 (checks Redis queue connection) | `GET /healthz` every 5s, fail after 30 |
| **Notification Svc** | `GET /healthz` every 15s, fail after 3 | `GET /readyz` every 10s, fail after 3 (checks Redis + SMTP connectivity) | `GET /healthz` every 5s, fail after 30 |
| **PostgreSQL** | `exec: pg_isready` every 15s, fail after 3 | `exec: pg_isready` every 10s, fail after 3 | `exec: pg_isready` every 5s, fail after 60 (allow recovery) |
| **Redis** | `exec: redis-cli ping` every 15s, fail after 3 | `exec: redis-cli ping` every 10s, fail after 3 | `exec: redis-cli ping` every 5s, fail after 30 |

**Why three different probes?**
```
Startup Probe:    "Has the container finished starting?"
                  (Prevents liveness probe from killing a slow-starting app)

Liveness Probe:   "Is the container still alive and not stuck?"
                  (If fails → K8s RESTARTS the container)

Readiness Probe:  "Is the container ready to receive traffic?"
                  (If fails → K8s REMOVES from Service, stops sending traffic)
```

---

## 13. Storage Design

### Persistent Volume Claims

| PVC Name | Used By | Size | Access Mode | Storage Class | Reasoning |
|----------|---------|------|-------------|---------------|-----------|
| `postgres-data-postgres-0` | PostgreSQL primary | 20Gi | ReadWriteOnce | `ssd-retain` | SSD for IOPS. Retain policy — data survives PVC deletion (safety net). |
| `postgres-data-postgres-1` | PostgreSQL replica | 20Gi | ReadWriteOnce | `ssd-retain` | Same size as primary for seamless failover promotion. |
| `redis-data-redis-{0,1,2}` | Redis instances | 5Gi each | ReadWriteOnce | `ssd-retain` | Smaller — Redis data is mostly transient. Persist for queue durability. |

**Storage Class: `ssd-retain`**
```
Provisioner:        cloud-provider SSD (e.g., gp3 on AWS, pd-ssd on GCP)
Reclaim Policy:     Retain (do NOT auto-delete when PVC is removed)
Volume Binding:     WaitForFirstConsumer (provision in same zone as Pod)
```

**Total storage: 20Gi + 20Gi + 15Gi = 55Gi**

---

## 14. Security Hardening

### Network Policies

```
┌──────────────────────────────────────────────────────────┐
│              NETWORK POLICY RULES                         │
│                                                           │
│  Default: DENY ALL ingress to every Pod                  │
│           (then explicitly allow what's needed)          │
│                                                           │
│  ┌─────────────────────────────────────────────────┐     │
│  │ postgres-svc:                                    │     │
│  │   Allow ingress FROM: backend-api Pods ONLY      │     │
│  │   Port: 5432                                     │     │
│  │   (Task Agent and Notification CANNOT reach DB)  │     │
│  └─────────────────────────────────────────────────┘     │
│                                                           │
│  ┌─────────────────────────────────────────────────┐     │
│  │ redis-svc:                                       │     │
│  │   Allow ingress FROM: backend-api,               │     │
│  │                        task-agent,                │     │
│  │                        notification-svc           │     │
│  │   Port: 6379                                     │     │
│  │   (UI CANNOT reach Redis)                        │     │
│  └─────────────────────────────────────────────────┘     │
│                                                           │
│  ┌─────────────────────────────────────────────────┐     │
│  │ backend-api-svc:                                 │     │
│  │   Allow ingress FROM: ingress-controller,        │     │
│  │                        task-agent (callbacks)     │     │
│  │   Port: 8080                                     │     │
│  └─────────────────────────────────────────────────┘     │
│                                                           │
│  ┌─────────────────────────────────────────────────┐     │
│  │ ui-svc:                                          │     │
│  │   Allow ingress FROM: ingress-controller ONLY    │     │
│  │   Port: 3000                                     │     │
│  └─────────────────────────────────────────────────┘     │
└──────────────────────────────────────────────────────────┘
```

### Pod Security Standards

| Security Measure | Applied To | Setting |
|-----------------|-----------|---------|
| Run as non-root | All Pods | `runAsNonRoot: true` |
| Read-only filesystem | UI, Backend API, Task Agent, Notification | `readOnlyRootFilesystem: true` |
| Drop all capabilities | All Pods | `drop: ["ALL"]` |
| No privilege escalation | All Pods | `allowPrivilegeEscalation: false` |
| Seccomp profile | All Pods | `RuntimeDefault` |
| `automountServiceAccountToken` | UI (doesn't need K8s API) | `false` |

---

## 15. Deployment Strategy

### Rollout Order

When deploying the full stack for the first time or performing major updates:

```
Phase 1 (Infrastructure):
  PostgreSQL StatefulSet ──► Wait for primary healthy
  Redis StatefulSet ──► Wait for master + sentinels healthy

Phase 2 (Core Services):
  Backend API Deployment ──► Wait for readiness (DB migration runs on startup)

Phase 3 (Workers):
  Task Agent Deployment ──► Connects to Redis queue
  Notification Service Deployment ──► Subscribes to Redis events

Phase 4 (Frontend):
  UI Interface Deployment ──► Serves application

Phase 5 (Routing):
  Ingress rules applied ──► Traffic flows
```

### Update Strategy

| Service | Strategy | Details |
|---------|----------|---------|
| UI | RollingUpdate | maxSurge: 1, maxUnavailable: 0. Users never see downtime. |
| Backend API | RollingUpdate | maxSurge: 1, maxUnavailable: 1. With 3 replicas, 2 always serve traffic. |
| Task Agent | RollingUpdate | maxSurge: 1, maxUnavailable: 0. In-progress tasks complete before old Pod terminates (graceful shutdown with `preStop` hook). |
| Notification Svc | RollingUpdate | maxSurge: 1, maxUnavailable: 1. Brief notification delay is acceptable. |
| PostgreSQL | OnDelete | Manual, controlled updates. Never auto-restart a database. |
| Redis | OnDelete | Manual, controlled updates. Sentinel handles failover during update. |

### Rollback Plan

```
If deployment fails:
  kubectl rollout undo deployment/<name> -n ai-task-manager

Automatic rollback trigger:
  - Readiness probe fails for 3+ minutes after deploy
  - Error rate exceeds 5% (monitored by external alerting)
  - progressDeadlineSeconds: 300 (5 min — if deploy doesn't complete, it's marked failed)
```

---

## Summary

```
┌──────────────────────────────────────────────────────────┐
│         AI NATIVE TASK MANAGER — K8s SUMMARY              │
│                                                           │
│  Namespace:     ai-task-manager                          │
│  Deployments:   4 (UI, API, Agent, Notification)         │
│  StatefulSets:  2 (PostgreSQL, Redis)                    │
│  Services:      7 (all ClusterIP, exposed via Ingress)   │
│  ConfigMaps:    4 (one per application service)          │
│  Secrets:       7 (managed by External Secrets Operator) │
│  ServiceAccts:  6 (one per workload, least privilege)    │
│  Roles:         3 + 3 human roles                        │
│  PVCs:          5 (2 PostgreSQL + 3 Redis)               │
│  HPAs:          4 (auto-scale all application services)  │
│  Network Policies: Default deny + explicit allow rules   │
│                                                           │
│  Total Resources: ~4 CPU cores, ~6 GB RAM minimum       │
│  Total Storage:   55 Gi persistent SSD                    │
│  External Dependencies: LLM API, SMTP, FCM               │
└──────────────────────────────────────────────────────────┘
```
