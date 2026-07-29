# Refresh Token Support

> **STATUS**: Implemented

This document specifies refresh tokens for Easy OIDC and the implementation plan.

## Versioning

This design targets Easy OIDC 2.0 on a fresh instance. Startup provisions a new local
SQLite database. No v1 upgrade path, schema migration, backfill, compatibility adapter,
or legacy token support is required.

## Goals

- Let interactive public clients renew short-lived tokens without repeating login.
- Support ordinary login continuity and explicitly authorized offline access.
- Detect refresh-token replay and provide application and user-initiated revocation.
- Revalidate connector-backed identities and authorization with the upstream provider
  before every downstream refresh.
- Fail closed when local state is unavailable or lost.
- Keep the storage model compatible with a future shared database and DPoP.

Client credentials, machine-to-machine grants, DPoP, and mTLS are out of scope.

## Token Policies

Refresh tokens are disabled for a client unless explicitly configured.

### Session refresh

A refresh-enabled client receives a refresh token from the authorization-code
exchange without requesting an additional scope. This is the default mode for
kubelogin, CLIs, mobile applications, and backend-for-frontend applications.

- Idle lifetime: **30 minutes**
- Absolute lifetime: **10 hours** from interactive authentication
- No Easy OIDC consent screen beyond login; the upstream provider may require consent
- Application logout revokes the grant through the revocation endpoint

Easy OIDC has no persistent browser login session. “Session” therefore means the
per-client refresh grant identified by `sid`; Easy OIDC cannot observe an
application's local logout unless the application calls the revocation endpoint.

### Offline refresh

Offline refresh is issued only when all of the following are true:

- The client is explicitly allowed to request offline access.
- The authorization request includes the `offline_access` scope.
- The user gives explicit consent during that authorization flow.

Consent is required for every new offline grant. An offline grant can outlive an
application login or unattended process, but explicit revocation always ends it.

- Idle lifetime: **30 days**
- Absolute lifetime: **90 days** from interactive authentication

All lifetimes are secure defaults and are configurable per client. Idle expiry
moves after each successful rotation; absolute expiry never moves. Configuration
must reject non-positive values and idle lifetimes greater than absolute lifetimes.
For connector-backed grants, absolute expiry is the earliest of the configured limit,
any known upstream refresh-token expiry, and when no refresh token exists, any known
upstream access-token expiry. With a renewable refresh token, access-token expiry caps
new access/ID tokens but does not shorten the grant's absolute expiry.

Access and ID tokens default to **15 minutes**, configured independently with
`access_token_ttl` and `id_token_ttl`. Both refresh modes use the same rotation,
replay detection, client binding, and revocation mechanisms.

All v2 configuration durations use positive strings accepted by Go's
`time.ParseDuration`; examples include `"5m"`, `"1h30m"`, and `"720h"`. The `d`
unit is not supported. Reject empty, zero, negative, invalid, and overflow values.
Omitted values select documented defaults.

Client configuration uses this shape; omitted lifetime fields select the defaults:

```json
"refresh_tokens": {
  "enabled": true,
  "allow_offline_access": false,
  "session_idle_ttl": "30m",
  "session_absolute_ttl": "10h",
  "offline_idle_ttl": "720h",
  "offline_absolute_ttl": "2160h"
}
```

## Protocol Requirements

### Authorization

- Support only Authorization Code with PKCE S256 for initial grants.
- Validate `response_type=code`, requested scopes, and the required `openid` scope.
- Preserve the authorized scopes, client, authentication time, identity, and refresh
  policy through state and the authorization code.
- Preserve the connector ID, immutable upstream subject, and upstream OAuth credential
  server-side through identity selection, OTP/consent, and code redemption. Browser
  state and code values contain only opaque references. Encrypt every temporary SQLite
  copy, authenticating its flow/code identifier, client ID, and connector ID; re-encrypt
  it for the final `sid`. When moving between temporary records, atomically create the
  successor and delete the predecessor. Code redemption deletes all temporary copies
  in the transaction that creates the grant. Direct-email grants have no credential.
- Generate a cryptographically random `sid` for each authorization-code exchange
  eligible to receive a refresh token. Refresh-disabled exchanges need not use one.
- For a validated client and redirect URI, reject unknown or unauthorized scopes,
  including `offline_access` for a non-allowlisted client, by redirecting with
  `error=invalid_scope` and the original `state`. Never redirect to an unvalidated URI.
- Before issuing an offline authorization code, display explicit consent. Denial
  redirects with `error=access_denied` and the original `state`. If `prompt=none`
  prevents required consent, redirect with `error=consent_required`.

### Token issuance

