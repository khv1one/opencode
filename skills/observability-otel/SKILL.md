---
name: observability-otel
description: >
  OpenTelemetry instrumentation: traces, metrics, structured logging.
  Health checks, graceful degradation, and span attributes.
  Use when adding observability to services or reviewing gaps.
---

# Observability with OpenTelemetry

Guide for instrumenting Go services with OpenTelemetry.

## When to Use

- Adding tracing/metrics to a new service
- Reviewing observability gaps
- Debugging latency in distributed systems
- Setting up health checks and alerts

## Three Pillars

### Traces

```go
import "go.opentelemetry.io/otel"

var tracer = otel.Tracer("service-name")

func HandleRequest(ctx context.Context, req Request) error {
    ctx, span := tracer.Start(ctx, "HandleRequest")
    defer span.End()

    span.SetAttributes(
        attribute.String("user.id", req.UserID),
        attribute.Int("items.count", len(req.Items)),
    )

    if err := process(ctx, req); err != nil {
        span.RecordError(err)
        span.SetStatus(codes.Error, err.Error())
        return err
    }

    return nil
}
```

Rules:
- Start span at every service boundary (handler, service method, external call).
- Set meaningful attributes. Don't log large payloads.
- Record errors with `span.RecordError(err)`.
- Always `defer span.End()`.
- Propagate context through all calls.

### Metrics

```go
import "go.opentelemetry.io/otel/metric"

var (
    requestCount, _ = meter.Int64Counter("requests_total")
    requestDuration, _ = meter.Int64Histogram("request_duration_ms")
)

func HandleRequest(ctx context.Context, req Request) error {
    start := time.Now()
    defer func() {
        requestDuration.Record(ctx, time.Since(start).Milliseconds())
    }()

    requestCount.Add(ctx, 1)
    // ...
}
```

Standard metrics:
- `requests_total` (counter) — total requests
- `request_duration_ms` (histogram) — latency distribution
- `errors_total` (counter) — error count by type
- `active_connections` (updowncounter) — current connections

### Logs (Structured)

```go
import "log/slog"

slog.Info("request handled",
    slog.String("trace_id", traceID),
    slog.String("user_id", userID),
    slog.Duration("duration", duration),
    slog.Int("status", status),
)
```

Rules:
- Use `log/slog` (Go 1.21+). JSON handler in production.
- Include `trace_id` and `span_id` in every log for correlation.
- No unstructured string concatenation.
- No sensitive data (passwords, tokens) in logs.

## Context Propagation

### HTTP

```go
import "go.opentelemetry.io/contrib/instrumentation/net/http/otelhttp"

handler := otelhttp.NewHandler(myHandler, "server")
```

### gRPC

```go
import (
    "go.opentelemetry.io/contrib/instrumentation/google.golang.org/grpc/otelgrpc"
    "google.golang.org/grpc"
)

grpc.NewServer(
    grpc.StatsHandler(otelgrpc.NewServerHandler()),
)
```

### Kafka

Propagate trace context in message headers:
```go
// Producer
 carrier := propagation.MapCarrier{}
 propagator.Inject(ctx, carrier)
 msg.Headers = toKafkaHeaders(carrier)

// Consumer
 carrier := fromKafkaHeaders(msg.Headers)
 ctx := propagator.Extract(ctx, carrier)
```

## Health Checks

```go
http.HandleFunc("/health", func(w http.ResponseWriter, r *http.Request) {
    // Check DB, Redis, etc.
    if !dbHealthy || !redisHealthy {
        w.WriteHeader(http.StatusServiceUnavailable)
        return
    }
    w.WriteHeader(http.StatusOK)
})
```

Types:
- **Liveness:** Is the process running? (kubelet restarts if failing)
- **Readiness:** Is the service ready to accept traffic? (removed from load balancer if failing)
- **Startup:** Has the service finished starting? (for slow-starting services)

## Graceful Degradation

When dependencies fail:
1. **Cache fallback:** Return stale cached data.
2. **Default response:** Return defaults for non-critical data.
3. **Circuit breaker:** Fail fast after threshold. Hystrix-like pattern.
4. **Always log and trace** the degradation.

## Alerting Rules

- P99 latency > threshold for 5 minutes
- Error rate > 1% for 5 minutes
- Goroutine count > baseline * 2
- Memory usage > limit
- Dependency (DB, Kafka) down

## Common Mistakes

| Mistake | Problem | Solution |
|---------|---------|----------|
| Missing trace propagation | Broken distributed traces | Inject/extract in all transports |
| Logging without trace_id | Can't correlate logs with traces | Add trace_id to every log |
| Spans without attributes | Low trace value | Add key identifiers |
| Panic in handler | Missing error spans | Recover and record error |
| No health checks | K8s can't detect failures | Implement /health, /ready |

## References

- [OpenTelemetry Go](https://opentelemetry.io/docs/languages/go/)
- [Go slog](https://pkg.go.dev/log/slog)
- [Google SRE Book - Monitoring](https://sre.google/sre-book/monitoring-distributed-systems/)
