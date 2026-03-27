---
author: Lynn Wang
pubDatetime: 2026-03-17T10:00:00Z
modDatetime: 2026-03-18T14:20:00Z
title: "Splitting and Surviving Failures: HTTP Workers and Heartbeat Detection in Go"
slug: splitting-and-surviving-failures-workron
featured: true
draft: false
tags:
  - distributed-systems
  - go
  - fault-tolerance
  - engineering-tradeoffs
description: "Splitting Workron into separate binaries, then solving the harder problem: detecting when a worker dies mid-job without telling anyone."
---

In the [previous post](/posts/building-concurrent-monolith-atomic-job-claiming-go), Workron was a single Go process: a REST API, a shared job store, and a pool of worker goroutines all living in the same memory space. The mutex guaranteed correctness. Everything was simple.

This post breaks that simplicity on purpose. The scheduler and workers split into separate binaries communicating over HTTP. Then, once they are separated, a harder problem appears: what happens when a worker crashes mid-job and never reports back?

---

## The Split

In the monolithic version, workers called `store.ClaimJob()` directly. They held a pointer to the same `MemoryStore` as the HTTP server. The mutex mediated all access. This is fast and correct, but it means every worker must live in the same process on the same machine.

The goal now is to run the scheduler on one machine and workers on others. Workers can no longer touch the store directly. They need to talk to the scheduler over the network.

```
Before:
  Worker → store.ClaimJob() → shared map (same process)

After:
  Worker → HTTP GET /jobs/next → Scheduler → store.ClaimJob() → shared map
```

The question is how to make this change without rewriting the worker.

---

## The JobSource Interface

The worker does not actually need much from the store. It needs to claim a job, report results, and send heartbeats. That is it. So instead of depending on the concrete `MemoryStore`, the worker depends on an interface:

```go
type JobSource interface {
    ClaimJob() (*store.Job, bool)
    UpdateJobStatus(id string, status store.JobStatus)
    SendHeartbeat(id string) error
}
```

`MemoryStore` already has these methods. A new `SchedulerClient` can implement the same interface by making HTTP calls instead. The worker does not know or care which one it is talking to:

```go
// Standalone mode — direct store access
w := worker.NewWorker(1, memoryStore)

// Distributed mode — HTTP calls to scheduler
client := worker.NewSchedulerClient("http://scheduler:8080")
w := worker.NewWorker(1, client)
```

Same `NewWorker`, same `Start` loop, same `process` method. The transport changed; the logic did not.

This is not a theoretical nicety. It means every worker test written in the monolithic design still passes without modification. The tests use `MemoryStore` directly, which is fast and deterministic. The HTTP path gets its own integration tests. Neither set knows about the other.

---

## The HTTP Client

The `SchedulerClient` translates interface methods into HTTP calls. Claiming a job is a GET request:

```go
func (c *SchedulerClient) ClaimJob() (*store.Job, bool) {
    resp, err := c.httpClient.Get(c.baseURL + "/jobs/next")
    if err != nil {
        return nil, false
    }
    defer resp.Body.Close()

    if resp.StatusCode == http.StatusNoContent {
        return nil, false
    }

    var job store.Job
    json.NewDecoder(resp.Body).Decode(&job)
    return &job, true
}
```

Two things worth noting about this compared to the in-process version.

**No copy is needed.** In the monolithic store, `ClaimJob` returns a deep copy to prevent workers from holding a pointer into the shared map. Over HTTP, `json.Decode` creates an entirely new `Job` struct from the response bytes. The network serialization *is* the copy.

**Atomicity moves to the server.** When multiple remote workers call `GET /jobs/next` concurrently, each request hits the scheduler's `handleClaimJob` handler, which calls `MemoryStore.ClaimJob()`, which holds the mutex. The atomicity boundary moved from "mutex in shared memory" to "HTTP request to a single server." The workers do not coordinate with each other at all. They just race to the scheduler, and the scheduler serializes their claims.

---

## Reporting Results

In the monolithic version, the worker decided whether to retry a failed job. It checked `job.Attempts < job.MaxRetries` and either re-queued or permanently failed the job. This worked because the worker had direct store access.

Over HTTP, that responsibility shifts to the scheduler. The worker simply reports success or failure:

```go
func (c *SchedulerClient) UpdateJobStatus(id string, status store.JobStatus) {
    switch status {
    case store.StatusDone:
        c.ReportDone(id)    // POST /jobs/{id}/done
    case store.StatusFailed, store.StatusPending:
        c.ReportFail(id)    // POST /jobs/{id}/fail
    }
}
```

The scheduler's `/jobs/{id}/fail` handler inspects the attempt count and decides whether to re-queue or permanently fail:

```go
if job.Attempts < job.MaxRetries {
    s.store.UpdateJobStatus(id, store.StatusPending)  // retry
} else {
    s.store.UpdateJobStatus(id, store.StatusFailed)   // permanent
}
```