- Add `refresh_token` to the token endpoint's supported grant types.
- Accept only `POST` with an `application/x-www-form-urlencoded` body of at most 16 KiB.
  Reject OAuth parameters in the URL query and reject duplicate recognized parameters.
  Authorization-code requests require exactly one `grant_type=authorization_code`,
  `code`, `client_id`, exact `redirect_uri`, and `code_verifier`. Refresh requests
  require exactly one `grant_type=refresh_token`, `refresh_token`, and `client_id`;
  `scope` is optional.
- In one SQLite transaction, load the authorization code; verify its expiry,
  `client_id`, exact `redirect_uri`, and PKCE verifier; conditionally consume it
  exactly once; recheck that the client and connector still exist, refresh remains
  enabled, and offline access remains allowed when selected; and create the initial
  grant/token. Mode and consent come from the code, while lifetimes are snapshotted
  from current configuration. Failure returns `invalid_grant`. Do not implement this
  as a read followed by a separate consume operation. Generate and sign the response
  tokens before the transaction so signing failure does not consume the code.
- Public-client refresh requests must include the original `client_id`.
- The grant stores the originally authorized scopes. If `scope` is omitted during
  refresh, use that full set. If present, it must be a subset and narrows only the
  new access token; it does not mutate the grant or replacement refresh token. The
  response includes `scope` when narrowed, and the access JWT contains the effective
  space-delimited `scope` claim.
- Every successful refresh returns a fresh ID token, access token, and replacement
  refresh token. An eligible code exchange returns an initial refresh token; a
  refresh-disabled exchange does not.
- Refreshed ID tokens do not repeat the original `nonce`.
- Tokens issued under a refresh grant contain its stable `sid`; refresh-disabled
  exchanges need not contain `sid`.
- On refresh, require that the client still exists and remains refresh-enabled; an
  offline grant additionally requires offline access to remain allowed. Otherwise,
  revoke the grant and return `invalid_grant`. Recompute groups from the stored email
  using current client policy. Lifetimes are snapshotted at grant creation, so later
  lifetime configuration changes are not retroactive.
- Before issuing tokens for a connector-backed grant, validate the upstream credential
  and retrieve current identity as specified below. Direct-email grants perform
  only the local client and group-policy checks.
- Token responses use OAuth JSON errors, `Cache-Control: no-store`, and
  `Pragma: no-cache`.
- Successful responses include `token_type=Bearer`, the tokens applicable to the grant,
  and `expires_in` equal to the new access token's actual capped lifetime in seconds.

Issue distinct signed ID and access JWTs. ID tokens contain OIDC identity claims,
`auth_time`, and `nonce` only on the initial exchange. Access tokens contain the
effective `scope` claim. Both contain issuer, subject, audience, issuance time, and
their independently configured expiry; tokens issued under a refresh grant also
contain its stable `sid`. Every JWT has its own cryptographically random `jti` to support
future per-token status and introspection without changing the token format.

### Upstream revalidation

Persist the upstream OAuth credential with each connector-backed grant. On every
downstream refresh:

1. In one transaction, treat a retained consumed token as replay; otherwise acquire a
   claim only if its complete-token hash verifies, it is unconsumed, its grant is active
   and unexpired, and no live claim exists. Never overwrite a live claim. Concurrent
   requests wait briefly, then reread for replay or return `temporarily_unavailable`.
   Reclaim an expired claim only if upstream refresh had not started; otherwise mark
   the grant unusable. Assign a random claim ID and expiry. Before calling the upstream
   token endpoint, durably mark that upstream refresh has started.
2. Derive the key from the verified token's decoded 32-byte secret and decrypt the
   upstream credential. If its access token is expired or near expiry, refresh it first.
   Do not refresh an otherwise valid upstream access token solely because a downstream
   refresh was requested.
3. Call the connector's authenticated identity/userinfo operation with the valid
   upstream access token. If it returns `401` or `invalid_token`, refresh once when
   possible and retry identity retrieval once.
4. Require the same connector ID and immutable upstream subject, and require the
   grant's exact normalized email to remain among current provider email assertions;
   never switch to another email. Re-run current provider/local email-verification,
   connector-specific authorization, and local group/client policy. For Google, a
   configured hosted domain must exactly match the returned `hd` claim; the request
   hint or email domain alone is insufficient.
5. In one transaction conditional on the same live claim, unconsumed downstream token,
   and active/unexpired grant, generate the replacement downstream token, encrypt the
   complete updated upstream credential under its new client-held secret, preserve the
   previous upstream refresh token if a successful response omits it, consume and
   replace the downstream token, tighten expiries, and release the claim.

