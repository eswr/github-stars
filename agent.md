# GitHub Stars (Effect) Monorepo Guide

This repo is a compact, production-style "Effect-first" monorepo that mirrors common patterns:

- Effect runtime + Layers for dependency injection and lifecycle management
- Schema-first contracts shared between server and client
- SQLite + Drizzle for storage + migrations
- FTS5 for fast local search
- Observability via OpenTelemetry (traces + Prometheus metrics)
- React Native (Expo) client using React Query
- Detox-based E2E scaffolding

---

## Repo map

### Apps

#### `apps/github-stars-effect-server/` (Node + Effect HTTP API)
Service responsibilities:
1. Refresh your GitHub starred repos (paginated) via GitHub API
2. Upsert into SQLite
3. Expose endpoints:
   - `GET /stars?q=...` for full-text search
   - `POST /refresh-stars` to pull latest stars
   - Returns typed `GetStarsResponseSchema`

Key files:
- `src/main.ts` — composition root: provides layers + starts server
- `src/app.ts` — API definition + handlers (Effect-HTTP)
- `src/layers/*` — dependency layers (config, sqlite client, drizzle, telemetry)
- `src/repository/*` — ports (interfaces) + adapters (live implementations)
- `src/db/schema.ts` — DB schema used at runtime (includes FTS5 view)
- `drizzle.config.ts`, `drizzle/*` — Drizzle config + generated artifacts

#### `apps/github-stars-effect/` (Expo / React Native client)
Client responsibilities:
- Single screen `SearchStarsScreen` to query local stars
- React Query for caching
- Effect Platform HTTP client in `fetchStars.tsx` for decode/validation

Key files:
- `src/components/data/fetchStars.tsx` — HTTP call + schema decode
- `src/components/screens/SearchStarsScreen.tsx` — UI + query orchestration
- `src/providers/*` — app-wide providers (fonts, tamagui, react-query)
- `src/fixes/fixMissingTextEncoder.ts` — Hermes TextEncoder fix

#### `apps/github-stars-effect-e2e/` (Detox)
Basic E2E scaffolding.

---

### Libraries

#### `libs/github-stars/shared/schema/`
Shared API contract using `@effect/schema`.
- `src/index.ts` defines `GetStarsResponseSchema`, `StarredRepoSchema`, etc.
- Used by server (API contract + runtime validation) and client (decode + type inference)

This is the single source of truth for API shape.

---

### Presentation
`presentation/` contains Slidev theme + snippets. Not required to run the app.

---

## Architecture at a glance

### Server flow: refresh pipeline
`POST /refresh-stars` (in `src/app.ts`) runs:
1. `GithubApiRepository.getStarred({ page })` — HTTP request to GitHub API
2. Map to DB records
3. `StarsDbRepository.insertOrUpdateStarredRepo(...)` — batched upsert via Drizzle
4. Iterate pages until GitHub returns an empty list
5. Run multiple workers in parallel (see `GITHUB_STARS_FETCH_CONCURRENCY` in `src/app.ts`).

### Server flow: query pipeline
`GET /stars?q=...`:
1. Validate query param (schema)
2. `StarsDbRepository.fullTextSearch(q)` — join FTS view + order by rank
3. Return `{ results, error: null }`

---

## Where to start reading

1. Composition root
   - `apps/github-stars-effect-server/src/main.ts`
   - See how Layers are provided and how the app is launched.
2. HTTP API + handlers
   - `apps/github-stars-effect-server/src/app.ts`
   - Schema-first endpoint definition, error mapping, spans, and middleware in `effect-http`.
3. Config + runtime dependencies
   - `apps/github-stars-effect-server/src/layers/AppConfig.ts`
   - `apps/github-stars-effect-server/src/layers/SqliteClient.ts`
   - `apps/github-stars-effect-server/src/layers/SqliteDrizzle.ts`
   - `apps/github-stars-effect-server/src/layers/OpenTelemetryService.ts`
4. Ports / adapters
   - `apps/github-stars-effect-server/src/repository/githubApi/*`
   - `apps/github-stars-effect-server/src/repository/starsDb/*`
5. Shared schema + client decode
   - `libs/github-stars/shared/schema/src/index.ts`
   - `apps/github-stars-effect/src/components/data/fetchStars.tsx`

---

## Key patterns to copy (and why they matter)

### 1) Context Tags as interfaces
`Context.Tag("X")` defines your port.
- Encourages testing + swapping implementations.

