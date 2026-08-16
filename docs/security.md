# Security

This application holds OAuth tokens for people's social media accounts and can publish or destroy content on
their behalf. Treat every token as a credential of the highest sensitivity.

## Threat model, briefly

| Asset | Threat | Consequence |
| --- | --- | --- |
| TikTok access and refresh tokens | Database dump, log leak, SSRF, insider access | Attacker posts or deletes as the user |
| Session cookies | XSS, CSRF, network interception | Full account takeover |
| User content and taxonomy | Broken authorization | Data leak between users |
| Batch and automation execution | Confused deputy, unauthenticated trigger | Mass deletion or unwanted posting |
| The publish pipeline | Malicious upload, SSRF through a supplied URL | Server compromise, abuse of our app credentials |

## Authentication

- **Password hashing:** Argon2id, parameters from the environment, tuned so a hash takes roughly 250 ms on the
  target hardware. Never MD5, SHA family or bcrypt with a low cost.
- **Password policy:** minimum 12 characters, checked against a breached password list. No composition rules,
  no forced rotation. Length and breach checking beat symbol theatre.
- **Sessions:** short lived access token (15 minutes) plus a rotating refresh token (30 days) in a cookie that
  is `HttpOnly`, `Secure`, `SameSite=Lax`, host scoped, with `Path=/`.
- **Refresh rotation with reuse detection.** A replayed refresh token revokes the entire token family and
  forces re-authentication. This is what limits the damage of a stolen token.
- **Enumeration resistance:** registration, login and password reset return the same response and take the same
  time whether or not the account exists.
- **Rate limiting and lockout** on login, registration, password reset and OAuth callback, per IP and per
  account. Exponential backoff, not a hard permanent lock.

## Authorization

- Enforced in the **repository layer**, not in controllers. Every query is scoped by the owning user, so a
  forgotten check in a new controller cannot leak data.
- A resource that exists but is not owned by the caller returns `404`, never `403`. Existence is itself
  information.
- Every resource type gets an explicit cross user isolation test. This is a Phase 2 gate item and is not
  optional.
- Job and flow execution runs **as the owning user's context**, never with elevated privileges. A flow cannot
  reach data its owner cannot reach.

## Token handling

This is the part most likely to hurt if we get it wrong.

1. **Encrypted at rest.** Envelope encryption with AES-256-GCM. The data key is derived from
   `TOKEN_ENCRYPTION_KEY`, which comes from the environment and never from the repository.
2. **Never logged.** A redaction filter runs over every log payload for `access_token`, `refresh_token`,
   `client_secret`, `password`, `authorization` and `cookie`. There is a test asserting a token cannot appear
   in log output.
3. **Never returned by the API.** No endpoint exposes a token, not even to its owner, not even partially.
4. **Decrypted only inside the TikTok client**, at the moment of the call, and never held in a long lived
   variable.
5. **Refreshed centrally** with a single flight lock in Redis, so ten concurrent workers cannot race and
   invalidate each other's refresh token.
6. **Revoked on disconnect.** Disconnecting an account calls the upstream revoke endpoint and then deletes the
   local tokens.
7. **Key rotation** is documented in the runbook: re-encrypt with a new key while accepting the old one, then
   retire the old key.

## OAuth

- Authorization code flow with **PKCE**.
- `state` is a random, single use value bound to the session and verified on the callback.
- Redirect URI is an **exact match** against an allow list. No wildcards, no open redirects, no path suffix
  tricks.
- Only the scopes actually needed are requested, and what was **granted** is stored, since the user may grant
  less than was asked for. The UI reflects granted scopes, not requested ones.
- The callback is rate limited and requires an authenticated session, so nobody can link an account onto
  someone else's profile.

## Input handling

- Every request body, query parameter and path parameter is validated by a schema at the edge. Unknown
  properties are rejected, not silently accepted.
- Uploads are validated on **content**, not on filename or client supplied MIME type. Probe the file, check the
  container, codec, duration, resolution and size, then decide.
