# DPoP Sender-Constrained Tokens

> **STATUS**: Ready for implementation

> **Dependency:** This design depends on the refresh-grant and distinct-token model in
> [REFRESH.md](./REFRESH.md). Implement and verify REFRESH.md before starting DPoP.
>
> **Normative standards:** RFC 9449, RFC 7638, RFC 8032, RFC 8037, RFC 9126, and
> RFC 9864.

This document specifies RFC 9449 Demonstrating Proof of Possession (DPoP) for
Easy OIDC and the implementation plan. DPoP binds OAuth access and public-client
refresh tokens to a key held by the client so copying a token alone is insufficient
to use it.

## Goals

- Implement interoperable RFC 9449 authorization-code, token-endpoint, and protected-
  resource behavior rather than an application-specific proof protocol.
- Bind authorization codes, access tokens, and public-client refresh grants to the
  same client-generated key.
- Prevent a DPoP-bound token from being downgraded to bearer authentication.
- Preserve ordinary bearer operation for clients that do not enable DPoP.
- Make proof validation reusable and strict enough for Easy OIDC's `/userinfo`
  endpoint and document the same contract for application resource servers.
- Enforce short proof lifetimes and durable replay detection without requiring an extra
  nonce-challenge round trip.
- Support browser clients whose private key is non-extractable and whose tokens are
  held by a backend-for-frontend (BFF).

Dynamic client registration, token introspection, confidential clients, mTLS, and
DPoP-bound RFC 8693 token exchange are out of scope. DPoP is not client authentication,
does not replace PKCE or TLS, does not sign request bodies or general headers, and does
not protect against code actively executing in the legitimate client context.

## Client Configuration

DPoP is selected per downstream client:

```jsonc
"clients": {
  "browser-app": {
    "redirect_uris": ["https://app.example.com/callback"],
    "dpop": "required",
    "dpop_signing_alg": "ES512",
    "require_pushed_authorization_requests": true,
    "refresh_tokens": { "enabled": true }
  },
  "podplane-cli": {
    "redirect_uris": ["http://localhost:8000/callback"],
    "dpop": "disabled"
  }
}
```

The allowed values are:

- `disabled` (the default): issue only bearer tokens. Reject `dpop_jkt` at
  `/authorize` or `/par` and reject an unexpected `DPoP` header at those endpoints or
  `/token` with `invalid_request` rather than silently ignoring it.
- `optional`: accept either a complete bearer flow or a complete DPoP flow. Presence
  of `dpop_jkt` on a direct authorization request, or `dpop_jkt` or a proof at PAR,
  selects DPoP and binds the resulting code; absence of all applicable inputs selects
  bearer. A code or refresh grant can never change modes later.
- `required`: require key binding through `dpop_jkt` or the PAR proof and a matching
  DPoP proof at code redemption and every refresh. Never issue bearer access or refresh
  tokens.

Optional mode is for migrations and clients that deliberately support both modes.
Security-sensitive clients should use `required` and must verify that successful token
responses contain a case-insensitive `token_type` value of `DPoP`. The client mode
governs authorization-code access-token issuance, public-client refresh, and the token-
independent revocation proof policy. The RFC 8693 ID-token exchange specified by
TRUST.md remains unchanged for every mode.

For an optional or required client, `dpop_signing_alg` selects exactly one proof
profile and defaults to `ES256` for compatibility. The accepted values are `ES256`,
`ES384`, `ES512`, and `Ed448`. Authorization parameters and proof headers cannot
override the configured profile. Every proof throughout a grant must use that profile;
changing it invalidates existing DPoP grants and requires reauthorization rather than
silently grandfathering or downgrading them. Disabled clients reject this setting.
Whenever a pushed request, authorization state, code, or grant is consumed, require its
stored profile to equal the client's current configuration. A mismatch invalidates that
artifact and starts a new authorization flow. Configuration is immutable during a
process lifetime; reload takes effect through restart before requests resume.

`require_pushed_authorization_requests` defaults to `false` for compatibility. When
true, Easy OIDC accepts authorization parameters only through an RFC 9126 pushed
authorization request (PAR) and the browser authorization request contains only its
one-time `request_uri` and `client_id`. Production required-DPoP browser clients should
enable it.

Easy OIDC implements this closed profile table:

| JOSE `alg` | Required public JWK | Classical security | JOSE signature |
| --- | --- | ---: | ---: |
| `ES256` | `kty=EC`, `crv=P-256`, 32-byte `x` and `y` | about 128 bits | 64 bytes |
| `ES384` | `kty=EC`, `crv=P-384`, 48-byte `x` and `y` | about 192 bits | 96 bytes |
| `ES512` | `kty=EC`, `crv=P-521`, 66-byte `x` and `y` | about 256 bits | 132 bytes |
| `Ed448` | `kty=OKP`, `crv=Ed448`, 57-byte `x`, no `y` | about 224 bits | 114 bytes |

