# Kubernetes Deployment Planning

Production-grade Kubernetes deployment plans for two AI application scenarios — no code, pure architecture and system design.

> **Course:** Panaversity — Kubernetes
> **Project:** 02 — Kubernetes Deployment Planning
> **Author:** Faisal

---

## Project Structure

```
├── plan-1-ai-native-task-manager.md    # Scenario 1: AI Task Manager
├── plan-2-ai-employee-openclaw.md      # Scenario 2: AI Employee (OpenClaw)
├── k8-planning-skill.md                # Reusable K8 Planning Skill
└── README.md
```

## Scenario 1: AI Native Task Manager

A microservices-based task management application with AI-powered task processing.

**Services:** UI Interface, Backend APIs, Task Agent, Notification Service
**Infrastructure:** PostgreSQL, Redis

| Highlight | Details |
|-----------|---------|
| Namespace | Single namespace (`ai-task-manager`) |
| Deployments | 4 stateless services |
| StatefulSets | 2 (PostgreSQL + Redis) |
| Services | 7 (all ClusterIP, exposed via Ingress) |
| Security | RBAC per service, network policies, non-root Pods |
| Scaling | HPA on all application services |

## Scenario 2: AI Employee — OpenClaw

A personal AI agent that autonomously executes real-world actions (email, calendar, code execution) with strict security controls.

**Services:** API Gateway, Agent Orchestrator, LLM Service, Auth & Policy Service, Memory Service, Tool Executor
**Infrastructure:** PostgreSQL, Redis, Qdrant (Vector DB)

| Highlight | Details |
|-----------|---------|
| Namespaces | 5 (gateway, core, tools, data, audit) |
| Deployments | 6 stateless services |
| StatefulSets | 3 (PostgreSQL + Redis + Qdrant) |
| Services | 9 (all ClusterIP, single Ingress entry) |
| Security | Zero-trust, sandboxed tool execution, fail-closed auth, immutable audit logs |
| Scaling | HPA on all 6 application services |

## K8 Planning Skill

A reusable Claude Code skill (`/k8-planner`) that generates Kubernetes deployment plans for **any** application.

**Features:**
- 9-phase planning process (Discovery → Quality Check)
- Decision frameworks for workload types, service types, namespaces, replicas
- Resource sizing reference tables for all common service types
- Security templates (RBAC, Network Policies, Secret rotation)
- Health check and scaling strategy templates
- 15-section standardized output format
- Quality checklist with 20+ verification checks
- 7 evaluation test cases with scoring rubric

## Kubernetes Concepts Covered

| Concept | Plan 1 | Plan 2 |
|---------|--------|--------|
| Pods | Yes | Yes |
| Deployments | Yes | Yes |
| StatefulSets | Yes | Yes |
| Services (ClusterIP) | Yes | Yes |
| Ingress | Yes | Yes |
| ConfigMaps | Yes | Yes |
| Secrets + Rotation | Yes | Yes |
| Resource Requests/Limits | Yes | Yes |
| Namespaces | Single | Multi (5) |
| RBAC (Roles/RoleBindings) | Yes | Yes (strict) |
| Network Policies | Basic | Zero-trust |
| HPA (Autoscaling) | Yes | Yes |
| Health Probes | Yes | Yes |
| PV/PVC (Storage) | Yes | Yes |
| Pod Security Standards | Yes | Yes (hardened) |
| Inter-Service Communication | Sync + Async | Sync + Async + Sandboxed |

## Plan Comparison

```
                    Plan 1 (Task Manager)     Plan 2 (AI Employee)
                    ─────────────────────     ────────────────────
Namespaces          1                         5
Services            7                         9
Secrets             7                         10
Security Model      Standard                  Zero-trust
Tool Execution      N/A                       Sandboxed
Audit Logging       Basic                     Immutable, append-only
Network Policies    Default deny ingress      Default deny ingress + egress
Total CPU           ~4 cores                  ~8 cores
Total Memory        ~6 GB                     ~14 GB
Total Storage       55 Gi                     190 Gi
```

## License

This project is submitted as part of the Panaversity Kubernetes course.
