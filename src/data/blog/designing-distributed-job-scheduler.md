---
author: Lynn Wang
pubDatetime: 2026-03-09T10:00:00Z
title: "Before the Code: Designing a Distributed Job Scheduler in Go"
slug: designing-distributed-job-scheduler-go
featured: true
draft: false
tags:
  - distributed-systems
  - go
  - system-design
  - engineering-tradeoffs
description: "Architectural decisions, trade-offs, and the blueprint behind building a job scheduler from scratch."
---

Two questions keep distributed systems engineers up at night: how do you guarantee that two parallel workers never process the same job simultaneously, and what happens to a job that was 80% complete when the worker handling it suddenly crashes?

I decided to stop wondering and build **[Workron](https://github.com/lrdinsu/workron)**, a job scheduler from scratch to find out. This post is not a step-by-step code tutorial. It is more of a blueprint: the architectural decisions I made before writing a single line of code, the trade-offs I faced, and why I chose the patterns I did.


---

## Why Build This?

Background job queues are everywhere. Sending emails, transcoding videos, processing payments, generating reports, any task too slow for a synchronous HTTP response ends up in a queue. Tools like Celery, Sidekiq, and BullMQ abstract all of this away beautifully.

But abstraction has a cost: it is easy to stop understanding what is actually happening underneath. When a job queue starts dropping tasks under load, or a worker silently crashes mid-job, the black box stops being friendly. I wanted to confront those assumptions directly, so I decided to build one from scratch.

---

## The Architectural Evolution

The single biggest mistake, at least from what I've read and learned, is trying to distribute a system from day one. Distributed systems add an entirely new class of problems, network failures, split-brain scenarios, partial writes, on top of the concurrency problems you already need to solve. Get the concurrency right first.

This is why Workron is being built in two deliberate stages.

### Stage 1: The Concurrent Monolith

Everything runs in a single Go process. A REST API accepts jobs, stores them in an in-memory data structure, and a pool of worker goroutines concurrently polls that store for work. The entire engineering challenge of this stage is one question: how do you prevent two goroutines from claiming the same job at the same time?

```
┌──────────────────────────────────────────────┐
│               Single Go Process              │
│                                              │
│   REST API  ──►  Job Store  ◄──  Workers     │
│   (HTTP)        (in-memory)     (goroutines) │
│                  [pending]       Worker 1    │
│                  [running]       Worker 2    │
│                  [done]          Worker 3    │
└──────────────────────────────────────────────┘
```

### Stage 2: The Distributed Split

Once the monolithic queue is solid, the architecture splits into two separate binaries: a scheduler and a worker, that communicate over HTTP. Workers can now run on entirely different machines. Memory is no longer shared; the scheduler becomes the central source of truth, eventually backed by a database. The challenge shifts from managing shared memory to managing network unreliability and node failures.

```
┌─────────────────────┐         ┌──────────────────────┐
│      Scheduler      │         │       Workers        │
│                     │  HTTP   │                      │
│  REST API           │◄───────►│  Worker Process A    │
│  Job Store (DB)     │         │  Worker Process B    │
│  Heartbeat Monitor  │         │  Worker Process C    │
└─────────────────────┘         └──────────────────────┘
    Source of truth                  Any machine
```

---

## Engineering Trade-offs & Decisions

Architecture is mostly a series of trade-offs disguised as decisions. Here are the three that shaped Workron the most.

### 1. Channels vs. Mutexes

Go is famous for its concurrency proverb: *"Do not communicate by sharing memory; instead, share memory by communicating"*, meaning prefer channels. For a job scheduler, I explicitly chose the opposite: a shared map protected by a `sync.RWMutex`. Here is why.

Channels are fundamentally black boxes. If you push 100 jobs into a Go channel, a job loses its identity the moment it enters. You cannot build a `GET /jobs/:id` API endpoint to query the status of a specific job, because the job is now an anonymous value flowing through a pipe. You cannot list all running jobs, retry a specific failed job, or inspect what a worker is currently doing.

A job scheduler is not a pure data pipeline, it is a **database of work**. Jobs have identity, lifecycle, and state that needs to be queryable at any time.

```
Channel approach (job loses identity):

 Producer ──► [job] ──► [job] ──► [job] ──► Channel ──► Worker
                                                 ↑
                                        Can't query in here

Mutex + Map approach (job keeps identity):

 Producer ──► Store.AddJob()  ──► map["job-123"] = { status: pending }
                                                         ↑
 REST API ◄── Store.GetJob()  ◄─────────────── Queryable anytime
 Worker   ◄── Store.ClaimJob() ─── atomically sets status: running
```

The mutex approach also has a clean migration path. The logic for claiming a job atomically, lock, find a pending job, update its status, unlock, maps almost identically to how a relational database handles an atomic `UPDATE ... WHERE status = 'pending' LIMIT 1`. When Workron moves to SQLite later, the core claim logic will not change, only where the data lives.

The honest cost of this choice: workers have to poll the store periodically rather than being pushed to work instantly. A channel-based approach wakes workers up the moment a job arrives. With mutex polling, there is a small latency between job submission and execution. For Workron's use case, orchestrating background jobs, not microsecond trading, this is an acceptable trade-off.

### 2. HTTP vs. gRPC for Worker Communication

In the distributed stage, the scheduler and workers need to talk over a network. gRPC is the standard choice for service-to-service communication in modern infrastructure. It is fast, strongly typed via Protobuf contracts, and streams efficiently.

For Workron, I chose standard HTTP with JSON, for one practical reason: **debuggability**. Go's `net/http` standard library is extraordinarily capable and requires zero external dependencies. More importantly, every interaction between the scheduler and a worker can be tested with a single `curl` command. When something breaks in production, you want the simplest possible tool to diagnose it.

gRPC's performance advantages matter enormously at scale. Workron's goal is robust orchestration, not microsecond latency, so HTTP felt like the right pragmatic starting point, though I'd be curious to revisit this decision as the project grows.

### 3. Heartbeats vs. Timeouts: Knowing When a Worker Dies

This is the hardest problem in the distributed stage, and honestly the one I spent the most time thinking about before writing any code.

Imagine a worker claims a job and starts processing it. Halfway through, the worker's machine loses power. From the scheduler's perspective, the job is still marked as `running`, but nothing is actually running it. Without intervention, it will stay `running` forever.

The naive solution is a simple timeout: if a job has been `running` for more than N minutes, mark it as failed and re-queue it. This breaks immediately in practice. What if the job legitimately takes a long time? What if the network between the worker and scheduler is slow, but the worker is perfectly healthy?

Workron uses a **heartbeat mechanism** instead. While processing a job, the worker periodically sends a lightweight ping to the scheduler, essentially saying "I'm still alive, still working on job-123." The scheduler records the timestamp of each heartbeat. A separate background goroutine on the scheduler continuously checks: if any `running` job has not received a heartbeat in the last 30 seconds, the worker is assumed dead and the job is re-queued.

```
Worker                              Scheduler
  │                                     │
  │── POST /jobs/:id/heartbeat ────────►│  records timestamp
  │                        (every 5s)   │
  │── POST /jobs/:id/heartbeat ────────►│  records timestamp
  │                                     │
  │  [worker crashes]                   │
  │                                     │  background checker:
  │                                     │  last heartbeat > 30s ago?
  │                                     │  → reset job to "pending"
  │                                     │  → another worker picks it up
```

One subtle failure mode worth noting: what if the heartbeat packet is delayed by the network, but the worker is still healthy? This is the classic **false positive** problem in distributed systems. Workron handles this by making jobs **idempotent where possible**, running the same job twice should produce the same result as running it once. The alternative which is guaranteeing exactly-once execution requires distributed transactions and is significantly more complex. At-least-once with idempotency feels like the pragmatic starting point for this project, even if it is not a perfect guarantee.

---

## The Implementation Roadmap
- **Concurrent monolith** — Atomic job claiming with mutex-protected state.
  The question: how do N goroutines share one queue without stepping on each other?
- **The distributed split** — Separating into two binaries.
  The question: how do two processes coordinate without shared memory?
- **Failure detection** — Heartbeat-based monitoring.
  The question: how does the scheduler know a worker is dead without the worker telling it?
- **Persistence** — Database-backed storage.
  The question: how does the system survive a complete scheduler restart without losing job state?
- **Advanced scheduling** — Cron scheduling, priority queues, and DAG dependencies.
  The question: how do you model complex real-world workflows?

---

## What's Next

The next post dives into the code for Concurrent monolith, specifically, how to implement the `ClaimJob()` function using `sync.RWMutex` to guarantee that no two goroutines ever process the same job, and how to wire it to a live REST API you can test with curl from day one.
