# TODO: phased delivery plan

The single source of truth for what gets built, in what order, and how we know it is done.

## How to use this document

- Work **top to bottom**. Tasks inside a phase may be parallelised only where marked `[parallel]`.
- Every task is numbered `<phase>.<group>.<task>`, for example `1.2.3`. Reference the number in commits.
- **Commit after every completed task.** The commit message is given with the task.
- **Push at the end of each phase**, after the phase gate passes. Not before.
- A **gate** is a hard stop. If a gate fails, fix it before starting the next phase. No "we will come back to it".
- Tick boxes as you go: `- [ ]` becomes `- [x]`. Keep this file honest, it is the project status board.

### Legend

| Marker | Meaning |
| --- | --- |
| `S-xx` | Research spike, timeboxed, output is a written decision |
| `[parallel]` | Can be worked at the same time as its siblings |
| `[blocked-by X]` | Cannot start until X is done |
| **Gate** | Quality checkpoint, all listed checks must pass |

### Definition of done, for every task

1. Code compiles, lints clean and is type safe.
2. Tests for the change exist and pass at the level required by [testing-strategy.md](testing-strategy.md).
3. Docs touched by the change are updated in the same commit.
4. No new `TODO` comment without a corresponding entry in this file.
5. Committed with a conventional commit message referencing the task number.

---

## Phase 0: foundations

**Goal:** decisions are made, the skeleton runs, CI is green.
**Exit:** `docker compose up` serves web and API on a clean machine, and CI passes on a pull request.

### 0.1 Research spikes and decisions

Timebox each spike. The output of a spike is an ADR, never code that ships.

- [ ] **0.1.1 `S-01` TikTok API capability audit** (highest priority, blocks scope)
  - [ ] Enumerate available endpoints: Login Kit scopes, Display API, Content Posting API.
  - [ ] Answer explicitly: can a third party app **delete** a video? **change privacy** of an existing video?
        **edit** a caption after posting?
  - [ ] Record quotas, rate limits, sandbox restrictions and audit requirements.
  - [ ] Define the fallback UX for anything not supported (guided checklist, tracked as "manual action").
  - [ ] Write [docs/tiktok-integration.md](tiktok-integration.md) capability matrix with a link to each source.
  - **Commit:** `docs(spike): S-01 tiktok api capability audit`
- [ ] **0.1.2 `S-02` Automation canvas library** `[parallel]`
  - [ ] Compare Svelte Flow (`@xyflow/svelte`) against alternatives and against building it in house.
  - [ ] Build a throwaway prototype: 3 nodes, drag, connect, save graph to JSON, reload it.
  - [ ] Check licence, bundle size, a11y, touch support and maintenance activity.
  - **Commit:** `docs(spike): S-02 automation canvas library`, update [ADR-0006](adr/0006-automation-canvas-library.md)
- [ ] **0.1.3 `S-03` Backend framework and ORM** `[parallel]`
  - [ ] Validate NestJS + Prisma against the onion layering in [architecture.md](architecture.md).
  - [ ] Confirm OpenAPI generation fits [api-guidelines.md](api-guidelines.md) (snake case, problem+json).
  - **Commit:** `docs(spike): S-03 backend stack`, update [ADR-0003](adr/0003-backend-stack.md)
- [ ] **0.1.4 `S-04` Job queue and scheduling** `[parallel]`
  - [ ] BullMQ versus alternatives. Needs: delayed jobs, cron, retries with backoff, concurrency limits,
        rate limiting, progress events, a usable dashboard.
  - **Commit:** `docs(spike): S-04 job queue`, update [ADR-0005](adr/0005-job-queue.md)
- [ ] **0.1.5 Licence and repository policy**
  - [ ] Choose a licence and add `LICENSE`.
  - [ ] Add `CODE_OF_CONDUCT.md`, issue templates and a pull request template.
  - **Commit:** `chore: add licence and repository policy files`
- [ ] **0.1.6 Register the TikTok developer app**
  - [ ] Create the app, request scopes, note client key and secret handling, submit for audit early.
  - [ ] Document the sandbox setup in [tiktok-integration.md](tiktok-integration.md).
  - **Commit:** `docs: document tiktok developer app setup`

**Gate 0.1**
- All ADRs in [adr/](adr/) that were `Proposed` are now `Accepted` or `Superseded`, with reasoning.
- `S-01` capability matrix exists and every headline feature is marked supported, degraded or dropped.
- The plan below is updated if a spike invalidated it.

