# ADR-0005: BullMQ on Redis for jobs and scheduling

- **Status:** Proposed, pending spike `S-04`
- **Date:** 2026-08-16
- **Related:** [features/batch-actions.md](../features/batch-actions.md), [features/automation-engine.md](../features/automation-engine.md), task `0.1.4`

## Context

Almost everything valuable in this product is asynchronous: batch uploads, publishing, syncing, scheduled flows
and metric threshold triggers. Requirements:

- Delayed jobs and cron style repeatable jobs, with timezone awareness
- Retries with exponential backoff, and a distinction between retryable and permanent errors
- Per queue concurrency and rate limiting, since TikTok quotas are the binding constraint
- Progress reporting per item, for honest progress bars
- Resumable after a worker crash, verified by killing a container
- Exactly once semantics per trigger occurrence, through idempotency keys
- An inspection UI for development

## Options considered

**A. BullMQ on Redis.**
Pros: covers every requirement directly, including rate limiting and repeatable jobs. Mature, actively
maintained, TypeScript native, good NestJS integration, Bull Board gives a usable dashboard. Redis is already in
the stack for sessions, caching and single flight locks.
Cons: Redis persistence configuration matters, since a badly configured Redis can lose jobs. Exactly once needs
our own idempotency keys, the queue only gives at least once.

**B. PostgreSQL backed queue (pg-boss or Graphile Worker).**
Pros: one fewer datastore, transactional enqueue in the same transaction as the state change, which removes a
whole class of consistency bug. Durable by default.
Cons: heavier database load from polling, less mature rate limiting, and we want Redis anyway for locks and
caching, so the "one fewer service" benefit is partly illusory.

**C. Temporal.**
Pros: genuinely excellent for long running, resumable workflows, which the automation engine resembles.
Cons: a large operational dependency for a self hostable single deployable, and a steep learning curve. Massive
overkill for version 1.

**D. A hand rolled table plus polling.**
Pros: total control, no dependency.
Cons: we would reimplement backoff, scheduling, rate limiting, concurrency and observability, and we would do it
worse.

## Decision (proposed)

Option A. BullMQ on Redis, with:

- One queue per action type, so a slow upload queue cannot starve instant local operations.
- Redis configured with AOF persistence, since losing a publish job is a real harm.
- **Idempotency keys owned by our domain**, not delegated to the queue. At least once delivery is assumed and
  designed for.
- Job state mirrored into PostgreSQL (`batch_jobs`, `batch_job_items`), because the queue is transport and the
  database is the record. History must survive a Redis flush.

Spike `S-04` verifies repeatable jobs with timezones, rate limiting and crash resumption before this is
Accepted.

## Consequences

**Easier:** everything on the requirements list is available immediately. Redis is already present. The
development dashboard makes queue problems visible instead of mysterious.

**Harder:** two sources of truth to keep aligned (queue state and database state). Our rule: **the database is
authoritative**, the queue only drives execution. Redis persistence becomes an operational concern documented in
the runbook.

**Accepted costs:** at least once delivery, handled by idempotency keys everywhere.

**Revisit if:** `S-04` shows a gap in rate limiting or crash resumption, in which case pg-boss is the fallback,
or if the automation engine grows into genuinely long running workflows where Temporal would earn its cost.
