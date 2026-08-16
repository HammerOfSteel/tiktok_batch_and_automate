# Testing strategy

Tests are part of the change, not a follow up task. A task is not done until its tests pass, and a phase does
not end until its gate passes.

## The shape of the suite

```
        /\
       /  \        E2E (Playwright)          ~30 specs, minutes
      /----\
     /      \      Integration (Supertest    ~200 specs, ~2 min
    /        \      + Testcontainers)
   /----------\
  /            \   Unit (Vitest)             ~800 specs, seconds
 /______________\
```

Plus two cross cutting sets that do not fit the pyramid: **contract tests** against the OpenAPI spec, and
**smoke tests** against a running stack.

## Levels

### Unit

**Tool:** Vitest. **Scope:** one module, no IO, no framework.

Covers the domain layer above all. Sorting comparators, filter predicates, selection maths, state machines,
graph validation, value object construction, error mapping, backoff maths.

Rules:
- No database, no HTTP, no filesystem, no real clock. Inject them.
- One behaviour per test. The name states the behaviour, not the method name.
  Good: `rejects a caption over 2200 characters`. Bad: `test caption`.
- Test boundaries and nasty inputs: empty, one, many, null, duplicate, unicode, maximum, maximum plus one.
- No mocking of the thing under test. If a unit test needs five mocks, the design is wrong, not the test.

```ts
describe('sortByViewCount', () => {
  it('orders descending and breaks ties by id for a stable cursor', () => { /* ... */ });
  it('places videos with unknown metrics last regardless of direction', () => { /* ... */ });
});
```

### Integration

**Tool:** Vitest + Supertest + Testcontainers. **Scope:** the API from the HTTP edge down to a real Postgres
and a real Redis. Only external third parties are faked.

Covers: every endpoint, repository queries, transactions, migrations, queue processing, authorization scoping.

Rules:
- Real database, real migrations. An in-memory substitute proves nothing about the SQL we actually ship.
- Each test gets a clean schema or runs in a rolled back transaction. Tests must pass in any order.
- Every endpoint in the spec needs at least a happy path and a failure path before the phase gate.
- Authorization is tested per resource type: user A must not reach user B's data. No exceptions and no
  "it obviously works".
- TikTok is faked with MSW or WireMock using **recorded real responses**, including their error bodies.

### End to end

**Tool:** Playwright against the Docker Compose stack. **Scope:** the browser, for real journeys.

Keep this layer small and valuable. E2E is where flaky tests come from, so it earns its place by covering
journeys, not permutations.

The permanent set:

| Spec | Journey |
| --- | --- |
| `auth.spec.ts` | Register, verify, log in, log out, guarded redirect |
| `dashboard.spec.ts` | Load, sort, filter, search, paginate, URL state survives reload |
| `selection.spec.ts` | Select, shift range, select all matching, selection survives pagination |
| `batch.spec.ts` | Select, preview impact, confirm, watch progress, verify the result |
| `taxonomy.spec.ts` | Create a category, assign videos, filter by it |
| `upload.spec.ts` | Drag several files, set per file options, publish, see progress |
| `automation.spec.ts` | Build a flow by dragging, save, reload, validate, run, read the trace |
| `a11y.spec.ts` | axe scan on every route, zero critical violations |

Rules:
- Select elements by role and accessible name, or by `data-testid`. Never by CSS class or DOM position.
- No arbitrary waits. Wait for a condition, always.
- Deterministic seed data, and freeze time where dates are asserted.
- Trace, video and screenshot on first retry, uploaded as CI artefacts.
- **A flaky test is a bug.** Fix it or delete it. Never wrap it in a retry and move on.

### Contract

The OpenAPI document is the source of truth, so it gets tested:

- Spectral lint with the Zalando ruleset, zero errors, in CI.
- Response validation middleware in the test environment fails any response that does not match the spec.
- The generated client must compile against how the web app actually calls it. Drift breaks the build.
- Every documented example is executed against the seeded stack, so examples cannot go stale.

### Smoke

Fast checks that a deployed or freshly composed stack is genuinely alive. Run after `docker compose up`, in CI
after the stack starts, and after any deployment.

```bash
pnpm smoke
```

| Check | Passing means |
| --- | --- |
| `GET /health` returns 200 | Process is up |
| `GET /ready` returns 200 | Database and Redis are reachable |
| `GET /` on the web app returns HTML containing the app shell | Frontend built and served |
| `GET /docs` returns 200 | The spec is published |
| A worker heartbeat key exists in Redis | The worker is consuming |
| Seed data query returns the expected row count | Migrations and seeds ran |

Smoke tests are shallow by design. They answer "is it running", not "is it correct".

## Specialist checks

| Type | Tool | Where |
| --- | --- | --- |
| Accessibility | `@axe-core/playwright` | Every route, in the E2E suite. Zero critical violations |
| Visual regression | Playwright screenshots | Dashboard grid, card states, canvas. Reviewed on change, not auto approved |
| Load | k6 or autocannon | Phase 6 gate: dashboard query and the batch pipeline |
| Chaos | Manual scripted container kills | Phase 6 gate, findings become tasks |
| Dependency and secrets | `pnpm audit`, gitleaks | Every CI run |

## Coverage

Coverage is a smoke alarm, not a goal. A high number with weak assertions is worse than an honest low one.

| Layer | Line threshold | Reasoning |
| --- | --- | --- |
| Domain | 90 percent | Pure and cheap to test. No excuse |
| Application | 80 percent | Orchestration, main paths and error paths |
| Infrastructure | 60 percent | Covered mostly through integration tests |
| Web `lib/domain` | 90 percent | Pure client logic |
| Web components | No threshold | Behaviour is covered by E2E, not by snapshot noise |
| Overall gate | 75 percent by Phase 6 | |

## Test data

- **One seed script, used by everything**: local development, integration tests, E2E and screenshots.
- Deterministic, with a fixed random seed. The same data on every machine, every run.
- Realistic: a spread of dates over two years, a long tail of view counts, some videos with no metrics, some
  with unicode and emoji captions, some with very long captions, some failed uploads.
- Builders over fixtures for unit tests: `aVideo().withViews(5000).published(daysAgo(30)).build()`. Tests then
  state only what they care about, which keeps them readable when the model grows.

## Commands

```bash
pnpm test              # unit + integration
pnpm test:unit         # fast, watch friendly
pnpm test:integration  # containers, slower
pnpm test:e2e          # Playwright, needs the stack running
pnpm test:e2e:ui       # Playwright UI mode for debugging
pnpm smoke             # shallow liveness checks
pnpm test:coverage     # coverage report with thresholds enforced
```

## What we do not test

Saying this out loud prevents busywork.

- Framework behaviour. NestJS routing and Prisma query building are their maintainers' problem.
- Generated code. The OpenAPI client is validated by compilation, not by unit tests.
- Getters, DTO shapes and other code with no branching.
- Exact CSS values. Visual regression covers appearance, assertions cover behaviour.
- Third party availability. TikTok being up is not a test, it is monitoring.
