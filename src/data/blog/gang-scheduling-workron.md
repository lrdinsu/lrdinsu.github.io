---
author: Lynn Wang
pubDatetime: 2026-04-21T10:00:00Z
title: "All or Nothing: Gang Scheduling in Workron"
slug: gang-scheduling-workron
featured: true
draft: false
tags:
  - distributed-systems
  - go
  - gang-scheduling
  - interface-design
  - engineering-tradeoffs
description: "Adding gang scheduling to Workron: a reserved state, an admission cycle that places N tasks simultaneously, and why running siblings are left untouched when one task fails."
---

After the [observability work](/posts/observability-slog-prometheus-workron), Workron had structured logging, Prometheus metrics, and per-request tracing. The system explained itself well. But all jobs were still treated identically: submit, wait for a free worker, run.

In the meantime, workers gained the ability to register with resource specs. Each worker reports its available VRAM and memory when joining the cluster. That made a new class of job possible.

Distributed training jobs require N workers running coordinated processes in parallel. PyTorch with NCCL, for example, uses a collective communication ring: all N processes must join before any of them can exchange gradients. If only three of four nodes start, the fourth never joins, the ring never forms, and the job hangs at initialization. Partial starts are not degraded performance. They are functionally broken.

This post covers gang scheduling: submitting a single request that creates N coordinated tasks, where all N must be placed on suitable workers at the same time before any of them begin.

## Table of Contents

## A Third Status

The first question is how to prevent workers from claiming individual gang tasks before the full set is placed.

One option: keep gang tasks as `pending` and add a filter to `ClaimJob`. Before handing out a task, check if all sibling tasks in the gang have also been claimed. Skip it if they have not.

That does not work. Workers poll independently. Worker 1 polls, sees a gang task, skips it because the gang is not fully claimed. Worker 2 polls, same result. None of them ever claim enough tasks to satisfy the condition, because satisfying the condition requires claiming tasks in the first place.

The solution is to separate placement from claiming. Before workers see any gang task, the scheduler decides where each task goes and reserves the assignment. Workers do not compete for gang tasks. Each worker is told "this task is yours."

That requires a new status:

```
Non-gang jobs:
  submit → pending → running → done/failed

Gang jobs:
  submit → blocked → (admission cycle) → reserved → running → done/failed
```

Gang tasks start as `blocked`. No worker can claim a blocked task through the normal `GET /jobs/next` path. The admission cycle runs in the background and, when it finds enough suitable workers, atomically transitions all N tasks to `reserved`, each with a specific worker ID assigned. Workers claim their pre-assigned tasks by passing `?worker_id=` to the claim endpoint.

The status names carry their weight here. `blocked` means "waiting for gang placement." `reserved` means "placed but not yet started." The distinction is visible in any job listing without needing to cross-reference gang state separately.

## The Admission Cycle

The admission cycle is a background goroutine that ticks every five seconds. Each tick:

1. Roll back any stale reservations (more on this shortly).
2. Find all gangs where every task is `blocked`. Partial gangs are skipped.
3. Sort the candidates by placement priority.
4. Compute available capacity across all active workers.
5. For each gang in order: try to place it. If placement succeeds, call `ReserveGang` and subtract the used capacity from the available pool before moving to the next candidate.

Step 5 matters. After reserving gang A, the available capacity for gang B is immediately reduced, even though no tasks have started running yet. Reserved capacity is committed capacity. Without this subtraction step, the cycle could place multiple gangs on the same workers in a single tick.

`ReserveGang` is atomic. It either transitions all N tasks to `reserved` with their assigned worker IDs, or it fails entirely. In Postgres, this is a transaction with `SELECT ... FOR UPDATE ORDER BY id` on the gang's task rows before updating them. The `ORDER BY id` is load-bearing: two concurrent admission cycles grabbing the same rows in different orders would deadlock. Sorting the IDs before locking prevents that.

## Placement Order: Largest Gang First

When multiple gangs are waiting, the admission cycle tries the largest gang first, using priority as a tiebreaker and submission time as the final tiebreaker.

The intuition: suppose you have four available workers and two waiting gangs, A with three tasks and B with four tasks. If you place A first, it takes three workers. B needs four and cannot be placed. B starves until A finishes.

If you place B first, it takes all four workers. A must wait. But when B finishes, all four workers are free and A places immediately. Larger gangs are harder to place because they demand more simultaneous capacity. Trying them first avoids the situation where small gangs accumulate and permanently crowd out large ones.

