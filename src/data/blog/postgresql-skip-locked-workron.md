---
author: Lynn Wang
pubDatetime: 2026-04-12T10:00:00Z
title: "From SQLite to PostgreSQL: Unlocking Concurrent Job Claiming with SKIP LOCKED"
slug: postgresql-skip-locked-workron
featured: true
draft: false
tags:
  - distributed-systems
  - go
  - postgresql
  - interface-design
  - engineering-tradeoffs
description: "Why I went back on my own decision to stop at SQLite, what FOR UPDATE SKIP LOCKED actually does, and how context.Context propagation turned a one-file change into a twenty-file refactor."
---

In the [previous post](/posts/dag-dependencies-workron), I called the DAG implementation "a natural stopping point." Workron had an in-memory store, SQLite persistence, heartbeat detection, and dependency graphs. The `JobStore` interface was clean. Everything worked.

Then I started thinking about the next step: running multiple scheduler instances against the same database. SQLite cannot do that. Its single-writer model is a hard wall. If two scheduler processes try to claim jobs concurrently, one of them gets `SQLITE_BUSY`. The architecture I was heading toward required a database that treats concurrent connections as normal, not as an error.

So I went back on my own decision. This post covers the PostgreSQL migration, the `FOR UPDATE SKIP LOCKED` pattern that makes concurrent claiming actually work, and an unexpected detour into Go's context propagation conventions.

## Table of Contents

## Why PostgreSQL, Why Now

The honest answer is that SQLite proved the abstraction works. Adding a second `JobStore` implementation without touching business logic validated the interface design. But SQLite's ont writer at a time concurrency model, serialized through `SetMaxOpenConns(1)` is fundamentally a single-process bottleneck.

PostgreSQL gives three things SQLite cannot:

1. **Connection pooling.** Multiple goroutines can hold separate connections and execute queries concurrently. No serialization at the Go level.
2. **Row-level locking.** Two transactions can claim two different jobs simultaneously without blocking each other.
3. **Multi-process access.** Two scheduler instances can connect to the same database. This is the foundation for multi-scheduler coordination.

The `JobStore` interface meant the migration was scoped to one new file. The business logic server, workers, reaper did not change. That was the whole point of the interface design from the [SQLite iteration](/posts/persisting-jobs-with-sqlite-workron).

## The Driver Choice: pgx, Not database/sql

SQLite uses Go's `database/sql` package with a driver. PostgreSQL has two major options: `lib/pq` (the older driver that also uses `database/sql`) and `pgx` (a newer, native PostgreSQL driver with its own connection pool).

I chose `pgx` with `pgxpool` for a practical reason: it handles `*time.Time` natively. The SQLite store is littered with `sql.NullTime` conversions, scan into `NullTime`, check `.Valid`, convert to `*time.Time`. With pgx, nullable timestamps scan directly into `*time.Time`. Nil means null. The code is cleaner.

The trade-off is that the two stores do not share scan helpers. SQLite's `scanJob` uses `*sql.Row`, pgx uses `pgx.Row`, different types, different APIs. I accepted the duplication because the scan logic is simple (ten fields) and the alternative which is an abstraction layer over two different row types would add complexity to avoid duplicating fifteen lines.

## FOR UPDATE SKIP LOCKED: The Key Pattern

This is the most important thing in the entire migration. The SQLite store claims a job with:

```sql
UPDATE jobs SET status = 'running'
WHERE id = (SELECT id FROM jobs WHERE status = 'pending' LIMIT 1)
RETURNING ...
```

This works because SQLite serializes all writes. Two concurrent callers cannot execute this simultaneously, the second one waits for the first to finish.

PostgreSQL does not serialize. Two transactions can run this query at the same time, and they will both find the same pending job in the subquery. The first one to commit wins. The second one updates zero rows because the job is no longer pending. At best, the second caller wastes work. At worst, with less careful SQL, you get a deadlock.

The fix is `FOR UPDATE SKIP LOCKED`:

