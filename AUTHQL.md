# AuthQL PostgreSQL Authorization Source

> **Dependency:** This design depends on the external OIDC trust model in
> [TRUST.md](./TRUST.md). Implement and verify TRUST.md before starting "AuthQL".

This document specifies a PostgreSQL-backed authorization source for Easy OIDC.
It lets an application database supplement static client, group override, trust
policy, and trust binding configuration while preserving Easy OIDC's protocol
validation and policy evaluation.

## Goals

- Let an application database remain authoritative for clusters, users, group
  assignments, trust policies, and their relationships.
- Use operator-configured, parameterized SQL rather than prescribe an application
  database schema.
- Keep redirect and common client settings in Easy OIDC configuration so client
  lookup is only an existence check.
- Reuse TRUST.md's JSON Schema compiler and exactly-one-binding evaluator.
- Keep authorization reads low-latency, bounded, observable, and fail-closed.
- Allow a read-only PostgreSQL role and replica without coupling authorization
  queries to Easy OIDC's primary storage.

Webhooks, database writes, dynamic issuer registration, and a prescribed
application schema are out of scope initially.

## Conceptual Data Model

An application may model authorization approximately as:

```text
clusters
 - id

users
 - id
 - email

cluster_users
 - cluster_id
 - user_id
 - groups text[]

trust_policies
 - ...

cluster_trust_bindings
 - cluster_id
 - trust_policy_id
 - subject
 - claims_override jsonb
 - groups text[]
```

A trust binding is an explicit entity because the cluster-to-policy relationship
owns the downstream subject, claim overlay, and groups:

```text
trust policy
     │
     ▼
cluster trust binding ───▶ subject
     │
     └───────────────────▶ groups
```

Kubernetes group names are opaque strings, so illustrative implementations may
store them directly as `text[] NOT NULL DEFAULT '{}'` on each relationship.
This intentionally forgoes independent group metadata, foreign keys, and central
renaming. Easy OIDC still validates, bounds, deduplicates, and sorts returned
group strings. The SQL query contract, not these table names, columns, or
relationships, is normative.

## Configuration

`authorization_source` may coexist with static `clients`, `groups_overrides`,
trust policies, and client trust bindings. Trusted issuer definitions remain
static under `oidc_trust.issuers`; database rows reference those configured
issuer IDs and cannot introduce a new issuer.

```jsonc
{
  "oidc_trust": {
    "issuers": {
      "github-actions": { "provider": "github" },
      "buildkite": { "provider": "buildkite" }
    }
  },

  "authorization_source": {
    "type": "postgres",
    "connection_string_secret": "authorization-database-url",

    "redirect_uris": [
      "http://localhost:8000/callback"
    ],

    "client_defaults": {
      "require_groups": true,
      "refresh_tokens": {
        "enabled": true,
        "allow_offline_access": false
      }
    },

    "queries": {
      "client_exists": "SELECT EXISTS (SELECT 1 FROM clusters WHERE oidc_client_id = $1 AND enabled) AS exists",
      "user_authorization": "SELECT authorized, groups FROM resolve_oidc_user($1, $2)",
      "trust_bindings": "SELECT ... WHERE c.oidc_client_id = $1 AND i.issuer_id = $2"
    },

    "client_cache": {
      "ttl": "5m",
      "negative_ttl": "30s",
      "max_entries": 10000
    },

    "schema_cache": {
      "max_entries": 10000
    },

    "query_timeout": "500ms",
    "max_connections": 4
  }
}
```

All database-backed clients share `redirect_uris` and `client_defaults`. A future
extension may return per-client settings, but the initial contract deliberately
keeps the pre-authentication query to a cacheable existence check.

### Client source selection

Easy OIDC first looks up the client ID in static configuration. If present, that
client uses only static authorization configuration. Otherwise, Easy OIDC asks
`authorization_source` whether the client exists and, if so, uses only that
source for the client. It never falls through to PostgreSQL because a static
user or trust binding was denied. Static client IDs therefore have unconditional
precedence and incur no database query.

## Query Contracts

Queries are trusted configuration and use PostgreSQL positional parameters.
Easy OIDC prepares them and binds all values; it never interpolates client IDs,
subjects, issuers, or token claims into SQL.

### Client existence

`client_exists` receives:

```text
$1 = Easy OIDC client ID
```

It returns exactly one row with one non-null boolean column named `exists`.
Easy OIDC runs this before redirecting an interactive user or accepting a token
exchange. `exists=false` means unknown client. Missing, duplicate, malformed, or
null results are indeterminate resolver failures.