### 0.2 Repository and tooling scaffolding

- [ ] **0.2.1 Monorepo skeleton**
  - [ ] `apps/api`, `apps/web`, `packages/contracts`, `packages/config`, `docker/`, `e2e/`.
  - [ ] Package manager workspaces (pnpm), pinned Node version via `.nvmrc` and `engines`.
  - **Commit:** `chore(repo): scaffold monorepo workspaces`
- [ ] **0.2.2 Shared tooling** `[blocked-by 0.2.1]`
  - [ ] TypeScript base config in strict mode, path aliases per layer.
  - [ ] ESLint with an **architecture boundary rule** (`eslint-plugin-boundaries` or equivalent) that fails
        the build when `domain` imports from `infrastructure`.
  - [ ] Prettier, EditorConfig, `commitlint` for conventional commits, `lint-staged` + Husky pre-commit.
  - **Commit:** `chore(repo): add lint, format and commit tooling`
- [ ] **0.2.3 Environment and secrets**
  - [ ] `.env.example` with every variable documented and safe defaults.
  - [ ] Runtime env validation (Zod schema) that fails fast at boot with a readable message.
  - [ ] `.gitignore` covering `.env`, build output, coverage, Playwright artefacts.
  - **Commit:** `chore(config): add validated environment configuration`

**Gate 0.2**
- `pnpm install && pnpm lint && pnpm typecheck` passes from a clean clone.
- Deliberately importing `infrastructure` from `domain` fails lint. Prove it, then revert.
- Committing with a non conventional message is rejected by the hook. Prove it, then revert.

### 0.3 Running skeleton

- [ ] **0.3.1 API skeleton**
  - [ ] NestJS app with the four layer folder structure from [architecture.md](architecture.md).
  - [ ] `GET /health` (liveness) and `GET /ready` (checks database and Redis).
  - [ ] Structured JSON logging with a request id, and a global `problem+json` exception filter.
  - **Commit:** `feat(api): add application skeleton with health endpoints`
- [ ] **0.3.2 Web skeleton** `[parallel]`
  - [ ] SvelteKit app, Tailwind, shadcn-svelte initialised, base layout, light and dark theme.
  - [ ] App shell: sidebar, top bar, empty routes for `/dashboard`, `/automations`, `/jobs`, `/settings`.
  - **Commit:** `feat(web): add application shell and base theme`
- [ ] **0.3.3 Docker Compose stack** `[blocked-by 0.3.1, 0.3.2]`
  - [ ] Services: `web`, `api`, `worker`, `postgres`, `redis`, `minio`, `mailpit`.
  - [ ] Hot reload with bind mounts for `web` and `api`, named volumes for dependencies.
  - [ ] Healthchecks and `depends_on: condition: service_healthy` so boot order is deterministic.
  - [ ] `docker/` holds multi stage Dockerfiles with a `dev` and a `prod` target.
  - **Commit:** `feat(docker): add one command development stack`
- [ ] **0.3.4 Contracts package** `[blocked-by 0.3.1]`
  - [ ] `packages/contracts` holds the OpenAPI document as the source of truth.
  - [ ] Generate the typed client for the web app from the spec, wired into the build.
  - [ ] Spectral lint the spec against the Zalando ruleset in CI.
  - **Commit:** `feat(contracts): add openapi source of truth and client generation`
- [ ] **0.3.5 CI pipeline** `[blocked-by 0.2.2]`
  - [ ] GitHub Actions: install, lint, typecheck, unit tests, build, spec lint.
  - [ ] Cache the package store, run jobs in parallel, fail fast.
  - [ ] Branch protection on `main`: CI must pass before merge.
  - **Commit:** `ci: add build, lint and test pipeline`
- [ ] **0.3.6 Test harness bootstrap** `[blocked-by 0.3.3]`
  - [ ] Vitest configured for `apps/api` and `apps/web` with coverage thresholds.
  - [ ] Supertest integration harness booting the Nest app against Testcontainers Postgres and Redis.
  - [ ] Playwright configured against the Compose stack, with trace on first retry.
  - [ ] One example test at each level so the harness is proven, not theoretical.
  - **Commit:** `test: bootstrap unit, integration and e2e harnesses`

**Gate 0 (phase gate)**

