# External OIDC Trust

This document specifies how Easy OIDC trusts external OIDC tokens and how the
Podplane CLI uses that trust for secretless CI authentication.

## Goals

- Let `podplane login` authenticate in GitHub Actions and Buildkite without a
  browser or stored credential.
- Support federating any standards-compliant OIDC issuer through generic configuration.
- Make trusted issuers, reusable policies, client authorization, subjects, and
  Kubernetes groups explicit and reviewable.
- Make dangerous ambiguity and accidental policy broadening fail closed.

Client credentials, refresh tokens for exchanged identities, non-OIDC tokens,
and a general policy expression language are out of scope.

## Configuration

Issuer verification, reusable trust policy, and client authorization are
separate:

```jsonc
{
  "oidc_trust": {
    "issuers": {
      "github-actions": {
        "provider": "github"
      },
      "buildkite": {
        "provider": "buildkite"
      },
      "company-ci": {
        "provider": "oidc",
        "issuer_url": "https://ci.example.com",
        "signing_algs": ["RS256"],
        "max_token_age": "5m"
      }
    },

    "policies": {
      "acme-github": {
        "issuer": "github-actions",
        "subject": "trusted:github:acme:deploy",
        "groups": ["podplane:operators"],
        "required_claims": {
          "repository_owner_id": { "const": "123456" }
        },
        "claims": {
          "repository_visibility": { "enum": ["private", "internal"] },
          "ref": { "const": "refs/heads/main" }
        }
      },
      "acme-buildkite": {
        "issuer": "buildkite",
        "required_claims": {
          "organization_id": {
            "const": "0184990a-477b-4fa8-9968-496074483cee"
          }
        }
      }
    }
  },

  "clients": {
    "cluster-production": {
      "redirect_uris": ["http://localhost:8000/callback"],
      "trust_bindings": [
        {
          "id": "github-production",
          "trust_policy": "acme-github",
          "subject": "trusted:github:acme/app:production",
          "groups": ["podplane:operators"],
          "claims": {
            "repository_id": { "const": "987654" },
            "ref": {
              "type": "string",
              "pattern": "^refs/heads/release/.*$"
            },
            "environment": { "const": "production" },
            "job_workflow_ref": {
              "const": "acme/deploy/.github/workflows/deploy.yml@refs/heads/main"
            },
            "event_name": { "enum": ["push", "workflow_dispatch"] }
          }
        },
        {
          "id": "buildkite-production",
          "trust_policy": "acme-buildkite",
          "subject": "trusted:buildkite:acme/app:production",
          "groups": ["podplane:operators"],
          "claims": {
            "pipeline_id": {
              "const": "0184990a-4782-42b5-afc1-16715b10b1f0"
            },
            "build_branch": { "const": "main" },
            "step_key": { "const": "deploy-production" }
          }
        }
      ]
    }
  }
}
```

### Inheritance

A binding inherits its policy, then overrides matching ordinary keys:

```text
Policy claims:    A={const: 1}, B={const: 2}
Binding claims:   A={const: 3}
Effective claims: A={const: 3}, B={const: 2}
```

- Each claim value is a JSON Schema Draft 2020-12 fragment evaluated against
  that verified claim value.
- Binding `claims` replace policy `claims` fragments with the same claim name.
- Unspecified policy claims are inherited.
- Every configured claim is implicitly required to exist and must satisfy its
  effective fragment; operators never write a separate JSON Schema `required`
  array.
- `required_claims` are inherited and cannot be removed or replaced by a
  binding. A binding fragment for the same claim is an additional restriction.
- Binding `subject` and `groups` replace policy values rather than merging them.
- Each binding has an explicit `id` unique within its client. IDs are stable
  diagnostic and audit identities; array position has no identity or precedence.
- The effective configuration must contain a non-empty `subject` and `groups`.

Fragments may use claim-value validation and applicator keywords such as
`type`, `const`, `enum`, `pattern`, `not`, `allOf`, `anyOf`, and `contains`.
`$ref`, remote loading, custom vocabularies, and content keywords are rejected.
Provider presets reject unknown claim names; generic OIDC policies permit them.