```sql
WITH claimed AS (
    SELECT id FROM jobs
    WHERE status = 'pending'
    ORDER BY created_at
    LIMIT 1
    FOR UPDATE SKIP LOCKED
)
UPDATE jobs SET status = 'running', started_at = $1,
       attempts = attempts + 1, last_heartbeat = NULL
FROM claimed
WHERE jobs.id = claimed.id
RETURNING jobs.id, jobs.command, jobs.status, ...
```

`FOR UPDATE` locks the selected row. `SKIP LOCKED` tells the second transaction "if the row I would have selected is already locked by someone else, skip it and find the next one." Two concurrent claims get two different jobs, instantly, with no blocking and no retries.

This is the production-standard pattern. Graphile Worker, Queue Classic, and most PostgreSQL-backed job queues use it. The alternative approaches are instructive to understand:

- **`FOR UPDATE`** (without SKIP LOCKED) — blocks. The second transaction waits until the first commits or rolls back. Safe but slow under contention.
- **`FOR UPDATE NOWAIT`** — fails immediately if the row is locked. Fast but the caller has to retry.
- **`FOR UPDATE SKIP LOCKED`** — skips to the next available row. No waiting, no retrying, no wasted work. The right choice for job claiming.

## JSONB for Dependencies

SQLite stores `depends_on` as a `TEXT` column with JSON inside it, queried using `json_each()`. PostgreSQL has a native `JSONB` type with richer operators.

The schema is almost identical:

```sql
-- SQLite
depends_on TEXT DEFAULT '[]'

-- PostgreSQL
depends_on JSONB DEFAULT '[]'::jsonb
```

The `UnblockReady` query changes one function name:

```sql
-- SQLite: json_each
SELECT 1 FROM json_each(jobs.depends_on) AS dep
WHERE dep.value NOT IN (SELECT id FROM jobs WHERE status = 'done')

-- PostgreSQL: jsonb_array_elements_text
SELECT 1 FROM jsonb_array_elements_text(jobs.depends_on) AS dep
WHERE dep NOT IN (SELECT id FROM jobs WHERE status = 'done')
```

A normalized `job_dependencies` join table would be the textbook relational approach. But the JSON column keeps the PostgreSQL and SQLite implementations conceptually parallel, the same storage model, the same query pattern, just different function names. When the next backend is added (if one ever is), the pattern is clear.

## The Context Detour

Here is where a one-file migration turned into a twenty-file refactor.

pgx methods require `context.Context` as the first parameter. Every `pool.Exec()`, `pool.QueryRow()`, and `pool.Query()` call needs a context. The initial approach was to store a context inside the `PostgresStore` struct:

```go
type PostgresStore struct {
    pool   *pgxpool.Pool
    ctx    context.Context    // stored on the struct
    cancel context.CancelFunc
}
```

This works. It also violates a Go convention documented in the standard library:

> Do not store Contexts inside a struct type; instead, pass a Context explicitly to each function that needs it.

The reason is about lifecycle and cancellation. A context stored on a struct has one cancellation boundary, the struct's lifetime. A context passed as a parameter carries the caller's cancellation signal. An HTTP handler's context is cancelled when the client disconnects. A reaper's context is cancelled when the scheduler shuts down. These are different lifetimes, and a stored context collapses them into one.

So I changed the `JobStore` interface:

```go
// Before
type JobStore interface {
    AddJob(command string, dependsOn []string) string
    ClaimJob() (*Job, bool)
    GetJob(id string) (*Job, bool)
    // ...
}

// After
type JobStore interface {
    AddJob(ctx context.Context, command string, dependsOn []string) string
    ClaimJob(ctx context.Context) (*Job, bool)
    GetJob(ctx context.Context, id string) (*Job, bool)
    // ...
}
```

This is where the blast radius hit. Every method on every implementation (Memory, SQLite, PostgreSQL) gained a `ctx` parameter. Every caller the HTTP server, the reaper, the worker, the SchedulerClient, the DAG validator, and every test function needed to pass one.

