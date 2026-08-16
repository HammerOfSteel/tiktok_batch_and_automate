# tiktok_batch_and_automate

An API first platform for **batch managing TikTok content** and **building publishing automations without code**.

Manage every video you own from one dashboard, select many at once, act on them in bulk, and wire up
"when X happens, do Y" flows on a visual canvas that feels like n8n but is purpose built for TikTok.

> **Status: pre-alpha.** Nothing is implemented yet. This repository currently holds the plan, the
> architecture and the specs. Start with [docs/overview.md](docs/overview.md) and
> [docs/TODO.md](docs/TODO.md).

---

## Table of contents

- [What it does](#what-it-does)
- [Design principles](#design-principles)
- [Proposed stack](#proposed-stack)
- [Repository layout](#repository-layout)
- [Quick start](#quick-start)
- [Documentation map](#documentation-map)
- [Roadmap at a glance](#roadmap-at-a-glance)
- [Known constraints](#known-constraints)
- [Contributing](#contributing)
- [License](#license)

---

## What it does

| Capability | Description |
| --- | --- |
| **Video dashboard** | Every video rendered as a card, with cover, caption, metrics and status. |
| **Sorting and filtering** | Sort by date, views, likes, comments, duration or title, ascending or descending. Filter by status, tag or category. |
| **Multi select** | Select many videos with click, shift click and "select all matching filter", then act on the selection. |
| **Batch actions** | Bulk upload, bulk delete, bulk privacy change, bulk tagging and bulk metadata edit, executed as trackable jobs. |
| **Tags and categories** | A hierarchical taxonomy (category, subcategory) plus free form tags, applied individually or in bulk. |
| **Automation workspace** | A node based canvas for building publish and cleanup flows. Triggers, conditions and actions, no code required. |
| **Job centre** | Every batch and automation run is a first class job with progress, per item result, retry and audit trail. |

Full feature specs live in [docs/features/](docs/features/).

## Design principles

1. **Modularity.** Every capability is a bounded module with an explicit contract. Modules talk through
   interfaces, never through each other's internals.
2. **API first.** The OpenAPI contract is written before the code. The frontend is just the first consumer,
   never a privileged one.
3. **Work smarter, not harder.** Before building anything, research what already exists. If a mature library
   or service solves the problem and fits the architecture, adopt it and record why in an
   [ADR](docs/adr/).
4. **Docker Compose native development.** The whole stack, frontend included, runs with a single
   `docker compose up`. No "install these seven things first" onboarding.
5. **Onion architecture.** Domain at the centre, then application, then infrastructure, then the delivery
   layer. Dependencies only ever point inwards.
6. **Domain Driven Design, lightly applied.** Ubiquitous language, aggregates and domain events where they
   earn their keep. No ceremony for its own sake.
7. **SOLID and functional patterns.** Pure functions for domain logic, immutable value objects, side effects
   pushed to the edges.
8. **Zalando RESTful API Guidelines.** Kebab case paths, snake case JSON, `application/problem+json` errors,
   cursor pagination, explicit versioning.

Details and the reasoning behind each: [docs/architecture.md](docs/architecture.md).

## Proposed stack

Every choice below is **proposed, not locked**. Each one is recorded as an ADR and gets confirmed or replaced
during the Phase 0 research spikes.

| Layer | Proposal | ADR |
| --- | --- | --- |
| Frontend | SvelteKit 2 + Svelte 5 + TypeScript + Tailwind + [shadcn-svelte](https://github.com/huntabyte/shadcn-svelte) | [ADR-0002](docs/adr/0002-frontend-stack.md) |
| Expressive UI layer | [Aceternity UI Svelte](https://aceternity.sveltekit.io/components), allowlisted and budgeted | [ADR-0008](docs/adr/0008-visual-effects-layer.md) |
| Automation canvas | [Svelte Flow](https://svelteflow.dev/) (`@xyflow/svelte`) | [ADR-0006](docs/adr/0006-automation-canvas-library.md) |
| Backend API | NestJS + TypeScript, layered per onion architecture | [ADR-0003](docs/adr/0003-backend-stack.md) |
| Database | PostgreSQL + Prisma | [ADR-0003](docs/adr/0003-backend-stack.md) |
| Jobs and scheduling | BullMQ on Redis | [ADR-0005](docs/adr/0005-job-queue.md) |
| Object storage | MinIO locally, S3 compatible in production | [ADR-0003](docs/adr/0003-backend-stack.md) |
| Auth | Email and password (Argon2id) plus TikTok OAuth for account linking | [ADR-0004](docs/adr/0004-authentication.md) |
| Tests | Vitest, Supertest, Testcontainers, Playwright | [docs/testing-strategy.md](docs/testing-strategy.md) |
| Local dev | Docker Compose, one command | [ADR-0001](docs/adr/0001-docker-compose-development.md) |

## Repository layout

The target layout once Phase 0 scaffolding lands:

```
.
├── apps/
│   ├── api/                 # NestJS backend (domain, application, infrastructure, http)
│   └── web/                 # SvelteKit frontend
├── packages/
│   ├── contracts/           # OpenAPI spec + generated TS client, shared by api and web
│   └── config/              # Shared eslint, tsconfig, prettier
├── docs/                    # You are here
├── docker/                  # Dockerfiles and compose overrides
├── e2e/                     # Playwright suites
├── compose.yaml             # The one command dev stack
└── README.md
```

## Quick start

Not runnable yet. This is the intended workflow once Phase 0 is complete:

```bash
git clone https://github.com/HammerOfSteel/tiktok_batch_and_automate.git
cd tiktok_batch_and_automate
cp .env.example .env
docker compose up
```

| Service | URL |
| --- | --- |
| Web UI | http://localhost:5173 |
| API | http://localhost:3000 |
| API docs (Swagger UI) | http://localhost:3000/docs |
| MinIO console | http://localhost:9001 |
| Mailpit (dev mail) | http://localhost:8025 |

Full setup, troubleshooting and day to day commands: [docs/development.md](docs/development.md).

## Documentation map

| Document | Read it when you want to |
| --- | --- |
| [docs/overview.md](docs/overview.md) | Understand the product, the users and the scope. |
| [docs/TODO.md](docs/TODO.md) | See the phased plan, tasks, quality gates and commit points. |
| [docs/architecture.md](docs/architecture.md) | Understand the layering, module boundaries and dependency rules. |
| [docs/api-guidelines.md](docs/api-guidelines.md) | Design or review an endpoint. |
| [docs/data-model.md](docs/data-model.md) | Understand entities, aggregates and the schema. |
| [docs/testing-strategy.md](docs/testing-strategy.md) | Know what to test, at what level and with which tool. |
| [docs/development.md](docs/development.md) | Get the stack running or debug it. |
| [docs/design-system.md](docs/design-system.md) | Build UI, add an effect, or check a motion or performance budget. |
| [docs/security.md](docs/security.md) | Handle tokens, secrets, authorization or user data. |
| [docs/features/dashboard.md](docs/features/dashboard.md) | Build or change the video dashboard. |
| [docs/features/batch-actions.md](docs/features/batch-actions.md) | Build or change bulk operations. |
| [docs/features/automation-engine.md](docs/features/automation-engine.md) | Build or change the automation canvas or runtime. |
| [docs/features/tagging.md](docs/features/tagging.md) | Build or change tags and categories. |
| [docs/tiktok-integration.md](docs/tiktok-integration.md) | Touch anything that talks to TikTok. |
| [docs/adr/](docs/adr/) | Understand or change a technical decision. |
| [docs/glossary.md](docs/glossary.md) | Check what a term means in this codebase. |
| [CONTRIBUTING.md](CONTRIBUTING.md) | Commit, branch, review or open a PR. |

## Roadmap at a glance

| Phase | Goal | Exit condition |
| --- | --- | --- |
| **0. Foundations** | Research spikes, decisions, running skeleton | `docker compose up` serves web and API, CI is green |
| **1. Frontend on a real API with seeded data** | Full UI, no hardcoded data | Every listed feature is clickable end to end |
| **2. Auth and accounts** | Email and password auth plus TikTok account linking | A user logs in and sees only their own data |
| **3. Real TikTok reads** | Live video sync from TikTok | Dashboard shows real videos, refreshed on a schedule |
| **4. Real TikTok writes** | Upload, publish and privacy changes for real | Batch upload of five videos succeeds against the sandbox |
| **5. Automation engine** | Visual builder plus execution runtime | A scheduled flow publishes a real video unattended |
| **6. Hardening** | Refactor, observability, performance, docs | Load target met, error budget defined, runbook written |

The detailed version with tasks, subtasks, gates and commit points is in [docs/TODO.md](docs/TODO.md).

## How it talks to TikTok

The single most important design fact, resolved by spike `S-01` and decided in
[ADR-0010](docs/adr/0010-browser-session-execution.md). Full analysis in
[docs/tiktok-integration.md](docs/tiktok-integration.md).

**The official TikTok API cannot delete, unpublish, re-privatise or edit an existing video.** Only three video
scopes exist (`video.list`, `video.publish`, `video.upload`), and `video.list` returns *public* videos only.
That is a hard limit of the API.

It is not a limit on the product. Working tools (Redact, SocialEraser, DeleteTik) do bulk deletion today by
acting as the **signed in user in a real browser**, driving TikTok Studio's own interface. SocialEraser is
[MIT licensed](https://github.com/socialeraser/SocialEraser), so the approach is readable rather than guessed
at. Redact is a locally installed desktop app, which is the same shape as our self hosted Compose stack.

So there are three execution strategies behind one port:

| Strategy | Role | Can delete? | Reads private videos? | Needs app audit? |
| --- | --- | --- | --- | --- |
| `SEEDED` | Development, CI, every test, demo mode | n/a | n/a | No |
| `BROWSER` | **Primary.** Library sync and lifecycle actions | **Yes** | **Yes** | No |
| `API` | Optional, best for uploading and publishing | No | No | Yes |

Consequences worth knowing before reading the plan:

- **The browser strategy is fragile by nature.** A TikTok UI change breaks it. Selectors live in versioned
  config, a canary check runs before every job, and a scheduled drift alert catches breakage before a user
  does.
- **It is very likely contrary to TikTok's terms of service**, and carries account risk. It acts only on the
  signed in user's own account and own content, at human pace, entirely on their own machine, with no
  credential ever reaching a server we control. The product discloses this and requires explicit opt in.
- **Nothing waits on app audit any more.** That was the longest lead time item in the plan.
- **The manual checklist survives** as the degradation path when the browser strategy is unavailable.
- **Rate limits and pacing.** Roughly one action per second with jitter, so a 1,000 video operation is a
  background job of about fifteen minutes, not an instant one.

## Contributing

Read [CONTRIBUTING.md](CONTRIBUTING.md). Short version: small commits, conventional commit messages, tests
alongside the change, one commit per completed task, push at the end of each phase.

## License

Not yet chosen. Tracked as task `0.1.5` in [docs/TODO.md](docs/TODO.md).