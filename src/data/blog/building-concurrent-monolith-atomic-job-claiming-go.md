---
author: Lynn Wang
pubDatetime: 2026-03-13T10:00:00Z
modDatetime: 2026-03-14T23:20:00Z
title: "Building the Concurrent Monolith: Atomic Job Claiming in Go"
slug: building-concurrent-monolith-atomic-job-claiming-go
featured: true
draft: false
tags:
  - distributed-systems
  - go
  - concurrency
  - engineering-tradeoffs
description: "How to build a job queue where multiple goroutines compete for work without stepping on each other, mutex-protected maps, atomic claiming, and retry logic."
---

In my [previous post](/posts/designing-distributed-job-scheduler-go), I built the architectural blueprint for Workron: the trade-offs between channels and mutexes, why I chose HTTP over gRPC, and the roadmap from a single-process queue to a distributed system. 

The first two iterations tackle what I think is the hardest concurrency problem in a job scheduler: how do N goroutines share one queue without ever processing the same job twice? Everything runs in a single Go process. No network. No database. Just goroutines, a shared map, and a mutex standing between correctness and chaos.

Full code source: [Workron](github.com/lrdinsu/workron)

---

## The Job Store: A Map Behind a Mutex

The heart of Workron is a `MemoryStore`: a Go map of jobs protected by a `sync.RWMutex`. Every operation that reads or writes job state must go through this lock.

```go
type MemoryStore struct {
    mu   sync.RWMutex
    jobs map[string]*Job
}
```

It enforces: no goroutine ever touches a job directly. Every interaction goes through methods that acquire the lock first.

Adding a job is straightforward. Lock, insert, unlock:

```go
func (s *MemoryStore) AddJob(command string) string {
    s.mu.Lock()
    defer s.mu.Unlock()

    id := fmt.Sprintf("job-%d", s.nextID)
    s.nextID++
    s.jobs[id] = &Job{
        ID:         id,
        Command:    command,
        Status:     StatusPending,
        CreatedAt:  time.Now(),
        MaxRetries: 3,
    }
    return id
}
```

Read operations use `RLock` instead, allowing multiple goroutines to read concurrently as long as nobody is writing. This distinction matters when the REST API is fielding status queries while workers are simultaneously claiming jobs.

But the operation that makes the entire system is `ClaimJob`.

---

## ClaimJob: The Critical Section

`ClaimJob` is where correctness lives or dies. Three workers call it simultaneously. Only one should win. The loser should not get the same job, should not get a half-updated job, and should not corrupt state for the winner.

The implementation is deceptively simple:

```go
func (s *MemoryStore) ClaimJob() (*Job, bool) {
    s.mu.Lock()
    defer s.mu.Unlock()

    for _, job := range s.jobs {
        if job.Status == StatusPending {
            job.Status = StatusRunning
            job.Attempts++
            now := time.Now()
            job.StartedAt = &now
            return s.copyJob(job), true
        }
    }
    return nil, false
}
```

Three details here are worth unpacking.

**The full lock.** This is `mu.Lock()`, not `mu.RLock()`. Even though we are "finding" a job, we are also mutating it. If two goroutines both found the same pending job under a read lock, they would both think they claimed it. The write lock guarantees that find-and-update is a single atomic operation.

**Returning a copy.** `copyJob` creates a deep copy of the job before returning it. Without this, the caller receives a pointer into the map. If the worker modifies the job (or if another goroutine claims a different job and triggers a map resize), you get a data race. The copy severs the link between the worker's view and the shared state.

**Attempt tracking.** `job.Attempts++` happens inside the lock, during the claim itself, not after execution. This means the attempt count is always accurate regardless of what happens next. If the worker crashes after claiming but before finishing, the attempt was still recorded.

This pattern, lock → find → mutate → copy → unlock, maps directly to how a database handles the equivalent operation: `UPDATE jobs SET status = 'running' WHERE status = 'pending' LIMIT 1`. The mutex is doing the work that a database row lock would do later.

---

