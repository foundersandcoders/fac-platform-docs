# Repo Consolidation Tracker

> **Status as of 2026-02-11**
>
> Tracks incremental progress toward consolidating fac-cra, fac-web, and fac-distribution into a unified API architecture.

---

## Phase 1: Establish the Contract Layer

### 1a. Standardise session key format ⚠️ IN PROGRESS

**Current state:**
- ✅ fac-cra/api/roots/auth.js uses `session:${unix}:${uuid}` format
- ❌ fac-cra/api/bloom auth.routes.ts writes raw hex keys (NO prefix)
- ✅ fac-distribution reads with configurable prefix awareness

**Next action:**
Update Bloom auth to use `session:${unix}:${uuid}` format.

**Files to modify:**
- `/fac-cra/api/bloom/src/routes/auth.routes.ts` (lines 64, 135)

**Changes needed:**
```typescript
// Current (lines 64, 135):
const sessionId = randomBytes(32).toString('hex');

// New:
const sessionId = `session:${email.split('@')[0]}:${randomBytes(16).toString('hex')}`;
```

**Trigger:** Next Bloom auth change (or do proactively to fix inconsistency)

---

### 1b. Add fac-distribution resolvers to fac-cra ⏸️ NOT STARTED

**Current state:**
- fac-distribution has ~4 resolver operations in `/fac-distribution/api/src/resolvers/email.ts`
- fac-cra has no `distribution_*` operations in schema.gql
- fac-cra has no `api/roots/distribution.js` file

**Next action:**
Wait for a distribution feature change, then move that resolver to fac-cra incrementally.

**Trigger:** Any change to distribution email functionality

---

### 1c. Centralise migrations in fac-cra ⏸️ NOT STARTED

**Current state:**
- fac-distribution has 6 migration files (needs investigation)
- fac-web has test DB init in `test-db/init.sql` with schema creation
- fac-cra has migration system (needs investigation)

**Next action:**
Audit migration files in all three repos.

**Trigger:** Next migration needed in any repo

---

## Phase 2: Point fac-web at fac-cra's API

### 2a. Proxy /g endpoint ⏸️ NOT STARTED

**Current state:**
- fac-web has full GraphQL API in `/fac-web/api/`
- fac-web has resolvers in `/fac-web/api/roots/`

**Next action:**
Add proxy to fac-cra's `/g` endpoint as part of next fac-web API change.

**Trigger:** Next fac-web feature touching the API layer

---

### 2b-2d. Lead auth, API removal, CDN ⏸️ BLOCKED BY 2a

---

## Phase 3: Point fac-distribution at fac-cra's API

### 3a-3c. Resolver-by-resolver HTTP calls ⏸️ BLOCKED BY 1b

---

## Phase 4: Cleanup

### 4a-4c. Secrets, ports, schema ⏸️ BLOCKED BY PHASES 2-3

---

## Quick Reference

| Symbol | Meaning |
|--------|---------|
| ✅ | Complete |
| ⚠️ | In progress |
| ⏸️ | Not started |
| 🚫 | Blocked |

---

## Session Format Reference

**Target format:** `session:${unix}:${uuid}`

**Session shape (JSON):**
```json
{
	"id": 123,
	"unix": "jasonwarren",
	"email": "jason@example.com",
	"photo": "https://...",
	"role": "LEARNER"
}
```

**Redis TTL:** 90 days (Bloom), 365 days (CRA auth)
