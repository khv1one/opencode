# OpenCode Personal Configuration

> Staff Go Engineer profile with multi-agent orchestration for enterprise-grade Go development.

## Overview

This repository contains my custom [OpenCode](https://opencode.ai/) configuration — an AI-powered development environment optimized for Go 1.26+ enterprise projects. The setup uses a multi-agent architecture where specialized agents handle distinct phases of the development lifecycle.

## Architecture

```
┌─────────────┐     ┌──────────┐     ┌──────────┐     ┌──────────┐
│   manager   │────▶│ explore  │────▶│architect │────▶│  goplan  │
│(orchestrator│     │(context) │     │  (design)│     │ (plan)   │
└─────────────┘     └──────────┘     └──────────┘     └────┬─────┘
                                                           │
                              ┌────────────────────────────┘
                              ▼
                    ┌─────────────────────┐
                    │       godev         │
                    │   (implementation)  │
                    └─────────┬───────────┘
                              │
                    ┌─────────┴───────────┐
                    ▼                     ▼
            ┌──────────┐          ┌──────────┐
            │  tester  │          │  dbdev   │
            │(validate)│          │ (schema) │
            └────┬─────┘          └──────────┘
                 │
                 ▼
            ┌──────────┐
            │ reviewer │──── loop (max 3x)
            │ (audit)  │
            └────┬─────┘
                 │
                 ▼
            ┌──────────┐
            │   docs   │
            │(finalize)│
            └──────────┘
```

## Agents

| Agent | Role |
|-------|------|
| `manager` | Orchestrates workflow, manages MCP, delegates tasks |
| `explore` | Fast context gathering via MCP context7, glob, grep |
| `architect` | System design, ADRs, bounded contexts |
| `goplan` | Plans Go implementation with long-context retention |
| `godev` | Writes idiomatic Go code (Go 1.26+ generation) |
| `tester` | Validates tests, writes fuzzing/benchmarks |
| `reviewer` | Audits code before merge (races, leaks, arch) |
| `dbdev` | DB schema, queries, migrations (PG/ClickHouse) |
| `docs` | API contracts, README, godoc, ADR prose |

## Skills

Domain-specific skill modules loaded on demand:

| Skill | Domain |
|-------|--------|
| `api-design` | REST API design patterns |
| `clickhouse-io` | ClickHouse analytics & queries |
| `go-concurrency-patterns` | errgroup, pipelines, worker pools |
| `go-foundation-core` | Go core patterns |
| `go-foundation-db` | Database patterns |
| `go-foundation-transport` | Transport layer patterns |
| `go-foundation-worker` | Background worker patterns |
| `go-performance-optimization` | Profiling, sync.Pool, escape analysis |
| `monorepo-layered-architecture` | Transport→Service→Repo isolation |
| `observability-otel` | OpenTelemetry instrumentation |
| `sql-migrations` | golang-migrate, zero-downtime |

## MCP Servers

| Server | Type | Purpose |
|--------|------|---------|
| `context7` | Remote | Semantic code documentation search |
| `serena` | Local | LSP-powered code navigation & editing |
| `memory` | Local | Persistent project memory |
| `sequential-thinking` | Local | Dynamic problem-solving |

## Key Principles

- **Layered Architecture:** No layer bypass. Transport → Service → Repository.
- **Explicit Errors:** `fmt.Errorf("op: %w", err)` everywhere.
- **Concurrency:** Context cancellation, `errgroup` for fan-out.
- **Testing:** Table-driven tests, race detector, fuzzing, benchmarks.
- **Backward Compatibility:** All changes are backward-compatible.
- **OTEL:** Instrumentation on critical paths.

## Commands

| Command | Agent | Description |
|---------|-------|-------------|
| `/review` | `reviewer` | Code review for races, leaks, arch violations |
| `/plan` | `goplan` | Plan Go implementation |
| `/test` | `tester` | Write tests and fuzzing |
| `/arch` | `architect` | Architectural design |
| `/explore` | `explore` | Explore codebase context |

## Files

| File | Purpose |
|------|---------|
| `opencode.json` | Main configuration — agents, MCP servers, permissions |
| `AGENTS.md` | Agent orchestration rules & pipeline definition |
| `tui.json` | Terminal UI settings |
| `agents/*.md` | Per-agent system prompts |
| `skills/*/SKILL.md` | Domain skill definitions |

## Usage

This configuration lives at `~/.config/opencode/`. OpenCode loads it automatically on startup.

---

*Configured for Go 1.26+, enterprise monorepo, polyglot persistence (PostgreSQL/ClickHouse/Redis/Kafka), K8s/Helm, GitLab CI, OTEL.*
