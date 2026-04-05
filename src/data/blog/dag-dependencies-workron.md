---
author: Lynn Wang
pubDatetime: 2026-04-04T10:00:00Z
title: "DAG Dependencies: Teaching a Job Scheduler to Wait"
slug: dag-dependencies-workron
featured: true
draft: false
tags:
  - distributed-systems
  - go
  - dag
  - interface-design
  - engineering-tradeoffs
description: "Adding job dependencies to Workron — a new status, a cycle detection algorithm that guards against a problem that can't happen yet, and a SQL trick for querying into JSON arrays."
---

In the [previous post](/posts/persisting-jobs-with-sqlite-workron), Workron gained SQLite persistence. The scheduler can crash and restart without losing a single job. Workers can die and the reaper re-queues their work. The system handles failure well.

What it cannot handle is *order*. If step 2 depends on step 1 finishing first, there is no way to express that. You would have to watch the API, wait for step 1 to reach `done`, and only then submit step 2. That is fragile and manual — exactly the kind of thing a scheduler should automate.

This post adds DAG-based job dependencies. Jobs can declare what they depend on. The scheduler validates the dependency graph at submission time, rejects cycles, and only makes downstream jobs available for execution once all their upstream dependencies have completed.

## Table of Contents

## The Design: A New Status

The simplest approach would be to keep all jobs as `pending` and add a filter to `ClaimJob`: before handing out a job, check if its dependencies are done. Skip it if they are not.

That works, but it has a visibility problem. A job stuck waiting for a dependency looks the same as a job waiting for a free worker — both are `pending`. When you call `GET /jobs` and see five pending jobs, you cannot tell which ones are actually claimable and which are waiting. Debugging becomes guesswork.

Instead, I introduced a `blocked` status. The lifecycle now looks like this:

```
Without dependencies:
  submit → pending → running → done/failed

With dependencies:
  submit → blocked → pending → running → done/failed
```

A job with `depends_on` starts as `blocked`. It transitions to `pending` only when every job in its dependency list has reached `done`. From there, the normal lifecycle takes over — workers claim it, execute it, report results.

This makes the system state immediately legible. `blocked` means "waiting for upstream." `pending` means "ready for a worker." No ambiguity.

The trade-off is a slightly larger state machine and a new method (`UnblockReady`) that the scheduler must call at the right times. But `ClaimJob` itself does not change at all — it still only looks for `pending` jobs, same as before. The complexity stays in one place.

## Storing Dependencies

The `Job` struct gains one field:

```go
type Job struct {
    // ... existing fields
    DependsOn []string `json:"depends_on,omitempty"`
}
```

A slice of job IDs. Simple and direct. In the memory store, it is stored as-is on the struct. In SQLite, it is a JSON-encoded text column:

```sql
depends_on TEXT DEFAULT '[]'
```

An alternative would be a separate `job_dependencies` join table with `job_id` and `depends_on_id` columns. That is the normalized relational approach, and it would be the right call for a production system with complex queries. For Workron, a JSON column is simpler: no extra table, no joins for basic operations, and SQLite has built-in JSON functions that make querying into the array straightforward.

`AddJob` decides the initial status based on whether dependencies are provided:

```go
func (s *MemoryStore) AddJob(command string, dependsOn []string) string {
    // ...
    status := StatusPending
    if len(dependsOn) > 0 {
        status = StatusBlocked
    }
    // ...
}
```

No dependencies means `pending`, same as before. The change is fully backward compatible — existing API calls without `depends_on` behave identically.

## Cycle Detection

Here is a question that sounds important but has a surprising answer: what if someone creates a circular dependency?

Job A depends on Job B, Job B depends on Job A. Both are `blocked`, waiting for each other. Neither will ever run. A deadlock.

The surprising part is that **this cannot happen** with the current API. When you submit a job, you can only reference jobs that already exist. Job A is created first with no dependencies. Job B is created and can depend on A. But A can never depend on B, because B did not exist when A was submitted. The submission order naturally enforces a topological ordering.

So why build cycle detection at all?

Because the guarantee is structural, not enforced. If a future API change — batch submission, dependency modification, data import — breaks the "you can only point backward" invariant, cycles become possible. The detection catches them regardless of how the dependency data arrived. It is a safety net for a problem that cannot happen today but could if the system evolves.

