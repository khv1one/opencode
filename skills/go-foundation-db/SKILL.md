---
name: go-foundation-db
description: a
---

# go-foundation Database Packages

Skill guide for `gitlab.wildberries.ru/reviews-ext/go-foundation` database connectivity packages.
Teaches AI agents (`godev`, `dbdev`) how to correctly initialize, configure, and use
PostgreSQL, ClickHouse, and Cassandra connections via the library.

---

## Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [PostgreSQL — `pkg/postgresdb`](#postgresql--pkgpostgresdb)
   - [Config](#postgresql-config)
   - [Initialization](#postgresql-initialization)
   - [Read/Write Pools](#postgresql-readwrite-pools)
   - [Retry & Recovery](#postgresql-retry--recovery)
   - [Sentinel Errors](#postgresql-sentinel-errors)
   - [Graceful Shutdown](#postgresql-shutdown)
3. [ClickHouse — `pkg/clickdb`](#clickhouse--pkgclickdb)
   - [Config](#clickhouse-config)
   - [Initialization](#clickhouse-initialization)
   - [Accessing the Connection](#clickhouse-accessing-the-connection)
   - [Graceful Shutdown](#clickhouse-shutdown)
4. [Cassandra — `pkg/cassandradb`](#cassandra--pkgcassandradb)
   - [Config](#cassandra-config)
   - [Consistency & Host Selection](#cassandra-consistency--host-selection)
   - [Initialization](#cassandra-initialization)
   - [Keyspace Management](#cassandra-keyspace-management)
   - [Graceful Shutdown](#cassandra-shutdown)
5. [Common Patterns](#common-patterns)
6. [Anti-Patterns](#anti-patterns)

---

## Architecture Overview

All three database packages follow the same lifecycle pattern:

```
New(config, logger) → *DB      // create struct, no I/O
DB.Prepare(ctx) → error        // connect, ping, validate
DB.Shutdown(ctx) → error       // close connections, cleanup
```

- **Constructor (`New`)**: allocates the struct only; does **not** open connections.
- **`Prepare(ctx)`**: establishes the connection pool/session, pings the server, and marks the DB as initialized.
- **`Shutdown(ctx)`**: closes pools/sessions and cancels internal contexts.

Two convenience constructors exist where init-on-create is desired:
- `postgresdb.NewWithInit(config, log)` — calls `Prepare` immediately.
- `clickdb.New(appName, config)` + `db.Init()` — calls `Prepare` with `context.Background()`.

---

## PostgreSQL — `pkg/postgresdb`

Uses `github.com/jackc/pgx/v5/pgxpool` with OpenTelemetry tracing via `github.com/exaring/otelpgx`.

### PostgreSQL Config

```go
type Config struct {
    AppName          string        `env:"APP_NAME" envDefault:"points" yaml:"app-name"`
    Host             string        `env:"HOST" envDefault:"localhost" yaml:"host"`
    DbName           string        `env:"NAME" envDefault:"postgres" yaml:"name"`
    User             string        `env:"USER" envDefault:"user" yaml:"user"`
    Password         string        `env:"PASSWORD" yaml:"password"`
    HostReplicas     string        `env:"HOST_REPLICAS" yaml:"host-replicas"`
    UserReplicas     string        `env:"USER_REPLICAS" yaml:"user-replica"`
    PasswordReplicas string       `env:"PASSWORD_REPLICAS" yaml:"password-replica"`
    MaxOpenConns     int32         `env:"MAX_OPEN_CONNS" envDefault:"10" yaml:"max-open-conns"`
    ConnIdleLifetime time.Duration `env:"CONN_IDLE_LIFETIME" envDefault:"10m" yaml:"conn-idle-lifetime"`
    ConnMaxLifetime  time.Duration `env:"CONN_MAX_LIFETIME" envDefault:"1h" yaml:"conn-max-lifetime"`
    MigrationsPath   string        `env:"MIGRATIONS_PATH" yaml:"migrations-path"`
}
```

**DSN methods**:
- `config.DsnPostgres()` — returns a key-value DSN for the primary (read-write) host.
- `config.DsnReplicasPostgres()` — returns a DSN for the replica host (only if `HostReplicas` is set).
- `config.UrlPostgres()` — returns a `postgres://` URL-style connection string.

All DSN methods include `sslmode=disable` and `application_name=<AppName>`.

### PostgreSQL Initialization

**Deferred init** (recommended for services with lifecycle hooks):

```go
import (
    "gitlab.wildberries.ru/reviews-ext/go-foundation/pkg/postgresdb"
    "gitlab.wildberries.ru/reviews-ext/go-foundation/pkg/logger"
)

db := postgresdb.New(cfg, log)
if err := db.Prepare(ctx); err != nil {
    return fmt.Errorf("postgres prepare: %w", err)
}
defer db.Shutdown(ctx)
```

**Eager init** (convenience, panics on failure if used without checking):

```go
db, err := postgresdb.NewWithInit(cfg, log)
if err != nil {
    return fmt.Errorf("postgres init: %w", err)
}
defer db.Shutdown(ctx)
```

### PostgreSQL Read/Write Pools

The `DB` struct maintains two `*pgxpool.Pool` instances:

| Method | Description |
|--------|-------------|
| `db.P()` | Returns the **read-write** (primary) pool. Auto-initializes if `Prepare` hasn't been called (panics on error). |
| `db.RP()` | Returns the **read-only** (replica) pool. Falls back to the primary pool if no replica is configured. Logs a warning. |
| `db.Pool(opts ...PoolOpt)` | Dispatches to `P()` or `RP()` based on options. |
| `db.Conn(ctx, opts ...)` | Acquires a single `*pgxpool.Conn` from the selected pool. |

**Selecting pool via options**:

```go
// Read from replica
conn, err := db.Conn(ctx, postgresdb.FromReplica())

// Read from primary (default)
conn, err := db.Conn(ctx, postgresdb.FromMaster())

// Or access pools directly
rows, err := db.P().Query(ctx, "SELECT ...")       // primary
rows, err := db.RP().Query(ctx, "SELECT ...")       // replica
```

**`Pooler` interface** — inject into services instead of the concrete `*DB`:

```go
type Pooler interface {
    Conn(ctx context.Context, opts ...PoolOpt) (*pgxpool.Conn, error)
}
```

### PostgreSQL Retry & Recovery

The package provides generic retry helpers for transient PostgreSQL errors (serialization failures `40001`, deadlock `40P01`).

**`Recovery`** — retry a function that returns only `error`:

```go
err := postgresdb.Recovery(ctx, func() error {
    return db.P().QueryRow(ctx, "SELECT ...").Scan(&val)
})
```

**`RecoveryWithResult`** — retry a function that returns `(T, error)`:

```go
result, err := postgresdb.RecoveryWithResult[MyType](ctx, func() (MyType, error) {
    return repo.GetByID(ctx, id)
})
```

**`RecoveryReadOnReplicasConflict`** — retry reads on replica conflicts (same signature as `RecoveryWithResult`):

```go
result, err := postgresdb.RecoveryReadOnReplicasConflict[MyType](ctx, func() (MyType, error) {
    return repo.GetByID(ctx, id)
})
```

**Customizing retry behavior**:

```go
// Override defaults (500ms timeout, 3 attempts)
err := postgresdb.Recovery(ctx, fn,
    postgresdb.WithTimeout(1*time.Second),
    postgresdb.WithLimit(5),
    postgresdb.WithErrCodeRecover(map[string]struct{}{
        "40001": {},
        "40P01": {},
    }),
)
```

**`IsErrRecoveryConflict(err)`** — check if an error is a retryable serialization/deadlock error.

### PostgreSQL Sentinel Errors

```go
var (
    ErrNotFound                    = errors.New("not found")
    ErrFailedToAcquireConn         = errors.New("failed to acquire connection")
    ErrFailedToBeginTransaction    = errors.New("failed to begin transaction")
    ErrFailedToCommitTransaction   = errors.New("failed to commit transaction")
    ErrFailedToRollbackTransaction = errors.New("failed to rollback transaction")
    ErrFailedToExecuteQuery        = errors.New("failed to execute query")
    ErrNotAllRowsAffected          = errors.New("number of rows affected is not equal to the number of entries")
)
```

Use these with `errors.Is()` for consistent error handling in repository code.

### PostgreSQL Shutdown

```go
if err := db.Shutdown(ctx); err != nil {
    log.Error("postgres shutdown", logger.Err(err))
}
```

`Shutdown` cancels the internal context and closes both pools (primary + replica).

---

## ClickHouse — `pkg/clickdb`

Uses `github.com/ClickHouse/clickhouse-go/v2`.

### ClickHouse Config

```go
type Config struct {
    Hosts           []string      `env:"HOSTS"`
    ConnMaxLifetime time.Duration `env:"CONN_MAX_LIFETIME" envDefault:"1h"`
    DialTimeout     time.Duration `env:"DIAL_TIMEOUT" envDefault:"30s"`
    ReadTimeout     time.Duration `env:"READ_TIMEOUT" envDefault:"180s"`
    MaxOpenConns    int           `env:"MAX_OPEN_CONNS" envDefault:"10"`
    MaxIddleConns   int           `env:"MAX_IDLE_CONNS" envDefault:"5"` // NOTE: typo in source — "Iddle"
    Name            string        `env:"NAME"`
    User            string        `env:"USER"`
    Password        string        `env:"PASSWORD"`
}
```

> **Note**: The field `MaxIddleConns` has a typo ("Iddle" instead of "Idle"). This is present in the source code and must be used as-is for env/yaml tag mapping.

### ClickHouse Initialization

```go
import (
    "gitlab.wildberries.ru/reviews-ext/go-foundation/pkg/clickdb"
)

db := clickdb.New("my-service", cfg)
if err := db.Prepare(ctx); err != nil {
    return fmt.Errorf("clickhouse prepare: %w", err)
}
defer db.Shutdown(ctx)
```

**Eager init shortcut**:

```go
db := clickdb.New("my-service", cfg)
if err := db.Init(); err != nil {
    return fmt.Errorf("clickhouse init: %w", err)
}
defer db.Shutdown(ctx)
```

`Init()` is a convenience wrapper around `Prepare(context.Background())`.

### ClickHouse Accessing the Connection

```go
conn := db.C() // returns clickhouse.Conn

// Example: batch insert
batch, err := conn.CreateBatch()
if err != nil { /* handle */ }
batch.Append(...) 
if err := batch.Send(); err != nil { /* handle */ }

// Example: query
rows, err := conn.Query(ctx, "SELECT ...")
```

### ClickHouse Shutdown

```go
// Preferred — with context
if err := db.Shutdown(ctx); err != nil {
    log.Error("clickhouse shutdown", logger.Err(err))
}

// Alternative — no context
db.Stop() // logs errors internally, does not return error
```

---

## Cassandra — `pkg/cassandradb`

Uses `github.com/gocql/gocql` with OpenTelemetry tracing via `pkg/telemetry/otelgocql`.

### Cassandra Config

```go
type Config struct {
    Hosts               []string            `env:"HOSTS" envDefault:"localhost:9042"`
    User                string              `env:"USER"`
    Password            string              `env:"PASSWORD"`
    Keyspace            string              `env:"KEYSPACE"`
    ReplicationFactor   int                 `env:"REPLICATION_FACTOR" envDefault:"3"`
    Consistency         Consistency         `env:"CONSISTENCY" envDefault:"QUORUM"`
    Connections         int                 `env:"CONNECTIONS" envDefault:"64"`
    RequestTimeout      time.Duration       `env:"REQUEST_TIMEOUT" envDefault:"10s"`
    ConnectTimeout      time.Duration       `env:"CONNECT_TIMEOUT" envDefault:"60s"`
    HostSelectionPolicy HostSelectionPolicy `env:"HOST_SELECTION_POLICY" envDefault:"tokenAware"`
    LocalDC             string              `env:"LOCAL_DC" envDefault:"dc1"`
    WriteThreads        int                 `env:"WRITE_THREADS" envDefault:"64"`
    ReadThreads         int                 `env:"READ_THREADS" envDefault:"64"`
}
```

### Cassandra Consistency & Host Selection

**Consistency levels** (type `Consistency`, string-based with `Value()` converter):

| String Value   | gocql Constant     |
|----------------|--------------------|
| `"ANY"`        | `gocql.Any`        |
| `"ONE"`        | `gocql.One`        |
| `"QUORUM"`     | `gocql.Quorum`     |
| `"TWO"`        | `gocql.Two`        |
| `"THREE"`      | `gocql.Three`      |
| `"ALL"`        | `gocql.All`        |
| `"LOCAL_QUORUM"` | `gocql.LocalQuorum` |
| `"EACH_QUORUM"` | `gocql.EachQuorum` |
| `"LOCAL_ONE"`  | `gocql.LocalOne`    |

> **Panics** on unknown consistency levels. Validate input at config time.

**Host selection policies** (type `HostSelectionPolicy`):

| Constant             | Value         | Behavior                                          |
|----------------------|---------------|---------------------------------------------------|
| `PolicyTokenAware`   | `"tokenAware"` | Token-aware with round-robin fallback (default)  |
| `PolicyRoundRobin`  | `"roundRobin"` | Pure round-robin                                  |
| `PolicyDcAware`     | `"dcAware"`    | Token-aware with DC-aware round-robin (dc1)       |

### Cassandra Initialization

```go
import (
    "gitlab.wildberries.ru/reviews-ext/go-foundation/pkg/cassandradb"
    "gitlab.wildberries.ru/reviews-ext/go-foundation/pkg/logger"
)

db := cassandradb.NewDB(cfg, log)
if err := db.Prepare(ctx); err != nil {
    return fmt.Errorf("cassandra prepare: %w", err)
}
defer db.Shutdown(ctx)
```

**Accessing the session**:

```go
session := db.Conn() // returns *gocql.Session

query := session.Query("SELECT ...", args...)
if err := query.Exec(); err != nil { /* handle */ }
```

**Per-query options** (consistency, timeout overrides):

```go
// Use ConnOpt to customize individual queries (for future use)
type Opt struct {
    Consistency gocql.Consistency
    Timeout     time.Duration
}

cassandradb.WithConsistency(gocql.LocalQuorum)
cassandradb.WithTimeout(30 * time.Second)
```

### Cassandra Keyspace Management

The package provides helpers for keyspace lifecycle operations.

**Create a keyspace**:

```go
// SimpleStrategy (single datacenter)
opts := cassandradb.SimpleStrategyParams("my_keyspace", 3)
if err := cassandradb.CreateKeyspace(session, opts); err != nil {
    return fmt.Errorf("create keyspace: %w", err)
}

// NetworkTopologyStrategy (multi-datacenter)
opts := cassandradb.KeyspaceOptions{
    KeyspaceName:        "my_keyspace",
    ReplicationStrategy: cassandradb.CassandraNetworkTopologyStrategy,
    Datacenters: map[string]int{
        "dc1": 3,
        "dc2": 2,
    },
}
if err := cassandradb.CreateKeyspace(session, opts); err != nil {
    return fmt.Errorf("create keyspace: %w", err)
}
```

**Drop a keyspace**:

```go
if err := cassandradb.DropKeyspace(session, "my_keyspace"); err != nil {
    return fmt.Errorf("drop keyspace: %w", err)
}
```

**Low-level session creation** (for migrations or keyspace creation before main session):

```go
// Connect without keyspace (empty keyspace) to create/drop keyspaces
session, err := cassandradb.NewCassandraSession(cfg, "")
if err != nil {
    return fmt.Errorf("create session: %w", err)
}
defer session.Close()

// Now create the keyspace
// Then create a second session with the keyspace
appSession, err := cassandradb.NewCassandraSession(cfg, cfg.Keyspace)
```

**Replication params validation**:

```go
opts := cassandradb.KeyspaceOptions{...}
params, err := opts.ReplicationParams() // validates and generates CQL params string
if err != nil {
    // ErrDatacenterParamsNotFound or ErrBadReplicationFactor
}
```

### Cassandra Shutdown

```go
if err := db.Shutdown(ctx); err != nil {
    log.Error("cassandra shutdown", logger.Err(err))
}
```

`Shutdown` closes the `*gocql.Session` and logs start/stop events.

---

## Common Patterns

### Service Initialization (Dependency Injection)

```go
// service/transport.go or cmd/main.go

type App struct {
    pg    *postgresdb.DB
    ch    *clickdb.DB
    cass  *cassandradb.DB
}

func NewApp(
    pgCfg postgresdb.Config,
    chCfg clickdb.Config,
    cassCfg cassandradb.Config,
    log logger.Logger,
) (*App, error) {
    pg := postgresdb.New(pgCfg, log)
    if err := pg.Prepare(context.Background()); err != nil {
        return nil, fmt.Errorf("postgres: %w", err)
    }

    ch := clickdb.New("my-service", chCfg)
    if err := ch.Prepare(context.Background()); err != nil {
        return nil, fmt.Errorf("clickhouse: %w", err)
    }

    cass := cassandradb.NewDB(cassCfg, log)
    if err := cass.Prepare(context.Background()); err != nil {
        return nil, fmt.Errorf("cassandra: %w", err)
    }

    return &App{pg: pg, ch: ch, cass: cass}, nil
}

func (a *App) Shutdown(ctx context.Context) error {
    var errs error
    if err := a.pg.Shutdown(ctx); err != nil {
        errs = errors.Join(errs, fmt.Errorf("postgres: %w", err))
    }
    if err := a.ch.Shutdown(ctx); err != nil {
        errs = errors.Join(errs, fmt.Errorf("clickhouse: %w", err))
    }
    if err := a.cass.Shutdown(ctx); err != nil {
        errs = errors.Join(errs, fmt.Errorf("cassandra: %w", err))
    }
    return errs
}
```

### Repository Pattern

```go
// internal/repository/user_repo.go

type UserRepo struct {
    db *postgresdb.DB
}

func NewUserRepo(db *postgresdb.DB) *UserRepo {
    return &UserRepo{db: db}
}

func (r *UserRepo) GetByID(ctx context.Context, id string) (User, error) {
    var u User
    err := postgresdb.RecoveryWithResult(ctx, func() (User, error) {
        err := r.db.RP().QueryRow(ctx,
            "SELECT id, name, email FROM users WHERE id = $1", id,
        ).Scan(&u.ID, &u.Name, &u.Email)
        return u, err
    })
    if err != nil {
        return User{}, fmt.Errorf("get user by id: %w", err)
    }
    return u, nil
}

func (r *UserRepo) Create(ctx context.Context, u User) error {
    _, err := r.db.P().Exec(ctx,
        "INSERT INTO users (id, name, email) VALUES ($1, $2, $3)",
        u.ID, u.Name, u.Email,
    )
    if err != nil {
        return fmt.Errorf("create user: %w", err)
    }
    return nil
}
```

### Using `Pooler` Interface for Testability

```go
// In repository, depend on the interface, not the concrete type
type OrderRepo struct {
    pool postgresdb.Pooler
}

func NewOrderRepo(pool postgresdb.Pooler) *OrderRepo {
    return &OrderRepo{pool: pool}
}

func (r *OrderRepo) Get(ctx context.Context, id string) (Order, error) {
    conn, err := r.pool.Conn(ctx, postgresdb.FromReplica())
    if err != nil {
        return Order{}, fmt.Errorf("acquire conn: %w", err)
    }
    defer conn.Release()
    // ...
}
```

---

## Anti-Patterns

| ❌ Don't | ✅ Do Instead |
|----------|---------------|
| Call `db.P()` before `Prepare()` (will auto-init and panic on error) | Always call `Prepare(ctx)` explicitly at startup |
| Use raw SQL in service layer | Use the repository pattern; keep SQL in `repo` layer |
| Ignore replica configuration | Set `HostReplicas` in config and use `db.RP()` / `FromReplica()` for reads |
| Skip `Recovery*` for write-heavy workloads | Wrap serialization-sensitive operations with `Recovery` / `RecoveryWithResult` |
| Close `*pgxpool.Conn` without `Release()` | Always `defer conn.Release()` after `db.Conn(ctx)` |
| Pass `context.Background()` to queries that should respect cancellation | Use request-scoped contexts with timeouts |
| Create Cassandra keyspaces on every startup | Use `CreateKeyspace` only in migrations; normal startup uses an existing keyspace |
| Use ClickHouse `Init()` in request handlers | Call `Init()` or `Prepare(ctx)` once at application startup |
| Mix `Stop()` and `Shutdown()` on ClickHouse | Prefer `Shutdown(ctx)` for consistent error handling |

---

## Quick Reference — Package Summary

| Package | Driver | Connection Type | Init Fn | Accessor | Shutdown |
|---------|--------|----------------|---------|----------|----------|
| `postgresdb` | `pgx/v5` | `*pgxpool.Pool` | `New(cfg, log)` → `Prepare(ctx)` | `.P()`, `.RP()`, `.Pool(opts)`, `.Conn(ctx, opts)` | `.Shutdown(ctx)` |
| `clickdb` | `clickhouse-go/v2` | `clickhouse.Conn` | `New(app, cfg)` → `Prepare(ctx)` or `Init()` | `.C()` | `.Shutdown(ctx)` or `.Stop()` |
| `cassandradb` | `gocql` | `*gocql.Session` | `NewDB(cfg, log)` → `Prepare(ctx)` | `.Conn()` | `.Shutdown(ctx)` |