### Schema compilation and evaluation

Easy OIDC uses `github.com/santhosh-tekuri/jsonschema/v6` with JSON Schema Draft
2020-12 and its default Go/RE2 regular-expression engine. Patterns are
unanchored unless explicitly anchored with `^` and `$`.

At startup, Easy OIDC resolves every trust binding. It overlays ordinary claim
fragments by claim name, retains all required fragments, generates one object
schema whose `required` list contains every configured claim, and compiles that
schema once. If required and ordinary fragments constrain the same claim, the
generated property uses `allOf` so both must pass. Invalid schemas fail startup.

The compiler must use a deny-all external loader, pin the dialect, and treat
compiled schemas as immutable. Easy OIDC bounds schema size and depth, JWT and
decoded-claim size, string length, collection size, and composition count
before evaluation. Validation diagnostics are bounded and never returned by the
live token endpoint.

After cryptographic verification, Easy OIDC selects compiled bindings for the
target client and verified issuer and evaluates the verified claim object
against every candidate schema. Exactly one must match. Zero or multiple
matches deny the exchange; binding order has no meaning.

The configured `subject` becomes the downstream token's `sub` claim and must
begin with `trusted:`. This prevents collision with Easy OIDC's email-subject
interactive identities while allowing the trusted external token to represent
a human, service, or pipeline. Easy OIDC also includes the verified upstream
issuer and subject as informational `upstream_issuer` and `upstream_subject`
claims.

Groups are assigned directly by policy or binding instead of using the
email-keyed `groups_overrides` mechanism. Kubernetes remains authoritative for
what those groups may do.

## Trust Policy Testing

Operators can test a real external token through the complete verification and
matching path:

```sh
easy-oidc trust test \
  --config config.jsonc \
  --client-id cluster-production \
  --token-file github-token.jwt
```

The command reports issuer and standard-claim verification, every candidate
binding's match or bounded validation failure, and the effective subject and
groups when exactly one binding matches. It never accepts a token directly on
the command line or prints token material. `--token-file -` reads the token from
standard input:

```sh
cat github-token.jwt | easy-oidc trust test \
  --config config.jsonc \
  --client-id cluster-production \
  --token-file -
```

File and standard-input forms perform identical full token verification and use
the same compiled effective schemas and exactly-one-match evaluator as `/token`.
The command exits successfully only for exactly one match.

## Issuer Verification

The `github` and `buildkite` presets pin their official issuer URLs,
discovery behavior, acceptable signing algorithms, required standard claims,
and known provider claims. Generic `oidc` requires an HTTPS `issuer_url`; HTTP
is allowed only for local development. It also requires an asymmetric
`signing_algs` allowlist and positive `max_token_age`; provider presets supply
secure values.

For every external token, Easy OIDC must:

- require discovery metadata `issuer` to exactly equal the configured issuer;
- fetch JWKS only from that validated metadata's HTTPS `jwks_uri`, without
  cross-origin redirects;
- verify the signature with an explicitly allowed asymmetric algorithm;
- require non-empty `iss`, `sub`, `aud`, `exp`, and `iat`, and validate `nbf`
  when present;
- require `aud` to identify only the target Easy OIDC client: it must be either
  that client ID as a string or a one-element array containing it;
- reject tokens intended for multiple audiences;
- if the token includes an `azp` (authorized party) claim, require it to contain
  the same client ID as `aud`; a different value rejects the token;
- enforce the configured maximum token age;
- refresh JWKS once for an unknown key ID without accepting an unverified token;
- evaluate policies only after cryptographic and standard-claim validation; and
- use hard HTTP deadlines and fail closed when discovery or JWKS is unavailable.

Podplane asks GitHub or Buildkite to issue the external token for the target
Easy OIDC client ID. Easy OIDC requires that audience to match the `client_id`
in the exchange request. For example, a token issued for `cluster-production`
cannot be exchanged for `cluster-staging`, preventing reuse across clients.

## Token Exchange