## The Worker: A Polling Loop

Each worker is a goroutine running a simple loop: try to claim a job, execute it if found, wait if not.

```go
func (w *Worker) Start(ctx context.Context) {
    ticker := time.NewTicker(pollInterval)
    defer ticker.Stop()

    for {
        select {
        case <-ctx.Done():
            return
        default:
        }

        job, found := w.source.ClaimJob()
        if found {
            w.process(job)
            continue
        }

        select {
        case <-ctx.Done():
            return
        case <-ticker.C:
        }
    }
}
```

The structure is deliberately three-step: check for shutdown, try to claim, and if there is nothing to do, wait for either the next tick or a shutdown signal. The two `select` statements serve different purposes. The first is a non-blocking check so we do not start new work after a shutdown signal. The second is a blocking wait that keeps the goroutine idle instead of spinning the CPU when the queue is empty.

An earlier version used `time.Sleep` instead of a ticker. The problem: `Sleep` cannot be interrupted. If you cancel the context during a one-second sleep, the worker does not notice until the sleep finishes. The ticker-based `select` responds to cancellation immediately.

---

## Command Execution

The executor is intentionally the simplest component. It takes a command string, runs it through the system shell, and returns an error if anything goes wrong:

```go
func (e *Executor) Execute(command string) error {
    if command == "" {
        return fmt.Errorf("empty command")
    }
    cmd := exec.Command("sh", "-c", command)
    output, err := cmd.CombinedOutput()
    if err != nil {
        return fmt.Errorf("command failed: %w: %s", err, output)
    }
    return nil
}
```

Using `sh -c` means the executor supports pipes, redirects, and shell built-ins. `CombinedOutput` captures both stdout and stderr, which gets attached to the error message on failure. This is critical for debugging: when a job fails, you want to know *why*, not just that it failed.

---

## Retry Logic: Giving Jobs a Second Chance

Not every failure is permanent. A network blip, a temporary disk full condition, a transient dependency being unavailable, these all resolve themselves. Workron gives every job three attempts by default.

The retry decision happens in the worker's `process` method:

```go
func (w *Worker) process(job *store.Job) {
    err := w.executor.Execute(job.Command)
    if err != nil {
        if job.Attempts < job.MaxRetries {
            w.source.UpdateJobStatus(job.ID, store.StatusPending)
            return
        }
        w.source.UpdateJobStatus(job.ID, store.StatusFailed)
        return
    }
    w.source.UpdateJobStatus(job.ID, store.StatusDone)
}
```

When a job fails and has attempts remaining, it goes back to `pending`. The next time any worker calls `ClaimJob`, this job is eligible again. The `Attempts` field, which was incremented during claiming, ensures the job eventually exhausts its retries and lands in `failed` permanently.

One subtlety: the worker decides whether to retry, not the store. The store just records state transitions. This separation matters when workers communicate over HTTP, the retry decision moves to the scheduler. The worker simply reports "this failed" and the scheduler decides what to do next. Keeping the store dumb makes that migration easier.

---

## The REST API

The HTTP layer is thin by design. Each handler validates input, calls a store method, and returns JSON:

```go
func (s *Server) handleSubmitJob(w http.ResponseWriter, r *http.Request) {
    var req submitJobRequest
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        http.Error(w, "invalid request body", http.StatusBadRequest)
        return
    }
    if req.Command == "" {
        http.Error(w, "command is required", http.StatusBadRequest)
        return
    }
    id := s.store.AddJob(req.Command)
    job, _ := s.store.GetJob(id)
    writeJSON(w, http.StatusCreated, job)
}
```

Go 1.22's enhanced `ServeMux` handles method-based routing natively, so there is no need for a third-party router:

```go
s.mux.HandleFunc("POST /jobs", s.handleSubmitJob)
s.mux.HandleFunc("GET /jobs/{id}", s.handleGetJob)
s.mux.HandleFunc("GET /jobs", s.handleListJobs)
```

The API is deliberately minimal at this stage. Submit a job, check its status, list all jobs. That is enough to verify the entire pipeline works end to end.

