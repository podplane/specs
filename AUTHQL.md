# PostgreSQL Policy Database

> **STATUS**: Implemented

> **Dependency:** This design builds on the external OIDC trust model in
> [TRUST.md](./TRUST.md).

This document specifies the PostgreSQL policy database for Easy OIDC. It lets an
application database supplement static client, group override, trust policy, and
trust binding configuration while preserving Easy OIDC's protocol validation and
policy evaluation.

## Goals

- Let an application database remain authoritative for clusters, users, group
  assignments, trust policies, and their relationships.
- Use operator-configured, parameterized SQL rather than prescribe an application
  database schema.
- Keep redirect and common client settings in Easy OIDC configuration so client
  lookup is only an existence check.
- Reuse TRUST.md's JSON Schema compiler and exactly-one-binding evaluator.
- Keep policy reads low-latency, bounded, observable, and fail-closed.
- Allow a read-only PostgreSQL role and replica without coupling policy queries
  to Easy OIDC's state database.

Webhooks, database writes, dynamic issuer registration, and requiring applications
to adopt a prescribed schema are out of scope.

## Conceptual Data Model

An application may model auth policy approximately as:

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

`policy_database` may coexist with static `clients`, `groups_overrides`, trust
policies, and client trust bindings. Trusted issuer definitions remain static
under `oidc_trust.issuers`; database rows reference those configured issuer IDs
and cannot introduce a new issuer.

```jsonc
{
  "oidc_trust": {
    "issuers": {
      "github-actions": { "provider": "github" },
      "buildkite": { "provider": "buildkite" }
    }
  },

  "policy_database": {
    "driver": "postgresql",
    "connection_string_secret": "EASYOIDC_POLICY_DB_URL",

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
      "user_access": "SELECT allowed, groups FROM resolve_oidc_user($1, $2)",
      "trust_bindings": "SELECT ... WHERE c.oidc_client_id = $1 AND i.issuer_id = $2"
    },

    "client_lookup_cache": {
      "ttl": "5m",
      "negative_ttl": "30s",
      "max_entries": 10000
    },

    "policy_build_cache": {
      "max_entries": 10000
    },

    "query_timeout": "500ms",
    "max_connections": 4,
    "max_trust_rows": 100,
    "max_groups": 100,
    "max_group_bytes": 256,
    "max_json_bytes": 65536
  }
}
```

All clients supplied by database policy share `redirect_uris` and
`client_defaults`. Returning per-client settings is out of scope; the contract
deliberately keeps the pre-authentication query to a cacheable existence check.

### Client policy selection

Easy OIDC first looks up the client ID in static configuration. If present, that
client uses only static policy. Otherwise, Easy OIDC asks the policy database
whether the client exists and, if so, uses only database policy for the client.
It never falls through to PostgreSQL because a static user or trust binding was
denied. Static client IDs therefore have unconditional precedence and incur no
database query.

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

### User access

`user_access` receives:

```text
$1 = Easy OIDC client ID
$2 = normalized downstream user subject (email)
```

It returns exactly one row with non-null `allowed` boolean and `groups` text
array columns. The query must return `allowed=false` and an empty group array for
an unknown or disabled client and for an unknown, disabled, or denied subject.
When true, Easy OIDC validates, deduplicates, and sorts groups; an empty array
follows the effective `require_groups` policy. Missing, duplicate, malformed, or
null results are indeterminate resolver failures.

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

These flows describe a client supplied by database policy after static client
lookup has failed and `policy_database` has claimed the client ID. Static clients
use their static flows and execute none of these queries.

### Interactive login and refresh

```text
1. client_exists before accepting /authorize or trusting its redirect URI
2. client_exists + user_access after identity verification, before code
3. client_exists + user_access again at authorization-code redemption
4. client_exists + user_access on every refresh before rotation/issuance
```

These checks use the identity stored in the flow/code/grant and occur before
issuing an authorization code, consuming a code, creating a refresh grant,
rotating a refresh token, or signing new tokens. A cached positive client result
may retain the documented TTL delay; stored flow state and code lifetime do not
extend it.

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

User access and trust binding query results are not cached. This keeps grants and
revocations as fresh as the selected PostgreSQL server. Easy OIDC uses a bounded
LRU policy-build cache whose values contain only immutable compiled schemas. The
key is a cryptographic digest of the canonical generated effective schema,
namespaced by client/issuer/binding for diagnostics. Every exchange still
executes `trust_bindings`; subject and normalized groups always come from the
current query result, and removed bindings are never read from the cache.

A read replica is supported, but replica lag delays database policy changes.
User/group/trust revocation delay is replica lag; client revocation delay is
replica lag plus up to the positive cache TTL. Operators needing a bounded
revocation window must use the primary, synchronous replication, or a replica
endpoint with an enforced lag bound.

## Database and Failure Safety