ES384, ES512, and Ed448 give clients security margins beyond ES256/P-256. Ed448 uses
the fully specified JOSE algorithm registered by RFC 9864, pure Ed448 with an empty
context, and never Ed448ph. Do not advertise or accept polymorphic `EdDSA`, Ed25519,
RSA, MAC, `none`, or an algorithm supplied by an installed provider. Ed25519 is in the
same approximate security class as ES256. RSA algorithm names do not constrain modulus
size, and longer RSA hash suffixes alone do not provide a stronger public-key security
level. Adding another profile requires a specification decision, security review,
interoperability tests, and updated discovery metadata.

## Protocol Flow

```text
client                     Easy OIDC                    resource server
  │                             │                              │
  │ /par + parameters + proof   │                              │
  │────────────────────────────▶│                              │
  │◀────────────── request_uri  │                              │
  │                             │                              │
  │ /authorize + request_uri    │                              │
  │────────────────────────────▶│                              │
  │◀────────────────────── code bound to jkt                   │
  │                             │                              │
  │ /token + code + DPoP proof  │                              │
  │────────────────────────────▶│                              │
  │◀──── DPoP access token + bound refresh token               │
  │                             │                              │
  │ Authorization: DPoP token + request proof                  │
  │───────────────────────────────────────────────────────────▶│
  │◀──────────────────────── protected response                │
  │                             │                              │
  │ /token + refresh token + matching DPoP proof               │
  │────────────────────────────▶│                              │
  │◀──── rotated refresh token + new DPoP access token          │
```

### Key generation and authorization

The client generates a key matching its configured `dpop_signing_alg` before starting
authorization. It computes `dpop_jkt` as the unpadded base64url encoding of the RFC 7638
SHA-256 JWK thumbprint of the public key and includes it in the authorization request
alongside PKCE:

```http
GET /authorize?response_type=code
    &client_id=browser-app
    &redirect_uri=https%3A%2F%2Fapp.example.com%2Fcallback
    &code_challenge=<challenge>
    &code_challenge_method=S256
    &dpop_jkt=<jwk-thumbprint>
```

Easy OIDC requires `dpop_jkt` to be one canonical, unpadded base64url value decoding
to exactly 32 bytes. Duplicate, empty, padded, malformed, or wrong-length values are
`invalid_request`. For a validated redirect URI, authorization errors use the normal
OAuth redirect and preserve `state`; Easy OIDC never redirects to an unvalidated URI.

Preserve the exact validated thumbprint and configured algorithm through opaque browser
state, connector callbacks, identity selection, consent, and the authorization code.
Neither value is secret, but the browser-visible state and code remain opaque references
as required by REFRESH.md. Do not accept either value from a callback or later flow
step.

PKCE S256 remains required. `dpop_jkt` supplements PKCE by preventing a captured code
and verifier from being redeemed under an attacker-controlled DPoP key. The client
should use a key scoped to the account or grant so logout or account removal can discard
that key without affecting unrelated grants.

### Pushed authorization requests

Implement RFC 9126 at a published `pushed_authorization_request_endpoint`. PAR accepts
only `POST` with a bounded `application/x-www-form-urlencoded` body, applies the same
duplicate-parameter, client, redirect URI, response type, scope, and PKCE validation as
`/authorize`, and stores a request object for at most 60 seconds. Public clients identify
themselves with the body `client_id`; confidential-client authentication remains out of
scope. A successful response returns HTTP 201 with a cryptographically random,
unguessable `request_uri` URN and `expires_in`; the object and every authorization
parameter stay server-side.

For DPoP clients:

- A required client must supply `dpop_jkt`, a DPoP proof targeting the public PAR
  endpoint, or both. An optional client selects DPoP the same way; if neither is
  present, the pushed request selects bearer mode. A disabled client rejects either.
- A proof-bearing PAR request does not require a preliminary nonce challenge. A request
  using only `dpop_jkt` does not require a proof; code redemption still proves possession
  of that key as RFC 9449 requires.
- Before signature verification, require a supplied proof's protected algorithm and key
  shape to equal the client's configured profile. Its JWK thumbprint is the authoritative
  `dpop_jkt`. If the body also contains `dpop_jkt`, require exact equality or reject the
  PAR request.
- Store the authoritative thumbprint and configured algorithm in the one-time pushed
  request object. The browser authorization request may not add or override either.
  Preserve them through authorization, the code, and token redemption as specified
  below.

For a proof-bearing request, apply the same freshness, replay, strict proof parsing, and
storage-failure rules as `/token`. The distinct `htu` and replay-hash inputs keep the
endpoints separate. Insert its replay reservation and create the pushed request in one
transaction, so a storage failure consumes neither.

The browser presents exactly `client_id` and `request_uri` to `/authorize`; all other
authorization parameters, including `redirect_uri`, PKCE, scope, `state`, and
`dpop_jkt`, come exclusively from the stored object. Require exact client equality,
the stored profile to equal current client configuration, unexpired unused state, and an
ordinary validated redirect URI before any redirect. Atomically consume the object when
creating the opaque authorization state; a failed or repeated consume never resumes
authorization. Recheck the profile when consuming that state after every callback,
identity, or consent step. A client with
`require_pushed_authorization_requests=true` rejects ordinary front-channel parameters
and cannot fall back when PAR is unavailable. Other clients may use the direct
authorization request described above, but the two forms may never be combined.

