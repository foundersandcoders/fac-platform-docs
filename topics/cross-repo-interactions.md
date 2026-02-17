# Cross-Repo Interaction Analysis: How `fac-cra`, `fac-web` & `fac-distribution` Affect Each Other

> Investigation date: 2026-02-11
> Scope: How three FAC platform repos interact, contradict, and create implications for developers with partial access.

---

## Executive Summary

Three repos form the FAC platform. They share a **single PostgreSQL database**, a **single Redis instance**, and **overlapping database schemas** — but have **no code-level integration**, **no shared type definitions**, and **no coordination mechanism**. Each repo implements its own GraphQL-like API independently. The result is a system where changes in one repo can silently break another, and developers working in isolation will make assumptions that are wrong.

---

## 1. Repo Profiles

| Aspect | fac-cra | fac-web | fac-distribution |
|--------|---------|---------|-----------------|
| **Purpose** | Internal platform (40+ apps for learners, staff, admin) | Public-facing applicant portal + curriculum | Lead management, email campaigns, project review |
| **Tech** | React 18 + Express + GraphQL | Next.js 16 + Express + GraphQL | React 19 + Express + resolver pattern |
| **Port** | 8010 | 8010 (API) / 3000 (Next.js) | 8011 (API) / 5173 (Vite) |
| **Language** | JavaScript (CommonJS) | JavaScript (CommonJS) | TypeScript |
| **Package manager** | pnpm | pnpm/npm | pnpm |
| **Deployment** | Kubernetes (Helm, K3s) | Unknown (likely Railway or similar) | Railway.app |
| **API style** | Full GraphQL (schema + resolvers) | Full GraphQL (schema + resolvers) | GraphQL-like (resolver dispatch, no schema validation) |

---

## 2. The Shared Database Problem

### 2.1 All three repos connect to the same PostgreSQL instance

```
fac-cra   → DB_URL=postgres://...mlx_test (dev) / mlx (prod)
fac-web   → DB_URL=postgres://...mlx_test (dev) / mlx (prod)
fac-dist  → DATABASE_URL=postgres://...distribution (dev) / mlx (prod)
```

In production, **all three apps hit the same database**. In development, fac-distribution uses a separate `distribution` database, but its migrations create schemas (`eureka`, `hermes`, `bosco`, `analytics`) that also exist in the main `mlx` database used by fac-cra and fac-web.

### 2.2 Overlapping schemas

| Schema | Used by fac-cra | Used by fac-web | Used by fac-distribution |
|--------|:-:|:-:|:-:|
| `core` (users, cohorts, houses) | Yes | Yes (implied via GraphQL) | No |
| `eureka` (leads, events, bookings, scores) | Yes | Yes | Yes |
| `hermes` (emails, campaigns) | Yes | No direct access | Yes |
| `bosco` (integrations, store) | Yes | Yes (queries) | Yes (Gmail credentials) |
| `analytics` | Yes | Yes | Yes |
| `cortex` (learning modules) | Yes | No | No |
| `sapienza` (quizzes) | Yes | No | No |
| `attendance` | Yes | No | No |
| `agora` (events) | Yes | No | No |
| `oracle` (outcomes) | Yes | No | No |

### 2.3 Contradictions and risks

**The `eureka` schema is modified by all three repos.** Each has its own SQL queries that assume specific column names, types, and constraints. There is no shared migration history. If fac-distribution adds a column to `eureka.leads`, fac-cra and fac-web won't know about it and their queries will still work — until someone adds a `SELECT *` that returns unexpected columns, or a migration in one repo conflicts with another.

**The `hermes` schema is owned by fac-distribution but read by fac-cra.** fac-cra has GraphQL resolvers for Hermes (email campaigns, pixel tracking) and fac-distribution has its own comprehensive email system. Both write to `hermes.fct_gmails`. If either changes the table structure, the other breaks silently.

---