- Use a dedicated connection pool even if Easy OIDC later uses PostgreSQL for its
  state database or both pools connect to the same server.
- Load the connection string through Easy OIDC's secrets provider.
- Require TLS except for localhost development.
- Require a database role that cannot write application data; SQL inspection is
  not a sufficient read-only boundary.
- Set PostgreSQL read-only session defaults, a hard statement timeout, bounded
  connection acquisition, and a small independent pool.
- Bound returned rows, JSON bytes, group count/length, and schema complexity
  before allocation or compilation.
- Distinguish definitive policy results from indeterminate resolver failures as
  specified below; never retain or silently fall back to stale policy.
- Emit structured `policy database query` logs containing query name, duration,
  row count, cache result, client ID, issuer ID, and outcome without logging
  credentials, raw tokens, SQL, errors, or sensitive subjects.

The PostgreSQL driver may be shared with a future PostgreSQL state database, but
pool configuration, credentials, health, and failure handling remain separate.

### Failure classification

A well-formed `exists=false`, `allowed=false`, empty groups when groups are
required, or valid zero/ambiguous trust match is a definitive policy denial. On
refresh, definitive loss of access revokes the grant and returns
`invalid_grant` only after revocation commits.

Timeout, cancellation, connection/acquisition failure, missing/duplicate/null
contract rows, decoding error, client/issuer mismatch, invalid dynamic schema,
or result/complexity limit violation is indeterminate. Easy OIDC issues nothing,
returns a non-detailing temporary server error appropriate to the endpoint,
does not negative-cache the result, and does not consume or revoke an
authorization code or refresh grant. The existing credential remains retryable.

## Implementation and Verification

The implementation is owned by `github.com/easy-oidc/easy-oidc`.

1. **Auth policy resolution and configuration**
   - `internal/authpolicy` exposes source-agnostic client, user, and trust
     resolution to consumers while keeping static and PostgreSQL policy handling
     separate.
   - Static and database policy coexist with deterministic static-first client
     selection. Static clients never query PostgreSQL.
   - Configuration, defaults, validation, and the public JSON Schema use
     `policy_database`, `driver: postgresql`, `user_access`,
     `client_lookup_cache`, and `policy_build_cache`.

2. **PostgreSQL policy database**
   - Easy OIDC loads credentials through its secrets provider and creates an
     independent, bounded, read-only pgx pool.
   - Policy queries use deadlines and strict typed decoding with row, group,
     JSON, wire-size, and schema-complexity limits.
   - Unit and PostgreSQL integration tests cover TLS requirements, read-only
     operation, pool isolation and exhaustion, timeout, cancellation, malformed
     and partial results, excess results, and database outages.

3. **Client lookup**
   - After static lookup fails, the resolver checks policy database client
     existence before redirects and exchanges and applies shared client defaults.
   - Bounded positive and negative caches coalesce concurrent misses and never
     use expired values when PostgreSQL is unavailable. Credential issuance uses
     a fresh lookup.
   - Tests cover exact redirects, unknown clients, expiry, eviction, concurrent
     misses, definitive false, malformed results, and failures around expiry.

4. **User access and groups**
   - Easy OIDC resolves current user access and groups after identity
     verification, before code issuance and redemption, and during every refresh.
   - Groups are validated, deduplicated, sorted, bounded, and evaluated under the
     effective `require_groups` setting.
   - Tests cover unknown users, empty optional and required groups, membership
     changes, removal during the flow, refresh re-evaluation, malformed groups,
     and retry-safe query failures.

5. **Dynamic trust bindings**
   - PostgreSQL rows are decoded into TRUST.md policy and binding inputs,
     validated, overlaid, compiled, and evaluated with exactly-one-match
     semantics.
   - Only immutable compiled schemas are cached in a bounded LRU keyed by their
     canonical digest. Subjects, groups, and binding existence always come from
     the current query snapshot.
   - `easy-oidc trust test` resolves both static and database policy through the
     same auth policy resolver.
   - Tests cover mismatches, policy and group updates, binding removal, cache
     invalidation and eviction, zero and ambiguous matches, invalid schemas,
     transient retry, bounds, and lagging then current replica snapshots.

6. **Documentation, benchmarks, and end-to-end coverage**
   - Easy OIDC documents the query contracts, default schema and queries,
     read-only role creation, TLS, replicas, cache and revocation tradeoffs, and
     operational monitoring in `docs/policy-database.md`.
   - `examples/policy-db/postgresql.sql` provides a ready-to-use PostgreSQL schema
     compatible with the built-in queries.
   - End-to-end coverage runs static flows first, then PostgreSQL-backed
     interactive login, refresh, token exchange, and trust testing.
   - Benchmarks cover cached client lookup, group normalization, trust queries,
     the default result ceilings, unchanged schema reuse, and changed schema
     compilation.
   - The repository unit suite, PostgreSQL integration coverage, end-to-end suite,
     formatting, module checks, and linter pass.