The algorithm is a standard DFS with three-color marking. Each node is white (unvisited), gray (in the current path), or black (fully explored). If the DFS reaches a gray node, it has found a cycle:

```go
func ValidateDependencies(s JobStore, dependsOn []string) error {
    if len(dependsOn) == 0 {
        return nil
    }

    // Verify all referenced IDs exist.
    for _, depID := range dependsOn {
        if _, found := s.GetJob(depID); !found {
            return fmt.Errorf("dependency %q does not exist", depID)
        }
    }

    // Build adjacency list from all existing jobs plus the new one.
    graph := make(map[string][]string)
    for _, job := range s.ListJobs() {
        graph[job.ID] = job.DependsOn
    }
    graph["__new__"] = dependsOn

    // DFS from the new node only — existing jobs were validated
    // at their own submission time.
    // ... three-color DFS, returns error on cycle ...
}
```

Two things worth noting. First, the DFS only starts from the new node. There is no need to scan the entire graph — existing jobs were already validated when they were submitted. The only new edges are from the new job to its dependencies. Second, the error message includes the full cycle path, like `"cycle detected: job-1 → job-2 → job-3 → job-1"`, which makes debugging straightforward.

The validation runs in `handleSubmitJob` before the job is created:

```go
if len(req.DependsOn) > 0 {
    if err := store.ValidateDependencies(s.store, req.DependsOn); err != nil {
        http.Error(w, err.Error(), http.StatusBadRequest)
        return
    }
}

id := s.store.AddJob(req.Command, req.DependsOn)
```

Reject first, create second. A bad dependency graph never enters the store.

## Unblocking: The `json_each` Trick

When a job completes, the scheduler needs to check all `blocked` jobs and transition any whose dependencies are fully satisfied to `pending`. In the memory store, this is a straightforward nested loop:

```go
func (s *MemoryStore) UnblockReady() {
    s.mu.Lock()
    defer s.mu.Unlock()

    for _, job := range s.jobs {
        if job.Status != StatusBlocked {
            continue
        }

        ready := true
        for _, depID := range job.DependsOn {
            dep, exists := s.jobs[depID]
            if !exists || dep.Status != StatusDone {
                ready = false
                break
            }
        }

        if ready {
            job.Status = StatusPending
        }
    }
}
```

In SQLite, the same logic becomes a single statement using `json_each`, which expands a JSON array into rows:

```sql
UPDATE jobs SET status = 'pending'
WHERE status = 'blocked'
AND NOT EXISTS (
    SELECT 1 FROM json_each(jobs.depends_on) AS dep
    WHERE dep.value NOT IN (SELECT id FROM jobs WHERE status = 'done')
)
```

The double negative reads awkwardly, but the meaning is clear: update blocked jobs where there does NOT EXIST any dependency that is NOT IN the done jobs. If every dependency is done, the subquery finds nothing, the `NOT EXISTS` condition is true, and the job gets unblocked.

This is where storing dependencies as a JSON column pays off. With a join table, you would write a `LEFT JOIN` with a `HAVING COUNT` clause. With `json_each`, the query reads almost like the Go code — iterate the dependencies, check each one. No extra table, no extra index.

## Where Unblocking Happens

`UnblockReady` is called in two places, for two different reasons.

**In the HTTP handler, after a job completes.** When a worker reports `POST /jobs/{id}/done`, the handler marks the job as done and immediately calls `UnblockReady`. This makes unblocking responsive — downstream jobs become available within the same request cycle, and the next worker poll picks them up.

```go
func (s *Server) handleJobDone(w http.ResponseWriter, r *http.Request) {
    // ... validate job exists and is running ...
    s.store.UpdateJobStatus(id, store.StatusDone)
    s.store.UnblockReady()
    // ...
}
```

**In the reaper, on every tick.** The reaper already runs every 10 seconds checking for dead workers. Adding `UnblockReady` to the same tick costs almost nothing and catches an edge case: in standalone mode, workers call `UpdateJobStatus` directly on the store, bypassing the HTTP handler. Without the reaper calling `UnblockReady`, a job completed in standalone mode would leave its downstream jobs blocked until the next HTTP-based completion triggered a check.

```go
case <-ticker.C:
    reap(s)
    s.UnblockReady()
```

The HTTP handler gives you responsiveness. The reaper gives you correctness in all deployment modes. Belt and suspenders.

## The Compliance Test Pattern Continues