## 3. Shared Redis, Separate Assumptions

All three repos connect to the same Redis instance:

```
fac-cra   → REDIS_URL=redis://localhost:6380 (dev) / redis.foundersandcoders.com (prod)
fac-web   → REDIS_URL=redis://localhost:6380 (dev) / redis.foundersandcoders.com (prod)
fac-dist  → REDIS_URL=redis://localhost:6379 (dev) / shared prod Redis
```

### 3.1 Session key format divergence

- **fac-cra** stores sessions as `session:{unix}:{uuid}` with JSON-serialised user objects
- **fac-web** reads the `_mlx` cookie and looks up the session in Redis (same format as fac-cra)
- **fac-distribution** reads the `_mlx` cookie (configurable via `SESSION_COOKIE_NAME`) and looks up with a `sess:` prefix (configurable via `SESSION_PREFIX`)

**Risk:** If the session key format in fac-cra changes, fac-distribution's auth breaks. If fac-distribution's `SESSION_PREFIX` doesn't match what fac-cra writes, auth silently fails. There's no shared session library — each repo reimplements session lookup independently.

### 3.2 The `_eka` lead cookie

- **fac-cra** sets the `_eka` cookie via `GET /auth/lead/:hash` and uses it for lead identification in the Eureka domain
- **fac-web** has an identical `GET /auth/lead/:hash` endpoint that sets the same `_eka` cookie
- **fac-distribution** does not use `_eka` at all — it only uses `_mlx` for staff auth

**Risk:** Both fac-cra and fac-web can set the `_eka` cookie. If a lead visits both the main platform and the public site, the cookie domain (`.foundersandcoders.com`) means they share the same cookie. This is probably intentional but undocumented.

---

## 4. Duplicate GraphQL Schemas

### 4.1 fac-cra and fac-web have nearly identical schemas

Both repos contain a `schema.gql` file with the same GraphQL types and operations. The schemas define:
- 129+ queries
- 231+ mutations
- Shared types: User, Cohort, House, etc.

**They are not generated from a single source.** They are manually maintained copies. Any change to one must be manually applied to the other. There is no verification that they stay in sync.

### 4.2 fac-distribution has no GraphQL schema

fac-distribution uses a resolver-dispatch pattern: `POST /g` receives `{ source: "distribution_get_leads", variableValues: {} }` and routes to a named function. There is no schema validation, no type checking on the wire. This is simpler but means:
- No introspection
- No client-side type generation
- No contract between frontend and backend

### 4.3 Cross-repo operation naming

| Operation pattern | fac-cra | fac-web | fac-distribution |
|---|---|---|---|
| Get leads | `eureka_get_leads` | `eureka_get_me` | `distribution_get_lead` |
| Create lead | `eureka_create_lead` | `programme_create_lead` | N/A (reads only) |
| Update lead | `eureka_update_lead` | `eureka_update_me` | `distribution_update_lead` |
| Send email | `hermes_create_gmail` | N/A | `distribution_send_emails` |
| Events | `agora_admin_create_event` / `eureka_admin_create_event` | `eureka_lead_attend_event` | `distribution_get_upcoming_workshops` |

**The same database row can be modified by three different APIs with three different validation rules.** For example, `eureka.leads.status` can be changed by:
1. fac-cra's `eureka_update_lead` (requires role >= 5, assistant/admin)
2. fac-web's `eureka_update_me` (requires valid `_eka` cookie, lead self-service)
3. fac-distribution's `distribution_update_lead` (requires valid `_mlx` session, no role check)

---

## 5. Authentication Divergence

### 5.1 Role systems

**fac-cra** defines a 7-level role hierarchy:

```
pending: 0, bloom: 1, base: 2, user: 3, learner: 4, assistant: 5, admin: 6
```

338 operations are mapped to minimum role levels. This is comprehensive RBAC.

