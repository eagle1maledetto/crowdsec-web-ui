# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
# Install dependencies
pnpm install

# Development (starts server + client concurrently)
pnpm dev
pnpm dev:server          # server only (tsx watch, port 3000)
pnpm dev:client          # client only (Vite HMR)

# Build (client via Vite, server via tsup)
pnpm build

# Production
pnpm start               # runs dist/server/index.js

# Tests
pnpm test                # runs server then client tests
pnpm test:server         # vitest with vitest.server.config.ts
pnpm test:client         # vitest with vitest.client.config.ts (jsdom)

# Single test file
pnpm vitest run --config vitest.server.config.ts server/path/to/file.test.ts
pnpm vitest run --config vitest.client.config.ts client/src/path/to/file.test.tsx

# Coverage (has threshold checks)
pnpm coverage

# Quality checks
pnpm typecheck           # tsc --noEmit
pnpm lint                # eslint (client + config files only)
pnpm verify              # test + coverage + typecheck + lint + build (full CI check)
```

## Architecture

Full-stack TypeScript app — a web dashboard for [CrowdSec](https://crowdsec.net) that acts as a caching proxy in front of the CrowdSec Local API (LAPI).

### Three code boundaries

- **`client/`** — React 19 SPA built with Vite. React Router v7, Tailwind CSS 4, Recharts/Nivo for charts, Lucide icons.
- **`server/`** — Hono web framework on Node.js. Serves the API (`/api/*`) and the built SPA as static files.
- **`shared/`** — `contracts.ts` defines request/response types shared between client and server.

### Key server modules

| File | Purpose |
|---|---|
| `server/app.ts` | All HTTP route definitions |
| `server/lapi.ts` | CrowdSec LAPI client (JWT auth, delta sync) |
| `server/database.ts` | SQLite schema, prepared statements, all queries |
| `server/config.ts` | Runtime config from environment variables |
| `server/auth.ts` | CrowdSec auth (password or mTLS, not user-facing) |
| `server/notifications.ts` | Notification rule engine and dispatch |
| `server/notifications/` | Providers: email, Gotify, MQTT, ntfy, webhooks |

### Key client modules

| File | Purpose |
|---|---|
| `client/src/App.tsx` | Router setup with lazy-loaded pages |
| `client/src/lib/api.ts` | Typed fetch wrapper for all API calls |
| `client/src/components/Layout.tsx` | Shell: sidebar, theme toggle, base layout |
| `client/src/contexts/` | React Context providers (Refresh, Notifications) |

### Data flow

1. Server authenticates to CrowdSec LAPI on startup (JWT or mTLS)
2. Server syncs alerts/decisions into local **SQLite** (better-sqlite3) using delta updates
3. Client fetches from server's `/api/*` endpoints, never directly from LAPI
4. No user authentication — designed to sit behind a reverse proxy (Authentik, Authelia, etc.)

### Path aliases

Defined in `tsconfig.json` and mirrored in `vite.config.ts`:
- `@client/*` → `./client/*`
- `@server/*` → `./server/*`
- `@shared/*` → `./shared/*`

### Database

Raw SQL with prepared statements in `server/database.ts` — no ORM, no migration system. Schema is created on startup if tables don't exist.

SQLite pragmas configured in `openDatabase()`: WAL mode, `synchronous=NORMAL`, 32MB page cache, memory-mapped I/O (256MB), 5s busy timeout.

### Performance architecture

The server has several performance layers for handling large datasets (20k+ alerts):

1. **Batch decision hydration** — `hydrateAlertsBatch()` in `app.ts` collects all decision IDs upfront and does a single `getDecisionStopAtBatch()` query instead of N+1 individual lookups. The original `hydrateAlertWithDecisions()` is kept for single-alert detail views.

2. **In-memory response cache** — `responseCache` in `app.ts` caches computed responses for `/api/alerts`, `/api/stats/alerts`, `/api/stats/decisions`. TTL = `max(refreshIntervalMs, 5000ms)`. Invalidated by `invalidateResponseCache()` which must be called after: sync (delta or full), data mutations (delete, add decision), and manual cache clear.

3. **SQL indexes** — Compound index on `decisions(value, stop_at DESC)` for IP lookup, plus indexes on `decisions(value)` and `decisions(created_at)`.

4. **Materialized stats columns** — `alerts` table has denormalized columns (`source_cn`, `source_as_name`, `source_scope`, `source_range`, `target`, `simulated`) populated during insert. Stats endpoints query these columns directly without JSON.parse. The `raw_data` column is kept for full alert detail views. When adding new fields to stats responses, add them as columns too. Schema migration in `migrateStatsColumns()` adds columns automatically on startup.

5. **Server-side pagination & search** — `/api/alerts` and `/api/decisions` support `?page=N&page_size=N` query params. With `page`: returns `PaginatedResponse` envelope `{ data: [...], pagination: { page, page_size, total, total_pages } }`. Without `page`: returns bare array (backward compatible). Search via `?q=` does SQL LIKE across scenario, source_ip, source_cn, source_as_name, target, message. Additional filter params: `?ip=`, `?scenario=`, `?country=`, `?as=`, `?target=`, `?dateStart=`, `?dateEnd=`, `?simulation=`. Search filters are built dynamically in `buildAlertSearchQuery()` / `buildDecisionSearchQuery()` in `database.ts`.

When modifying data mutation paths (delete alerts/decisions, add decisions, cleanup by IP), always call `invalidateResponseCache()` after the mutation.

6. **Dashboard aggregation endpoint** — `GET /api/stats/dashboard?granularity=day|hour` returns pre-aggregated stats (~12KB) instead of raw records (~37MB). Supports filter params (`country`, `scenario`, `as_name`, `ip`, `target`, `simulated`). Aggregation runs in SQL via `getDashboardStats()` in `database.ts`. Cached in `responseCache.dashboard` (only unfiltered day-granularity requests).

7. **Unscoped LAPI fetch** — `lapi.ts:fetchAlerts()` fetches with `scope=ip`, `scope=range`, AND `undefined` (no scope). The unscoped fetch captures batch alerts from `cscli decisions import` which have empty `source.scope`. Results are merged by alert ID to deduplicate.

When adding columns to alerts/decisions tables, update: the INSERT statement, `AlertInsertParams`/`DecisionInsertParams` interfaces, `processAlertForDatabase()` in `app.ts`, and the `insertAlert`/`insertDecision` calls in test files.

### UI patterns

- **Loading state**: Never hide existing data during a reload. Use `{loading && data.length === 0 ? <Loading /> : data.map(...)}` instead of `{loading ? <Loading /> : data.map(...)}`. This prevents flicker when multiple effects trigger concurrent loads.
- **Suspense**: Route fallbacks are set to `null` (not a loading component) to prevent flash during lazy chunk loads.
- **Refresh effect**: Use `isLoadingInitial` ref to prevent background refresh from firing before the mount load completes.

### Testing

- Server tests: Vitest in Node environment (`vitest.server.config.ts`) — 73 tests across 9 files
- Client tests: Vitest in jsdom (`vitest.client.config.ts`) with Testing Library
- Client coverage thresholds: 90% lines/functions/branches/statements
- Run on vishnu: `pnpm test:server` (Node 24 + pnpm 10 installed on host)

### Build output

- `dist/client/` — static SPA assets
- `dist/server/` — bundled ESM server code

### Deploy on vishnu.parvati.it

**IMPORTANT: Follow this procedure exactly. Do not improvise or skip steps.**

1. Verify override exists: `cat docker-compose.override.yml` — if missing, the deploy WILL break
2. Run tests: `pnpm test:server`
3. Build and deploy: `docker compose down && docker compose up -d --build`
4. Verify: `sleep 25 && docker ps --filter name=crowdsec_web_ui --format '{{.Status}}'` — must show "healthy"
5. Check logs: `docker logs crowdsec_web_ui 2>&1 | tail -10` — must show sync activity, no "Authentication failed"

The `docker-compose.override.yml` provides vishnu-specific config (network_mode: host, volume path). It is gitignored and can be deleted by git operations. If missing, recreate it:

```yaml
services:
  crowdsec-web-ui:
    network_mode: host
    ports: !reset []
    volumes: !override
      - /opt/crowdsec-web-ui/data:/app/data
```

**Never merge a PR without the user explicitly confirming they have tested and approved it.**

### Tooling

- **pnpm 10** (workspace with `pnpm-workspace.yaml` for native dep builds)
- **Node 24** (pinned in `engines`)
- **TypeScript 6** with strict mode
- **ESLint** flat config (no Prettier)
- **tsup** for server bundling
