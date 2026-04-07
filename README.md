<div align="center">
  <img src="client/public/logo.svg" alt="CrowdSec Web UI Logo" width="400" />
</div>

# CrowdSec Web UI — Large Deployment Edition

A high-performance fork of [TheDuffman85/crowdsec-web-ui](https://github.com/TheDuffman85/crowdsec-web-ui), optimized for **large multi-server CrowdSec deployments** with tens of thousands of alerts and decisions.

> **Credits**: This project is based on the excellent work of [@TheDuffman85](https://github.com/TheDuffman85). All original features, architecture, and design are his. This fork adds performance optimizations and UX changes specific to large-scale deployments.

## Why this fork?

The original project works great for small deployments (a few hundred alerts, a handful of decisions). At larger scale — **~100 CrowdSec agents, 50k+ alerts, 20k+ active decisions, imported blocklists** — the dashboard took ~30 seconds to load and transferred ~37MB of JSON per request. Mobile was unusable.

This fork reduces that to **0.13 seconds and 12KB**.

## What's different from upstream

### Performance (invisible to users, no UX change)

| Optimization | Impact |
|---|---|
| SQLite pragmas (WAL, synchronous=NORMAL, 32MB cache, mmap 256MB) | 10-20% faster queries |
| Missing indexes on decisions(value, created_at) | Faster lookups |
| Batch decision hydration (N+1 fix) | Eliminates ~60k individual queries |
| In-memory response cache with sync-based invalidation | Repeat loads <50ms |
| Materialized stats columns (source_cn, target, machine_id, etc.) | Eliminates JSON.parse on stats |

### UX changes (intentional differences from upstream)

These are deliberate design choices for large deployments, not bugs:

| Change | Upstream behavior | Our behavior | Why |
|---|---|---|---|
| **Dashboard aggregation** | Client downloads all raw records (~37MB) and aggregates in JavaScript | Server runs SQL GROUP BY and returns ~12KB of pre-computed stats | 37MB per page load is not viable with 50k alerts. Mobile browsers crash. |
| **Alerts/Decisions pagination** | Infinite scroll, loads entire dataset client-side | Server-side pagination with page controls (100 items/page) + server-side SQL search | With 50k alerts, infinite scroll loads 20MB of JSON. Pagination sends 46KB per page. Search is instant via SQL LIKE instead of client-side string matching on 50k records. |
| **Machine column always visible** | Hidden unless multiple machine_ids detected at runtime, configurable via `CROWDSEC_ALWAYS_SHOW_MACHINE` | Always visible | With ~100 servers, the machine column is essential. The auto-detection adds complexity for zero benefit in our case. |
| **Origin column on Decisions** | Not shown | Visible column showing `crowdsec`, `cscli-import`, `cscli` | Essential for distinguishing organic detections from imported blocklists. |
| **Blocklist decisions visible** | Batch alerts from `cscli decisions import` (empty `source.scope`) not fetched | Unscoped LAPI fetch captures all alert types | We import Spamhaus/GreenSnow/AbuseIPDB blocklists — ~15k decisions that were invisible without this fix. |
| **Decision sort order** | Sorted by `stop_at` (expiration) | Sorted by `created_at` (creation time) | Manual bans with 10-year TTL always appeared at the top, pushing recent detections out of view. |
| **CROWDSEC_ALERT_ORIGINS default** | Defaults to `none` (no origin filter, but excludes CAPI) | Fetches all origins including unscoped alerts | Our blocklist imports create alerts with empty scope that were excluded by the scoped-only fetch. |
| **Loading state** | Shows "Loading..." replacing data on every refresh | Keeps existing data visible during background reloads | Prevents UI flicker when navigating between pages. |

### Benchmarks

Measured with ~50k alerts, ~20k active decisions, ~100 CrowdSec agents:

| Metric | Upstream | This fork |
|---|---|---|
| Dashboard load | ~30s / 37MB | 0.13s / 12KB |
| Alerts page | ~30s / 20MB | 7ms / 46KB |
| Decisions page | ~30s | 8ms / 46KB |
| Search | Client-side on full dataset | Server-side SQL, 7-40ms |
| Mobile | Unusable | Instant |

## Features (inherited from upstream)

All original features are preserved:

- **Dashboard**: High-level statistics, world map, top countries/scenarios/AS/targets
- **Alerts Management**: Detailed security event logs with event breakdown
- **Decisions Management**: Active bans with manual add/delete
- **Notification Center**: Alert spikes, thresholds, CVE detection — Email, Gotify, MQTT, ntfy, Webhooks
- **Dark/Light Mode**: Full theme support
- **Update Notifications**: Detects new container images on GHCR
- **Simulation Mode**: Shows simulated vs live alerts when enabled
- **Base Path**: Reverse proxy deployments supported

> [!CAUTION]
> **Security Notice**: This application **does not provide any built-in authentication mechanism**. Deploy behind a reverse proxy with an Identity Provider (Authentik, Authelia, Keycloak, etc.).

## Quick Start

### Prerequisites

- A running CrowdSec instance
- Machine authentication configured (watcher password or mTLS)

```bash
# Generate password and register machine
openssl rand -hex 32
docker exec crowdsec cscli machines add crowdsec-web-ui --password <generated_password> -f /dev/null
```

### Docker Compose

```yaml
services:
  crowdsec-web-ui:
    build: https://github.com/eagle1maledetto/crowdsec-web-ui.git
    container_name: crowdsec_web_ui
    ports:
      - "3000:3000"
    environment:
      - CROWDSEC_URL=http://crowdsec:8080
      - CROWDSEC_USER=crowdsec-web-ui
      - CROWDSEC_PASSWORD=<generated_password>
      # Optional
      - CROWDSEC_SIMULATIONS_ENABLED=true
      - CROWDSEC_LOOKBACK_PERIOD=7d
      - CROWDSEC_REFRESH_INTERVAL=30s
    volumes:
      - ./data:/app/data
    restart: unless-stopped
```

### Environment Variables

| Variable | Default | Description |
|---|---|---|
| `CROWDSEC_URL` | required | CrowdSec LAPI URL |
| `CROWDSEC_USER` | — | Machine login (password auth) |
| `CROWDSEC_PASSWORD` | — | Machine password (password auth) |
| `CROWDSEC_TLS_CERT_PATH` | — | Agent certificate path (mTLS auth) |
| `CROWDSEC_TLS_KEY_PATH` | — | Agent key path (mTLS auth) |
| `CROWDSEC_TLS_CA_CERT_PATH` | — | CA certificate path (mTLS auth) |
| `CROWDSEC_LOOKBACK_PERIOD` | `168h` | How far back to sync alerts (e.g. `7d`, `168h`) |
| `CROWDSEC_REFRESH_INTERVAL` | `30s` | Auto-refresh interval (`0`=off, `5s`, `30s`, `1m`, `5m`) |
| `CROWDSEC_IDLE_REFRESH_INTERVAL` | `5m` | Refresh interval when no users active |
| `CROWDSEC_IDLE_THRESHOLD` | `2m` | Time without requests before idle mode |
| `CROWDSEC_FULL_REFRESH_INTERVAL` | `5m` | Interval for full cache resync |
| `CROWDSEC_SIMULATIONS_ENABLED` | `false` | Show simulated alerts/decisions |
| `CROWDSEC_ALERT_ORIGINS` | — | CSV of LAPI alert origin filters |
| `CROWDSEC_ALERT_EXTRA_SCENARIOS` | — | CSV of extra scenarios to include |
| `BASE_PATH` | — | URL path prefix for reverse proxy (e.g. `/crowdsec`) |
| `PORT` | `3000` | Server port |
| `DB_DIR` | `/app/data` | SQLite database directory |
| `NOTIFICATION_SECRET_KEY` | auto | Encryption key for notification secrets |
| `NOTIFICATION_ALLOW_PRIVATE_ADDRESSES` | `true` | Allow private IPs in webhook destinations |

### Health Check

```bash
curl http://localhost:3000/api/health
# {"status":"ok"}
```

## API Endpoints

### Paginated endpoints

```
GET /api/alerts?page=1&page_size=100
GET /api/alerts?page=1&page_size=100&q=ssh-bf
GET /api/alerts?page=1&q=vishnu&scenario=ssh-bf&country=CN
GET /api/decisions?page=1&page_size=100
GET /api/decisions?page=1&q=cscli-import
```

Without `page` parameter, returns the full array (backward compatible).

### Dashboard aggregation

```
GET /api/stats/dashboard?granularity=day
GET /api/stats/dashboard?granularity=hour&country=CN&scenario=ssh-bf
```

Returns ~12KB of pre-aggregated stats instead of ~37MB of raw records.

## Persistence

All data is stored in SQLite at `/app/data/crowdsec.db`. Mount `/app/data` to persist across restarts.

Alerts are kept for the duration of `CROWDSEC_LOOKBACK_PERIOD` (default 7 days), then cleaned up automatically. To force a full resync: `POST /api/cache/clear`.

## Local Development

```bash
# Requirements: Node.js 24, pnpm 10
pnpm install
pnpm dev          # server (3000) + client (5173)
pnpm test         # run all tests (73 server + client)
pnpm build        # production build
pnpm start        # start production server
```

## Relationship to upstream

This is an independent fork of [TheDuffman85/crowdsec-web-ui](https://github.com/TheDuffman85/crowdsec-web-ui). The upstream project is designed for small to medium deployments and makes different UX choices that are appropriate for that scale. This fork is tailored for large multi-server deployments where performance at scale is the priority.

We periodically sync useful features from upstream, but the two projects have diverged in ways that make a full merge impractical. See [issue #192](https://github.com/TheDuffman85/crowdsec-web-ui/issues/192) for the full discussion.

## License

[GNU Affero General Public License v3.0](LICENSE) — same as the original project.