**fac-web** uses the same hierarchy (imported from `api/roles.js`) but only exposes a subset of operations, mostly at level -1 (public) for the applicant portal.

**fac-distribution** has **no role-based access control at all**. If you have a valid session (`_mlx` cookie), you have full access to everything — including bulk email sending, lead data modification, and SQL query execution. The only protection is that the `_mlx` cookie is only issued to FAC staff.

**Implication:** A developer working on fac-distribution might assume all authenticated users are admins. A developer working on fac-cra knows there are 7 role levels. If fac-distribution ever needs to support non-admin users, the entire auth layer needs rebuilding.

### 5.2 Auth middleware implementations

Each repo has its own auth middleware with subtle differences:

| Aspect | fac-cra (`mws.js`) | fac-web (`mws.js`) | fac-distribution (`auth.ts`) |
|--------|----|----|-----|
| Cookie name | `_mlx` (hardcoded) | `_mlx` (hardcoded) | `_mlx` (configurable via env) |
| Lead cookie | `_eka` | `_eka` | Not used |
| Redis key format | `session:{unix}:{uuid}` | Same as fac-cra | Configurable prefix (`sess:`) |
| Session data | Full user object (JSON) | Full user object (JSON) | User object (JSON) |
| Permission check | Per-operation role level | Per-operation role level | None (all-or-nothing) |
| Dev bypass | Unknown | Unknown | Skips auth entirely |

### 5.3 The `_mlx` cookie domain

All three repos set or read cookies on `.foundersandcoders.com` (production) or `.foundersandcoders.localhost` (dev). This means:
- A single login to fac-cra authenticates the user across all three apps
- Logging out of any app should invalidate the session in Redis, affecting all apps
- But fac-distribution has its own `/logout` endpoint that only clears the cookie — it doesn't destroy the Redis session

---

## 6. Email System Overlap

This is one of the most problematic areas.

### 6.1 Two email systems

**fac-cra** has a Hermes module:
- GraphQL mutations: `hermes_create_campaign`, `hermes_update_campaign`, `hermes_create_gmail`
- Writes to `hermes.fct_evts` and `hermes.campaigns`
- Used by staff for ad-hoc email campaigns
- Pixel tracking via `/hermes/*` route on the static server

**fac-distribution** has a comprehensive email system:
- 30+ email resolver functions
- Writes to `hermes.scheduled_emails`, `hermes.fct_gmails`, `hermes.recurring_email_schedules`
- Rate limiting, blocking, eligibility checks
- Recurring schedules, workshop campaigns
- Gmail API integration with service account
- Plain text to HTML conversion (OpenAI)

### 6.2 Contradictions

1. **Both write to `hermes.fct_gmails`** — Each inserts rows with slightly different column usage. If fac-cra sends an email and fac-distribution queries email history, it will see fac-cra's emails but may not handle them correctly (different column expectations).

2. **Rate limiting only exists in fac-distribution** — fac-cra can send unlimited emails via `hermes_create_gmail`. A staff member using fac-cra to send emails bypasses all rate limits, blocked lists, and eligibility checks that fac-distribution enforces.

3. **No deduplication** — Both systems can send emails to the same lead independently. There's no shared "last contacted" timestamp or coordination.

4. **Different Gmail credentials paths** — fac-cra uses `googleapis` OAuth directly. fac-distribution reads service account credentials from `bosco.store` (id=109). If credentials rotate in one place, the other may break.

---

## 7. Event/Workshop Systems

### 7.1 Three event systems touching the same data

**fac-cra — Agora module:**
- Full event CRUD for internal events (workshops, talks, meetups)
- KSB attachment to events
- Attendance tracking integration
- Writes to: `agora.events`, `agora.bookings`, `agora.locations`

**fac-cra — Eureka module (admin events):**
- Separate event system for lead-facing events
- Different table structure under `eureka` schema
- Writes to: `eureka.events`, `eureka.bookings`, `eureka.locations`

