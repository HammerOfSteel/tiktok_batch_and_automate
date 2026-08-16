# ADR-0007: A seeded adapter behind a port, not a mock server

- **Status:** Accepted
- **Date:** 2026-08-16
- **Related:** [architecture.md](../architecture.md), Phase 1 in [TODO.md](../TODO.md)

## Context

Phase 1 builds the entire frontend before any TikTok integration exists. The stated requirement is explicit: no
hardcoded data in the frontend, use a real backend with test data, so that switching to real API calls later is
straightforward.

The question is where the test data lives.

## Options considered

**A. A `SeededTikTokAdapter` implementing the same port as the future live adapter, backed by real database
rows.**
Pros: the frontend, the API, the domain, the database and the queue are all exercised for real. The Phase 3
change is one new class plus a configuration value. Test data doubles as demo data, seed data and E2E fixtures.
Failure modes (rate limits, latency, partial failure) can be simulated deliberately.
Cons: seeded rows must be kept plausible as the model evolves, and the seeded adapter is code that ships but is
never used in production. That is a real, if small, maintenance cost.

**B. A standalone mock API server (Prism, WireMock, json-server) that the frontend calls directly.**
Pros: quick to start, needs no backend implementation.
Cons: the real backend is never exercised, so every domain rule, query and error path is deferred to a later
phase and then discovered all at once. It creates a second contract implementation that will drift. The Phase 3
switch becomes a genuine migration rather than a configuration change.

**C. MSW intercepting requests in the browser.**
Pros: excellent for frontend unit tests, no server needed.
Cons: the same fundamental problem as B, and it directly violates the "no hardcoded frontend data" rule, since
the fixtures live in the frontend.

**D. Point at the real TikTok API from day one.**
Pros: maximum realism.
Cons: blocked on app audit, unusable in CI, rate limited, non deterministic, and it makes early UI work depend
on the slowest external process in the plan.

## Decision

Option A. One port, two adapters, selected by `TIKTOK_ADAPTER=seeded|live`.

```
application/ports/tiktok-video.port.ts     <- the contract
infrastructure/tiktok/seeded.adapter.ts    <- phase 1, tests, CI, demo
infrastructure/tiktok/live.adapter.ts      <- phase 3 onwards
```

The seeded adapter is not a stub. It simulates realistic latency, paginates properly, occasionally fails, and
honours a simulated rate limit, so the UI is built against reality rather than against an unrealistically
perfect backend.

MSW is still used, but for a different job: faking **upstream HTTP responses** in integration tests of the live
adapter.

## Consequences

**Easier:** the whole stack is real from Phase 1. Domain rules, queries, error handling and job processing are
proven before TikTok is involved. CI runs the full E2E suite with no external dependency and no credentials.
The product has a permanent demo mode. Phase 3 is a configuration switch plus one class.

**Harder:** the seeded adapter must be maintained alongside the model. Its simulated behaviour must stay honest,
because a seeded adapter that never fails teaches the UI to ignore failure.

**Accepted costs:** production code that exists for development and demonstration. Worth it, and it doubles as
the fixture source for every test level.

**Revisit if:** never, realistically. This pattern is the reason the phase plan works at all.