---

## Testing Concurrent Behavior

The most important test in the project verifies that multiple workers never double-claim a job:

```go
func TestWorker_MultipleWorkerNoDuplicates(t *testing.T) {
    s := store.NewMemoryStore()

    ids := make([]string, 5)
    for i := range ids {
        ids[i] = s.AddJob("echo hello")
    }

    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()

    for i := 1; i <= 3; i++ {
        w := NewWorker(i, s)
        go w.Start(ctx)
    }

    for _, id := range ids {
        waitForStatus(t, s, id, store.StatusDone, 5*time.Second)
    }

    for _, id := range ids {
        job, _ := s.GetJob(id)
        if job.Attempts != 1 {
            t.Errorf("job %s: expected 1 attempt, got %d", id, job.Attempts)
        }
    }
}
```

Five jobs, three workers. Every job should complete exactly once. If any job shows more than one attempt, it means two workers claimed the same job, and our mutex has a bug. Running this with `go test -race` adds the race detector on top, which catches data races that might not manifest as wrong results but are still undefined behavior.

The `waitForStatus` helper deserves a mention. Since workers run in separate goroutines, you cannot assert on job status immediately after starting them. The helper polls the store until the expected status appears or a timeout expires. It is a small utility, but without it every concurrent test becomes either flaky or artificially slow.

---

## Graceful Shutdown

The main function wires everything together: start the store, start the HTTP server, spin up workers, and wait for a shutdown signal.

```go
quit := make(chan os.Signal, 1)
signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)
<-quit

cancel()  // signal all workers to stop
wg.Wait() // wait for in-progress jobs to finish
```

`context.WithCancel` propagates the shutdown signal to every worker. Each worker finishes whatever job it is currently processing, then exits its polling loop. `sync.WaitGroup` ensures the main goroutine does not exit until every worker has confirmed it is done.

This is important: we do not kill workers mid-job. A job that was halfway through execution completes normally. Only *idle* workers (waiting on the ticker) exit immediately. 

---

## What This Gets Right and What It Defers

Workron can accept jobs over HTTP, execute them concurrently across multiple workers, retry transient failures, and shut down without losing in-progress work. The race detector confirms no data races under contention.

What it cannot do yet: survive a process restart (all state is in memory), detect a crashed worker (no heartbeats), or coordinate across multiple machines (everything is in one process).

The [next post](/posts/splitting-and-surviving-failures-workron) tackles those limitations. The scheduler and workers split into separate binaries communicating over HTTP instead of shared memory. Once they are separate, a new problem emerges: what happens when a worker dies mid-job with no one watching? The answer involves an interface with two methods, a background goroutine, and a 30-second timeout.

---

## References and Further Reading

- [Introducing the Go Race Detector](https://go.dev/blog/race-detector) — The official Go blog post on the race detector. Workron uses `go test -race` to verify that concurrent job claiming has no data races.
- [Data Race Detector](https://go.dev/doc/articles/race_detector) — Go's official documentation on the race detector, including common data race patterns and how to detect them. Several of the patterns listed (unprotected map access, race on loop counter) were relevant during Workron's development.
- [Go Concurrency Patterns](https://go.dev/talks/2012/concurrency.slide) — Rob Pike's foundational talk on Go concurrency. The polling worker loop in Workron uses the `select` + ticker pattern described here.
- [Share Memory By Communicating](https://go.dev/blog/codelab-share) — The Go blog post that explains the proverb Workron deliberately goes against. Worth reading to understand why channels are the default recommendation, and when mutexes are the better choice.
- [Go `sync` package documentation](https://pkg.go.dev/sync) — Official docs for `sync.Mutex` and `sync.RWMutex`, the primitives protecting Workron's job store.
- [Go 1.22 enhanced `ServeMux` routing](https://go.dev/blog/routing-enhancements) — The routing improvements in Go 1.22 that let Workron use `"POST /jobs"` and `"GET /jobs/{id}"` patterns without a third-party router.
