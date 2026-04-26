---
name: go-foundation-worker
description: a
---

# go-foundation Worker Package Skill

> **Module:** `gitlab.wildberries.ru/reviews-ext/go-foundation/pkg/worker`
> **Audience:** AI agents (e.g., `godev`) that need to create, configure, and manage background workers using this library.

---

## Table of Contents

1. [Overview](#overview)
2. [Core Concepts](#core-concepts)
3. [Worker Types](#worker-types)
   - [SoloWorker](#soloworker)
   - [UniqWorker](#uniqworker)
4. [Leader Election (RedisSync)](#leader-election-redissync)
5. [Configuration Reference](#configuration-reference)
6. [Lifecycle & Interface Compliance](#lifecycle--interface-compliance)
7. [Usage Patterns](#usage-patterns)
   - [SoloWorker — Basic Background Job](#soloworker--basic-background-job)
   - [UniqWorker — Single-Instance Distributed Job](#uniqworker--single-instance-distributed-job)
   - [UniqWorker with Job Interface](#uniqworker-with-job-interface)
   - [Integration with app.Manager](#integration-with-appmanager)
   - [Graceful Shutdown Tuning](#graceful-shutdown-tuning)
8. [Prometheus Metrics](#prometheus-metrics)
9. [Decision Guide: SoloWorker vs UniqWorker](#decision-guide-soloworker-vs-uniqworker)
10. [Common Pitfalls](#common-pitfalls)
11. [Redis Key Format](#redis-key-format)

---

## Overview

The `worker` package provides two types of background job runners for Go services:

| Type | Coordination | Use Case |
|---|---|---|
| **SoloWorker** | None (single-instance) | Local background tasks, no Redis required |
| **UniqWorker** | Redis-based leader election | Distributed single-instance jobs across multiple pods |

Both workers:
- Execute a `DoFunc` on a configurable interval.
- Expose Prometheus metrics (leader state, job duration, error counts).
- Support `ManagedService` lifecycle (`Prepare`/`Start`/`Shutdown`).
- Support legacy `Service` lifecycle via `ManagedServiceAdapter` (`Init`/`Run`/`Stop`).

---

## Core Concepts

### DoFunc

The business logic function signature shared by both worker types:

```go
type DoFunc func(context.Context) error
```

- Receives a `context.Context` that is cancelled on shutdown or job timeout.
- Must be idempotent — the worker may re-execute after a leader change or restart.
- Return `nil` on success; the worker logs errors but does not retry.

### Role Model

```go
type Role int

const (
    Follower Role = iota // Not executing jobs
    Leader               // Executing jobs
)
```

- **SoloWorker** is always `Leader`.
- **UniqWorker** starts as `Follower` and transitions to `Leader` via Redis-based election.

### Job Interface

A convenience interface for services with multiple named jobs:

```go
type Job interface {
    Do(ctx context.Context) error
    Name() string
    Interval() time.Duration
}
```

---

## Worker Types

### SoloWorker

A simple timer-based background worker with no distributed coordination.

**When to use:**
- No need for single-instance guarantees (multiple pods can run the same job concurrently).
- No Redis available in the deployment environment.
- Local/non-critical periodic tasks (metrics scraping, cache warming, etc.).

**Key characteristics:**
- Always considers itself `Leader` (sets metric to 1).
- Executes `DoFunc` on a fixed interval using `time.Timer`.
- No graceful shutdown delay — context is cancelled immediately on `Stop`.
- No external dependencies (no Redis).

#### Constructor

```go
func NewSoloWorker(
    conf     SoloWorkerConf,
    l        Logger,
    do       DoFunc,
    interval time.Duration,
    jobName  string,
) *SoloWorker
```

#### Configuration

```go
type SoloWorkerConf struct {
    EpicName    string        `env:"EPIC_NAME"     envDefault:""    yaml:"epic-name"`
    AppName     string        `env:"APP_NAME"      envDefault:""    yaml:"app-name"`
    InitTimeout time.Duration `env:"INIT_TIMEOUT"  envDefault:"10s" yaml:"init-timeout"`
}
```

- `EpicName` / `AppName`: Used for Prometheus metric labels and Redis key construction.
- `InitTimeout`: Timeout for the `Init()` / `Prepare()` call (default 10s).

---

### UniqWorker

A distributed leader-election worker backed by Redis. Only one instance across all pods holds `Leader` role and executes the job.

**When to use:**
- Job must run on **exactly one instance** at a time (e.g., DB cleanup, report generation, state reconciliation).
- Running multiple instances concurrently would cause data corruption or duplicates.
- Redis is available (Cluster, Sentinel, or single-node).

**Key characteristics:**
- Leader election via Redis `SETNX` with TTL (default 15s).
- Leader renews its TTL periodically (default every 4s).
- On shutdown, leader deletes its Redis key for fast failover.
- If the leader loses its lock (network partition, Redis failure), it immediately cancels the running job context.
- Supports **graceful shutdown** with `CloseJobCtxDelay`: gives running jobs time to complete before context cancellation.
- Job timeout enforced via `JobTimeout` (default 10 minutes).

#### Constructors

| Constructor | Redis Mode | Use When |
|---|---|---|
| `NewUniqWorker(conf, l, do, interval, jobName, client)` | Custom `redis.UniversalClient` | You manage the Redis client yourself |
| `NewUniqWorkerWithDefaultClient(conf, l, do, jobName, interval)` | Sentinel/Failover | Production with Redis Sentinel |
| `NewUniqWorkerWithClusterClient(conf, l, do, jobName, interval)` | Cluster | Production with Redis Cluster |
| `NewUniqWorkerWithSingleClient(conf, l, do, jobName, interval)` | Single node | Development / testing |

> **Note the argument order difference:**
> - `NewUniqWorker`: `interval` before `jobName`
> - `NewUniqWorkerWith*Client`: `jobName` before `interval`

#### Configuration

```go
type UniqWorkerConf struct {
    RedisConf RedisWorkConf `envPrefix:"SYNC_" yaml:"sync"`

    EpicName    string        `env:"EPIC_NAME"     envDefault:""    yaml:"epic-name"`
    AppName     string        `env:"APP_NAME"      envDefault:""    yaml:"app-name"`
    InitTimeout time.Duration `env:"INIT_TIMEOUT"  envDefault:"10s" yaml:"init-timeout"`

    // Job execution & graceful shutdown
    JobTimeout       time.Duration `env:"JOB_TIMEOUT"        envDefault:"600s" yaml:"job-timeout"`
    ShutdownTimeout  time.Duration `env:"SHUTDOWN_TIMEOUT"   envDefault:"25s"  yaml:"shutdown-timeout"`
    CloseJobCtxDelay time.Duration `env:"CLOSE_JOB_CTX_DELAY" envDefault:"0"  yaml:"close-job-ctx-delay"`
}
```

| Field | Default | Description |
|---|---|---|
| `JobTimeout` | 10m | Maximum time a single job execution may run. Context is cancelled after this. |
| `ShutdownTimeout` | 25s | Overall timeout for `Shutdown()`. Must be ≥ `CloseJobCtxDelay`. |
| `CloseJobCtxDelay` | 0 | Grace period after shutdown signal before cancelling job context. `0` = no grace (immediate cancel). |

```go
type RedisWorkConf struct {
    ClientConf                RedisClientConf `envPrefix:"CLIENT_" yaml:"client"`
    LeaderTryGetInterval      time.Duration   `env:"LEADER_TRY_GET_INTERVAL"      envDefault:"1s"  yaml:"leader-try-get-interval"`
    LeaderBlockDuration       time.Duration   `env:"LEADER_BLOCK_DURATION"         envDefault:"15s" yaml:"leader-block-duration"`
    LeaderBlockUpdateInterval time.Duration   `env:"LEADER_BLOCK_UPDATE_INTERVAL"  envDefault:"4s"  yaml:"leader-block-update-interval"`
    StopTimeout               time.Duration   `env:"STOP_TIMEOUT"                  envDefault:"1s"  yaml:"stop-timeout"`
}
```

| Field | Default | Description |
|---|---|---|
| `LeaderTryGetInterval` | 1s | How often a follower tries to acquire leadership. |
| `LeaderBlockDuration` | 15s | Redis key TTL. If leader fails to renew within this window, a new leader is elected. |
| `LeaderBlockUpdateInterval` | 4s | How often the leader renews its TTL. Must be < `LeaderBlockDuration`. |
| `StopTimeout` | 1s | Timeout for Redis DEL during graceful shutdown. |

```go
type RedisClientConf struct {
    Username           string        `env:"USER"                  envDefault:"client" yaml:"user"`
    Password            string        `env:"PASSWORD"              envDefault:""       yaml:"password"`
    MasterName          string        `env:"MASTER_NAME"           envDefault:"master" yaml:"master-name"`
    Addresses           []string      `env:"ADDRESSES"             yaml:"addresses"`
    PoolSize            int           `env:"POOL_SIZE"             envDefault:"10"     yaml:"pool-size"`
    PoolTimeout         time.Duration `env:"POOL_TIMEOUT"          envDefault:"2s"     yaml:"pool-timeout"`
    ReadTimeout         time.Duration `env:"READ_TIMEOUT"          envDefault:"1s"     yaml:"read-timeout"`
    WriteTimeout        time.Duration `env:"WRITE_TIMEOUT"         envDefault:"1s"     yaml:"write-timeout"`
    MaxRetries          int           `env:"MAX_RETRIES"           envDefault:"6"      yaml:"max-retries"`
    MinRetriesInterval  time.Duration `env:"MIN_RETRIES_INTERVAL"  envDefault:"500ms"  yaml:"min-retries-interval"`
    MaxRetriesInterval  time.Duration `env:"MAX_RETRIES_INTERVAL"  envDefault:"1s"     yaml:"max-retries-interval"`
}
```

---

## Leader Election (RedisSync)

The `RedisSync` struct implements the `Syncer` interface and manages leader election via Redis.

### Algorithm

1. **Acquire leadership**: The worker calls `SETNX` on a Redis key (`uniq_worker:{epic}:{app}:{jobName}`) with the `LeaderBlockDuration` TTL. The value contains the worker's UUID.
2. **Maintain leadership**: A goroutine periodically calls `EXPIRE` on the key (every `LeaderBlockUpdateInterval`, default 4s) to renew the TTL.
3. **Verify leadership**: Another goroutine periodically reads the key value. If the stored UUID does not match the local UUID, the worker demotes itself to `Follower`.
4. **Release leadership**: On `Stop()`, the worker calls `DEL` on the key to allow immediate failover.
5. **Network resilience**: Redis client retries up to `MaxRetries` (default 6) with exponential backoff between `MinRetriesInterval` (500ms) and `MaxRetriesInterval` (1s).

### Role Transitions

```
┌──────────┐  SETNX succeeds  ┌──────────┐
│ Follower │ ────────────────► │  Leader  │
└──────────┘                  └──────────┘
     ▲                            │
     │  UUID mismatch /           │  EXPIRE fails /
     │  Redis error /             │  leadershipCheck fails
     │  Stop signal               │
     └────────────────────────────┘
```

When a `Follower` becomes `Leader`, the job timer is immediately reset (fires at `0` duration), so the first execution happens right away.

---

## Lifecycle & Interface Compliance

### ManagedService Interface (Preferred)

Both `SoloWorker` and `UniqWorker` implement `ManagedService`:

```go
type ManagedService interface {
    Prepare(ctx context.Context) error
    Start(ctx context.Context) error
    Shutdown(ctx context.Context) error
}
```

| Method | Behavior |
|---|---|
| `Prepare(ctx)` | Initializes metrics, connects to Redis (UniqWorker), pings Redis. Respects `InitTimeout`. |
| `Start(ctx)` | Starts the leader election loop (UniqWorker) and the job execution goroutine. |
| `Shutdown(ctx)` | Cancels context, releases leadership (UniqWorker), waits for goroutines. For UniqWorker: supports graceful delay via `CloseJobCtxDelay`. |

### Legacy Service Interface (Deprecated)

```go
type Service interface {
    Init() error
    Run(ctx context.Context)
    Stop()
}
```

Both workers also support this interface. Use `ManagedServiceAdapter` to wrap for `app.Manager`:

```go
app.AddService(NewManagedServiceAdapter(worker))
```

> **Prefer `Prepare`/`Start`/`Shutdown` directly.** The legacy adapter wraps `Init→Prepare`, `Run→Start`, and `Stop→Shutdown` with `context.Background()`.

---

## Usage Patterns

### SoloWorker — Basic Background Job

```go
package main

import (
    "context"
    "time"

    "gitlab.wildberries.ru/reviews-ext/go-foundation/pkg/worker"
    "gitlab.wildberries.ru/reviews-ext/go-foundation/pkg/logger"
)

func main() {
    log := logger.Default()

    conf := worker.SoloWorkerConf{
        EpicName: "my-epic",
        AppName:  "my-service",
    }

    do := func(ctx context.Context) error {
        log.Info("running periodic cleanup")
        // Business logic here...
        return nil
    }

    w := worker.NewSoloWorker(conf, log, do, 30*time.Second, "cleanup-job")

    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()

    if err := w.Prepare(ctx); err != nil {
        log.Error("failed to prepare worker", "error", err)
        return
    }

    if err := w.Start(ctx); err != nil {
        log.Error("failed to start worker", "error", err)
        return
    }

    // Block until signal...
    // On shutdown:
    w.Shutdown(context.Background())
}
```

### UniqWorker — Single-Instance Distributed Job

#### With Single Redis (Dev/Test)

```go
conf := worker.UniqWorkerConf{
    EpicName: "my-epic",
    AppName:  "my-service",
    RedisConf: worker.RedisWorkConf{
        ClientConf: worker.RedisClientConf{
            Addresses: []string{"localhost:6379"},
        },
    },
}

w := worker.NewUniqWorkerWithSingleClient(
    conf, log, doFunc, "report-gen", 5*time.Minute,
)
```

#### With Redis Sentinel (Production)

```go
conf := worker.UniqWorkerConf{
    EpicName: "my-epic",
    AppName:  "my-service",
    RedisConf: worker.RedisWorkConf{
        ClientConf: worker.RedisClientConf{
            MasterName: "mymaster",
            Addresses:  []string{"sentinel1:26379", "sentinel2:26379", "sentinel3:26379"},
        },
    },
}

w := worker.NewUniqWorkerWithDefaultClient(
    conf, log, doFunc, "report-gen", 5*time.Minute,
)
```

#### With Redis Cluster (Production)

```go
conf := worker.UniqWorkerConf{
    EpicName: "my-epic",
    AppName:  "my-service",
    RedisConf: worker.RedisWorkConf{
        ClientConf: worker.RedisClientConf{
            Addresses: []string{"node1:6379", "node2:6379", "node3:6379"},
        },
    },
}

w := worker.NewUniqWorkerWithClusterClient(
    conf, log, doFunc, "report-gen", 5*time.Minute,
)
```

#### With Custom Redis Client

```go
import "github.com/redis/go-redis/v9"

client := redis.NewClient(&redis.Options{
    Addr: "localhost:6379",
})

w := worker.NewUniqWorker(conf, log, doFunc, 5*time.Minute, "report-gen", client)
```

### UniqWorker with Job Interface

When a service has multiple jobs, implement the `Job` interface:

```go
type ReportJob struct{}

func (j *ReportJob) Do(ctx context.Context) error {
    // Generate report...
    return nil
}
func (j *ReportJob) Name() string          { return "daily-report" }
func (j *ReportJob) Interval() time.Duration { return 24 * time.Hour }

type CleanupJob struct{}

func (j *CleanupJob) Do(ctx context.Context) error {
    // Clean up stale records...
    return nil
}
func (j *CleanupJob) Name() string          { return "stale-cleanup" }
func (j *CleanupJob) Interval() time.Duration { return 1 * time.Hour }

func registerWorkers(app *app.Manager, log logger.Logger, cfg worker.UniqWorkerConf) {
    jobs := []worker.Job{
        &ReportJob{},
        &CleanupJob{},
    }

    for _, job := range jobs {
        w := worker.NewUniqWorkerWithSingleClient(
            cfg, log,
            func(ctx context.Context) error {
                return job.Do(ctx)
            },
            job.Name(),
            job.Interval(),
        )
        app.Add(w)
    }
}
```

### Integration with app.Manager

The `app.Manager` orchestrates lifecycle for multiple services:

```go
package main

import (
    "context"
    "time"

    "gitlab.wildberries.ru/reviews-ext/go-foundation/pkg/app"
    "gitlab.wildberries.ru/reviews-ext/go-foundation/pkg/logger"
    "gitlab.wildberries.ru/reviews-ext/go-foundation/pkg/worker"
)

func main() {
    log := logger.Default()

    workerConf := worker.UniqWorkerConf{
        EpicName:         "my-epic",
        AppName:          "my-service",
        CloseJobCtxDelay: 30 * time.Second, // Give jobs 30s to finish on shutdown
        ShutdownTimeout:  45 * time.Second, // Must be > CloseJobCtxDelay
        RedisConf: worker.RedisWorkConf{
            ClientConf: worker.RedisClientConf{
                Addresses: []string{"redis:6379"},
            },
        },
    }

    doFunc := func(ctx context.Context) error {
        // Business logic...
        return nil
    }

    w := worker.NewUniqWorkerWithSingleClient(
        workerConf, log, doFunc, "my-job", 1*time.Minute,
    )

    mngr := app.NewManager(
        log,
        app.WithShutdownDelay(500*time.Millisecond),
        app.WithShutdownTimeout(60*time.Second),
    )
    mngr.Add(w)

    if err := mngr.Run(context.Background()); err != nil {
        log.Error("manager exited with error", "error", err)
    }
}
```

**Manager lifecycle:**

1. Calls `Prepare(ctx)` on each service in order.
2. Calls `Start(ctx)` on each service (each runs in its own goroutine).
3. Waits for OS signal (SIGINT/SIGTERM) or context cancellation.
4. Calls `Shutdown(ctx)` on each service in **reverse** order, respecting `shutdownDelay` and `shutdownMaxWaitTime`.

### Graceful Shutdown Tuning

UniqWorker supports two shutdown modes:

#### Hard Shutdown (Default: `CloseJobCtxDelay = 0`)

- When `Stop()`/`Shutdown()` is called, the job context is cancelled immediately.
- The job must check `ctx.Done()` and exit promptly.
- Use when jobs are cheap to restart or idempotent.

#### Graceful Shutdown (`CloseJobCtxDelay > 0`)

```go
conf := worker.UniqWorkerConf{
    // ...
    JobTimeout:       10 * time.Minute,  // Max time for a single job run
    ShutdownTimeout:  45 * time.Second,  // Overall shutdown timeout
    CloseJobCtxDelay: 30 * time.Second,  // Grace period for running job
}
```

How it works:
1. `Shutdown()` is called. The worker cancels its internal context and stops accepting new job ticks.
2. The running job's context is **not** immediately cancelled. Instead, `context.WithoutCancel()` detaches it.
3. After `CloseJobCtxDelay` (30s in this example), the job context is forcefully cancelled.
4. `Shutdown()` waits up to `ShutdownTimeout` for all goroutines to finish.

**Constraint:** `ShutdownTimeout ≥ CloseJobCtxDelay`. If violated, `CloseJobCtxDelay` is clamped to `ShutdownTimeout`.

---

## Prometheus Metrics

The package registers the following Prometheus metrics (via `GetUniqWorkerMetrics`):

| Metric | Type | Labels | Description |
|---|---|---|---|
| `wb_worker_leader_role_state` | Gauge | `epic`, `service`, `method` | 1 if this instance is leader for the job, 0 otherwise |
| `wb_worker_sync_errors_count` | Counter | `epic`, `service`, `method`, `msg` | Count of leader election errors (by type: `errNotLeader`, `errRedisGet`, `errTakeLeadership`, `errUnexpected`) |

Additionally, `WorkerMetrics` (from `telemetry/metrics` package) provides:

| Metric | Type | Description |
|---|---|---|
| Job state | Gauge | 1 if job is currently running, 0 otherwise |
| Job duration | Histogram | Duration of each job execution |
| Last job duration | Gauge | Duration of the most recent job execution |
| Job execution count | Counter | Total number of job executions (with error label) |

Metrics are automatically registered on first `Init()`/`Prepare()` call. The `epic` and `service` labels come from `SoloWorkerConf.EpicName`/`AppName` or `UniqWorkerConf.EpicName`/`AppName`.

---

## Decision Guide: SoloWorker vs UniqWorker

```
Does your job need to run on EXACTLY ONE instance at a time?
├── YES → Use UniqWorker
│   └── Which Redis deployment?
│       ├── Sentinel/Failover → NewUniqWorkerWithDefaultClient
│       ├── Cluster           → NewUniqWorkerWithClusterClient
│       ├── Single node       → NewUniqWorkerWithSingleClient
│       └── Custom client     → NewUniqWorker (provide redis.UniversalClient)
└── NO → Use SoloWorker
    └── (No Redis needed)
```

**Key considerations:**
- If you deploy a single replica and don't plan to scale, `SoloWorker` is simpler and avoids Redis.
- If you have ≥2 replicas, use `UniqWorker` to prevent duplicate execution.
- If Redis is unavailable in a specific environment (e.g., dev), use `SoloWorker` with a feature flag.
- The `reviews-ext-tools` namespace provides a dedicated Redis for these workers if your service doesn't have one.

---

## Common Pitfalls

### 1. Constructor Argument Order Differs

```go
// NewUniqWorker: interval BEFORE jobName
worker.NewUniqWorker(conf, log, do, 5*time.Minute, "job", client)

// NewUniqWorkerWith*Client: jobName BEFORE interval
worker.NewUniqWorkerWithSingleClient(conf, log, do, "job", 5*time.Minute)
```

Mixing up these arguments leads to silent bugs (job name used as interval, etc.).

### 2. CloseJobCtxDelay Must Be ≤ ShutdownTimeout

```go
// BAD: CloseJobCtxDelay > ShutdownTimeout — will be clamped
conf := worker.UniqWorkerConf{
    CloseJobCtxDelay: 60 * time.Second,
    ShutdownTimeout:  25 * time.Second, // will force CloseJobCtxDelay to 25s
}

// GOOD: ShutdownTimeout > CloseJobCtxDelay
conf := worker.UniqWorkerConf{
    CloseJobCtxDelay: 30 * time.Second,
    ShutdownTimeout:  45 * time.Second,
}
```

### 3. LeaderBlockUpdateInterval Must Be < LeaderBlockDuration

If `LeaderBlockUpdateInterval ≥ LeaderBlockDuration`, the leader may fail to renew its TTL before expiry, causing frequent leader flips.

```go
// GOOD
conf.RedisConf.LeaderBlockDuration = 15 * time.Second
conf.RedisConf.LeaderBlockUpdateInterval = 4 * time.Second

// BAD — will cause leadership flapping
conf.RedisConf.LeaderBlockDuration = 2 * time.Second
conf.RedisConf.LeaderBlockUpdateInterval = 4 * time.Second
```

### 4. DoFunc Must Be Idempotent

Leader failover can cause the same job to execute on a different instance. The `DoFunc` must handle this gracefully.

### 5. Always Call Prepare() or Init() Before Start()

Both workers need initialization (metrics, Redis connection). Skipping this step causes panics or nil-pointer dereferences.

### 6. Context Propagation in DoFunc

The `DoFunc` receives a context that:
- Is cancelled after `JobTimeout` (always).
- Is cancelled after `CloseJobCtxDelay` on shutdown (if > 0).
- Is cancelled immediately on shutdown (if `CloseJobCtxDelay = 0`).

**Always respect `ctx.Done()` in your business logic:**

```go
do := func(ctx context.Context) error {
    for _, item := range items {
        select {
        case <-ctx.Done():
            return ctx.Err()
        default:
            process(item)
        }
    }
    return nil
}
```

### 7. Redis Key Namespace Collision

Job names are namespaced as `uniq_worker:{epic}:{app}:{jobName}`. If two services share the same `epic:app` pair but have different jobs with the same name, they will conflict. Ensure job names are unique within an epic+app scope.

### 8. Don't Forget to Handle Logger Interface

The `Logger` interface requires:

```go
type Logger interface {
    Info(string, ...any)
    Error(string, ...any)
}
```

Use `logger.Default()` from the `logger` package, or inject your own implementation.

---

## Redis Key Format

All UniqWorker Redis keys follow the pattern:

```
uniq_worker:{EpicName}:{AppName}:{JobName}
```

Example: `uniq_worker:reviews:report-service:daily-report`

The value stored is JSON:

```json
{
  "ClientUUID": "550e8400-e29b-41d4-a716-446655440000",
  "LastRunTime": "0001-01-01T00:00:00Z"
}
```

- `ClientUUID`: Unique identifier for the current leader instance.
- `LastRunTime`: Reserved for future use (period synchronization between instances).