| Check | Command or method |
| --- | --- |
| Unit | `pnpm test:unit` green, example tests run in both apps |
| Integration | `pnpm test:integration` green, containers start and tear down |
| Smoke | `curl -f localhost:3000/health` and `/ready` return 200 after `docker compose up` |
| Smoke | `curl -f localhost:5173` returns the app shell |
| E2E | `pnpm test:e2e` green: load the app shell, navigate all four routes, no console errors |
| Contract | Spectral lint of the OpenAPI spec passes with zero errors |
| Cold start | On a machine with only Docker and git, clone plus `docker compose up` works. Actually try it. |
| CI | Pipeline green on a pull request |

**Push:** `git push origin main` (or merge the phase branch). Tag `v0.0.1-skeleton`.

---

## Phase 1: the frontend, on a real API with seeded data

**Goal:** the complete UI, driven by a real backend serving seeded data. No hardcoded arrays in components.
**Exit:** every feature listed in the README is clickable end to end, and swapping the seeded source for the
TikTok source later means changing one adapter, not the UI.

> **Rule for this phase:** the web app only ever talks to the API. Test data lives in database seeds behind a
> `SeededTikTokAdapter` implementing the same port the real adapter will implement. When Phase 3 lands, the
> port stays, the adapter changes.

### 1.1 Domain and contract for the library

- [ ] **1.1.1 Model the video aggregate**
  - [ ] Entities, value objects and invariants per [data-model.md](data-model.md).
  - [ ] Pure domain functions: sorting comparators, filter predicates, selection maths.
  - [ ] Unit tests first, domain layer stays free of framework imports.
  - **Commit:** `feat(domain): add video aggregate and invariants`
- [ ] **1.1.2 Define the video API contract** `[blocked-by 1.1.1]`
  - [ ] `GET /v1/videos` with cursor pagination, `sort`, `order`, `filter[...]`, `q`.
  - [ ] `GET /v1/videos/{video_id}`, `PATCH /v1/videos/{video_id}` for local only fields.
  - [ ] Error model, field names and pagination follow [api-guidelines.md](api-guidelines.md).
  - **Commit:** `feat(contracts): define video resource endpoints`
- [ ] **1.1.3 Persistence and seed data** `[blocked-by 1.1.2]`
  - [ ] Prisma schema and first migration.
  - [ ] Repository implementation behind the domain port.
  - [ ] Seed script: about 250 videos with realistic spread of dates, metrics, durations, statuses and covers.
  - [ ] Deterministic seeding with a fixed random seed so tests and screenshots are stable.
  - **Commit:** `feat(api): add video persistence and deterministic seed data`
- [ ] **1.1.4 Query implementation** `[blocked-by 1.1.3]`
  - [ ] Sorting: date, views, likes, comments, duration, title, ascending and descending.
  - [ ] Filtering: status, category, tag, date range, metric range, free text.
  - [ ] Index the columns that sorting and filtering actually use.
  - **Commit:** `feat(api): implement video listing, sorting and filtering`

**Gate 1.1**
- Unit: comparators and predicates, including ties, nulls and unicode titles.
- Integration: each sort field ascending and descending returns correctly ordered pages, cursor pagination
  never skips or repeats an item across pages.
- Contract: generated client compiles against the spec, spec lint clean.

### 1.2 Dashboard UI

Spec: [features/dashboard.md](features/dashboard.md).

- [ ] **1.2.1 Video card component**
  - [ ] Cover with aspect ratio box, caption clamp, metrics row, status badge, selection checkbox.
  - [ ] Loading skeleton and error state. Keyboard focusable, correct roles.
  - **Commit:** `feat(web): add video card component`
- [ ] **1.2.2 Grid, pagination and empty states** `[blocked-by 1.2.1]`
  - [ ] Responsive grid, infinite scroll or "load more" against the cursor API.
  - [ ] Distinct empty states: no videos at all, no results for this filter, and failed to load.
  - **Commit:** `feat(web): add video grid with pagination and empty states`
- [ ] **1.2.3 Sort and filter controls** `[blocked-by 1.2.2]`
  - [ ] Sort dropdown plus direction toggle. Filter panel for status, category, tag, date and metric ranges.
  - [ ] State lives in the URL query string so views are shareable and survive reload.
  - [ ] Debounced search input.
  - **Commit:** `feat(web): add sorting and filtering controls`