This is a deliberate design choice. The scheduler is the source of truth for job state. Letting the worker make retry decisions over HTTP would mean the worker and scheduler could disagree about how many attempts have been made, especially if a request is lost or duplicated. Centralizing the decision eliminates that class of bug.

---

## The New Problem

The split works. Two binaries, communicating over HTTP, coordinating job execution. But it introduces a failure mode that did not exist before.

In the monolith, if a worker goroutine panicked, Go's runtime handled it. The process either recovered or crashed entirely, taking all workers with it. Either way, the state was consistent.

Now imagine: a worker process claims a job, starts executing it, and the machine loses power. The scheduler still shows the job as `running`. No one is working on it. No one will ever report it as done or failed. Without intervention, it stays `running` forever.

This is the fundamental challenge of distributed systems. In a single process, a crash is total. Across processes, a crash can be *partial*, and partial failure is much harder to reason about.

---

## Heartbeats: Proving You Are Alive

The solution is to require workers to continuously prove they are still alive. While processing a job, the worker spawns a background goroutine that pings the scheduler every 5 seconds:

```go
func (w *Worker) sendHeartbeats(ctx context.Context, jobID string) {
    ticker := time.NewTicker(heartbeatInterval)
    defer ticker.Stop()

    for {
        select {
        case <-ctx.Done():
            return
        case <-ticker.C:
            if err := w.source.SendHeartbeat(jobID); err != nil {
                log.Printf("[worker-%d] heartbeat failed for job %s: %v",
                    w.id, jobID, err)
            }
        }
    }
}
```

The `process` method starts this goroutine before executing the command, and cancels it when the command finishes:

```go
func (w *Worker) process(job *store.Job) {
    hbCtx, hbCancel := context.WithCancel(context.Background())
    defer hbCancel()  // stops heartbeat when process() returns
    go w.sendHeartbeats(hbCtx, job.ID)

    err := w.executor.Execute(job.Command)
    // ... handle result
}
```

The `defer hbCancel()` is critical. Without it, every completed job would leave behind a dangling goroutine still sending heartbeats for a job that no longer exists. The context cancellation cleanly stops the goroutine regardless of whether the job succeeded, failed, or the worker is shutting down.

On the scheduler side, each heartbeat hits a simple endpoint that records the timestamp:

```go
func (s *Server) handleHeartbeat(w http.ResponseWriter, r *http.Request) {
    id := r.PathValue("id")
    job, found := s.store.GetJob(id)
    if !found {
        http.Error(w, "job not found", http.StatusNotFound)
        return
    }
    if job.Status != store.StatusRunning {
        http.Error(w, "job is not running", http.StatusConflict)
        return
    }
    s.store.UpdateHeartbeat(id)
    w.WriteHeader(http.StatusOK)
}
```

Now the scheduler has a timestamp for every running job: the last time its worker checked in.

---

## The Reaper: Detecting the Dead

A timestamp is only useful if something checks it. That is the reaper's job. It runs as a background goroutine on the scheduler, ticking every 10 seconds:

```go
func StartReaper(ctx context.Context, s store.JobStore) {
    ticker := time.NewTicker(reapInterval)
    defer ticker.Stop()

    for {
        select {
        case <-ctx.Done():
            return
        case <-ticker.C:
            reap(s)
        }
    }
}
```

Each tick, it scans all running jobs and checks their heartbeat:

```go
func reap(s store.JobStore) {
    now := time.Now()
    for _, job := range s.ListRunningJobs() {
        if job.LastHeartbeat != nil && now.Sub(*job.LastHeartbeat) <= heartbeatTimeout {
            continue
        }

        if job.Attempts < job.MaxRetries {
            s.UpdateJobStatus(job.ID, store.StatusPending)
        } else {
            s.UpdateJobStatus(job.ID, store.StatusFailed)
        }
    }
}
```

A job is considered orphaned if its heartbeat is either nil (worker never sent one) or older than 30 seconds. Orphaned jobs with retries remaining go back to `pending`. Jobs that have exhausted their retries are permanently `failed`.

The `ListRunningJobs` method deserves a mention. An earlier design used `ListJobs()` and filtered in the reaper. That works, but it means the reaper scans every completed job on every tick, which is wasteful as the job count grows. `ListRunningJobs` pushes the filter into the store, where it can eventually become a `WHERE status = 'running'` query when the store moves to a database.

---

## The False Positive Problem

There is a subtle failure mode worth acknowledging. What if the network between a worker and the scheduler is slow? The worker is healthy and actively processing the job, but its heartbeat packets are delayed. The reaper sees a stale timestamp and re-queues the job. Now two workers are processing the same job.

This is the classic false positive problem in distributed systems. Workron handles it by designing jobs to be **idempotent where possible**: running the same job twice produces the same result as running it once. The alternative, guaranteeing exactly-once execution, requires distributed transactions or fencing tokens, which is significantly more complex.

At-least-once execution with idempotency is the pragmatic starting point. It is also how most real-world job schedulers, from Sidekiq to SQS, handle this problem. Exactly-once is something to revisit if the use case demands it, but it is not free, and it is not always necessary.

