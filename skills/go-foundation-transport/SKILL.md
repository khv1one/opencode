---
name: go-foundation-transport
description: a
---

# go-foundation Transport & Messaging Skill

> **Purpose:** Teach an AI agent (e.g. `godev`) how to correctly build HTTP servers and Kafka workers using the `go-foundation` library (`gitlab.wildberries.ru/reviews-ext/go-foundation`).

---

## 1. Architecture Overview

All three packages — `pkg/http/server`, `pkg/producer/v2`, `pkg/consumer/v2` — follow a **uniform lifecycle contract** defined by the `app.ManagedService` interface:

```go
type ManagedService interface {
    Prepare(ctx context.Context) error
    Start(ctx context.Context) error
    Shutdown(ctx context.Context) error
}
```

The `app.Manager` orchestrates multiple services:

1. Calls `Prepare()` on each service **sequentially** (init order = add order).
2. Calls `Start()` on each service **concurrently** (goroutine per service).
3. On `SIGTERM`/`os.Interrupt` or context cancellation, calls `Shutdown()` in **reverse add order** with a configurable timeout (`DefaultShutdownMaxWaitTime = 27s`).

**Key rule:** Always register services via `app.Add()` before calling `app.Run()`, or use `app.RunConfigurableService()` for a functional setup pattern.

---

## 2. HTTP Server — `pkg/http/server`

### 2.1 Core Types

| Type | Role |
|---|---|
| `FiberServer` | Wraps `*fiber.App`; owns config, logger, router, middleware stack |
| `Router` | Interface with a single method `Route(*FiberServer) error` — registers routes |
| `Config` | Server configuration (host, port, timeouts, body limits, etc.) |
| `AuthConfig` | Auth middleware configuration (key URLs, changelog) |
| `CorsConfig` | CORS configuration (origins, credentials, headers) |
| `FiberServerOpt` | Functional option type for `FiberServer` |

### 2.2 Configuration (`Config`)

Populated via environment variables or YAML. Key fields:

| Field | Env var | Default | Description |
|---|---|---|---|
| `AppName` | `APP_NAME` | — | Application name (used in metrics) |
| `Host` | `HOST` | — | Bind address |
| `Port` | `PORT` | `8080` | Bind port |
| `RetryCount` | `RETRY_COUNT` | `3` | Listen retry attempts |
| `FiberReadTimeout` | `FIBER_READ_TIMEOUT` | `10` (seconds) | Read timeout |
| `FiberWriteTimeout` | `FIBER_WRITE_TIMEOUT` | `10` (seconds) | Write timeout |
| `FiberIdleTimeout` | `FIBER_IDLE_TIMEOUT` | `60` (seconds) | Idle timeout |
| `FiberBodyLimit` | `FIBER_BODY_LIMIT` | `10485760` (10 MB) | Max request body |
| `FiberConcurrency` | `FIBER_CONCURRENCY` | `524288` | Max concurrent connections |

### 2.3 Functional Options

```go
// Middleware options
func WithCommonMiddlewares(mws ...func(*fiber.Ctx) error) FiberServerOpt
func WithAuthMiddleware(amw func(*fiber.Ctx) error) FiberServerOpt
func WithCorseMiddleware(cmw func(*fiber.Ctx) error) FiberServerOpt  // note: library spelling

// Default middleware builders (auto-creates middleware from config)
func WithDefaultAuthMiddleware(authCfg AuthConfig) FiberServerOpt
func WithDefaultCorseMiddleware(corsCfg CorsConfig) FiberServerOpt

// Error handler options
func WithDefaultErrHandler() FiberServerOpt
func WithErrorHandler(emw func(*fiber.Ctx, error) error) FiberServerOpt

// Panic recovery
func WithRecover() FiberServerOpt
```

### 2.4 Router Pattern

The `Router` interface decouples route registration from server construction:

```go
type Router interface {
    Route(*FiberServer) error
}
```

Inside `Route()`, you have access to the full `FiberServer`:

```go
func (r *MyRouter) Route(s *server.FiberServer) error {
    // Public routes (no auth)
    s.App.Get("/health", r.healthHandler)

    // Protected routes (with auth)
    auth := s.AuthMiddleware()
    s.App.Get("/api/v1/resource", auth, r.listResources)
    s.App.Post("/api/v1/resource", auth, r.createResource)

    return nil
}
```

### 2.5 Middleware Order