### DPoP proof JWT

Every proof is a compact JWS sent as the single value of the `DPoP` HTTP request
header. Its protected JOSE header is:

```json
{
  "typ": "dpop+jwt",
  "alg": "ES512",
  "jwk": {
    "kty": "EC",
    "crv": "P-521",
    "x": "<coordinate>",
    "y": "<coordinate>"
  }
}
```

An EC `jwk` has required key-bearing members `kty`, `crv`, `x`, and `y`; an OKP JWK has
`kty`, `crv`, and `x`. It must not contain private key material. Strictly parse but
ignore additional public-key metadata; derive the verification key and RFC 7638
thumbprint only from the required members, and never resolve or trust `kid`, `x5*`,
`use`, `key_ops`, `alg`, or other metadata as a key source. If `jwk.alg` is present as
ignored metadata, require exact equality with the protected `alg` before ignoring it.

Map the protected `alg` to exactly one table profile before invoking cryptography.
Require canonical unpadded base64url and exact key lengths. For EC, reject an infinity,
invalid, or off-curve point and nonzero unused high P-521 coordinate bits. For Ed448,
reject a non-canonical encoding, identity or small-order point, and a key outside the
prime-order subgroup. Reject symmetric or private keys, unlisted algorithms, duplicate
JOSE members, unprotected headers, any `crit` or `b64` parameter, detached payloads, and
unencoded payloads. The three compact-JWS segments are non-empty canonical unpadded
base64url.

The token-endpoint proof payload contains:

```json
{
  "jti": "<unique proof identifier>",
  "htm": "POST",
  "htu": "https://auth.example.com/token",
  "iat": 1785200000
}
```

A protected-resource proof additionally contains:

```json
{
  "ath": "<base64url(SHA-256(ASCII(access-token)))>"
}
```

Apply all of these checks before accepting a proof:

1. Require exactly one `DPoP` header containing one compact JWS and enforce an 8 KiB
   encoded-proof limit before decoding.
2. Strictly decode JSON objects, rejecting duplicate member names, invalid UTF-8,
   non-numeric, non-finite, or out-of-range `iat`, non-string required claims, and
   malformed JWKs. Compare NumericDate values without truncating fractional seconds.
3. Require protected `typ` to be the exact string `dpop+jwt`; reject it when missing,
   non-string, or different. Require the client's configured JOSE profile and verify
   with that key. For ECDSA, enforce the profile's fixed-width raw `R || S` signature
   and reject invalid scalar values or ASN.1 encodings. For Ed448, require a canonical
   114-byte pure-Ed448 signature with an empty context; never accept Ed448ph.
4. Require non-empty `jti`, `htm`, and `htu`. Limit `jti` to 128 UTF-8 bytes. Require
   `htm` to exactly equal the request method. Clients should generate `jti` with at
   least 96 bits of pseudorandomness or use a UUIDv4.
5. Require `htu` to be an absolute URI with no userinfo, query, or fragment. Construct
   the expected URI from the configured public endpoint while excluding the request
   query and fragment, then apply RFC 3986 syntax- and scheme-based normalization to
   both values. Never derive it from untrusted `Host`, `Forwarded`, or `X-Forwarded-*`
   headers.
6. Accept `iat` from at most five seconds in the future and no more than ten seconds in
   the past. Boundary values are accepted; values outside the window are rejected.
7. Atomically reject reuse of the same proof `jti` by the same JWK thumbprint, method,
   and target URI during the acceptance window.
8. For protected-resource requests, require `ath` and compare it in constant time to
   the unpadded base64url-encoded SHA-256 hash of the exact ASCII access-token value.
   For token requests, `ath` is not required and has no binding effect.
9. When validating a bound credential, compare the RFC 7638 thumbprint of the proof
   JWK in constant time to the code, grant, or access token's `jkt`.

Each HTTP retry uses a newly generated proof and `jti`. Proofs are never credentials
on their own and must not influence identity or authorization before the associated
code, refresh token, or access token is independently validated.

### Public URL matching

For Easy OIDC, one URL builder produces discovery metadata and every expected public
endpoint URL, including `/par`, `/token`, `/revoke`, and `/userinfo`, from the configured
issuer URL. Require the issuer to be absolute with a host and no userinfo, query, or
fragment; remove a trailing slash before appending endpoint paths while preserving any
issuer path prefix. URI comparison normalizes case in scheme and host, removes a default
port, and normalizes percent-encoding and dot segments as permitted by RFC 3986. It does
not decode reserved characters or otherwise make two distinct paths equivalent.

Application resource servers must configure or safely reconstruct their externally
visible URL using only trusted proxy configuration. A proof for an internal upstream
URL, another host, another route, or another HTTP method is invalid even if the token
and signature are otherwise valid.

### Authorization-code exchange

