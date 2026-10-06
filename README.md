*This project has been created as part of the 42 curriculum by hassende, drahwanj, <LOGIN-3>, <LOGIN-3>.
<!-- TODO(team): replace every <LOGIN-N> with a real 42 intra login (and adjust the count to the roster) before evaluation. -->

# HrmSystem

## Description

**HrmSystem** is a multi-tenant HR (Human Resources) SaaS platform: one deployment serves many
companies, each in a fully isolated workspace. A company registers itself from the public landing
page, its founder becomes the workspace administrator, and the team then manages employees,
departments, attendance, leave approvals, live analytics, announcements and real-time chat —
with role-based permissions on every action.

Key features:

- **Self-service signup** — landing page creates tenant + admin account atomically, auto-login.
- **Core HRM** — employees, departments, attendance (clock-in/out), leave request workflow.
- **Live analytics dashboard** — KPIs and charts that refresh over SignalR, CSV/PDF export.
- **Real-time everywhere** — chat, notification bell, status page, import progress (SignalR).
- **Admin panel** — provision users, change roles, suspend/reinstate, role→permission matrix.
- **Data export/import** — JSON/CSV/XML exports, background CSV imports with live progress.
- **Ops** — health checks, anonymous status page, automated daily SQL backups + DR runbook.

## Instructions

Prerequisites: **Docker** (with Compose). For local development additionally: **.NET 10 SDK**.

```powershell
cp .env.example .env        # fill in the SA password and JWT key placeholders
docker compose up -d --build
```

| URL | What |
| --- | --- |
| https://localhost:5100 | Blazor client — nginx terminates TLS (self-signed cert: accept the one-time browser warning) and proxies `/api` + `/hubs` to the API (same origin, no CORS) |
| http://localhost:5180 | Plain HTTP — permanently redirects to the HTTPS origin |
| https://localhost:5100/status | Anonymous status page (live SignalR updates) |
| http://localhost:5000/health | API liveness, container-internal (readiness at `/health/ready`) |
| http://localhost:8082 | Seq (logs + OTLP traces) |

### Demo accounts (Development seed)

Password for all: `Seed1234`. Tenant slugs: `acme`, `globex`, `initech`.

| Identifier | Role |
| --- | --- |
| `host-admin` | Platform admin (no tenant — sees zero tenant rows by design) |
| `admin-acme` | Tenant admin |
| `manager-acme` | Manager |
| `employee-acme` | Employee |

The stack ships with `ASPNETCORE_ENVIRONMENT=Development` on purpose: it enables the full Bogus
demo seed so the evaluation starts from a populated system. Production would seed from
configuration only.

### Local development (without full Docker)

```powershell
docker compose up -d hrmsystem.sqlserver hrmsystem.redis hrmsystem.seq
dotnet run --project src/Web/HrmSystem.Web        # API
dotnet run --project src/Client/HrmSystem.Client  # WASM dev server on http://localhost:5217
```

### Tests

```powershell
dotnet test                                                  # everything
dotnet test tests/HrmSystem.ArchitectureTests                # layer + tenant-safety conventions
dotnet test tests/HrmSystem.Infrastructure.IntegrationTests  # needs Docker (Testcontainers)
```

## Team Information

| Member | Role(s) | Responsibilities |
| --- | --- | --- |
| <LOGIN-1> | Product Owner / Developer | Product vision and backlog, feature priorities, validates completed work (demos), speaks for the product during evaluation |
| <LOGIN-2> | Project Manager / Scrum Master / Developer | Planning and task tracking, deadlines, removes blockers, keeps the audit files in the repository current |
| <LOGIN-3> | Technical Lead / Developer | Architecture oversight and code review, deployment and CI, release checks |

## Project Management

- Work was split by **vertical feature slices** (e.g. "leave requests end-to-end" = domain +
  application + API + Blazor page), so each member owns features rather than layers.
- Task tracking: issues list + the living module map in
  `docs/ft_transcendence-modules-and-points.md` + the audit files in the repository root.
- Communication: team chat channel + in-person syncs.

## Technical Stack