- [ ] **1.2.4 Multi select** `[blocked-by 1.2.2]`
  - [ ] Click to toggle, shift click for a range, `ctrl/cmd+a` for the page, "select all N matching filter".
  - [ ] Sticky selection bar showing the count and available actions, with a clear "clear selection".
  - [ ] Selection survives pagination and is representable as a filter, not just an id list.
  - **Commit:** `feat(web): add multi select with selection bar`

**Gate 1.2**
- Unit: component tests for card states, selection reducer including shift range across page boundaries.
- E2E (Playwright): sort by views descending and assert the visual order matches the API order.
- E2E: filter narrows results, URL updates, reload restores the same view.
- E2E: select 5 videos, paginate, come back, selection is intact.
- A11y: automated axe scan on the dashboard has zero critical violations. Full keyboard pass, no mouse.
- Visual: Playwright screenshot baselines for grid, empty and selected states.

### 1.3 Tags and categories

Spec: [features/tagging.md](features/tagging.md).

- [ ] **1.3.1 Taxonomy domain and contract**
  - [ ] Category and subcategory tree with depth limits, plus flat tags. Slug rules and uniqueness per user.
  - [ ] `GET/POST/PATCH/DELETE /v1/categories` and `/v1/tags`.
  - **Commit:** `feat(domain): add taxonomy model and endpoints`
- [ ] **1.3.2 Assignment API** `[blocked-by 1.3.1]`
  - [ ] Assign and unassign for a single video and for a selection, idempotent.
  - **Commit:** `feat(api): add tag and category assignment`
- [ ] **1.3.3 Taxonomy UI** `[blocked-by 1.3.2]`
  - [ ] Sidebar tree with counts, tag chips with colour, create and rename inline.
  - [ ] Drag videos onto a category, and assign from the selection bar.
  - **Commit:** `feat(web): add taxonomy sidebar and assignment ui`

**Gate 1.3**
- Unit: slug generation, cycle prevention in the tree, depth limit enforcement.
- Integration: assigning the same tag twice is a no-op, deleting a category reassigns or orphans per spec.
- E2E: create a category, drag two videos into it, filter by it, see exactly those two.

### 1.4 Batch actions, UI and job shell

Spec: [features/batch-actions.md](features/batch-actions.md).

- [ ] **1.4.1 Batch job domain**
  - [ ] `BatchJob` aggregate with `items`, state machine `queued -> running -> succeeded | failed | partial`.
  - [ ] Per item result with an error code, so partial failure is a normal outcome and not an exception.
  - **Commit:** `feat(domain): add batch job aggregate and state machine`
- [ ] **1.4.2 Batch API** `[blocked-by 1.4.1]`
  - [ ] `POST /v1/batch-jobs` accepting an action, a selection and parameters. Returns `202` with a job id.
  - [ ] `GET /v1/batch-jobs/{id}` and `GET /v1/batch-jobs`.
  - [ ] **Idempotency key** support so a double click cannot double execute.
  - [ ] A **dry run** mode returning the impact preview without doing anything.
  - **Commit:** `feat(api): add batch job endpoints with idempotency and dry run`
- [ ] **1.4.3 Queue wiring** `[blocked-by 1.4.2]`
  - [ ] Worker process, one queue per action type, concurrency and rate limits from config.
  - [ ] Progress events, retry with exponential backoff, dead letter handling.
  - [ ] Against seeded data the handlers are no-ops that only update local state.
  - **Commit:** `feat(worker): add batch job processing with retries`
- [ ] **1.4.4 Job centre UI** `[blocked-by 1.4.3]`
  - [ ] Confirmation dialog with the dry run impact summary, explicit typed confirmation for destructive actions.
  - [ ] Live progress (polling in this phase, streaming later), per item results, retry failed items only.
  - **Commit:** `feat(web): add job centre with progress and retry`

**Gate 1.4**
- Unit: state machine transitions, illegal transitions rejected, partial result aggregation.
- Integration: the same idempotency key posted twice creates one job. Prove it.
- Integration: a job with 100 items where 3 fail ends `partial`, and retry re-runs only those 3.
- E2E: select 10 videos, run a bulk tag, watch progress reach 100 percent, verify the result in the grid.
- Smoke: the worker survives a restart mid job and the job completes. Kill the container and see.

### 1.5 Automation workspace, builder only

Spec: [features/automation-engine.md](features/automation-engine.md). Execution comes in Phase 5.

- [ ] **1.5.1 Flow schema and contract**
  - [ ] Versioned JSON graph: nodes, edges, node configuration, validation rules.
  - [ ] `GET/POST/PATCH/DELETE /v1/flows`, plus `POST /v1/flows/{id}/validate`.
  - **Commit:** `feat(contracts): add flow graph schema and endpoints`