The sort is stable enough to be deterministic within a tick: size descending, then priority descending, then creation time ascending.

## Capacity Accounting

Available capacity for a worker is its total resources minus what is currently in use. "In use" means both `running` jobs and `reserved` jobs.

The `reserved` part is easy to miss. A task in `reserved` status has not started executing, so a naive implementation would not subtract it from the worker's available capacity. But the worker is committed to that task. Skipping the subtraction would allow a second gang to be placed on the same worker, promising that worker to two gangs at once.

```go
for _, j := range jobs {
    if j.WorkerID == "" || j.Resources == nil {
        continue
    }
    if j.Status != store.StatusRunning && j.Status != store.StatusReserved {
        continue
    }
    cap := capMap[j.WorkerID]
    cap.Used.VRAMMB += j.Resources.VRAMMB
    cap.Used.MemoryMB += j.Resources.MemoryMB
}
```

Both statuses consume capacity. The capacity snapshot is built once per admission tick from the live job list, so it reflects the world as it was at the start of that tick.

## Stale Reservations

After a gang is reserved, each worker has 30 seconds to claim its task. If it does not, the admission cycle rolls back the reservation: reserved tasks return to `blocked`, worker IDs are cleared, and the gang becomes eligible for placement again on the next tick.

Stale reservations happen when a worker dies between the admission cycle running and that worker calling `GET /jobs/next`. Without rollback, those tasks would sit in `reserved` permanently, assigned to a dead worker that will never come back.

Each task stores a `reserved_at` timestamp when it transitions to `reserved`. On each admission tick, the cycle scans for reserved tasks older than 30 seconds and calls `RollbackGang` for the affected gangs. This scan runs before placement, so just-rolled-back gangs are immediately re-eligible in the same tick.

Two fields on the job struct serve different purposes here. `ReservationEpoch` is a logical counter that increments each time a gang is reserved. If a worker somehow holds a stale reference from a previous reservation and tries to claim the task after a rollback and re-reservation, the epoch will not match, and the claim is rejected. `ReservedAt` is the wall-clock timestamp used only for the 30-second timeout check. They are separate because they answer different questions.

## When a Task Fails

When a running task reports failure, the rest of the gang cannot proceed. The communication ring is broken. The scheduler needs to prevent the remaining tasks from being partially re-placed or claimed while some siblings are still running.

`FailGang` only changes tasks in `blocked`, `reserved`, or `pending` status. Running and `done` tasks are left untouched.

This is intentional. When one worker in a distributed training job fails, the other workers are still running. They will detect the failure themselves through a broken NCCL connection or a failed collective operation, and report failure via `POST /jobs/{id}/fail` or get reaped when their heartbeat goes stale. Sending a preemption signal from the scheduler is a separate, more complex operation that Workron does not do yet.

The practical effect: `FailGang` clears out the queued work immediately, so the gang stops accumulating partial state. The running siblings are expected to fail on their own. When each one eventually does, `FailGang` is called again, finds no remaining blocked or reserved tasks to act on, and returns cleanly.

One detail about the failed task itself: `handleJobFail` sets it to `pending` first (for retry accounting), then `FailGang` processes pending siblings and sets them to `blocked`. The originally failed task ends up at `blocked`. This means it participates as a full member in the gang's next placement attempt once all siblings have settled.

## The GangStore Interface

Adding gang scheduling to the `JobStore` interface would force all three backends to implement it at the same time. Instead, `GangStore` is a separate optional interface, discovered via type assertion in the server constructor:

```go
if gs, ok := s.(store.GangStore); ok {
    srv.gangStore = gs
}
```

Gang-specific handlers return `501 Not Implemented` if `gangStore` is nil. The admission cycle and reaper check for the interface before calling gang methods. This follows the same pattern as `WorkerStore` and `ReaperLocker` elsewhere in the codebase.

The six methods on the interface map directly to the operations the admission cycle and HTTP handlers need:

```go
type GangStore interface {
    AddGang(ctx context.Context, params AddJobParams) (gangID string, taskIDs []string)
    ListGangTasks(ctx context.Context, gangID string) []*Job
    ReserveGang(ctx context.Context, gangID string, assignments map[string]string, epoch int) error
    ClaimReservedJob(ctx context.Context, workerID string) (*Job, bool)
    RollbackGang(ctx context.Context, gangID string) error
    FailGang(ctx context.Context, gangID string, retry bool) error
}
```

