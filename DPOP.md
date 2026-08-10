# DPoP Sender-Constrained Tokens

> **STATUS**: Implemented

> **Dependency:** This design depends on the refresh-grant and distinct-token model in
> [REFRESH.md](./REFRESH.md). Implement and verify REFRESH.md before starting DPoP.
>
> **Normative standards:** RFC 9449, RFC 7638, RFC 9126, and RFC 7009.

This document specifies Demonstrating Proof of Possession (DPoP) for Easy OIDC.
DPoP binds authorization codes, access tokens, and public-client refresh grants to an
ES256/P-256 or ES512/P-521 key held by the client. The implementation owner is
`github.com/easy-oidc/easy-oidc`.

## Goals and Scope

- Implement the RFC 9449 authorization-code, token-endpoint, and protected-resource
  contract with pushed authorization requests (PAR).
- Prevent bearer downgrade and require the same key throughout a DPoP grant.
- Retain ordinary bearer behavior under separate client IDs.
- Enforce short-lived, single-use proofs with shared durable replay storage and no
  nonce challenge.
- Prevent a copied DPoP token from being used to revoke its grant.

Dynamic client registration, token introspection, confidential clients, mTLS, and
DPoP-bound RFC 8693 token exchange are out of scope. DPoP is not client authentication,
does not replace PKCE or TLS, does not sign bodies or general headers, and does not
protect against code executing in the legitimate client context.

## Client Policy

Each client has a `dpop` mode:

```jsonc
"clients": {
  "browser-dpop": {
    "redirect_uris": ["https://app.example.com/callback"],
    "dpop": {
      "mode": "required",
      "signing_algorithm": "ES512"
    },
    "require_par": true,
    "refresh_tokens": { "enabled": true }
  },
  "browser-bearer": {
    "redirect_uris": ["https://app.example.com/callback"],
    "dpop": { "mode": "disabled" }
  }
}
```

Only these values are valid:

- `disabled` (default) issues bearer tokens. The authorization, PAR, token, refresh,
  and revocation endpoints reject `dpop_jkt` or a `DPoP` header with
  `invalid_request`.
- `required` requires a DPoP binding at authorization and a matching proof at code
  redemption, refresh, and revocation. It never issues bearer access or refresh tokens.

A deployment that supports both presentation types uses separate client IDs. There is
no optional or per-request mode selection. Changing a client ID's mode or signing
algorithm is not an in-place migration: operators must create a new client ID and users
must start new authorization flows. Already-issued access tokens remain governed by
their `cnf.jkt` until expiry.

`dpop.signing_algorithm` accepts `ES256` or `ES512` and defaults to `ES256` when the
mode is `required`. ES256 requires an embedded public `EC`/`P-256` JWK; ES512 requires
`EC`/`P-521`. A signing algorithm on a disabled profile is invalid. Key rotation within
a grant is unsupported; loss or replacement of the private key requires a new
authorization flow.

`require_par` defaults to `false`. When true, `/authorize`
accepts only `client_id` and a PAR `request_uri`; it cannot fall back to direct
authorization when PAR is unavailable.

Policy is checked when creating and consuming PAR, issuing the final authorization
code, redeeming a code, refreshing, and revoking. It is not rechecked at every login,
connector, identity-selection, or consent UI hop. Opaque state carries the validated
binding across those hops. Opaque state and authorization codes also carry whether the
flow used PAR, so enabling `require_par` rejects an already-started direct flow at final
code issuance or code redemption.

## Proof Contract

A proof is exactly one compact JWS in exactly one `DPoP` request header. Reject the
request before decoding if the encoded proof exceeds 8 KiB. The protected header is:

```json
{
  "typ": "dpop+jwt",
  "alg": "<configured ES256 or ES512>",
  "jwk": {
    "kty": "EC",
    "crv": "<P-256 for ES256 or P-521 for ES512>",
    "x": "<public x-coordinate>",
    "y": "<public y-coordinate>"
  }
}
```

The implementation must:

1. Require protected `typ` to equal `dpop+jwt`, protected `alg` to equal the client's
   configured algorithm at client-policy boundaries, and an embedded public JWK on the
   corresponding curve. Reject symmetric keys, private key material, another curve or
   algorithm, and invalid or off-curve points.