A DPoP-selected code exchange requires a fresh proof targeting Easy OIDC's token
endpoint. A preliminary non-mutating read may identify the code's client, DPoP mode,
expected `dpop_jkt`, and `dpop_alg`, but it is not authoritative and does not consume
the code. Validate the request shape, code association, current authorization, proof,
freshness, algorithm, and proof JWK thumbprint, and prepare and sign the response tokens
before the final transaction.

In one SQLite transaction, insert the proof replay reservation; reload and revalidate
the unchanged, unexpired code, its profile equality with current client configuration,
and every REFRESH.md redemption invariant; consume the code; delete temporary flow
credentials; and create the initial grant/token with the same `dpop_jkt` and `dpop_alg`.
Any failure rolls back both the replay reservation and code/grant changes. Never
implement redemption as an authoritative read followed by a separate consume operation.

If the code has no `dpop_jkt`, reject a supplied `DPoP` header with `invalid_request`
rather than upgrading the code.
If the code has `dpop_jkt`, a missing proof is `invalid_dpop_proof`; a malformed,
expired, replayed, wrong-target, or wrong-profile proof is also `invalid_dpop_proof`;
and a valid pinned-profile proof under a different key is `invalid_grant`. None of these
failures consumes the code. A storage failure in the final transaction returns HTTP 503
with OAuth JSON `temporarily_unavailable` and consumes neither the proof nor the code.

For a refresh-eligible exchange, copy the code's thumbprint into the grant's existing
`dpop_jkt` field and its profile into a new `dpop_alg` field in the same transaction that
consumes the code and creates the grant and initial refresh token. For a refresh-disabled
exchange, use both values while issuing the access token; no persistent grant is needed.

### Access and ID token issuance

Every access token issued from a DPoP-selected code or bound refresh grant contains:

```json
"cnf": {
  "jkt": "<jwk-thumbprint>"
}
```

The value is the unpadded base64url RFC 7638 SHA-256 thumbprint already stored with
the code or grant. The token response contains `token_type=DPoP`. Bearer flows omit
`cnf` and continue returning `token_type=Bearer`.

ID tokens remain ordinary OIDC identity assertions: never add `cnf` merely because
the accompanying access and refresh tokens are DPoP-bound. Keep REFRESH.md's distinct
ID/access token claims, `sid`, `jti`, expiry, and scope rules unchanged. DPoP does not
change the refresh-token wire format, family lineage, expiry, consent, upstream-
credential encryption, family-wide effect of a successfully authorized revocation, or
refresh-token replay semantics.

### Refresh

Parse the refresh token and perform a non-mutating authenticated lookup by complete-
token hash and `client_id`. This lookup returns the grant's `dpop_jkt` and consumed
state and its `dpop_alg`, but must not revoke replay, acquire a claim, or otherwise
mutate the grant. Unknown, malformed, wrong-client, expired, or revoked credentials
return `invalid_grant`.

Every use of a DPoP-bound public-client refresh token then requires a fresh proof
targeting the token endpoint whose JWK thumbprint and algorithm match
`refresh_grants.dpop_jkt` and `refresh_grants.dpop_alg`.
A missing, invalid, stale, or replayed proof must not acquire a refresh processing
claim, call an upstream provider, rotate or consume the refresh token, or revoke its
family. Return `invalid_dpop_proof` for those failures, a protected algorithm that does
not equal the stored and currently configured profile, or a key shape that does not map
to it. Return `invalid_grant` only for a valid pinned-profile proof whose key does not
match the grant.

Only after successful proof validation and replay reservation may a transaction apply
REFRESH.md's consumed-token replay handling or acquire the refresh processing claim.
Continue with upstream revalidation and atomic rotation unchanged. The replacement
refresh token remains bound to the same `dpop_jkt` and `dpop_alg`; key or profile
rotation within a grant is not supported. Every new access token contains the same
`cnf.jkt`, and the response uses `token_type=DPoP`. Storage APIs must therefore separate
authenticated inspection from replay revocation and claim acquisition.

A bearer grant rejects a supplied `DPoP` header with `invalid_request` and cannot be
upgraded during refresh. A DPoP grant rejects a proofless request and cannot be
downgraded. Loss of the private key therefore requires a new authorization flow; it is
not an `invalid_grant` event that revokes the old family automatically. The client
should revoke the old family when it still possesses a credential capable of doing so.

### Protected resources and `/userinfo`

Present a DPoP-bound access token using both headers:

```http
Authorization: DPoP <access-token>
DPoP: <fresh-proof-with-ath>
```

Easy OIDC's `/userinfo` endpoint, every future Easy OIDC endpoint that accepts access
tokens, and every DPoP-aware application resource server must:

1. Reject multiple `Authorization` methods or multiple `DPoP` headers.
2. Verify the access JWT's signature, issuer, audience, expiry, and normal application
   claims independently of the proof.
3. Require `Authorization: DPoP` and a valid proof when the token has `cnf.jkt`.
4. Verify proof signature, `htm`, public `htu`, `iat`, replay state, `ath`, and exact
   thumbprint equality with the access token's `cnf.jkt`; require the algorithm/key
   mapping in the fixed profile table and, where client policy is available, its pinned
   profile.