Upstream calls use hard deadlines that leave time to commit before claim expiry. Claim
expiry fences SQLite writes but cannot prove a rotating provider request did not run.
If a claim expires after upstream refresh started, the stored refresh token is
indeterminate: do not reuse it; require re-login.

Cap each new downstream access/ID token expiry to the remaining upstream access-token
lifetime when known. Cap the grant's absolute expiry to a known upstream refresh-token
expiry. Providers do not expose refresh-token expiry consistently; when unknown, the
configured Easy OIDC absolute limit remains authoritative. If an expiring upstream
access token has no refresh token, the grant cannot refresh after that access token
expires. A non-expiring upstream access token may remain usable until revoked or the
Easy OIDC grant expires, but userinfo is still called on every downstream refresh.
A later provider response may only tighten the grant's absolute expiry, never extend
it. Missing access-token expiry is unknown, not proof that it is non-expiring; treat it
as such only when guaranteed by the connector's provider contract.

Use these expiry rules, with `now >= expiry` treated as expired:

- `grant_absolute_expiry` is authentication time plus configured absolute TTL, tightened
  by applicable known upstream expiry.
- Initial and rotated refresh-token expiry is
  `min(now + idle_ttl, grant_absolute_expiry)`.
- Access/ID expiry is `min(now + configured token TTL, refresh-token expiry,
  grant_absolute_expiry, known upstream access-token expiry)`; omit inapplicable terms.

If a verified token cannot authenticate or decrypt its credential ciphertext,
atomically mark the grant unusable and return `invalid_grant`; storage failure returns
HTTP 503. Never call upstream or release that claim for retry. If replacement-token
generation or encryption fails before possible upstream rotation, release the claim
and return `temporarily_unavailable`; afterward, apply the indeterminate-credential
rule and require re-login.

Classify upstream failures:

- Upstream `invalid_grant`, repeated `401`/`invalid_token` after retry, explicitly
  disabled/denied accounts, unacceptable identity, or identity mismatch revoke the
  grant and return `invalid_grant`. Return it only after revocation commits; storage
  failure returns HTTP 503.
- When no upstream rotation could have occurred, transport errors, timeout, HTTP
  429/5xx, provider rate-limit responses (including GitHub 403), malformed responses,
  and provider/client configuration failures such as `invalid_client` release the
  claim and return HTTP 503 with OAuth JSON `error=temporarily_unavailable`. Include
  `Retry-After` when known.
- If upstream rotation may have succeeded but its response or commit is lost through
  timeout, cancellation, crash, or claim loss, the credential is indeterminate and
  requires re-login. No transaction can span SQLite and the upstream provider.

### Generic connector renewable credentials

Generic connectors optionally configure renewable credentials as follows:

```json
"refresh": {
  "scopes": ["offline_access"],
  "authorization_params": {"prompt": "consent"}
}
```

Apply this configuration whenever creating either a session or offline downstream
grant. Merge `scopes` as a set with the connector's normal scopes. Reject authorization
parameters owned by Easy OIDC: `client_id`, `redirect_uri`, `response_type`, `scope`,
`state`, `nonce`, `code_challenge`, and `code_challenge_method`. Easy OIDC's fixed
parameters always win; configuration must never silently override them.

### Rotation and replay detection

- Refresh tokens use the canonical URL-safe form `ert1.<handle>.<secret>`: exactly
  three dot-separated components, unpadded base64url encoding, a 16-byte decoded
  handle, and a 32-byte decoded secret. Reject non-canonical encodings and unsupported
  versions as `invalid_grant`. The encoded token is approximately 71 characters and
  remains comfortably within browser cookie limits.
- Define `handle_hash` as SHA-256 of the decoded handle and `token_hash` as SHA-256 of
  the complete canonical ASCII token. Make `handle_hash` unique and retry generation
  on conflict. Lookup by handle hash, then verify the complete-token hash before claim
  or decryption. Never store or log either raw component.
- A refresh token is single-use and rotates on every successful exchange.
- In one database transaction, conditionally consume the presented unconsumed token,
  create its replacement, and update the grant's last-used and idle-expiry timestamps.
- Exactly one concurrent refresh may succeed. After processing completes, presentation
  of the consumed token must reread committed state, revoke the family as replay, and
  return `invalid_grant`, not a generic lock error.
- There is no retry grace period. Clients must serialize refresh attempts.
- Reuse of a consumed token marks the whole grant compromised and revokes every
  token in that family. Return `invalid_grant` and emit a security log event.
- Retain consumed hashes until the grant's absolute expiry so replay remains
  detectable.

### Revocation and logout