The HTTP handlers already had context available through `r.Context()`. The reaper already had context from its `StartReaper(ctx, ...)` parameter. The workers had context from `Start(ctx)`. Tests used `context.Background()`. No caller needed to *create* a new context, they all already had one. The refactor was mechanical, not architectural.

SQLite also benefited. `db.Exec()` became `db.ExecContext(ctx, ...)`, meaning SQLite queries now respect context cancellation and deadlines. The MemoryStore accepts context on every method but does not use it, pure in-memory operations do not block on I/O, so there is nothing to cancel. The parameter is there for interface compliance.

Was this worth doing in the same PR as the PostgreSQL store? Arguably no, it could have been a separate change. But the stored-context approach felt wrong enough that shipping it as "temporary tech debt" would have meant explaining a Go antipattern in a portfolio project. Better to do it right once.

## The Compliance Suite Pays Off Again

The factory-based test pattern from the [SQLite post](/posts/persisting-jobs-with-sqlite-workron) proved its value for the third time. Adding PostgreSQL compliance tests meant writing one factory function and twenty-five one-line wrappers:

```go
func newTestPostgresStore(t *testing.T) JobStore {
    url := os.Getenv("WORKRON_PG_URL")
    if url == "" {
        t.Skip("WORKRON_PG_URL not set")
    }
    s, err := NewPostgresStore(ctx, url, 5)
    // ... cleanup ...
    return s
}

func TestPostgres_ClaimJobReturnsJob(t *testing.T) {
    testClaimJobReturnsJob(t, newTestPostgresStore)
}
// ... 24 more wrappers ...
```

Same tests, third backend, same guarantees. The `//go:build postgres` tag keeps these tests out of `go test ./...`, they only run with `-tags postgres` when a PostgreSQL instance is available.

Two PostgreSQL-specific tests were added beyond the compliance suite. One verifies `SKIP LOCKED` under real concurrency: 30 goroutines claiming 20 jobs through a 10-connection pool, verifying exactly 20 unique claims with zero duplicates. The other verifies persistence across a pool close and reconnect.

## What We Have Now

Workron now supports three storage backends, selectable at startup:

```
--db-driver=memory   (default, for development)
--db-driver=sqlite   (single-node persistence)
--db-driver=postgres (concurrent multi-connection access)
```

All three pass the same compliance test suite. The server, workers, and reaper are unaware of which backend they are using. Adding PostgreSQL required zero changes to business logic, only a new store implementation and context threading.

The system is now ready for multi-scheduler coordination. Two scheduler instances can connect to the same PostgreSQL database, and `SKIP LOCKED` guarantees they will never double-claim a job. That coordination, advisory locks for the reaper, health endpoints, and verifying zero conflicts under real multi-process load is the next problem to solve.

Full source: [github.com/lrdinsu/workron](https://github.com/lrdinsu/workron)

---

## References and Further Reading

- [PostgreSQL: Explicit Locking — Row-Level Locks](https://www.postgresql.org/docs/current/explicit-locking.html#LOCKING-ROWS). — Official docs covering `FOR UPDATE`, `FOR UPDATE NOWAIT`, and `FOR UPDATE SKIP LOCKED`. This was the primary reference for understanding the trade-offs between blocking, failing, and skipping when claiming jobs.
- [pgx: PostgreSQL Driver and Toolkit for Go](https://github.com/jackc/pgx). — The native PostgreSQL driver Workron uses instead of `database/sql` with `lib/pq`. The README covers the type-handling and performance differences that motivated the choice.
- [Contexts and structs (Go blog)](https://go.dev/blog/context-and-structs). — Official Go blog post on why context should be a function parameter, not a struct field. This directly motivated the interface refactor that turned a one-file change into a twenty-file one.
- [Graphile Worker](https://github.com/graphile/worker). — A PostgreSQL-backed job queue for Node.js that uses `SKIP LOCKED` for job claiming. Useful reference for seeing the pattern applied in a production system.
- [Queue Classic](https://github.com/QueueClassic/queue_classic). — A Ruby PostgreSQL job queue that helped popularize the `SKIP LOCKED` approach for background job processing.