2. Verify the ES256 or ES512 signature using only that embedded key and derive its RFC 7638
   SHA-256 JWK thumbprint (`jkt`). A reviewed JOSE library is preferred. Its algorithm
   selection must be constrained to the selected supported profile and its verification
   key must be the already constrained embedded JWK. Automatic or remote key discovery through `kid`, `jku`,
   `x5u`, `x5c`, or any similar header is prohibited.
3. Require typed, non-empty string claims `jti`, `htm`, and `htu`, and a numeric `iat`.
   `jti` is limited to 128 UTF-8 bytes and should contain at least 96 random bits.
   `htm` must exactly equal the HTTP method.
4. Accept `iat` no more than ten seconds old and no more than five seconds in the
   future, including the boundary values.
5. Require `htu` to exactly equal the configured trusted public endpoint string for
   the request, excluding query and fragment. Easy OIDC constructs these strings from
   its configured issuer; a resource server explicitly configures its public endpoint.
   Never derive the expected value from untrusted `Host`, `Forwarded`, or
   `X-Forwarded-*` headers. Do not apply general RFC 3986 normalization during proof
   comparison.
6. Reserve the proof's replay hash and reject a uniqueness conflict as replay.
7. Where a credential is bound, require the proof JWK thumbprint to equal its stored or
   asserted `jkt`. Protected-resource proofs additionally require `ath`, equal to the
   unpadded base64url SHA-256 hash of the exact ASCII access-token value.

Required claim types are checked without coercion. Canonical base64 encoding, duplicate
JSON-member rejection, and arbitrary-precision treatment of fractional NumericDate
values are not additional custom profile requirements beyond the selected JOSE
library's behavior and these typed checks. The implementation must not silently accept
a malformed proof, but it need not build a second general-purpose JSON or JOSE parser.

Proofs never establish identity by themselves. Every retry uses a new proof and `jti`.
Easy OIDC does not issue `DPoP-Nonce` or `use_dpop_nonce`, and no endpoint requires a
nonce challenge.

## Authorization and PAR

PKCE S256 remains required. A required-DPoP authorization carries `dpop_jkt`, the
unpadded base64url RFC 7638 SHA-256 thumbprint of the client's public key. It must be a
single, non-empty value decoding to 32 bytes; invalid or duplicate values produce
`invalid_request`. For a validated redirect URI, authorization errors use the normal
OAuth redirect and preserve `state`; errors never redirect to an unvalidated URI.

Easy OIDC implements RFC 9126 PAR at the discovery-advertised
`pushed_authorization_request_endpoint`:

- Accept only `POST` with a strictly bounded `application/x-www-form-urlencoded` body.
  Apply the same client authentication/identification, duplicate parameter, redirect
  URI, response type, scope, and PKCE validation as ordinary authorization.
- Bound aggregate admission before policy or database work. Each Easy OIDC process
  admits at most 100 PAR requests per second with a burst of 200 and returns HTTP 429
  with `temporarily_unavailable` and `Retry-After` when exhausted. A trusted ingress
  should additionally enforce per-source limits appropriate to the deployment.
- Return HTTP 201 with a client-bound, cryptographically unpredictable `request_uri`
  and `expires_in`. Store all parameters server-side for at most 60 seconds and permit
  exactly one consumption.
- A required client supplies `dpop_jkt`, a proof targeting the exact public PAR
  endpoint, or both. If a proof is supplied, its JKT is authoritative; if both are
  supplied they must be equal. A request with neither is `invalid_request`.
- A proof must use the client's configured signing algorithm. A request carrying only
  `dpop_jkt` cannot reveal its curve; a later proof with the wrong profile fails. PAR
  with a proof detects that mismatch before browser authorization begins.
- Persist only `dpop_jkt` in the pushed request. The browser cannot add or override
  stored parameters.
- `/authorize` consumes only an unexpired request bound to its exact `client_id`.
  Required-PAR clients reject direct authorization parameters. PAR and direct forms
  cannot be combined.

Proof reservation may commit before pushed-request creation. If later validation or
creation fails, the proof remains consumed and retry requires a new proof. Likewise,
PAR consumption may commit before ordinary browser authorization state is created. If
state creation fails, the pushed request remains consumed and retry requires a new PAR.
These orderings deliberately favor one-use guarantees over retrying the same artifact.

Persist the authoritative `dpop_jkt` through PAR, opaque browser state, authorization
code, and refresh grant. Do not persist `dpop_alg`; current client policy selects the
algorithm at code redemption, refresh, and revocation boundaries.
Persist a boolean PAR provenance marker through opaque browser state and authorization
code, but not through the refresh grant: PAR policy applies to authorization, not later
refreshes.
Do not accept the binding from callbacks or later UI steps. At final code issuance,
recheck client policy and issue a code only if the required binding is present.

