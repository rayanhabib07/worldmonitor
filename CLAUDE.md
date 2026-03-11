# CLAUDE.md — World Monitor

This file provides guidance for AI assistants working in this codebase.

## Project Overview

**World Monitor** is a real-time OSINT (Open-Source Intelligence) dashboard built with **Vanilla TypeScript** — no UI framework. It aggregates geopolitical, military, market, and infrastructure data into a live map-based interface. The app targets three distinct audiences through a variant system sharing a single codebase.

- **Production web**: [worldmonitor.app](https://www.worldmonitor.app)
- **API gateway**: [api.worldmonitor.app](https://api.worldmonitor.app)
- **License**: AGPL-3.0-only

---

## Tech Stack

| Layer | Technology |
|---|---|
| Language | TypeScript (strict mode, ES2020) |
| Frontend build | Vite 6 + PWA plugin |
| Map rendering | MapLibre GL 5 (base map), deck.gl 9 (WebGL overlays) |
| Data viz | D3 |
| API framework | **Sebuf** — custom Proto-first HTTP RPC (see below) |
| Protobuf toolchain | Buf CLI + custom protoc-gen plugins |
| Desktop app | Tauri v2 (Rust + Node.js sidecar) |
| Edge functions | Vercel Edge Functions |
| Backend-as-a-service | Convex (email registration only) |
| E2E testing | Playwright |
| Unit/data tests | Node.js native test runner (`tsx --test`) |
| Error tracking | Sentry |
| Analytics | Vercel Analytics |
| Caching | Upstash Redis |
| i18n | i18next (14 languages) |
| Node.js version | 22 (see `.nvmrc`) |

---

## Directory Structure

```
worldmonitor/
├── src/                        # Frontend source
│   ├── App.ts                  # Root app class
│   ├── main.ts                 # Entry point (Sentry, analytics, App init)
│   ├── components/             # UI components — Panel subclasses, map, modals (~70 files)
│   ├── services/               # Data fetching, client wrappers, AI, signal analysis (~80 files)
│   ├── config/                 # Static data and variant configs
│   │   ├── variants/           # Per-variant panel/layer/feed overrides (base/full/tech/finance/happy/commodity)
│   │   ├── variant-meta.ts     # OG metadata per variant (keep in sync with middleware.ts)
│   │   ├── feeds.ts            # 100+ RSS feed definitions
│   │   ├── map-layer-definitions.ts
│   │   └── ...                 # airports, bases, military, pipelines, ports, etc.
│   ├── generated/              # AUTO-GENERATED — do not edit by hand
│   │   ├── client/             # Sebuf TypeScript clients (e.g. MarketServiceClient)
│   │   └── server/             # Sebuf server route factories
│   ├── types/                  # TypeScript type definitions
│   ├── locales/                # i18n JSON files
│   ├── workers/                # Web Workers (ML analysis)
│   ├── utils/                  # DOM helpers, formatting, geo utils
│   └── app/                    # App subsystem managers (panel layout, data loader, etc.)
├── server/                     # Sebuf handler implementations (per-domain)
│   ├── gateway.ts              # Shared edge function gateway (CORS, auth, rate limiting, cache headers)
│   ├── router.ts               # Request routing
│   ├── cors.ts                 # CORS helpers
│   ├── error-mapper.ts         # Error → HTTP response mapping
│   └── worldmonitor/           # Domain handlers: aviation, climate, conflict, cyber, …
│       └── {domain}/v1/handler.ts
├── api/                        # Vercel Edge Functions
│   ├── [domain]/v1/[rpc].ts    # Sebuf gateway per domain (auto-routes to server/ handlers)
│   └── *.js                    # Legacy standalone endpoints (RSS proxy, AIS snapshot, etc.)
├── proto/                      # Protobuf definitions
│   └── worldmonitor/{domain}/v1/
├── src-tauri/                  # Tauri v2 desktop app
│   ├── src/                    # Rust source
│   ├── sidecar/                # Node.js local API server for desktop offline mode
│   ├── tauri.conf.json         # Full variant desktop config
│   ├── tauri.tech.conf.json
│   └── tauri.finance.conf.json
├── blog-site/                  # Astro-based blog (built into public/blog/)
├── pro-test/                   # Pro landing page
├── convex/                     # Convex schema + mutations (email registration)
├── data/                       # Static JSON datasets
├── docs/                       # Mintlify documentation + generated OpenAPI specs
├── e2e/                        # Playwright end-to-end tests
├── tests/                      # Data integrity unit tests
├── scripts/                    # Build, packaging, and utility scripts
├── public/                     # Static assets served as-is
├── middleware.ts               # Vercel Edge Middleware (bot filtering, OG for variants)
├── vite.config.ts              # Vite config (variant injection, brotli pre-compression, PWA)
├── tsconfig.json               # Frontend TypeScript config
├── tsconfig.api.json           # API/server TypeScript config
├── vercel.json                 # Vercel routing, headers, CSP, caching
├── Makefile                    # Proto toolchain commands
└── package.json                # NPM scripts and dependencies
```

---

## Variant System

The same codebase produces multiple app flavors for different audiences:

| Variant | Dev command | Domain | Focus |
|---|---|---|---|
| `full` | `npm run dev` | worldmonitor.app | Geopolitics, military, conflict, infrastructure |
| `tech` | `npm run dev:tech` | tech.worldmonitor.app | AI/ML, startups, cloud, cybersecurity |
| `finance` | `npm run dev:finance` | finance.worldmonitor.app | Markets, trading, central banks, commodities |
| `happy` | `npm run dev:happy` | happy.worldmonitor.app | Positive news, progress data |
| `commodity` | `npm run dev:commodity` | — | Commodities focus |

Variant configs live in `src/config/variants/`. Variants share all code and differ only in default panels, map layers, and RSS feeds. The active variant is determined by:
- `VITE_VARIANT` env var at build time (desktop builds)
- Runtime hostname detection in the browser (web builds)

When `variant-meta.ts` changes, also update `middleware.ts` OG metadata (they must be kept in sync).

---

## Development Commands

```bash
# First-time setup
make install           # Installs buf CLI, sebuf plugins, npm deps, Playwright browsers

# Development server
npm run dev            # full variant — http://localhost:3000
npm run dev:tech       # tech variant
npm run dev:finance    # finance variant

# Type checking (run before committing)
npm run typecheck      # Frontend (src/)
npm run typecheck:api  # API/server code
npm run typecheck:all  # Both

# Testing
npm run test:data      # Data integrity tests (tests/*.test.mts)
npm run test:sidecar   # Sidecar/API unit tests
npm run test:feeds     # RSS feed validation
npm run test:e2e       # Playwright e2e (requires built app or dev server)
npm run test:e2e:full  # Full variant e2e
npm run test:e2e:tech  # Tech variant e2e

# Linting
npm run lint:md        # Markdown lint
make lint              # Lint proto files
make breaking          # Check proto breaking changes vs main

# Production builds
npm run build          # full variant (includes blog build)
npm run build:tech
npm run build:finance
npm run build:full

# Proto code generation (when .proto files change)
make generate          # Regenerates src/generated/ from proto/

# Desktop builds
npm run desktop:dev    # Local Tauri dev
npm run desktop:build:full
npm run desktop:build:tech

# Markdown lint
npm run lint:md
```

**Key**: Always run `npm run typecheck` before opening a PR. CI enforces it.

---

## Sebuf — The RPC Framework

Sebuf is the project's custom **Proto-first HTTP RPC** framework. All typed API communication between frontend and backend uses it.

### Data flow

```
proto/worldmonitor/{domain}/v1/*.proto
    → make generate
    → src/generated/client/{domain}/  (TypeScript client stubs)
    → src/generated/server/{domain}/  (server route factories)

Frontend:
  src/services/{domain}/index.ts  → wraps generated client → calls api/{domain}/v1/[rpc].ts

Backend:
  api/{domain}/v1/[rpc].ts  → createDomainGateway(routes)  → server/worldmonitor/{domain}/v1/handler.ts
```

### Proto conventions

- **Time fields**: Use `int64` (Unix epoch milliseconds), **not** `google.protobuf.Timestamp`
- **int64 encoding**: Add `[(sebuf.http.int64_encoding) = INT64_ENCODING_NUMBER]` on time fields so TypeScript receives `number` not `string`
- **HTTP annotations**: Every RPC needs `option (sebuf.http.config) = { path: "...", method: POST }`

### Adding a new RPC method

1. Edit the `.proto` file in `proto/worldmonitor/{domain}/v1/`
2. Run `make generate` (regenerates `src/generated/`)
3. Implement the method in `server/worldmonitor/{domain}/v1/handler.ts`
4. Use the generated client from `src/services/{domain}/`

> **Never edit `src/generated/` by hand.** Those files are overwritten by `make generate`.

For non-JSON payloads (XML feeds, binary, HTML embeds), add a standalone edge function in `api/` instead of Sebuf. For JSON APIs, always prefer Sebuf for the typed contracts.

---

## API Architecture

Vercel Edge Functions in `api/` serve as the API gateway. They are split by domain to minimize cold-start bundle sizes (~20× smaller per function).

### Auth & rate limiting (`server/gateway.ts`)

- API key validation via `X-WorldMonitor-Key` header or `?key=` query param
- Upstash Redis-backed rate limiting per endpoint
- CORS handled centrally; bot traffic blocked in `middleware.ts`

### Cache tiers (defined in `server/gateway.ts`)

| Tier | `s-maxage` | Use case |
|---|---|---|
| `fast` | 5 min | Frequently updated data |
| `medium` | 15 min | Semi-live data |
| `slow` | 1 hour | Infrequently changing data |
| `static` | 24 hours | Near-static reference data |
| `daily` | 24 hours | Daily updates |
| `no-store` | — | Real-time or user-specific |

---

## Frontend Conventions

### TypeScript

- **Strict mode** with `noUnusedLocals`, `noUnusedParameters`, `noUncheckedIndexedAccess`
- No `any` — use proper typing or `unknown` with type guards
- Path alias `@/*` maps to `src/*`
- `const` by default; `let` only when reassignment is needed
- Prefer functional patterns (map, filter, reduce)

### Component model (Panel system)

All dashboard panels extend the `Panel` base class (`src/components/Panel.ts`). Each panel is a class with:
- A constructor receiving `PanelOptions` + app state
- A `render()` or equivalent update method
- DOM built with `h()` and `safeHtml()` utilities from `src/utils/dom-utils.ts`

### No UI framework

The project uses **no React/Vue/Svelte**. UI is vanilla DOM manipulation. Don't introduce framework dependencies.

### File placement

| What | Where |
|---|---|
| UI panels/components | `src/components/` |
| Data fetching / services | `src/services/` |
| Static geo/config data | `src/config/` |
| Sebuf handler impls | `server/worldmonitor/{domain}/v1/handler.ts` |
| Edge function gateway | `api/{domain}/v1/[rpc].ts` |
| Proto definitions | `proto/worldmonitor/{domain}/v1/` |
| i18n strings | `src/locales/{lang}.json` |

### Adding a data layer

1. Define the proto service (if backend proxy needed)
2. `make generate`
3. Implement handler in `server/worldmonitor/{domain}/v1/handler.ts`
4. Register in `api/[domain]/v1/[rpc].ts` and `vite.config.ts` dev proxy
5. Create service module in `src/services/{domain}/`
6. Add layer config + renderer following existing layer patterns
7. Wire into layer toggles UI
8. Document in `docs/DOCUMENTATION.md`

---

## Environment Variables

Copy `.env.example` → `.env.local`. All keys are optional — the app works without them but corresponding features are disabled.

Key categories:

| Category | Key(s) | Required for |
|---|---|---|
| AI summarization | `GROQ_API_KEY`, `OPENROUTER_API_KEY` | AI news summaries |
| Cache | `UPSTASH_REDIS_REST_URL`, `UPSTASH_REDIS_REST_TOKEN` | Cross-user caching |
| Market data | `FINNHUB_API_KEY` | Stock quotes |
| Energy | `EIA_API_KEY` | Oil/production data |
| Economic | `FRED_API_KEY` | FRED economic data |
| Aviation | `AVIATIONSTACK_API`, `ICAO_API_KEY` | Flight data |
| Aircraft tracking | `WINGBITS_API_KEY` | Aircraft enrichment |
| Conflict data | `ACLED_ACCESS_TOKEN`, `UCDP_ACCESS_TOKEN` | Conflict events |
| Internet outages | `CLOUDFLARE_API_TOKEN` | Radar outage data |
| Satellite fires | `NASA_FIRMS_API_KEY` | Fire detection |
| AIS vessels | `AISSTREAM_API_KEY` | Live ship positions |
| Aircraft | `OPENSKY_CLIENT_ID`, `OPENSKY_CLIENT_SECRET` | OpenSky network |
| Telegram OSINT | `TELEGRAM_API_ID`, `TELEGRAM_API_HASH`, `TELEGRAM_SESSION` | Telegram ingestion |
| Relay | `WS_RELAY_URL`, `RELAY_SHARED_SECRET`, `VITE_WS_RELAY_URL` | AIS/aircraft relay |
| Site config | `VITE_VARIANT`, `VITE_SENTRY_DSN`, `VITE_WS_API_URL` | Core app config |
| Map tiles | `VITE_PMTILES_URL`, `VITE_PMTILES_URL_PUBLIC` | Self-hosted tiles |
| Desktop auth | `WORLDMONITOR_VALID_KEYS` | Desktop cloud fallback |
| Registration | `CONVEX_URL` | Email registration DB |

---

## CI/CD

| Workflow | Trigger | What it does |
|---|---|---|
| `typecheck.yml` | PRs + pushes to `main` | `npm run typecheck` + `npm run typecheck:api` |
| `lint.yml` | PRs touching `.md` files | `npm run lint:md` |
| `test-linux-app.yml` | Manual (`workflow_dispatch`) | Full Tauri build + AppImage smoke test |
| `build-desktop.yml` | Automated | Desktop app build + packaging |
| `docker-publish.yml` | Automated | Docker image publishing |

Vercel handles preview and production deploys automatically. The `scripts/vercel-ignore.sh` script prevents unnecessary deploys for non-web paths.

---

## Desktop App (Tauri v2)

The desktop app wraps the web UI in Tauri. Key differences from web:

- Runs a **Node.js sidecar** (`src-tauri/sidecar/`) as a local API server for offline data
- Uses `VITE_DESKTOP_RUNTIME=1` env var during build
- Has a `tauri-bridge.ts` service for IPC with Rust
- Three variant configs: `tauri.conf.json` (full), `tauri.tech.conf.json`, `tauri.finance.conf.json`
- Version must be synced before builds: `npm run version:sync`

Desktop build commands:
```bash
npm run desktop:dev          # Local dev with devtools
npm run desktop:build:full   # Release build (full variant)
npm run desktop:build:tech
```

---

## Git & PR Conventions

### Commit / PR title format

```
feat: add earthquake magnitude filtering
fix: resolve RSS feed timeout for Al Jazeera
perf: optimize marker clustering at low zoom levels
refactor: extract threat classifier into separate module
docs: update API dependencies section
chore: upgrade @vercel/analytics to v2
```

### Before opening a PR

1. `npm run typecheck` — must pass
2. Test the affected variant(s) locally
3. Run `make lint` if proto files were changed
4. Run `make breaking` to verify no breaking proto changes vs `main`
5. Keep PRs focused: one feature or fix per PR

---

## Important Files to Know

| File | Purpose |
|---|---|
| `src/App.ts` | Root app class — orchestrates all subsystems |
| `src/main.ts` | Entry point — Sentry init, analytics, app bootstrap |
| `src/config/index.ts` | Central config exports (variants, storage keys, intervals) |
| `src/config/feeds.ts` | 100+ RSS feed definitions with tier/category metadata |
| `src/config/variant-meta.ts` | OG/SEO metadata per variant |
| `src/components/Panel.ts` | Base Panel class all dashboard panels extend |
| `server/gateway.ts` | Edge function gateway — CORS, auth, rate limiting, cache |
| `middleware.ts` | Vercel middleware — bot filtering, variant OG responses |
| `vite.config.ts` | Build config — variant injection, dev proxies, PWA, brotli |
| `vercel.json` | Vercel routing rules, security headers, CSP |
| `Makefile` | Proto toolchain targets |
| `proto/worldmonitor/` | All Sebuf service/message definitions |
| `src/generated/` | Auto-generated stubs — **never edit manually** |

---

## What Not to Do

- **Don't edit `src/generated/`** — run `make generate` instead
- **Don't introduce UI frameworks** (React, Vue, Svelte) — the project is deliberately framework-free
- **Don't use `any`** in TypeScript — use `unknown` with type guards or proper types
- **Don't add `google.protobuf.Timestamp`** — use `int64` epoch milliseconds in protos
- **Don't add paid APIs** as core functionality requirements
- **Don't bypass TypeScript strict checks** — fix the types properly
- **Don't edit `middleware.ts` OG data without also updating `src/config/variant-meta.ts`** (and vice versa)
