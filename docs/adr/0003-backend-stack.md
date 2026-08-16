# ADR-0003: NestJS, Prisma and PostgreSQL for the backend

- **Status:** Proposed, pending spike `S-03`
- **Date:** 2026-08-16
- **Related:** [architecture.md](../architecture.md), [api-guidelines.md](../api-guidelines.md), task `0.1.3`

## Context

The backend must support onion layering with enforced boundaries, DDD tactics, an OpenAPI first contract, a job
queue, scheduled work and a worker process that shares code with the API. The frontend is TypeScript, so a
TypeScript backend allows shared types and one toolchain.

## Options considered

**A. NestJS + Prisma + PostgreSQL.**
Pros: dependency injection makes port and adapter inversion natural, module system maps onto our bounded
modules, mature OpenAPI generation, first party queue and scheduling integration, Prisma gives type safe queries
and a good migration story, one language across the stack.
Cons: NestJS is opinionated and decorator heavy, which sits awkwardly beside functional purity in the domain
(mitigated by keeping the domain framework free). Prisma's abstraction can fight complex queries, and the
dashboard query is the most complex thing we have.

**B. Fastify or Express with a hand rolled structure.**
Pros: minimal, fast, no framework opinions to work around, full control of layering.
Cons: we build and maintain DI, validation, OpenAPI generation, module wiring and lifecycle ourselves. That is
weeks of undifferentiated work that NestJS already solved.

**C. .NET with ASP.NET Core and EF Core.**
Pros: the most natural fit for onion and DDD, excellent tooling, strong typing, superb performance.
Cons: two languages and two toolchains, no shared types with the frontend, a heavier Docker development loop,
and a much smaller overlap with the frontend skill set.

**D. Python with FastAPI and SQLAlchemy.**
Pros: excellent OpenAPI support, fast to write.
Cons: two languages, weaker compile time guarantees for a domain heavy codebase, and a less pleasant background
worker story than BullMQ.

## Decision (proposed)

Option A. NestJS with Prisma and PostgreSQL, with two guardrails:

1. **The domain layer imports nothing from NestJS or Prisma.** It is plain TypeScript, pure functions and
   classes. Enforced by a lint boundary rule, task `0.2.2`.
2. **Escape hatch for queries.** Where Prisma cannot express the dashboard query efficiently, use raw
   parameterised SQL inside the repository. The port stays the same, so the escape hatch never leaks upwards.

Spike `S-03` validates both guardrails on a real vertical slice before this moves to Accepted.

## Consequences

**Easier:** one language, shared types with the frontend, DI makes the port and adapter pattern natural, and
`api` and `worker` genuinely share an image and a codebase.

**Harder:** keeping decorators out of the domain requires discipline and lint enforcement. Prisma migrations
need review for lock behaviour before they touch a large table.

**Accepted costs:** some NestJS ceremony in the delivery and infrastructure layers.

**Revisit if:** `S-03` shows the OpenAPI output cannot satisfy [api-guidelines.md](../api-guidelines.md)
(snake case, `problem+json`, cursor pagination) without excessive fighting, or if the boundary rule proves
unenforceable in practice.