## Token Endpoint

### Authorization-code redemption

A code with `dpop_jkt` requires a fresh proof targeting the exact public token endpoint.
Validate the code, client, redirect URI, PKCE, current client policy, proof, and matching
JKT before consuming the code. Final code revalidation, code consumption, temporary
credential deletion, and grant creation must be atomic. Replay reservation may be in
that transaction or may commit earlier; therefore a failed later operation can consume
the proof but must not consume the code unless the final code transaction commits.

Errors are:

- missing, malformed, stale, replayed, wrong-method, wrong-target, or wrong-profile proof:
  `invalid_dpop_proof`;
- a valid proof under a different key: `invalid_grant`;
- a proof supplied for an unbound bearer code: `invalid_request`;
- replay-store or final storage failure: HTTP 503 with OAuth
  `temporarily_unavailable`.

A proof failure does not consume the code. A storage failure does not consume the code
unless its final transaction committed; a separately committed proof reservation is
not rolled back.

### Token issuance

Every DPoP access token contains:

```json
"cnf": { "jkt": "<jwk-thumbprint>" }
```

The response uses `token_type=DPoP`. Bearer access tokens omit `cnf` and use
`token_type=Bearer`. ID tokens remain ordinary OIDC assertions and do not gain `cnf`.

An access token's `cnf` determines how that token must be presented. Current client
configuration must not retroactively reinterpret a stateless token. Bound tokens always
require DPoP presentation; unbound tokens remain bearer tokens until expiry. This rule
does not authorize in-place mode changes, which remain unsupported.

### Refresh

Authenticate and inspect the complete refresh token and `client_id` without mutating
refresh replay or acquiring a processing claim. Unknown, malformed, wrong-client,
expired, or revoked tokens return `invalid_grant`.

A grant with `dpop_jkt` requires a fresh token-endpoint proof under that same key.
Missing or invalid proofs return `invalid_dpop_proof`; a valid proof under another key
returns `invalid_grant`. Proof failure must not consume or rotate the refresh token,
acquire a claim, invoke an upstream provider, or revoke the family. Only after proof
validation and replay reservation may REFRESH.md replay handling, claim acquisition,
upstream revalidation, and atomic rotation proceed.

Rotation copies the same `dpop_jkt` to the replacement refresh token/grant and to every
new access token's `cnf.jkt`; the response uses `token_type=DPoP`. A bearer grant rejects
a proof with `invalid_request` and cannot be upgraded. A DPoP grant cannot be downgraded
or rekeyed. Private-key loss requires reauthorization.

## Protected Resources and UserInfo

Present a bound access token as:

```http
Authorization: DPoP <access-token>
DPoP: <fresh-proof-with-ath>
```

Easy OIDC's `/userinfo` verifier must independently validate the access token's
signature, issuer, audience, expiry, and claims, then validate the proof, replay
reservation, `ath`, and `cnf.jkt` equality. It must reject a bound token presented as
Bearer and an unbound token presented as DPoP. Every Easy OIDC access-token consumer
must use this shared verifier.

Response behavior is:

- malformed or multiple authentication methods/headers: HTTP 400 `invalid_request`;
- missing credentials: HTTP 401 challenge without an error;
- invalid or expired token, including a valid wrong-key proof: HTTP 401 `invalid_token`;
- missing, malformed, stale, replayed, wrong-method, or wrong-target proof: HTTP 401
  `invalid_dpop_proof`;
- valid credentials without required scope: HTTP 403 `insufficient_scope`;
- replay-store failure: HTTP 503, with no protected response.

Put authentication errors on the `WWW-Authenticate` challenge for the attempted
scheme. A DPoP challenge may advertise `algs="ES256 ES512"`. Resource servers supporting
both schemes must inspect `cnf` on the bearer path and reject bearer downgrade. This is
integration guidance for external resource servers, not functionality Easy OIDC can
enforce outside its own endpoints.

## Proof-Bound Revocation

`/revoke` preserves RFC 7009 token privacy while preventing a copied DPoP token from
revoking its family:

1. Validate `client_id` and select its public policy before any token lookup.
2. A required client must supply a fresh proof targeting the exact public `/revoke`
   endpoint. Validate and independently reserve it without consulting the token.
   `ath` is not required. Missing or invalid proof errors reveal only client policy and
   proof validity.
