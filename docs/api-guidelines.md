# API guidelines

Our house rules, derived from the
[Zalando RESTful API Guidelines](https://opensource.zalando.com/restful-api-guidelines/). Where Zalando gives
a rule that only makes sense for a large multi team estate, we say so and skip it.

The OpenAPI document in `packages/contracts` is the **source of truth**. Write the spec first, generate the
client from it, and let CI reject drift.

## Naming

| Thing | Convention | Example |
| --- | --- | --- |
| Path segments | kebab-case, plural nouns | `/v1/batch-jobs` |
| Path parameters | snake_case | `/v1/videos/{video_id}` |
| Query parameters | snake_case | `?sort_by=view_count&page_size=50` |
| JSON properties | snake_case | `"created_at"`, `"view_count"` |
| Enum values | UPPER_SNAKE_CASE | `"PUBLIC"`, `"PARTIALLY_FAILED"` |
| Header names | Hyphenated-Pascal | `Idempotency-Key` |

Resources are nouns. Verbs in paths are a last resort and only for genuine actions that are not CRUD, expressed
as a sub resource: `POST /v1/flows/{flow_id}/runs`, not `POST /v1/run-flow`.

## Versioning

- URL versioning: `/v1/...`. Simple, visible, easy to route.
- Additive changes (new optional field, new endpoint, new enum value the client can ignore) do not bump the version.
- Breaking changes require `/v2` and an overlap period. Deprecation is announced with the `Deprecation` and
  `Sunset` headers.
- Clients must tolerate unknown fields. Never break on an addition.

## Methods and status codes

| Method | Use | Success |
| --- | --- | --- |
| `GET` | Read, always safe and idempotent | `200`, `404` |
| `POST` | Create, or start an asynchronous action | `201` with `Location`, or `202` with a job id |
| `PATCH` | Partial update, `application/merge-patch+json` | `200` or `204` |
| `PUT` | Full replace, only where replace is the natural semantic | `200` or `204` |
| `DELETE` | Remove, idempotent | `204`, and `404` only on the first call |

| Code | When |
| --- | --- |
| `400` | Malformed syntax |
| `401` | Missing or invalid credentials |
| `403` | Authenticated but not permitted, for a resource whose existence is not a secret |
| `404` | Not found, **or** found but not owned by the caller. We do not leak existence |
| `409` | Conflict, for example a duplicate slug or an optimistic concurrency failure |
| `422` | Syntactically valid but semantically wrong, for example a caption over the platform limit |
| `429` | Rate limited, always with `Retry-After` |
| `503` | Upstream unavailable, for example TikTok is down. Always with `Retry-After` |

## Errors

Always `application/problem+json` per RFC 9457. Never a bare string, never a `200` with `{"error": ...}`.

```json
{
  "type": "https://api.example.com/problems/video-caption-too-long",
  "title": "Caption exceeds the maximum length",
  "status": 422,
  "detail": "The caption is 2,340 characters. TikTok allows 2,200.",
  "instance": "/v1/videos/01HQ8X",
  "trace_id": "9f2c1b3e4d5a6b7c",
  "errors": [
    { "field": "caption", "code": "MAX_LENGTH", "message": "Maximum 2200 characters" }
  ]
}
```

Rules:

- `type` is a stable, documented URI. Clients branch on `type` and `errors[].code`, never on `title` or `detail`.
- `detail` is written for a human and may include specifics. It must never contain a token, a secret or another
  user's data.
- `trace_id` is always present so a user's screenshot is enough to find the request in the logs.
- Validation failures list every problem at once. Do not make the user fix errors one at a time.

## Pagination

**Cursor based only.** Offset pagination skips and repeats items when data changes underneath, and our data
changes constantly through syncing.

Request:

```http
GET /v1/videos?page_size=50&page_token=eyJpZCI6...&sort_by=created_at&order=desc
```

Response:

```json
{
  "items": [ ... ],
  "next_page_token": "eyJpZCI6...",
  "prev_page_token": null,
  "page_size": 50
}
```

- `page_size` defaults to 25, maximum 100. A larger value is clamped, not rejected.
- The cursor is opaque, encodes the sort key plus a tiebreaker id, and is signed so it cannot be tampered with.
- Total counts are **not** returned by default. `?include_total=true` is available where the UI genuinely needs
  it, and it is documented as the slower path.

## Filtering, sorting and search

```http
GET /v1/videos
  ?sort_by=view_count&order=desc
  &filter[status]=PUBLISHED
  &filter[category_id]=01HQ8X
  &filter[tag]=tutorial&filter[tag]=svelte
  &filter[created_at][gte]=2025-01-01T00:00:00Z
  &filter[view_count][gte]=1000
  &q=cooking
```

- `sort_by` accepts only a documented allow list. An unknown value is `400`, never silently ignored.
- Repeating a filter key means OR within that key. Different keys are ANDed.
- Range filters use `gte`, `gt`, `lte`, `lt`.
- `q` is free text over caption and title. It is a convenience, not a search engine.

## Asynchronous operations

Anything that can exceed roughly one second is a job.

```http
POST /v1/batch-jobs
Idempotency-Key: 01HQ8XR3M9

{
  "action": "SET_PRIVACY",
  "selection": { "type": "IDS", "video_ids": ["01HQ...", "01HR..."] },
  "parameters": { "privacy": "PRIVATE" },
  "dry_run": false
}
```

```http
HTTP/1.1 202 Accepted
Location: /v1/batch-jobs/01HQ8XS7
```

- **`Idempotency-Key` is mandatory** on every job creating and every destructive endpoint. The same key with
  the same body returns the original job. The same key with a different body is `409`.
- Keys are retained for 24 hours.
- `dry_run: true` returns `200` with the impact preview and performs no side effects at all.

## Concurrency

- Mutable resources return an `ETag`. `PATCH` and `PUT` should send `If-Match`.
- A mismatch is `412 Precondition Failed`, with enough detail to show a useful merge or reload prompt.
- Flows always require `If-Match`, because two tabs editing one canvas is a realistic accident.

## Consistency and shared conventions

- Timestamps are RFC 3339, UTC, with the `_at` suffix: `created_at`, `updated_at`, `published_at`, `last_synced_at`.
- Durations are integer **seconds** with a `_seconds` suffix. No "1m30s" strings.
- Identifiers are ULIDs as strings. They are opaque, so never parse them client side.
- Money does not exist in this system. If it ever does, it is minor units plus a currency code, never a float.
- Booleans read positively: `is_enabled`, not `is_not_disabled`.
- Nullable means "not known or not applicable". Absent means the same thing. Do not give them different meanings.

## Documentation requirements

Every operation in the spec has:

- A `summary` and a `description` that explain **why** you would call it, not just what it does.
- At least one realistic request and response example, using the seeded data so examples stay runnable.
- Every error `type` it can return.
- The required scope or permission.
- Rate limit notes where they differ from the default.

CI runs Spectral with the Zalando ruleset. Zero errors is the merge bar.

## What we deliberately skip from Zalando

| Rule | Why we skip it |
| --- | --- |
| `x-api-id`, `x-audience` and the full API registry metadata | Single team, one API, no internal registry to feed |
| Event type registry and required event streams | We use in-process domain events. Revisit if webhooks ship |
| Hypermedia and HAL style link relations | Our only client is generated from the spec. Links would be ceremony |
| Multi tenant `X-Tenant-ID` conventions | Tenancy is the authenticated user, resolved from the session |
