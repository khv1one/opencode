---
name: go-performance-optimization
description: >
  Go performance optimization techniques: profiling, allocation reduction,
  sync.Pool, escape analysis, and GC tuning. Use when optimizing hot paths
  or reviewing performance-critical code.
---

# Go Performance Optimization

Guide for data-driven performance optimization in Go.

## When to Use

- Hot path optimization
- Memory pressure reduction
- Benchmark regression investigation
- Reviewing performance-critical code

## Golden Rule

**Profile first. Optimize second. Measure always.**

## Profiling Tools

### CPU Profile

```bash
go test -bench=. -cpuprofile=cpu.out ./...
go tool pprof cpu.out
(pprof) top
(pprof) list FunctionName
(pprof) web
```

### Memory Profile

```bash
go test -bench=. -memprofile=mem.out ./...
go tool pprof mem.out
(pprof) top
(pprof) list FunctionName
```

### Execution Trace

```bash
go test -trace=trace.out ./...
go tool trace trace.out
```

Useful for:
- Goroutine scheduling delays
- Blocking operations
- GC pauses

### Benchmark Comparison

```bash
go test -bench=. -count=10 -benchmem > old.txt
# apply changes
go test -bench=. -count=10 -benchmem > new.txt
benchstat old.txt new.txt
```

## Allocation Reduction

### sync.Pool

```go
var bufPool = sync.Pool{
    New: func() any {
        b := make([]byte, 0, 1024)
        return &b
    },
}

func Process() {
    b := bufPool.Get().(*[]byte)
    defer bufPool.Put(b)
    *b = (*b)[:0]
    // use b
}
```

Rules:
- Pool is for reducing GC pressure, not for correctness.
- Objects may be garbage collected between Get and Put.
- Reset pooled object state before reuse.
- Don't pool objects with complex lifecycles or references.

### Preallocate Slices

```go
// Bad
var results []Result
for _, item := range items {
    results = append(results, process(item))
}

// Good
results := make([]Result, 0, len(items))
for _, item := range items {
    results = append(results, process(item))
}
```

### strings.Builder

```go
// Bad
var s string
for _, part := range parts {
    s += part
}

// Good
var b strings.Builder
b.Grow(estimatedSize)
for _, part := range parts {
    b.WriteString(part)
}
s := b.String()
```

### Escape Analysis

Check heap allocations:
```bash
go build -gcflags='-m' ./... 2>&1 | grep escapes
```

Common escape causes:
- Returning pointer to local variable (escape to heap)
- Interface method calls (receiver escapes)
- `...` variadic calls with slice literal
- Closure capturing loop variables

Mitigation:
- Pass by value for small structs.
- Avoid interfaces in hot paths if possible.
- Preallocate outside loops.

## GC Tuning

Environment variables:
- `GOGC=100` — default. Increase to reduce GC frequency (more memory).
- `GOMEMLIMIT=1GiB` — soft memory limit. GC will try to stay under it.

For services with large heaps:
```bash
GOMEMLIMIT=4GiB GOGC=200 ./service
```

## Concurrency Performance

- **GOMAXPROCS:** Usually leave at CPU count. For I/O-bound, may increase.
- **Channel buffering:** Unbuffered for synchronization, buffered for throughput.
- **Mutex vs atomic:** `atomic` for simple counters, `sync.Mutex` for complex state.

## Common Mistakes

| Mistake | Problem | Solution |
|---------|---------|----------|
| Optimizing without profiling | Wasted effort | Profile first |
| Premature optimization | Complex, brittle code | Measure benefit |
| sync.Pool for small objects | Overhead > benefit | Pool only large or frequently allocated objects |
| `interface{}` in hot paths | Allocation + type assertion | Use generics or concrete types |
| Reflection in hot paths | Slow, allocates | Code generation or type switches |

## References

- [Go Performance Wiki](https://github.com/golang/go/wiki/Performance)
- [High Performance Go](https://dave.cheney.net/high-performance-go-workshop)
- [Go Diagnostics](https://go.dev/doc/diagnostics)