3. After reservation, perform one conditional token lookup/revoke operation. Revoke
   only if the token belongs to that client and its binding equals the proof JKT.
   Unknown, malformed, wrong-client, expired, revoked, unbound, and wrong-key tokens all
   produce the same empty HTTP 200 and no mutation.
4. Commit the replay reservation even when token lookup or conditional revocation finds
   nothing. A replay conflict is a token-independent `invalid_dpop_proof` and performs
   no token lookup.
5. A disabled client follows ordinary RFC 7009 behavior and rejects any `DPoP` header
   with `invalid_request`.

A replay-store or transaction outage returns HTTP 503 and changes no token state;
token-dependent failures alone use empty HTTP 200. Revocation by any token type must
resolve to the same client and grant binding before changing the family. Local logout
deletes local credentials regardless of remote revocation success.

The RFC 8693 exchange described by TRUST.md remains bearer-only and rejects a `DPoP`
header with `invalid_request`; it is not changed by this spec.

## Replay Storage and Operations

Every verifier uses a shared durable `dpop_proofs` table:

- `replay_hash` primary key, computed as SHA-256 over a versioned, length-delimited
  encoding of JKT, `jti`, `htm`, and the exact `htu`;
- `expires_at`, after which the row can be removed.

A unique insert reserves a proof. Keep the row for the complete acceptance window and
delete expired rows in bounded batches. Never store the proof, raw `jti`, public JWK,
access token, or access-token hash. All replicas accepting proofs for the same trust
domain must share this table; an in-process or per-replica cache is insufficient.
Resource-server deployments maintain their own equivalently shared durable table.

Database unavailability returns HTTP 503 and fails closed. PAR creates no pushed
request; token operations do not consume or rotate credentials or acquire claims;
revocation changes nothing; and protected resources grant no access. A normal process
restart preserves unexpired reservations.

If the original replay table returns intact after an outage, resume normally. If its
records were lost through truncation, recreation, or another confirmed failure, continue
failing every DPoP request with HTTP 503 for at least the complete 15-second acceptance
window before accepting proofs against an empty replacement. Clients retry afterward
with fresh proofs; the interval ensures every forgotten proof has expired.
This operational procedure does not require a replay epoch, metadata identity,
high-water mark, rollback quarantine, startup count scan, active counters, fixed global
capacity, or credential invalidation protocol. Arbitrary wall-clock rollback and
administrative database truncation are operational failures outside the protocol's
guarantees; operators must protect and monitor the database and maintain reliable time.

Log replay outcomes and storage failures without logging proofs, tokens, token hashes,
full thumbprints, public JWKs, or raw `jti` values. Expose cleanup backlog, uniqueness
conflicts, and storage failures. Revocation has the same per-process aggregate admission
limit as PAR: 100 requests per second with a burst of 200, applied before body parsing,
policy, cryptography, or database work. Cleanup removes at most 2,000 expired replay rows
per five-second pass, leaving throughput above the combined unauthenticated admission
rate while keeping each cleanup transaction bounded.

## Browser, BFF, and Resource-Server Guidance

This section is integration guidance. It does not require Easy OIDC to implement a
browser, BFF, or external resource-server state machine.

Browsers should create a non-extractable P-256 or P-521 Web Crypto key matching the
client profile per account/grant slot and retain it for the grant lifetime.
Non-extractability reduces direct key theft but does
not prevent active same-origin code from signing; CSP and normal XSS defenses remain
necessary.

A BFF that keeps tokens in secure `HttpOnly` cookies needs browser-assisted PAR, code
redemption, refresh, resource access, and revocation because only the browser can sign.
It should bind pending work to the cookie session, account slot, client, JKT, PKCE state,
and current access-token `ath`; forward proofs unchanged to the exact public endpoint;
and update local state only after the downstream result. It must not expose raw access
or refresh tokens to JavaScript. CORS must allow `DPoP` and expose
`WWW-Authenticate` where applicable.

At a proxy boundary, choose one verifier and one public target: either the BFF validates
the proof for its browser-visible route, or the upstream validates a proof for its
explicitly configured public route. Never validate a proof against a rewritten internal
URL. External resource servers must validate token audience and claims, `cnf.jkt`, the
DPoP scheme and proof, exact method and public URL, `ath`, and shared replay state.

