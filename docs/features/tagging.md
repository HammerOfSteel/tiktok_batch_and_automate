# Feature: tags and categories

Local organisation for a library the platform gives you no way to organise. Built in Phase 1 group `1.3` of
[../TODO.md](../TODO.md).

## Two systems, on purpose

| | Categories | Tags |
| --- | --- | --- |
| Shape | Hierarchical tree, maximum depth 3 | Flat |
| Cardinality | A video has **at most one** category | A video has **any number** of tags |
| Mental model | "Where does this live" | "What is true about this" |
| UI | Sidebar tree with counts | Chips, autocomplete input |
| Typical use | Client, campaign, series, format | `tutorial`, `evergreen`, `sponsored`, `queue`, `archived` |

One would have been simpler, but the two answer genuinely different questions. A folder tree cannot express
overlapping truths, and a flat tag list cannot express a hierarchy without naming conventions that nobody keeps
consistent.

Both are **local only**. Nothing here is ever pushed to TikTok, and nothing here is ever destroyed by a sync.
That guarantee is asserted by an integration test in the Phase 3 gate.

## Categories

- A tree, maximum depth 3 (category, subcategory, sub-subcategory). Deeper than that and nobody can find
  anything.
- Name plus an auto generated slug, unique among siblings for the same user.
- Optional colour, used for the sidebar dot and card accent.
- Manual ordering through a `position` field, since alphabetical is rarely the order a person thinks in.
- Cycles are impossible: a category cannot be moved under its own descendant. Enforced in the domain and unit
  tested.

Filtering by a category **includes its descendants by default**, with a "this category only" toggle. Counts in
the sidebar show the inclusive number, which is what people expect.

Deleting a category asks explicitly what to do with its contents:

| Choice | Effect |
| --- | --- |
| Move children up | Subcategories become children of the deleted node's parent |
| Delete the subtree | Everything under it goes too, with a typed confirmation |
| Videos | Always uncategorised, never deleted. A taxonomy operation must not destroy content |

## Tags

- Flat, per user, case insensitive uniqueness (`Tutorial` and `tutorial` are the same tag).
- Optional colour, auto assigned from a palette on creation so a library is visually scannable immediately.
- Created inline from the assignment input. Do not make people visit a settings page to invent a word.
- Rename updates every assignment, since the tag is an entity and not a string on the video.
- Merge two tags into one, because duplicates will happen no matter how good the autocomplete is.
- Deleting a tag removes the assignments and nothing else.

Assignment is idempotent by construction: the join table's primary key is `(video_id, tag_id)`, so assigning
the same tag twice is a no-op rather than an error.

## Assignment surfaces

| Surface | Interaction |
| --- | --- |
| Card menu | Quick tag and quick category for one video |
| Detail sheet | Full editing with autocomplete |
| Selection bar | Apply to the whole selection, as a batch job |
| Drag and drop | Drag a card or a selection onto a category in the sidebar |
| Automation | `Add tag`, `Remove tag`, `Set category` action nodes |

Bulk assignment goes through the batch job machinery in [batch-actions.md](batch-actions.md), so it is
consistent, progress reporting works, and it is audited. It is fast enough to feel instant because it is a
purely local operation, but it is still a job.

## Suggestions, not automation

Version 1 keeps this dumb on purpose:

- Suggest recently used tags first in the autocomplete.
- Suggest tags derived from hashtags already present in the caption, as a one click accept. No automatic
  application.

No machine learning, no clustering, no "smart" auto categorisation. It would be wrong often enough to destroy
trust in the organisation the user built by hand.

## API summary

Conventions in [../api-guidelines.md](../api-guidelines.md).

| Endpoint | Purpose |
| --- | --- |
| `GET /v1/categories` | The tree with inclusive video counts |
| `POST /v1/categories` | Create, optionally with a `parent_id` |
| `PATCH /v1/categories/{id}` | Rename, recolour, reparent, reorder |
| `DELETE /v1/categories/{id}` | With an explicit strategy for children |
| `GET /v1/tags` | List with usage counts |
| `POST /v1/tags` | Create |
| `PATCH /v1/tags/{id}` | Rename or recolour |
| `POST /v1/tags/{id}/merge` | Merge into another tag |
| `DELETE /v1/tags/{id}` | Delete and unassign |
| `PATCH /v1/videos/{id}` | Set category and tags for one video |
| `POST /v1/batch-jobs` | Bulk assignment with `ADD_TAG`, `REMOVE_TAG` or `SET_CATEGORY` |

## Testing focus

- Slug generation: unicode, emoji, punctuation, collisions among siblings, and very long names.
- Cycle prevention when reparenting, including moving a node under its own grandchild.
- Depth limit enforcement on both create and reparent.
- Case insensitive tag uniqueness, including unicode case folding.
- Merge preserves every assignment and leaves no duplicate rows.
- Deleting a category never deletes a video. This is worth an explicit test with an assertive name.
- A full resync leaves every tag, category and note exactly as it was.