Easy OIDC supports RFC 8693 token exchange at `/token`:

```text
grant_type=urn:ietf:params:oauth:grant-type:token-exchange
subject_token=<external OIDC JWT>
subject_token_type=urn:ietf:params:oauth:token-type:id_token
requested_token_type=urn:ietf:params:oauth:token-type:id_token
client_id=<target Easy OIDC client ID>
```

Successful exchange returns:

```jsonc
{
  "access_token": "<Easy OIDC ID token>",
  "issued_token_type": "urn:ietf:params:oauth:token-type:id_token",
  "token_type": "Bearer",
  "expires_in": 900
}
```

As specified by RFC 8693, `access_token` is the response container for the
requested security token even when that token is an ID token. Easy OIDC returns
no separate `id_token` field. The short-lived ID token contains the effective
binding's `sub` and `groups`, the target client as audience, a unique `jti`, and
upstream provenance claims. The exchange returns no refresh token or `sid`;
another external token must be exchanged when renewal is needed. Invalid or
unacceptable subject tokens return RFC 8693 `invalid_request`. All token
responses use OAuth JSON errors, `Cache-Control: no-store`, and
`Pragma: no-cache`.

Repeated exchange of a still-valid external token is permitted. Easy OIDC and
Podplane's provider-acquisition modes must never persist or log that bearer
credential. Caller-managed token-file mode is the explicit persistence
exception. Easy OIDC logs the issuer, client, matched policy/binding, effective
subject, provider run/job identifiers when available, and result.

Discovery advertises the token-exchange grant in `grant_types_supported`.

## Podplane Login Experience

With `podplane.cluster.jsonc` in the working directory, the normal flow is:

```sh
podplane login
podplane deploy web --name app --image ghcr.io/acme/app:latest
```

Login mode precedence is:

1. Explicit provider or token file.
2. A recognized CI environment.
3. Interactive authorization code with PKCE.

Supported controls are:

```sh
podplane login --ci-provider github-actions
podplane login --ci-provider buildkite
podplane login --oidc-token-file token.jwt
podplane login --no-ci
```

GitHub detection requires `GITHUB_ACTIONS=true` and the Actions OIDC token
request variables. Buildkite detection requires `BUILDKITE=true` and a usable
`buildkite-agent`. Generic `CI=true` is insufficient because it provides no
standard token acquisition mechanism. Explicit flags override detection and
conflicting recognized environments fail with an actionable error.

Podplane requests the cluster's Easy OIDC client ID as the external token
audience. For Buildkite it also requests immutable organization and pipeline ID
claims needed by the preset. The generic token-file path accepts a JWT acquired
by another tool and never accepts the token directly on the command line.

CI login must not depend on an interactive OS keyring. The kubectl exec hook
reacquires an external CI token and exchanges it when its Easy OIDC token needs
renewal; it stores neither token persistently. Non-secret metadata records the
authentication mode needed by the hook. For caller-managed token-file mode, the
hook rereads the restrictively permissioned file before every exchange. A
long-running producer must atomically replace it before expiry; otherwise
renewal fails and a new token or login is required.

Podplane clusters using trusted-token identities must configure Kubernetes to
use `sub` as its username claim with an explicitly empty username prefix. This
preserves existing Easy OIDC human usernames because their `sub` is already
their normalized email, while allowing namespaced trusted subjects. New
Podplane/Easy OIDC clusters should use both settings; an incompatible username
claim or prefix makes CI login fail before changing kubeconfig.

No additional trust policy belongs in `podplane.cluster.jsonc`. It continues to
provide the Easy OIDC issuer and client ID; Easy OIDC owns issuers, policies,
trust bindings, subjects, and groups.

## Production Guidance

Production GitHub bindings should require immutable organization and repository
IDs, an exact approved workflow or reusable workflow on a protected ref, and a
protected GitHub environment where applicable. Branch name alone is
insufficient. Policies should exclude untrusted pull-request execution unless a
separately constrained preview identity is intentional.