- [ ] **1.5.2 Canvas** `[blocked-by 1.5.1]`
  - [ ] Node palette, drag onto canvas, connect, move, delete, undo and redo, zoom, pan, minimap.
  - [ ] Autosave with an explicit "saved" indicator, and optimistic concurrency on save.
  - **Commit:** `feat(web): add automation canvas`
- [ ] **1.5.3 Node configuration panel** `[blocked-by 1.5.2]`
  - [ ] Per node type schema driven forms with inline validation and plain language help.
  - [ ] Graph validation surfaced on the canvas: unreachable node, missing trigger, cycle, unset required field.
  - **Commit:** `feat(web): add node configuration and graph validation`

**Gate 1.5**
- Unit: graph validation rules, one test per rule, both the valid and invalid case.
- Integration: save a graph, reload it, get an identical graph. Round trip must be lossless.
- E2E: build a three node flow by dragging, reload the page, the flow is still there and still valid.
- E2E: attempt an invalid connection, get a clear inline error, no crash, no silent failure.

**Gate 1 (phase gate)**

| Check | Threshold |
| --- | --- |
| Unit coverage | Domain layer at least 90 percent lines, overall at least 70 percent |
| Integration | All endpoints in the spec have at least one happy path and one failure path test |
| E2E | Full journey passes: browse, filter, sort, select, batch act, tag, build a flow |
| Contract | Zero Spectral errors, generated client matches handwritten usage |
| A11y | Zero critical axe violations on every route, keyboard only pass documented |
| Performance | `GET /v1/videos` p95 under 300 ms with 10,000 seeded rows |
| No hardcoded data | Grep the web app for fixture arrays, there must be none outside tests |
| Design review | Walkthrough against [features/dashboard.md](features/dashboard.md), sign off recorded |

**Push:** push the phase branch, open a pull request, merge after review. Tag `v0.1.0-ui`.

---

## Phase 2: authentication and accounts

**Goal:** real users, real isolation. Spec: [security.md](security.md).
**Exit:** two users cannot see each other's data, proven by a test, not by inspection.

### 2.1 Identity

- [ ] **2.1.1 User aggregate and registration**
  - [ ] Email and password with Argon2id, password policy, email normalisation.
  - [ ] Email verification through Mailpit locally.
  - **Commit:** `feat(domain): add user aggregate and registration`
- [ ] **2.1.2 Session handling** `[blocked-by 2.1.1]`
  - [ ] Short lived access token plus rotating refresh token in an httpOnly, secure, SameSite cookie.
  - [ ] Refresh token reuse detection revokes the whole family.
  - [ ] Logout, logout everywhere, session list in settings.
  - **Commit:** `feat(api): add session issuance, refresh and revocation`
- [ ] **2.1.3 Password reset and rate limiting** `[blocked-by 2.1.2]`
  - [ ] Single use, expiring reset tokens. Responses do not reveal whether an account exists.
  - [ ] Rate limit and lockout on login, reset and registration.
  - **Commit:** `feat(api): add password reset with rate limiting`
- [ ] **2.1.4 Auth UI** `[blocked-by 2.1.2]`
  - [ ] Register, login, verify, forgot and reset screens. Route guards. Redirect back after login.
  - **Commit:** `feat(web): add authentication screens and route guards`

### 2.2 Authorization and tenancy

- [ ] **2.2.1 Ownership enforcement** `[blocked-by 2.1.2]`
  - [ ] Every query is scoped by `user_id` at the repository layer, not sprinkled through controllers.
  - [ ] Unknown and unowned resources both return `404`, never `403`, to avoid leaking existence.
  - **Commit:** `feat(api): enforce per user data isolation`
- [ ] **2.2.2 Audit log** `[blocked-by 2.2.1]`
  - [ ] Append only record of actor, action, target, parameters, outcome and timestamp for every mutation.
  - **Commit:** `feat(api): add audit log for mutations`

### 2.3 TikTok account linking

- [ ] **2.3.1 OAuth flow** `[blocked-by 2.1.2, 0.1.6]`
  - [ ] Authorisation code with PKCE, `state` bound to the session, exact redirect URI matching.
  - [ ] Token storage encrypted at rest with a key from the environment, never logged, never returned by the API.
  - **Commit:** `feat(api): add tiktok oauth account linking`