Implement RFC 7009 at `/revoke`:

- Accept only `POST` with an `application/x-www-form-urlencoded` body. `token` and
  the public client's registered `client_id` are required; `token_type_hint` is optional.
- Accept refresh, access, and ID tokens. For JWTs, verify signature, issuer, expiry,
  audience against the supplied client ID, and the presence of `sid` before acting.
- Revoke the entire grant when an active or retained consumed refresh-token hash belongs
  to the supplied client ID, or when a valid access/ID token identifies that client's
  grant through `sid`. Revocation is family-wide, not limited to one token.
- After an authoritative lookup, unknown, expired, already-revoked, or wrong-client
  tokens receive an empty HTTP 200 response and change no state. Malformed requests
  receive an OAuth JSON error. Storage failures receive HTTP 503 and must never be
  reported as successful revocation.
- An application logout calls `/revoke` and deletes its local tokens even if the
  request fails.

Revocation prevents future renewal. Existing stateless access/ID JWTs remain valid
until `exp`. The initial release exposes no OP-side `sid` status or event feed, so
resource servers cannot immediately observe replay or self-service revocation. An
application makes its own logout immediate by revoking the same `sid` in its database
before calling `/revoke`, as described in `docs/app-integration.md`.

Self-service grant management is part of the initial release. `GET /grants` starts a
dedicated, non-token-issuing authentication flow and creates no refresh grant. The
upstream provider may reuse its browser session; credential re-entry is not guaranteed.
After identity verification, render active grants with `POST` revoke forms. Each form
contains a random action token in a hidden field, never a URL. Store only its hash,
bind it to the exact normalized email, `sid`, and revoke action, expire it after five
minutes, and consume it in the same transaction that revokes the grant. This provides
CSRF protection without adding a persistent Easy OIDC session or cookie.

### Discovery and client compatibility

- Advertise `authorization_code` and `refresh_token` in
  `grant_types_supported`.
- Advertise `offline_access` in `scopes_supported` and publish the
  `revocation_endpoint`.
- kubelogin works with session refresh using its existing default `openid` scope.
- Document `--oidc-extra-scope=offline_access` for explicitly allowed kubelogin
  clients that need offline grants.

## State and Storage

SQLite is authoritative refresh state, not a cache. Losing or recreating it invalidates
all refresh grants and forces interactive authentication. An unknown token returns
`invalid_grant`. A future shared PostgreSQL store can use the same records and
transaction semantics.

Use SQLite WAL mode with `synchronous=FULL`, bounded busy handling, and database/WAL
files on durable local storage. Commit the upstream-refresh-started marker before the
provider call. After upstream validation, generate the replacement refresh-token
material, encrypt the updated credential, and sign both JWTs before the final conditional
rotation commit. Signing, RNG, or encryption failure follows the temporary/indeterminate
rules above. Once committed, response-write failure or response loss leaves the old
token consumed; retrying it triggers strict replay revocation and requires re-login.

Add three logical tables:

### `refresh_grants`

- `sid` primary key
- client ID, normalized email, email-verification state, and authorized scopes
- connector ID and immutable upstream subject for connector-backed grants
- AES-GCM-encrypted upstream access token, token type, and refresh token, plus
  normalized absolute access- and refresh-token expiry metadata
- mode (`session` or `offline`) and authentication/creation/last-used timestamps
- idle and absolute expiry timestamps
- revoked/compromised timestamp and reason
- refresh processing claim identifier, expiry, and upstream-refresh-started marker
- nullable `dpop_jkt` reserved for future sender constraint

### `refresh_tokens`

- unique handle hash for indexed lookup and complete-token hash for verification
- grant `sid`
- issued and expiry timestamps
- consumed timestamp
- replacement-token hash or identifier

### `grant_actions`

- action-token hash primary key
- normalized email, grant `sid`, and action
- creation and expiry timestamps

Foreign keys and indexes must support lookup by token hash, listing grants by user,
and expiry cleanup. Rotation and replay revocation must be atomic. Cleanup may delete
a grant and its token history only after absolute expiry.
Enable SQLite foreign-key enforcement on every connection and use cascading deletion
for token/action children. Configure bounded busy handling and test real concurrency.

Never embed upstream credentials in downstream refresh tokens. For established grants,
derive 32 bytes with HKDF-SHA256 using the decoded client-held secret as input keying
material, empty salt, and `easy-oidc upstream credential v1` as `info`. Encrypt the
complete credential with AES-256-GCM and a fresh random 96-bit nonce stored with the
ciphertext. Encode AAD (`sid`, client ID, and connector ID) using an unambiguous,
versioned, length-delimited format. Every encryption, including temporary copies, uses
a fresh nonce. Rotation generates a new handle/secret and re-encrypts in one transaction.