Buildkite bindings should prefer immutable organization and pipeline IDs and
constrain the deployment branch and step key. Both providers should use
dedicated, least-privilege Kubernetes groups rather than cluster administration.

## Implementation Plan

1. **Easy OIDC configuration**
   - **Repository:** `easy-oidc/easy-oidc`
   - Add issuer, policy, per-claim schema, and trust-binding types and update the
     Easy OIDC configuration JSON Schema.
   - Add `github.com/santhosh-tekuri/jsonschema/v6`, pin Draft 2020-12, disable
     external loading, and compile generated effective schemas at startup.
   - Implement inheritance and effective-binding validation, including
     non-overridable required fragments, provider-known claims, unique IDs,
     `trusted:` subjects, effective groups, schema/claim limits, and startup
     rejection of invalid configuration.
   - Test generated `required` properties, fragment overrides, required-fragment
     composition, prohibited keywords/loaders, limits, and schema errors.
   - Benchmark 1, 10, 100, and 1,000 candidate schemas and set a validated
     per-client/issuer binding limit before release.

2. **External issuer verification**
   - **Repository:** `easy-oidc/easy-oidc`
   - Add bounded discovery/JWKS clients and generic JWT verification.
   - Add GitHub Actions and Buildkite presets with pinned issuer details and
     provider claim validation.
   - Test discovery issuer mismatch and redirects, issuer/audience confusion,
     algorithm substitution, multi-audience tokens, expiry/age, unknown keys,
     malformed claims, outages, and key rotation.

3. **Policy and token exchange**
   - **Repository:** `easy-oidc/easy-oidc`
   - Add the RFC 8693 grant to `/token` and discovery metadata.
   - Evaluate every compiled candidate schema for the target client and verified
     issuer, require exactly one match, and issue one short-lived ID token with
     explicit subject, groups, provenance, audience, and `jti` in the standard
     RFC 8693 `access_token` response field.
   - Add OAuth error/no-store behavior and security-safe structured logging.
   - Test strict JSON types, schema composition, zero/ambiguous matches,
     cross-client audiences, bounded diagnostics, and absence of refresh tokens.
   - Update Easy OIDC configuration and security documentation, then run the
     repository's full checks.

4. **Trust policy testing**
   - **Repository:** `easy-oidc/easy-oidc`
   - Add `easy-oidc trust test` with full token verification from a file or
     standard input using the production evaluator.
   - Report bounded per-binding diagnostics, effective subject/groups, and
     exactly-one-match exit status without exposing token material.
   - Test file/stdin input, valid, invalid, zero-match, ambiguous-match, and
     redaction behavior.

5. **Podplane acquisition and login**
   - **Repository:** `podplane/podplane`
   - Add GitHub and Buildkite environment detection and token acquisition behind
     a generic external-token exchange interface.
   - Add `--ci-provider`, `--oidc-token-file`, and `--no-ci` with the precedence
     and conflict behavior above.
   - Extend auth metadata and the kubectl exec hook for non-persistent CI token
     reacquisition and exchange; redact all token material from errors and logs.
   - Test provider reacquisition, atomically rotated token files, and expired
     static token files after both upstream and downstream token expiry.

6. **Kube-apiserver username configuration**
   - **Repository:** `podplane/vmconfig`
   - Accept the OIDC username claim and prefix through vmconfig's stable input
     contract and configure kube-apiserver accordingly.
   - Test `sub` with an explicitly empty prefix, preserve existing defaults for
     callers that omit the new inputs, and document the contract.

7. **Podplane cluster integration and verification**
   - **Repository:** `podplane/podplane`
   - Add cluster configuration and schema support for the username prefix, use
     `sub` with an empty prefix for new Easy OIDC clusters, and pass both values
     through the vmconfig input contract.
   - Reject trusted-token login against incompatible clusters and document
     existing-cluster migration impact.
   - Add GitHub Actions and Buildkite end-to-end tests covering login followed by
     `podplane deploy`, renewal, denied claims, wrong audiences, and ambiguous
     bindings.
   - Update Podplane CI, login, cluster configuration, and RBAC documentation,
     then run the repository's full checks.