- [ ] **2.3.2 Token lifecycle** `[blocked-by 2.3.1]`
  - [ ] Central refresh with a single flight lock so concurrent workers cannot double refresh.
  - [ ] Clear "reconnect required" state surfaced in the UI when refresh fails permanently.
  - **Commit:** `feat(api): add tiktok token refresh and reconnect state`
- [ ] **2.3.3 Connected accounts UI** `[blocked-by 2.3.2]`
  - [ ] Connect, disconnect, show scopes granted, show connection health and last sync.
  - **Commit:** `feat(web): add connected accounts settings`

**Gate 2 (phase gate)**

| Check | Detail |
| --- | --- |
| Unit | Password hashing, token generation, expiry maths, state machine for reconnect |
| Integration | User A requesting user B's video gets `404`. One test per resource type, no exceptions |
| Integration | Refresh rotation works, replaying an old refresh token revokes the family |
| Integration | Login rate limit trips at the configured threshold |
| E2E | Register, verify, log in, land on the dashboard, log out, protected route redirects |
| E2E | OAuth linking against a mocked TikTok authorisation server, including the denial path |
| Security | Dependency audit clean, secrets never in logs, cookie flags asserted in a test |
| Security | Manual check against the [security.md](security.md) checklist, recorded |
| Smoke | Fresh database plus `docker compose up`, register and log in works with no manual steps |

**Push:** tag `v0.2.0-auth`.

---

## Phase 3: real TikTok reads

**Goal:** the dashboard shows real, synced videos.
**Exit:** a connected account's real videos appear and stay fresh, and the seeded path still works for tests.

- [ ] **3.1.1 TikTok client**
  - [ ] One HTTP client with retries, exponential backoff with jitter, a circuit breaker and a rate limiter.
  - [ ] Every response parsed through a schema so upstream shape changes fail loudly and early.
  - [ ] Structured logging of every call with the correlation id, never with tokens.
  - **Commit:** `feat(infra): add rate limited tiktok api client`
- [ ] **3.1.2 Real adapter behind the existing port** `[blocked-by 3.1.1]`
  - [ ] Implement the same port as the seeded adapter. Choose the adapter by configuration.
  - [ ] Map upstream fields to the domain model in one place, with the mapping unit tested.
  - **Commit:** `feat(infra): add live tiktok video adapter`
- [ ] **3.1.3 Sync engine** `[blocked-by 3.1.2]`
  - [ ] Full sync on first connect, incremental sync on a schedule, manual "sync now".
  - [ ] Reconcile: new, updated, and remotely deleted videos. Never destroy local only fields.
  - [ ] Track `last_synced_at` per account and per video, and expose staleness in the API.
  - **Commit:** `feat(api): add video sync engine with reconciliation`
- [ ] **3.1.4 Metrics history** `[blocked-by 3.1.3]`
  - [ ] Snapshot metrics over time so automations can act on trends, with a retention policy.
  - **Commit:** `feat(api): add metrics history snapshots`
- [ ] **3.1.5 Sync UI** `[blocked-by 3.1.3]`
  - [ ] "Last synced" indicator, manual sync button, sync failure banner with a real remedy.
  - **Commit:** `feat(web): surface sync state and manual sync`

**Gate 3 (phase gate)**

| Check | Detail |
| --- | --- |
| Unit | Field mapping, including missing optional fields and unexpected enum values |
| Unit | Backoff and circuit breaker behaviour with a fake clock |
| Integration | Sync against a recorded fixture server (MSW or WireMock): create, update and remote delete cases |
| Integration | Rate limit response triggers backoff and the job resumes, it does not fail the whole sync |
| Integration | Local only fields (tags, notes) survive a full resync. Non negotiable |
| Smoke | Against the real sandbox, connect an account and complete a full sync manually. Record the result |
| E2E | Dashboard renders synced videos, "sync now" updates the timestamp |
| Performance | Sync of 1,000 videos stays inside the rate limit and completes within the documented window |

**Push:** tag `v0.3.0-sync`.

---

## Phase 4: real TikTok writes

**Goal:** upload, publish and whatever lifecycle actions `S-01` confirmed as available.
**Exit:** a batch upload of five videos succeeds against the sandbox, unattended.