When `FiberServer` is created with `NewFiberServer`, the middleware stack is built in this order:

1. `recover.New()` — **only if** `WithRecover()` is passed
2. `metrics.NewFiberMiddleware(cfg.AppName)` — OTEL metrics (always)
3. `tracer.NewFiberMiddleware()` — OTEL tracing (always)
4. Custom `commonMiddlewares` — from `WithCommonMiddlewares()`

Auth and CORS middlewares are **not** in the global stack; they are applied per-route or per-group via `s.AuthMiddleware()` / `s.CorseMiddleware()`.

### 2.6 Error Handling

The `ErrHandler` returns all errors as `200 OK` with a JSON body:

```go
type ErrResponse struct {
    Message string `json:"message"`
}
```

Known auth errors (`ErrInvalidToken`, `ErrTokenExpired`, `ErrSessionIsLoggedOff`) map to `"Unauthorized"`. All other errors map to `"Failed to perform operation"`.

Use `ErrHandlerWithLogger(log)` to also log the failed path.

### 2.7 Full HTTP Server Example

```go
package main

import (
    "context"

    "github.com/gofiber/fiber/v2"
    "gitlab.wildberries.ru/reviews-ext/go-foundation/pkg/app"
    "gitlab.wildberries.ru/reviews-ext/go-foundation/pkg/http/server"
    "gitlab.wildberries.ru/reviews-ext/go-foundation/pkg/logger"
)

// MyRouter implements server.Router
type MyRouter struct {
    log logger.Logger
}

func (r *MyRouter) Route(s *server.FiberServer) error {
    // Health check — no auth
    s.App.Get("/health", func(c *fiber.Ctx) error {
        return c.JSON(fiber.Map{"status": "ok"})
    })

    // Protected routes — auth required
    auth := s.AuthMiddleware()
    api := s.App.Group("/api/v1", auth)
    api.Get("/items", r.listItems)
    api.Post("/items", r.createItem)

    return nil
}

func (r *MyRouter) listItems(c *fiber.Ctx) error {
    return c.JSON(fiber.Map{"items": []string{}})
}

func (r *MyRouter) createItem(c *fiber.Ctx) error {
    return c.Status(fiber.StatusCreated).JSON(fiber.Map{"id": "new"})
}

func main() {
    application := app.NewApp[struct{}]("1.0.0")

    cfg := server.Config{
        AppName: "my-service",
        Host:   "0.0.0.0",
        Port:   8080,
    }
    authCfg := server.AuthConfig{
        AuthzKeysLoadURL: "https://auth.example.com/keys",
    }
    corsCfg := server.CorsConfig{}

    router := &MyRouter{log: application.Logger()}

    srv := server.NewFiberServer(
        application.Logger(),
        cfg,
        router,
        server.WithDefaultAuthMiddleware(authCfg),
        server.WithDefaultCorseMiddleware(corsCfg),
        server.WithDefaultErrHandler(),
        server.WithRecover(),
    )

    application.Add(srv)
    application.Run(context.Background())
}
```

---

## 3. Kafka Producer — `pkg/producer/v2`

### 3.1 Core Types

| Type | Role |
|---|---|
| `Service` | Kafka producer wrapping `kgo.Client` |
| `IProducer` | Interface: `Produce(ctx, []*kgo.Record) error` and `ProduceSync(ctx, []*kgo.Record) error` |
| `Config` | Producer configuration |
| `ServiceOpt` | Functional option type |

### 3.2 Configuration (`Config`)

| Field | Env var | Default | Description |
|---|---|---|---|
| `Brokers` | `BROKERS` | — | Kafka cluster addresses |
| `SASLMechanism` | `SASL` | `PLAIN` | Auth mechanism: `PLAIN`, `SCRAM-SHA-256`, `SCRAM-SHA-512`, `NONE` |
| `User` | `USER` | — | SASL username |
| `Password` | `PASSWORD` | — | SASL password |
| `RequestRetries` | `REQUEST_RETRIES` | `3` | Retry count for produce requests |
| `RetryTimeout` | `RETRY_TIMEOUT` | `5s` | Retry timeout |
| `ConnIdleTimeout` | `CONN_IDLE_TIMEOUT` | `30s` | Connection idle timeout |
| `DialTimeout` | `DIAL_TIMEOUT` | `30s` | Connection dial timeout |
| `Topic` | `TOPIC` | — | Default produce topic |

