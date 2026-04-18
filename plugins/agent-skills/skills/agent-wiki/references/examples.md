# Examples

Concrete examples of each operation for reference.

## Table of Contents
- [init example](#init-example)
- [ingest example](#ingest-example)
- [query example](#query-example)
- [lint example](#lint-example)

---

## init example

**Prompt:** "Initialize the wiki for this project."

**After scanning a Node.js/Express API, the agent creates:**

`<wiki-root>/pages/project-overview.md`:
```yaml
---
title: Project Overview
type: architecture
created: 2026-04-11
last_updated: 2026-04-11
tags: [overview, express, postgresql]
related: [api-architecture, database-layer]
---
```
```markdown
# Project Overview

Task management REST API built with Express and PostgreSQL.

- **Entry point**: `src/index.js`
- **Auth**: JWT-based, see [[auth-middleware]]
- **Database**: PostgreSQL via pg-pool, see [[database-layer]]
- **Endpoints**: CRUD for tasks with role-based access (admin, user, viewer)
```

`<wiki-root>/pages/auth-middleware.md`:
```yaml
---
title: Auth Middleware
type: architecture
created: 2026-04-11
last_updated: 2026-04-11
tags: [auth, jwt, middleware]
related: [project-overview, api-architecture]
---
```
```markdown
# Auth Middleware

File: `src/auth/middleware.js`

- Reads `Authorization: Bearer <token>` header
- Verifies with `JWT_SECRET` (env var, falls back to `'dev-secret-change-in-prod'`)
- Token expiry: 24h
- `requireRole(...roles)` middleware for RBAC on specific routes
```

`<wiki-root>/index.md` (updated):
```markdown
# Wiki Index

## Architecture
- [Project Overview](pages/project-overview.md) — Express + PostgreSQL task management API | tags: overview, express
- [Auth Middleware](pages/auth-middleware.md) — JWT auth and role-based access control | tags: auth, jwt
- [Database Layer](pages/database-layer.md) — pg-pool configuration and query patterns | tags: database, postgresql
```

`<wiki-root>/log.md` (updated):
```markdown
# Wiki Log

## [2026-04-11] init | Bootstrapped wiki from codebase scan
Created: pages/project-overview.md, pages/auth-middleware.md, pages/database-layer.md
```

---

## ingest example

**Prompt:** "The tests are flaky because the DB connection pool timeout is 5 seconds but CI containers take 8-12s to start. File this."

**Agent creates/updates:**

`<wiki-root>/pages/flaky-ci-tests.md` (new — type: bug):
```yaml
---
title: Flaky CI Tests — DB Pool Timeout
type: bug
created: 2026-04-11
last_updated: 2026-04-11
tags: [ci, testing, database, flaky]
related: [database-layer, ci-postgres-timing]
---
```
```markdown
# Flaky CI Tests — DB Pool Timeout

## Symptoms
Tests pass locally, fail intermittently in CI.

## Root cause
`connectionTimeoutMillis: 5000` in `src/db/pool.js` is too short.
CI postgres containers take 8-12s to fully initialize.

## Fix
Increase to 15000ms or make configurable via `DB_CONNECTION_TIMEOUT` env var.

## What made it tricky
Failures are intermittent — depends on container startup speed.
Locally postgres is already running so the timeout never triggers.

See also: [[database-layer]], [[ci-postgres-timing]]
```

`<wiki-root>/pages/database-layer.md` (updated — added Known Issues section):
```markdown
## Known Issues
- CI flaky tests due to connection timeout — see [[flaky-ci-tests]]
```

`<wiki-root>/log.md` (appended):
```markdown
## [2026-04-11] ingest | Flaky CI tests caused by DB pool timeout (5s vs 8-12s container startup)
Created: pages/flaky-ci-tests.md, pages/ci-postgres-timing.md
Updated: pages/database-layer.md
Cross-refs: flaky-ci-tests <-> database-layer, flaky-ci-tests <-> ci-postgres-timing
```

---

## query example

**Prompt:** "What do we know about the auth system?"

**Agent reads** `<wiki-root>/index.md`, finds auth-related pages, reads them, and synthesizes:

> The auth system uses JWT with 24h token expiry (`src/auth/middleware.js`). Tokens are
> verified against `JWT_SECRET` which defaults to a hardcoded dev secret. Role-based access
> uses `requireRole()` middleware supporting admin, user, and viewer roles.
>
> Known issue: when tokens expire during async operations, the error bubbles to the global
> error handler in `src/index.js` which returns a generic 500 instead of 401. See
> [[silent-auth-500-gotcha]] for details.

---

## lint example

**Prompt:** "Lint the wiki."

**Agent output:**

> **Wiki lint — 2026-04-11**
>
> Checked 7 pages.
>
> **Fixed:**
> - `project-overview.md` was orphaned (no inbound links). Added references from `auth-middleware.md` and `database-layer.md`.
>
> **Gaps found:**
> - No page for bcrypt password hashing (used in auth but undocumented)
> - No page for CORS configuration
> - JWT secret default value is a security gotcha worth its own page
>
> **No issues:**
> - No contradictions found
> - No stale pages (all updated today)
> - No broken wikilinks
> - Index is in sync with pages/