---

## The Timing

The specific numbers, 5-second heartbeat interval and 30-second timeout, are not arbitrary. The heartbeat interval needs to be short enough that the reaper has recent data, but not so short that it floods the scheduler with requests. Five seconds means a healthy worker sends 6 heartbeats before the timeout expires. Even if one or two are delayed or lost, the reaper still sees a recent timestamp.

The 30-second timeout needs to be long enough to tolerate network jitter and temporary slowdowns, but short enough that orphaned jobs do not sit idle for minutes. Thirty seconds is a common default in systems like Consul and etcd for similar health checks.

The 10-second reap interval means the worst case for detecting a dead worker is about 40 seconds: the last heartbeat was sent just before the worker died (0 seconds stale), plus 30 seconds until it is considered stale, plus up to 10 seconds until the next reaper tick. For a background job scheduler, this is acceptable.

---

## Testing Failure Detection

The reaper is easy to test because it is a pure function. `reap` takes a store, inspects it, and mutates it. No network, no timing, no goroutines:

```go
func TestReap_RequeuesStaleJob(t *testing.T) {
    s := store.NewMemoryStore()
    id := s.AddJob("echo hello")
    s.ClaimJob()

    s.SetLastHeartbeat(id, time.Now().Add(-60*time.Second))

    reap(s)

    job, _ := s.GetJob(id)
    if job.Status != store.StatusPending {
        t.Errorf("expected status pending, got %s", job.Status)
    }
}
```

`SetLastHeartbeat` is a test helper that sets a specific timestamp on a job, letting us simulate a stale heartbeat without actually waiting 30 seconds. The reaper does not know or care whether the staleness is real or simulated. It just checks the math.

The heartbeat goroutine gets a separate test that runs a long job and verifies the timestamp is being updated:

```go
func TestWorker_SendsHeartbeatsDuringLongJob(t *testing.T) {
    s := store.NewMemoryStore()
    id := s.AddJob("sleep 12")

    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()

    w := NewWorker(1, s)
    go w.Start(ctx)

    time.Sleep(12 * time.Second)

    job, _ := s.GetJob(id)
    if job.LastHeartbeat == nil {
        t.Fatal("expected heartbeat to be set")
    }
    if time.Since(*job.LastHeartbeat) > 6*time.Second {
        t.Error("heartbeat is stale — goroutine may not be running")
    }
}
```

And a complementary test verifies the goroutine stops after the job completes, catching goroutine leaks before they become a problem in production.

---

## What We Have Now

After these changes, Workron handles the full lifecycle of a distributed job:

1. A client submits a job via the REST API
2. The scheduler stores it as `pending`
3. A remote worker claims it over HTTP, moving it to `running`
4. While executing, the worker sends heartbeats every 5 seconds
5. On completion, the worker reports success or failure
6. If the worker dies, the reaper detects the missing heartbeat and re-queues the job

The system tolerates worker crashes without human intervention. No job gets stuck in `running` forever.

What it still cannot do: survive a *scheduler* crash. All job states live in memory. Kill the scheduler process and everything is gone. The [next post](/posts/persisting-jobs-with-sqlite-workron) adds SQLite persistence, swapping the in-memory store for a database without changing a single line of business logic. Along the way, SQLite's concurrency model turns out to be more surprising than the persistence itself.

---

## References and Further Reading

- [How we designed Dropbox ATF: an async task framework](https://dropbox.tech/infrastructure/asynchronous-task-scheduling-at-dropbox) —  Dropbox’s ATF uses heartbeat-based liveness monitoring during task execution. It was a useful reference for Workron’s heartbeat goroutine and reaper design.
- [Sidekiq Wiki: Reliability](https://github.com/sidekiq/sidekiq/wiki/Reliability) — Describes how Sidekiq detects orphaned jobs and recovers work from dead workers. This was a helpful reference for stale-heartbeat detection and job re-queuing in Workron.
- [Designing a Distributed Job Scheduler (PhonePe Clockwork)](https://snehasishroy.com/deep-dive-of-the-distributed-job-scheduler-that-powers-over-2-billion-daily-jobs-at-phonepe) — A production-scale scheduler architecture covering job acceptance, scheduling, execution, and heartbeat monitoring. It provides a useful large-scale reference point for Workron’s design.
- [From Cron to Distributed Schedulers](https://itnext.io/from-cron-to-distributed-schedulers-scaling-job-execution-to-thousands-of-jobs-per-second-ef05955bf3d9) —  A walkthrough of how job execution systems evolve from single-machine scheduling to distributed coordination. It maps well to the progression of Workron so far.
- [Go `net/http/httptest` package](https://pkg.go.dev/net/http/httptest) — Official documentation for the test server used in Workron’s client integration tests. The `httptest.NewServer` pattern makes it easy to test HTTP clients against a real server in process.
- [Go `context` package](https://pkg.go.dev/context) — Official documentation for Go’s context package. Workron uses `context.WithCancel` for worker shutdown, heartbeat lifecycle management, and reaper termination.
