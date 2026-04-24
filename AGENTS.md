# Staff Go Engineer Profile

## About Me

**Role:** Staff Software Engineer specializing in Go, distributed systems, and enterprise monorepo architecture.
**Context:** Large-scale monorepositories with layered architecture (transport → service → repository), event-driven systems, and polyglot persistence (PostgreSQL, ClickHouse, Redis, Kafka).
**Go Version:** 1.26+

### Language Preferences

- **Responses:** Russian (русский) — concise, technical, no fluff.
- **Reasoning / ADR / Code:** English — all architectural reasoning, decision records, code comments, and documentation must be in English.
- **Code:** English only. Comments in English. Variable names in English.

### Response Style

1. **Main point first.** Lead with the conclusion or action.
2. **Details second.** Provide supporting evidence, context, or alternatives.
3. **No filler.** No "I think", "perhaps", "it seems". State facts or qualified risks.
4. **Structured reasoning.** Use bullet points, numbered lists, or short paragraphs. One idea per line.
5. **Code blocks for everything executable.** Shell commands, Go snippets, config fragments.

### Caveman Mode (Optional)

When asked for quick fixes or trivial tasks:
- Minimal words. Preserve meaning.
- Sentence fragments. No full sentences.
- Remove articles, filler, politeness.
- Keep code, paths, errors unchanged.
- Cause → effect → fix.

## Development General Guidelines

### Architecture Principles

- **Layered monorepo:** `transport` → `service` → `repository`. No bypassing layers.
- **Explicit dependencies:** Constructor injection. No globals, no init() side effects.
- **Context propagation:** Every I/O operation accepts `context.Context`. Respect cancellation and deadlines.
- **Error handling:** Explicit. No swallowed errors. Wrap with domain context using `fmt.Errorf("...: %w", err)`.
- **Observability-first:** Traces, metrics, structured logs on every critical path. Use OpenTelemetry.
- **Backwards compatibility:** API changes must be non-breaking or versioned. Database migrations must be zero-downtime.
- **Domain isolation:** Bounded contexts. Anti-corruption layers at service boundaries.

### Go-Specific Rules

- **Idiomatic Go.** Prefer simplicity over cleverness. Avoid `interface{}` and `any` when possible.
- **Concurrency safety.** Share memory by communicating. Use `errgroup` for fan-out. Protect shared state with `sync.Mutex` or channels.
- **Performance awareness.** Avoid allocations in hot paths. Use `sync.Pool` when justified. Profile before optimizing (`pprof`, `trace`).
- **Testing.** Table-driven tests. Mock via interfaces. Race detector (`-race`) on CI. Benchmark regressions checked with `benchstat`.
- **Linting.** `golangci-lint` is mandatory before commit. Default linters: `errcheck`, `gosimple`, `govet`, `ineffassign`, `staticcheck`, `unused`, `gosec`, `bodyclose`, `noctx`, `paralleltest`. Configurable via `.golangci.yml`.

### Database & Migrations

- **PostgreSQL** for transactional data. **ClickHouse** for analytics/event store.
- **Migrations:** `golang-migrate`. Every migration has `up` and `down`. Zero-downtime: add column → deploy code → remove column.
- **Repository pattern:** Interface per aggregate. No raw SQL in service layer. Query optimization via `EXPLAIN ANALYZE`.

### Communication & Transport

- **gRPC** for internal service communication. **REST (Fiber)** for public/client-facing APIs.
- **Kafka** for async event streaming. Idempotent consumers. At-least-once delivery.
- **Redis** for caching and rate limiting. Cache invalidation is explicit.

### DevOps & Deployment

- **K8s via Helm.** Charts live in-repo. No imperative `kubectl` in CI. `helm lint` + `helm template` validation.
- **GitLab CI.** Stages: lint → test (unit, race, integration) → build → security scan → deploy (staging/prod).
- **Observability:** OpenTelemetry traces + metrics. Jaeger/Tempo for tracing. Prometheus for metrics. Structured logs with `slog`.

### Agent Orchestration

When to spawn sub-agents:

- ** manager** → Entry point for any task. Orchestrates planning and delegation.
- ** architect** → New feature, system redesign, service integration, ADR needed.
- ** goplan** → Any Go code change needing detailed planning. Use before `godev`.
- ** godev** → Any Go code change, performance issue, concurrency problem.
- ** reviewer** → Before PR, after significant change, security concern.
- ** test-engineer** → New feature, bug fix, refactoring, coverage gap.
- ** sqldev** → Schema change, migration, query optimization, repository interface change.
- ** docs** → Feature completion, release, API contract change.

Parallel execution rules:
- **Sequential:** `manager` (analysis) → `architect` (if needed) → `goplan` (if Go) → `godev` → `test-engineer` → `reviewer` → `docs` (if needed).
- **Reviewer feedback loop:** If `reviewer` finds issues → delegate back to `godev` or `sqldev` → re-run `reviewer`. Max 3 iterations. After 3rd iteration, if CRITICAL/HIGH remain → stop and show user.
- **Parallel:** `godev` + `sqldev` (after `architect` and `goplan`).
