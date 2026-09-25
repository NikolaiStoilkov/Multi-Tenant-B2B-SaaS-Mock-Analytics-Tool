# Multi-Tenant B2B Endpoint Analytics Platform

**Stack:** Java 21 desktop **agent** (JNA / native Windows event hooks) · Java 21 + Spring Boot 3.x ingestion & analytics **API** · Angular 17+ **dashboard** · PostgreSQL
**Audience:** Engineers building this project end-to-end, task by task
**Format:** Reference + step-by-step. Each task states a *Goal*, *Steps*, key *Code*, *Acceptance criteria*, and *Gotchas*.

> **What this is.** A B2B SaaS **workforce / endpoint activity-analytics** product. A lightweight Java agent runs on an organization's **managed Windows workstations**, collects **activity telemetry** (see [§2](#2-what-the-agent-collects-and-what-it-does-not)), and streams it to a multi-tenant Spring Boot API. Each customer is an isolated tenant; admins in that tenant see dashboards of aggregate device and application usage for the fleet **they own**.
>
> This is the endpoint-analytics equivalent of tools like ActivTrak, Nexthink, or a Digital-Employee-Experience (DEX) platform — used for IT asset/software inventory, license optimization, productivity analytics, and security signals (e.g. unexpected software installs).

---

## ⚠️ Responsible-use contract (read first — it shapes the design)

This platform is built for **transparent, consent-based monitoring of organization-managed devices only.** These are not optional add-ons; they are baked into the architecture below.

1. **Org-managed devices only.** The agent is deployed by an organization's IT (via MDM/GPO/Intune) onto devices the organization owns or manages. It is not for putting on someone else's personal machine.
2. **Disclosure & consent.** Monitored users must be informed. The agent ships with a visible tray presence, an in-product "what we collect" notice, and an install manifest — it is **not** designed to be hidden, disguised, or to evade security software. Deployments must comply with local law (e.g. works-council agreements, GDPR/employee-data rules, US state notice laws).
3. **Metadata, not content, by default.** The agent captures *activity signals* — that a key/mouse was active, which app had focus, that an installer ran — **not keystroke content, passwords, message bodies, or screen contents.** Any richer collection is off by default, gated behind explicit tenant configuration, and surfaced in the disclosure notice.
4. **Tenant isolation is the security boundary.** One customer can never see another's fleet data. This is the whole product (see [§5](#5-multi-tenancy-the-core-decision)).
5. **Retention & deletion.** Every tenant has a configurable retention window; data ages out automatically, and an admin can purge a device or user on request.

If a requirement conflicts with the above, the above wins.

---

## Table of Contents