5. Reject a bound token presented as `Bearer`. Also reject an unbound token presented
   as `DPoP`; it remains valid only through the existing bearer path.
6. Grant access only after every token and proof check succeeds.

Use this response matrix for protected resources:

- Multiple token-presentation methods or malformed authentication requests return HTTP
  400 with `invalid_request`.
- Missing credentials return HTTP 401 with a challenge and no `error` or
  `error_description`.
- An invalid or expired access token, or a valid proof under a key different from
  `cnf.jkt`, returns HTTP 401 with `invalid_token`.
- A missing, malformed, stale, replayed, wrong-target, or unacceptable-profile proof
  returns HTTP 401 with `invalid_dpop_proof`.
- Valid credentials lacking required scope return HTTP 403 with `insufficient_scope`.

Put error parameters on the `WWW-Authenticate` challenge corresponding to the attempted
scheme. Include only the known client's pinned profile, for example `algs="ES512"`, on
its DPoP challenge. A challenge issued before a client is known may list the complete
closed profile. Bound error descriptions and never reveal which sensitive claim or
stored value differed. HTTP authentication scheme names are case-insensitive. Resource
servers that also support bearer tokens may advertise both challenges. Browser-facing
endpoints expose `WWW-Authenticate` through CORS.

A resource server accepting both schemes must inspect `cnf` even on its bearer path;
otherwise a copied DPoP-bound JWT can be replayed as an ordinary bearer token. JWT
signature validation alone is insufficient. Easy OIDC cannot enforce this for external
APIs, so operators must not enable DPoP for a client until every resource server that
accepts its access tokens enforces this rule.

Easy OIDC must route every access-token-authenticated endpoint through one shared
verifier that implements these checks. New endpoints may not decode or verify access
tokens through a bypass path. Startup and endpoint tests inventory all registered
access-token consumers. If any configured client is optional or required but DPoP
verification or replay storage is unavailable, startup fails rather than silently
issuing or accepting bearer tokens.

### Revocation, logout, and other endpoints

Extend REFRESH.md's `/revoke` contract so possession of a copied DPoP-bound token cannot
be used to revoke its family while preserving RFC 7009 token privacy. Authenticate the
public `client_id` and select proof handling from client configuration before looking up
the token; externally visible proof requirements never depend on whether the submitted
token exists or is bound:

- A required client always supplies a fresh proof targeting the public `/revoke` URL
  when revoking. An optional client supplies one when it wants to revoke a bound grant
  and may omit it for a bearer grant. A disabled client rejects a supplied proof with
  `invalid_request`.
- When configuration requires a proof or an optional client supplies one, validate it
  without consulting the submitted token. `ath` is not required because revocation is
  not protected-resource access. Missing or malformed proof errors therefore disclose
  only public client policy and supplied proof state, never token state.
- Unknown, malformed, wrong-client, expired, already revoked, or otherwise unusable
  tokens return the same empty HTTP 200 response. A known bound token without a valid
  proof under its exact key also returns that response and changes no state. Never
  return a wrong-key or missing-proof error based on token lookup.
- For a proofless optional-client request, perform REFRESH.md's non-mutating token lookup
  and revoke only an unbound token. A bound or unknown token receives the same empty
  response and no mutation.
- For every proof-bearing request, perform no preliminary token lookup. After token-
  independent proof validation, one transaction reserves the proof; loads the submitted
  refresh, access, or ID token for the first and only time; obtains its `dpop_jkt` and
  `dpop_alg` from the refresh grant, access token/client policy, or ID-token `sid`; and
  conditionally revokes only an unbound token or an exactly matching bound family.
  Reserve every otherwise-valid proof regardless of token existence, usability, or key
  match. Commit the reservation even when no token is found or no revocation occurs, so
  the proof cannot later be reused with another submitted token. A uniqueness conflict
  returns the same proof-replay error without token lookup and leaves token state
  unchanged. The HTTP response remains the empty RFC 7009 success response for every
  token-dependent outcome.

Application logout still deletes local credentials even when revocation fails. A BFF
or direct DPoP client should send the proof while it still holds the grant key. The
self-service `/grants` flow uses its separately authenticated one-time action and is
unchanged.

Easy OIDC's initial RFC 8693 exchange returns an ID token in the OAuth `access_token`
response field, not a resource access token. It therefore remains bearer-only, never
adds `cnf`, and rejects a supplied `DPoP` header with `invalid_request` regardless of
the client's DPoP mode. Supporting DPoP-bound access tokens from token exchange requires
a separate extension to TRUST.md and is not implied by this design.

## Browser and BFF Integration

For browser hardening, use one non-extractable Web Crypto EC key matching the client's
configured ES256, ES384, or ES512 profile per account slot, stored in IndexedDB. Native
Web Crypto does not provide Ed448; do not emulate it with JavaScript or Wasm and claim
equivalent non-extractability. Ed448 is for CLI, native, mobile, HSM, or other clients
with a reviewed implementation. A browser private key cannot be exported, but same-
origin JavaScript can still invoke signing; DPoP reduces harm from copied tokens and
cookies but does not stop active XSS. Use CSP and ordinary XSS defenses independently.

