---
name: go-foundation-core
description: a
---

# go-foundation Core Packages Skill

Guidance for AI agents (e.g., `godev`) on correctly initializing and using the
`go-foundation` library's core packages: `app`, `config`, `logger`, `slogger`.

**Module path:** `gitlab.wildberries.ru/reviews-ext/go-foundation`

---

## Table of Contents

1. [Overview](#overview)
2. [pkg/config — Configuration](#pkg-config--configuration)
3. [pkg/logger — Structured Logging](#pkg-logger--structured-logging)
4. [pkg/slogger — DEPRECATED](#pkg-slogger--deprecated)
5. [pkg/app — Application Lifecycle](#pkg-app--application-lifecycle)
6. [Full Bootstrap Example](#full-bootstrap-example)
7. [Common Patterns](#common-patterns)
8. [Anti-Patterns & Pitfalls](#anti-patterns--pitfalls)

---

## Overview

`go-foundation` provides a standardised bootstrap for microservices:

| Package | Purpose | Status |
|---------|---------|--------|
| `pkg/config` | Generic config loading (env / YAML / `.env`) | **Stable** |
| `pkg/logger` | Structured logging via `slog` (OTEL + Sentry + context attrs) | **Stable** |
| `pkg/slogger` | Legacy `slog` helpers | **DEPRECATED — use `pkg/logger`** |
| `pkg/app` | Application lifecycle (service manager, graceful shutdown) | **Stable** |

**Typical initialisation order:**

```
config → logger → sentry → tracer → metrics → pyroscope → app.Run()
```

The `App[T]` type automates this entire chain. You should almost never call the
individual init functions yourself—let `NewApp` handle it.

---

## pkg/config — Configuration

### `Config[T]` — Generic Configuration Container

```go
import "gitlab.wildberries.ru/reviews-ext/go-foundation/pkg/config"

type Config[T any] struct {
    Namespace      string        `env:"NAMESPACE" yaml:"namespace"`
    AppName        string        `env:"APP_NAME" yaml:"app-name"`
    Env            string        `env:"ENV" yaml:"env"`
    Tag            string        `env:"TAG" yaml:"tag"`
    EpicName       string        `env:"EPIC_NAME" yaml:"epic-name"`
    LogicalEnv     string        `env:"LOGICAL_ENV" yaml:"logical-env"`
    Region         string        `env:"REGION" yaml:"region"`
    Zone           string        `env:"ZONE" yaml:"zone"`
    Log            logger.Config `envPrefix:"LOG_" yaml:"log"`
    Sentry         sentryinit.Config `envPrefix:"SENTRY_" yaml:"sentry"`
    Telemetry      telemetry.Config  `envPrefix:"TELEMETRY_" yaml:"telemetry"`
    Http           server.Config `envPrefix:"REST_" yaml:"rest"`
    ShutdownDelay  time.Duration `env:"SHUTDOWN_DELAY"`
    ShutdownMaxWait time.Duration `env:"SHUTDOWN_MAX_WAIT"`
    ServiceConfig  T             // ← your service-specific config
}
```

**Key points:**

- `T` is your service-specific config struct. Everything above `ServiceConfig`
  is provided by the library.
- `envPrefix` tags on nested structs mean their fields are prefixed, e.g.
  `LOG_LEVEL`, `LOG_HANDLER`.
- `IsLocal()` returns `true` when `Env` is **not** one of `"stage"`, `"prod"`,
  `"prd"`. Use it to toggle dev-friendly defaults (e.g. pretty logging).

### Defining a Service Config

```go
type MyServiceConfig struct {
    DatabaseURL string `env:"DB_URL" yaml:"database-url"`
    MaxWorkers  int    `env:"MAX_WORKERS" envDefault:"4" yaml:"max-workers"`
    KafkaBroker string `env:"KAFKA_BROKER" yaml:"kafka-broker"`
}
```

### Config Loading Rules

`ReadConfig(configFile, &cfg)`:

| `configFile` value | Behaviour |
|--------------------|-----------|
| `"none"` or `""`  | Loads `.env` (walks up `../../.env`), then reads env vars via `caarlos0/env`. |
| Any other string   | Parses the given YAML file. |

> **Important:** `App[T].init()` calls `ReadConfig` internally with the
> `-config-file` flag (default: `"none"`). You normally do **not** call
> `ReadConfig` directly.

### Helper Constructors

These are called internally by `App[T]`. Documented for reference only.

```go
// OTEL tracer config derived from Config[T]
config.NewTracerConfig[T](cfg) tracer.OTelConfig

// Sentry config derived from Config[T]
config.NewSentryConfig[T](cfg, version) sentryinit.Config

// OTEL metrics config derived from Config[T]
config.NewOtelMetricsConfig[T](cfg) metrics.OtelMetricsConfig

// Pyroscope config derived from Config[T]
config.NewPyroscopeConfig[T](cfg) pyroscopinit.PyroscopeConfig
```

### Safe Config Logging

Use `LogMaskedConfig` to print config without leaking credentials:

```go
fmt.Println(config.LogMaskedConfig(&cfg))
// Passwords, tokens, secrets, API keys, users are masked: "*****ab"
```

Fields masked (case-insensitive): `password`, `user`, `apikey`, `secret`,
`token`, `feedbacktoken`.

### Generating Config Templates

```go
// Print all env vars for deployment manifests
config.PrintEnv(&cfg)

// Generate a YAML config template file
config.CreateYaml("config.yaml", &cfg)
```

---

## pkg/logger — Structured Logging

### Logger Interface

```go
type Logger interface {
    Debug(msg string, args ...any)
    Info(msg string, args ...any)
    Warn(msg string, args ...any)
    Error(msg string, args ...any)
    DebugCtx(ctx context.Context, msg string, args ...any)
    InfoCtx(ctx context.Context, msg string, args ...any)
    WarnCtx(ctx context.Context, msg string, args ...any)
    ErrorCtx(ctx context.Context, msg string, args ...any)
    With(args ...any) Logger
}
```

**Always prefer the `*Ctx` variants** in request handlers and service methods
— they inject `trace_id`, `span_id`, and `sampled` from the OTel span in the
context.

### Creating a Logger

```go
import "gitlab.wildberries.ru/reviews-ext/go-foundation/pkg/logger"

// Production logger (JSON handler, info level)
log := logger.New(logger.Config{
    Level:   logger.LogLevelInfo,
    Handler: logger.LogHandlerJson,
})

// Local / dev logger (pretty handler)
log := logger.New(logger.Config{
    Level:   logger.LogLevelDebug,
    Handler: logger.LogHandlerPretty,
})

// Discard logger for tests
log := logger.NewMock()
```

> **Note:** `App[T]` creates the logger automatically. When `Env` is local
> (i.e. not stage/prod/prd) and `Log.Handler` is empty, it defaults to
> `LogHandlerPretty`.

### Config Struct

```go
type Config struct {
    Level       LogLevel   `env:"LEVEL" envDefault:"info" yaml:"level"`
    Handler     LogHandler `env:"HANDLER" yaml:"handler"`
    AddSource   bool       `env:"ADD_SOURCE" envDefault:"false" yaml:"add-source"`
    SkipCallers int        `env:"SKIP_CALLERS" envDefault:"3" yaml:"skip-collers"`
}
```

| Handler constant | Output format |
|------------------|--------------|
| `LogHandlerJson` (`"json"`) | `slog.JSONHandler` |
| `LogHandlerText` (`"text"`) | `slog.TextHandler` |
| `LogHandlerPretty` (`"pretty"`) | `beautyslog` (human-friendly) |

### Derived Loggers

```go
// .With() returns a new Logger with pre-set attributes
requestLog := log.With(
    logger.String("method", "GET"),
    logger.String("path", "/api/v1/reviews"),
)
requestLog.InfoCtx(ctx, "request received")
```

### Global Logger

Package-level functions delegate to a global `*Log` instance:

```go
logger.SetDefault(myLog)          // set global logger
logger.Info("bootstrap complete")  // use global logger
logger.ErrorCtx(ctx, "db failed", logger.Err(err))
```

> The global logger is initialised from `slog.Default()` on package load. `App[T]`
> replaces it with a properly configured instance via `logger.SetDefault()`.

### Attribute Helpers

All helpers return `slog.Attr`. Use them as variadic `args` in log calls:

| Helper | Key | Example |
|--------|-----|---------|
| `logger.Err(err)` | `"error"` | `logger.Error("failed", logger.Err(err))` |
| `logger.String(k, v)` | custom | `logger.String("service", "reviews")` |
| `logger.Int64(k, v)` | custom | `logger.Int64("offset", 42)` |
| `logger.Int(k, v)` | custom | `logger.Int("count", 10)` |
| `logger.Bool(k, v)` | custom | `logger.Bool("cached", true)` |
| `logger.Float64(k, v)` | custom | `logger.Float64("ratio", 0.95)` |
| `logger.Duration(k, v)` | custom | `logger.Duration("elapsed", 150*time.Millisecond)` |
| `logger.Time(k, v)` | custom | `logger.Time("started_at", time.Now())` |
| `logger.Method(v)` | `"method"` | `logger.Method("POST")` |
| `logger.Offset(v)` | `"offset"` | `logger.Offset(12345)` |
| `logger.EventTime(v)` | `"event_time"` | `logger.EventTime(ts)` |
| `logger.MessageTime(v)` | `"message_time"` | `logger.MessageTime(ts)` |
| `logger.EventType(v)` | `"event_type"` | `logger.EventType("order.created")` |
| `logger.ServiceName(v)` | `"service_name"` | `logger.ServiceName("reviews")` |
| `logger.NmID(v)` | `"nm_id"` | `logger.NmID(567890)` |
| `logger.FirstOffset(v)` | `"first_offset"` | `logger.FirstOffset(100)` |
| `logger.LastOffset(v)` | `"last_offset"` | `logger.LastOffset(200)` |
| `logger.TraceInfo(ctx)` | returns `[]any` | `logger.InfoCtx(ctx, "msg", logger.TraceInfo(ctx)...)` |
| `logger.Group(k, args...)` | nested group | `logger.Group("request", logger.String("url", u))` |
| `logger.Any(k, v)` | custom | `logger.Any("payload", data)` |

### Context-Based Attributes

Inject attributes into a `context.Context` so they are automatically attached to
every log call using that context:

```go
// Enrich context with attributes (call this in middleware)
ctx = logger.AppendCtx(ctx,
    logger.String("request_id", reqID),
    logger.String("user_id", userID),
)

// All *Ctx calls using this ctx will include those attributes
logger.InfoCtx(ctx, "processing request")
```

### Handler Pipeline (Internal)

When `logger.New(cfg)` is called, the resulting `*slog.Logger` pipelines through
`slog-multi.Fanout`:

1. **OTEL handler** — bridges logs to OpenTelemetry via `otelslog`.
2. **Context handler** — injects `slog.Attr` stored via `AppendCtx`.
3. **Trace handler** — injects `trace_id`, `span_id`, `sampled` from `ctx`.
4. **Sentry handler** — captures `Error`-level logs as Sentry events.
5. **Format handler** — the user-chosen handler (JSON / Text / Pretty).

You do **not** need to construct this pipeline manually. `logger.New(cfg)` does
it for you.

---

## pkg/slogger — DEPRECATED

The entire `pkg/slogger` package is **deprecated**. All of its functionality has
been migrated to `pkg/logger`.

| Deprecated | Replacement |
|-----------|--------------|
| `slogger.SetupLogger(level)` | `logger.New(logger.Config{Level: ...})` |
| `slogger.Err(err)` | `logger.Err(err)` |
| `slogger.Method(m)` | `logger.Method(m)` |
| `slogger.Offset(o)` | `logger.Offset(o)` |
| `slogger.EventTime(t)` | `logger.EventTime(t)` |
| `slogger.MessageTime(t)` | `logger.MessageTime(t)` |
| `slogger.EventType(t)` | `logger.EventType(t)` |
| `slogger.ServiceName(n)` | `logger.ServiceName(n)` |
| `slogger.AppendCtx(ctx, attrs...)` | `logger.AppendCtx(ctx, attrs...)` |
| `slogger.AttrFromContext(ctx)` | `logger.AttrFromContext(ctx)` |
| `slogger.NewContextHandler(next)` | internal; handled by `logger.New()` |
| `slogger.SentryHandler` | internal; handled by `logger.New()` |

**Rule:** Never import `pkg/slogger` in new code.

---

## pkg/app — Application Lifecycle

### Core Types

#### `ManagedService` — Service Interface (Preferred)

```go
type ManagedService interface {
    Prepare(ctx context.Context) error  // Initialisation (DB pools, connections)
    Start(ctx context.Context) error     // Blocking: runs until ctx cancelled or error
    Shutdown(ctx context.Context) error  // Graceful cleanup
}
```

**Lifecycle:**

1. `Prepare` is called **sequentially** on all services. If any `Prepare` fails,
   already-prepared services are shut down in reverse order and `Run` returns
   the error.
2. `Start` is called **concurrently** for all services (each in its own
   goroutine). A service must block until the context is cancelled or an error
   occurs.
3. `Shutdown` is called **sequentially in reverse order** after receiving a
   SIGTERM, context cancellation, or a runtime error/panic. Each service's
   `Shutdown` receives a **fresh context** with a timeout (`ShutdownMaxWait`).

#### `Service` — Legacy Interface (Deprecated)

```go
type Service interface {
    Init() error
    Run(ctx context.Context)
    Stop()
}
```

Use `NewManagedServiceAdapter(svc)` to wrap a legacy `Service` into a
`ManagedService`. Prefer `ManagedService` for all new code.

#### `App[T]` — Application Entry Point

```go
app := app.NewApp[MyServiceConfig]("v1.2.3")
```

`NewApp[T]` performs the following on construction:

1. Parses `-config-file` flag (default: `"none"` → env-based config).
2. Calls `config.ReadConfig` to populate `Config[T]`.
3. Calls `cfg.InjectAppName()`.
4. Creates a `logger.Logger` (auto-selects `pretty` handler if local env).
5. Initialises Sentry via `sentryinit.Init`.
6. Initialises OTel tracer and metrics.
7. Initialises Pyroscope if `Telemetry.Pyroscope.Enabled`.
8. Creates a `Manager` with shutdown options from config.

**Run options:**

```go
// Disable telemetry init (tracer, metrics, pyroscope)
app := app.NewApp[MyServiceConfig]("v1.0.0", app.WithoutTelemetry())
```

#### `Manager` — Service Orchestrator

```go
type Manager struct { /* ... */ }

func NewManager(log logger.Logger, opts ...ManagerOpt) *Manager

// Options
func WithShutdownTimeout(timeout time.Duration) ManagerOpt  // default: 27s
func WithShutdownDelay(delay time.Duration) ManagerOpt       // default: 17s
```

**Behaviour:**

- On `Prepare` failure: rolls back already-prepared services in reverse order.
- On `Start` error or panic: cancels the run context → triggers shutdown of all
  services in reverse order.
- On SIGTERM / `os.Interrupt` / context cancellation: triggers shutdown.
- `ShutdownDelay` (default 17s): pauses before starting shutdown to allow
  load-balancers to deregister the pod.
- `ShutdownMaxWait` (default 27s): maximum time to wait for all services to
  shut down. If exceeded, `Run` returns `ErrShutdownMaxWaitTime`.
- **Panic recovery**: both `Start` and `Shutdown` goroutines recover from
  panics, log the stack, and continue shutdown.

### `App[T]` Methods

```go
// Access config
app.Config()          → config.Config[T]
app.ServiceConfig()   → T
app.AppName()          → string
app.Namespace()        → string
app.Version()          → string
app.Env()              → string

// Access logger
app.Logger()           → logger.Logger

// Add services (before Run)
app.Add(svc1, svc2)

// Run the application (blocking)
err := app.Run(ctx, svc3, svc4)

// Higher-level pattern: configure services and run
err := app.RunConfigurableService(ctx, configureServices)
```

### `ConfigureService[T]` — Functional Configuration Pattern

```go
type ConfigureService[T any] func(
    appconfig config.Config[T],
    l logger.Logger,
) ([]ManagedService, error)
```

This is the recommended way to wire services. It receives the full config and
logger, then returns a slice of `ManagedService`.

---

## Full Bootstrap Example

### 1. Define Your Service Config

```go
package main

import (
    "time"

    "gitlab.wildberries.ru/reviews-ext/go-foundation/pkg/config"
)

type ReviewsConfig struct {
    DBHost     string        `env:"DB_HOST" envDefault:"localhost:5432" yaml:"db-host"`
    DBName     string        `env:"DB_NAME" envDefault:"reviews" yaml:"db-name"`
    DBUser     string        `env:"DB_USER" yaml:"db-user"`
    DBPassword string        `env:"DB_PASSWORD" yaml:"db-password"`
    KafkaBrokers string      `env:"KAFKA_BROKERS" yaml:"kafka-brokers"`
    HTTPPort   int           `env:"HTTP_PORT" envDefault:"8080" yaml:"http-port"`
    MaxRetries int           `env:"MAX_RETRIES" envDefault:"3" yaml:"max-retries"`
    CacheTTL   time.Duration `env:"CACHE_TTL" envDefault:"5m" yaml:"cache-ttl"`
}
```

### 2. Implement ManagedService

```go
package main

import (
    "context"
    "database/sql"
    "fmt"

    "gitlab.wildberries.ru/reviews-ext/go-foundation/pkg/logger"
)

type DBService struct {
    db  *sql.DB
    log logger.Logger
    dsn string
}

func NewDBService(log logger.Logger, dsn string) *DBService {
    return &DBService{log: log, dsn: dsn}
}

func (s *DBService) Prepare(ctx context.Context) error {
    var err error
    s.db, err = sql.Open("pgx", s.dsn)
    if err != nil {
        return fmt.Errorf("open db: %w", err)
    }
    if err := s.db.PingContext(ctx); err != nil {
        return fmt.Errorf("ping db: %w", err)
    }
    s.log.InfoCtx(ctx, "database connected")
    return nil
}

func (s *DBService) Start(ctx context.Context) error {
    <-ctx.Done() // block until context cancelled
    return ctx.Err()
}

func (s *DBService) Shutdown(ctx context.Context) error {
    if s.db != nil {
        return s.db.Close()
    }
    return nil
}
```

### 3. Implement HTTP Server as ManagedService

```go
package main

import (
    "context"
    "net/http"

    "gitlab.wildberries.ru/reviews-ext/go-foundation/pkg/logger"
)

type HTTPServer struct {
    server *http.Server
    log    logger.Logger
}

func NewHTTPServer(log logger.Logger, handler http.Handler, addr string) *HTTPServer {
    return &HTTPServer{
        log: log,
        server: &http.Server{Addr: addr, Handler: handler},
    }
}

func (s *HTTPServer) Prepare(_ context.Context) error {
    return nil // no preparation needed
}

func (s *HTTPServer) Start(ctx context.Context) error {
    s.log.InfoCtx(ctx, "http server starting", logger.String("addr", s.server.Addr))
    if err := s.server.ListenAndServe(); err != nil && err != http.ErrServerClosed {
        return err
    }
    return nil
}

func (s *HTTPServer) Shutdown(ctx context.Context) error {
    return s.server.Shutdown(ctx)
}
```

### 4. Wire Everything in main.go

```go
package main

import (
    "context"
    "fmt"
    "net/http"
    "os"

    "gitlab.wildberries.ru/reviews-ext/go-foundation/pkg/app"
    "gitlab.wildberries.ru/reviews-ext/go-foundation/pkg/config"
    "gitlab.wildberries.ru/reviews-ext/go-foundation/pkg/logger"
)

func main() {
    version := os.Getenv("TAG")
    if version == "" {
        version = "dev"
    }

    application := app.NewApp[ReviewsConfig](version)

    err := application.RunConfigurableService(
        context.Background(),
        configureServices,
    )
    if err != nil {
        application.Logger().Error("service exited with error", logger.Err(err))
        os.Exit(1)
    }
}

func configureServices(
    cfg config.Config[ReviewsConfig],
    log logger.Logger,
) ([]app.ManagedService, error) {
    // Log the config (safe: masks passwords/secrets)
    log.Info("config loaded", logger.String("config", config.LogMaskedConfig(&cfg)))

    // Build DSN
    dsn := fmt.Sprintf(
        "postgres://%s:%s@%s/%s",
        cfg.ServiceConfig.DBUser,
        cfg.ServiceConfig.DBPassword,
        cfg.ServiceConfig.DBHost,
        cfg.ServiceConfig.DBName,
    )

    // Create services
    dbSvc := NewDBService(log, dsn)

    mux := http.NewServeMux()
    // ... register handlers ...
    httpSvc := NewHTTPServer(log, mux, fmt.Sprintf(":%d", cfg.ServiceConfig.HTTPPort))

    return []app.ManagedService{dbSvc, httpSvc}, nil
}
```

### 5. Alternative: Manual Wiring

```go
application := app.NewApp[ReviewsConfig]("v1.0.0")

dbSvc := NewDBService(application.Logger(), dsn)
httpSvc := NewHTTPServer(application.Logger(), mux, ":8080")

application.Add(dbSvc)
application.Add(httpSvc)

if err := application.Run(context.Background()); err != nil {
    os.Exit(1)
}
```

### 6. Skipping Telemetry (e.g., for CLI Tools or Tests)

```go
application := app.NewApp[MyConfig]("v1.0.0", app.WithoutTelemetry())
```

This skips OTel tracer init, metrics init, and Pyroscope init.

---

## Common Patterns

### Using Context-Based Logging Attributes in Middleware

```go
func RequestIDMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        reqID := uuid.New().String()
        ctx := logger.AppendCtx(r.Context(),
            logger.String("request_id", reqID),
        )
        next.ServeHTTP(w, r.WithContext(ctx))
    })
}
```

Every `logger.InfoCtx(ctx, ...)` call downstream will automatically include
`request_id`.

### Accessing the Global Logger from Anywhere

```go
// After App[T] has been created, the global logger is set
logger.Info("this uses the global logger")
logger.ErrorCtx(ctx, "db error", logger.Err(err))

// In tests or before App[T] init, you can set your own:
logger.SetDefault(logger.NewMock()) // discard all output
```

### Custom Shutdown Timers via Config

Set environment variables to control shutdown behaviour:

```bash
# .env
SHUTDOWN_DELAY=5s        # Time for LB to deregister (default: 17s)
SHUTDOWN_MAX_WAIT=30s    # Max time to wait for all Shutdown() calls (default: 27s)
```

Or programmatically via `Manager`:

```go
mng := app.NewManager(log,
    app.WithShutdownTimeout(30*time.Second),
    app.WithShutdownDelay(5*time.Second),
)
```

### Running a Single Background Worker

```go
type KafkaConsumer struct { /* ... */ }

func (k *KafkaConsumer) Prepare(ctx context.Context) error {
    // connect to broker
    return nil
}

func (k *KafkaConsumer) Start(ctx context.Context) error {
    // block reading from Kafka until ctx.Done()
    <-ctx.Done()
    return ctx.Err()
}

func (k *KafkaConsumer) Shutdown(ctx context.Context) error {
    // close consumer, commit offsets
    return nil
}
```

### Logging with Trace Info Explicitly

```go
// *Ctx methods automatically include trace_id/span_id from ctx.
// You can also extract them manually:
attrs := logger.TraceInfo(ctx)  // returns []any
logger.Info("manual trace info", attrs...)
```

---

## Anti-Patterns & Pitfalls

### ❌ Using `pkg/slogger` in New Code

```go
// BAD — deprecated
import "gitlab.wildberries.ru/reviews-ext/go-foundation/pkg/slogger"
log := slogger.SetupLogger("debug")
```

```go
// GOOD — use pkg/logger
import "gitlab.wildberries.ru/reviews-ext/go-foundation/pkg/logger"
log := logger.New(logger.Config{Level: logger.LogLevelDebug})
```

### ❌ Calling `config.ReadConfig` Directly When Using `App[T]`

`App[T].init()` already reads config from the `-config-file` flag. Calling
`ReadConfig` again will overwrite or conflict with the parsed config.

### ❌ Using the Legacy `Service` Interface

```go
// BAD — deprecated
type MySvc struct{}
func (s *MySvc) Init() error        { return nil }
func (s *MySvc) Run(ctx context.Context)  {}
func (s *MySvc) Stop()              {}
```

```go
// GOOD — use ManagedService
type MySvc struct{}
func (s *MySvc) Prepare(ctx context.Context) error  { return nil }
func (s *MySvc) Start(ctx context.Context) error     { <-ctx.Done(); return ctx.Err() }
func (s *MySvc) Shutdown(ctx context.Context) error  { return nil }
```

### ❌ Ignoring the Context in `*Ctx` Log Methods

```go
// BAD — loses trace_id and context attributes
logger.Info("handled request", logger.String("path", r.URL.Path))

// GOOD — propagates trace_id, span_id, and any context attributes
logger.InfoCtx(r.Context(), "handled request", logger.String("path", r.URL.Path))
```

### ❌ Returning `nil` from `Start` in Long-Running Services

`Start` should block. If it returns `nil` immediately, the goroutine exits and
the service is considered stopped.

```go
// BAD — returns immediately
func (s *Worker) Start(ctx context.Context) error {
    go s.process(ctx) // fire-and-forget
    return nil         // goroutine exits
}

// GOOD — blocks until context cancellation
func (s *Worker) Start(ctx context.Context) error {
    s.process(ctx) // blocks
    return ctx.Err()
}
```

If you must fire a background goroutine, use a channel or `sync.WaitGroup` to
block `Start` until `ctx.Done()`.

### ❌ Not Handling Panics in `Start`

The `Manager` recovers panics in `Start`, but your `Shutdown` should be
idempotent and safe to call even if `Prepare` was interrupted. The Manager calls
`Shutdown` in reverse order, so later services may have `Shutdown` called even
though `Prepare` succeeded for them but `Start` was never called.

### ❌ Hardcoding Config Values Instead of Using Environment Variables

Always use env tags with `envDefault` for local development defaults:

```go
// BAD
type Config struct {
    Port int // no env tag
}

// GOOD
type Config struct {
    Port int `env:"PORT" envDefault:"8080" yaml:"port"`
}
```

### ✅ Always Use `logger.Err(err)` for Error Attributes

```go
// BAD — loses error type information
logger.Error("db failed", "error", err.Error())

// GOOD — preserves full error structure for Sentry/OTEL
logger.Error("db failed", logger.Err(err))
```

### ✅ Use `config.LogMaskedConfig` When Logging Config

```go
// BAD — may leak credentials
log.Info("config", logger.Any("cfg", cfg))

// GOOD — masks secrets
log.Info("config loaded", logger.String("config", config.LogMaskedConfig(&cfg)))
```

---

## Environment Variable Reference

The `Config[T]` struct reads these environment variables (prefixes shown):

| Variable | Description | Default |
|----------|-------------|---------|
| `NAMESPACE` | Kubernetes namespace | — |
| `APP_NAME` | Service name | — |
| `ENV` | Environment (`stage`, `prod`, `prd`, or local) | — |
| `TAG` | Service version/tag | — |
| `EPIC_NAME` | Epic/project name | — |
| `LOGICAL_ENV` | Logical environment (e.g., `ru`) | — |
| `REGION` | Deployment region | — |
| `ZONE` | Availability zone | — |
| `LOG_LEVEL` | Log level (`debug`, `info`, `warn`, `error`) | `info` |
| `LOG_HANDLER` | Log handler (`json`, `text`, `pretty`) | auto (pretty if local) |
| `LOG_ADD_SOURCE` | Add source file/line to logs | `false` |
| `LOG_SKIP_CALLERS` | Skip caller frames | `3` |
| `SENTRY_DSN` | Sentry DSN | — |
| `SHUTDOWN_DELAY` | Delay before shutdown (for LB deregistration) | `17s` |
| `SHUTDOWN_MAX_WAIT` | Max time to wait for Shutdown | `27s` |

Plus any fields from your `ServiceConfig T` struct.