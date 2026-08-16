# ADR-0009: How far we go to support delete and privacy changes

- **Status:** Accepted
- **Date:** 2026-08-16
- **Related:** [tiktok-integration.md](../tiktok-integration.md), [features/batch-actions.md](../features/batch-actions.md), spike `S-01`

## Context

Batch deletion and batch privacy changes are headline features. The official TikTok API very likely does not
offer them to third party applications, which is what spike `S-01` is confirming.

Yet tools that do exactly this demonstrably exist. Redact and several browser extensions perform bulk deletion
of a user's own TikTok videos, filtered by date range and other criteria. So the capability is reachable, just
not through the documented third party API.

Understanding **how** they reach it is the whole decision. These tools run **inside the user's own logged in
browser session** and drive the same internal web endpoints the TikTok web application itself calls, or they
automate the interface directly. They are not using a public, documented, supported API. That means:

- The endpoints are undocumented and can change without notice or deprecation period.
- Behaviour may be indistinguishable from automated abuse from TikTok's side, whatever the user's intent.
- It sits outside, and plausibly against, the platform's terms of service.
- If done from a server rather than the user's browser, it requires holding the user's **web session
  credentials**, which is a categorically different and much larger security liability than an OAuth token.

This is a product and risk decision, not a technical one, so it is written down rather than decided by whoever
implements the feature first.

## Options considered

**A. Official API only. Anything unsupported becomes a tracked manual checklist.**
Pros: zero terms of service risk, zero account risk for users, nothing breaks when TikTok changes an internal
endpoint, no session credentials anywhere near our servers. Fully defensible if the app is ever audited.
Cons: the headline promise degrades to "we find and organise the videos, you tap delete". Weaker than the tools
the user has already seen working.

**B. An optional browser extension companion, acting only in the user's own session.**
Pros: this is what the existing tools do, and it is the least bad way to reach the capability. Credentials never
leave the user's browser and our server never holds a session.
Cons: a second artefact to build, review and distribute, plus store review processes. Fragile against TikTok
changes. Still a terms of service question. Considerable extra scope for a web application project.
**Declined by the product owner.** The existing extensions were cited as evidence that the capability is
reachable, not as a template for what we should build. We are building a web application, not an extension.

**C. Server side session replay. Store the user's TikTok web cookies and call internal endpoints from our
backend.**
Pros: it would work without any extra artefact.
Cons: we would hold credentials that grant full account access, far beyond OAuth scopes, for every user. A
breach would be catastrophic and unbounded. Traffic originates from our servers, so a whole deployment can be
blocked or flagged at once. Terms of service exposure sits with us rather than with the user. **This is
rejected**, not merely deprioritised.

**D. Local automation in the self hosted deployment. Drive a headless browser on the user's own machine.**
Pros: it stays on the user's hardware, matching the self hosted model.
Cons: fragile, heavyweight in the Compose stack, needs the user's login inside a container, and it is still
option C's credential problem wearing a different hat.

## Decision

1. **We ship option A.** The official API for everything it supports, and a tracked manual action list for
   everything it does not. The UI states plainly why a step is manual, rather than pretending.
2. **Option C is rejected outright**, permanently. We never hold a user's TikTok web session credentials.
   Option D is rejected for the same underlying reason.
3. **Option B is declined.** This project is a web application. A companion extension is a different product
   with its own distribution, review and maintenance burden, and it would still carry the terms of service
   question. Reopening it requires a new ADR superseding this one.
4. **The architecture stays neutral.** Actions go through the capability matrix and the batch job machinery,
   so an additional execution strategy could be added later next to `API` and `MANUAL` without touching the
   domain, the UI or the audit trail. That is good design, not a plan.

## Consequences

**Easier:** Phase 4 is unblocked regardless of what `S-01` finds, since the manual fallback is fully specified.
No legal or security exposure, and no second artefact to maintain. All effort goes into the genuinely
differentiating work (library, taxonomy, filtering, selection, automation, audit), none of which depends on
this decision.

**Harder:** the product is less capable than a single purpose extension for the narrow task of mass deletion.
That is a real gap and the product copy should be honest about it rather than hide it.

**Accepted costs:** we lose a headline feature if `S-01` comes back negative. We gain a product that cannot get
a user's account restricted, which is worth more than the feature.

**Revisit when:** `S-01` completes and tells us how much of the gap is actually real.
