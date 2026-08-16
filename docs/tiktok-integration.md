# TikTok integration

Everything we know, everything we assume, and everything that must be verified before code depends on it.

> **This document is a stub until spike `S-01` completes.** The capability matrix below contains **assumptions
> marked as unverified**. Do not build on an unverified row. Task `0.1.1` in [TODO.md](TODO.md) replaces every
> `UNVERIFIED` with a link to the official documentation and a dated note.

## Why this document exists first

The product promises "batch delete" and "batch privacy change". Those promises are only deliverable if TikTok
exposes third party endpoints for them. Publicly documented TikTok APIs are strong on **creating** posts and
**reading** post data, and it is entirely possible that **modifying or deleting an existing post is not
available to third party apps at all**.

If that is the case, we find out in Phase 0 and adjust the product honestly, rather than discovering it in
Phase 4 with a UI already built around it.

## Surfaces

| API | Purpose | Used for |
| --- | --- | --- |
| **Login Kit / OAuth** | User authorisation, access and refresh tokens | Account linking |
| **Display API** | Read user info and the user's own videos | Dashboard sync, metrics |
| **Content Posting API** | Create posts, direct post or upload to inbox | Upload and publish |
| **Webhooks** | Server side event notifications, if available for our scopes | Sync freshness, `UNVERIFIED` |

## Capability matrix

Fill this in as the first deliverable of `S-01`. Every row needs a verdict, a source link and a date.

| Capability | Needed by | Status | Source | Fallback if unsupported |
| --- | --- | --- | --- | --- |
| OAuth authorisation and refresh | Everything | `UNVERIFIED` | | None. Blocking |
| Read profile (name, avatar) | Account UI | `UNVERIFIED` | | Show a placeholder |
| List the user's own videos with pagination | Dashboard | `UNVERIFIED` | | None. Blocking for Phase 3 |
| Read per video metrics | Dashboard, automation conditions | `UNVERIFIED` | | Hide metric sorting and metric conditions |
| Read cover image and share URL | Dashboard cards | `UNVERIFIED` | | Generic placeholder cover |
| Upload a video file | Batch upload | `UNVERIFIED` | | None. Blocking for Phase 4 |
| Direct post (publish immediately) | Publish | `UNVERIFIED` | | Upload to inbox and require a manual finish |
| Schedule a post for a future time | Automation | `UNVERIFIED` | | Our own scheduler publishes at the right moment |
| Set privacy at post time | Publish | `UNVERIFIED` | | Post with the default and record a manual action |
| **Change privacy of an existing video** | Batch privacy | `UNVERIFIED` | | Tracked manual action list |
| **Delete an existing video** | Batch delete | `UNVERIFIED` | | Tracked manual action list |
| **Edit caption of an existing video** | Batch edit | `UNVERIFIED` | | Tracked manual action list |
| Webhooks for post status | Sync freshness | `UNVERIFIED` | | Polling on a schedule |

The API exposes this matrix per connected account at `GET /v1/accounts/{id}/capabilities`, and the UI renders
from it. An action that is not supported is never offered, greyed out with an explanation instead of failing
after the click.

## The manual fallback, stated plainly

Where a capability is unsupported, we do **not** automate the TikTok web interface, drive a headless browser or
use private endpoints. That risks the user's account, breaks constantly, and violates platform terms.

Instead: a tracked manual action list. See the fallback section in
[features/batch-actions.md](features/batch-actions.md). The user still gets the organisation, the list, the
deep links and the audit trail. They just perform the final tap themselves.

## App registration and audit

Record in `S-01` and task `0.1.6`:

- App creation steps and where the client key and secret live.
- Exactly which scopes we request, and the justification for each. Request the minimum.
- Sandbox constraints: which test users can be used, whether posts are forced to private, and what changes
  after audit.
- The audit process, what evidence it requires (a demo video, a privacy policy, a terms page), and the expected
  lead time.
- Any usage restrictions relevant to a self hosted application, since each deployment may need its own
  registered app.

**Apply for audit as early as possible.** It is the longest lead time item in the entire plan and it blocks
nothing until suddenly it blocks everything.

## Client design

One client, in the infrastructure layer, behind the port defined in [architecture.md](architecture.md).

| Concern | Approach |
| --- | --- |
| Rate limiting | Token bucket per account and per app, sized from the documented quotas, shared across workers through Redis |
| Retries | Exponential backoff with jitter on `429`, `5xx`, timeouts and connection errors. Never on `4xx` validation errors |
| Circuit breaker | Open after N consecutive failures, half open probe, so a TikTok outage does not exhaust our workers |
| Timeouts | Explicit connect and read timeouts on every call. No unbounded waits |
| Response parsing | Every response validated against a schema. An unexpected shape fails loudly instead of writing garbage |
| Logging | Every call logged with endpoint, status, duration and correlation id. Never with tokens |
| Metrics | Call count, error rate and latency per endpoint, plus remaining quota where the API reports it |
| Fixtures | Real responses recorded and stored as test fixtures, including error bodies |

## Sync design

| Aspect | Decision |
| --- | --- |
| First sync | Full pagination through the library, in a job, with visible progress |
| Incremental | On a schedule, plus a manual "sync now" with its own rate limit |
| Reconciliation | New videos inserted, remote fields updated, videos no longer returned marked `deleted_remotely_at` |
| Local fields | **Never touched.** Tags, category, notes and job history survive every sync. Gate tested |
| Metrics | Snapshot into `metric_snapshots` on each sync, feeding trend based automation conditions |
| Freshness | `last_synced_at` per account and `metrics_updated_at` per video, both exposed in the API and shown in the UI |
| Failure | A failed sync leaves the previous data intact and surfaces a banner with a real remedy |

## Upload and publish design

- Validate **before** sending anything upstream: container, codec, resolution, aspect ratio, duration, size and
  frame rate, against the documented limits. A clear rejection locally beats a cryptic upstream error.
- Chunked and resumable upload, with a checksum verified after the final part.
- Poll the publish status until a terminal state, with a timeout and a sensible interval.
- Map every documented upstream error code to a domain error with a plain language remedy. Exhaustively, with
  a unit test over the list.
- Never retry a publish without confirming it did not already succeed. A duplicate post is a real harm, not a
  minor annoyance.

## Constraints to encode in validation

Fill these in from the official documentation during `S-01`, then encode them as constants with a link to the
source next to each one:

- Maximum file size, maximum duration, minimum duration
- Accepted containers and codecs
- Resolution and aspect ratio limits
- Caption maximum length and hashtag and mention handling
- Posts per day per user, and any per app quota
- Rate limits per endpoint

Values change. Each constant carries a comment with the source URL and the date it was checked, and the Phase 6
documentation pass re-verifies them.

## Compliance

- Only official, documented APIs. No scraping, no private endpoints, no browser automation of the TikTok UI.
- Only data the authenticated user owns. No harvesting of other people's content or profiles.
- The user's data is theirs: export and deletion are supported. See [security.md](security.md).
- Platform terms of service and developer policies are reviewed during `S-01` and re-checked before any public
  release. Anything in this plan that conflicts with them loses.