### 3.3 Functional Options

```go
func WithLogger(l logger.Logger) ServiceOpt       // Custom logger
func KotelTracer(t *otelKafka.Tracer) ServiceOpt   // Custom OTEL Kafka tracer
func KgoOpts(opts ...kgo.Opt) ServiceOpt           // Raw franz-go client options
```

### 3.4 Produce Methods

**`Produce(ctx, records)` — Asynchronous with errgroup**

- Uses `errgroup.WithContext` to produce each record concurrently.
- Each record gets OTEL trace injection via `kotelTracer.Inject()`.
- The callback-based produce is wrapped: a channel receives the result per record.
- If context is cancelled, returns `ctx.Err()`.

**`ProduceSync(ctx, records)` — Synchronous**

- Injects OTEL propagation headers into each record.
- Calls `kgo.Client.ProduceSync()` and returns the first error via `.FirstErr()`.

### 3.5 Lifecycle

```go
// Two creation patterns:

// 1. Manual lifecycle (recommended with app.Manager)
svc := producer.New(&cfg, producer.WithLogger(log))
if err := svc.Prepare(ctx); err != nil { /* handle */ }
// svc.Start(ctx) is a no-op (returns nil)
// Use svc.Produce() / svc.ProduceSync() as needed
// svc.Shutdown(ctx) closes the kgo.Client

// 2. One-step creation
svc, err := producer.NewWithInit(&cfg)
// Already initialized; ready to produce
```

### 3.6 Full Producer Example

```go
package main

import (
    "context"
    "fmt"

    "github.com/twmb/franz-go/pkg/kgo"
    "gitlab.wildberries.ru/reviews-ext/go-foundation/pkg/app"
    "gitlab.wildberries.ru/reviews-ext/go-foundation/pkg/producer/v2"
)

func main() {
    application := app.NewApp[struct{}]("1.0.0")

    cfg := producer.Config{
        Brokers:       []string{"kafka:9092"},
        SASLMechanism: "PLAIN",
        User:          "user",
        Password:      "pass",
        Topic:         "my-topic",
    }

    p := producer.New(&cfg, producer.WithLogger(application.Logger()))
    if err := p.Prepare(context.Background()); err != nil {
        panic(err)
    }

    application.Add(p) // registers as ManagedService

    // Somewhere in your business logic:
    records := []*kgo.Record{
        {Key: []byte("key-1"), Value: []byte(`{"event":"created"}`)},
    }
    if err := p.Produce(context.Background(), records); err != nil {
        application.Logger().Error("produce failed", logger.Err(err))
    }

    application.Run(context.Background())
}
```

### 3.7 Important Notes

- **`P()` accessor** — `svc.P()` returns the underlying `*kgo.Client` for advanced usage (e.g., manual flush).
- **`Start()`** is effectively a no-op — it only stores a cancel function. Production happens via `Produce()` / `ProduceSync()`.
- **`Shutdown()`** calls `producer.Close()` which flushes pending records and closes the connection.
- Always use `ProduceSync` when you need guaranteed delivery before proceeding (e.g., in request-response patterns).

---

## 4. Kafka Consumer — `pkg/consumer/v2`

### 4.1 Core Types

| Type | Role |
|---|---|
| `Service` | Kafka consumer wrapping `kgo.Client` with polling loop |
| `HandleFunc` | `func(ctx context.Context, records []*kgo.Record) error` — your message handler |
| `Config` | Consumer configuration |
| `ServiceOpt` | Functional option type |

### 4.2 Configuration (`Config`)