### User authorization

`user_authorization` receives:

```text
$1 = Easy OIDC client ID
$2 = normalized downstream user subject (email)
```

It returns exactly one row with non-null `authorized` boolean and `groups` text
array columns. The query must return `authorized=false` for an unknown or
disabled client and for an unknown, disabled, or unauthorized subject. When
true, Easy OIDC validates, deduplicates, and sorts groups; an empty array follows
the effective `require_groups` policy. Missing, duplicate, malformed, or null
results are indeterminate resolver failures.

### Trust bindings

`trust_bindings` receives:

```text
$1 = Easy OIDC client ID
$2 = verified configured issuer ID, such as github-actions
```

It returns zero or more rows with these columns:

| Column | Type | Meaning |
| --- | --- | --- |
| `client_id` | text | Must exactly equal the `$1` request value |
| `issuer_id` | text | Must exactly equal the `$2` verified issuer ID |
| `binding_id` | text | Stable identifier unique within the client |
| `subject` | text | Downstream subject beginning with `trusted:` |
| `required_claims` | JSON/JSONB object | Non-overridable claim schema fragments |
| `policy_claims` | JSON/JSONB object | Ordinary policy claim schema fragments |
| `binding_claims` | JSON/JSONB object | Binding overlay keyed by claim name |
| `groups` | text array | Downstream groups for this binding |

The query should join the application's policy and cluster binding tables in one
statement, reading groups directly from the binding, and return bindings only
for an active client. Each response is therefore one PostgreSQL statement
snapshot. Easy OIDC applies TRUST.md inheritance: `binding_claims` override
`policy_claims` by claim name, while every `required_claims` fragment remains.

Every row is validated and bounded before use. A mismatched client/issuer,
duplicate binding ID, invalid subject/group, malformed schema fragment, excess
binding, or partial result is an indeterminate resolver failure. Cache and
diagnostic identities are namespaced by client ID, issuer ID, and binding ID.

## Runtime Flows

These flows describe a database-backed client after static client lookup has
failed and `authorization_source` has claimed the client ID. Static clients use
their static flows and execute none of these queries.

### Interactive login and refresh

```text
1. client_exists before accepting /authorize or trusting its redirect URI
2. client_exists + user_authorization after identity verification, before code
3. client_exists + user_authorization again at authorization-code redemption
4. client_exists + user_authorization on every refresh before rotation/issuance
```

These checks use the identity stored in the flow/code/grant and occur before
issuing an authorization code, consuming a code, creating a refresh grant,
rotating a refresh token, or signing new tokens. A cached positive client result
may retain the documented TTL delay; authorization state and code lifetime do
not extend it.

### External token exchange

```text
client_exists(client ID), usually cached
          │
          ▼
verify external token and configured issuer
          │
          ▼
trust_bindings(client ID, verified issuer ID)
          │
          ▼
compile/cache effective schemas and evaluate every binding
          │
          ├─ zero or multiple matches: deny
          ▼
exactly one match: issue its subject and groups
```

Trust groups come from the matched binding query row; there is no second groups
query for that exchange.

## Caching and Freshness

Client existence is heavily cached because it changes infrequently and is used
before every flow. Positive and negative entries have separate bounded TTLs and
the cache has a size limit and request coalescing. Entries are never used after
expiry when PostgreSQL is unavailable.

User authorization and trust binding query results are not cached initially.
This keeps grants and revocations as fresh as the selected PostgreSQL server.
Easy OIDC uses a bounded LRU cache whose values contain only immutable compiled
schemas. The key is a cryptographic digest of the canonical generated effective
schema, namespaced by client/issuer/binding for diagnostics. Every exchange
still executes `trust_bindings`; subject and normalized groups always come from
the current query result, and removed bindings are never read from the cache.

A read replica is supported, but replica lag delays authorization changes.
User/group/trust revocation delay is replica lag; client revocation delay is
replica lag plus up to the positive cache TTL. Operators needing a bounded
revocation window must use the primary, synchronous replication, or a replica
endpoint with an enforced lag bound.

## Database and Failure Safety

- Use a dedicated connection pool even if Easy OIDC later uses PostgreSQL for
  primary storage or both pools connect to the same server.
- Load the connection string through Easy OIDC's secrets provider.
- Require TLS except for localhost development.
- Require a database role that cannot write application data; SQL inspection is
  not a sufficient read-only boundary.
