# Multi-Tenant Fintech Platform — System Architecture

> **Note:** This repository contains no application code. It is an architecture reference document describing the system design, technical decisions, and infrastructure of a production multi-tenant financial SaaS platform I built as Lead Full-Stack Engineer.

---

## Table of Contents

- [Project Summary](#project-summary)
- [High-Level Architecture](#high-level-architecture)
- [Multi-Tenancy Design](#multi-tenancy-design)
- [Backend Architecture](#backend-architecture)
- [Authentication & Security](#authentication--security)
- [Real-Time Notification System](#real-time-notification-system)
- [Background Job Processing](#background-job-processing)
- [Frontend Architecture](#frontend-architecture)
- [Database Design](#database-design)
- [Infrastructure & DevOps](#infrastructure--devops)
- [Key Engineering Decisions](#key-engineering-decisions)
- [System Metrics](#system-metrics)

---

## Project Summary

A production multi-tenant SaaS platform for financial operations — built solo from architecture through deployment. The platform manages capital history, expense workflows, manager charges, closings, and real-time financial reporting for enterprise clients in the fund management space.

| Attribute | Detail |
|---|---|
| Type | Multi-tenant B2B SaaS |
| Tenants | 5–6 enterprise clients |
| Users | 200–300 active users per tenant |
| Transaction volume | ~50 financial transactions/month/tenant |
| Uptime since launch | 99%+ · zero unplanned downtime |
| Build model | Solo engineer — architecture to production |

**Stack:** FastAPI · SQLModel · PostgreSQL · Celery · Redis · Angular 18 · Tailwind CSS · Angular Material · WebSockets · Firebase · Azure VMs · Nginx · Alembic

---

## High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        CLIENT LAYER                             │
│                                                                 │
│   Angular 18 SPA          Firebase Push          Mobile Web    │
│   (browser)               (notifications)        (responsive)  │
└────────────────┬───────────────────┬────────────────────────────┘
                 │ HTTPS             │ FCM Push
                 ▼                   ▼
┌─────────────────────────────────────────────────────────────────┐
│                      NGINX REVERSE PROXY                        │
│              SSL Termination · Rate Limiting · Routing          │
└────────────────┬────────────────────────────────────────────────┘
                 │
       ┌─────────┴──────────┐
       │                    │
       ▼                    ▼
┌─────────────┐    ┌────────────────┐
│  FastAPI    │    │  WebSocket     │
│  REST API   │    │  Server        │
│  (Gunicorn) │    │  (FastAPI WS)  │
└──────┬──────┘    └───────┬────────┘
       │                   │
       ▼                   ▼
┌─────────────────────────────────────────────────────────────────┐
│                      SERVICE LAYER                              │
│                                                                 │
│   Auth Service    Tenant Router    Notification Engine          │
│   Report Engine   Audit Logger     Background Dispatcher        │
└────────┬────────────────┬──────────────────┬────────────────────┘
         │                │                  │
         ▼                ▼                  ▼
┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│  PostgreSQL  │  │    Redis     │  │    Celery    │
│  (per-tenant │  │  (cache +    │  │  (task       │
│   databases) │  │   broker)    │  │   workers)   │
└──────────────┘  └──────────────┘  └──────────────┘
```

---

## Multi-Tenancy Design

### Strategy: Separate Database Per Tenant

The platform uses a **separate PostgreSQL database per tenant** — the strictest form of multi-tenancy isolation. Each enterprise client has their own database instance with no shared tables or schemas.

```
┌─────────────────────────────────────────────┐
│              Application Layer              │
│         (single FastAPI codebase)           │
└──────┬──────────┬──────────────┬────────────┘
       │          │              │
       ▼          ▼              ▼
┌──────────┐ ┌──────────┐ ┌──────────┐
│ tenant_1 │ │ tenant_2 │ │ tenant_N │
│   (DB)   │ │   (DB)   │ │   (DB)   │
└──────────┘ └──────────┘ └──────────┘
```

**Why separate databases instead of shared schema?**

| Approach | Data Isolation | Risk | Complexity |
|---|---|---|---|
| Shared schema (row-level) | Low — a missing WHERE clause exposes all tenants | High | Low |
| Separate schema | Medium — schema-level isolation | Medium | Medium |
| **Separate database** ✅ | **Hard — infrastructure-level isolation** | **Low** | **Higher** |

For a financial platform, shared schema row-level isolation was ruled out immediately. A single missing filter in a query, a misconfigured ORM relationship, or a junior developer's mistake could expose one client's financial data to another. With separate databases, that class of bug is architecturally impossible.

**Tenant routing:**
```
Request → Extract tenant ID from JWT → Resolve database URL → Inject DB session
```

Each request carries a JWT containing the tenant ID. A FastAPI dependency resolves the correct database connection from a connection pool registry and injects it into the route handler. The route handler never directly references a database URL.

**Tenant provisioning flow:**
1. New tenant registered in the master registry database
2. A new PostgreSQL database is created for the tenant
3. Alembic migrations are applied to bring the schema to current version
4. An admin user is seeded with a temporary password
5. Tenant is live — their users can log in immediately

---

## Backend Architecture

### FastAPI + SQLModel

The API layer is built with **FastAPI** and **SQLModel** (Pydantic v2-native ORM). All request and response bodies are fully typed via Pydantic schemas — this means:
- Runtime type validation on all inputs — invalid requests are rejected before reaching business logic
- Auto-generated OpenAPI docs always reflect the actual schema
- IDE type-checking across the entire codebase catches errors before runtime

**Directory structure (simplified):**
```
app/
├── api/
│   ├── routes/
│   │   ├── auth.py
│   │   ├── capital.py
│   │   ├── expenses.py
│   │   ├── closings.py
│   │   ├── reports.py
│   │   └── notifications.py
│   └── dependencies/
│       ├── auth.py          # JWT validation + RBAC enforcement
│       └── database.py      # Tenant-aware DB session injection
├── core/
│   ├── security.py          # JWT, hashing, token rotation
│   ├── tenant.py            # Tenant resolution + DB routing
│   └── config.py
├── models/                  # SQLModel table definitions
├── schemas/                 # Pydantic request/response schemas
├── services/                # Business logic layer
├── tasks/                   # Celery task definitions
└── migrations/              # Alembic migration scripts
```

---

## Authentication & Security

### JWT + Refresh Token Rotation

```
Login Request
     │
     ▼
Validate credentials
     │
     ├──► Issue Access Token (short-lived: 15 min)
     │         │ Contains: user_id, tenant_id, roles, permissions
     │
     └──► Issue Refresh Token (long-lived: 7 days, stored in DB)
               │ Stored: hashed in DB, bound to user + device fingerprint
               │
               ▼
         On Refresh Request:
         Validate → Rotate (issue new pair) → Invalidate old refresh token
```

**Why token rotation?** If a refresh token is stolen and used, the next legitimate refresh from the real user will detect the mismatch (old token already invalidated), triggering a forced logout and security alert — limiting the attacker's window.

### Full-Stack RBAC

Permissions are enforced at three independent layers:

```
Request
  │
  ├─ 1. FastAPI route dependency: checks JWT roles against required permissions
  │      → 403 if insufficient → request never reaches handler
  │
  ├─ 2. Service layer: business logic checks ownership and resource-level permissions
  │      → Cannot approve your own transactions, cannot access other tenant data
  │
  └─ 3. Angular frontend: role-aware rendering
         → Action buttons, menu items, and data columns hidden for unauthorized roles
         → (Security enforcement is backend-side; frontend is UX only)
```

Roles are stored per-tenant in the database, not hardcoded — administrators can create custom roles with granular permission sets without a code deployment.

### Additional Security Measures
- Rate limiting middleware on all auth endpoints (login, refresh, password reset)
- bcrypt password hashing with per-user salts
- HTTP security headers via Nginx (HSTS, X-Frame-Options, CSP)
- OS-level hardening on Azure VMs: UFW firewall, SSH key-only, fail2ban

---

## Real-Time Notification System

### Architecture

```
Financial Event Occurs (e.g. transaction flagged)
        │
        ▼
  Service Layer publishes event to Redis channel
        │
        ├──► WebSocket Manager: pushes to connected user sessions
        │         └── Per-tenant channel isolation (tenants never share channels)
        │
        └──► Celery Task: sends Firebase Cloud Messaging push notification
                  └── For users not currently in the browser
```

### WebSocket Connection Lifecycle

```
User logs in
    │
    ▼
Angular opens WebSocket connection (authenticated via JWT query param)
    │
    ▼
Server registers connection in per-tenant connection registry (Redis)
    │
    ▼
Events published to this user's channels are pushed in real time
    │
    ▼
On disconnect/logout → connection deregistered, resources cleaned up
```

**Tenant isolation:** Each tenant's WebSocket connections are managed in a separate Redis key namespace. There is no mechanism by which a message intended for Tenant A could be delivered to a Tenant B session.

**Result:** Admin response time to flagged financial events dropped from several minutes to under 10 seconds.

---

## Background Job Processing

### Celery + Redis Task Queue

```
API Request
    │
    ├── Lightweight tasks → handled inline (< 50ms)
    │
    └── Heavy tasks → dispatched to Celery queue → API returns immediately
              │
              ▼
        Celery Worker picks up task
              │
              ├── Scheduled reports → generate PDF/data → store → notify user
              ├── Bulk exports      → process → upload to storage → send download link
              ├── Notification jobs → FCM dispatch → delivery confirmation
              └── Data aggregation  → pre-compute KPIs → cache results in Redis
```

**Reliability configuration:**
- Retry policy with exponential backoff on task failure (max 3 retries)
- Dead-letter queue for tasks that exhaust retries — visible in admin dashboard for manual review
- Task result backend in Redis — API can poll task status for long-running jobs
- Celery Beat for scheduled tasks (period-end report generation, daily summaries)

---

## Frontend Architecture

### Angular 18 SPA

```
app/
├── core/
│   ├── auth/           # JWT handling, token refresh interceptor, auth guard
│   ├── tenant/         # Tenant context service
│   └── interceptors/   # HTTP auth headers, error handling, loading state
├── features/
│   ├── dashboard/      # KPI cards, summary widgets, trend charts
│   ├── capital/        # Capital history, transactions, closings
│   ├── expenses/       # Expense management, approvals, categories
│   ├── reports/        # Parameterized report builder and viewer
│   ├── notifications/  # Real-time notification centre (WebSocket consumer)
│   └── admin/          # Tenant settings, user management, role assignment
└── shared/
    ├── components/     # Reusable UI components
    └── guards/         # RBAC route guards (mirrors backend permissions)
```

**Performance architecture:**
- Lazy-loaded feature modules — only the current route's bundle is loaded
- OnPush change detection throughout — re-renders only on explicit input changes
- API response pagination with virtual scrolling for large financial data tables
- Route resolvers pre-fetch critical data before route activation (no loading flash)
- Service workers for offline-capable static assets

**Result:** Sub-200ms interaction latency on all key views · Lighthouse performance score: 99

---

## Database Design

### Schema Principles

**1. Append-only financial records**
No committed financial transaction is ever hard-deleted or updated in place. All changes create a new versioned record:
```
capital_transactions
├── id
├── tenant_id
├── amount
├── transaction_type
├── status
├── created_by (user_id)
├── created_at
├── superseded_by (→ next version's id, null if current)
└── is_current (boolean flag for fast "current state" queries)
```

**2. Full audit log**
Every write operation across the platform is captured:
```
audit_log
├── id
├── tenant_id
├── actor_id (user who made the change)
├── action (CREATE / UPDATE / DELETE / APPROVE / REJECT)
├── resource_type (e.g. "capital_transaction")
├── resource_id
├── before_state (JSON snapshot)
├── after_state (JSON snapshot)
├── ip_address
└── timestamp
```

**3. Alembic migrations**
All schema changes are versioned, reversible migration scripts:
```
migrations/
├── env.py
└── versions/
    ├── 001_initial_schema.py
    ├── 002_add_rbac_tables.py
    ├── 003_add_audit_log.py
    ├── 004_add_notification_preferences.py
    └── ...
```
Each migration is tested against a production snapshot in staging before being applied to tenant databases.

---

## Infrastructure & DevOps

### Deployment Architecture

```
Internet
    │
    ▼
Azure VM (Ubuntu 22.04)
    │
    ├── Nginx (reverse proxy)
    │     ├── SSL termination (Let's Encrypt · auto-renewal via certbot)
    │     ├── HTTP → HTTPS redirect
    │     ├── Rate limiting (auth endpoints)
    │     └── Upstream: Gunicorn (FastAPI workers)
    │
    ├── Gunicorn
    │     └── FastAPI application (multiple worker processes)
    │
    ├── Celery Workers (systemd services)
    │     ├── Default queue worker
    │     └── Celery Beat (scheduler)
    │
    └── Redis (local instance · task broker + cache)

External:
    ├── PostgreSQL (per-tenant databases)
    └── Firebase Cloud Messaging (push notifications)
```

### Release Process (Manual Runbook)

```
Pre-deployment
1. git log → review commits since last tag
2. Generate changelog from commit messages
3. Run Alembic migration dry-run against production DB snapshot
4. Deploy to staging → run smoke test checklist

Deployment
5. git pull on production VM
6. Apply Alembic migrations (per tenant, sequentially)
7. Restart Gunicorn workers (graceful reload — zero dropped connections)
8. Restart Celery workers

Post-deployment
9. Run smoke test checklist (login, key workflows, WebSocket connection)
10. Monitor error logs for 15 minutes
11. Tag release in git

Rollback (if needed)
12. git checkout previous tag
13. Reverse Alembic migrations (if schema changed)
14. Restart services
15. Estimated RTO: < 10 minutes
```

### Backup & Recovery
- Azure VM snapshots: daily, retained for 7 days
- PostgreSQL logical backups: nightly pg_dump per tenant database
- Restore drills: run quarterly to validate recovery procedures and time RTO
- Tested recovery time: < 30 minutes for full tenant restore

---

## Key Engineering Decisions

| Decision | Option Chosen | Why |
|---|---|---|
| Multi-tenancy model | Separate DB per tenant | Hard financial data isolation; eliminates cross-tenant data leak risk at infrastructure level |
| ORM | SQLModel (Pydantic v2) | Single model definition for both DB schema and API schema — reduces duplication and drift |
| Task queue | Celery + Redis | Mature, well-documented; Redis doubles as cache — fewer moving parts than RabbitMQ |
| WebSocket framework | FastAPI native WS | No additional service needed; integrates with existing auth and dependency injection |
| Frontend framework | Angular 18 | Strong TypeScript support, built-in DI, enterprise-grade for complex role-based UIs |
| Deployment | Azure VMs (manual) | Client infrastructure preference; manual runbook provides full control without CI/CD complexity |
| Schema migrations | Alembic | Only mature migration tool for SQLAlchemy; versioned, reversible, per-tenant applicable |

---

## System Metrics

| Metric | Value |
|---|---|
| Active tenants | 5–6 enterprise clients |
| Users per tenant | 200–300 |
| Transactions per tenant/month | ~50 financial transactions |
| API response time (p95) | < 200ms |
| Frontend Lighthouse score | 99 |
| Real-time alert latency | < 10 seconds (was: several minutes) |
| Auth brute-force reduction | 90% after rate limiting |
| Uptime since launch | 99%+ |
| Unplanned downtime | Zero |
| Data incidents across tenants | Zero |

---

## About the Author

**Usama Ali** — Full-Stack Engineer  
📍 Pakistan · Available for remote contracts

[![LinkedIn](https://img.shields.io/badge/LinkedIn-0A66C2?style=flat-square&logo=linkedin&logoColor=white)](https://linkedin.com/in/usamaali012)
[![Email](https://img.shields.io/badge/Email-EA4335?style=flat-square&logo=gmail&logoColor=white)](mailto:usamaali012@gmail.com)
[![Upwork](https://img.shields.io/badge/Upwork-6FDA44?style=flat-square&logo=upwork&logoColor=white)](https://www.upwork.com/freelancers/usamaali012)
