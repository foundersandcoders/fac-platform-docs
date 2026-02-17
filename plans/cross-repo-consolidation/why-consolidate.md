# Why We Need to Consolidate

Right now, our three repos (fac-cra, fac-web, fac-distribution) all talk to the same database, but each one has its own copy of the API logic, its own authentication code, and its own database migrations. When someone changes a query in one repo, they have to remember to change it in the others too. Sometimes they don't. Things break.

**What's actually going wrong:**

- **Bugs from copy-paste drift.** fac-web's API resolvers are near-identical copies of fac-cra's. They started the same; they're quietly diverging. A fix in one doesn't reach the other.
- **Two email systems writing to the same tables.** fac-cra and fac-distribution both write emails to the same database table. Only fac-distribution has rate limiting. Nothing stops them trampling each other.
- **Database migrations run from multiple repos.** Three repos independently create schemas and tables in the same database. There's no single record of what the database should look like.
- **Secrets duplicated everywhere.** API keys, tokens, and credentials are copy-pasted across repos. Rotating a key means updating it in three places.
- **No access control on distribution operations.** fac-cra has a detailed role-based permission system (338 rules). fac-distribution has none — if you're logged in, you can do anything.

**What the consolidation fixes:**

One API (fac-cra) handles all data access. The other two repos become frontends that call it. No duplicated logic, no manual sync, no drift. Each phase is small enough to ship alongside normal feature work — nothing stops to "do migration."

The full plan is in `docs/consolidation-plan.md`.