| Choice | Why |
| --- | --- |
| **ASP.NET Core (.NET 10)** backend | Mature framework with first-class DI, middleware, Identity and SignalR; enables the Clean Architecture split (Domain → Application → Infrastructure → Web) |
| **Blazor WebAssembly** frontend | Single language (C#) across the stack, real SPA with client-side routing, strong typing shared with API models |
| **SQL Server 2022** | Relational integrity for a data-model-heavy HR domain; first-class EF Core support; backup/restore tooling used by the DR module |
| **EF Core 10 (writes) + Dapper (reads)** | EF gives change tracking, interceptors and global query filters (tenant isolation); Dapper gives lean, fast read models for analytics — each tool where it is strongest |
| **SignalR** | WebSockets with automatic fallback + reconnect for chat, notifications, live analytics and status |
| **Redis** | Cache for analytics read models |
| **nginx** | Serves the published WASM app, terminates TLS (HTTPS), reverse-proxies `/api` + `/hubs` so the browser sees one origin |
| **Quartz.NET** | Scheduled jobs: automated database backups, retention, CSV import processing |
| **Seq + OpenTelemetry** | Traces/logs for debugging and the observability story |

## Database Schema

Tenant-scoped tables all carry a `TenantId` and are guarded by EF global query filters + a write
interceptor (see the custom module justification below).

```
Tenant 1 ──── n  Employee ── n Attendance
   │                │  └───── n LeaveRequest
   │                └── n ── 1 Department (1 Department : n Employees)
   ├──── n  Department
   ├──── n  Announcement
   ├──── n  ChatMessage        (sender/recipient are Users)
   ├──── n  ImportJob
   └──── n  User (domain)  ── 1 AspNetUsers (Identity: hashed+salted PBKDF2 passwords)

Host-level (NOT tenant-scoped): Tenant, BackupHistory, HealthCheckSnapshot, Identity tables.
```

Details: EF configurations under
`src/Infrastructure/HrmSystem.Infrastructure/Persistence/Data(base)?/Configurations`, decisions in
`docs/architecture.md`.

## Features List

| Feature | Member(s) | Description |
| --- | --- | --- |
| Company signup + auth (JWT + refresh) | ahirzallah | Landing registration, login, token refresh, rate-limited auth endpoints |
| Employees & departments CRUD | <LOGIN-?> | Paged/filtered directory, hire/edit/terminate, department management |
| Attendance | <LOGIN-?> | Clock-in/out, per-employee history, worked-hours math |
| Leave requests | <LOGIN-?> | Submit → approve/reject workflow, overlap guard, self-approval forbidden |
| Analytics dashboard | <LOGIN-?> | Live KPIs + 4 chart types, date filters, CSV/PDF export |
| Chat & announcements | <LOGIN-?> | 1:1 real-time DMs, unread badges; tenant-wide announcements with live bell |
| Admin panel | <LOGIN-?> | User provisioning, role changes, suspend/reinstate, permission matrix |
| Import/export | <LOGIN-?> | Content-negotiated exports; background CSV import with live progress |
| Health & status + backups | <LOGIN-?> | Probes, public status page, Quartz backups + DR runbook |

## Modules

Points: **Major = 2, Minor = 1 — minimum required: 14.**

| # | Module | Type | Pts | Evidence |
| --- | --- | --- | --- | --- |
| 1 | Frontend + backend framework (Blazor WASM + ASP.NET Core) | Major | 2 | Whole stack |
| 2 | Real-time features — WebSockets (SignalR) | Major | 2 | `/hubs/notifications`, `/hubs/analytics`, `/hubs/status`, chat, import progress |
| 3 | ORM for the database (EF Core 10) | Minor | 1 | All writes via repositories + UnitOfWork |
| 4 | Advanced permissions system | Major | 2 | 4 roles, role→permission matrix, per-endpoint authorization, admin panel |
| 5 | Advanced analytics dashboard with data visualization | Major | 2 | ApexCharts, date filters, SignalR refresh, CSV/PDF export, Redis cache |
| 6 | Data export and import | Minor | 1 | JSON/CSV/XML export, background CSV import |
| 7 | Health check + status page + automated backups | Minor | 1 | Probes, anonymous status page, Quartz backups, DR runbook |
| 8 | **Module of choice (Major): multi-tenant SaaS isolation** | Major | 2 | See justification below |
| 9 | Advanced search — filters, sorting, pagination | Minor | 1 | Employees directory: free-text search, department/status filters, `sortBy`/`sortDirection` query params + sort control in the UI, offset pagination |
| | **Total** | | **14** | |

> **Framework definition note.** Blazor WebAssembly and ASP.NET Core count as the
> frontend and backend frameworks respectively: both provide a structured architecture
> with conventions, built-in routing/state management/DI, and a complete ecosystem
> on the .NET platform.

> **Module of choice justification.** We chose full multi-tenancy because it is the defining
> constraint of a real HR SaaS: one deployment, many companies, zero data leakage. Every
> aggregate carries a `TenantId` stamped by an EF Core save interceptor; reads go through named
> global query filters (tenant + soft delete); cross-tenant writes throw
> `CrossTenantWriteException`, including writes reached through owned value objects; Dapper read
> models bypass EF filters, so a reflection-based architecture test forces every Dapper query
> method to take an explicit `TenantId` and filter on it. Isolation is not asserted — it is
> proven by integration tests against a real SQL Server container (Testcontainers). It is a
> cross-cutting architectural guarantee enforced in four layers, not a single feature.

## Individual Contributions

- **ahirzallah** — Clean Architecture skeleton, domain model (Value Objects, Result pattern,
  domain events), multi-tenancy enforcement pipeline, JWT/Identity auth, SignalR hubs.
  Hardest problem: guaranteeing tenant isolation through *every* data path (EF, owned VOs,
  Dapper) and proving it with tests instead of trusting conventions.
- **<LOGIN-2>** — …
- **<LOGIN-3>** — …

## Resources

- Subject: ft_transcendence (v21.1) — module map kept in `docs/ft_transcendence-modules-and-points.md`.
- Reference implementations studied and ported by hand (kept in `Helpers/` for transparency):
  MilestoneStack (Clean Architecture layout), DevHabit (REST API conventions), Vuexy
  (Bootstrap admin template — licensed template assets).
- Libraries: EF Core, Dapper, MediatR, FluentValidation, Quartz.NET, QuestPDF (Community
  license), ApexCharts, SignalR, Testcontainers, NetArchTest.

### How AI was used

AI assistance (Claude / ChatGPT) was used as a pair-programming tool for: scaffolding
repetitive vertical slices after the first hand-written one, generating test data seeds,
reviewing code for bugs, and drafting documentation. All architectural decisions (Clean
Architecture layering, multi-tenancy strategy, Result pattern, CQRS split) were made and
implemented by the team; AI-generated code was read, reworked and is understood by every
member. The bug-audit files in the repo root document our own manual evaluation passes.

## Architecture (extra detail)

```
src/
  Core/HrmSystem.Domain                    ← entities, VOs, Result, domain events (zero outward deps)
  Core/HrmSystem.Application               ← CQRS features, behaviors, port interfaces
  Infrastructure/HrmSystem.Infrastructure  ← EF Core 10 + Dapper, Quartz jobs, Identity/JWT, Redis
  Web/HrmSystem.Web                        ← REST controllers, SignalR hubs, health endpoints, OTEL
  Client/HrmSystem.Client                  ← Blazor WASM (REST-only), Vuexy theme, ApexCharts interop
tests/
  ...UnitTests | ...ArchitectureTests (NetArchTest) | ...IntegrationTests (Testcontainers)
```

Layer boundaries are **enforced by architecture tests**, not just convention. See
`docs/architecture.md`.

## Operations

- **Backups**: Quartz `DatabaseBackupJob` (02:00 UTC daily, compressed + checksummed) into the
  shared `sqlserver_backups` volume; retention 7 daily / 4 weekly; `POST /api/backups/run` for
  on-demand. See `docs/disaster-recovery-runbook.md`.
- **Auth hardening**: `/api/auth/*` is rate-limited (10 req/min per client IP, 429 + `Retry-After`).
- **Observability**: OpenTelemetry traces/metrics via OTLP to Seq.
