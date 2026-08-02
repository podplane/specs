# Shared State Database

> **STATUS**: Draft

This document specifies an optional external shared state database for Easy
OIDC. Its primary purpose is to let multiple Easy OIDC replicas safely serve
the same issuer at higher scale. It is distinct from the read-only policy
database in [AUTHQL.md](./AUTHQL.md).

## Goals

- Replace local SQLite state with shared PostgreSQL state when configured.
- Preserve all existing single-use, replay, rotation, revocation, and encryption
  guarantees across concurrent replicas.
- Keep SQLite as the zero-configuration default.
- Provide explicit, versioned, and concurrency-safe schema migration.
- Leave room for a future MySQL driver without claiming SQL portability.

Live database switching, automatic SQLite-to-PostgreSQL data migration,
multi-region active-active operation, and coupling state to the policy database
are out of scope.

## Configuration

When `state_database` is absent, Easy OIDC continues to use
`data_dir/easy-oidc-state.db`. When present, Easy OIDC stores all protocol state
in the configured database and does not open the SQLite state file.

```jsonc
"state_database": {
  "driver": "postgresql",
  "connection_string_secret": "EASYOIDC_STATE_DB_URL",
  "max_connections": 16,
  "query_timeout": "5s",
  "migrations": {
    "connection_string_secret": "EASYOIDC_STATE_DB_MIGRATION_URL"
  }
}
```

The initial implementation accepts only `driver: postgresql`. The driver field
is retained so a future MySQL implementation does not require a configuration
shape change. `max_connections` defaults to `16`; `query_timeout` defaults to
`5s`. Both must be positive and are validated at startup.

The connection string is loaded through the configured secrets provider. TLS is
required except for loopback development. `state_database` and `policy_database`
always use independent credentials, pools, schemas, migrations, and health
handling, even when they connect to the same PostgreSQL server.

## State and Concurrency

The shared database is authoritative for all mutable protocol state, including:

- authorization flows, codes, temporary encrypted credentials, and identity
  selections;
- OTP challenges, send limits, and verification records; and
- refresh grants, token families, processing claims, and grant actions.

The storage package exposes backend-independent operations rather than SQL to
protocol consumers. SQLite and PostgreSQL implementations must pass the same
behavioral and concurrency tests.

PostgreSQL transactions use row locking, conditional updates, unique constraints,
and foreign keys to preserve the current atomic contracts. In particular:

- a flow, code, selection, or action is consumed at most once;
- code redemption and initial grant creation are atomic;
- refresh claim acquisition, rotation, replay revocation, and family revocation
  remain atomic under concurrent requests from different replicas; and
- temporary failure never partially consumes a credential or loses a revocation.

Expired-state cleanup runs in bounded, idempotent batches. Multiple replicas may
run cleanup concurrently using row locking or conditional deletion; no application
leader election is required.

Every replica must use the same signing, encryption, OTP, and other shared secrets.
Startup fails if the state database is unavailable or its schema is incompatible.
Runtime database failures fail closed, and database readiness is reflected in the
service health signal without exposing connection details.

## Schema Migration

Easy OIDC never changes a shared schema while serving. Operators run migrations
explicitly before starting a new binary:

```console
easy-oidc migrate
```

The command reads the normal configuration and secrets, embeds migrations in the
binary, applies only migrations for the configured driver, and exits successfully
when no change is required. Server startup checks the schema version and refuses a
version outside the binary's supported range or a schema marked dirty.

Migration internals use `github.com/golang-migrate/migrate/v4` as a Go library.
Migrations are stored by driver, for example `migrations/postgresql`, and loaded
from `embed.FS` through its `iofs` source. The PostgreSQL driver provides advisory
locking, so accidentally concurrent migration jobs serialize without any custom
leader-election system.

Migrations are forward-only and use expand-and-contract changes that remain
compatible with the previous Easy OIDC release during a rolling deployment.
PostgreSQL migrations use explicit transactions when the statements support them
and isolate non-transactional operations in dedicated migrations.
`golang-migrate` records the current schema version and whether a migration was
left dirty. A failed migration leaves the schema dirty, and subsequent migrations
and server startup refuse to continue until the operator completes the documented
recovery procedure.

`state_database.migrations.connection_string_secret` is optional and inherits
`state_database.connection_string_secret` when omitted. Only the `migrate` command
loads the migration secret; the server never does. This lets simple deployments
use one role while hardened deployments give a separate migration role DDL
permissions and restrict the server role to required runtime DML operations.
The migration driver is always the `state_database` driver and cannot be
overridden.

A container entrypoint may run `easy-oidc migrate` before starting the server.
Concurrent entrypoints safely serialize through the database migration lock.

## Moving from SQLite

The initial release does not copy existing SQLite state. To move an installation:

1. Stop all Easy OIDC replicas.
2. Configure `state_database` and run `easy-oidc migrate`.
3. Start every replica with the same configuration and shared secrets.
4. Require users and clients with stateful grants to authenticate again.

The previous SQLite file is retired state and must never be reused as a rollback
target, because doing so could revive consumed codes or revoked grants. A rollback
to SQLite must use a new empty database.

## Implementation and Verification Plan

1. Change the default SQLite path to `data_dir/easy-oidc-state.db` with no
   compatibility lookup or migration of `easy-oidc.db`, then add
   `state_database` configuration, schema validation, secret loading, and
   PostgreSQL pool lifecycle. Verify PostgreSQL mode opens neither SQLite file.
2. Define a narrow state-store interface and preserve behavior while separating
   SQLite SQL from protocol consumers.
3. Add PostgreSQL schema and operations with transaction and concurrency tests
   shared across both drivers.
4. Add embedded driver-specific migrations and the explicit `migrate` command.
5. Test schema locking, version mismatch, dirty state, outages, pool exhaustion,
   cleanup races, and multi-replica code and refresh-token races against real
   PostgreSQL.
6. Add PostgreSQL-backed end-to-end login, refresh, revocation, restart, and
   horizontal-replica coverage; benchmark representative state operations and
   document sizing, backup, migration, and recovery procedures.
