# Gradual Repo Consolidation Plan

## Constraints
1. Feature dev continues at pace — refactoring piggybacks on normal work
2. Three repos stay as three repos
3. API lives in fac-cra
4. No manual sync between repos (eventual state)

## Architecture Target

```
fac-cra (API + backend)
├── All GraphQL resolvers (eureka, hermes, cdn, bosco, distribution)
├── All database migrations
├── Auth as single session authority
└── Role-based access for all consumers

fac-web (frontend only)
├── SvelteKit app
├── Calls fac-cra API over HTTP (no local API)
└── No database access, no Redis access

fac-distribution (frontend + scheduler)
├── Frontend UI
├── Calls fac-cra API over HTTP (no local resolvers)
├── Email scheduler (reads from queue table, calls fac-cra API)
└── No direct database writes except scheduler consumption
```

---

## Phase 1: Establish the Contract Layer

**Do this as part of any feature touching auth or the /g endpoint.**

### 1a. Standardise session key format in fac-cra
- Bloom currently writes raw hex keys (no `session:` prefix) — align to `session:{unix}:{uuid}` format
- Single session shape: `{ id, unix, email, photo, role }`
- fac-distribution already reads with configurable `SESSION_PREFIX` — no change needed there

### 1b. Add fac-distribution's resolvers to fac-cra
- Create `api/roots/distribution.js` in fac-cra
- Move the ~100 SQL queries from `fac-distribution/api/src/queries/` into fac-cra resolver format
- Add `distribution_*` operations to fac-cra's `schema.gql`
- Add role permissions for distribution operations in `roles.js`
- **Do this incrementally**: each time a distribution feature is modified, move that resolver to fac-cra and point fac-distribution at it

### 1c. Centralise migrations in fac-cra
- Move fac-distribution's 6 migration files into fac-cra's migration system
- Tables to absorb: `email_templates`, `blocked_emails`, `audit_logs`, `activity_logs`, `scheduled_emails`, `recurring_email_schedules`, `send_rate_limits`, `rate_limit_exceptions`, `agent_*`
- Remove schema creation (`CREATE SCHEMA IF NOT EXISTS`) from fac-web's `test-db/init.sql` — derive test DB setup from fac-cra's migrations
- **Trigger**: do this the next time any migration is needed in fac-distribution

---

## Phase 2: Point fac-web at fac-cra's API

**Do this as part of any feature touching fac-web's API layer.**

### 2a. Proxy first, then eliminate
- Add `INTERNAL_API_URL` env var to fac-web pointing to fac-cra (e.g. `http://fac-cra:8010`)
- Replace fac-web's `/g` handler with a proxy: forward the request body to fac-cra, relay the response
- This is a single-file change in `fac-web/api/index.js` — zero resolver changes needed
- Lead auth (`/auth/lead/:hash`) stays in fac-web temporarily (see 2b)

### 2b. Move lead auth to fac-cra
- fac-web's `/auth/lead/:hash` endpoint writes `_eka` cookie — move this to fac-cra
- fac-web proxies or redirects lead auth requests to fac-cra
- **Trigger**: next time lead auth logic changes

### 2c. Remove fac-web's API code
- Once proxy is stable: delete `fac-web/api/roots/`, `fac-web/api/queries/`, `fac-web/api/schema.gql`
- Keep `fac-web/api/index.js` as a thin proxy (or use SvelteKit server routes to proxy)
- Remove fac-web's direct PostgreSQL and Redis dependencies
- **Trigger**: after proxy has been running in production for a reasonable period

### 2d. CDN/MinIO consolidation
- fac-web's cdn.js points to `localhost:9000` (dev MinIO), fac-cra's points to `cdn.${REACT_APP_DOMAIN}`
- Once fac-web proxies to fac-cra, CDN operations use fac-cra's config automatically
- Remove fac-web's MinIO credentials from its env

---

## Phase 3: Point fac-distribution at fac-cra's API

**Do this resolver-by-resolver as distribution features are worked on.**

### 3a. Replace resolver-dispatch with HTTP calls
- fac-distribution's `/g` endpoint currently dispatches to local resolvers
- Change each resolver to call fac-cra's `/g` endpoint instead (same `{ source, variableValues }` format)
- fac-distribution authenticates to fac-cra using an internal service token or by forwarding the `_mlx` cookie

### 3b. Add RBAC for distribution operations
- fac-cra's `roles.js` has no `distribution_*` permissions yet
- Add them as resolvers move across (Phase 1b)
- fac-distribution gets a service role or inherits the requesting user's role

### 3c. Scheduler stays in fac-distribution
- The 60-second email scheduler is fac-distribution's unique responsibility
- It should read from the queue table and call fac-cra's API to send/log emails
- Direct DB reads for queue consumption are acceptable (read-only polling)

---

## Phase 4: Eliminate Remaining Sync Points

### 4a. Single source for shared secrets
- fac-web and fac-cra share identical secrets (MINIO keys, HETZNER_TOKEN, RUN_POD_KEY, OPENAI_API_KEY, FB tokens)
- Once fac-web proxies to fac-cra, fac-web no longer needs these — remove them
- fac-distribution keeps only: `DATABASE_URL` (read-only for scheduler), `REDIS_URL`, `SESSION_COOKIE_NAME`

### 4b. Port conflict resolution
- fac-cra and fac-web both use port 8010 — doesn't matter in production (separate containers) but breaks local dev
- Change fac-web's dev server to 8012 (or any unused port)
- **Trigger**: next time local dev setup is touched

### 4c. GraphQL schema is fac-cra's canonical source
- After Phase 2, fac-web has no schema.gql
- After Phase 3, fac-distribution has no resolvers
- fac-cra's `schema.gql` is the single source of truth
- Consider generating TypeScript types from it for fac-web and fac-distribution to consume (optional, nice-to-have)

---

## Sequencing Summary

| Phase | Trigger | Risk | Reversible |
|-------|---------|------|------------|
| 1a: Session format | Next auth change | Low — only affects new sessions | Yes |
| 1b: Distribution resolvers to fac-cra | Any distribution feature work | Medium — resolver behaviour must match | Yes (keep old resolvers until verified) |
| 1c: Centralise migrations | Next distribution migration | Low — additive | Yes |
| 2a: fac-web proxy | Any fac-web API change | Low — transparent proxy | Yes (revert to local resolvers) |
| 2b: Lead auth to fac-cra | Next lead auth change | Low | Yes |
| 2c: Remove fac-web API | After proxy stabilises | Medium — point of no return | Partially (code is in git) |
| 3a: Distribution HTTP calls | Per-resolver during feature work | Medium | Yes (keep local fallback) |
| 4a-c: Cleanup | After Phases 2-3 complete | Low | N/A |

## What This Does NOT Change
- Number of repos (stays at 3)
- Deployment topology (each repo deploys independently)
- Frontend frameworks (Svelte stays Svelte, React stays React)
- Database structure (same schemas, same tables)

## End State
- **fac-cra**: single API authority, all resolvers, all migrations, all auth
- **fac-web**: pure frontend, zero backend logic, consumes fac-cra API
- **fac-distribution**: thin frontend + email scheduler, consumes fac-cra API for everything else
- **Zero manual sync**: no duplicated schemas, no copied resolvers, no shared secrets beyond what's needed