## Discovery and Compatibility

Publish:

```json
{
  "dpop_signing_alg_values_supported": ["ES256", "ES512"],
  "pushed_authorization_request_endpoint": "https://auth.example.com/par"
}
```

Discovery advertises server capability; client policy remains authoritative. Existing
disabled clients and bearer tokens retain their shape and behavior. DPoP clients must
compare `token_type` case-insensitively to `DPoP`, preserve the grant key, and create a
fresh proof for every retry. Enabling DPoP requires a new client ID and coordinated
resource-server enforcement; it is not a migration of existing bearer grants.

## Security Boundaries

- A copied bound access token is unusable at an enforcing resource server without the
  key; a copied refresh token cannot mint tokens; and a copied token cannot revoke its
  family.
- PKCE remains mandatory. `dpop_jkt` prevents authorization material from being rebound
  to another key.
- TLS, token signature, issuer, audience, expiry, scope, and ordinary authorization
  checks remain mandatory and independent of DPoP.
- DPoP provides no body integrity, client authentication, immediate access-token
  revocation, or protection after compromise of the client runtime, key, issuer, or
  resource server.
- DPoP is effective only when every token-accepting path checks `cnf.jkt` and prevents
  bearer downgrade.

## Implementation Plan and Verification Checklist

1. **Configuration and schema — `github.com/easy-oidc/easy-oidc`**
   - Add `dpop.mode=disabled|required` and `dpop.signing_algorithm=ES256|ES512` to
     static and policy-database client defaults; reject optional mode.
   - Add `require_par`, ES256/ES512 discovery, and durable
     replay storage. Persist only `dpop_jkt` through PAR, state, code, and refresh.
   - Test new-client migration rules and unchanged disabled-client bearer behavior.

2. **Proof verifier and replay — `github.com/easy-oidc/easy-oidc`**
   - Use a reviewed JOSE library with constrained ES256/P-256 and ES512/P-521 profiles;
     implement typed claims, exact trusted endpoint matching, JKT, `ath`, limits, and
     replay hashing/reservation.
   - Test algorithm/key confusion, remote key headers, malformed claims, signature and
     key failures, clock boundaries, replay, exact URL/method matching, and DB outage.
   - Test restart persistence, bounded expiry cleanup, and that ordinary replay-store
     failures return HTTP 503 before changing protocol state. Document the operator-
     enforced 15-second fail-closed interval after confirmed replay-record loss; silent
     truncation cannot be detected automatically without the rejected epoch metadata.

3. **PAR and authorization — `github.com/easy-oidc/easy-oidc`**
   - Implement bounded, authenticated, 60-second one-use PAR; proof/JKT equality;
     required-PAR enforcement; and client-bound unpredictable request URIs.
   - Carry only JKT through opaque state and code. Check policy at PAR creation and
     consumption and final code issuance, not each UI hop.
   - Test proof-first reservation failure semantics, PAR-first consumption failure
     semantics, PKCE, duplicate parameters, override rejection, expiry, and races.

4. **Token issuance and refresh — `github.com/easy-oidc/easy-oidc`**
   - Require matching proofs for code redemption and refresh; issue `cnf.jkt` access
     tokens and `token_type=DPoP`; preserve the key through refresh rotation.
   - Test exact errors, response-loss retries, code/refresh races, key mismatch, proof
     failure without credential mutation, key loss, and bearer downgrade rejection.

5. **UserInfo and revocation — `github.com/easy-oidc/easy-oidc`**
   - Route access-token use through one verifier enforcing scheme, `ath`, JKT, and
     replay. Make token `cnf` authoritative independently of current client policy.
   - Implement policy-first revocation, independent proof reservation, and one
     conditional lookup/revoke with identical empty 200 token-dependent outcomes.
   - Test every bound/unbound and Bearer/DPoP combination, copied-token revocation DoS,
     unknown/wrong-client/wrong-key tokens, replay, and storage outages.

6. **Integration and end-to-end verification — `github.com/easy-oidc/easy-oidc`**
   - Document concise browser/BFF/resource-server integration and provide non-secret
     ES256 and ES512 conformance fixtures.
   - Exercise direct and browser-assisted PAR, authorization, code exchange, resource
     access, refresh rotation, and logout/revocation with an independent RFC 9449 client.
   - Verify discovery, no nonce challenges, required-PAR failure behavior, disabled
     clients, full repository checks, and bearer downgrade prevention.