### 2) Layers as wiring + lifecycle
`Layer.effect(...)` / `Layer.scopedContext(...)` keeps construction separate from usage.
- Enables test layers (in-memory DB, fake GitHub API, etc.).

### 3) Schema-first everything
- GitHub responses are validated via schema decode.
- API responses use shared schemas.
- DB schemas map to typed domain records.

### 4) Concurrency done right
- `Effect.iterate` for paging loops
- Multiple workers + `Effect.all({ concurrency })` for controlled parallelism

### 5) Observability built in
- Spans around endpoint + repository calls
- Prometheus exporter (when enabled)

### 6) Runtime boundaries
- Client uses Effect for HTTP + schema decode only; UI remains React Query + Tamagui.

---

## How to run (typical local dev)

### Prereqs
- Node + your package manager (repo appears Nx-based)
- GitHub token (`GITHUB_TOKEN`) with access to read starred repos
- `APP_ROOT_DIR` and optionally `DB_FILENAME` and `PORT`

### Env (minimum)
- `APP_ROOT_DIR` — absolute path to `apps/github-stars-effect-server`
- `GITHUB_TOKEN`
- Optional:
  - `PORT` (default 4000)
  - `DB_FILENAME` (default `${APP_ROOT_DIR}/db/github-stars.db`)
- `dotenv-flow` is used (`import "dotenv-flow/config"`), so `.env*` files are supported (for example: `.env`, `.env.local`).

### Start server
- Run the Nx target or package script that starts `apps/github-stars-effect-server`.
- Hit:
  - `POST http://localhost:4000/refresh-stars`
  - `GET  http://localhost:4000/stars?q=effect`

### Quick commands (examples)
Examples (targets may vary by Nx config):
- `nx serve github-stars-effect-server`
- `nx serve github-stars-effect --web`

### Start client
- Expo web: open `http://localhost:8081` (or the port shown by Expo)
- Search from UI (min length 3)

Note: client uses `http://localhost:4000/stars` in `fetchStars.tsx`.
On device/emulator you will need to replace `localhost` with your machine IP.

---

## Testing

### Unit tests
- Jest configs exist for server, client, and shared schema libs.

### E2E
- Detox setup in `apps/github-stars-effect-e2e/`
- `app.spec.ts` checks a basic heading.

---

## Gotchas / TODOs you will likely hit

1. Mobile networking
   - `localhost` from a phone is not your dev machine. Use LAN IP or Expo config.
2. FTS5 setup
   - The project expects `starred_repo_idx` to exist (declared as an existing SQLite view via Drizzle).
   - Ensure migrations or DB initialization create the FTS table/view.
3. GitHub rate limits
   - Paging + concurrency can hit rate limits; consider backoff/retry schedules.
4. OpenTelemetry toggle
   - `ENABLE_DOCKER_TELEMETRY` controls exporters.
   - If you do not run collectors, switch to console exporter for quick debugging.

---

## Suggested next extensions (good learning tasks)

1. Add `GET /health` and include a DB connectivity check.
2. Add retry with exponential backoff for GitHub API calls (`Schedule.exponential`).
3. Add `GET /stars/:id` endpoint to fetch a single repo record.
4. Add `POST /refresh-stars?since=...` to update incrementally.
5. Add caching to the GitHub API layer (ETags or KV store).

---

## Glossary (repo terms)

- Effect: typed, lazy computation: `Effect<A, E, R>`
- Layer: a recipe to build and provide dependencies (services) to Effects
- Tag: typed token for a service in the Effect environment
- Schema: runtime validation + type inference (decode/encode)
- Scoped: resource-safe execution; ensures finalizers run on interruption

---

## File index

### Server
- `apps/github-stars-effect-server/src/main.ts`
- `apps/github-stars-effect-server/src/app.ts`
- `apps/github-stars-effect-server/src/layers/AppConfig.ts`
- `apps/github-stars-effect-server/src/layers/OpenTelemetryService.ts`
- `apps/github-stars-effect-server/src/layers/SqliteClient.ts`
- `apps/github-stars-effect-server/src/layers/SqliteDrizzle.ts`
- `apps/github-stars-effect-server/src/repository/githubApi/*`
- `apps/github-stars-effect-server/src/repository/starsDb/*`
- `apps/github-stars-effect-server/src/db/schema.ts`

### Client
- `apps/github-stars-effect/src/components/data/fetchStars.tsx`
- `apps/github-stars-effect/src/components/screens/SearchStarsScreen.tsx`

### Shared
- `libs/github-stars/shared/schema/src/index.ts`