- Size limits are enforced at the proxy, at the API and at the storage layer. Defence in depth, since any one
  of them can be misconfigured.
- No user supplied URL is ever fetched server side without an allow list check. That is the SSRF door.
- All database access goes through parameterised queries via Prisma. Raw SQL requires review and must never
  interpolate user input.

## Output handling

- Svelte escapes by default. `{@html ...}` is banned outside a reviewed, sanitised rendering helper.
- A strict Content Security Policy: no `unsafe-inline`, no `unsafe-eval`, explicit allow list for image and
  media sources, since covers are served from TikTok CDNs.
- Security headers on every response: `Strict-Transport-Security`, `X-Content-Type-Options: nosniff`,
  `Referrer-Policy: strict-origin-when-cross-origin`, `X-Frame-Options: DENY`, a restrictive
  `Permissions-Policy`.
- Error responses never include a stack trace, a query, an internal path or another user's data.

## Destructive action safety

Batch operations can destroy a lot of work very quickly. Guardrails are a security control, not a nicety.

- **Dry run first.** Every destructive batch supports a preview that performs zero side effects, proven by
  asserting on the audit log.
- **Typed confirmation** for irreversible actions. The user types the count or the word, a single click is not
  enough.
- **Idempotency keys are mandatory**, so a double submit or a retry cannot double execute.
- **A cap on batch size** per job, with anything larger split into explicit jobs the user must approve.
- **A global kill switch** halts all in-flight jobs and flow runs.
- **Everything is audited**: actor, action, target, parameters, outcome, trace id.

## Secrets management

- Secrets come from the environment. Nothing sensitive is ever committed, including in tests and fixtures.
- `.env` is gitignored, `.env.example` holds only placeholders.
- gitleaks runs in CI on every pull request and on the full history.
- Rotation procedure for each secret is in the Phase 6 runbook. A secret with no documented rotation path is an
  incident waiting to happen.
- If a secret is ever committed, rotate it first and rewrite history second. Assume it is compromised the
  moment it is pushed.

## Dependencies

- `pnpm audit` in CI, and the build fails on a high or critical advisory.
- Lockfile committed, dependencies pinned. Renovate or Dependabot proposes updates, a human merges them.
- Adding a dependency requires a note in the pull request on why it beats writing it ourselves. Small, trivial
  packages are usually a liability.

## Privacy and data handling

- We store only what the product needs. No follower data, no other people's content, no comment contents.
- Account deletion removes user data within a documented window, including tokens, uploaded assets and audit
  entries beyond the legal retention requirement.
- Data export gives the user their videos metadata, taxonomy and flows as portable JSON.
- We do not sell, share or train on user data. That is stated in the product, not just here.

## OWASP Top 10 mapping

| Risk | Our control |
| --- | --- |
| A01 Broken access control | Repository level scoping, `404` over `403`, per resource isolation tests |
| A02 Cryptographic failures | Argon2id, AES-256-GCM envelope encryption, TLS everywhere, no plaintext tokens |
| A03 Injection | Parameterised queries, schema validation at the edge, escaped output, CSP |
| A04 Insecure design | Threat model above, dry runs, idempotency, kill switch, batch caps |
| A05 Security misconfiguration | Validated environment schema, security headers, no debug endpoints in production |
| A06 Vulnerable components | Audit in CI, pinned lockfile, human reviewed updates |
| A07 Authentication failures | Rate limiting, lockout, refresh rotation with reuse detection, breach list check |
| A08 Integrity failures | Checksums on upload, signed cursors, pinned base images by digest |
| A09 Logging and monitoring failures | Structured logs with trace ids, append only audit log, alerts on failure rates |
| A10 SSRF | Allow list for any outbound URL, no user supplied fetch targets |

## Reporting a vulnerability

Do not open a public issue. Contact details go in `SECURITY.md` at the repository root when the licence and
policy files land in task `0.1.5`.