All three backends implement this interface now. The compliance test approach from earlier posts applies here too: tests are written once against the interface and wired to each backend's factory:

```go
func TestMemory_FailGangRetry(t *testing.T)   { testFailGangRetry(t, newTestMemoryGangJobStore) }
func TestSQLite_FailGangRetry(t *testing.T)   { testFailGangRetry(t, newTestSQLiteGangJobStore) }
func TestPostgres_FailGangRetry(t *testing.T) { testFailGangRetry(t, newTestPostgresGangJobStore) }
```

The same scenarios run against all three: reserve succeeds when tasks are blocked, reserve fails when tasks are not all blocked, claim by the wrong worker returns nil, rollback returns tasks to blocked, fail leaves running and done tasks untouched.

## A Known Limitation

There is a specific stuck state in the current implementation that I want to name explicitly, because it is the kind of edge case that a design section can gloss over.

Suppose a gang of three tasks is reserved across workers A, B, C. Workers A and B claim their tasks and start running. Worker C dies before claiming task 3. After 30 seconds, the admission cycle rolls back task 3 to `blocked`. Tasks 1 and 2 are still `running`, waiting for a peer that will never join.

On the next admission tick, the placement loop looks for gangs where every task is `blocked`. This gang does not qualify: two tasks are running, one is blocked. Task 3 stays blocked indefinitely. Tasks 1 and 2 eventually fail through their own peer detection, but `FailGang` on a blocked sibling with `retry=true` just moves it from blocked to blocked. No progress.

The right fix is preemption. When a reservation goes stale and siblings are already running, the scheduler should interrupt the running peers, wait for them to tear down, and return the whole gang to `blocked` atomically so it can be re-placed. That is the next feature, and the current implementation is the baseline it builds on.

## What We Have Now

Workron now supports gang jobs alongside regular jobs. Submitting with `gang_size > 1` creates N tasks that wait until the scheduler can place all of them at once:

```
Submit (gang_size=4, resources={VRAM: 8GB})
  → 4 tasks created, all blocked
      → Admission cycle: 4 workers found with enough VRAM
          → All 4 tasks reserved, each assigned to a specific worker
              → Each worker claims its reserved task → running
                  → All 4 run concurrently
                      → Each reports done
```

Regular jobs are unaffected. They never enter `blocked` status through the gang path, never interact with the admission cycle, and workers claim them via `GET /jobs/next` without a `worker_id` parameter. Gang tasks are invisible to that path entirely.

What this does not handle yet: active preemption of running siblings, as discussed in the limitation above. When one gang task fails or its reservation goes stale, running peers are left to fail naturally through peer detection or heartbeat timeout. Sending an interrupt signal to abort running workers cleanly and rolling back the whole gang atomically is the next step.

Full source: [github.com/lrdinsu/workron](https://github.com/lrdinsu/workron)

---

## References and Further Reading

- [Gang scheduling (Wikipedia)](https://en.wikipedia.org/wiki/Gang_scheduling). — The original concept from operating systems: co-scheduling related threads across processors so they run simultaneously. Workron adapts this to distributed workers over HTTP rather than CPU cores in shared memory.
- [NCCL: NVIDIA Collective Communications Library](https://docs.nvidia.com/deeplearning/nccl/user-guide/docs/overview.html). — The communication backend that makes multi-GPU training sensitive to all-or-nothing placement. If any worker in an NCCL ring is missing, the others block at the initialization barrier.
- [PyTorch Distributed Overview](https://pytorch.org/tutorials/beginner/dist_overview.html). — Covers the process group model that requires all N processes to call `init_process_group` before any of them can proceed. This is the exact scenario gang scheduling is designed for.
- [Kubernetes Coscheduling Plugin](https://github.com/kubernetes-sigs/scheduler-plugins/tree/master/pkg/coscheduling). — Kubernetes's approach to gang scheduling via `PodGroup` resources and `minMember` constraints. A useful reference for seeing how the same problem looks at the container orchestration level.
- [PostgreSQL Advisory Locks](https://www.postgresql.org/docs/current/explicit-locking.html#ADVISORY-LOCKS). — Workron uses `pg_try_advisory_xact_lock` to coordinate the admission cycle across multiple scheduler instances. The admission cycle uses a different lock ID than the reaper so they never block each other.