The compliance test approach from the persistence post — writing tests once and running them against both MemoryStore and SQLiteStore — proved its value again here. Every unblocking scenario (single dependency, partial dependencies, chains, diamond DAGs) is written once and runs against both backends:

```go
func TestMemory_UnblockReadyChain(t *testing.T) {
    testUnblockReadyChain(t, newTestMemoryStore)
}
func TestSQLite_UnblockReadyChain(t *testing.T) {
    testUnblockReadyChain(t, newTestSQLiteStore)
}
```

The chain test is the most illustrative. It creates a linear dependency A ← B ← C, completes A, and verifies that B unblocks but C stays blocked. Then it completes B and verifies C unblocks. Each `UnblockReady` call only looks one level deep — C does not care that A is done, only that B (its direct dependency) is done.

One subtlety that came up during testing: `ClaimJob` on a MemoryStore iterates a Go map, and map iteration order is randomized. Tests that claim a job and then mark a *specific* job as done must account for the possibility that a different job was claimed. The fix is simple — claim all pending jobs upfront, then complete them individually by ID. The order does not matter when every pending job is already running.

## A Known Limitation

If a dependency fails permanently, its downstream jobs remain `blocked` forever. There is no cascading failure. A pipeline where step 1 fails will leave steps 2 and 3 stuck in `blocked` with no automatic resolution.

This is a deliberate scope choice, not an oversight. The right behavior depends on the use case. Some workflows should cascade failure immediately — if the build step fails, do not deploy. Others should leave downstream jobs blocked so a human can inspect, fix step 1, re-run it, and let the pipeline continue. Choosing wrong is worse than not choosing.

If I were to add this, the clean approach would be a `cancel` endpoint that transitions a blocked job to `failed` with a reason like `"dependency failed: job-123"`. A cascading cancel could then propagate through the graph. But that is a feature with its own design decisions, and Workron is already demonstrating the core DAG concepts without it.

## What We Have Now

Workron can now:

- Accept jobs via REST API, with optional dependencies forming a DAG
- Validate the dependency graph at submission time, rejecting cycles and missing references
- Automatically unblock downstream jobs when upstream dependencies complete
- Distribute work across multiple workers (local or remote)
- Detect dead workers via heartbeats and re-queue their jobs
- Persist all job state to SQLite, surviving complete scheduler restarts

```
Submit ──► blocked ──► pending ──► running ──► done
            │                        │            │
            │ (deps complete)        │            └─► Unblock downstream
            │                        │
            │                   Crash? ──► Reaper ──► Re-queue
            │
       Kill scheduler ──► Restart ──► All jobs still there
```

This felt like a natural stopping point for the first arc of the Workron series. The project started as an exploration of one question: how do N goroutines share one queue without stepping on each other, and grew into a distributed system with separate binaries, failure detection, persistence, and workflow dependencies. Each iteration introduced a new problem that only exists because of the previous solution: splitting into separate processes created the need for heartbeats, heartbeats created the need for a reaper, persistence created the need for SQL atomicity, and dependencies created the need for graph validation.

That compounding complexity is what makes distributed systems interesting. Every answer raises a new question. The trick is knowing which questions are worth answering now and which are worth deferring, and sometimes the deferred ones pull you back in.

Full source: [github.com/lrdinsu/workron](https://github.com/lrdinsu/workron)

---

## References and Further Reading

- [Apache Airflow Architecture Overview](https://airflow.apache.org/docs/apache-airflow/stable/core-concepts/overview.html). — Overview of Airflow's scheduler, worker, and DAG-based orchestration model. A useful reference point for thinking about how dependency graphs drive execution order.
- [Introduction to Algorithms — DFS and Topological Sort](https://mitpress.mit.edu/9780262046305/introduction-to-algorithms/). — The three-color DFS algorithm used for cycle detection comes from CLRS (Chapter 22). The white/gray/black marking scheme maps directly to Workron's implementation.
- [SQLite JSON functions](https://www.sqlite.org/json1.html). — Official docs for `json_each()` and related functions. This was the key reference for querying into the JSON-encoded `depends_on` column without a join table.
- [Directed Acyclic Graph (Wikipedia)](https://en.wikipedia.org/wiki/Directed_acyclic_graph). — A clear overview of DAG properties and terminology. Helpful for framing the dependency model and understanding why topological ordering prevents cycles in normal submission flows.