A BFF holding tokens in `HttpOnly` cookies cannot create a proof for a browser-held
key. The browser must participate at every binding point:

1. Generate the key, submit the pushed authorization request and its proof through the
   BFF, and redirect with the returned `request_uri`. The BFF must not create a pending
   authorization or redirect until PAR succeeds.
2. After callback, let the BFF retain the short-lived code as a pending login. The
   browser signs a token-endpoint proof and sends it to the BFF, which forwards it
   unchanged. Consume the pending code only after token exchange succeeds.
3. Return the non-secret access-token `ath` value to the browser while storing access
   and refresh tokens only in secure `HttpOnly` cookies.
4. For each API request, have the browser sign a proof over the public method and URL
   with that `ath`. Choose exactly one validation topology:
   - The BFF validates against the cookie-held access token and browser-visible BFF
     route. It must not forward that proof as if it targeted a rewritten upstream URL.
   - An upstream API validates while the BFF injects `Authorization: DPoP <token>` and
     forwards the proof without changing its externally configured method and target
     URI. The API validates the browser-visible URI, not an internal proxy URL.
5. Refresh through an explicit browser-assisted route: the browser signs a new token-
   endpoint proof, the BFF forwards it with the cookie-held refresh token, rotates the
   cookies only after success, and returns the new access-token `ath`.
6. Revoke through a browser-assisted route before deleting the slot: the browser signs
   for Easy OIDC's public `/revoke` URL, the BFF forwards the proof with a slot token,
   and local logout deletes the cookies regardless of the remote result.

The BFF must bind each pending login and assisted API request to the cookie session,
account slot, client, expected `dpop_jkt`, and current `ath`; also bind pending login to
redirect URI and PKCE state, expire it quickly, and consume it once. For token exchange
refresh, and revocation it forwards the proof without rewriting `htu`, so the browser
signs Easy OIDC's public endpoint URL, not the BFF route. Pending login, refresh, revoke,
and API state transitions occur only after the downstream response is final. CORS
deployments must allow the `DPoP` request header and expose `WWW-Authenticate`. The BFF
never puts raw access or refresh tokens in JavaScript-readable storage or responses.

CLI, mobile, and direct API clients may hold both tokens and the private key themselves.
They use the standard `Authorization: DPoP` scheme and do not need the BFF adaptation.

## State, Replay, and Limits

Add a logical `dpop_proofs` table to Easy OIDC's authoritative SQLite database:

- `replay_hash` primary key: SHA-256 of a versioned, length-delimited encoding of JWK
  thumbprint, `jti`, normalized `htm`, and normalized `htu`
- creation and expiry timestamps

Also persist a replay-store initialization/epoch marker so startup can distinguish a
valid empty table from lost, corrupt, or recreated replay history, plus a wall-clock
high-water mark advanced transactionally with every accepted proof and every cleanup or
deletion operation.

Extend opaque authorization state, authorization codes, and pushed requests with the
pinned `dpop_alg` alongside `dpop_jkt`, and add `refresh_grants.dpop_alg` alongside the
reserved `dpop_jkt`. Treat either both values as present or both as absent. A migration
must reject impossible partial states and preserve all existing bearer grants as the
both-absent case.

Never store the proof, raw `jti`, public JWK, access-token hash, or access token. Insert
the replay hash only after the endpoint's non-mutating prerequisites, proof signature,
and expected key binding where one exists have been validated. Revocation is the
exception: reserve every otherwise-valid proof before token-dependent binding as defined
above. For PAR the insert shares pushed-request creation; for code redemption it shares
final consumption and grant creation; for revocation it shares the non-enumerating
conditional revoke. For refresh it precedes replay handling or claim acquisition, and
for protected resources it precedes granting access. A uniqueness conflict is proof
replay. Keep each row through the complete inclusive acceptance window: cleanup may
delete it only when wall time is strictly greater than 15 seconds after acceptance.
Delete rows in bounded batches and never create a gap in the active replay window.

Normal process restart preserves every unexpired row. If the replay table, epoch marker,
or high-water mark is missing, corrupt, or recreated while previously issued credentials
or authorization artifacts may remain valid, fail closed until the state is restored or
the wall clock is verified against a trusted source and every affected credential and
artifact is invalidated. A blind time-based quarantine is insufficient when history and
its high-water mark are both unavailable; never treat a newly empty store as valid
replay history.

A wall clock below the persisted high-water mark enters fail-closed quarantine. Service
resumes only after wall time is strictly greater than the high-water mark plus 15 seconds,
or after all affected credentials and artifacts are invalidated. Each cleanup transaction
first advances the high-water mark to at least its observed wall time and only then
deletes rows; both changes commit atomically. The mark never moves backward.