**fac-web — Eureka lead events:**
- Reads `eureka.events` and `eureka.bookings`
- Leads can book/cancel events
- Writes to: `eureka.bookings`

**fac-distribution — Workshop campaigns:**
- Reads `eureka.events` for upcoming workshops
- Sends automated email campaigns tied to events
- Does not create or modify events — read-only

### 7.2 The Agora vs Eureka event confusion

fac-cra has **two separate event systems**: Agora (for learner events) and Eureka (for lead events). They have similar schemas but different tables. A developer working in fac-distribution sees only Eureka events and might assume that's the complete picture. A developer in fac-cra working on Agora might not realise that Eureka events also exist and serve a different audience.

---

## 8. Lead Status Lifecycle

The `eureka.leads.status` field is the most critical shared state across all three repos.

### 8.1 Status values

```sql
lead_status ENUM:
  NO_CV, GOOD_CV, BAD_CV,
  INVITED_ROUND_ONE, BOOKED_ROUND_ONE, COMPLETED_ROUND_ONE,
  INVITED_ROUND_TWO, BOOKED_ROUND_TWO, COMPLETED_ROUND_TWO,
  WAITLISTED, ON_PROGRAMME, REJECTED, WITHDRAWN, ALUMNI, EMAIL_BOUNCED
```

### 8.2 Who changes status and when

| Status transition | fac-cra | fac-web | fac-distribution |
|---|---|---|---|
| → `NO_CV` | Initial creation | Initial creation | N/A |
| → `GOOD_CV` / `BAD_CV` | CV scoring (OpenAI) | CV scoring (OpenAI) | N/A |
| → `INVITED_ROUND_ONE` | Admin action | N/A | N/A |
| → `BOOKED_ROUND_ONE` | Admin action | Lead self-service | N/A |
| → `REJECTED` | Admin action | N/A | `distribution_reject_project()` |
| → `ON_PROGRAMME` | Admin action | N/A | `distribution_accept_project()` |
| → `WITHDRAWN` | Admin action | Lead self-service | N/A |
| Any → Any | `eureka_update_lead()` | N/A | `distribution_update_lead()` |

**Risk:** fac-distribution can change a lead's status to anything (e.g., `ON_PROGRAMME`) without going through fac-cra's admin workflow. There's no status transition validation — any repo can set any status at any time.

---

## 9. CV Scoring Duplication

Both fac-cra and fac-web implement CV scoring via OpenAI:

**fac-cra:** `api/roots/roots.utils.js` — Uses OpenAI to score CVs, stores in `eureka.scores`
**fac-web:** `api/roots/roots.utils.js` — Identical implementation, same prompts, same table

These are copy-pasted implementations. If the scoring prompt is updated in one repo, the other continues using the old version. Leads could receive different scores depending on which entry point processed their CV.

---

## 10. Environment Variable Conflicts

### 10.1 Port conflicts

```
fac-cra API:   port 8010
fac-web API:   port 8010
fac-dist API:  port 8011
```

fac-cra and fac-web **cannot run simultaneously** in development — they both use port 8010. A developer working on both needs to manually change ports.

### 10.2 Shared secrets in `.env` files

Both fac-cra and fac-web contain `.env.local` files with identical secrets:
- Same `MINIO_ACCESS_KEY` / `MINIO_SECRET_KEY`
- Same `HETZNER_TOKEN`
- Same `RUN_POD_KEY`
- Same `OPENAI_API_KEY`
- Same `FB_TOKEN` / `FB_PIXEL_ID`

If any of these are rotated, they must be updated in both repos. There's no centralised secret management for development.

### 10.3 Database connection variables

```
fac-cra:   DB_URL (env var name)
fac-web:   DB_URL (env var name)
fac-dist:  DATABASE_URL (env var name)
```

Different variable names for the same thing. A developer setting up all three repos needs to know this.

---

