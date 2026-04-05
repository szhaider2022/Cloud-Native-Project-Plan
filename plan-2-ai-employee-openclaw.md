# Plan 2: AI Employee (OpenClaw) — Kubernetes Deployment Plan

## Table of Contents

1. [Application Overview](#1-application-overview)
2. [Architecture Diagram](#2-architecture-diagram)
3. [Threat Model & Security Philosophy](#3-threat-model--security-philosophy)
4. [Namespace Design](#4-namespace-design)
5. [Pods, Deployments & StatefulSets](#5-pods-deployments--statefulsets)
6. [Services & Networking](#6-services--networking)
7. [ConfigMaps](#7-configmaps)
8. [Secrets Management & Expiry Handling](#8-secrets-management--expiry-handling)
9. [Resource Requests & Limits](#9-resource-requests--limits)
10. [RBAC — Roles & RoleBindings](#10-rbac--roles--rolebindings)
11. [Inter-Service Communication](#11-inter-service-communication)
12. [Sandboxed Tool Execution](#12-sandboxed-tool-execution)
13. [Audit Logging & Observability](#13-audit-logging--observability)
14. [Scaling Strategy](#14-scaling-strategy)
15. [Health Checks](#15-health-checks)
16. [Storage Design](#16-storage-design)
17. [Network Policies (Zero-Trust)](#17-network-policies-zero-trust)
18. [Deployment Strategy](#18-deployment-strategy)

---

## 1. Application Overview

An **AI Employee** is a personal autonomous agent that can perform real-world actions on your behalf — sending emails, reading documents, writing code, scheduling meetings, querying databases. Unlike a chatbot that only answers questions, an AI Employee **takes actions**. This makes it powerful but also **dangerous if not secured properly**.

### Core Services

| Service | Role | Nature | Risk Level |
|---------|------|--------|------------|
| **API Gateway** | Single entry point for all user interactions — authentication, request validation, rate limiting, audit logging | Stateless | HIGH — front door to the entire system |
| **Agent Orchestrator** | The "brain" — receives user instructions, breaks them into steps, decides which tools to call, manages multi-step reasoning chains | Stateless | CRITICAL — controls all autonomous actions |
| **LLM Service** | Manages LLM inference — prompt construction, token management, model routing, response parsing | Stateless | HIGH — handles sensitive prompts containing user data |
| **Tool Executor** | Sandboxed execution environment — runs tool actions (send email, read file, call API) in isolated containers | Stateless (ephemeral) | CRITICAL — directly interacts with external systems |
| **Memory Service** | Long-term agent memory — stores conversation history, learned preferences, task context using vector embeddings | Stateful | MEDIUM — contains historical user data |
| **Auth & Policy Service** | Centralized authorization — validates every tool action against user-defined policies before execution | Stateless | CRITICAL — the security gatekeeper |

### Supporting Infrastructure

| Component | Role | Nature | Risk Level |
|-----------|------|--------|------------|
| **PostgreSQL** | Primary database — user profiles, agent configurations, action audit logs, policy definitions | Stateful | CRITICAL — contains all persistent state |
| **Redis** | Task queue + session cache — queues tool execution jobs, caches active sessions and rate limit counters | Stateful | HIGH — if compromised, attacker can inject tool actions |
| **Qdrant (Vector DB)** | Vector similarity search — stores and retrieves agent memory embeddings for context-aware reasoning | Stateful | MEDIUM — contains semantic user history |

---

## 2. Architecture Diagram

```
                            ┌──────────────────────────┐
                            │       INTERNET            │
                            └────────────┬─────────────┘
                                         │
                                         ▼
                            ┌──────────────────────────┐
                            │    Ingress Controller     │
                            │    TLS Termination        │
                            │    Rate Limiting          │
                            │    WAF Rules              │
                            └────────────┬─────────────┘
                                         │
                                         ▼
                  ┌──────────────────────────────────────────────┐
                  │              API GATEWAY                      │
                  │                                               │
                  │  • JWT Authentication                        │
                  │  • Request Validation                        │
                  │  • Audit Log (every request)                 │
                  │  • Rate Limiting (per user)                  │
                  │  • Input Sanitization (prompt injection      │
                  │    defense)                                   │
                  └──────────────────────┬───────────────────────┘
                                         │
                                         ▼
                  ┌──────────────────────────────────────────────┐
                  │          AGENT ORCHESTRATOR                   │
                  │                                               │
                  │  • Receives user instruction                 │
                  │  • Plans multi-step execution                │
                  │  • Calls LLM for reasoning                  │
                  │  • Requests tool execution                   │
                  │  • Assembles final response                  │
                  └───┬─────────────┬──────────────┬────────────┘
                      │             │              │
           ┌──────────┘    ┌───────┘      ┌───────┘
           ▼               ▼              ▼
  ┌─────────────┐  ┌──────────────┐  ┌──────────────────────────────┐
  │ LLM Service │  │Memory Service│  │    AUTH & POLICY SERVICE      │
  │             │  │             │  │                               │
  │ • Prompt    │  │ • Store     │  │  "Can this agent send email  │
  │   building  │  │   memory    │  │   to external addresses?"    │
  │ • Model     │  │ • Retrieve  │  │                               │
  │   routing   │  │   context   │  │  • Check user policy         │
  │ • Token     │  │ • Vector    │  │  • Validate tool parameters  │
  │   counting  │  │   search    │  │  • Log decision (allow/deny) │
  └──────┬──────┘  └──────┬──────┘  └──────────────┬───────────────┘
         │                │                         │
         │                │                         │ APPROVED
         │                │                         ▼
         │                │         ┌──────────────────────────────┐
         │                │         │       TOOL EXECUTOR           │
         │                │         │       (Sandboxed)             │
         │                │         │                               │
         │                │         │  ┌─────────┐  ┌───────────┐ │
         │                │         │  │ Email   │  │ Calendar  │ │
         │                │         │  │ Tool    │  │ Tool      │ │
         │                │         │  └─────────┘  └───────────┘ │
         │                │         │  ┌─────────┐  ┌───────────┐ │
         │                │         │  │ Code    │  │ Database  │ │
         │                │         │  │ Runner  │  │ Query     │ │
         │                │         │  └─────────┘  └───────────┘ │
         │                │         │  ┌─────────┐  ┌───────────┐ │
         │                │         │  │ Web     │  │ File      │ │
         │                │         │  │ Search  │  │ Manager   │ │
         │                │         │  └─────────┘  └───────────┘ │
         │                │         └──────────────────────────────┘
         │                │
         ▼                ▼
  ┌──────────────────────────────────────────────────┐
  │              DATA LAYER                           │
  │                                                   │
  │  ┌────────────┐  ┌──────────┐  ┌──────────────┐ │
  │  │ PostgreSQL │  │  Redis   │  │   Qdrant     │ │
  │  │ (Primary + │  │ (Queue + │  │  (Vector DB) │ │
  │  │  Replica)  │  │  Cache)  │  │              │ │
  │  └────────────┘  └──────────┘  └──────────────┘ │
  └──────────────────────────────────────────────────┘
```

### Action Execution Flow (Critical Path)

```
User: "Send an email to john@company.com summarizing today's meeting notes"
  │
  ▼
┌─ API Gateway ─────────────────────────────────────────────────────────┐
│  1. Authenticate user (JWT)                                           │
│  2. Log: "user_123 submitted instruction at 2024-03-28T14:30:00Z"   │
│  3. Sanitize input (strip injection attempts)                        │
│  4. Forward to Orchestrator                                          │
└───────────────────────────────────────┬───────────────────────────────┘
                                        │
                                        ▼
┌─ Agent Orchestrator ──────────────────────────────────────────────────┐
│  5. Call LLM: "Break this instruction into steps"                    │
│  6. LLM returns plan:                                                │
│     Step A: Retrieve today's meeting notes (memory-search)           │
│     Step B: Summarize notes (llm-generate)                           │
│     Step C: Send email to john@company.com (email-send)              │
│  7. Execute Step A → Memory Service → returns meeting context        │
│  8. Execute Step B → LLM Service → returns summary text              │
│  9. Execute Step C → needs AUTHORIZATION first                       │
└───────────────────────────────────────┬───────────────────────────────┘
                                        │
                                        ▼
┌─ Auth & Policy Service ───────────────────────────────────────────────┐
│  10. Check policy for user_123:                                       │
│      Rule: "email-send" → allowed to: @company.com domains          │
│      Target: john@company.com → MATCHES @company.com → ✅ APPROVED   │
│  11. Log: "ALLOW email-send to john@company.com by user_123"        │
│                                                                       │
│  (If target was hacker@evil.com → ❌ DENIED + alert triggered)       │
└───────────────────────────────────────┬───────────────────────────────┘
                                        │
                                        ▼
┌─ Tool Executor (Sandboxed) ───────────────────────────────────────────┐
│  12. Spawn ephemeral container for email-send tool                    │
│  13. Inject SMTP credentials (mounted read-only, scoped to this job) │
│  14. Execute: send email with summary to john@company.com            │
│  15. Result: "Email sent successfully, ID: msg_abc123"               │
│  16. Container destroyed (sandbox cleaned up)                        │
│  17. Log: "EXECUTED email-send, result: success, msg_id: msg_abc123" │
└───────────────────────────────────────┬───────────────────────────────┘
                                        │
                                        ▼
┌─ Agent Orchestrator ──────────────────────────────────────────────────┐
│  18. All steps complete                                               │
│  19. Store interaction in Memory Service (for future context)        │
│  20. Return to user: "Done! I sent john@company.com a summary of     │
│      today's meeting notes."                                         │
└───────────────────────────────────────────────────────────────────────┘
```

---

## 3. Threat Model & Security Philosophy

**Why security is paramount:** Unlike Plan 1 (a task manager that reads/writes its own database), this AI Employee **acts on behalf of the user in the real world**. A compromised agent can send emails, delete files, execute code, and access sensitive data — all autonomously.

### Threat Matrix

| Threat | Impact | Mitigation |
|--------|--------|------------|
| **Prompt Injection** | Attacker injects instructions into data the agent reads, hijacking its behavior | Input sanitization at API Gateway + LLM output validation + tool parameter whitelisting at Auth Service |
| **Tool Abuse** | Agent is tricked into performing unauthorized actions (e.g., sending data to attacker's email) | Auth & Policy Service gates EVERY tool action. Explicit allow-lists per tool. Deny by default. |
| **Secret Exfiltration** | Attacker extracts API keys through prompt manipulation ("print your environment variables") | Secrets mounted read-only in isolated tool containers. Agent never sees raw credentials. LLM prompts are scrubbed. |
| **Lateral Movement** | Compromised Pod accesses other services | Zero-trust network policies. Each service only reaches what it explicitly needs. Default deny all. |
| **Data Leakage** | Agent memory (vector DB) contains sensitive conversation history | Memory Service encrypted at rest. Access only via Agent Orchestrator. User-scoped isolation in Qdrant (collection per user). |
| **Denial of Service** | Attacker floods agent with expensive LLM calls | Per-user rate limits at API Gateway. Token budgets per request. Queue depth limits on Redis. |
| **Replay Attack** | Attacker replays a captured valid request to re-execute an action | Request IDs with TTL. Idempotency keys on all tool executions. JWT tokens with short expiry (15 min). |

### Security Principles Applied

```
┌────────────────────────────────────────────────────────────────┐
│                                                                 │
│  1. ZERO TRUST        Nothing is trusted by default.           │
│                        Every request is authenticated.          │
│                        Every tool action is authorized.         │
│                                                                 │
│  2. LEAST PRIVILEGE    Each service can ONLY access what it    │
│                        needs. Tool Executor cannot reach the    │
│                        database. LLM Service cannot send        │
│                        emails.                                  │
│                                                                 │
│  3. DEFENSE IN DEPTH   Multiple layers of security:            │
│                        Ingress WAF → API Gateway auth →        │
│                        Policy check → Sandboxed execution      │
│                                                                 │
│  4. AUDIT EVERYTHING   Every action logged with:               │
│                        who, what, when, why, result             │
│                        Immutable audit trail in PostgreSQL      │
│                                                                 │
│  5. FAIL CLOSED        If Auth Service is down → ALL tool      │
│                        executions BLOCKED (not allowed).        │
│                        Safety over availability.                │
│                                                                 │
│  6. BLAST RADIUS       Tool Executor runs in ephemeral,        │
│     CONTAINMENT        sandboxed containers. If compromised,   │
│                        the container is destroyed after the     │
│                        single action. No persistent state.      │
│                                                                 │
└────────────────────────────────────────────────────────────────┘
```

---

## 4. Namespace Design

Unlike Plan 1 (single namespace), this deployment uses **multiple namespaces** to enforce hard isolation boundaries between components with different trust levels.

```
┌──────────────────────────────────────────────────────────────────┐
│                         CLUSTER                                   │
│                                                                   │
│  ┌──────────────────────────────────────────┐                    │
│  │  Namespace: openclaw-gateway              │                    │
│  │                                           │                    │
│  │  API Gateway (the only internet-facing    │                    │
│  │  component — isolated from internal       │                    │
│  │  services for blast radius containment)   │                    │
│  └──────────────────────────────────────────┘                    │
│                                                                   │
│  ┌──────────────────────────────────────────┐                    │
│  │  Namespace: openclaw-core                 │                    │
│  │                                           │                    │
│  │  Agent Orchestrator, LLM Service,         │                    │
│  │  Auth & Policy Service, Memory Service    │                    │
│  │  (trusted internal services — no direct   │                    │
│  │  internet access)                         │                    │
│  └──────────────────────────────────────────┘                    │
│                                                                   │
│  ┌──────────────────────────────────────────┐                    │
│  │  Namespace: openclaw-tools                │                    │
│  │                                           │                    │
│  │  Tool Executor + ephemeral tool Pods      │                    │
│  │  (HIGHEST RISK — sandboxed, network       │                    │
│  │  restricted, resource-capped)             │                    │
│  └──────────────────────────────────────────┘                    │
│                                                                   │
│  ┌──────────────────────────────────────────┐                    │
│  │  Namespace: openclaw-data                 │                    │
│  │                                           │                    │
│  │  PostgreSQL, Redis, Qdrant                │                    │
│  │  (data layer — only accessible by         │                    │
│  │  openclaw-core services)                  │                    │
│  └──────────────────────────────────────────┘                    │
│                                                                   │
│  ┌──────────────────────────────────────────┐                    │
│  │  Namespace: openclaw-audit                │                    │
│  │                                           │                    │
│  │  Audit Log Collector, Log Storage         │                    │
│  │  (write-only from other namespaces —      │                    │
│  │  no service can DELETE audit logs)        │                    │
│  └──────────────────────────────────────────┘                    │
└──────────────────────────────────────────────────────────────────┘
```

**Why multiple namespaces (vs. single namespace in Plan 1)?**

| Reason | Explanation |
|--------|-------------|
| **Blast radius** | If Tool Executor is compromised, it cannot reach the data layer (different namespace + network policy) |
| **RBAC granularity** | Roles are namespace-scoped. Tool Executor's ServiceAccount has zero permissions in `openclaw-data`. |
| **Resource isolation** | Each namespace has its own quota. A runaway tool can't starve the database of CPU. |
| **Network policy** | Default-deny per namespace. Cross-namespace traffic requires explicit allow rules. |
| **Compliance** | Audit namespace is append-only. Even cluster admins cannot delete audit logs without triggering alerts. |

### Resource Quotas Per Namespace

| Namespace | CPU Request | Memory Request | Max Pods | Max PVCs |
|-----------|------------|---------------|----------|----------|
| `openclaw-gateway` | 2 cores | 2Gi | 10 | 0 |
| `openclaw-core` | 8 cores | 16Gi | 30 | 0 |
| `openclaw-tools` | 4 cores | 8Gi | 50 (ephemeral Pods) | 0 |
| `openclaw-data` | 6 cores | 16Gi | 15 | 15 |
| `openclaw-audit` | 2 cores | 4Gi | 5 | 5 |

---

## 5. Pods, Deployments & StatefulSets

### 5.1 API Gateway — Deployment

| Property | Value | Reasoning |
|----------|-------|-----------|
| **Kind** | Deployment | Stateless — authenticates and forwards requests. All session state is in Redis. |
| **Namespace** | `openclaw-gateway` | Isolated from internal services. Only component exposed to internet. |
| **Replicas** | 3 | High availability — this is the single entry point. If Gateway is down, entire system is down. 3 replicas across different nodes. |
| **Image** | `openclaw/api-gateway:1.0.0` | |
| **Container Port** | 8443 | HTTPS within cluster (mTLS between Gateway and Ingress) |
| **Strategy** | RollingUpdate (maxSurge: 1, maxUnavailable: 0) | Zero-downtime — users must never lose access |
| **Security Context** | Non-root, read-only filesystem, drop all capabilities | Gateway is internet-facing — maximum hardening |

### 5.2 Agent Orchestrator — Deployment

| Property | Value | Reasoning |
|----------|-------|-----------|
| **Kind** | Deployment | Stateless — orchestration state is held in-memory per-request (short-lived). Long-term state in PostgreSQL. |
| **Namespace** | `openclaw-core` | Internal-only. No direct internet access. |
| **Replicas** | 3 | Handles concurrent user sessions. Each replica can manage multiple agent chains simultaneously. |
| **Image** | `openclaw/agent-orchestrator:1.0.0` | |
| **Container Port** | 8080 | Internal HTTP |
| **Strategy** | RollingUpdate (maxSurge: 1, maxUnavailable: 1) | |
| **Termination Grace Period** | 120 seconds | Agent may be mid-chain — allow time to checkpoint state before termination |

### 5.3 LLM Service — Deployment

| Property | Value | Reasoning |
|----------|-------|-----------|
| **Kind** | Deployment | Stateless — constructs prompts, calls external LLM APIs, parses responses. No local state. |
| **Namespace** | `openclaw-core` | |
| **Replicas** | 2 | LLM calls are I/O bound (waiting for external API response). 2 replicas is sufficient; each handles many concurrent requests. |
| **Image** | `openclaw/llm-service:1.0.0` | |
| **Container Port** | 8081 | Internal HTTP |
| **Strategy** | RollingUpdate (maxSurge: 1, maxUnavailable: 0) | |

### 5.4 Auth & Policy Service — Deployment

| Property | Value | Reasoning |
|----------|-------|-----------|
| **Kind** | Deployment | Stateless — reads policies from PostgreSQL, evaluates in-memory. Policies are cached in Redis with short TTL. |
| **Namespace** | `openclaw-core` | |
| **Replicas** | 3 | **CRITICAL SERVICE** — if Auth is down, all tool executions are blocked (fail-closed). Must be highly available. 3 replicas with pod anti-affinity (spread across nodes). |
| **Image** | `openclaw/auth-policy-svc:1.0.0` | |
| **Container Port** | 8082 | Internal HTTP |
| **Strategy** | RollingUpdate (maxSurge: 1, maxUnavailable: 0) | NEVER have fewer than 3 healthy replicas during deploy |
| **Pod Anti-Affinity** | `requiredDuringSchedulingIgnoredDuringExecution` | Replicas MUST be on different nodes. If a node fails, Auth service survives. |

### 5.5 Memory Service — Deployment

| Property | Value | Reasoning |
|----------|-------|-----------|
| **Kind** | Deployment | Stateless — acts as an interface to Qdrant vector DB. Converts text to embeddings and queries. |
| **Namespace** | `openclaw-core` | |
| **Replicas** | 2 | Moderate traffic — called once per agent chain to retrieve context. |
| **Image** | `openclaw/memory-service:1.0.0` | |
| **Container Port** | 8083 | Internal HTTP |

### 5.6 Tool Executor — Deployment

| Property | Value | Reasoning |
|----------|-------|-----------|
| **Kind** | Deployment | Stateless — pulls approved jobs from Redis queue, spawns ephemeral tool containers, returns results. |
| **Namespace** | `openclaw-tools` | **Isolated namespace** — highest-risk component. Even if compromised, cannot reach data layer. |
| **Replicas** | 2 | Each executor processes tool jobs concurrently (up to 3 parallel jobs per replica). |
| **Image** | `openclaw/tool-executor:1.0.0` | |
| **Container Port** | 9090 | Internal HTTP (health check only) |
| **Strategy** | RollingUpdate (maxSurge: 1, maxUnavailable: 0) | Never lose in-progress tool executions |
| **Security Context** | Non-root, read-only filesystem, no privilege escalation, seccomp `RuntimeDefault`, NO host network, NO host PID | Maximum sandboxing for a component that runs arbitrary tool logic |

### 5.7 PostgreSQL — StatefulSet

| Property | Value | Reasoning |
|----------|-------|-----------|
| **Kind** | StatefulSet | Stateful — stores user data, policies, and immutable audit logs. Persistent identity and storage required. |
| **Namespace** | `openclaw-data` | Data isolation. Only `openclaw-core` services can reach this namespace. |
| **Replicas** | 2 | Primary + 1 synchronous replica. **Synchronous** replication (not async) because audit logs must not be lost — if primary dies, replica has every record. |
| **Image** | `postgres:16-alpine` | |
| **Container Port** | 5432 | |
| **Volume** | 50Gi SSD per replica | Larger than Plan 1 — stores audit logs which grow continuously |
| **Encryption** | `pgcrypto` extension enabled | Sensitive columns (user data, policy definitions) encrypted at the application level |

### 5.8 Redis — StatefulSet

| Property | Value | Reasoning |
|----------|-------|-----------|
| **Kind** | StatefulSet | Stateful — Sentinel mode for automatic failover. Stable identities for replication. |
| **Namespace** | `openclaw-data` | |
| **Replicas** | 3 | 1 master + 2 replicas with Sentinel. Critical for tool execution queue reliability. |
| **Image** | `redis:7-alpine` | |
| **Container Port** | 6379, 26379 | |
| **Volume** | 10Gi SSD per replica | Larger than Plan 1 — tool execution queue may hold multi-step jobs with payloads |
| **Auth** | `requirepass` enabled + ACL rules | Separate Redis ACL users per service (Orchestrator gets read/write, Tool Executor gets queue-only access) |

### 5.9 Qdrant (Vector DB) — StatefulSet

| Property | Value | Reasoning |
|----------|-------|-----------|
| **Kind** | StatefulSet | Stateful — stores vector embeddings on disk. Must survive Pod restarts. |
| **Namespace** | `openclaw-data` | |
| **Replicas** | 2 | 1 primary + 1 replica for high availability. Vector search can tolerate brief inconsistency. |
| **Image** | `qdrant/qdrant:v1.8` | |
| **Container Port** | 6333 (HTTP), 6334 (gRPC) | |
| **Volume** | 30Gi SSD per replica | Vector embeddings are dense — 30Gi supports millions of memory entries |
| **Collection Strategy** | One collection per user | User isolation at the data level — user A's memory never appears in user B's search results |

---

## 6. Services & Networking

### Service Map

```
┌────────────────────────────────────────────────────────────────────────┐
│                                                                        │
│  NAMESPACE: openclaw-gateway                                          │
│  ┌──────────────────────────────────────────────────────────────┐     │
│  │  Ingress: openclaw-ingress                                    │     │
│  │  Rule: agent.example.com  ──► gateway-svc (ClusterIP:8443)  │     │
│  │  TLS: cert-manager + Let's Encrypt                           │     │
│  │  Annotations: WAF rules, rate-limit 30 req/min per IP       │     │
│  └──────────────────────────────────────────────────────────────┘     │
│                                                                        │
│  NAMESPACE: openclaw-core                                             │
│  ┌──────────────────────────┐  ┌────────────────────────────┐        │
│  │ orchestrator-svc          │  │ llm-svc                     │        │
│  │ ClusterIP:8080            │  │ ClusterIP:8081              │        │
│  └──────────────────────────┘  └────────────────────────────┘        │
│  ┌──────────────────────────┐  ┌────────────────────────────┐        │
│  │ auth-policy-svc           │  │ memory-svc                  │        │
│  │ ClusterIP:8082            │  │ ClusterIP:8083              │        │
│  └──────────────────────────┘  └────────────────────────────┘        │
│                                                                        │
│  NAMESPACE: openclaw-tools                                            │
│  ┌──────────────────────────┐                                         │
│  │ tool-executor-svc         │                                         │
│  │ ClusterIP:9090            │                                         │
│  └──────────────────────────┘                                         │
│                                                                        │
│  NAMESPACE: openclaw-data                                             │
│  ┌──────────────────────────┐  ┌────────────────────────────┐        │
│  │ postgres-svc              │  │ redis-svc                   │        │
│  │ ClusterIP:5432            │  │ ClusterIP:6379              │        │
│  └──────────────────────────┘  └────────────────────────────┘        │
│  ┌──────────────────────────┐                                         │
│  │ qdrant-svc                │                                         │
│  │ ClusterIP:6333            │                                         │
│  └──────────────────────────┘                                         │
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘
```

### Service Definitions

| Service | Namespace | Type | Port | Reasoning |
|---------|-----------|------|------|-----------|
| `gateway-svc` | openclaw-gateway | ClusterIP | 8443 | Accessed via Ingress. HTTPS within cluster (mTLS). |
| `orchestrator-svc` | openclaw-core | ClusterIP | 8080 | Internal only — called by Gateway. Never exposed externally. |
| `llm-svc` | openclaw-core | ClusterIP | 8081 | Internal only — called by Orchestrator. |
| `auth-policy-svc` | openclaw-core | ClusterIP | 8082 | Internal only — called by Orchestrator before tool execution. |
| `memory-svc` | openclaw-core | ClusterIP | 8083 | Internal only — called by Orchestrator for context retrieval. |
| `tool-executor-svc` | openclaw-tools | ClusterIP | 9090 | Internal only — receives approved tool jobs from Redis queue. Health check endpoint only. |
| `postgres-svc` | openclaw-data | ClusterIP | 5432 | Internal only — accessed by core services. Cross-namespace via FQDN. |
| `redis-svc` | openclaw-data | ClusterIP | 6379 | Internal only — queue and cache. Cross-namespace via FQDN. |
| `qdrant-svc` | openclaw-data | ClusterIP | 6333 | Internal only — accessed by Memory Service. |

**Cross-Namespace DNS Pattern:**
```
From openclaw-core to openclaw-data:
  postgres-svc.openclaw-data.svc.cluster.local:5432
  redis-svc.openclaw-data.svc.cluster.local:6379
  qdrant-svc.openclaw-data.svc.cluster.local:6333

From openclaw-gateway to openclaw-core:
  orchestrator-svc.openclaw-core.svc.cluster.local:8080
```

---

## 7. ConfigMaps

### 7.1 `gateway-config` (Namespace: openclaw-gateway)

| Key | Value | Purpose |
|-----|-------|---------|
| `ORCHESTRATOR_URL` | `http://orchestrator-svc.openclaw-core.svc.cluster.local:8080` | Cross-namespace routing to core service |
| `RATE_LIMIT_RPM` | `30` | Conservative limit — AI actions are expensive and irreversible |
| `MAX_REQUEST_BODY_SIZE` | `1MB` | Prevent oversized payloads |
| `JWT_ISSUER` | `https://agent.example.com` | JWT validation config |
| `JWT_EXPIRY_MINUTES` | `15` | Short-lived tokens — if stolen, limited window of abuse |
| `CORS_ORIGINS` | `https://agent.example.com` | Strict CORS |
| `AUDIT_LOG_ENDPOINT` | `http://audit-collector-svc.openclaw-audit.svc.cluster.local:9200` | Where to send audit events |
| `PROMPT_INJECTION_FILTER` | `enabled` | Activate prompt injection detection patterns |

### 7.2 `orchestrator-config` (Namespace: openclaw-core)

| Key | Value | Purpose |
|-----|-------|---------|
| `LLM_SERVICE_URL` | `http://llm-svc:8081` | Same namespace — short DNS |
| `AUTH_POLICY_URL` | `http://auth-policy-svc:8082` | Authorization check endpoint |
| `MEMORY_SERVICE_URL` | `http://memory-svc:8083` | Context retrieval endpoint |
| `REDIS_HOST` | `redis-svc.openclaw-data.svc.cluster.local` | Cross-namespace Redis |
| `REDIS_PORT` | `6379` | |
| `TOOL_QUEUE_NAME` | `approved-tool-jobs` | Redis queue for authorized tool executions |
| `RESULT_QUEUE_NAME` | `tool-results` | Redis queue for receiving tool execution results |
| `MAX_CHAIN_STEPS` | `10` | Safety limit — agent cannot plan more than 10 steps per request (prevents infinite loops) |
| `CHAIN_TIMEOUT_SECONDS` | `300` | 5-minute total timeout per user request |
| `DATABASE_HOST` | `postgres-svc.openclaw-data.svc.cluster.local` | |
| `DATABASE_PORT` | `5432` | |
| `DATABASE_NAME` | `openclaw` | |

### 7.3 `llm-config` (Namespace: openclaw-core)

| Key | Value | Purpose |
|-----|-------|---------|
| `PRIMARY_MODEL` | `claude-sonnet-4-6` | Default model for reasoning |
| `PLANNING_MODEL` | `claude-opus-4-6` | Complex multi-step planning uses more capable model |
| `SIMPLE_MODEL` | `claude-haiku-4-5-20251001` | Quick classification and parsing tasks |
| `MAX_TOKENS_PER_REQUEST` | `4096` | Cap output tokens to control cost |
| `MAX_CONTEXT_TOKENS` | `100000` | Limit context window usage |
| `TIMEOUT_MS` | `60000` | 60-second timeout for LLM API calls |
| `RETRY_COUNT` | `2` | Retry failed LLM calls twice |
| `TEMPERATURE` | `0.3` | Low temperature for consistent, predictable agent behavior |
| `PROMPT_TEMPLATE_DIR` | `/etc/openclaw/prompts` | Mounted from ConfigMap volume |

### 7.4 `auth-policy-config` (Namespace: openclaw-core)

| Key | Value | Purpose |
|-----|-------|---------|
| `DATABASE_HOST` | `postgres-svc.openclaw-data.svc.cluster.local` | Read policies from PostgreSQL |
| `DATABASE_PORT` | `5432` | |
| `REDIS_HOST` | `redis-svc.openclaw-data.svc.cluster.local` | Cache policies for fast evaluation |
| `POLICY_CACHE_TTL_SECONDS` | `60` | Re-read policies from DB every 60 seconds (balance between freshness and performance) |
| `DEFAULT_POLICY` | `deny` | **DENY by default** — tools are blocked unless explicitly allowed |
| `AUDIT_LOG_ENDPOINT` | `http://audit-collector-svc.openclaw-audit.svc.cluster.local:9200` | Log every allow/deny decision |
| `MAX_ACTIONS_PER_HOUR` | `50` | Global safety limit per user — even with valid policies |

### 7.5 `memory-config` (Namespace: openclaw-core)

| Key | Value | Purpose |
|-----|-------|---------|
| `QDRANT_HOST` | `qdrant-svc.openclaw-data.svc.cluster.local` | Vector DB connection |
| `QDRANT_PORT` | `6333` | |
| `EMBEDDING_MODEL` | `text-embedding-3-small` | Model for converting text to vectors |
| `EMBEDDING_DIMENSIONS` | `1536` | Vector size for the embedding model |
| `TOP_K_RESULTS` | `10` | Return top 10 most relevant memory entries |
| `SIMILARITY_THRESHOLD` | `0.75` | Minimum relevance score (0-1) to include in results |

### 7.6 `tool-executor-config` (Namespace: openclaw-tools)

| Key | Value | Purpose |
|-----|-------|---------|
| `REDIS_HOST` | `redis-svc.openclaw-data.svc.cluster.local` | Pull jobs from approved queue |
| `REDIS_PORT` | `6379` | |
| `JOB_QUEUE_NAME` | `approved-tool-jobs` | Listen for authorized tool execution jobs |
| `RESULT_QUEUE_NAME` | `tool-results` | Push execution results back |
| `EXECUTION_TIMEOUT_SECONDS` | `30` | Kill tool execution after 30 seconds (prevent hangs) |
| `MAX_CONCURRENT_JOBS` | `3` | Limit parallel tool executions per Pod |
| `SANDBOX_CPU_LIMIT` | `200m` | Each tool container gets max 200m CPU |
| `SANDBOX_MEMORY_LIMIT` | `128Mi` | Each tool container gets max 128Mi RAM |
| `ALLOWED_EGRESS_DOMAINS` | `smtp.sendgrid.net,www.googleapis.com,api.openai.com` | Whitelist of external domains tool containers can reach |

---

## 8. Secrets Management & Expiry Handling

### 8.1 Secret Definitions

| Secret Name | Namespace | Keys | Used By | Sensitivity |
|------------|-----------|------|---------|-------------|
| `jwt-signing-keys` | openclaw-gateway | `JWT_PRIVATE_KEY`, `JWT_PUBLIC_KEY` | API Gateway | CRITICAL — if leaked, attacker can forge valid auth tokens for any user |
| `db-credentials` | openclaw-core | `DB_USER`, `DB_PASSWORD` | Orchestrator, Auth Service | CRITICAL — full database access |
| `db-credentials-readonly` | openclaw-core | `DB_RO_USER`, `DB_RO_PASSWORD` | Memory Service | HIGH — read-only DB access |
| `redis-credentials` | openclaw-core, openclaw-tools | `REDIS_PASSWORD` | Orchestrator, Auth Service, Tool Executor | HIGH — queue access means ability to inject tool jobs |
| `llm-api-keys` | openclaw-core | `ANTHROPIC_API_KEY`, `OPENAI_API_KEY` | LLM Service | CRITICAL — costly if leaked (thousands in API charges) |
| `qdrant-api-key` | openclaw-core | `QDRANT_API_KEY` | Memory Service | MEDIUM — access to vector memory data |
| `smtp-credentials` | openclaw-tools | `SMTP_USER`, `SMTP_PASSWORD` | Tool Executor (email tool) | HIGH — ability to send emails as the system |
| `google-service-account` | openclaw-tools | `GOOGLE_SA_KEY_JSON` | Tool Executor (calendar, drive tools) | HIGH — access to Google Workspace |
| `encryption-master-key` | openclaw-data | `MASTER_ENCRYPTION_KEY` | PostgreSQL (pgcrypto) | CRITICAL — decrypts all encrypted database columns |
| `audit-write-token` | openclaw-audit | `WRITE_TOKEN` | All services (write-only) | MEDIUM — append-only access to audit logs |

### 8.2 Secret Isolation by Namespace

```
┌──────────────────────────────────────────────────────────────┐
│              SECRET ISOLATION MODEL                            │
│                                                               │
│  openclaw-gateway:                                           │
│  └── jwt-signing-keys       (Gateway ONLY — no other         │
│                               service can read JWT keys)     │
│                                                               │
│  openclaw-core:                                              │
│  ├── db-credentials          (Orchestrator, Auth only)       │
│  ├── db-credentials-readonly (Memory Service only)           │
│  ├── redis-credentials       (Orchestrator, Auth only)       │
│  ├── llm-api-keys            (LLM Service ONLY)             │
│  └── qdrant-api-key          (Memory Service ONLY)           │
│                                                               │
│  openclaw-tools:                                             │
│  ├── redis-credentials       (Tool Executor queue access)    │
│  ├── smtp-credentials        (mounted ONLY during email      │
│  │                            tool execution, read-only)     │
│  └── google-service-account  (mounted ONLY during Google     │
│                                tool execution, read-only)    │
│                                                               │
│  openclaw-data:                                              │
│  └── encryption-master-key   (PostgreSQL only)               │
│                                                               │
│  openclaw-audit:                                             │
│  └── audit-write-token       (append-only, no read/delete)  │
│                                                               │
│  KEY PRINCIPLE: Tool Executor NEVER sees LLM API keys.       │
│  LLM Service NEVER sees SMTP credentials. Each service       │
│  only has the secrets it needs for its specific function.    │
└──────────────────────────────────────────────────────────────┘
```

### 8.3 Expiry & Rotation Strategy

| Secret | Rotation Period | Method | Special Considerations |
|--------|----------------|--------|----------------------|
| `jwt-signing-keys` | 30 days | **Dual-key validation**: generate new key pair → both old and new keys are valid for 24h → remove old key. Prevents invalidating active sessions during rotation. | Use RS256 (asymmetric) — public key distributed to all validators, private key stays in Gateway only. |
| `db-credentials` | 60 days | Vault dynamic secrets with TTL. Vault generates temporary PostgreSQL credentials that auto-expire. | Synchronous replica uses the same credentials — rotate on primary, replica picks up new creds via streaming replication config. |
| `redis-credentials` | 90 days | Update in Vault → ESO syncs to both `openclaw-core` and `openclaw-tools` namespaces → rolling restart. | Redis ACL update must happen BEFORE Pod restart. Pipeline: update Redis ACL → sync Secret → restart Pods. |
| `llm-api-keys` | 90 days | Rotate in provider dashboard → update in Vault → ESO auto-syncs. | Support dual-key: both old and new keys valid for 1 hour during rotation to prevent request failures. |
| `smtp-credentials` | 180 days | Rotate in email provider → update in Vault. | Low frequency — email tool is not always active. |
| `google-service-account` | 365 days | Generate new key in Google Cloud Console → revoke old key after 24h. | Google allows max 10 keys per service account. Delete old keys to stay within limit. |
| `encryption-master-key` | Never rotate (versioned) | Key versioning — new data uses v2, old data decrypted with v1. Both versions stored. | **NEVER** delete old key versions — encrypted data would become permanently unreadable. |

### 8.4 Rotation Automation Flow

```
┌──────────────┐     ┌──────────────┐     ┌───────────────────┐
│  HashiCorp   │────►│  External    │────►│  K8s Secret       │
│  Vault       │     │  Secrets     │     │  (in namespace)   │
│              │     │  Operator    │     │                   │
│  Source of   │     │  (ESO)       │     │  Auto-synced      │
│  truth       │     │              │     │  every 60 min     │
└──────────────┘     │  Watches     │     └────────┬──────────┘
                     │  Vault for   │              │
                     │  changes     │              ▼
                     └──────────────┘     ┌───────────────────┐
                                          │  Rolling restart  │
                                          │  triggered by     │
                                          │  Reloader         │
                                          │  (detects Secret  │
                                          │  change → restart │
                                          │  affected Pods)   │
                                          └───────────────────┘
```

---

## 9. Resource Requests & Limits

### Resource Allocation Table

| Service | Namespace | CPU Request | CPU Limit | Mem Request | Mem Limit | Reasoning |
|---------|-----------|------------|-----------|------------|-----------|-----------|
| **API Gateway** | gateway | 200m | 500m | 256Mi | 512Mi | Lightweight request routing. TLS termination is handled by Ingress, not Gateway. |
| **Agent Orchestrator** | core | 500m | 2000m | 512Mi | 1Gi | Manages multi-step chains in memory. CPU spikes during chain planning. Needs burst capacity. |
| **LLM Service** | core | 250m | 1000m | 256Mi | 512Mi | I/O bound (waiting on external LLM API). Low steady CPU. Memory for prompt buffers. |
| **Auth & Policy Service** | core | 250m | 500m | 256Mi | 512Mi | Fast policy evaluation (in-memory with Redis cache). Must respond in <10ms. Low resource but latency-critical. |
| **Memory Service** | core | 250m | 1000m | 256Mi | 512Mi | Embedding computation is mildly CPU-intensive. Vector search results buffered in memory. |
| **Tool Executor** | tools | 500m | 2000m | 512Mi | 1Gi | Manages ephemeral tool containers. Needs overhead for spawning and monitoring sandboxed jobs. |
| **PostgreSQL** | data | 500m | 2000m | 1Gi | 2Gi | Audit logs generate write-heavy workload. Buffer pool needs ample memory for query caching. |
| **Redis** | data | 250m | 1000m | 1Gi | 2Gi | Higher memory than Plan 1 — stores tool job payloads, session data, and policy cache. |
| **Qdrant** | data | 500m | 2000m | 2Gi | 4Gi | Vector search is memory-intensive — keeps HNSW index in RAM for fast similarity search. |

### Total Resource Footprint

```
Namespace            Service              Reps  CPU Req  Mem Req
─────────            ───────              ────  ───────  ───────
openclaw-gateway     API Gateway          3     600m     768Mi
openclaw-core        Orchestrator         3     1500m    1.5Gi
                     LLM Service          2     500m     512Mi
                     Auth & Policy        3     750m     768Mi
                     Memory Service       2     500m     512Mi
openclaw-tools       Tool Executor        2     1000m    1Gi
openclaw-data        PostgreSQL           2     1000m    2Gi
                     Redis                3     750m     3Gi
                     Qdrant               2     1000m    4Gi
                                               ───────  ───────
TOTAL                                          7600m    14Gi
                                               (~8 CPU)  (~14 GB)

Recommended cluster: 4 worker nodes × 4 CPU × 8Gi RAM = 16 CPU, 32Gi RAM
(allows 50% headroom for autoscaling and system pods)
```

---

## 10. RBAC — Roles & RoleBindings

### 10.1 Service Accounts (One Per Service, Strict Isolation)

| ServiceAccount | Namespace | Used By |
|---------------|-----------|---------|
| `sa-gateway` | openclaw-gateway | API Gateway |
| `sa-orchestrator` | openclaw-core | Agent Orchestrator |
| `sa-llm` | openclaw-core | LLM Service |
| `sa-auth-policy` | openclaw-core | Auth & Policy Service |
| `sa-memory` | openclaw-core | Memory Service |
| `sa-tool-executor` | openclaw-tools | Tool Executor |
| `sa-postgres` | openclaw-data | PostgreSQL |
| `sa-redis` | openclaw-data | Redis |
| `sa-qdrant` | openclaw-data | Qdrant |

### 10.2 Roles

**Role: `core-config-reader`** (Namespace: openclaw-core)
```
Rules:
  - apiGroups: [""]
    resources: ["configmaps"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["secrets"]
    resourceNames: ["db-credentials", "redis-credentials"]    ← named secrets only
    verbs: ["get"]
```

**Role: `llm-secret-reader`** (Namespace: openclaw-core)
```
Rules:
  - apiGroups: [""]
    resources: ["secrets"]
    resourceNames: ["llm-api-keys"]    ← ONLY the LLM API key secret
    verbs: ["get"]
```

**Role: `memory-secret-reader`** (Namespace: openclaw-core)
```
Rules:
  - apiGroups: [""]
    resources: ["secrets"]
    resourceNames: ["db-credentials-readonly", "qdrant-api-key"]
    verbs: ["get"]
```

**Role: `tool-queue-reader`** (Namespace: openclaw-tools)
```
Rules:
  - apiGroups: [""]
    resources: ["configmaps"]
    verbs: ["get", "list"]
  - apiGroups: [""]
    resources: ["secrets"]
    resourceNames: ["redis-credentials", "smtp-credentials", "google-service-account"]
    verbs: ["get"]
```

**Role: `pvc-manager`** (Namespace: openclaw-data)
```
Rules:
  - apiGroups: [""]
    resources: ["persistentvolumeclaims"]
    verbs: ["get", "list", "watch", "create", "update", "patch"]
```

**Role: `gateway-minimal`** (Namespace: openclaw-gateway)
```
Rules:
  - apiGroups: [""]
    resources: ["secrets"]
    resourceNames: ["jwt-signing-keys"]
    verbs: ["get"]
```

### 10.3 RoleBindings

| RoleBinding | Namespace | ServiceAccount | Role | Access Granted |
|------------|-----------|----------------|------|---------------|
| `gateway-jwt-binding` | openclaw-gateway | `sa-gateway` | `gateway-minimal` | Read JWT signing keys only |
| `orchestrator-config-binding` | openclaw-core | `sa-orchestrator` | `core-config-reader` | Read ConfigMaps + DB/Redis secrets |
| `auth-config-binding` | openclaw-core | `sa-auth-policy` | `core-config-reader` | Read ConfigMaps + DB/Redis secrets |
| `llm-secret-binding` | openclaw-core | `sa-llm` | `llm-secret-reader` | Read LLM API key ONLY |
| `memory-secret-binding` | openclaw-core | `sa-memory` | `memory-secret-reader` | Read DB readonly creds + Qdrant key |
| `tool-executor-binding` | openclaw-tools | `sa-tool-executor` | `tool-queue-reader` | Read tool-specific secrets + config |
| `postgres-storage-binding` | openclaw-data | `sa-postgres` | `pvc-manager` | Manage PVCs |
| `redis-storage-binding` | openclaw-data | `sa-redis` | `pvc-manager` | Manage PVCs |
| `qdrant-storage-binding` | openclaw-data | `sa-qdrant` | `pvc-manager` | Manage PVCs |

### 10.4 RBAC Access Matrix

```
                         Gateway  Orchestrator  LLM    Auth   Memory  ToolExec  Postgres  Redis  Qdrant
                         ───────  ────────────  ───    ────   ──────  ────────  ────────  ─────  ──────
JWT Keys                 ✓        ✗             ✗      ✗      ✗       ✗         ✗         ✗      ✗
DB Credentials (RW)      ✗        ✓             ✗      ✓      ✗       ✗         ✗         ✗      ✗
DB Credentials (RO)      ✗        ✗             ✗      ✗      ✓       ✗         ✗         ✗      ✗
Redis Credentials        ✗        ✓             ✗      ✓      ✗       ✓         ✗         ✗      ✗
LLM API Keys             ✗        ✗             ✓      ✗      ✗       ✗         ✗         ✗      ✗
Qdrant API Key           ✗        ✗             ✗      ✗      ✓       ✗         ✗         ✗      ✗
SMTP Credentials         ✗        ✗             ✗      ✗      ✗       ✓         ✗         ✗      ✗
Google SA Key            ✗        ✗             ✗      ✗      ✗       ✓         ✗         ✗      ✗
Encryption Key           ✗        ✗             ✗      ✗      ✗       ✗         ✓         ✗      ✗
PVC Management           ✗        ✗             ✗      ✗      ✗       ✗         ✓         ✓      ✓
K8s API Access           ✗        ✗             ✗      ✗      ✗       ✗         ✗         ✗      ✗
```

**Key insight:** No single service has access to everything. Even if an attacker fully compromises the Tool Executor, they get SMTP and Google credentials — but NOT the database, NOT LLM API keys, NOT JWT signing keys. The blast radius is contained.

### 10.5 Human Access Roles

| ClusterRole | Who | Scope | Permissions |
|------------|-----|-------|-------------|
| `openclaw-platform-admin` | DevOps Lead | All openclaw-* namespaces | Full access — deploy, scale, manage secrets, view audit logs |
| `openclaw-developer` | Developers | openclaw-core, openclaw-tools | Read Pods, logs, ConfigMaps. No Secrets. Port-forward for debugging. |
| `openclaw-security-auditor` | Security Team | openclaw-audit (read), all namespaces (read Pods/NetworkPolicies) | Read audit logs, inspect network policies, verify RBAC configuration. Cannot modify anything. |
| `openclaw-incident-responder` | On-call | All openclaw-* namespaces | Read all + scale deployments + restart Pods. No Secret access. No delete. |

---

## 11. Inter-Service Communication

### Communication Flow Diagram

```
┌────────────────────────────────────────────────────────────────────────┐
│                                                                        │
│  EXTERNAL → INTERNAL (via Ingress)                                    │
│                                                                        │
│  User ──HTTPS──► Ingress ──HTTP──► Gateway (openclaw-gateway)         │
│                                       │                                │
│  GATEWAY → CORE (cross-namespace)     │                                │
│                                       │                                │
│  Gateway ──HTTP──► Orchestrator (openclaw-core)                       │
│                        │                                               │
│  CORE INTERNAL (same namespace, short DNS)                            │
│                        │                                               │
│                        ├──HTTP──► LLM Service                         │
│                        ├──HTTP──► Auth & Policy Service               │
│                        ├──HTTP──► Memory Service                      │
│                        │                                               │
│  CORE → DATA (cross-namespace)                                        │
│                        │                                               │
│                        ├──TCP──► PostgreSQL (openclaw-data)           │
│                        ├──TCP──► Redis (openclaw-data)                │
│                        │                                               │
│  CORE → TOOLS (async via Redis queue, cross-namespace)                │
│                        │                                               │
│  Orchestrator ──Redis Queue──► Tool Executor (openclaw-tools)         │
│                                    │                                   │
│  TOOLS → EXTERNAL (restricted egress)                                 │
│                                    │                                   │
│                                    ├──SMTP──► SendGrid                │
│                                    ├──HTTPS──► Google APIs            │
│                                    └──HTTPS──► Other whitelisted APIs │
│                                                                        │
│  RESULTS FLOW BACK (async)                                            │
│                                                                        │
│  Tool Executor ──Redis Queue──► Orchestrator ──HTTP──► Gateway ──► User│
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘
```

### Communication Table

| From → To | Protocol | DNS Name | Pattern | Why This Pattern |
|-----------|----------|----------|---------|-----------------|
| Ingress → Gateway | HTTP | `gateway-svc.openclaw-gateway.svc.cluster.local:8443` | Sync | User-facing, needs immediate response |
| Gateway → Orchestrator | HTTP | `orchestrator-svc.openclaw-core.svc.cluster.local:8080` | Sync (with streaming) | Gateway streams Orchestrator's response back to user in real-time |
| Orchestrator → LLM Service | HTTP | `llm-svc:8081` | Sync | Orchestrator waits for LLM reasoning before deciding next step |
| Orchestrator → Auth Service | HTTP | `auth-policy-svc:8082` | Sync | Tool execution BLOCKED until Auth responds allow/deny. Must be synchronous — cannot proceed without authorization. |
| Orchestrator → Memory Service | HTTP | `memory-svc:8083` | Sync | Orchestrator needs memory context before constructing LLM prompt |
| Orchestrator → Tool Executor | Redis Queue | `redis-svc.openclaw-data.svc.cluster.local:6379` | **Async** | Tool execution may take 1-30 seconds. Queue decouples Orchestrator from execution latency. Enables retry. |
| Tool Executor → Orchestrator | Redis Queue | Same Redis, different queue name | **Async** | Results pushed to queue. Orchestrator picks up and continues chain. |
| Orchestrator → PostgreSQL | TCP | `postgres-svc.openclaw-data.svc.cluster.local:5432` | Sync | Read/write user data, policies, logs |
| Memory Service → Qdrant | HTTP/gRPC | `qdrant-svc.openclaw-data.svc.cluster.local:6333` | Sync | Vector search must return before Orchestrator proceeds |
| Auth Service → Redis | TCP | `redis-svc.openclaw-data.svc.cluster.local:6379` | Sync | Cache hit → <1ms policy eval. Cache miss → read from PostgreSQL. |
| All Services → Audit | HTTP (one-way) | `audit-collector-svc.openclaw-audit.svc.cluster.local:9200` | **Fire-and-forget** | Audit logging must never block the critical path. If audit collector is slow, log is queued locally and retried. |

---

## 12. Sandboxed Tool Execution

This is the most security-critical component. The Tool Executor runs real-world actions (sending emails, accessing APIs), so it must be heavily sandboxed.

### Sandbox Architecture

```
┌──────────────────────────────────────────────────────────────────┐
│  TOOL EXECUTOR POD                                                │
│                                                                   │
│  ┌──────────────────────────────────────────────────────────┐    │
│  │  Executor Process                                         │    │
│  │                                                           │    │
│  │  1. Pull job from Redis queue: "approved-tool-jobs"      │    │
│  │  2. Validate job has auth-policy-svc approval token      │    │
│  │  3. Select tool handler based on job type                │    │
│  │  4. Execute tool in restricted subprocess:               │    │
│  │                                                           │    │
│  │     ┌───────────────────────────────────┐                │    │
│  │     │  SANDBOXED TOOL EXECUTION          │                │    │
│  │     │                                    │                │    │
│  │     │  • Read-only filesystem            │                │    │
│  │     │  • No network except whitelisted   │                │    │
│  │     │    domains                         │                │    │
│  │     │  • 30-second timeout (hard kill)   │                │    │
│  │     │  • 200m CPU / 128Mi RAM cap        │                │    │
│  │     │  • No access to K8s API            │                │    │
│  │     │  • Secret injected read-only,      │                │    │
│  │     │    scoped to THIS tool only        │                │    │
│  │     │  • stdout/stderr captured for      │                │    │
│  │     │    audit log                       │                │    │
│  │     └───────────────────────────────────┘                │    │
│  │                                                           │    │
│  │  5. Capture result (success/failure + output)            │    │
│  │  6. Push result to Redis queue: "tool-results"           │    │
│  │  7. Log execution details to audit                       │    │
│  └──────────────────────────────────────────────────────────┘    │
└──────────────────────────────────────────────────────────────────┘
```

### Available Tools & Their Permissions

| Tool | External Access | Secret Required | Risk | Safety Controls |
|------|----------------|-----------------|------|----------------|
| `email-send` | `smtp.sendgrid.net:587` | `smtp-credentials` | HIGH — can send emails as the user | Recipient whitelist per user policy. Max 10 emails/hour. Subject/body logged. |
| `calendar-read` | `www.googleapis.com:443` | `google-service-account` | LOW — read-only | User policy scopes to specific calendars |
| `calendar-write` | `www.googleapis.com:443` | `google-service-account` | MEDIUM — can create/modify events | Requires explicit policy. Max 5 events/hour. |
| `web-search` | `api.google.com:443` | `google-service-account` | LOW — read-only public data | Rate limited. No access to authenticated pages. |
| `code-execute` | None (air-gapped) | None | HIGH — runs arbitrary code | Completely network-isolated. 10-second timeout. 50m CPU / 64Mi RAM. stdout only. |
| `file-read` | None | None | MEDIUM — reads user files | Scoped to user's designated directory only. No `..` traversal. |
| `database-query` | Internal only (`postgres-svc`) | `db-credentials-readonly` | MEDIUM — read-only queries | Read-only DB user. Query timeout 10 seconds. No DDL/DML allowed. |

---

## 13. Audit Logging & Observability

### What Gets Logged

Every action in the system produces an immutable audit record:

```
┌────────────────────────────────────────────────────────────────────────┐
│                          AUDIT LOG ENTRY                               │
│                                                                        │
│  {                                                                     │
│    "timestamp": "2026-03-28T14:30:05.123Z",                          │
│    "request_id": "req_abc123",                                        │
│    "user_id": "user_456",                                             │
│    "service": "auth-policy-svc",                                      │
│    "action": "tool-authorization",                                    │
│    "tool": "email-send",                                              │
│    "parameters": {                                                    │
│      "to": "john@company.com",                                       │
│      "subject": "Meeting Summary"                                     │
│    },                                                                  │
│    "decision": "ALLOW",                                               │
│    "policy_matched": "allow-email-to-company-domain",                 │
│    "execution_result": "success",                                     │
│    "duration_ms": 1250                                                │
│  }                                                                     │
└────────────────────────────────────────────────────────────────────────┘
```

### Audit Events Captured

| Event | Source Service | What's Logged |
|-------|---------------|---------------|
| User request received | API Gateway | User ID, IP, request summary, timestamp |
| Authentication result | API Gateway | Success/failure, JWT claims, source IP |
| Agent chain started | Orchestrator | User instruction (sanitized), planned steps |
| LLM call made | LLM Service | Model used, token count, latency (NOT the full prompt — may contain sensitive data) |
| Tool authorization decision | Auth & Policy | Tool name, parameters, matched policy, ALLOW/DENY |
| Tool execution started | Tool Executor | Tool name, parameters, sandbox config |
| Tool execution completed | Tool Executor | Result (success/failure), duration, output summary |
| Memory stored | Memory Service | Entry ID, user ID (NOT the content — contains user data) |
| Error/exception | Any service | Error type, stack trace, affected request ID |

### Audit Storage

```
All Services ──fire-and-forget──► Audit Collector (openclaw-audit)
                                        │
                                        ▼
                                  ┌──────────────┐
                                  │  PostgreSQL   │
                                  │  (audit DB)   │
                                  │               │
                                  │  APPEND-ONLY  │
                                  │  No UPDATE    │
                                  │  No DELETE    │
                                  │               │
                                  │  Retention:   │
                                  │  365 days     │
                                  └──────────────┘
```

**Audit DB security:**
- Separate PostgreSQL instance (or separate database in the same instance with different credentials)
- DB user has INSERT permission only — no UPDATE, DELETE, or TRUNCATE
- Even cluster admins cannot delete audit records without triggering alerts
- Retention: 365 days (configurable by compliance requirements)

---

## 14. Scaling Strategy

### Horizontal Pod Autoscaler (HPA)

| Service | Min | Max | Scale Trigger | Cooldown |
|---------|-----|-----|--------------|----------|
| API Gateway | 3 | 8 | CPU > 60% OR connections > 1000 | Scale-down: 5 min |
| Orchestrator | 3 | 10 | CPU > 70% OR active chains > 50 per Pod | Scale-down: 5 min |
| LLM Service | 2 | 6 | CPU > 70% OR request queue > 20 | Scale-down: 10 min |
| Auth & Policy | 3 | 6 | Latency p99 > 50ms (must stay fast) | Scale-down: 5 min |
| Memory Service | 2 | 5 | CPU > 70% | Scale-down: 5 min |
| Tool Executor | 2 | 10 | Redis queue depth > 20 pending jobs | Scale-down: 10 min |

**StatefulSets (PostgreSQL, Redis, Qdrant) are NOT auto-scaled.** Database scaling requires manual planning.

### Scaling Scenario

```
Normal:    [GW×3] [Orch×3] [LLM×2] [Auth×3] [Mem×2] [Tool×2]
                │
   Peak hour: 200 concurrent users
                │
Scaled:    [GW×5] [Orch×7] [LLM×4] [Auth×4] [Mem×3] [Tool×8]
                │                                         ▲
                │                               Tool Executor scales most
                │                               (bottleneck: external API latency)
   Night: 10 users
                │
Minimum:   [GW×3] [Orch×3] [LLM×2] [Auth×3] [Mem×2] [Tool×2]
```

---

## 15. Health Checks

| Service | Liveness | Readiness | Startup |
|---------|----------|-----------|---------|
| **API Gateway** | `GET /healthz` every 10s | `GET /ready` every 5s (checks JWT key loaded + Orchestrator reachable) | `GET /healthz` every 3s, timeout 60s |
| **Orchestrator** | `GET /healthz` every 15s | `GET /ready` every 10s (checks DB + Redis + Auth Service connectivity) | `GET /healthz` every 5s, timeout 120s (DB migrations) |
| **LLM Service** | `GET /healthz` every 15s | `GET /ready` every 10s (checks LLM API key valid + can reach provider) | `GET /healthz` every 5s, timeout 60s |
| **Auth & Policy** | `GET /healthz` every 10s | `GET /ready` every 5s (checks policy cache loaded from DB) | `GET /healthz` every 3s, timeout 60s |
| **Memory Service** | `GET /healthz` every 15s | `GET /ready` every 10s (checks Qdrant connection) | `GET /healthz` every 5s, timeout 60s |
| **Tool Executor** | `GET /healthz` every 15s | `GET /ready` every 10s (checks Redis queue connection) | `GET /healthz` every 5s, timeout 30s |
| **PostgreSQL** | `exec: pg_isready` every 15s | `exec: pg_isready` every 10s | `exec: pg_isready` every 5s, timeout 120s |
| **Redis** | `exec: redis-cli ping` every 15s | `exec: redis-cli ping` every 10s | `exec: redis-cli ping` every 5s, timeout 60s |
| **Qdrant** | `GET /healthz` every 15s | `GET /readyz` every 10s | `GET /healthz` every 5s, timeout 90s |

**Critical dependency:** Auth & Policy Service has aggressive probing (every 5s readiness) because if Auth is unhealthy, all tool executions are blocked. Fast detection = fast recovery.

---

## 16. Storage Design

### Persistent Volume Claims

| PVC Name Pattern | Namespace | Size | Storage Class | Used By |
|-----------------|-----------|------|---------------|---------|
| `postgres-data-postgres-{0,1}` | openclaw-data | 50Gi each | `ssd-retain` | PostgreSQL primary + replica |
| `redis-data-redis-{0,1,2}` | openclaw-data | 10Gi each | `ssd-retain` | Redis master + 2 replicas |
| `qdrant-data-qdrant-{0,1}` | openclaw-data | 30Gi each | `ssd-retain` | Qdrant primary + replica |

### Total Storage

```
PostgreSQL:  2 × 50Gi  = 100Gi   (larger than Plan 1 — audit logs grow continuously)
Redis:       3 × 10Gi  =  30Gi   (tool job payloads can be substantial)
Qdrant:      2 × 30Gi  =  60Gi   (vector embeddings are dense data)
                         ───────
TOTAL:                   190Gi SSD
```

### Backup Strategy

| Data Store | Backup Frequency | Retention | Method |
|-----------|-----------------|-----------|--------|
| PostgreSQL | Every 6 hours + continuous WAL archiving | 30 days | `pg_basebackup` to object storage (S3/GCS). WAL archiving for point-in-time recovery. |
| Redis | Every 12 hours (RDB snapshot) | 7 days | RDB dump to object storage. Queue data is transient — 7 days is sufficient. |
| Qdrant | Every 24 hours (snapshot) | 14 days | Qdrant snapshot API to object storage. Embeddings can be regenerated from source data if needed. |

---

## 17. Network Policies (Zero-Trust)

### Default Policy: Deny All

Every namespace starts with a default-deny policy. Then explicit allow rules are added.

```
┌────────────────────────────────────────────────────────────────────────┐
│                      NETWORK POLICY MAP                                │
│                                                                        │
│  DEFAULT: ALL namespaces have "deny all ingress + deny all egress"    │
│                                                                        │
│  ┌─ openclaw-gateway ────────────────────────────────────────────┐    │
│  │                                                                │    │
│  │  INGRESS:                                                      │    │
│  │    ✓ From: ingress-nginx namespace, port 8443                 │    │
│  │    ✗ Everything else: DENIED                                   │    │
│  │                                                                │    │
│  │  EGRESS:                                                       │    │
│  │    ✓ To: openclaw-core (orchestrator-svc:8080)                │    │
│  │    ✓ To: openclaw-audit (audit-collector-svc:9200)            │    │
│  │    ✓ To: kube-dns (port 53, for DNS resolution)               │    │
│  │    ✗ To: openclaw-data — DENIED (Gateway cannot reach DB)     │    │
│  │    ✗ To: openclaw-tools — DENIED                              │    │
│  │    ✗ To: internet — DENIED (Gateway doesn't make external     │    │
│  │                              calls)                            │    │
│  └────────────────────────────────────────────────────────────────┘    │
│                                                                        │
│  ┌─ openclaw-core ───────────────────────────────────────────────┐    │
│  │                                                                │    │
│  │  INGRESS:                                                      │    │
│  │    ✓ From: openclaw-gateway (gateway Pods → orchestrator)     │    │
│  │    ✓ From: within openclaw-core (services talk to each other) │    │
│  │    ✗ From: openclaw-tools — DENIED (Tool Executor cannot      │    │
│  │           call core services directly)                        │    │
│  │    ✗ From: openclaw-data — DENIED                             │    │
│  │    ✗ From: internet — DENIED                                  │    │
│  │                                                                │    │
│  │  EGRESS:                                                       │    │
│  │    ✓ To: openclaw-data (postgres, redis, qdrant)              │    │
│  │    ✓ To: openclaw-audit (audit logging)                       │    │
│  │    ✓ To: external LLM APIs (api.anthropic.com,               │    │
│  │          api.openai.com) — LLM Service only                   │    │
│  │    ✓ To: kube-dns (port 53)                                   │    │
│  │    ✗ To: openclaw-tools — Orchestrator uses Redis queue,      │    │
│  │          NOT direct HTTP to Tool Executor                     │    │
│  └────────────────────────────────────────────────────────────────┘    │
│                                                                        │
│  ┌─ openclaw-tools ──────────────────────────────────────────────┐    │
│  │                                                                │    │
│  │  INGRESS:                                                      │    │
│  │    ✗ ALL DENIED (Tool Executor receives work via Redis         │    │
│  │      queue, not direct HTTP calls)                             │    │
│  │                                                                │    │
│  │  EGRESS:                                                       │    │
│  │    ✓ To: openclaw-data redis-svc:6379 ONLY (queue access)    │    │
│  │    ✓ To: openclaw-audit (audit logging)                       │    │
│  │    ✓ To: WHITELISTED external domains only:                   │    │
│  │         smtp.sendgrid.net:587                                 │    │
│  │         www.googleapis.com:443                                │    │
│  │         api.google.com:443                                    │    │
│  │    ✓ To: kube-dns (port 53)                                   │    │
│  │    ✗ To: openclaw-core — DENIED                               │    │
│  │    ✗ To: openclaw-data postgres/qdrant — DENIED               │    │
│  │    ✗ To: any other external domain — DENIED                   │    │
│  └────────────────────────────────────────────────────────────────┘    │
│                                                                        │
│  ┌─ openclaw-data ───────────────────────────────────────────────┐    │
│  │                                                                │    │
│  │  INGRESS:                                                      │    │
│  │    ✓ From: openclaw-core (postgres, redis, qdrant access)     │    │
│  │    ✓ From: openclaw-tools → redis-svc:6379 ONLY              │    │
│  │    ✗ From: openclaw-gateway — DENIED (Gateway cannot          │    │
│  │           reach database directly)                            │    │
│  │    ✗ From: internet — DENIED                                  │    │
│  │                                                                │    │
│  │  EGRESS:                                                       │    │
│  │    ✓ To: within openclaw-data (replication traffic)           │    │
│  │    ✓ To: kube-dns (port 53)                                   │    │
│  │    ✗ To: everything else — DENIED (databases never make       │    │
│  │          outbound calls)                                      │    │
│  └────────────────────────────────────────────────────────────────┘    │
│                                                                        │
│  ┌─ openclaw-audit ──────────────────────────────────────────────┐    │
│  │                                                                │    │
│  │  INGRESS:                                                      │    │
│  │    ✓ From: ALL openclaw-* namespaces (everyone writes         │    │
│  │          audit logs)                                          │    │
│  │    Port: 9200 only (write endpoint)                           │    │
│  │                                                                │    │
│  │  EGRESS:                                                       │    │
│  │    ✓ To: within openclaw-audit (internal storage)             │    │
│  │    ✓ To: kube-dns (port 53)                                   │    │
│  │    ✗ To: everything else — DENIED                             │    │
│  └────────────────────────────────────────────────────────────────┘    │
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘
```

### Visual: What Can Reach What

```
                    Gateway  Core  Tools  Data  Audit  Internet
                    ───────  ────  ─────  ────  ─────  ────────
Gateway    →          -       ✓     ✗      ✗     ✓      ✗
Core       →          ✗       ✓     ✗*     ✓     ✓      ✓**
Tools      →          ✗       ✗     -      ✓***  ✓      ✓****
Data       →          ✗       ✗     ✗      ✓     ✗      ✗
Audit      →          ✗       ✗     ✗      ✗     ✓      ✗

*    Core → Tools: via Redis queue only (not direct HTTP)
**   Core → Internet: LLM APIs only (api.anthropic.com, api.openai.com)
***  Tools → Data: Redis only (not PostgreSQL or Qdrant)
**** Tools → Internet: Whitelisted domains only (SMTP, Google APIs)
```

---

## 18. Deployment Strategy

### Rollout Order (First-Time Deployment)

```
Phase 1 — Data Layer (openclaw-data):
  ┌────────────────────────────────────────────────┐
  │  1. PostgreSQL StatefulSet                      │
  │     Wait: primary healthy + replica streaming   │
  │  2. Redis StatefulSet                           │
  │     Wait: master + sentinels + replicas ready  │
  │  3. Qdrant StatefulSet                          │
  │     Wait: primary healthy + collection created │
  └────────────────────────────────────────────────┘
                         │
                         ▼
Phase 2 — Audit Layer (openclaw-audit):
  ┌────────────────────────────────────────────────┐
  │  4. Audit Collector Deployment                  │
  │     Wait: healthy + accepting log writes       │
  └────────────────────────────────────────────────┘
                         │
                         ▼
Phase 3 — Core Services (openclaw-core):
  ┌────────────────────────────────────────────────┐
  │  5. Auth & Policy Service (FIRST — other       │
  │     services depend on it)                     │
  │     Wait: policies loaded from DB + cache warm │
  │  6. LLM Service                                │
  │     Wait: API key validated + provider healthy │
  │  7. Memory Service                              │
  │     Wait: Qdrant connection + collections ready│
  │  8. Agent Orchestrator                          │
  │     Wait: all dependencies connected           │
  └────────────────────────────────────────────────┘
                         │
                         ▼
Phase 4 — Tool Execution (openclaw-tools):
  ┌────────────────────────────────────────────────┐
  │  9. Tool Executor Deployment                    │
  │     Wait: Redis queue connected + ready        │
  └────────────────────────────────────────────────┘
                         │
                         ▼
Phase 5 — Gateway (openclaw-gateway):
  ┌────────────────────────────────────────────────┐
  │  10. API Gateway Deployment                     │
  │      Wait: JWT keys loaded + Orchestrator      │
  │      reachable                                 │
  │  11. Ingress rules applied                      │
  │      Traffic flows to users                    │
  └────────────────────────────────────────────────┘
                         │
                         ▼
Phase 6 — Network Policies (applied LAST):
  ┌────────────────────────────────────────────────┐
  │  12. Apply all NetworkPolicies                  │
  │      (Applied after all services are running   │
  │       to avoid blocking during startup)        │
  └────────────────────────────────────────────────┘
```

### Update Strategy Per Service

| Service | Strategy | Grace Period | Notes |
|---------|----------|-------------|-------|
| API Gateway | RollingUpdate (maxSurge: 1, maxUnavailable: 0) | 30s | Zero-downtime required |
| Orchestrator | RollingUpdate (maxSurge: 1, maxUnavailable: 1) | 120s | Allow in-progress chains to complete |
| LLM Service | RollingUpdate (maxSurge: 1, maxUnavailable: 0) | 60s | Wait for active LLM calls to finish |
| Auth & Policy | RollingUpdate (maxSurge: 1, maxUnavailable: 0) | 30s | NEVER fewer than 3 healthy (fail-closed protection) |
| Memory Service | RollingUpdate (maxSurge: 1, maxUnavailable: 1) | 30s | |
| Tool Executor | RollingUpdate (maxSurge: 1, maxUnavailable: 0) | 60s | Wait for active tool executions to complete |
| PostgreSQL | OnDelete (manual) | 300s | Never auto-restart databases |
| Redis | OnDelete (manual) | 120s | Sentinel handles failover |
| Qdrant | OnDelete (manual) | 120s | Manual to prevent data inconsistency |

### Rollback Plan

```
If deployment fails in any phase:

  1. Automatic: progressDeadlineSeconds = 300 (5 min)
     If not complete → marked as failed

  2. Manual rollback:
     kubectl rollout undo deployment/<name> -n <namespace>

  3. Emergency: If Auth & Policy Service fails:
     → ALL tool executions auto-blocked (fail-closed)
     → Users can still read data but AI cannot take actions
     → Prioritize Auth Service recovery above all others

  4. Database rollback:
     → PostgreSQL: point-in-time recovery from WAL archive
     → Redis: restore from latest RDB snapshot
     → Qdrant: restore from snapshot (embeddings can also be regenerated)
```

---

## Summary: Plan 2 vs Plan 1 Comparison

```
┌────────────────────────────────────────────────────────────────────┐
│           AI EMPLOYEE (OpenClaw) — K8s SUMMARY                     │
│                                                                    │
│  Namespaces:    5 (gateway, core, tools, data, audit)             │
│  Deployments:   6 (Gateway, Orchestrator, LLM, Auth,             │
│                    Memory, Tool Executor)                          │
│  StatefulSets:  3 (PostgreSQL, Redis, Qdrant)                     │
│  Services:      9 (all ClusterIP, single Ingress entry)           │
│  ConfigMaps:    6 (one per application service)                   │
│  Secrets:       10 (namespace-isolated, ESO + Vault managed)      │
│  ServiceAccts:  9 (strict per-service isolation)                  │
│  Roles:         6 + 4 human roles                                 │
│  PVCs:          7 (2 Postgres + 3 Redis + 2 Qdrant)              │
│  HPAs:          6 (all application services)                      │
│  Network Policies: Zero-trust, default-deny, cross-namespace     │
│                                                                    │
│  Total Resources: ~8 CPU cores, ~14 GB RAM minimum               │
│  Total Storage:   190 Gi persistent SSD                           │
│  External Deps:   LLM APIs, SMTP, Google APIs                    │
│                                                                    │
│  KEY DIFFERENCE FROM PLAN 1:                                      │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │  Plan 1: Single namespace, moderate security              │     │
│  │  Plan 2: 5 namespaces, zero-trust, sandboxed execution,  │     │
│  │          audit logging, defense-in-depth, fail-closed     │     │
│  │          auth, cross-namespace network policies           │     │
│  └──────────────────────────────────────────────────────────┘     │
└────────────────────────────────────────────────────────────────────┘
```