1. [What you are building](#1-what-you-are-building)
2. [What the agent collects (and what it does not)](#2-what-the-agent-collects-and-what-it-does-not)
3. [Domain & feature scope](#3-domain--feature-scope)
4. [System architecture](#4-system-architecture)
5. [Multi-tenancy: the core decision](#5-multi-tenancy-the-core-decision)
6. [Data model](#6-data-model)
7. [The Java agent](#7-the-java-agent)
8. [Build order checklist](#8-build-order-checklist)

---

## 1. What you are building

Three deployables that share one tenancy model:

| Component | Runtime | Responsibility |
|---|---|---|
| **Agent** | Java 21 desktop app on managed Windows PCs | Hook OS activity events, batch them, ship them to the API over mTLS/JWT. Runs as a Windows service with a visible tray indicator. |
| **API** | Spring Boot 3.x | Authenticate agents & humans, resolve tenant, ingest telemetry, roll it up, serve analytics. |
| **Dashboard** | Angular 17+ SPA | Tenant admins view fleet dashboards, device inventory, software inventory, and activity trends. |

A logged-in **tenant admin** can:

- Enroll devices (issue an enrollment token; the agent redeems it for a device identity + credentials).
- See a **fleet dashboard**: active devices, aggregate active-time, top applications by focus time, software-install events, session/lock patterns.
- Drill into a **single device**: timeline of app focus, install events, uptime.
- Manage the **collection policy** for the tenant (which signal categories are enabled).
- Manage **members** (Owner / Admin / Analyst / Viewer) and **tenant settings** (timezone, retention window, disclosure text shown to end users).

Because building a real fleet is impractical for development, the project also ships a **mock telemetry generator** that fabricates realistic per-tenant device streams — so you can build and demo the dashboards without deploying agents. The wire format the generator emits is identical to the agent's, so the ingestion path is exercised the same way in both cases.

---

## 2. What the agent collects (and what it does not)

The agent collects **activity metadata** — signals *about* usage, not the content of what the user typed, read, or saw. Collection is organized into categories; each is independently toggleable by the tenant's collection policy and reflected in the end-user disclosure notice.

### Collected (metadata)

| Category | Example signal | Windows source |
|---|---|---|
| **Input activity** | "input was active for 45 of the last 60 s" — an activity *level*, no keycodes/content | Low-level `WH_KEYBOARD_LL` / `WH_MOUSE_LL` hooks reduced to counts (via JNA) |
| **App focus** | Foreground window changed to `EXCEL.EXE` at 14:03 | `SetWinEventHook` + `EVENT_SYSTEM_FOREGROUND` |
| **Process lifecycle** | Process `Teams.exe` started / stopped | WMI `__InstanceCreationEvent` / ETW |
| **Software install** | MSI/installer executed; program added to Add/Remove Programs | WMI `Win32_Product`, registry `Uninstall` key watch |
| **Session state** | Logon, lock, unlock, idle→active | `WTSRegisterSessionNotification` |
| **Device inventory** | OS build, hostname, hardware id (hashed) | WMI / `GetSystemInfo` |

### Not collected (by default, and never covertly)

- **No keystroke content / keylogging of text, passwords, or credentials.**
- **No screenshots or screen recording.**
- **No clipboard, file, email, or message contents.**
- **No browser history or URL content** (an optional, disclosed "active-domain" category can be enabled by policy, capturing domain only — not full URLs or page content).
- **No microphone or camera.**

The agent is visible: a system-tray icon, an entry in Add/Remove Programs, a signed service, and an in-agent "What is collected" screen. It does not attempt to hide from Task Manager or antivirus. A user-facing kill switch / pause is available where the tenant's policy and local law require it.

> **Why metadata-first?** It delivers the actual B2B value (software inventory, license usage, DEX/productivity trends, unexpected-install alerts) while keeping the data proportionate and defensible. Deep content capture is where legitimate monitoring becomes surveillance; this project deliberately stays on the metadata side of that line.

---

## 3. Domain & feature scope

### 3.1 Bounded contexts

| Context | Responsibility | Key entities |
|---|---|---|
| **Identity & Access** | Human users, credentials, sessions, tokens | `User`, `Credential`, `RefreshToken` |
| **Tenancy** | Organizations, memberships, roles, invitations | `Organization`, `Membership`, `Invitation` |
| **Fleet** | Enrolled devices & their agents | `Device`, `EnrollmentToken`, `AgentCredential` |
| **Telemetry** | Ingested events + derived metrics | `TelemetryEvent`, `MetricDaily` (rollup) |
| **Platform** | Cross-cutting: collection policy, settings, disclosure, audit | `CollectionPolicy`, `TenantSettings`, `AuditLogEntry` |

### 3.2 In scope vs out of scope

**In scope:** tenant isolation, JWT auth (humans) + mTLS/token auth (agents), device enrollment, RBAC, collection-policy management, telemetry ingestion with batching & offline buffering, time-series + breakdown analytics, both multi-tenancy strategies behind one abstraction, isolation tests, Docker deployment, a mock telemetry generator.

**Out of scope (non-goals):** billing, SSO/SAML, real-time streaming pipeline (batch ingest is fine), content-level capture, cross-platform agents (Windows only here — macOS/Linux agents are a clean follow-up), mobile devices.

### 3.3 Non-functional requirements

- **Isolation:** fail *closed* — missing tenant context ⇒ no rows, never "all rows."
- **Auditability:** every write carries `tenant_id`, `actor` (human or `device_id`), timestamp.
- **Agent resilience:** buffer to local disk when offline; retry with backoff; never block the user's machine; bounded CPU/RAM budget.
- **Ingest throughput:** accept batched events; dashboards read pre-aggregated rollups (< 300 ms P95).
- **Statelessness (API):** all context derived per-request from JWT/mTLS identity.

---

## 4. System architecture

```
   Managed Windows workstations (one tenant's fleet)
┌───────────────────────────────────────────────────────────┐
│  Java Agent (Windows service + tray)                        │
│   ├─ Event hooks (JNA): foreground, input-level, session    │
│   ├─ WMI/ETW watchers: process + software-install events    │
│   ├─ Collector → normalizer → local buffer (offline-safe)   │
│   └─ Uploader (mTLS + agent JWT) ── batches ──┐             │
└───────────────────────────────────────────────┼───────────┘
                                                 │ HTTPS (JSON batches)
                                                 ▼
┌───────────────────────────────────────────────────────────┐
│                   Spring Boot API                           │
│  Filter chain:                                              │
│   AuthFilter(JWT human | mTLS agent) → TenantResolutionFilter│
│  Ingest layer   → /agent/telemetry (validate, dedupe, store)│
│  Web layer      → Controllers (DTOs, validation)            │
│  Service layer  → rules, @PreAuthorize, tx boundaries       │
│  Tenant layer   → TenantContext (ThreadLocal), TenantResolver│
│  Rollup         → raw events → metric_daily (scheduled)     │
│  Data layer     → Spring Data JPA + Hibernate               │
│                    ├─ Strategy 1: row-level (tenant_id)     │
│                    └─ Strategy 2: schema-per-tenant router  │
│  Migrations     → Flyway (shared + per-tenant)              │
└───────────────┬─────────────────────────┬───────────────────┘
                │ JDBC                      │ HTTPS (JSON)
                ▼                           ▼
┌────────────────────────┐   ┌────────────────────────────────┐
│      PostgreSQL         │   │        Angular SPA              │
│  Strategy 1: tenant_id  │   │  Fleet & device dashboards,     │
│  Strategy 2: schema/ten │   │  software inventory, policy UI  │
└────────────────────────┘   └────────────────────────────────┘
```

**Two identity types.** Humans authenticate with a JWT (login → access/refresh). Agents authenticate with a per-device credential (mTLS client cert and/or a signed device token) issued at enrollment. Both resolve to a `tenant_id` before any query runs. The `TenantResolutionFilter` reads it from the human's active-org claim or the agent's device identity — never from a client-supplied header alone.

**Why a ThreadLocal `TenantContext`?** Spring's per-request threading makes a `ThreadLocal` the natural place to stash the current tenant so every layer (repository filters, Hibernate multi-tenant connection provider, audit interceptor) reads it without threading it through method signatures. The rule: **set it in a filter, clear it in `finally`**, and mind async boundaries.

---

## 5. Multi-tenancy: the core decision

You implement **two** strategies behind a common abstraction, switchable by config (`app.tenancy.strategy=DISCRIMINATOR|SCHEMA`). Being able to speak to both — and why you'd pick one — is exactly what senior B2B interviews probe. Here a "tenant" is a **customer organization**, and its telemetry is the fleet's device events.

### 5.1 Model 1 — Shared database, shared schema (discriminator column)

Every tenant-owned table has a `tenant_id` column; all tenants share tables. Isolation is enforced in the query layer and optionally hardened with PostgreSQL **Row-Level Security**.

```
Table telemetry_event
┌────┬───────────┬───────────┬──────────────┬───────────┬────────────┐
│ id │ tenant_id │ device_id │ event_type   │ app       │ occurred   │
├────┼───────────┼───────────┼──────────────┼───────────┼────────────┤
│ 1  │ acme      │ dev-01    │ app_focus    │ EXCEL.EXE │ 2026-09-01 │
│ 2  │ globex    │ dev-77    │ sw_install   │ Zoom.msi  │ 2026-09-01 │  ← same table
│ 3  │ acme      │ dev-01    │ session_lock │ —         │ 2026-09-02 │
└────┴───────────┴───────────┴──────────────┴───────────┴────────────┘
```

**Isolation (defense in depth):**
1. **Application filter** — Hibernate `@Filter` auto-appends `WHERE tenant_id = :currentTenant` on every tenant entity.
2. **Database RLS (optional, strongest)** — a Postgres policy rejects rows where `tenant_id <> current_setting('app.tenant_id')`, so even a forgotten filter returns nothing.

**Pros:** cheapest to run (one pool, one schema, one migration run); scales to thousands of small tenants; trivial cross-tenant platform metrics; onboarding is one `INSERT`.
**Cons:** isolation is a *policy* not a *wall* (one missing `WHERE` = leak — the #1 real multi-tenant bug; RLS mitigates); noisy-neighbor on shared indexes; per-tenant customization/backup awkward.
**Best for:** most SaaS, many small/medium tenants — this project's **default**.

### 5.2 Model 2 — Shared database, schema per tenant

One database; each tenant gets its own schema (`tenant_acme`, `tenant_globex`) with the same tables and **no `tenant_id` column**. Isolation via `SET search_path = tenant_acme` per request.

```
Database: analytics
├── public              (shared: organizations, users, device registry)
├── tenant_acme
│   ├── telemetry_event
│   └── metric_daily
└── tenant_globex
    ├── telemetry_event
    └── metric_daily
```

**Pros:** stronger isolation (a query physically can't see another schema); clean per-tenant dump/restore/drop; smaller noisy-neighbor blast radius.
**Cons:** migrations multiply (run across N schemas; handle new/partially-migrated tenants); metadata overhead grows with tenant count; cross-tenant reporting needs schema iteration; heavier provisioning.
**Best for:** fewer, larger tenants; regulated industries; contracts demanding stronger isolation.

### 5.3 Model 3 — Database (or instance) per tenant (documented, not implemented)

Each tenant gets a separate DB/server. **Pros:** maximum isolation, per-tenant tuning/region/compliance, trivial per-tenant backup/delete. **Cons:** most expensive and operationally heavy. Usually reserved for enterprise/"single-tenant deployment" tiers. Mature SaaS often run a **hybrid** (pooled shared-schema for the long tail, dedicated DB for enterprise) — using the same code because everything routes through one tenancy abstraction, which is why you build that abstraction first.

### 5.4 Decision matrix

| Criterion | Shared schema | Schema per tenant | DB per tenant |
|---|---|---|---|
| Isolation strength | Policy-level (RLS hardens) | Strong (physical schema) | Strongest (physical DB) |
| Cost / density | Best | Medium | Worst |
| Migration effort | One run | N runs | N runs |
| Noisy-neighbor risk | Highest | Medium | Lowest |
| Per-tenant customization | Hard | Easier | Easiest |
| Cross-tenant reporting | Trivial | Harder | Hardest |
| Provisioning speed | Instant | Seconds | Slow |
| Good default for | Most SaaS | Fewer/larger tenants | Enterprise tier |

### 5.5 What this project does

- **Default:** shared schema + Hibernate filter, **plus PostgreSQL RLS** as the safety net.
- **Second:** schema-per-tenant via Hibernate `MultiTenantConnectionProvider` + `CurrentTenantIdentifierResolver`, with a per-tenant Flyway runner.
- Both sit behind a `TenancyStrategy` abstraction and one `TenantContext`, flipped with `app.tenancy.strategy`. Controllers, services, the ingest endpoint, and the Angular app don't change when you flip it — only the data-layer wiring does.

---

## 6. Data model

Shared/registry tables live in `public` under both strategies. Tenant-owned tables carry `tenant_id` (Model 1) or live inside the tenant schema without it (Model 2).

### 6.1 Registry & identity (always shared)

```
organization
  id UUID PK · slug TEXT UNIQUE · display_name TEXT
  schema_name TEXT NULL           -- SCHEMA strategy only ("tenant_acme")
  status TEXT                     -- ACTIVE | SUSPENDED
  created_at TIMESTAMPTZ

app_user
  id UUID PK · email TEXT UNIQUE · full_name TEXT
  password_hash TEXT · status TEXT · created_at TIMESTAMPTZ

membership                        -- user ↔ org + role
  id UUID PK · user_id FK · org_id FK
  role TEXT                       -- OWNER | ADMIN | ANALYST | VIEWER
  UNIQUE(user_id, org_id)

device                            -- an enrolled, managed workstation
  id UUID PK · org_id FK
  hostname TEXT · os_build TEXT
  hardware_hash TEXT              -- hashed, not raw serials
  agent_version TEXT · status TEXT-- ENROLLED | SUSPENDED | RETIRED
  last_seen_at TIMESTAMPTZ · enrolled_at TIMESTAMPTZ

enrollment_token                  -- redeemed once to enroll a device
  id UUID PK · org_id FK · token_hash TEXT
  expires_at TIMESTAMPTZ · redeemed_at TIMESTAMPTZ NULL

invitation · refresh_token        -- (as in a standard auth setup)
```

### 6.2 Tenant-owned (telemetry) tables

```
telemetry_event                   -- normalized activity metadata
  id BIGINT PK
  tenant_id UUID                  -- Model 1 only
  device_id UUID
  event_type TEXT                 -- app_focus | input_activity | process_start
                                  -- | process_stop | sw_install | session_lock
                                  -- | session_unlock | logon | idle
  app TEXT NULL                   -- process/app name where relevant
  detail JSONB NULL               -- category-specific metadata (never content)
  value_num NUMERIC(12,2) NULL    -- e.g. active-seconds, duration
  occurred_at TIMESTAMPTZ         -- client time
  received_at TIMESTAMPTZ         -- server time (for late/offline batches)

metric_daily                      -- pre-aggregated rollup (fast dashboards)
  id BIGINT PK · tenant_id UUID (Model 1)
  day DATE · metric TEXT          -- active_devices | active_minutes
                                  -- | app_focus_minutes | installs | sessions
  dimension TEXT · dim_key TEXT   -- '' for total, else app/device/category
  value_num NUMERIC(14,2)
  UNIQUE(tenant_id, day, metric, dimension, dim_key)

collection_policy                 -- which categories the agent may collect
  tenant_id UUID PK
  categories JSONB                -- {input_activity:true, app_focus:true,
                                  --  process:true, sw_install:true, active_domain:false}
  disclosure_text TEXT            -- shown to end users in the agent

tenant_settings
  tenant_id UUID PK · timezone TEXT · retention_days INT · feature_flags JSONB

audit_log
  id BIGINT PK · tenant_id UUID
  actor_user_id UUID NULL · actor_device_id UUID NULL
  action TEXT · target TEXT · metadata JSONB · created_at TIMESTAMPTZ
```

> **Design note:** `metric_daily` is the key performance move — dashboards read the small rollup, not millions of raw events. A scheduled job rolls raw `telemetry_event` rows into `metric_daily`. In Model 2, drop `tenant_id` from the tenant-owned tables — the schema *is* the tenant.

---

## 7. The Java agent

### 7.1 Responsibilities

1. **Enroll** once: redeem an enrollment token → receive a device identity + client credential (mTLS cert / signed token), stored in the Windows credential store / DPAPI.
2. **Collect** the categories the tenant policy allows, via native hooks (JNA):
   - Foreground-window changes → `SetWinEventHook(EVENT_SYSTEM_FOREGROUND)`.
   - Input *activity level* → low-level hooks reduced to per-interval counts (no keycodes).
   - Process start/stop → WMI event subscriptions / ETW.
   - Software installs → WMI `Win32_Product` + registry `Uninstall`-key watcher.
   - Session/lock/idle → `WTSRegisterSessionNotification` + `GetLastInputInfo`.
3. **Normalize** each raw OS event into the shared `TelemetryEvent` wire format (same JSON the mock generator emits).
4. **Buffer** to a bounded local store (e.g. an embedded queue / SQLite) so nothing is lost offline; flush oldest-first with size/age caps.
5. **Upload** in batches over HTTPS with mTLS + agent token; exponential backoff; idempotency key per batch so retries dedupe server-side.
6. **Be visible & controllable:** tray icon, "what we collect" screen reflecting the live policy, version/heartbeat, and (where required) a pause control. Pulls policy updates from the API so an admin disabling a category takes effect fleet-wide.

### 7.2 Constraints (make these acceptance criteria)

- **Bounded footprint:** cap CPU and memory; hooks must never block the UI thread — offload to a worker and drop-with-counter under backpressure rather than stall.
- **No content capture:** enforce in code that `input_activity` carries counts/durations only. Add a unit test asserting no keycode/character ever reaches a `TelemetryEvent`.
- **Fail safe, not loud:** if the API is unreachable, buffer and retry — never pop errors at the end user.
- **Signed & updatable:** the service binary is code-signed; support in-place agent updates.
- **Honors policy:** disabling a category in the tenant policy stops that collection within one policy-refresh interval; prove it with a test.

### 7.3 Suggested modules

```
agent/
  agent-core       -- collector interfaces, TelemetryEvent model, buffer, uploader
  agent-win        -- JNA bindings: WinEventHook, WMI, session notifications
  agent-service    -- Windows service wrapper, tray UI, disclosure screen
  agent-testkit    -- fake event sources for tests + the mock generator
```

---

## 8. Build order checklist

1. **Contracts first.** Define the `TelemetryEvent` JSON schema and the `/agent/telemetry` batch endpoint. Build the **mock generator** against it so the whole backend can be developed with zero real agents.
2. **Tenancy abstraction.** `TenantContext`, `TenantResolutionFilter`, `TenancyStrategy` — Model 1 first, RLS second.
3. **Identity & enrollment.** Human JWT auth + RBAC; device enrollment tokens → agent credentials; agent auth path.
4. **Ingestion.** Validate + dedupe (idempotency key) + store batches; write raw events with `tenant_id`/`device_id`.
5. **Rollups & analytics.** Scheduled `telemetry_event → metric_daily`; time-series + breakdown query APIs.
6. **Dashboard.** Angular fleet dashboard, device drill-down, software inventory, collection-policy editor, member/settings admin.
7. **Isolation tests.** Prove tenant A can never read tenant B (both strategies), and that a missing tenant context returns nothing.
8. **The real agent.** Build `agent-win` hooks; make its output byte-compatible with the mock generator; verify buffering/offline/backoff and the no-content-capture test.
9. **Second tenancy strategy.** Implement schema-per-tenant behind the same abstraction; flip via config and re-run the isolation suite.
10. **Packaging.** Docker for API + Postgres; signed installer/service for the agent; Flyway migrations (shared + per-tenant).