- Set PostgreSQL read-only session defaults, a hard statement timeout, bounded
  connection acquisition, and a small independent pool.
- Bound returned rows, JSON bytes, group count/length, and schema complexity
  before allocation or compilation.
- Distinguish definitive policy results from indeterminate resolver failures as
  specified below; never retain or silently fall back to stale authorization.
- Log query name, duration, row count, cache result, client ID, issuer ID, and
  outcome without logging credentials, raw tokens, or sensitive subjects.

The PostgreSQL driver may be shared with future PostgreSQL primary storage, but
pool configuration, credentials, health, and failure handling remain separate.

### Failure classification

A well-formed `exists=false`, `authorized=false`, empty groups when groups are
required, or valid zero/ambiguous trust match is a definitive policy denial. On
refresh, definitive loss of authorization revokes the grant and returns
`invalid_grant` only after revocation commits.

Timeout, cancellation, connection/acquisition failure, missing/duplicate/null
contract rows, decoding error, client/issuer mismatch, invalid dynamic schema,
or result/complexity limit violation is indeterminate. Easy OIDC issues nothing,
returns a non-detailing temporary server error appropriate to the endpoint,
does not negative-cache the result, and does not consume or revoke an
authorization code or refresh grant. The existing credential remains retryable.

## Implementation Plan

1. **Authorization source abstraction and configuration**
   - **Repository:** `easy-oidc/easy-oidc`
   - Complete TRUST.md first, then define client existence, user authorization,
     and trust binding resolver contracts around its compiled policy types.
   - Allow static and PostgreSQL sources to coexist; add deterministic
     static-first client source selection, client defaults, query text, cache
     settings, pool limits, and JSON Schema.
   - Validate durations, limits, redirect URIs, SQL presence, configured static
     issuers, and source configuration at startup; validate dynamic
     issuer/client columns at query time.

2. **PostgreSQL query runtime**
   - **Repository:** `easy-oidc/easy-oidc`
   - Add the PostgreSQL driver or reuse the selected primary-storage driver.
   - Load credentials, create the independent read-only pool, prepare queries,
     apply deadlines, and implement strict typed result decoding and limits.
   - Test TLS requirements, read-only operation, pool isolation, timeout,
     cancellation, malformed results, excess results, and database outages.

3. **Client existence and cache**
   - **Repository:** `easy-oidc/easy-oidc`
   - After static client lookup fails, resolve database client existence before
     redirect or exchange and apply shared redirect/client defaults.
   - Add bounded positive/negative caches with request coalescing and no stale
     use after expiry.
   - Test static clients never query PostgreSQL, plus unknown/disabled database
     clients, exact redirect matching, expiry, eviction, concurrent misses,
     definitive false, malformed results, and database failure before and after
     cache expiry.

4. **User authorization**
   - **Repository:** `easy-oidc/easy-oidc`
   - Resolve explicit authorization and groups after identity verification,
     before code issuance/redemption, and during every refresh, replacing static
     `groups_overrides` behavior in PostgreSQL mode.
   - Validate, deduplicate, sort, bound, and issue groups under current
     `require_groups` semantics.
   - Test unknown users with `require_groups=false`, membership grants/removals,
     removal between authorize/callback/code redemption, empty groups, refresh
     re-evaluation, malformed groups, and query failure without code/grant loss.

5. **Dynamic trust binding authorization**
   - **Repository:** `easy-oidc/easy-oidc`
   - Decode trust rows into TRUST.md policy/binding inputs, validate and overlay
     fragments, generate effective object schemas, and require exactly one match.
   - Cache only immutable compiled schemas in a bounded LRU keyed by canonical
     schema digest while using current query subjects/groups on every exchange.
   - Extend `easy-oidc trust test` to use PostgreSQL authorization sources.
   - Test client/issuer mismatch, policy and group updates, binding removal,
     digest invalidation/eviction, zero/ambiguous matches, invalid schemas,
     transient retry, bounds, and replica lag behavior.

6. **Documentation and end-to-end verification**
   - **Repository:** `easy-oidc/easy-oidc`
   - Document query contracts, example schemas/queries, read-only role creation,
     TLS, replicas, cache/revocation tradeoffs, and operational monitoring.
   - Add PostgreSQL-backed interactive login, refresh, token exchange, and trust
     test end-to-end coverage.
   - Benchmark cached client lookup, user groups, trust query, unchanged schema
     reuse, and changed schema compilation, then set validated defaults/limits.
   - Run the repository's full checks.