## 11. Deployment Architecture Mismatch

| Aspect | fac-cra | fac-web | fac-distribution |
|--------|---------|---------|-----------------|
| **Platform** | Self-hosted Kubernetes (K3s/Helm) | Unknown (likely Railway or self-hosted) | Railway.app |
| **CI/CD** | GitHub Actions → Docker → K8s | No CI/CD config found | GitHub Actions → Railway |
| **SSL** | Let's Encrypt via Traefik | Unknown | Railway handles it |
| **Scaling** | HPA (2-10 replicas) | Unknown | Single instance |
| **Secrets** | External Secrets operator (Vault/AWS) | `.env.local` file | Railway env injection |

A developer working on fac-distribution might assume Railway-style deployment patterns. A developer on fac-cra needs Kubernetes knowledge. These are fundamentally different deployment models for the same platform.

---

## 12. Code Quality and Convention Divergence

| Convention | fac-cra | fac-web | fac-distribution |
|-----------|---------|---------|-----------------|
| **Language** | JavaScript (CJS) | JavaScript (CJS) | TypeScript |
| **Module system** | `require()` / `exports` | `require()` / `exports` | `import` / `export` |
| **Error handling** | `try/catch` with `console.error` | `try/catch` with `console.error` | `try/catch` with structured errors |
| **Testing** | Minimal (Storybook stories) | Docker-based test DB | Vitest configured |
| **Linting** | None configured | None configured | ESLint configured |
| **Type safety** | None (raw JS) | None (raw JS) | Full TypeScript |

A TypeScript developer joining fac-distribution and then moving to fac-cra would face a completely different codebase with no type safety, no linting, and a different module system.

---

## 13. Critical Risk Matrix

| Risk | Severity | Likelihood | Impact |
|------|----------|------------|--------|
| Database migration conflict (schema changes in one repo break another) | **Critical** | High | Data corruption, app crashes |
| Email duplication (both fac-cra and fac-distribution send to same lead) | **High** | Medium | Lead gets duplicate/conflicting emails |
| Session format change breaks cross-app auth | **High** | Medium | Users locked out of some apps |
| Lead status changed without workflow validation | **High** | High | Leads in wrong pipeline stage |
| CV scoring produces different results across repos | **Medium** | High | Inconsistent lead evaluation |
| Secret rotation misses one repo | **Medium** | Medium | Service outage |
| Port conflict blocks local development | **Low** | High | Developer friction |
| GraphQL schema drift between fac-cra and fac-web | **Medium** | High | Frontend/backend type mismatches |

---

## 14. Recommendations

### Immediate (low effort, high impact)

1. **Document the shared database contract** — Create a canonical list of tables, columns, and which repo owns writes vs reads
2. **Centralise database migrations** — Move all migrations to a single repo or a shared migrations repo
3. **Add `.env.example` files** — Document the different variable names across repos (`DB_URL` vs `DATABASE_URL`)
4. **Fix the port conflict** — Change fac-web's API port in development (or use a reverse proxy)

### Medium-term

5. **Extract shared types** — Create a shared package with TypeScript interfaces for `eureka.leads`, `hermes.fct_gmails`, etc.
6. **Unify email sending** — Route all email operations through fac-distribution's rate-limited system
7. **Add status transition validation** — Enforce valid lead status transitions at the database level (triggers or constraints)
8. **Centralise session management** — Extract a shared session library used by all three repos

### Long-term

9. **Consolidate fac-web into fac-cra** — fac-web's API is a subset of fac-cra's. The public-facing Next.js app could use fac-cra's API directly
10. **Move fac-distribution to TypeScript conventions** — Already TypeScript; make it the reference for code quality
11. **Implement database-level access control** — The `002_distribution_app_role.sql` migration is a good start; extend to all repos

---

## 15. Architecture Diagram

