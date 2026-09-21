#GENERATED AI GUIDE CONTENT 
# Multi-Tenant B2B SaaS Mock Analytics Tool — Expert Build Guide

**Stack:** Java 21 + Spring Boot 3.x (REST API) · Angular 17+ (standalone components) · PostgreSQL
**Audience:** Engineers building this project end-to-end, task by task
**Format:** Reference + step-by-step. Each task states a *Goal*, *Steps*, key *Code*, *Acceptance criteria*, and *Gotchas*.

> This is a "mock" analytics tool: the numbers are generated/seeded rather than pulled from a real event pipeline. That deliberately keeps the focus on the hard part of B2B SaaS — **tenant isolation, auth, and multi-tenant data modeling** — instead of on stream processing. Everything here is production-shaped, though: the patterns are the same ones you'd use with real data.

---

## Table of Contents

1. [What you are building](#1-what-you-are-building)
2. [Domain & feature scope](#2-domain--feature-scope)
3. [System architecture](#3-system-architecture)
4. [Multi-tenancy: the core decision](#4-multi-tenancy-the-core-decision)
5. [Data model](#5-data-model)
6. [Phase A — Environment & prerequisites](#phase-a--environment--prerequisites)
7. [Phase B — Backend scaffolding](#phase-b--backend-scaffolding)
8. [Phase C — Persistence & migrations](#phase-c--persistence--migrations)
9. [Phase D — Multi-tenancy infrastructure](#phase-d--multi-tenancy-infrastructure)
10. [Phase E — Authentication & authorization](#phase-e--authentication--authorization)
11. [Phase F — Core domain APIs](#phase-f--core-domain-apis)
12. [Phase G — Mock data generation](#phase-g--mock-data-generation)
13. [Phase H — Analytics query & aggregation APIs](#phase-h--analytics-query--aggregation-apis)
14. [Phase I — Angular scaffolding](#phase-i--angular-scaffolding)
15. [Phase J — Angular auth & tenant context](#phase-j--angular-auth--tenant-context)
16. [Phase K — Dashboard & charts](#phase-k--dashboard--charts)
17. [Phase L — Admin, users & settings](#phase-l--admin-users--settings)
18. [Phase M — Testing (incl. tenant-isolation tests)](#phase-m--testing)
19. [Phase N — Security hardening](#phase-n--security-hardening)
20. [Phase O — Observability](#phase-o--observability)
21. [Phase P — Packaging & deployment](#phase-p--packaging--deployment)
22. [Appendix 1 — API reference](#appendix-1--api-reference)
23. [Appendix 2 — Glossary](#appendix-2--glossary)
24. [Appendix 3 — Build order checklist](#appendix-3--build-order-checklist)

---

## 1. What you are building

A B2B SaaS product where **each customer is an organization (a "tenant")**. People sign in, land in their org's workspace, and see analytics dashboards scoped strictly to their org. No user can ever see another org's data — that guarantee is the whole product.

Concretely, a logged-in user can:

- Belong to one or more **organizations** with a **role** in each (Owner / Admin / Member / Viewer).
- Switch the "active" organization (like Slack workspace switching).
- View a **dashboard**: KPI tiles (active users, sessions, revenue, conversion), time-series charts, and top-N breakdown tables.
- Filter analytics by **date range** and **dimension** (channel, plan, country, device).
- Manage **members** (invite, change role, remove) if they are an Admin/Owner.
- Manage **tenant settings** (display name, timezone, data-retention window, feature flags).

Because the data is mocked, you also build:

- A **seed/generator** that fabricates realistic events per tenant (with per-tenant "shape" so dashboards look different).
- An admin-only **"regenerate data"** action for demos.

The engineering value — and what makes this a strong portfolio/interview piece — is that it forces you to solve tenant isolation, tenant-aware auth, tenant-scoped queries, and the operational trade-offs of multi-tenancy correctly.

---

## 2. Domain & feature scope

### 2.1 Bounded contexts

Keep the codebase organized around four contexts. Even in a modular monolith, these map to packages/folders.

| Context | Responsibility | Key entities |
|---|---|---|
| **Identity & Access** | Users, credentials, sessions, tokens | `User`, `Credential`, `RefreshToken` |
| **Tenancy** | Organizations, memberships, roles, invitations | `Organization`, `Membership`, `Invitation` |
| **Analytics** | Mock events + derived metrics | `AnalyticsEvent`, `MetricDaily` (rollup) |
| **Platform** | Cross-cutting: settings, feature flags, audit log | `TenantSettings`, `AuditLogEntry` |

### 2.2 In scope vs out of scope

**In scope:** tenant isolation, JWT auth with refresh, RBAC, org switching, invitations, mock data generation, time-series + breakdown analytics, both multi-tenancy strategies implemented behind one abstraction, tests proving isolation, Docker deployment.

**Out of scope (state as non-goals so scope stays sane):** billing/Stripe, SSO/SAML, real event ingestion pipeline, horizontal sharding, GDPR data-subject tooling, email delivery (log invitations instead of sending). Each of these is a clean follow-up.

### 2.3 Non-functional requirements

- **Isolation:** a bug should *fail closed* — if the tenant context is missing, queries return nothing / throw, never "all rows."
- **Auditability:** every write to tenant data carries `tenant_id`, `actor_user_id`, timestamp.
- **Performance target (mock scale):** dashboard endpoint < 300 ms P95 with pre-aggregated daily rollups.
- **Statelessness:** the API holds no session state; all context is derived per-request from the JWT + request.

---

## 3. System architecture

```
┌──────────────────────────────────────────────────────────────┐
│                        Angular SPA                             │
│  Standalone components · Signals · Router guards               │
│  ├─ AuthInterceptor  (attaches access token)                   │
│  ├─ TenantInterceptor(attaches X-Tenant-Id of active org)     │
│  └─ ErrorInterceptor (401 → refresh, 403 → toast)             │
└───────────────┬──────────────────────────────────────────────┘
                │ HTTPS (JSON)
                ▼
┌──────────────────────────────────────────────────────────────┐
│                    Spring Boot REST API                        │
│                                                                │
│  Filter chain:                                                 │
│   JwtAuthFilter → TenantResolutionFilter → SecurityContext     │
│                                                                │
│  Web layer      → Controllers (DTOs, validation)               │
│  Service layer  → business rules, @PreAuthorize, tx boundaries │
│  Tenant layer   → TenantContext (ThreadLocal), TenantResolver  │
│  Data layer     → Spring Data JPA + Hibernate                  │
│                    ├─ Strategy 1: @Filter row-level (tenant_id)│
│                    └─ Strategy 2: schema-per-tenant router     │
│  Migrations     → Flyway (shared + per-tenant)                 │
└───────────────┬──────────────────────────────────────────────┘
                │ JDBC
                ▼
┌──────────────────────────────────────────────────────────────┐
│                        PostgreSQL                              │
│   Strategy 1: one schema, tenant_id column (+ optional RLS)    │
│   Strategy 2: schema per tenant (tenant_acme, tenant_globex)   │
└──────────────────────────────────────────────────────────────┘
```

**Why a ThreadLocal `TenantContext`?** Spring's per-request threading model makes a `ThreadLocal` the natural place to stash "who is the current tenant" so that any layer — repository filters, Hibernate multi-tenant connection provider, audit interceptor — can read it without threading the value through every method signature. The critical rule: **set it in a filter, clear it in a `finally`**, and be aware of async boundaries (see [Phase D gotchas](#d-gotchas)).

---

## 4. Multi-tenancy: the core decision

You asked to cover both approaches and compare trade-offs. This section is the heart of the guide. You will implement **two** strategies behind a common abstraction and make the strategy switchable by config (`app.tenancy.strategy=DISCRIMINATOR|SCHEMA`). Being able to speak to both — and to why you'd pick one — is exactly what senior B2B interviews probe.

There are three classic models. You implement the first two; the third is documented for completeness.

### 4.1 Model 1 — Shared database, shared schema (discriminator column)

Every tenant-owned table has a `tenant_id` column. All tenants share tables. Isolation is enforced in the query layer (and optionally hardened with PostgreSQL **Row-Level Security**).

```
Table analytics_event
┌────┬───────────┬──────────┬───────┬───────────┐
│ id │ tenant_id │ event    │ value │ occurred  │
├────┼───────────┼──────────┼───────┼───────────┤
│ 1  │ acme      │ signup   │ 1     │ 2026-09-01│
│ 2  │ globex    │ purchase │ 49.00 │ 2026-09-01│  ← same table
│ 3  │ acme      │ session  │ 1     │ 2026-09-02│
└────┴───────────┴──────────┴───────┴───────────┘
```

**How isolation is enforced (defense in depth):**
1. **Application filter** — Hibernate `@Filter` auto-appends `WHERE tenant_id = :currentTenant` to every query on tenant entities.
2. **Database RLS (optional, strongest)** — a Postgres policy rejects rows where `tenant_id <> current_setting('app.tenant_id')`, so even a query that *forgot* the filter returns nothing.

**Pros**
- Cheapest to run: one connection pool, one schema, one migration run. Scales to thousands of small tenants.
- Easiest cross-tenant analytics for *your own* platform metrics ("total signups across all tenants").
- Simplest onboarding: creating a tenant is one `INSERT` into `organizations`.

**Cons**
- Isolation is a *policy*, not a *wall*. One missing `WHERE` clause = cross-tenant leak. This is the #1 real-world multi-tenant bug. RLS mitigates it but must be configured carefully.
- "Noisy neighbor": one huge tenant's data bloats shared indexes and can slow everyone.
- Per-tenant customization (extra columns, different indexes) is awkward.
- Per-tenant backup/restore/export is harder (you filter, you don't just dump a schema).

**Best for:** most SaaS, especially many small/medium tenants, and this project's default.

### 4.2 Model 2 — Shared database, schema per tenant

One PostgreSQL database, but each tenant gets its own **schema** (`tenant_acme`, `tenant_globex`), each containing the same set of tables with **no `tenant_id` column**. Isolation is achieved by routing each request's connection to `SET search_path = tenant_acme`.

```
Database: analytics
├── public              (shared: organizations, users, tenant registry)
├── tenant_acme
│   ├── analytics_event
│   └── metric_daily
└── tenant_globex
    ├── analytics_event
    └── metric_daily
```

**Pros**
- Much stronger isolation: a query physically cannot see another schema without switching `search_path`. Leaks require an explicit, obvious mistake.
- Per-tenant operations are clean: dump/restore/drop one schema; add a per-tenant index.
- Noisy-neighbor blast radius is smaller (separate tables/indexes per tenant).

**Cons**
- **Migrations multiply:** every schema change must run across N schemas. You need a per-tenant migration runner and it must handle new tenants and partially-migrated states.
- Connection/metadata overhead grows with tenant count; Postgres gets unhappy in the tens-of-thousands-of-schemas range.
- Cross-tenant platform analytics require iterating schemas (or a separate warehouse).
- Provisioning a tenant is heavier: create schema + run all migrations for it.

**Best for:** fewer, larger tenants; regulated industries; contracts demanding stronger isolation.

### 4.3 Model 3 — Database (or instance) per tenant (documented, not implemented)

Each tenant gets a separate database or even a separate DB server.

**Pros:** maximum isolation, per-tenant tuning/region/compliance, trivial per-tenant backup and "delete customer."
**Cons:** most expensive and operationally heavy; connection-pool-per-tenant; migrations across many DBs; provisioning is slow. Usually reserved for enterprise/"single-tenant deployment" tiers or as an escape hatch for your biggest customers.

Many mature SaaS run a **hybrid**: pooled shared-schema for the long tail of small tenants, and dedicated DB for a handful of enterprise accounts — using the same code because everything routes through one tenancy abstraction (which is exactly why you build that abstraction below).

### 4.4 Decision matrix

| Criterion | Shared schema (discriminator) | Schema per tenant | DB per tenant |
|---|---|---|---|
| Isolation strength | Policy-level (RLS hardens) | Strong (physical schema) | Strongest (physical DB) |
| Cost / density | Best (thousands/DB) | Medium | Worst |
| Migration effort | One run | N runs (per schema) | N runs (per DB) |
| Noisy-neighbor risk | Highest | Medium | Lowest |
| Per-tenant customization | Hard | Easier | Easiest |
| Cross-tenant reporting | Trivial | Harder | Hardest |
| Provisioning speed | Instant (INSERT) | Seconds (create+migrate) | Slow |
| Ops: backup/delete tenant | Filtered | Per-schema (clean) | Per-DB (cleanest) |
| Good default for | Most SaaS | Fewer/larger tenants | Enterprise tier |

### 4.5 What this project does

- **Default strategy:** shared schema with discriminator column + Hibernate filter, **plus PostgreSQL RLS** as the safety net. This is the industry default and demonstrates the subtle bug class most cleanly.
- **Second strategy:** schema-per-tenant, implemented via Hibernate's `MultiTenantConnectionProvider` + `CurrentTenantIdentifierResolver`, with a per-tenant Flyway runner.
- Both sit behind a `TenancyStrategy` abstraction and a single `TenantContext`, switchable with `app.tenancy.strategy`. Your controllers, services, and Angular app do not change when you flip it — only the data layer wiring does. That is the payoff of doing the abstraction first.

---

## 5. Data model

Shared/registry tables live in `public` (or the shared schema) under both strategies. Tenant-owned tables either carry `tenant_id` (Model 1) or live inside the tenant schema without it (Model 2).

### 5.1 Registry & identity (always shared)

```
organization
  id            UUID  PK
  slug          TEXT  UNIQUE   -- "acme", used in URLs/schema names
  display_name  TEXT
  schema_name   TEXT  NULL     -- populated only in SCHEMA strategy ("tenant_acme")
  status        TEXT           -- ACTIVE | SUSPENDED
  created_at    TIMESTAMPTZ

app_user
  id            UUID  PK
  email         TEXT  UNIQUE   -- global identity; a person is one user across orgs
  full_name     TEXT
  password_hash TEXT
  status        TEXT           -- ACTIVE | INVITED | DISABLED
  created_at    TIMESTAMPTZ

membership                     -- user ↔ org, with role (the join that grants access)
  id            UUID  PK
  user_id       UUID  FK -> app_user
  org_id        UUID  FK -> organization
  role          TEXT           -- OWNER | ADMIN | MEMBER | VIEWER
  created_at    TIMESTAMPTZ
  UNIQUE(user_id, org_id)

invitation
  id            UUID  PK
  org_id        UUID  FK
  email         TEXT
  role          TEXT
  token         TEXT UNIQUE
  expires_at    TIMESTAMPTZ
  accepted_at   TIMESTAMPTZ NULL

refresh_token
  id            UUID  PK
  user_id       UUID  FK
  token_hash    TEXT
  expires_at    TIMESTAMPTZ
  revoked_at    TIMESTAMPTZ NULL
```

### 5.2 Tenant-owned (analytics) tables

```
analytics_event               -- raw mock events
  id            BIGINT PK
  tenant_id     UUID           -- present in Model 1 only; absent in Model 2 schema
  event_type    TEXT           -- signup | session | purchase | churn | pageview
  channel       TEXT           -- organic | paid | referral | email
  plan          TEXT           -- free | pro | enterprise
  country       TEXT
  device        TEXT           -- desktop | mobile | tablet
  value_num     NUMERIC(12,2)  -- revenue for purchase, else 1
  user_ref      TEXT           -- pseudo end-user id (for DAU/retention)
  occurred_at   TIMESTAMPTZ

metric_daily                  -- pre-aggregated rollup (drives fast dashboards)
  id            BIGINT PK
  tenant_id     UUID           -- Model 1 only
  day           DATE
  metric        TEXT           -- active_users | sessions | revenue | signups | conversion
  dimension     TEXT           -- '' for total, else channel/plan/country/device value
  dim_key       TEXT           -- which dimension the value belongs to
  value_num     NUMERIC(14,2)
  UNIQUE(tenant_id, day, metric, dimension, dim_key)

tenant_settings
  tenant_id     UUID PK
  timezone      TEXT
  retention_days INT
  feature_flags JSONB

audit_log
  id            BIGINT PK
  tenant_id     UUID
  actor_user_id UUID
  action        TEXT
  target        TEXT
  metadata      JSONB
  created_at    TIMESTAMPTZ
```

> **Design note:** `metric_daily` is the key performance move. Dashboards query the small rollup table, not millions of raw events. The generator writes both raw events (for drill-down/authenticity) and the daily rollup (for speed). In Model 2, drop `tenant_id` from `analytics_event`, `metric_daily`, `tenant_settings`, and `audit_log` — the schema *is* the tenant.

---
