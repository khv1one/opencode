---
name: go-concurrency-patterns
description: >
  Go concurrency patterns: errgroup, fan-out/fan-in, pipelines, worker pools,
  context cancellation, and rate limiting. Use when designing or reviewing
  concurrent Go code.
---

# Go Concurrency Patterns

Reference guide for safe, idiomatic concurrent Go code.

## When to Use

- Designing parallel task execution
- Handling multiple I/O operations concurrently
- Implementing worker pools or pipelines
- Reviewing concurrent code for races or leaks

## errgroup (Fan-Out)

```go
import "golang.org/x/sync/errgroup"

func ProcessItems(ctx context.Context, items []Item) error {
    g, ctx := errgroup.WithContext(ctx)
    g.SetLimit(10) // max concurrency

    for _, item := range items {
        item := item // capture loop var
        g.Go(func() error {
            return process(ctx, item)
        })
    }

    return g.Wait() // returns first error
}
```

Rules:
- Always pass `ctx` from `errgroup.WithContext` to goroutines.
- Set limit to prevent unbounded concurrency.
- First error cancels context; other goroutines should check `ctx.Done()`.

## Fan-Out / Fan-In

```go
func FanOut(ctx context.Context, inputs []Input) <-chan Result {
    out := make(chan Result)
    var wg sync.WaitGroup

    for _, in := range inputs {
        wg.Add(1)
        go func(in Input) {
            defer wg.Done()
            select {
            case out <- process(ctx, in):
            case <-ctx.Done():
            }
        }(in)
    }

    go func() {
        wg.Wait()
        close(out)
    }()

    return out
}
```

## Pipeline

```go
func Stage1(ctx context.Context, in <-chan int) <-chan int {
    out := make(chan int)
    go func() {
        defer close(out)
        for v := range in {
            select {
            case out <- v * 2:
            case <-ctx.Done():
                return
            }
        }
    }()
    return out
}
```

Rules:
- Close output channels only in the sender goroutine.
- Check `ctx.Done()` in long-running loops.
- Don't close input channels from consumers.

## Worker Pool

```go
func WorkerPool(ctx context.Context, jobs <-chan Job, workers int) <-chan Result {
    results := make(chan Result)
    var wg sync.WaitGroup

    for i := 0; i < workers; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            for job := range jobs {
                select {
                case results <- process(ctx, job):
                case <-ctx.Done():
                    return
                }
            }
        }()
    }

    go func() {
        wg.Wait()
        close(results)
    }()

    return results
}
```

## Context Cancellation

```go
func LongOperation(ctx context.Context) error {
    for {
        select {
        case <-ctx.Done():
            return ctx.Err()
        default:
        }
        // do work
    }
}
```

Rules:
- Pass `context.Context` as first argument.
- Respect `ctx.Done()`. Return `ctx.Err()` if cancelled.
- Use `context.WithTimeout` / `WithDeadline` for timeouts.
- Use `context.WithCancel` for manual cancellation.
- Don't store context in structs. Pass through call chain.

## sync.Map

Use only when:
- Keys are write-once, read-many.
- Multiple goroutines read/write different keys.
- Otherwise, `map + sync.RWMutex` is faster and clearer.

## Common Mistakes

| Mistake | Problem | Solution |
|---------|---------|----------|
| Closing channel from receiver | Panic | Close only in sender |
| Loop variable capture | All goroutines see last value | `v := v` inside loop |
| Missing ctx.Done() check | Goroutine leak | Check in loops, I/O |
| Unbounded goroutines | OOM | Use errgroup.SetLimit or buffered semaphores |
| Data race on map | Crash or corruption | `sync.Mutex` or `sync.Map` |
| Mutex + I/O | Slow, deadlock risk | Do I/O before/after lock |

## References

- [Go Memory Model](https://go.dev/ref/mem)
- [Concurrency in Go](https://www.oreilly.com/library/view/concurrency-in-go/9781491941294/)