For `/userinfo` and application resources, replay-state failure or quarantine returns
HTTP 503 and grants no access. An application resource server that accepts Easy OIDC
DPoP tokens must use its own durable replay store, shared across every replica serving
the same public endpoint and preserving the full acceptance window across normal
restarts. An in-process or per-replica cache is not conformant to this profile. Namespace
replay hashes by resource server and public target URI when sharing a store. Every such
resource server must implement the same epoch/high-water loss detection, transactional
high-water advancement, rollback quarantine, trusted-clock recovery, and fail-closed
rules above; durable rows alone are insufficient.

While replay state is unavailable or quarantined, `/token` returns HTTP 503 with OAuth
JSON `temporarily_unavailable` and does not consume a code or refresh token, acquire a
refresh claim, rotate credentials, or revoke a family. PAR returns HTTP 503 with OAuth
JSON `temporarily_unavailable` and creates no pushed request. Revocation returns HTTP 503
and must not substitute its ordinary empty success response; it reserves no proof and
changes no token state. Transaction failure has the same endpoint-specific behavior.

Apply these fixed limits before expensive parsing or signature verification:

- one DPoP header and one Authorization credential
- 8 KiB compact proof
- 128 UTF-8 bytes for `jti`
- ten-second proof age and five-second future clock skew
- only the four fixed algorithm/key profiles in the Client Configuration table

Log endpoint, client ID when known, JWK thumbprint prefix or a separate diagnostic
digest, result, remote IP, and user agent. Never log proofs, raw tokens, token hashes,
full thumbprints, public JWKs, or raw `jti` values. Emit a distinct security event for
proof replay without conflating it with refresh-token family replay. Bound replay-store
growth and insertion rates, clean up in bounded batches, and expose metrics for insert
conflicts, cleanup backlog, storage failures, and fail-closed quarantine.

## Nonce Challenges

Easy OIDC does not require the optional RFC 9449 authorization-server nonce mechanism at
PAR, token, revocation, UserInfo, or application resource endpoints. A valid fresh proof
is accepted in one request. The ten-second proof window and durable atomic replay store
provide freshness and single-use enforcement without challenge amplification, per-key
nonce state, or an additional fail-closed storage dependency.

RFC 9449 clients may implement `use_dpop_nonce` for interoperability with other servers,
but Easy OIDC does not issue that error or `DPoP-Nonce`. Adding nonce enforcement later
requires a separate specification and availability analysis; it must never become
enabled implicitly through a library or provider upgrade.

## Discovery and Compatibility

Publish this OAuth authorization-server metadata in Easy OIDC's OpenID Provider
configuration document:

```json
{
  "dpop_signing_alg_values_supported": ["ES256", "ES384", "ES512", "Ed448"],
  "pushed_authorization_request_endpoint": "https://auth.example.com/par"
}
```

OpenID Connect discovery permits metadata parameters registered by OAuth extensions.
Publishing the field advertises server capability, not that every client is enabled;
the per-client mode remains authoritative.

Existing disabled clients, kubelogin configurations, and bearer resource servers keep
their current behavior and token shape. Optional and required DPoP clients need updated
client libraries. Easy OIDC emits the canonical value `DPoP`; a DPoP client compares
`token_type` ASCII case-insensitively and must reject a successful response with another
value, create a fresh proof for every retry, preserve its key for the grant's whole
lifetime, and start a new authorization flow after key loss.

## Security Properties and Boundaries

- A copied DPoP access token cannot be used at an enforcing resource server without a
  fresh proof from the bound key.
- A copied public-client refresh token cannot mint new tokens without a matching proof;
  strict refresh rotation and family replay detection still apply independently.
- `dpop_jkt` prevents captured authorization material from being rebound to another
  key. PKCE remains mandatory because DPoP is not a replacement for it.
- A non-extractable browser key prevents direct private-key export but does not prevent
  same-origin malicious code from asking the key to sign requests or pre-generating
  proofs. The short acceptance window limits pre-generation, and durable replay state
  prevents any accepted proof from being reused.
- DPoP does not provide request-body integrity, client authentication, immediate access-
  token revocation, or protection after compromise of the Easy OIDC process, resource
  server, client runtime, or signing key.
- TLS and audience validation remain mandatory. A resource server must validate the
  token audience and public request URL in addition to the DPoP binding.
- DPoP only provides its benefit where every path accepting the bound token understands
  `cnf.jkt` and rejects bearer downgrade.

## Implementation Plan

1. **Proof primitives and tests**
   - **Repository:** `easy-oidc/easy-oidc`
   - Add strict compact-JWS parsing; exact P-256, P-384, P-521, and Ed448 public JWK
     validation; RFC 7638 thumbprints; fixed-profile signature verification; `ath`;
     canonical public-URL matching; limits; and typed proof errors in a reusable internal
     package.
   - Put an application-owned strict decoding boundary in front of the existing JOSE
     library: never use automatic key resolution or follow attacker-selected `jku`,
     `x5u`, `x5c`, or other key sources; select only the configured fixed profile and
     verify with the already validated embedded public JWK.
   - Use the Go standard library for ECDSA. Gate Ed448 on a pinned, reviewed dependency;
     do not implement the primitive locally or release it without RFC 8032, RFC 9864,
     Wycheproof, non-canonical encoding, low-order-key, and subgroup tests.
   - Test malformed compact forms, duplicate JSON members, private/symmetric/off-curve
     JWKs, algorithm/curve confusion, every fixed signature width, P-521 high bits,
     Ed448 edge cases, claim types, URL normalization, clock boundaries, `ath`, and
     constant-time binding comparisons.