| Field | Env var | Default | Description |
|---|---|---|---|
| `Brokers` | `BROKERS` | — | Kafka cluster addresses |
| `Topics` | `TOPICS` | — | Topics to consume |
| `ConsumerGroup` | `CONSUMER_GROUP` | — | Consumer group ID |
| `SASLMechanism` | `SASL` | `PLAIN` | Auth: `PLAIN`, `SCRAM-SHA-256`, `SCRAM-SHA-512`, `NONE` |
| `User` | `USER` | — | SASL username |
| `Password` | `PASSWORD` | — | SASL password |
| `TLS` | `TLS` | `false` | Enable TLS (currently InsecureSkipVerify) |
| `RequestRetries` | `REQUEST_RETRIES` | `10` | Broker request retries |
| `RetryTimeout` | `RETRY_TIMEOUT` | `120s` | Retry timeout |
| `ConnIdleTimeout` | `CONN_IDLE_TIMEOUT` | `600s` | Connection idle timeout |
| `DialTimeout` | `DIAL_TIMEOUT` | `60s` | Connection dial timeout |
| `FetchMaxWait` | `FETCH_MAX_WAIT` | — | Max wait for fetch response |
| `SessionTimeout` | `SESSION_TIMEOUT` | `180s` | Consumer group session timeout |
| `MaxRecords` | `MAX_RECORDS` | `100` | Records per poll batch |
| `CommitMarks` | `COMMIT_MARKS` | `false` | Use auto-commit marks |
| `HandlerTimeout` | `HANDLER_TIMEOUT` | `600s` (10 min) | Timeout per handler invocation |
| `CommitTimeout` | `COMMIT_TIMEOUT` | `180s` (3 min) | Timeout for offset commit |
| `ShutdownTimeout` | `SHUTDOWN_TIMEOUT` | `25s` | Graceful shutdown deadline |
| `CloseHandlerCtxDelay` | `CLOSE_HANDLER_CTX_DELAY` | `5s` | Delay before cancelling handler context on shutdown |
| `ConnMaxRetries` | `CONN_MAX_RETRIES` | `30` | Connection retry attempts |
| `ConnRetryDelay` | `CONN_RETRY_DELAY` | `10s` | Delay between connection retries |

### 4.3 Functional Options

```go
func WithLogger(l logger.Logger) ServiceOpt                              // Custom logger
func WithTracer(t *otelKafka.Tracer) ServiceOpt                         // Custom OTEL Kafka tracer
func WithKgoOpts(opts ...kgo.Opt) ServiceOpt                            // Raw franz-go client options
func WithCommitCallback(f func(cl *kgo.Client, err error)) ServiceOpt   // Callback after commit
```

### 4.4 Config Helpers

`Config` has value-receiver helper methods for common overrides:

```go
cfg := consumer.Config{ /* ... */ }
cfg = cfg.WithTopic("orders")              // sets single topic
cfg = cfg.WithConsumerGroup("order-worker") // sets group
```

### 4.5 Consumer Lifecycle

```
New() → Prepare() → Start()/Run() → Consume() loop → Shutdown()
```

**`New(cfg, handleFunc, opts...)`** — Creates the Service; does NOT connect.

**`Prepare(ctx)` / `Init()`** — Connects to Kafka:
- Resolves SASL mechanism.
- Configures `kgo.Client` (brokers, topics, group, hooks, timeouts).
- Registers Prometheus metrics (`kprom`).
- Applies `HandlerWithTimeout` wrapper if `CloseHandlerCtxDelay > 0`.
- Pings the cluster.

**`Start(ctx)`** — Enters the poll-consume loop with retry logic:
- Uses `recovery.RecoveryCall` for connection retries (`ConnMaxRetries` × `ConnRetryDelay`).
- Retries only on retryable broker errors (`kgo.IsRetryableBrokerErr`); `context.Canceled` is NOT retried.
- After exhausting retries, returns a fatal error.

**`Run(ctx)`** — Same as `Start()` but panics on error (legacy pattern, prefer `Start`).

**`Shutdown(ctx)`** — Graceful shutdown:
1. Cancels the context (stops poll loop).
2. Waits for `gracefulShutdownCh` or `ShutdownTimeout`.
3. Cancels commit offset context.
4. Calls `consumer.CloseAllowingRebalance()`.

### 4.6 Handler Function

```go
type HandleFunc func(ctx context.Context, records []*kgo.Record) error
```

Key behaviors:
- Handler receives a **batch** of records (up to `MaxRecords`).
- Handler context has a **timeout** (`HandlerTimeout`, default 10 min).
- On shutdown, `CloseHandlerCtxDelay` provides a grace period before handler context cancellation.
- OTEL middleware wraps your handler: creates a "Consume Batch" span, records batch metrics (processing time, latest consumed timestamp, records processed count), and extracts trace context from each record.

### 4.7 Commit Strategies

**Auto-commit marks (`CommitMarks = true`):**
- Records are marked via `consumer.MarkCommitRecords()`.
- `CommitMarkedOffsets()` is called on shutdown or when context is done.
- Best for at-least-once with automatic offset management.

**Manual commit (`CommitMarks = false`, default):**
- `consumer.CommitRecords()` is called after each successful handler invocation.
- Best for explicit at-least-once delivery.

