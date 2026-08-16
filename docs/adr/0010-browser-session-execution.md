# ADR-0010: Browser session automation as the primary execution strategy

- **Status:** Proposed, pending spike `S-06`
- **Date:** 2026-08-16
- **Supersedes:** [ADR-0009](0009-unofficial-access-boundary.md)
- **Related:** [tiktok-integration.md](../tiktok-integration.md), [ADR-0007](0007-seeded-adapter-over-mock-server.md), [architecture.md](../architecture.md)

## Context

Spike `S-01` established a hard fact: the official TikTok API has exactly three video scopes (`video.list`,
`video.publish`, `video.upload`) and therefore **cannot delete, unpublish, re-privatise or edit an existing
video**. It also cannot read a video's privacy, and `video.list` returns public videos only.

[ADR-0009](0009-unofficial-access-boundary.md) concluded from this that the feature was out of reach. That
conclusion was wrong, because it never examined how the working tools actually work.

They work, and the mechanism is not exotic:

| Tool | Form factor | Mechanism | Evidence |
| --- | --- | --- | --- |
| SocialEraser | Chrome extension, MIT licensed | **DOM automation of TikTok Studio.** Click row menu, click delete, click confirm. ~0.9 s per item | [Source](https://github.com/socialeraser/SocialEraser), `tiktok-automation.js` |
| DeleteTik | Chrome extension | Runs locally in the user's session, "no password required" | Product page |
| Redact | **Desktop app**, 1M+ users | User signs in to each platform **inside the app**, all processing on device, login never touches a Redact server | Product page |

Redact is the important one. It is not an extension. It is a locally installed application that embeds a login
and then acts as the signed in user. That is a form factor we can match exactly, because our product is
**self hosted via Docker Compose on the user's own machine**.

There is a second, larger consequence that ADR-0009 missed entirely. A logged in session does not just unlock
deletion. It unlocks **reading**:

| Problem with the official API | Under a browser session |
| --- | --- |
| `video.list` returns public videos only | The full library, including private and unlisted |
| Privacy of an existing video is unreadable | Visible in TikTok Studio |
| App audit required before a public audience | Not applicable |
| Sandbox restrictions during development | Not applicable |
| Videos "disappear" from sync when made private | No disappearance problem |

The app audit was risk R-2, the longest lead time item in the entire plan and a blocker for Phases 3 to 5.
A browser session removes it from the critical path.

## Options considered

**A. Official API only.** The ADR-0009 decision. Delete and privacy become manual checklists, reads are
limited to public videos, and the whole plan waits on app audit. Safe, and substantially less useful. Rejected,
because it discards the product's headline capability to satisfy a constraint nobody set.

**B. Companion browser extension.** What the existing tools do. Already declined by the product owner: this is
a web application, not an extension, and a second distributable with store review is a different product.

**C. Server side session replay.** Store the user's TikTok cookies centrally and replay them from a hosted
backend. Requires defeating request signing (`X-Bogus`, `X-Gnarly`, `msToken`, `_signature`, all JSVMP
obfuscated), and in a hosted deployment it concentrates full account credentials for every user. Still
rejected, for both reasons.

**D. Browser session automation inside the self hosted deployment.** A Playwright driven Chromium in the
`worker` container with a persistent profile volume. The user logs in once through a live view of that browser.
The session, the profile and the traffic never leave their machine. Actions are performed the way the tools
above perform them, at human pace.
Pros: full capability, including everything the API cannot do. Reads the complete library. No app audit, no
sandbox. Signing is a non issue, because TikTok's own page does it. Matches the Redact model that a million
people already use. Fits the self hosted architecture we already committed to.
Cons: DOM automation is inherently fragile and breaks when TikTok ships UI changes. Needs captcha and
verification handling. Heavier Compose stack (a browser image is not small). Slower than an API, at roughly
one action per second. Sits outside TikTok's terms of service, with the account risk that implies.

## Decision (proposed)

**Option D becomes the primary execution strategy, with the official API kept as a legitimate second
strategy where it is genuinely better.**

Three execution strategies behind the one `TikTokVideoPort` from [ADR-0007](0007-seeded-adapter-over-mock-server.md):

| Strategy | Used for | Notes |
| --- | --- | --- |
| `SEEDED` | Development, CI, every test, demo mode | Unchanged. Still the default in CI |
| `BROWSER` | Library sync, delete, privacy change, caption edit | Primary. Full capability |
| `API` | Upload and publish, optionally | Genuinely better for large file upload, and legitimate. Requires audit, so it stays optional |

The port and the capability matrix already exist for exactly this reason, so this is an adapter addition, not
a redesign. `capabilities()` becomes a function of the configured strategy.

Design requirements, taken directly from what the reference implementation had to solve:

1. **Selectors are configuration, never hardcoded.** SocialEraser's own project rule is "all selectors go
   through config, never written inline", and they ship selector updates from a CDN without reinstalling.
   We hold selectors in a versioned, updatable config file with an ordered fallback list per target.
2. **A canary check before every run.** Verify the expected selectors still resolve. If they do not, fail the
   job cleanly with `SELECTORS_STALE` rather than clicking something unintended. Never guess on a page we do
   not recognise.
3. **Human pacing, with jitter.** Roughly one action per second, configurable, never parallel within an
   account.
4. **Overlay defence.** Dismiss toasts and modals before every click, because the "Deleted successfully"
   notification lands on top of the next row's button. This is a real, documented failure in the reference
   implementation.
5. **Stuck detection.** A progress watchdog that aborts the run rather than looping, matching their 30 second
   timeout.
6. **Attention state.** Captcha, a login challenge or an unrecognised page moves the job to
   `AWAITING_ATTENTION` and surfaces a live view of the browser so the user can resolve it and resume.
7. **Resumable.** Per item state persisted, so a restart continues rather than repeats. Already true of our
   batch job model.
8. **Archive before destruction.** Prompt for TikTok's official "Download your data" archive before a first
   bulk delete. Deletion is irreversible within seconds.
9. **Session confined to the machine.** Profile stored in a named volume, encrypted at rest, never transmitted,
   never logged, never exposed through the API.

## Consequences

**Easier:** the product actually does what it promises. Batch delete, batch privacy and caption editing all
become real. The dashboard shows the complete library rather than the public subset. App audit leaves the
critical path, which removes the biggest schedule risk in the plan. Phases 3 and 4 no longer depend on an
external approval process.

**Harder:** we own a fragile integration. TikTok ships a UI change and our delete path breaks, so the selector
config, the canary check and a fast update path are load bearing, not nice to have. The Compose stack gains a
browser image. Testing needs a recorded fixture of TikTok Studio's DOM, refreshed periodically.

**Accepted risks, explicitly:**

- **Terms of service.** Automating the interface with the user's own session, against their own account and
  their own content, is very likely contrary to TikTok's terms. The product owner has accepted this. The
  product must state it plainly to the user before first use rather than bury it.
- **Account risk.** Human pacing and rate caps reduce it. They do not eliminate it. The tools doing this at
  scale carry the same caveat.
- **Breakage risk.** Assume it breaks a few times a year. The manual checklist from
  [features/batch-actions.md](../features/batch-actions.md) survives as the degradation path when the browser
  strategy is unavailable, which is exactly the fallback that work already bought us.

**Revisit if:** TikTok ships an official delete or privacy endpoint, in which case the `API` strategy takes
those capabilities immediately and the browser strategy narrows to reads. That switch is one line of
configuration, which is the entire point of the port.
