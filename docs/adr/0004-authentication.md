# ADR-0004: Self hosted sessions plus TikTok OAuth for account linking

- **Status:** Proposed
- **Date:** 2026-08-16
- **Related:** [security.md](../security.md), Phase 2 in [TODO.md](../TODO.md)

## Context

Two separate concerns are easy to conflate:

1. **Who is the user of this application?** Needed before any TikTok account exists, because a user may sign up,
   look around and connect an account later, or connect several.
2. **Which TikTok accounts may this application act on behalf of?** That is OAuth, and it is a linking concern,
   not an identity concern.

The application is self hostable, so an external identity provider cannot be a hard requirement.

## Options considered

**A. Self hosted email and password identity, plus TikTok OAuth purely for linking.**
Pros: works with zero external dependencies, supports multiple linked accounts per user cleanly, the user keeps
access to the app even when a TikTok token is revoked, and account deletion is entirely under our control.
Cons: we own password storage, reset flows, rate limiting and lockout, which is real security surface.

**B. TikTok OAuth as the only login.**
Pros: no passwords to store, one step onboarding.
Cons: no user record without a TikTok connection, multiple accounts per user becomes awkward, a revoked token
locks the user out of their own organisational data, and account recovery depends entirely on a third party.

**C. Keycloak or a hosted identity provider.**
Pros: battle tested, SSO, MFA and admin UI for free.
Cons: a heavy extra service in the Compose stack for a product whose version 1 has no teams and no roles.
Contradicts the "easy to run locally" principle.

**D. An auth library such as Auth.js or Better Auth.**
Pros: less code than option A, sensible defaults.
Cons: fits a frontend owned session model, but our sessions must be validated by the API and the worker.
Worth reconsidering during implementation if it fits the backend cleanly.

## Decision (proposed)

Option A. The application owns identity, with:

- Argon2id password hashing, a length and breach based password policy.
- Short lived access token plus a rotating refresh token in an `HttpOnly`, `Secure`, `SameSite=Lax` cookie.
- Refresh reuse detection that revokes the whole token family.
- Rate limiting and lockout on every credential endpoint.
- TikTok OAuth with PKCE, used **only** to link an account and obtain tokens for acting on it.

Full requirements in [security.md](../security.md).

## Consequences

**Easier:** the app is usable before any TikTok connection exists. Multiple linked accounts is a natural model.
A revoked TikTok token degrades one connection rather than locking the user out. No extra service in Compose.

**Harder:** we own the security surface of password authentication, including reset, enumeration resistance and
lockout. That surface is explicitly covered in the Phase 2 gate.

**Accepted costs:** more code than delegating login entirely.

**Revisit if:** teams and roles enter scope, at which point an external identity provider becomes worth its
weight. MFA is a likely follow up regardless, and is deliberately not in version 1.