**Commit callback:** Use `WithCommitCallback()` to observe commit results (useful for monitoring or logging).

### 4.8 `HandlerWithTimeout` Middleware

```go
func HandlerWithTimeout(delay time.Duration, next HandleFunc) HandleFunc
```

Wraps a handler to gracefully handle shutdown:
- Creates a new handler context (decoupled from consumer context).
- Watches consumer context; after it's cancelled, waits `delay` before cancelling the handler context.
- This gives the handler time to finish processing after the consumer receives a shutdown signal.

### 4.9 Full Consumer Example

```go
package main

import (
    "context"
    "encoding/json"
    "fmt"

    "github.com/twmb/franz-go/pkg/kgo"
    "gitlab.wildberries.ru/reviews-ext/go-foundation/pkg/app"
    "gitlab.wildberries.ru/reviews-ext/go-foundation/pkg/consumer/v2"
    "gitlab.wildberries.ru/reviews-ext/go-foundation/pkg/logger"
)

type OrderEvent struct {
    OrderID string `json:"order_id"`
    Action  string `json:"action"`
}

func handleOrders(ctx context.Context, records []*kgo.Record) error {
    for _, rec := range records {
        var event OrderEvent
        if err := json.Unmarshal(rec.Value, &event); err != nil {
            // Log and skip malformed messages — do NOT return error
            // unless you want the whole batch to fail and be retried.
            continue
        }
        fmt.Printf("Processing order %s: %s\n", event.OrderID, event.Action)
    }
    return nil
}

func main() {
    application := app.NewApp[struct{}]("1.0.0")

    cfg := consumer.Config{
        Brokers:       []string{"kafka:9092"},
        Topics:        []string{"orders"},
        ConsumerGroup: "order-processor",
        SASLMechanism: "PLAIN",
        User:          "user",
        Password:      "pass",
        MaxRecords:    50,
        CommitMarks:  true,
    }

    c := consumer.New(cfg, handleOrders,
        consumer.WithLogger(application.Logger()),
    )

    if err := c.Prepare(context.Background()); err != nil {
        panic(err)
    }

    application.Add(c)
    application.Run(context.Background())
}
```

### 4.10 Important Notes

- **Batch processing:** `HandleFunc` receives a batch, not a single record. Always iterate over `records`.
- **Error propagation:** Returning an error from `HandleFunc` prevents offset commit and causes the batch to be re-delivered. For malformed messages, log and skip.
- **Metrics:** Prometheus metrics are auto-registered via `kprom` with labels: `broker`, `consumer_group`, `topic`.
- **TLS:** Currently uses `InsecureSkipVerify: true`. Production deployments should configure proper certificates.
- **Retry logic:** Connection retries use constant backoff (`ConnRetryDelay`) with `ConnMaxRetries` attempts. Only retryable broker errors trigger retry; context cancellation does not.
- **`AllowRebalance()`** is called after each `PollRecords` cycle via `defer`.

---

## 5. Integration with `app.Manager`

All three services implement `ManagedService` and integrate seamlessly:

```go
func main() {
    application := app.NewApp[MyConfig]("1.0.0")
    log := application.Logger()

    // HTTP server
    srv := server.NewFiberServer(log, httpCfg, router,
        server.WithRecover(),
        server.WithDefaultErrHandler(),
    )

    // Kafka producer
    prod := producer.New(&prodCfg, producer.WithLogger(log))

    // Kafka consumer
    cons := consumer.New(consCfg, handleFunc, consumer.WithLogger(log))

    // Register all services — order matters for shutdown (reversed)
    application.Add(srv, prod, cons)

    // Run blocks until SIGTERM or context cancellation
    application.Run(context.Background())
}
```

**Shutdown order:** cons → prod → srv (reverse of add order).

### Using `RunConfigurableService`

For a functional setup pattern that also handles configuration:

```go
func main() {
    application := app.NewApp[MyConfig]("1.0.0")

    err := application.RunConfigurableService(context.Background(),
        func(cfg config.Config[MyConfig], log logger.Logger) ([]app.ManagedService, error) {
            // Build all services using cfg and log
            srv := server.NewFiberServer(log, cfg.ServiceConfig.HTTP, router, /* opts */)
            cons := consumer.New(cfg.ServiceConfig.Consumer, handleFunc)
            return []app.ManagedService{srv, cons}, nil
        },
    )
    if err != nil {
        panic(err)
    }
}
```

