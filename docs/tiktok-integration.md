# TikTok integration

What the official API can and cannot do, verified against the documentation.

> **Spike `S-01` is complete.** Sources are linked per row and were checked on 2026-08-16. Limits change, so
> the Phase 6 documentation pass re-verifies them.

**Our app:** `adhoctok`, registered on the TikTok developer portal with a sandbox environment. Credentials live
in `local_dev_variables.env`, which is gitignored and must never be committed.

## Headline finding

**The official API cannot delete, unpublish, change the privacy of, or edit an existing video.**

This is a limit of the official API, **not a limit of what the product can do**. A logged in browser session
can do all of it, which is the subject of [adr/0010-browser-session-execution.md](adr/0010-browser-session-execution.md).
Read both sections before concluding anything about scope.

The [scopes reference](https://developers.tiktok.com/doc/tiktok-api-scopes/) lists exactly three video scopes,
and no others exist:

| Scope | What it grants |
| --- | --- |
| `video.list` | Read a user's **public** videos. Targets `/v2/video/list/` and `/v2/video/query/` |
| `video.publish` | Directly post content to a user's profile. Targets `/v2/post/publish/...` |
| `video.upload` | Share content to the creator's account **as a draft**, to be finished in the TikTok app |

There is no `video.delete`, no `video.update` and no privacy management scope. This is not an oversight we can
work around with a cleverly chosen endpoint. The capability does not exist for third party apps.

Consequences **for the API strategy specifically**:

1. **Batch delete and batch privacy change are not available through the API.** Under the `API` strategy they
   degrade to tracked manual checklists, as specified in
   [features/batch-actions.md](features/batch-actions.md). Under the `BROWSER` strategy they are real actions.
2. **We cannot read the privacy of an existing video.** `privacy_level` is write only, at post time. It is not
   a field on the Video object, so there is nothing to sync.
3. **`video.list` returns public videos only.** A video the user makes private will **disappear** from an API
   sync. It has not been deleted, and we must not report it as deleted. See the disappearance problem below.
   The browser strategy does not have this problem, because TikTok Studio lists everything.

## Prior art, and what it actually tells us

This section exists because an earlier version of this document got it wrong. It asserted that the third party
tools call TikTok's undocumented internal endpoints. They do not.

| Tool | Form factor | Actual mechanism |
| --- | --- | --- |
| [SocialEraser](https://github.com/socialeraser/SocialEraser) | Chrome extension, **MIT licensed, source readable** | **DOM automation of TikTok Studio.** Click the row action button, wait for the popover, click the delete icon, wait for the confirm modal, click confirm. About 0.9 seconds per item, so roughly 15 minutes per 1,000 videos |
| DeleteTik | Chrome extension | Runs locally in the user's own session, no password |
| Redact | **Desktop application**, 1M+ users | The user signs in to each platform inside the app. All processing happens on device and the login never reaches a Redact server |

Three things follow, and they shaped [ADR-0010](adr/0010-browser-session-execution.md):

1. **No request signing is involved.** These tools never have to defeat `X-Bogus`, `X-Gnarly`, `msToken` or
   `_signature`, because TikTok's own page makes the requests. Signing is only a problem for approaches that
   call the endpoints directly, which is precisely why we are not taking that approach.
2. **The form factor is not fixed.** Redact proves a locally installed application works as well as an
   extension. Our Compose stack on the user's own machine is the same shape.
3. **The failure modes are known in advance**, because the reference implementation is open source: overlay
   toasts stealing clicks, list re-render after each delete, selectors changing under you, and the need for a
   stuck timeout. [ADR-0010](adr/0010-browser-session-execution.md) turns each into a design requirement.

## Capability matrix

Verified 2026-08-16. `YES` means documented and available to a third party app with the listed scope.

| Capability | Status | Endpoint and source | Notes |
| --- | --- | --- | --- |
| OAuth authorisation and refresh | **YES** | [Login Kit web](https://developers.tiktok.com/doc/login-kit-web/) | Access token 24 h, refresh token 365 days |
| Read profile | **YES** | [`GET /v2/user/info/`](https://developers.tiktok.com/doc/tiktok-api-v2-get-user-info) | `user.info.basic` gives open id, avatar, display name |
| Read follower and like totals | **YES** | Same | Needs `user.info.stats` |
| List the user's videos | **YES, public only** | [`POST /v2/video/list/`](https://developers.tiktok.com/doc/tiktok-api-v2-video-list/) | Sorted `create_time` desc. Not sortable or filterable server side |
| Read per video metrics | **YES** | [`POST /v2/video/query/`](https://developers.tiktok.com/doc/tiktok-api-v2-video-query/) | `like_count`, `comment_count`, `share_count`, `view_count` |
| Read cover and share URL | **YES** | Both endpoints | `cover_image_url` has a TTL and must be refreshed |
| Read privacy of an existing video | **NO** | Not a Video object field | Write only, at post time |
| Upload a video file | **YES** | [`POST /v2/post/publish/video/init/`](https://developers.tiktok.com/doc/content-posting-api-reference-direct-post/) | Chunked `PUT` to the returned `upload_url` |
| Direct post (publish immediately) | **YES** | Same | `video.publish`. Audit required to reach a public audience |
| Post as a draft to finish in the app | **YES** | Share Video API | `video.upload` |
| Set privacy at post time | **YES** | Direct post `post_info.privacy_level` | Must match `privacy_level_options` from `creator_info/query` |
| Schedule a post server side | **NO** | Not offered | Our own scheduler publishes at the chosen moment |
| **Change privacy of an existing video** | **NO** | No scope exists | Tracked manual action |
| **Delete an existing video** | **NO** | No scope exists | Tracked manual action |
| **Edit caption of an existing video** | **NO** | No scope exists | Tracked manual action |
| Webhooks for post status | **NO** | Not offered for these scopes | Poll the publish status endpoint |

The API exposes this matrix per connected account at `GET /v1/accounts/{id}/capabilities`, and the UI renders
from it. An action that is not supported is never offered, greyed out with an explanation instead of failing
after the click.

## Capability by execution strategy

The matrix above is the `API` strategy. This is the comparison that actually drives the product.

| Capability | `API` | `BROWSER` |
| --- | --- | --- |
| List videos | Public only | **Everything, including private and unlisted** |
| Read metrics | Yes | Yes, from TikTok Studio |
| Read privacy of an existing video | **No** | **Yes** |
| Upload and publish | **Yes, and better** | Possible, but file upload is clumsier |
| Schedule a post | No, our scheduler does it | No, our scheduler does it |
| **Delete a video** | **No** | **Yes** |
| **Change privacy of an existing video** | **No** | **Yes** |
| **Edit a caption** | **No** | **Yes** |
| Requires app audit | **Yes**, long lead time | No |
| Breaks when TikTok ships a UI change | No | **Yes**, this is the real cost |
| Within terms of service | Yes | **No**, accepted risk, see ADR-0010 |

The two strategies are complementary rather than competing. `BROWSER` for reading the library and for
lifecycle actions, `API` for publishing if and when the app is audited.

## Verified constants

Encode these as named constants with the source URL and check date beside each one.

| Constant | Value | Source |
| --- | --- | --- |
| Video list page size | Default 10, **maximum 20** | List Videos |
| Video list cursor | A UTC Unix timestamp in **milliseconds**, not an opaque token | List Videos |
| Video list ordering | `create_time` descending, fixed | List Videos |
| Video query batch size | Up to **20 video ids** per request | Query Videos |
| Caption maximum | **2200 UTF-16 runes** | Direct Post |
| Publish init rate limit | **6 requests per minute** per user access token | Direct Post |
| `upload_url` validity | **1 hour** from issuance | Direct Post |
| Upload content types | `video/mp4`, `video/quicktime`, `video/webm` | Direct Post |
| Access token lifetime | 86400 seconds (24 hours) | Display API get started |
| Refresh token lifetime | 31536000 seconds (365 days) | Display API get started |
| Suggested sync interval | Every 12 hours | Display API get started |
| Privacy levels | `PUBLIC_TO_EVERYONE`, `MUTUAL_FOLLOW_FRIENDS`, `FOLLOWER_OF_CREATOR`, `SELF_ONLY` | Direct Post |

### Video object fields

The complete set available to us: `id`, `create_time`, `cover_image_url`, `share_url`, `video_description`,
`duration`, `height`, `width`, `title`, `embed_html`, `embed_link`, `like_count`, `comment_count`,
`share_count`, `view_count`.

Note what is **absent**: privacy, structured hashtags, audience demographics, watch time, and any status field.
The data model must not promise what the source cannot provide.

## Consequences for the design

### Sorting and filtering must be ours

`/v2/video/list/` sorts by creation time and offers no filtering. Every headline feature (sort by views, filter
by metric range, free text search) therefore depends on **syncing the whole library into PostgreSQL and
querying locally**. This validates the architecture, and it makes the sync engine load bearing rather than a
convenience.

### Paging a large library is slow

20 videos per page, and metrics come in batches of 20 ids. A 2,000 video library is 100 list calls plus 100
query calls for a full sync. Design accordingly: full sync is a background job with visible progress,
incremental sync is the normal path, and manual "sync now" is rate limited.

### The disappearance problem

Because `video.list` returns public videos only, a video vanishing from the response has at least three
possible causes:

1. It was deleted.
2. It was made private or friends only.
3. It is temporarily unavailable, under review, or the response was incomplete.

**We must not treat absence as deletion.** The reconciliation rule: on absence, confirm with
`/v2/video/query/` by id, then mark the video `UNAVAILABLE` with a `last_seen_at`, never `deleted`, and say so
honestly in the UI ("no longer visible to the API, it may be private or removed"). This also gives the manual
privacy checklist a natural verification step, since a completed item should subsequently disappear from the
public list.

### Audit is the gate to being useful

Unaudited clients can only post to private accounts, and posted content is restricted to private viewing. The
error `unaudited_client_can_only_post_to_private_accounts` blocks `/publish/video/init/` outright. Apply for
the audit early, per task `0.1.6`.

### Quotas are per user and per client

`spam_risk_too_many_posts` is a per user daily cap, and `reached_active_user_cap` is a per client daily cap on
active publishing users. Both must surface as clear, non retryable domain errors with a real explanation, not
as a generic failure.

## The browser strategy

Decided in [adr/0010-browser-session-execution.md](adr/0010-browser-session-execution.md). Implementation
notes, most of them learned from the open source reference implementation rather than from first principles.

### Shape

A Playwright driven Chromium in the `worker` container, with a persistent profile in a named volume. The user
signs in once through a live view of that browser. Nothing leaves their machine.

```
worker container
├── Playwright + Chromium
├── /profile          (named volume, persistent session, encrypted at rest)
└── selectors.json    (versioned, updatable without a rebuild)
```

### Target surface

TikTok Studio, not the public profile page. It lists the full library including private videos, with metrics
and privacy visible, and it has a per row action menu. The delete interaction is three steps: row action
button, delete item in the popover, confirm in the modal.

### Design requirements

| Requirement | Why, concretely |
| --- | --- |
| Selectors in config, never inline | TikTok ships UI changes. The reference project's own rule is "all selectors go through config". Ours live in a versioned file with an ordered fallback list per target |
| Canary check before each run | If the expected selectors do not resolve, fail with `SELECTORS_STALE`. Never click blind on an unrecognised page |
| Dismiss overlays before every click | The "Deleted successfully" toast lands on top of the next row's button and steals the click. A documented failure in the reference implementation |
| Wait for list re-render after each delete | Studio refetches and re-renders the whole list. Counting row containers is wrong, because skeletons count too. Wait on action buttons |
| Human pacing with jitter | About one action per second, configurable, never parallel within an account |
| Stuck watchdog | Abort the run after a progress timeout rather than looping forever |
| `AWAITING_ATTENTION` state | Captcha or a login challenge pauses the job and surfaces the live browser view for the user to resolve, then resumes |
| Resumable per item | Already how batch jobs work. A restart continues rather than repeats |
| Archive prompt before first bulk delete | Deletion is irreversible within seconds. TikTok's "Download your data" is the only recovery |

### Testing it

DOM automation cannot be unit tested meaningfully, so the layers split:

- **Unit:** filter predicates, pacing maths, selector resolution with fallbacks, all pure.
- **Integration:** a **recorded snapshot of TikTok Studio's DOM** served locally. Fast, deterministic, and it
  catches selector regressions. Refresh the snapshot on a schedule.
- **Canary:** a scheduled job that runs the selector check against the real site and alerts on drift, so we
  learn about breakage before a user does.
- **Manual:** a documented smoke run against a real sandbox account before each release that touches this path.

## The manual fallback

Still specified, now as a **degradation path** rather than the primary mechanism. It is used when the browser
strategy is unavailable: selectors are stale, the session needs re-authentication, or the user has chosen the
`API` strategy.

A tracked manual action list. See the fallback section in
[features/batch-actions.md](features/batch-actions.md). The user still gets the organisation, the list, the
deep links and the audit trail, and performs the final tap themselves.

## App registration and audit

Record in task `0.1.6`:

- Products to request: Login Kit, Display API, Content Posting API.
- Scopes to request, minimum viable: `user.info.basic`, `video.list`, `video.publish`. Add `user.info.stats`
  only when profile statistics are actually shown, and `video.upload` only if the draft path is offered.
- Sandbox constraints: which test users can be used, and what changes after audit.
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

## Constraints still to confirm

The media transfer guide holds the remaining file level limits. Confirm and encode them alongside the verified
constants above, each with its source URL and check date:

- Maximum file size, maximum duration, minimum duration
- Accepted codecs beyond the three container types already confirmed
- Resolution, aspect ratio and frame rate limits
- Exact numeric value of the per user daily post cap

The Phase 6 documentation pass re-verifies every constant on this page.

## Compliance and boundaries

The browser strategy is a deliberate, documented exception, not a licence to do anything. The boundaries that
remain, and they are not negotiable:

- **Only the authenticated user's own account and own content.** No harvesting of other people's videos,
  profiles, followers or comments. Nothing this product does touches data the signed in user does not own.
- **No credential handling on our side.** The user signs in to TikTok directly, in their own browser instance,
  on their own machine. We never see, store or transmit a password, and the session never leaves the host.
- **No hosted multi tenant version of the browser strategy.** It is a self hosted capability. Running it
  centrally would concentrate account credentials and originate traffic from one place, which is the thing
  [ADR-0010](adr/0010-browser-session-execution.md) rejects.
- **No evasion beyond human pacing.** We act at human speed with jitter and rate caps because that is polite
  and reduces account risk. We do not fingerprint spoof, rotate proxies, solve captchas automatically, or
  otherwise disguise the traffic.
- **Honest disclosure.** The user is told, before first use of the browser strategy, that it automates their
  own session, that it is very likely contrary to TikTok's terms, and that account risk exists. No burying it
  in a licence file.
- **Deletion is irreversible.** Prompt for TikTok's official data archive before the first bulk delete.

The user's data is theirs: export and deletion are supported. See [security.md](security.md).

Terms of service position is recorded in [ADR-0010](adr/0010-browser-session-execution.md) and was accepted by
the product owner with the risk stated. If TikTok ships an official endpoint for these actions, the `API`
strategy takes over and this exception narrows.

