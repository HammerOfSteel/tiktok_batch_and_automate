# Glossary

The ubiquitous language. If a term here means something in this project, it means the same thing in the code,
the API, the UI and the documentation. When one of them drifts, fix the drift rather than inventing a synonym.

## Domain terms

| Term | Definition |
| --- | --- |
| **Account** (Connected Account) | A TikTok account a user has linked through OAuth. A user may have several. Holds tokens, scopes and sync state |
| **Asset** | A video file uploaded into this system that has not been published yet. Becomes linked to a Video once published |
| **Audit entry** | An append only record of who did what, to what, with what parameters and what outcome |
| **Batch job** | One user intent applied to many targets, expanded into items and executed asynchronously |
| **Capability** | Something the platform actually allows a third party app to do, reported per account and used to decide what the UI offers |
| **Category** | A node in the user's hierarchical taxonomy. A video has at most one. Maximum depth 3. Local only |
| **Cursor** | An opaque, signed token encoding a sort position plus a tiebreaker id. Used for pagination |
| **Dry run** | Executing a job or flow with every side effect suppressed, producing a full preview and writing nothing |
| **Flow** | A saved automation graph: nodes, edges and configuration |
| **Flow run** | One execution of a flow, against a snapshot of the graph as it was at trigger time |
| **Item** (Job item) | One target within a batch job, with its own status, attempts and error |
| **Local field** | Data this system owns: tags, category, notes, job history. Never overwritten by a sync |
| **Manual action** | A tracked checklist item for something the platform does not expose to third parties, completed by the user in the TikTok app |
| **Metric snapshot** | A point in time capture of a video's counters, used for trend based conditions |
| **Node** | One step in a flow. A trigger, a filter or logic step, or an action |
| **Remote field** | Data TikTok owns: caption, cover, metrics, privacy, publish time. Refreshed by syncing |
| **Run trace** | The step by step record of a flow run, with input, output, duration and outcome per step |
| **Selection** | The set of videos an action applies to. Either explicit ids or a filter, resolved once at job creation |
| **Stale** | Data older than the freshness threshold for its use. Stale metrics must not silently drive automation decisions |
| **Sync** | Reconciling our local mirror with TikTok. Updates remote fields, never touches local fields |
| **Tag** | A flat, per user label. A video may have many. Local only |
| **Taxonomy** | Categories and tags together |
| **Trigger** | The node that starts a flow: a schedule, a manual click, or an event |
| **Video** | Our record of a TikTok post, remote fields plus local fields. Also covers drafts that have no remote id yet |

## Architecture terms

| Term | Definition |
| --- | --- |
| **Adapter** | An infrastructure implementation of a port. For example `LiveTikTokAdapter` |
| **Aggregate** | A cluster of objects with one root, saved and invariant checked as a unit |
| **Application layer** | Use cases and port definitions. Orchestrates, does not decide business rules |
| **Bounded module** | A cohesive feature area with a published contract. `library`, `taxonomy`, `batch`, `automation` |
| **Delivery layer** | Controllers, queue consumers, cron entry points, the SvelteKit app |
| **Domain layer** | Pure business logic. No framework, no IO, no database awareness |
| **Domain event** | Something that happened in the domain that other modules may react to |
| **Infrastructure layer** | Database, queue, storage, mail, third party clients |
| **Port** | An interface defined by the application layer and implemented by infrastructure |
| **Repository** | The persistence port for one aggregate. Returns domain objects, never database rows |
| **Use case** | One application operation, with one transaction boundary. For example `PublishVideo` |
| **Value object** | An immutable, self validating type with no identity. `Slug`, `Caption`, `Privacy` |

## Process terms

| Term | Definition |
| --- | --- |
| **ADR** | Architecture Decision Record. A numbered, immutable record of a decision and its reasoning |
| **Gate** | A hard checkpoint in [TODO.md](TODO.md). Everything listed must pass before the next phase begins |
| **Phase** | A block of work with one goal and one exit condition. Ends with a push and a tag |
| **Smoke test** | A shallow check that a running stack is alive. Answers "is it up", not "is it correct" |
| **Spike** | Timeboxed research whose output is a written decision, never shipped code |
| **Task** | A numbered unit of work in [TODO.md](TODO.md), ending in exactly one commit |

## Words we avoid

Ambiguity costs more than typing.

| Avoid | Use instead | Why |
| --- | --- | --- |
| "Post" as a noun | Video | We already have `POST` as a verb, and the domain word is Video |
| "Delete" for local removal | Unassign, uncategorise, archive | Delete means the video is gone from TikTok. Never blur that |
| "Sync" for pushing changes upstream | Publish, apply | Sync is one direction only: TikTok to us |
| "Draft" for anything unpublished | Asset (a file) or a `DRAFT` Video | Two different things, two different words |
| "Workflow" | Flow | One word, used everywhere, including in the UI |
| "Bulk" and "batch" mixed | Batch | Pick one. It is batch |
| "User" for a TikTok profile | Account | User means a user of this application |