---

## 6. Common Patterns & Pitfalls

### ✅ DO

- **Always call `Prepare()` before `Start()`** — this establishes the connection and validates config.
- **Use `app.Add()` + `app.Run()`** for lifecycle management — it handles signals, sequential init, concurrent start, and ordered shutdown.
- **Handle batch semantics in `HandleFunc`** — iterate over all records, not just the first.
- **Use `ProduceSync()`** when delivery guarantee is needed before proceeding.
- **Set `CommitMarks = true`** for most consumer use cases — it simplifies offset management.
- **Use `WithDefaultAuthMiddleware()`** unless you have a custom auth setup.
- **Use `WithRecover()`** to prevent panics from crashing the HTTP server.

### ❌ DON'T

- **Don't call `Run()` on consumer** in new code — use `Start()` which returns errors properly.
- **Don't return errors from `HandleFunc` for malformed messages** — the entire batch will be retried. Log and skip instead.
- **Don't skip `Prepare()`** — without it, the client won't connect and you'll get nil pointer panics.
- **Don't create `kgo.Record` without a topic** when the producer `Config.Topic` is empty — set the `Topic` field explicitly on each record.
- **Don't rely on `Start()` on producer** for any behavior — it's a no-op that just stores a cancel function.

---

## 7. Quick Reference: Service Lifecycle Methods

| Method | `FiberServer` | `producer.Service` | `consumer.Service` |
|---|---|---|---|
| `New()` / `NewFiberServer()` | Creates struct, applies opts | Creates struct, applies opts | Creates struct, applies opts |
| `Init()` / `Prepare()` | Creates Fiber app, registers routes, applies middleware | Creates `kgo.Client`, pings broker | Creates `kgo.Client`, pings broker |
| `Start()` | Starts HTTP listener (with retries) | No-op (stores cancel) | Enters poll-consume loop with retries |
| `Run()` | Panics on error | No-op (stores cancel) | Panics on error |
| `Shutdown()` | Graceful Fiber shutdown | Closes `kgo.Client` | Cancels context, waits for handler, closes client |

---

## 8. Environment Variable Reference

### HTTP Server

```env
APP_NAME=my-service
HOST=0.0.0.0
PORT=8080
RETRY_COUNT=3
FIBER_READ_TIMEOUT=10
FIBER_WRITE_TIMEOUT=10
FIBER_IDLE_TIMEOUT=60
FIBER_BODY_LIMIT=10485760
FIBER_READ_BUFFER_SIZE=131072
FIBER_WRITE_BUFFER_SIZE=131072
FIBER_STRICT_ROUTING=true
FIBER_CASE_SENSITIVE=true
FIBER_DISABLE_STARTUP_MESSAGE=true
FIBER_DISABLE_KEEPALIVE=true
FIBER_CONCURRENCY=524288
AUTHZ_KEYS_LOAD_URL=https://auth.example.com/keys
AUTHZ_CONFIG_LOAD_URL=https://auth.example.com/config
AUTHZ_LOGOFFS_LOAD_URL=https://auth.example.com/logoffs
ALLOW_ORIGINS=https://www.example.com
```

### Kafka Producer

```env
BROKERS=kafka:9092
SASL=PLAIN
USER=producer-user
PASSWORD=producer-pass
TOPIC=my-topic
REQUEST_RETRIES=3
RETRY_TIMEOUT=5s
CONN_IDLE_TIMEOUT=30s
DIAL_TIMEOUT=30s
```

### Kafka Consumer

```env
BROKERS=kafka:9092
TOPICS=orders,events
CONSUMER_GROUP=order-processor
SASL=PLAIN
USER=consumer-user
PASSWORD=consumer-pass
TLS=false
REQUEST_RETRIES=10
RETRY_TIMEOUT=120s
CONN_IDLE_TIMEOUT=600s
DIAL_TIMEOUT=60s
FETCH_MAX_WAIT=5s
SESSION_TIMEOUT=180s
MAX_RECORDS=100
COMMIT_MARKS=true
HANDLER_TIMEOUT=600s
COMMIT_TIMEOUT=180s
SHUTDOWN_TIMEOUT=25s
CLOSE_HANDLER_CTX_DELAY=5s
CONN_MAX_RETRIES=30
CONN_RETRY_DELAY=10s
```