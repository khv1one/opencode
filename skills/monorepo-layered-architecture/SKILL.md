---
name: monorepo-layered-architecture
description: >
  Monorepo structure with layered architecture: transport, service, repository.
  Import restrictions, DTO mapping, domain isolation, and anti-corruption layers.
  Use when structuring new projects, adding features, or reviewing layer boundaries.
---

# Monorepo Layered Architecture

Reference for enterprise Go monorepo organization.

## When to Use

- Structuring a new monorepo or service
- Adding a feature (where does it belong?)
- Reviewing cross-layer imports or SRP violations
- Designing import restrictions

## Directory Structure

```
services/
├── user/
│   ├── cmd/
│   │   └── api/
│   │       └── main.go
│   ├── internal/
│   │   ├── transport/
│   │   │   ├── http/
│   │   │   │   ├── handler.go
│   │   │   │   ├── router.go
│   │   │   │   └── dto.go
│   │   │   └── grpc/
│   │   │       ├── server.go
│   │   │       └── converter.go
│   │   ├── service/
│   │   │   ├── user.go
│   │   │   ├── user_test.go
│   │   │   └── errors.go
│   │   └── repository/
│   │       ├── user.go
│   │       ├── postgres/
│   │       │   ├── user.go
│   │       │   └── queries.go
│   │       └── redis/
│   │           └── cache.go
│   ├── pkg/
│   │   └── domain/
│   │       └── user.go
│   └── migrations/
│       └── 001_init.up.sql
├── order/
│   └── ...
shared/
├── pkg/
│   ├── errors/
│   ├── logger/
│   ├── middleware/
│   └── tracing/
```

## Layer Rules

### Transport (http / grpc)
- **Responsibility:** Parse request, validate input, map to DTO, call service, format response.
- **No business logic.** No DB calls. No external service calls.
- **DTOs:** Separate from domain models. Conversion in transport layer.

### Service
- **Responsibility:** Business logic, orchestration, transaction boundaries.
- **No DB details.** Calls repository interfaces only.
- **No transport details.** Doesn't know about HTTP status codes or gRPC errors.
- **Errors:** Domain errors. Map to transport errors in transport layer.

### Repository
- **Responsibility:** Persistence abstraction. CRUD + queries.
- **Interface:** Defined in domain or service layer. Implementation in repository package.
- **No business logic.** Only data access patterns.
- **Transactions:** Managed at service layer. Repository methods accept tx or use connection pool.

## Dependency Direction

```
transport → service → repository → database
     ↓         ↓          ↓
   DTO      domain     models
```

- **Upper layers depend on lower layers.**
- **Lower layers don't know upper layers exist.**
- **Domain types flow upward.**

## Import Restrictions

Using `golang.org/x/tools/go/analysis` or custom lint:

```go
// Disallowed: transport importing repository directly
// Disallowed: service importing transport
// Disallowed: repository importing service
```

Example with `depguard` in `.golangci.yml`:
```yaml
linters-settings:
  depguard:
    rules:
      no_transport_to_repo:
        deny:
          - pkg: "services/user/internal/repository"
            desc: "transport must not import repository directly"
        files:
          - "services/user/internal/transport/**/*.go"
```

## Anti-Corruption Layer

When integrating with external services or legacy systems:

```go
// internal/acl/legacy/client.go
 type LegacyClient interface {
     GetUser(ctx context.Context, id string) (*domain.User, error)
 }
```

- Translate external models to domain models at the boundary.
- No external types leak into service layer.

## Monorepo Rules

- **No cross-service imports of `internal/` packages.** Use `pkg/` or API contracts.
- **Shared code in `shared/pkg/`.** No business logic in shared.
- **One `go.mod` per service** or one root `go.mod` with workspace. Choose and stick to it.
- **CI per service.** Only test affected services on change.

## DTO Mapping

```go
// transport/dto.go
 type CreateUserRequest struct {
     Name  string `json:"name" validate:"required"`
     Email string `json:"email" validate:"required,email"`
 }

 func (r CreateUserRequest) ToDomain() *domain.User {
     return &domain.User{
         Name:  r.Name,
         Email: r.Email,
     }
 }
```

Rules:
- Transport DTOs validate and convert.
- Domain models contain business rules.
- Repository models (if ORM) convert to/from domain.

## Common Mistakes

| Mistake | Problem | Solution |
|---------|---------|----------|
| Handler calls repository | Bypasses business logic | Route through service |
| Service returns HTTP errors | Leaks transport details | Return domain errors, map in transport |
| Domain types in DTOs | Couples transport to domain | Use separate DTOs |
| Cross-service internal imports | Breaks encapsulation | Use API or shared pkg |
| Business logic in repository | SRP violation | Move to service |

## References

- [Clean Architecture](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html)
- [Go Project Layout](https://github.com/golang-standards/project-layout)