- [ ] **4.1.1 Asset upload pipeline**
  - [ ] Resumable, chunked upload from the browser to object storage with a checksum.
  - [ ] Server side validation of container, codec, duration, resolution and size against TikTok's limits,
        with clear rejection messages before anything is sent upstream.
  - [ ] Local thumbnail generation for the pre-publish preview.
  - **Commit:** `feat(api): add resumable asset upload with validation`
- [ ] **4.1.2 Publish action** `[blocked-by 4.1.1]`
  - [ ] Init, upload and publish against the Content Posting API, with status polling until terminal.
  - [ ] Map every documented upstream error to a domain error with a user readable remedy.
  - **Commit:** `feat(api): add publish action`
- [ ] **4.1.3 Batch upload** `[blocked-by 4.1.2]`
  - [ ] Per file caption, privacy, allowed interactions and scheduled time, plus "apply to all" defaults.
  - [ ] Sequential processing inside the rate limit, resumable after a restart.
  - **Commit:** `feat(api): add batch upload job`
- [ ] **4.1.4 Lifecycle actions** `[blocked-by 0.1.1]`
  - [ ] Implement delete and privacy change **if supported**.
  - [ ] If not supported, ship the fallback: a tracked manual action queue with clear UI copy explaining why,
        deep links into the app, and a "mark as done" that keeps local state truthful.
  - **Commit:** `feat(api): add lifecycle actions or tracked manual fallback`
- [ ] **4.1.5 Upload UI** `[blocked-by 4.1.3]`
  - [ ] Drag and drop many files, per file progress, per file settings, retry one, cancel all.
  - **Commit:** `feat(web): add batch upload experience`

**Gate 4 (phase gate)**

| Check | Detail |
| --- | --- |
| Unit | File validation rules, one test per rule at the boundary value |
| Unit | Upstream error code to domain error mapping, exhaustive over the documented list |
| Integration | Full publish flow against a fixture server, including the "processing failed" terminal state |
| Integration | Interrupted upload resumes from the last chunk |
| Integration | Batch of 5 where 1 file is invalid: 4 publish, 1 fails with a clear reason, job is `partial` |
| Smoke | Real sandbox: publish 1 video manually end to end. Record the evidence |
| Smoke | Real sandbox: batch of 5, unattended, all succeed |
| E2E | Drag 3 files, set captions, submit, watch progress to completion |
| Safety | Destructive action requires typed confirmation, dry run preview matches the actual outcome |

**Push:** tag `v0.4.0-publish`.

---

## Phase 5: the automation engine

**Goal:** flows built in Phase 1 actually run.
**Exit:** a scheduled flow publishes a real video with nobody watching.

### 5.1 Runtime

- [ ] **5.1.1 Execution model**
  - [ ] Deterministic graph interpreter, node by node, with an execution context passed along the edges.
  - [ ] Every step persisted so a run is resumable and auditable after a crash.
  - [ ] Guardrails: max steps per run, max runs per hour, cycle detection, per run timeout.
  - **Commit:** `feat(domain): add flow execution model`
- [ ] **5.1.2 Trigger nodes** `[blocked-by 5.1.1]`
  - [ ] Schedule (cron with timezone), manual, on new video synced, on metric threshold crossed.
  - [ ] Exactly once semantics per trigger occurrence, with idempotency keys.
  - **Commit:** `feat(automation): add trigger nodes`
- [ ] **5.1.3 Condition and logic nodes** `[blocked-by 5.1.1]`
  - [ ] Filter, if/else branch, age comparison, metric comparison, tag or category membership.
  - [ ] Explicit, documented handling of stale metrics.
  - **Commit:** `feat(automation): add condition and logic nodes`
- [ ] **5.1.4 Action nodes** `[blocked-by 5.1.2, 4.1.4]`
  - [ ] Publish from queue, change privacy or the fallback, add or remove tag, notify, wait or delay.
  - [ ] Every action node reuses the Phase 4 batch job machinery. No parallel implementation.
  - **Commit:** `feat(automation): add action nodes`
- [ ] **5.1.5 Run observability** `[blocked-by 5.1.1]`
  - [ ] Run history, per step input, output and duration, error surface, replay a run, dry run a flow.
  - **Commit:** `feat(automation): add run history and dry run`

### 5.2 Builder polish

- [ ] **5.2.1 Templates** `[parallel]`
  - [ ] Three starter templates covering the scenarios in [overview.md](overview.md).
  - **Commit:** `feat(web): add flow templates`
- [ ] **5.2.2 Live run view** `[blocked-by 5.1.5]`
  - [ ] Highlight the active node on the canvas during a run, show the result per node, link to the job.
  - **Commit:** `feat(web): add live run visualisation`