The configured encryption key cannot decrypt an established grant's current
ciphertext; that requires the client-held secret. Temporary pre-grant copies remain
recoverable by an attacker with the configured key and SQLite material containing a
copy, including retained pages, WAL files, or backups. Logical deletion limits but
cannot guarantee forensic erasure of that temporary exposure. Changing the configured
key invalidates in-flight flows but not established grants. Control of the live process
can capture presented tokens and requires stronger host isolation; future DPoP addresses
a different client-token theft threat.

Every refresh attempt logs its result, client ID, remote IP, user agent, and `sid`
when the token is known. Never log raw tokens or token hashes. Replay uses a distinct
warning/security event; the initial implementation need not introduce an external
alerting subsystem.

## Future DPoP Compatibility

DPoP is not implemented now. The grant model reserves `dpop_jkt`; token issuance is
centralized around a grant context. Future DPoP will bind the grant to a JWK
thumbprint, require matching proofs during refresh, and add `cnf.jkt` to issued
tokens without changing refresh-token lineage or grant-revocation records. It may add
separate short-lived proof-`jti` replay and server-nonce state; authorization-code
binding will also preserve `dpop_jkt` through state and the code.

## Implementation Plan

1. **Configuration and validation**
   - Add per-client refresh enablement, offline-access allowlisting, and optional
     session/offline lifetime overrides to config types and the JSON schema.
   - Add one presence-aware duration-string type backed by `time.ParseDuration` and
     use it consistently for all v2 duration configuration, including access, ID,
     refresh, and email OTP lifetimes; remove numeric `*_seconds` configuration fields.
   - Make offline access valid only when refresh is enabled. Use presence-aware fields
     so omission selects defaults; validate all effective lifetime combinations.
   - Add separate `access_token_ttl` and `id_token_ttl` settings defaulting to `15m`.

2. **Authorization correctness and grant context**
   - Validate `response_type` and downstream scopes in `/authorize`.
   - Persist scopes and refresh mode through OAuth state and authorization codes.
   - Bind code redemption to `client_id` and `redirect_uri`, and validate all request
     data before atomically consuming the code.
   - Add offline consent templates and denial handling.

3. **Upstream connector contract**
   - Extend connectors to refresh expiring OAuth credentials, return updated tokens
     and expiry metadata, retrieve current identity, and classify terminal versus
     temporary failures.
   - Make renewable-credential acquisition connector-specific. Google uses
     `access_type=offline` and `prompt=consent` when creating a new downstream refresh
     grant. Generic connectors use the specified refresh scopes/parameters for both
     downstream modes. GitHub credentials are non-refreshable unless the response
     supplies a refresh token. Never treat Google's `access_type=offline` as generic.
   - Require the upstream-credential encryption key whenever any refresh-enabled
     client can authenticate through a non-email connector.

4. **Storage**
   - Add refresh-grant, refresh-token, and short-lived grant-action records using
     the schema created for a fresh v2 SQLite database. No migration or backfill is
     required.
   - Implement hashed token handles/verifiers, client-held-secret encryption, bounded
     processing claims, create, rotate/re-encrypt, replay-revoke, explicit revoke,
     list, and cleanup operations with focused transaction/concurrency tests.
   - Configure WAL durability and test process termination/restart before and after the
     refresh-started marker, upstream response, final commit, and response write.

5. **Issuance and refresh grant**
   - Generate `sid`, then atomically consume the code and create the initial grant and
     split-secret token with its re-encrypted upstream credential during eligible code
     exchanges.
   - Refactor signing around one grant/session context; add `sid` and `auth_time`.
   - Add refresh request validation, upstream refresh/userinfo revalidation, local
     policy re-evaluation, lifetime caps, atomic rotation, narrowed scopes, fresh
     tokens, OAuth errors, and no-store headers.

6. **Revocation and self-service management**
   - Add `/revoke` family revocation by refresh, access, or ID token.
   - Add dedicated-authentication `/grants` listing and one-time revoke actions.
   - Add structured refresh, replay, and revocation audit logs.

7. **Metadata, documentation, and verification**
   - Update discovery, examples, `docs/app-integration.md`, kubelogin documentation,
     and Kubernetes guidance.
   - Test expiry boundaries, concurrent refresh, replay, client/scope binding,
     upstream rotation and failures, identity changes, policy changes, offline consent,
     revocation by every token type, encryption-key loss, split-secret database theft,
     database loss, and cleanup.
   - Run `make check` and an end-to-end kubelogin refresh test.
