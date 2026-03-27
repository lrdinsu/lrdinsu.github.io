---
author: Lynn Wang
pubDatetime: 2026-03-27T10:00:00Z
title: "Surviving the Crash: Adding SQLite Persistence Without Touching Business Logic"
slug: persisting-jobs-with-sqlite-workron
featured: true
draft: false
tags:
  - distributed-systems
  - go
  - sqlite
  - interface-design
  - engineering-tradeoffs
description: "Swapping Workron's in-memory store for SQLite, and the interface design that made it a one-file change outside the store package."
---

In the [previous post](/posts/splitting-and-surviving-failures-workron), Workron gained heartbeat detection and a background reaper that re-queues orphaned jobs. Workers can crash and the system recovers. But there is still a single point of total failure: the scheduler itself. Kill the scheduler process and every pending, running or done job will vanish. All states lived in a Go map protected by a mutex.

So I try to fix that. Workron gets a SQLite-backed store that survives a complete scheduler restart. The interesting part is not SQLite itself, it is how little changed outside the store package, and what I learned about SQLite's concurrency model along the way.

## Table of Contents

## Why SQLite, Not PostgreSQL

The goal of this iteration was to prove that the storage abstraction works, that I could add a second `JobStore` implementation without touching any business logic. SQLite tests the same concepts as PostgreSQL: atomic SQL claims, schema migrations, transaction handling. But it requires zero infrastructure. No server process, no Docker container, no connection string. It is a single file on disk.

