---
name: sql-migrations
description: >
  Database migrations with golang-migrate, zero-downtime patterns,
  rollback strategies, ClickHouse specifics, and partitioning.
  Use for schema changes, migration review, or database design.
---

# SQL Migrations

Guide for safe database schema evolution in production.

## When to Use

- Designing new schema
- Creating or reviewing migrations
- Planning zero-downtime changes
- ClickHouse schema design

## Tools

### golang-migrate

```bash
# Create
migrate create -ext sql -dir migrations -seq add_user_sessions

# Up
migrate -path migrations -database "postgres://..." up

# Down
migrate -path migrations -database "postgres://..." down 1

# Version
migrate -path migrations -database "postgres://..." version

# Force dirty (emergency)
migrate -path migrations -database "postgres://..." force VERSION
```

Migration file naming:
```
000001_init.down.sql
000001_init.up.sql
000002_add_index.down.sql
000002_add_index.up.sql
```

## Zero-Downtime Patterns

### Add Column (nullable or with default)

```sql
-- up
ALTER TABLE users ADD COLUMN email_verified BOOLEAN NOT NULL DEFAULT false;

-- backfill if needed (batched)
UPDATE users SET email_verified = true WHERE created_at < '2024-01-01';

-- after backfill and code deploy, remove default
ALTER TABLE users ALTER COLUMN email_verified DROP DEFAULT;

-- down
ALTER TABLE users DROP COLUMN email_verified;
```

### Add Column (non-nullable, no default)

```sql
-- up
ALTER TABLE users ADD COLUMN temp_col TEXT;
-- Deploy code that writes to temp_col
-- Backfill
UPDATE users SET temp_col = ... WHERE temp_col IS NULL;
-- Add NOT NULL
ALTER TABLE users ALTER COLUMN temp_col SET NOT NULL;
-- Rename (optional)
ALTER TABLE users RENAME COLUMN temp_col TO new_col;

-- down
ALTER TABLE users DROP COLUMN new_col;
```

### Add Index

```sql
-- up
CREATE INDEX CONCURRENTLY idx_users_email ON users(email);

-- down
DROP INDEX CONCURRENTLY IF EXISTS idx_users_email;
```

Rules:
- Always `CONCURRENTLY` for PostgreSQL. Prevents table locks.
- Don't run in transaction (golang-migrate: `-- +migrate StatementBegin` / `StatementEnd`).
- Verify index usage with `EXPLAIN ANALYZE`.

### Rename Column / Table

Don't rename directly. Use add + migrate + drop:
```sql
-- up
ALTER TABLE users ADD COLUMN new_name TYPE;
UPDATE users SET new_name = old_name;
-- Deploy code reading new_name
-- After stable:
ALTER TABLE users DROP COLUMN old_name;

-- down
ALTER TABLE users ADD COLUMN old_name TYPE;
UPDATE users SET old_name = new_name;
ALTER TABLE users DROP COLUMN new_name;
```

### Drop Column

```sql
-- up
-- Step 1: Stop writing to column (code deploy)
-- Step 2: Ignore column (code deploy)
-- Step 3: Drop
ALTER TABLE users DROP COLUMN deprecated_column;

-- down
ALTER TABLE users ADD COLUMN deprecated_column TYPE;
-- Cannot recover data. Document this.
```

## Transaction Safety

- PostgreSQL: migrations run in transaction by default. Good for atomicity.
- `CREATE INDEX CONCURRENTLY` cannot run in transaction. Use `StatementBegin` / `StatementEnd`.
- ClickHouse: no transactions. Migrations are applied immediately.

## ClickHouse Specifics

### Creating Tables

```sql
CREATE TABLE events (
    event_id UUID,
    user_id UInt64,
    event_time DateTime64(3),
    event_type LowCardinality(String),
    payload String
) ENGINE = MergeTree()
PARTITION BY toYYYYMM(event_time)
ORDER BY (event_type, event_time, user_id)
TTL event_time + INTERVAL 90 DAY;
```

### Migrations

```sql
-- ON CLUSTER for replicated setups
CREATE TABLE events ON CLUSTER '{cluster}' (
    ...
) ENGINE = ReplicatedMergeTree(...)
```

Rules:
- No `UPDATE` or `DELETE` on large tables. Use `ReplacingMergeTree` with versions.
- `ALTER TABLE ... ADD COLUMN` is lightweight.
- `ALTER TABLE ... DROP COLUMN` on distributed tables requires care.
- Materialized views are rebuilt on schema changes.

### Materialized Views

```sql
CREATE MATERIALIZED VIEW events_daily_mv
ENGINE = SummingMergeTree()
PARTITION BY toYYYYMM(day)
ORDER BY (day, event_type)
AS SELECT
    toDate(event_time) AS day,
    event_type,
    count() AS event_count
FROM events
GROUP BY day, event_type;
```

## Rollback Strategy

1. **Down migrations:** For development. Not for production data recovery.
2. **Forward fixes:** Preferred in production. Write a new `up` migration to fix.
3. **Backups:** Always backup before major migrations.
4. **Canary:** Apply to single shard/replica first.

## Validation

Before applying:
```bash
# Dry run (check syntax)
psql -f migration.sql --dry-run  # or check in transaction + rollback

# Explain plan
EXPLAIN ANALYZE SELECT ...;

# Check locks
SELECT * FROM pg_locks WHERE relation = 'users'::regclass;
```

## Common Mistakes

| Mistake | Problem | Solution |
|---------|---------|----------|
| Adding NOT NULL without default | Migration fails on existing rows | Add nullable, backfill, then add constraint |
| Missing down migration | Can't rollback in dev | Always write down |
| `CREATE INDEX` without `CONCURRENTLY` | Table lock, downtime | Use `CONCURRENTLY` |
| Large UPDATE in single transaction | Lock contention, replication lag | Batch updates |
| No backfill plan | New column empty for old rows | Plan backfill before adding constraint |
| ClickHouse UPDATE/DELETE | Performance disaster | Use inserts, ReplacingMergeTree |

## References

- [golang-migrate](https://github.com/golang-migrate/migrate)
- [PostgreSQL Migrations](https://www.postgresql.org/docs/current/sql-altertable.html)
- [ClickHouse ALTER](https://clickhouse.com/docs/en/sql-reference/statements/alter)