- [ ] **5.2.3 Enable, disable and safety** `[blocked-by 5.1.2]`
  - [ ] Enable and disable a flow, a global kill switch, and a confirmation before a flow can act destructively.
  - **Commit:** `feat(automation): add flow enablement and kill switch`

**Gate 5 (phase gate)**

| Check | Detail |
| --- | --- |
| Unit | One test per node type: happy path, empty input, and error path |
| Unit | Cycle detection, step limit, timeout, all with a fake clock |
| Integration | Scheduled trigger fires once and only once, even with two workers running |
| Integration | A run interrupted mid flow resumes correctly after a worker restart |
| Integration | Dry run performs zero side effects. Assert on the audit log, not on vibes |
| E2E | Build, enable and manually trigger a flow, then verify the effect and the run trace |
| Smoke | Sandbox: a scheduled flow publishes a real video unattended. Record the evidence |
| Usability | 3 non technical testers build a working flow in under 10 minutes. Record friction points |
| Safety | Kill switch stops in-flight runs within the documented window |

**Push:** tag `v0.5.0-automations`.

---

## Phase 6: review, refactor and hardening

**Goal:** the honest pass. Find what we got wrong and fix it before more is built on top.

- [ ] **6.1.1 Architecture review**
  - [ ] Dependency graph audit, every boundary violation listed and either fixed or accepted in an ADR.
  - [ ] Delete dead code, unify duplicated logic, name things consistently with [glossary.md](glossary.md).
  - **Commit:** `refactor: address architecture review findings`
- [ ] **6.1.2 Observability**
  - [ ] OpenTelemetry traces across web, API and worker. Metrics for queue depth, job duration and error rate.
  - [ ] Dashboards and alerts for: sync failure rate, publish failure rate, queue backlog, token expiry.
  - **Commit:** `feat(ops): add tracing, metrics and alerts`
- [ ] **6.1.3 Performance**
  - [ ] Load test the dashboard query and the batch pipeline. Fix the top three findings.
  - [ ] Add the indexes the profiler asks for, not the ones we guessed at in Phase 1.
  - **Commit:** `perf: optimise dashboard queries and batch throughput`
- [ ] **6.1.4 Resilience**
  - [ ] Chaos pass: kill Redis, kill Postgres, make TikTok return `429` and `500`, pull the network.
  - [ ] Every failure mode must degrade visibly and recover automatically. Document what does not.
  - **Commit:** `fix: harden failure handling found in chaos testing`
- [ ] **6.1.5 Production readiness**
  - [ ] Production Compose profile, backup and restore procedure, tested restore, migration rollback plan.
  - [ ] Runbook: common incidents, how to diagnose, how to recover.
  - **Commit:** `docs(ops): add production runbook and backup procedure`
- [ ] **6.1.6 Documentation truth pass**
  - [ ] Re-read every file in `docs/` against the code. Fix everything that drifted.
  - [ ] Record what we would do differently, in an ADR or a retrospective note.
  - **Commit:** `docs: reconcile documentation with implementation`

**Gate 6 (phase gate)**

| Check | Threshold |
| --- | --- |
| Coverage | Domain at least 90 percent, application at least 80 percent, overall at least 75 percent |
| Load | Dashboard p95 under 300 ms at 50 concurrent users with 10,000 videos |
| Load | Batch of 100 completes within the documented window without breaching rate limits |
| Chaos | Every dependency killed one at a time, the system recovers with no manual intervention |
| Security | Dependency audit clean, OWASP Top 10 review recorded, secrets rotation documented |
| E2E | The whole suite green three consecutive runs. Flaky tests are fixed or deleted, never retried away |
| Docs | Every link resolves, every command in the docs actually runs |
| Restore | Restore from backup into a clean environment and the app works. Actually do it |

**Push:** tag `v1.0.0-rc.1`.

---

## Backlog, not scheduled

Ideas worth keeping, deliberately not in the plan above.

- Multi account switching and per client workspaces.
- Team accounts with roles and per role permissions.
- Content calendar view, a timeline instead of a grid.
- A/B testing captions and covers.
- Webhooks out, so other systems can react to our events.
- A second platform behind the existing port, for example YouTube Shorts.
- Mobile responsive polish beyond "usable", meaning a real tablet layout.
- Import and export of the taxonomy, and of flows, as portable JSON.
