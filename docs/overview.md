# Product overview

## The problem

Creators and small teams who post a lot of TikTok content end up doing the same repetitive work by hand:

- Uploading videos one at a time, retyping captions and settings for each one.
- Hunting through the app to find an old video that should be taken down or made private.
- Keeping a spreadsheet on the side because the app has no way to group or label content.
- Remembering to publish at the right time, and remembering to clean up afterwards.

The native TikTok apps are built for consuming and for posting one video at a time. There is no bulk mode,
no taxonomy and no scheduling logic beyond a basic schedule field.

## The product

A self hostable web application with two halves.

### 1. The library

A dashboard of everything the connected account owns. Cards, metrics, sorting, filtering, tagging and, most
importantly, **multi select with bulk actions**. Choose forty videos and act on all of them in one go, then
watch the job progress in real time.

### 2. The automation workspace

A visual, node based canvas where a non technical user builds flows such as:

- "Every Tuesday at 18:00, take the next video from the `Shorts` queue and publish it as public."
- "When a video is 30 days old **and** has fewer than 1,000 views, make it private and tag it `underperformer`."
- "When a video passes 100,000 views, add it to the `Best of` category and notify me."

Triggers, conditions and actions, connected with wires. No code, no YAML.

## Who it is for

| Persona | Needs | Primary surface |
| --- | --- | --- |
| **Solo creator** | Post consistently, clean up old content, stop babysitting the upload screen | Dashboard, batch upload, simple schedules |
| **Small agency or team** | Manage several accounts, keep content organised per client or campaign | Taxonomy, saved filters, job history |
| **Power user or automator** | Rules driven lifecycle management for content | Automation workspace |

## Product goals

1. **One place, everything.** No spreadsheet on the side.
2. **Bulk by default.** Any action that makes sense for one video makes sense for fifty.
3. **Explainable automation.** A user can always see why a flow did something, and what it will do next.
4. **Safe by construction.** Destructive actions are previewed, confirmed, rate limited and reversible where
   the platform allows it.
5. **Self hostable.** Clone, `docker compose up`, add credentials, done.

## Non goals

Explicitly out of scope, at least for version 1. Saying so keeps the plan honest.

- **Not an editor.** No trimming, no filters, no effects, no captions burn in. Videos arrive finished.
- **Not an analytics suite.** Metrics are shown to support decisions and automation conditions, not to
  replace TikTok Analytics or a BI tool.
- **Not a multi platform poster.** TikTok only. The integration layer is kept behind a port so a second
  platform stays possible, but no second platform is built.
- **Not a scraper.** Only official APIs and only data the authenticated user owns. Anything that requires
  automating the TikTok web UI or bypassing the API is off the table.
- **Not a comment or DM manager.**

## Success criteria for version 1

| # | Criterion | How it is measured |
| --- | --- | --- |
| SC-1 | A user connects a TikTok account and sees their real videos | Manual acceptance plus Playwright run against the sandbox |
| SC-2 | Batch upload of 10 videos completes without manual steps between them | Job report shows 10 of 10 succeeded |
| SC-3 | A dashboard of 1,000 videos filters and sorts in under 300 ms server side | k6 or autocannon run in the Phase 6 gate |
| SC-4 | A non technical tester builds a working scheduled flow in under 10 minutes with no help | Moderated usability test, 3 participants |
| SC-5 | Every batch and automation action is auditable after the fact | Audit log query returns actor, action, target and outcome |

## Core concepts

Short version. Full definitions in [glossary.md](glossary.md), full schema in [data-model.md](data-model.md).

- **Account.** A TikTok account connected through OAuth. A user may connect more than one.
- **Video.** A synced representation of a TikTok post, plus local only fields (tags, category, notes).
- **Asset.** A file uploaded to this system that has not been published yet.
- **Taxonomy.** Categories with subcategories, plus flat tags. Local only, never pushed to TikTok.
- **Selection.** The set of videos a user is acting on, either explicit ids or a saved filter.
- **Batch job.** One user intent (for example "make these 40 private") expanded into per video work items.
- **Flow.** A saved automation graph of nodes and edges.
- **Run.** One execution of a flow, with a step by step trace.

## How the phases map to the product

| Phase | What a user can do at the end of it |
| --- | --- |
| 0 | Nothing. Developers can run the stack and CI is green. |
| 1 | Explore the entire UI against a real API with seeded data. No auth, no TikTok. |
| 2 | Register, log in, connect a TikTok account, and see only their own data. |
| 3 | See their real TikTok videos, synced and refreshed automatically. |
| 4 | Batch upload and publish for real, and run whatever lifecycle actions the API allows. |
| 5 | Build, run and monitor automations. |
| 6 | Rely on it. Observability, performance and operational docs are in place. |

Detail and gates: [TODO.md](TODO.md).

## Key risks

| ID | Risk | Impact | Mitigation |
| --- | --- | --- | --- |
| R-1 | ~~TikTok's public API offers no delete or privacy update~~ **Confirmed by `S-01`, and resolved.** The official API cannot do it, the browser strategy can | Was: guts two headline features | [ADR-0010](adr/0010-browser-session-execution.md). Manual checklist survives as the degradation path |
| R-2 | ~~App audit is slow or rejected~~ **No longer on the critical path.** The browser strategy needs no audit | Was: blocks Phases 3 to 5 | Audit still pursued for the `API` publishing strategy, but nothing waits on it |
| R-6 | **TikTok ships a UI change and the browser strategy breaks** | Delete, privacy and full-library sync stop working | Selectors in versioned config, canary check before every run, scheduled drift alert, fast update path, manual fallback |
| R-7 | **Account restriction from automated activity** | User loses the account the product manages | Human pacing with jitter, rate caps, no evasion beyond pacing, explicit risk disclosure and opt in |
| R-3 | Rate limits make large batches slow | Poor UX on big selections | Central throttled client, queue backpressure, honest ETA in the UI, resumable jobs |
| R-4 | Automation canvas becomes a general purpose programming language | Endless scope | Fixed node catalogue per [features/automation-engine.md](features/automation-engine.md), no scripting node in v1 |
| R-5 | Metrics drift between local cache and TikTok | Wrong automation decisions | Freshness stamp on every metric, staleness threshold in conditions, visible "last synced" |