2. **Configuration, schema, and discovery**
   - **Repository:** `easy-oidc/easy-oidc`
   - Add per-client `dpop` mode with a secure disabled default while leaving RFC 8693
     ID-token exchange behavior unchanged for every mode.
   - Add the pinned `dpop_signing_alg` and `require_pushed_authorization_requests`, fail
     startup on algorithm/provider or DPoP storage drift, advertise the closed profile
     table and PAR endpoint, and update configuration JSON Schema, examples, migration
     behavior, and discovery tests.

3. **Authorization and code binding**
   - **Repository:** `easy-oidc/easy-oidc`
   - Validate `dpop_jkt` and the pinned algorithm according to client mode and preserve
     both through OAuth state, callbacks, consent, and authorization-code storage.
   - Implement bounded, single-use RFC 9126 PAR with proof/`dpop_jkt` equality, front-
     channel override rejection, and required-client fail-closed behavior.
   - Require exact code/proof thumbprint equality and put replay reservation, final code
     revalidation/consumption, temporary-credential deletion, and grant creation in one
     transaction; copy the thumbprint and algorithm into `refresh_grants`.
   - Test parameter duplication/canonicalization, optional/required/disabled modes,
     state tampering, key substitution, PKCE interaction, retry after proof failure,
     and code redemption races.

4. **Replay storage and token endpoint**
   - **Repository:** `easy-oidc/easy-oidc`
   - Add durable proof replay reservations, the initialization/epoch marker, fail-closed
     loss/clock-rollback quarantine, capacity controls, and bounded cleanup using the
     same SQLite durability and availability posture as refresh state.
   - Split refresh inspection from mutation; authenticate and inspect the credential,
     validate and reserve its proof, then permit consumed-token replay handling or claim
     acquisition. Preserve key binding through rotation and return exact OAuth DPoP
     errors without consuming otherwise retryable credentials.
   - Test stale/future/replayed proofs, wrong methods/URLs/keys, storage outage, process
     restart, response loss, exact inclusive retention boundaries, rollback and wall-
     clock catch-up, and separation from strict refresh-token replay revocation.
   - Verify PAR, token, revocation, UserInfo, and application-resource replay-store
     outages all return 503 without creating state, mutating credentials, or granting
     access.

5. **Token issuance and UserInfo enforcement**
   - **Repository:** `easy-oidc/easy-oidc`
   - Add `cnf.jkt` only to bound access JWTs, emit the correct `token_type`, and leave ID
     token shape unchanged.
   - Enforce `Authorization: DPoP`, proof validation, `ath`, key binding, and replay
     through one verifier used by `/userinfo` and every future access-token endpoint;
     preserve bearer behavior only for unbound tokens.
   - Test every bearer/DPoP scheme and bound/unbound token combination, malformed and
     multiple headers, downgrade attempts, challenges, and replay-store failure.

6. **Revocation and logout**
   - **Repository:** `easy-oidc/easy-oidc`
   - Require a matching proof and replay reservation before revoking a DPoP-bound grant
     while preserving RFC 7009 non-enumeration and local logout behavior.
   - Test revocation by refresh, access, and ID token; copied-token denial of service;
     required/optional error behavior, wrong keys, and storage failure.

7. **Resource-server and browser integration guidance**
   - **Repository:** `easy-oidc/easy-oidc`
   - Document the complete resource-server validation contract, trusted external URL
     handling behind proxies, shared replay storage, CORS, browser Web Crypto key
     lifecycle, BFF PAR and pending-login exchange, explicit assisted refresh, key loss,
     and the non-secret `ath` bridge.
   - Provide conformance fixtures containing public keys, proofs, access tokens, and
     expected validation results without publishing any production credential.

8. **End-to-end verification**
   - **Repository:** `easy-oidc/easy-oidc`
   - Add direct public-client and browser/BFF end-to-end tests for PAR authorization,
     code substitution resistance, resource access, refresh rotation, copied-token
     failure, logout/revocation, private-key loss, and optional-mode bearer compatibility.
   - Exercise every fixed profile with a direct client and all three Web Crypto EC
     profiles with the browser/BFF; verify algorithm pinning, configuration-change grant
     invalidation, and rejection of Keycloak's broader but non-profile algorithms.
   - Assert that every fresh valid proof succeeds in one downstream request and that no
     Easy OIDC endpoint emits `use_dpop_nonce` or `DPoP-Nonce`.
   - Verify that disabled clients and existing kubelogin bearer flows are unchanged,
     run the repository's full checks, and test against an independent RFC 9449 client
     implementation.