There was also a dependency trade-off. Before, Workron had zero external dependencies but stdlib only. SQLite was the first, and I chose [modernc.org/sqlite](https://pkg.go.dev/modernc.org/sqlite) specifically because it is a pure Go port of SQLite. The alternative using CGo with the C SQLite library would have meant requiring a C compiler to build, breaking easy cross-compilation, and losing the "just `go build`" story. With the pure Go driver, the build experience stays the same on any platform.

PostgreSQL would be the natural next step for a production scheduler with multiple replicas. But the `JobStore` interface means that migration is just writing a new `PostgresStore` and no business logic changes. I chose to start at SQLite because the architectural insight is the same, and the marginal resume value did not justify the infrastructure complexity.

## The Schema

The table maps directly to the existing `Job` struct:

```sql
CREATE TABLE IF NOT EXISTS jobs (
    id             TEXT PRIMARY KEY,
    command        TEXT NOT NULL,
    status         TEXT NOT NULL DEFAULT 'pending',
    created_at     DATETIME NOT NULL,
    started_at     DATETIME,
    done_at        DATETIME,
    last_heartbeat DATETIME,
    max_retries    INTEGER DEFAULT 3,
    attempts       INTEGER DEFAULT 0
);
```

Every field that existed on the in-memory `Job` struct gets a column. Nullable fields (`started_at`, `done_at`, `last_heartbeat`) are nullable columns, scanned into Go's `sql.NullTime` and converted to `*time.Time` on read.

## Atomic Claiming in SQL

The most interesting method is `ClaimJob`. In the memory store, atomicity comes from a mutex: lock, find a pending job, update its status, unlock. In SQLite, the same guarantee comes from a single SQL statement:

```go
func (s *SQLiteStore) ClaimJob() (*Job, bool) {
    row := s.db.QueryRow(`
        UPDATE jobs
        SET status = 'running', started_at = ?, attempts = attempts + 1, last_heartbeat = NULL
        WHERE id = (SELECT id FROM jobs WHERE status = 'pending' LIMIT 1)
        RETURNING id, command, status, created_at, started_at, done_at,
                  last_heartbeat, max_retries, attempts`,
        time.Now(),
    )
    return scanJob(row)
}
```

The subquery finds one pending job. The `UPDATE` atomically claims it. `RETURNING` gives back the updated row in a single round trip. If no pending jobs exist, `scanJob` sees `sql.ErrNoRows` and returns `(nil, false)`, same contract as the memory store.

This is the SQL equivalent of the mutex-protected claim loop. Two workers hitting this at the same instant will never get the same job, because SQLite executes the `UPDATE` atomically.

## SQLite's Concurrency Surprises

This is where things got educational.

### The Connection Pool Problem

Go's `database/sql` package maintains a connection pool. When multiple goroutines make queries, each one can get a different connection from the pool. That is great for PostgreSQL, which handles concurrent connections natively. For SQLite, it is a problem.

SQLite supports only one writer at a time. With a pool of five connections, five goroutines can each grab a connection and try to write simultaneously. Four of them get `SQLITE_BUSY` and the database is locked. My first concurrent test hit this immediately: ten goroutines racing to claim five jobs, and the process panicked.

The fix is the standard recommendation for SQLite in Go:

```go
db.SetMaxOpenConns(1)
```

One connection means Go serializes all database access internally. Goroutines take turns at the Go level instead of fighting over SQLite's write lock. The serialization happens either way, the question is just where. Doing it in Go's pool is clean and error-free. Doing it at the SQLite level means `SQLITE_BUSY` errors you have to handle.

The trade-off is that reads are also serialized. For Workron this does not matter, the scheduler processes jobs that take seconds to run, so microsecond-level database serialization is never the bottleneck. For a system where it would matter, you would need a client-server database like PostgreSQL.

### WAL Mode

SQLite defaults to a rollback journal: before every write, it copies the original page to a journal file so it can undo the change if something goes wrong. WAL (Write-Ahead Logging) flips this, writes go to an append-only log, and readers continue reading from the main database file.

With a single connection, WAL's concurrency benefit (readers not blocking writers) is unused. But WAL still provides faster writes because appending to a log is cheaper than copying pages. It is the better journal mode regardless of pool size:

```go
db.Exec("PRAGMA journal_mode=WAL")
```

### Busy Timeout

As a safety net, I also set a busy timeout:

```go
db.Exec("PRAGMA busy_timeout=5000")
```

With a single connection this should never trigger under normal operation. But if SQLite encounters internal lock contention during crash recovery, it will retry for up to five seconds instead of failing immediately.

## Testing: The Compliance Pattern

Here is the problem: `MemoryStore` and `SQLiteStore` both implement `JobStore`. They should behave identically. Writing the same test twice with copy-paste is fragile, when you add a new test case, you have to remember to add it in both files.

The solution is a factory pattern. Each test is written once as a shared function that accepts a `StoreFactory`, a function that creates a fresh store:

```go
type StoreFactory func(t *testing.T) JobStore

func testClaimJobReturnsJob(t *testing.T, factory StoreFactory) {
    t.Helper()
    s := factory(t)

    id := s.AddJob("echo hello")
    job, ok := s.ClaimJob()

    if !ok {
        t.Fatal("expected to claim a job")
    }
    if job.ID != id {
        t.Errorf("ID = %q, want %q", job.ID, id)
    }
    if job.Status != StatusRunning {
        t.Errorf("Status = %q, want %q", job.Status, StatusRunning)
    }
    // ... more assertions
}
```

Each implementation provides its own factory:

```go
// sqlite_test.go
func newTestSQLiteStore(t *testing.T) JobStore {
    t.Helper()
    s, err := NewSQLiteStore(filepath.Join(t.TempDir(), "test.db"))
    if err != nil {
        t.Fatal(err)
    }
    t.Cleanup(func() { _ = s.Close() })
    return s
}

func TestSQLite_ClaimJobReturnsJob(t *testing.T) {
    testClaimJobReturnsJob(t, newTestSQLiteStore)
}

// memory_test.go
func newTestMemoryStore(t *testing.T) JobStore {
    t.Helper()
    return NewMemoryStore()
}

func TestMemory_ClaimJobReturnsJob(t *testing.T) {
    testClaimJobReturnsJob(t, newTestMemoryStore)
}
```

One test function, two implementations, same guarantees. If a future `PostgresStore` is added, it gets the same compliance suite for free, just provide a factory.

The `t.Helper()` call at the top of each shared function tells Go "I am a helper, not the real test, if I fail, show the caller's line number." Without it, failures point at a line inside `testClaimJobReturnsJob`, which is less useful than knowing which specific `TestSQLite_*` or `TestMemory_*` wrapper triggered it.

SQLite also gets one test that does not apply to the memory store:

```go
func TestSQLite_PersistenceAcrossReopen(t *testing.T) {
    dbPath := filepath.Join(t.TempDir(), "test.db")

    s1, err := NewSQLiteStore(dbPath)
    if err != nil {
        t.Fatal(err)
    }
    id := s1.AddJob("echo persist")
    _ = s1.Close()

    s2, err := NewSQLiteStore(dbPath)
    if err != nil {
        t.Fatal(err)
    }
    defer func() { _ = s2.Close() }()

    job, found := s2.GetJob(id)
    if !found {
        t.Fatal("job not found after reopening database")
    }
    if job.Command != "echo persist" {
        t.Errorf("Command = %q, want %q", job.Command, "echo persist")
    }
}
```

Open, add a job, close, reopen, verify. That is the entire point of this iteration in one test.

## The Surprisingly Small Diff

Here is what changed in `cmd/scheduler/main.go` to support both storage backends:

```go
var s store.JobStore
if *dbPath != "" {
    sqliteStore, err := store.NewSQLiteStore(*dbPath)
    if err != nil {
        log.Fatalf("[main] failed to open database: %v", err)
    }
    defer func() { _ = sqliteStore.Close() }()
    s = sqliteStore
    log.Printf("[main] using SQLite store: %s", *dbPath)
} else {
    s = store.NewMemoryStore()
    log.Println("[main] using in-memory store")
}
```

And that was really all it took: one `if` block in `main.go`. The server, the workers, and the reaper did not need to change, because they all depend on the `JobStore` interface rather than a specific storage implementation.

This was a decision I made early in the project, before I knew exactly what the second backend would look like. The initial `JobStore` interface only included `AddJob`, `ClaimJob`, `GetJob`, and `UpdateJobStatus`. As the scheduler grew, I added methods such as `ListJobs`, `UpdateHeartbeat`, and `ListRunningJobs`. But the basic idea stayed the same: define a clear contract, implement it twice, and choose the implementation at the top level.

What I like most is that this part worked the way I had hoped. There was no need for adapter layers, no temporary shims, and no moment where I had to step back and restructure everything. It was a small payoff from making the abstraction clean early on. Interface design can feel unexciting at first, but moments like this are when it proves its value.
## What We Have Now

Workron can now:

- Accept jobs via REST API
- Distribute them across multiple workers (local or remote)
- Detect dead workers via heartbeats and re-queue their jobs
- Survive a complete scheduler crash and restart with all job state intact

```
Submit → Persist → Claim → Execute → Heartbeat → Done
                                         ↓
                              Crash? → Reaper → Re-queue
         ↓
   Kill scheduler → Restart → All jobs still there
```

What it cannot do yet: model relationships between jobs. If step 2 depends on step 1 finishing first, there is no way to express that. The [next post](/posts/dag-dependencies-workron) adds this piece: DAG-based job dependencies. Jobs declare what they depend on, the scheduler validates the graph at submission time, and downstream jobs only become claimable once all their upstream dependencies complete.

Full source: [Workron](https://github.com/lrdinsu/workron)

---

## References and Further Reading

- [modernc.org/sqlite](https://pkg.go.dev/modernc.org/sqlite). — A pure Go port of SQLite, requiring no CGo or C compiler. Chosen over the CGo-based `mattn/go-sqlite3` to preserve cross-compilation and the "just `go build`" experience.
- [SQLite: Write-Ahead Logging](https://www.sqlite.org/wal.html). — Official SQLite docs on WAL mode. Helpful for understanding why WAL improves write performance even with a single-connection pool.
- [Go `database/sql` package](https://pkg.go.dev/database/sql). — Official docs for Go's database abstraction layer. The `SetMaxOpenConns(1)` pattern for SQLite came from understanding how the connection pool interacts with SQLite's single-writer model.
- [How Litestream works](https://litestream.io/how-it-works/). — Overview of SQLite replication via WAL streaming. A useful reference for understanding WAL internals, even though Workron does not use Litestream.
- [Effective Go: Interfaces](https://go.dev/doc/effective_go#interfaces). — The Go convention of defining interfaces at the consumer, not the provider. Workron's `JobStore` interface follows this pattern, defined where it is used rather than alongside the implementations.
