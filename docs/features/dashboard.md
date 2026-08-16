# Feature: video dashboard

The home screen. Everything the user owns, as cards, with sorting, filtering, taxonomy and multi select.

Implemented in Phase 1 group `1.2` of [../TODO.md](../TODO.md).

## Layout

```
┌──────────────────────────────────────────────────────────────────────┐
│ Top bar: search · sync status · account switcher · user menu         │
├──────────┬───────────────────────────────────────────────────────────┤
│          │ Toolbar: [Sort ▾] [↑↓] [Filters ▾] [Grid|List] [12 of 248]│
│ Sidebar  ├───────────────────────────────────────────────────────────┤
│          │  ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐                      │
│ All      │  │ card │ │ card │ │ card │ │ card │                      │
│ Untagged │  └──────┘ └──────┘ └──────┘ └──────┘                      │
│          │  ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐                      │
│ Categories  │ card │ │ card │ │ card │ │ card │                      │
│  Tutorials  └──────┘ └──────┘ └──────┘ └──────┘                      │
│   Svelte │                                                           │
│  Shorts  │                                                           │
│          │                                                           │
│ Tags     ├───────────────────────────────────────────────────────────┤
│  #howto  │ Selection bar (appears on select):                        │
│  #demo   │ 12 selected · Tag · Category · Privacy · Delete · Clear   │
└──────────┴───────────────────────────────────────────────────────────┘
```

## The video card

| Element | Detail |
| --- | --- |
| Cover | 9:16 aspect box, lazy loaded, blurred placeholder, fallback for a broken or expired URL |
| Duration | Overlaid bottom right, `mm:ss` |
| Status badge | Only when the status is not `PUBLISHED`. Colour coded, with a tooltip |
| Caption | Two line clamp, full text on hover and in the detail view |
| Metrics | Views, likes, comments. Compact formatting (`12.4k`), full value in the title attribute |
| Published | Relative for recent (`3 days ago`), absolute beyond 30 days |
| Tags | Up to three chips, then `+n` |
| Checkbox | Top left, visible on hover and always when the card is selected |
| Menu | Top right, per card actions: open in TikTok, edit, tag, change privacy, delete |

States: `loading` (skeleton), `error` (retry affordance), `selected` (ring plus tinted background),
`processing` (progress overlay while a job touches it), `stale` (subtle indicator when metrics are older than
the freshness threshold).

## Sorting

| Sort key | Notes |
| --- | --- |
| Published date | Default, descending |
| Views, likes, comments, shares | Null metrics always sort last, in both directions |
| Duration | |
| Title or caption | Locale aware, case insensitive |
| Recently added to the library | Local `created_at`, useful after a first sync |

A direction toggle sits next to the sort control, and every sort has a stable tiebreaker on id so cursor
pagination cannot skip or repeat.

## Filtering

| Filter | Control |
| --- | --- |
| Status | Multi select chips |
| Privacy | Multi select chips. **Only meaningful for videos posted through this app**, since the API does not expose privacy on existing videos. Everything else is `UNKNOWN` and the filter says so |
| Category | Sidebar selection, includes descendants by default with a toggle for "this category only" |
| Tags | Multi select, AND or OR toggle |
| Published date | Presets (7, 30, 90 days, this year) plus a custom range |
| Metric ranges | Minimum and maximum for views, likes, comments |
| Has no tags | A single toggle, because "what have I not organised yet" is a real daily question |
| Free text | Debounced search over caption and title |

Rules:

- **All state lives in the URL.** Sort, direction, filters, search and page. A view is shareable and survives a
  reload, which also makes E2E assertions honest.
- Active filters are shown as removable chips above the grid, with a "clear all".
- Saved views (a named filter set) are in the backlog, not version 1.

## Multi select

The interaction that makes the product worth using. It must feel like a file manager.

| Interaction | Result |
| --- | --- |
| Click the checkbox | Toggle one |
| Click the card | Open the detail view, does not select. Selection is deliberate |
| `Shift` + click a checkbox | Select the range from the last selected to this one |
| `Cmd/Ctrl` + `A` | Select everything currently loaded |
| "Select all N matching" | Selects the **filter**, not the loaded ids. Crucial for a 2,000 video library |
| `Escape` | Clear the selection |

Rules:

- Selection survives pagination, sorting and filter narrowing. Narrowing a filter keeps only the still matching
  items and says so.
- A selection is represented as either an explicit id list or a filter reference, and it is sent to the API in
  that form. We never materialise 2,000 ids in a URL.
- The selection bar is sticky, shows the exact count, and disables actions that the connected account's
  capabilities do not allow, with a tooltip explaining why.
- Every destructive action from the selection bar goes through the dry run preview described in
  [batch-actions.md](batch-actions.md).

## Detail view

Opens as a side sheet, not a full page navigation, so the user does not lose their place in the grid.

Contents: large cover with a link out to TikTok, full caption, all metrics with a small trend chart from
`metric_snapshots`, category and tags editing, private notes, publish and sync timestamps, and the history of
every job and flow that touched this video.

## Performance requirements

| Requirement | Target |
| --- | --- |
| Server query | p95 under 300 ms with 10,000 videos, measured in the Phase 6 gate |
| First contentful paint | Under 1.5 s on a normal connection |
| Grid interaction | No dropped frames when scrolling 500 loaded cards |
| Page size | 25 by default, up to 100 |
| Images | Lazy loaded, correctly sized, never full resolution covers in the grid |

Virtualise the grid only if profiling shows it is needed. Do not virtualise on a hunch, it costs a lot of
accessibility and selection complexity.

## Accessibility

- Every action reachable by keyboard, with a visible focus ring.
- The grid is a list with proper roles. Cards are focusable and expose their selected state to assistive tech.
- Selection count changes are announced through a live region.
- Colour is never the only signal. Status badges carry text or an icon as well.
- Zero critical axe violations, enforced in the E2E suite.

## Empty and error states

| State | Message and action |
| --- | --- |
| No account connected | Explain the value, one button: connect a TikTok account |
| Connected, first sync running | Progress, with an explanation that it may take a while for a large library |
| No videos at all | Prompt to upload |
| No results for this filter | Show the active filters and offer to clear them |
| Load failed | The reason in plain language, a retry button, and the trace id for support |
| Account needs reconnect | Persistent banner, one button to reconnect, and an explanation of what is stale |

## Out of scope for version 1

Timeline and calendar views, a comparison mode, in-grid inline caption editing, drag to reorder (TikTok owns
ordering), and any bulk action not listed in [batch-actions.md](batch-actions.md).
