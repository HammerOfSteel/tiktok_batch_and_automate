# Development guide

Everything runs in Docker Compose, frontend included. If a step here requires installing a language runtime on
the host, that is a bug in this document.

## Prerequisites

| Tool | Version | Why |
| --- | --- | --- |
| Docker Desktop or Docker Engine | 24 or newer, with Compose v2 | Runs the whole stack |
| git | any recent | |
| pnpm | 9.x | Only needed for running scripts on the host. Optional |

Node is **not** a prerequisite. It lives in the containers.

## First run

```bash
git clone https://github.com/HammerOfSteel/tiktok_batch_and_automate.git
cd tiktok_batch_and_automate
cp .env.example .env
docker compose up
```

First boot builds images, runs migrations and seeds the database. Later boots are fast.

| Service | URL | Notes |
| --- | --- | --- |
| Web | http://localhost:5173 | SvelteKit with hot module reload |
| API | http://localhost:3000 | NestJS with watch mode |
| Swagger UI | http://localhost:3000/docs | Generated from the OpenAPI spec |
| MinIO console | http://localhost:9001 | Login from `.env` |
| Mailpit | http://localhost:8025 | Catches every outbound email |
| Queue dashboard | http://localhost:3000/admin/queues | Development only |
| Postgres | localhost:5432 | Exposed for a GUI client |
| Redis | localhost:6379 | |

## The stack

| Container | Purpose |
| --- | --- |
| `web` | SvelteKit dev server, source bind mounted |
| `api` | NestJS HTTP server, source bind mounted |
| `worker` | The same image as `api`, different entry point, consumes queues |
| `postgres` | Primary database |
| `redis` | Queues, rate limiting, cache |
| `minio` | S3 compatible object storage for uploads |
| `mailpit` | SMTP sink plus a web UI |

`api` and `worker` share an image on purpose. See [architecture.md](architecture.md).

### Compose conventions

- Bind mount source, use a **named volume for `node_modules`** so host and container never fight over binaries.
- Every service has a healthcheck. `depends_on` uses `condition: service_healthy` so boot order is deterministic
  rather than lucky.
- `compose.yaml` is the development stack. `compose.prod.yaml` is the production overlay. No `if development`
  branches inside application code.
- Nothing in the repository ever contains a real secret. `.env` is gitignored, `.env.example` is committed.

## Day to day

```bash
docker compose up                 # start everything
docker compose up -d              # detached
docker compose logs -f api        # follow one service
docker compose restart api        # after a dependency change
docker compose down               # stop
docker compose down -v            # stop and wipe data, full reset

docker compose exec api sh        # shell into the API container
docker compose exec postgres psql -U app -d app
```

### Database

```bash
docker compose exec api pnpm prisma migrate dev --name add_video_notes
docker compose exec api pnpm prisma migrate reset      # drop, migrate, seed
docker compose exec api pnpm prisma studio             # browse data
docker compose exec api pnpm seed                      # reseed only
```

Migrations are committed with the code change that needs them, in the same commit.

### Tests

```bash
docker compose exec api pnpm test:unit
docker compose exec api pnpm test:integration     # spins up its own containers
pnpm test:e2e                                     # host side, needs the stack running
pnpm smoke
```

### The API contract

The OpenAPI document in `packages/contracts` is the source of truth.

```bash
pnpm contracts:lint       # Spectral, Zalando ruleset
pnpm contracts:generate   # regenerate the typed client for the web app
```

Change the spec first, regenerate, then implement. A pull request that changes an endpoint without changing the
spec will fail CI.

## Adding a shadcn-svelte component

```bash
docker compose exec web pnpm dlx shadcn-svelte@latest add dialog
```

Components land in `src/lib/components/ui`. Treat them as our code from that point on: they are vendored, not a
dependency, so they can be adapted. Record any non trivial adaptation in a comment explaining why.

## Environment variables

Every variable is documented in `.env.example` and validated at boot by a schema. A missing or malformed value
stops the process immediately with a readable message.

| Group | Examples |
| --- | --- |
| Core | `NODE_ENV`, `LOG_LEVEL`, `APP_URL`, `API_URL` |
| Database | `DATABASE_URL` |
| Redis | `REDIS_URL` |
| Storage | `S3_ENDPOINT`, `S3_BUCKET`, `S3_ACCESS_KEY`, `S3_SECRET_KEY` |
| Auth | `JWT_SECRET`, `SESSION_COOKIE_DOMAIN`, `ARGON_MEMORY_COST` |
| Encryption | `TOKEN_ENCRYPTION_KEY` (see [security.md](security.md)) |
| TikTok | `TIKTOK_CLIENT_KEY`, `TIKTOK_CLIENT_SECRET`, `TIKTOK_REDIRECT_URI` |
| Adapters | `TIKTOK_ADAPTER=seeded\|live` |
| Mail | `SMTP_HOST`, `SMTP_PORT`, `MAIL_FROM` |

## Working against seeded data versus live TikTok

`TIKTOK_ADAPTER=seeded` is the default and is what CI uses. Everything works, nothing leaves the machine.

Switching to `live` needs a registered developer app and credentials in `.env`. See
[tiktok-integration.md](tiktok-integration.md). Local OAuth redirects require a public callback URL, so use a
tunnel:

```bash
cloudflared tunnel --url http://localhost:3000
# then set TIKTOK_REDIRECT_URI to the tunnel URL and register it in the developer portal
```

## Debugging

- **API:** the container exposes the Node inspector on 9229. Attach with the VS Code configuration in
  `.vscode/launch.json`.
- **Web:** browser devtools. SvelteKit source maps are enabled in development.
- **Queues:** the queue dashboard shows waiting, active, completed and failed jobs, with payloads.
- **Requests:** every log line carries a `trace_id`. Grab it from a `problem+json` response and grep the logs.
- **Database:** Prisma Studio, or psql, or any GUI on port 5432.

## Common problems

| Symptom | Cause and fix |
| --- | --- |
| `api` restarts in a loop on first boot | Postgres not ready yet. Check the healthcheck, then `docker compose logs postgres` |
| Web shows a blank page and a CORS error | `API_URL` in `.env` does not match where the API is actually reachable from the browser |
| Changes to `web` do not hot reload | Bind mount lost, usually after a Docker Desktop update. `docker compose down && docker compose up --build` |
| `node_modules` errors after adding a dependency | Rebuild the image, dependencies are installed at build time: `docker compose build web` |
| Port already in use | Something else owns 5173, 3000, 5432 or 6379. Change the host port mapping, not the container port |
| Migrations conflict after a rebase | `pnpm prisma migrate reset` locally, then regenerate a single clean migration |
| E2E flaky locally but green in CI | Usually a real race in the app. Do not add a wait, find the race |

## Editor setup

Recommended VS Code extensions are in `.vscode/extensions.json`: Svelte, ESLint, Prettier, Prisma, Playwright,
Docker. Format on save is on, and the workspace pins Prettier as the formatter so nobody reformats the codebase
by accident.

## Before you push

```bash
pnpm lint && pnpm typecheck && pnpm test && pnpm contracts:lint
```

CI runs the same commands. Running them locally first is faster than waiting for a red pipeline.