```
                    ┌─────────────────────────────────────────────────────┐
                    │                  SHARED INFRASTRUCTURE              │
                    │                                                     │
                    │  ┌──────────────┐        ┌──────────────┐           │
                    │  │  PostgreSQL  │        │    Redis     │           │
                    │  │              │        │              │           │
                    │  │ eureka.*     │        │ session:*    │           │
                    │  │ hermes.*     │        │ eureka:*     │           │
                    │  │ bosco.*      │        │              │           │
                    │  │ core.*       │        │              │           │
                    │  │ analytics.*  │        │              │           │
                    │  │ cortex.*     │        │              │           │
                    │  │ sapienza.*   │        │              │           │
                    │  │ attendance.* │        │              │           │
                    │  │ agora.*      │        │              │           │
                    │  │ oracle.*     │        │              │           │
                    │  └──────┬───────┘        └──────┬───────┘           │
                    │         │                       │                   │
                    └─────────┼───────────────────────┼───────────────────┘
                              │                       │
              ┌───────────────┼───────────────────────┼
              │               │                       │
     ┌────────┴─────────┐  ┌──┴───────────┐  ┌────────┴──────────┐
     │    fac-cra       │  │   fac-web    │  │  fac-distribution │
     │   (port 8010)    │  │  (port 8010) │  │   (port 8011)     │
     │                  │  │              │  │                   │
     │ 40+ React SPAs   │  │ Next.js 16   │  │ React 19 + Vite   │
     │ GraphQL API      │  │ GraphQL API  │  │ Resolver API      │
     │ Bloom API        │  │              │  │                   │
     │ Static server    │  │              │  │                   │
     │                  │  │              │  │                   │
     │ Audience:        │  │ Audience:    │  │ Audience:         │
     │ - Learners       │  │ - Applicants │  │ - FAC staff only  │
     │ - Staff          │  │ - Public     │  │                   │
     │ - Admins         │  │              │  │                   │
     │                  │  │              │  │                   │
     │ Auth: _mlx + RBAC│  │ Auth: _eka   │  │ Auth: _mlx (no    │
     │ (7 role levels)  │  │ + _mlx       │  │ role checks)      │
     │                  │  │              │  │                   │
     │ Deploy: K8s/Helm │  │ Deploy: ???  │  │ Deploy: Railway   │
     └──────────────────┘  └──────────────┘  └───────────────────┘
              │                    │                    │
              │                    │                    │
     ┌────────┴────────────────────┴────────────────────┴─────────┐
     │                   EXTERNAL SERVICES                        │
     │                                                            │
     │   MinIO (S3)  │  Gmail API  │  OpenAI  │  Hetzner  │  ...  │
     └────────────────────────────────────────────────────────────┘
```

---

## 16. For Developers With Partial Access

### If you only have access to fac-cra:
- You own the canonical user/session system. Changes to session format affect all three apps.
- The `eureka` schema is shared. Do not assume you are the only writer.
- The `hermes` email tables are also written to by fac-distribution. Do not truncate or restructure without checking.
- Your GraphQL schema must stay in sync with fac-web's copy. There is no automated check.

### If you only have access to fac-web:
- Your API is a subset of fac-cra's. The schema.gql is a copy — not the source of truth.
- The `_eka` cookie is also set by fac-cra. Both repos can authenticate leads.
- CV scoring logic is duplicated from fac-cra. If prompts change upstream, update yours too.
- You cannot run alongside fac-cra locally (port 8010 conflict).

### If you only have access to fac-distribution:
- You have no role-based access control. Every authenticated user is treated as an admin.
- You share the `eureka` and `hermes` schemas with two other apps. Do not assume you own these tables.
- Your session reading depends on fac-cra's session writing format. If they change it, your auth breaks.
- Rate limiting only exists in your app. Staff can bypass all limits by using fac-cra to send emails directly.
- Your `DATABASE_URL` env var is different from the other repos' `DB_URL`. Don't assume consistency.
