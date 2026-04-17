---
author: Lynn Wang
pubDatetime: 2026-04-16T10:00:00Z
title: "Making the Invisible Visible: Structured Logging, Metrics, and Request Tracing"
slug: observability-slog-prometheus-workron
featured: true
draft: false
tags:
  - distributed-systems
  - go
  - observability
  - slog
  - prometheus
  - engineering-tradeoffs
description: "Replacing log.Printf with slog, adding Prometheus metrics, and building request ID tracing. How observability turned Workron from a system that works into one that explains itself."
---

After the [PostgreSQL migration](/posts/postgresql-skip-locked-workron), Workron could store jobs in three backends, claim them concurrently with `SKIP LOCKED`, and survive scheduler restarts. It worked. But "it works" is not the same as "I can see how it works."

When a job takes longer than expected, is it stuck in the queue or is the command itself slow? When the reaper re-queues a job, how often does that happen? When something breaks at 3 AM, can you trace a single request through the logs without guessing?

The answer to all of these was no. Every log line was an unstructured `log.Printf` with a manually typed `[component]` prefix. There were no metrics. There was no way to correlate log lines across a single HTTP request. This post covers three changes that fix that: structured logging with `slog`, Prometheus metrics, and request ID tracing.

## Table of Contents

## Why log.Printf Is Not Enough

The existing logging looked like this:

```
2026/03/23 19:29:24 [server] job job-abc submitted: "echo hello" (depends_on: [])
2026/03/23 19:29:24 [worker-1] picking up job job-abc (attempt 1/3): "echo hello"
2026/03/23 19:29:24 [worker-1] job job-abc done
```

Readable by a human staring at a terminal. Useless for anything automated. If you want to filter by job ID, you need regex. If you want to count errors, you need to parse text. If you want to feed logs into Grafana or Datadog, you need a translation layer.

Go's `log/slog` package (standard library since 1.21) solves this by producing structured JSON with typed key-value fields:

```json
{"time":"2026-03-23T19:29:24Z","level":"INFO","msg":"job submitted","job_id":"job-abc","command":"echo hello"}
```

Every field is a named, typed value. A monitoring tool can filter `level=ERROR`, group by `job_id`, or count `msg="job claimed"` without touching a regex.

## Logger Injection, Not Globals

The first version used `slog.SetDefault(logger)` in `main.go` and called `slog.Info(...)` everywhere. That works for the entry point, but for library code it creates a hidden dependency. The `Server` struct calls a global function that tests cannot control without changing global state.

The fix is dependency injection. Each component receives a `*slog.Logger` through its constructor:

```go
func NewServer(s store.JobStore, logger *slog.Logger, ...) *Server {
    return &Server{
        store:  s,
        logger: logger,
        // ...
    }
}
```

The `Worker` struct takes this one step further. Every worker has an ID, and that ID should appear on every log line the worker produces. Instead of passing `worker_id` manually to each log call, the constructor creates a child logger with the field baked in:

```go
func NewWorker(id int, source JobSource, logger *slog.Logger) *Worker {
    return &Worker{
        id:       id,
        source:   source,
        executor: NewExecutor(),
        logger:   logger.With("worker_id", id),
    }
}
```

Now `w.logger.Info("job done", "job_id", job.ID)` automatically includes `worker_id=1` without the handler needing to remember. The `With` method creates a new logger that carries the field on every subsequent call. The parent logger is unchanged.

This pattern scaled cleanly across the codebase. The `main` function creates one logger, sets it as the default, and passes `slog.Default()` to each component. All 36 original `log.Printf` calls were replaced with structured slog calls. The migration was mechanical: find a log line, identify the embedded data, extract it into key-value pairs.

## Log Levels as Signal

Not all log lines are equal. A heartbeat being sent is normal operation. A job being re-queued by the reaper means a worker probably died. A permanent failure means something is broken. `log.Printf` treats all of these the same. `slog` gives them levels:

| Event | Level | Why |
|-------|-------|-----|
| Job submitted, claimed, done | Info | Normal lifecycle |
| Job re-queued by reaper | Warn | Worker probably died |
| Heartbeat failed | Warn | Transient, but worth watching |
| Job failed permanently | Error | Retries exhausted |
| Response encoding failed | Error | Something is wrong |

In production, you can set the minimum level to `Warn` and only see the unusual events. In development, `Info` shows everything. The levels act as a built-in filter.

## Prometheus Metrics

Structured logs tell you what happened. Metrics tell you how often and how fast. Workron exposes a `GET /metrics` endpoint using `prometheus/client_golang` with three types of metrics.

**Counters** track cumulative events. They only go up:

```
workron_jobs_submitted_total    12
workron_jobs_claimed_total      12
workron_jobs_completed_total    10
workron_jobs_failed_total        1
workron_jobs_retried_total       3
workron_jobs_reaped_total{outcome="requeued"}  1
workron_jobs_reaped_total{outcome="failed"}    0
```

The `reaped` counter uses a label to distinguish two outcomes from the same event. A single `CounterVec` with an `outcome` label is cleaner than two separate counters.

**Histograms** track distributions. Two are particularly useful:

- `workron_job_execution_duration_seconds`: how long jobs take to run, computed from `DoneAt - StartedAt` when a worker reports completion.
- `workron_job_queue_wait_seconds`: how long jobs sit in the queue before being claimed, computed from `StartedAt - CreatedAt` at claim time.

Both are calculated from timestamps that the store already tracks. No new data collection was needed.

**Gauges** track current state. Unlike counters, they go up and down. Workron needs three: `jobs_pending`, `jobs_running`, and `jobs_blocked`.

The naive approach is to increment the gauge when a job enters a state and decrement it when it leaves. This is error-prone. If any state transition path misses a decrement, the gauge drifts permanently. With three store backends (memory, SQLite, PostgreSQL), maintaining accurate increment/decrement logic across all of them is asking for bugs.

Instead, I used a custom `prometheus.Collector`. On each `/metrics` scrape, it calls `store.ListJobs()`, counts jobs by status, and reports the counts as gauge values:

```go
func (c *JobGaugeCollector) Collect(ch chan<- prometheus.Metric) {
    var pending, running, blocked float64
    for _, job := range c.store.ListJobs(context.Background()) {
        switch job.Status {
        case store.StatusPending:
            pending++
        case store.StatusRunning:
            running++
        case store.StatusBlocked:
            blocked++
        }
    }
    ch <- prometheus.MustNewConstMetric(c.pendingDesc, prometheus.GaugeValue, pending)
    ch <- prometheus.MustNewConstMetric(c.runningDesc, prometheus.GaugeValue, running)
    ch <- prometheus.MustNewConstMetric(c.blockedDesc, prometheus.GaugeValue, blocked)
}
```

The gauges are always accurate because they are computed fresh on every scrape. The trade-off is that each scrape runs a full `ListJobs` query. For a scheduler handling thousands of jobs, this might matter. For Workron's scale, Prometheus scrapes every 15 to 30 seconds, and `ListJobs` is fast on all three backends.

## Custom Registry, Not the Global Default

Prometheus provides a global `DefaultRegisterer` that most tutorials use. Workron creates a custom `prometheus.Registry` instead and passes it into the `Server`:

```go
registry := prometheus.NewRegistry()
registry.MustRegister(collectors.NewProcessCollector(collectors.ProcessCollectorOpts{}))
registry.MustRegister(collectors.NewGoCollector())

m := metrics.NewMetrics()
m.Register(registry)
```

The `/metrics` endpoint serves this registry specifically:

```go
s.mux.Handle("GET /metrics", promhttp.HandlerFor(s.registry, promhttp.HandlerOpts{}))
```

The reason is testability. If tests register metrics on the global default registry, parallel tests conflict with duplicate registration errors. A custom registry per test avoids that entirely. Tests create their own `metrics.NewMetrics()` and `prometheus.NewRegistry()` without affecting each other.

## Request ID Tracing

The final piece ties everything together. When a client submits a job and then asks "why is my job still pending?", you need to trace that specific request through the logs. Without a request ID, you are searching for a needle in a haystack of identical-looking log lines.

The implementation lives entirely in `ServeHTTP`, the single method that every HTTP request passes through:

```go
func (s *Server) ServeHTTP(w http.ResponseWriter, r *http.Request) {
    requestID := uuid.New().String()
    logger := s.logger.With("request_id", requestID)
    ctx := context.WithValue(r.Context(), loggerKey, logger)
    w.Header().Set("X-Request-ID", requestID)
    s.mux.ServeHTTP(w, r.WithContext(ctx))
}
```

Four things happen before the request reaches any handler:

1. A UUID is generated.
2. A child logger is created with `request_id` baked in (same `With` pattern as the worker).
3. The logger is stored in the request context.
4. The `X-Request-ID` header is set on the response so the client can reference it later.

Handlers extract the logger from context with a helper:

```go
func (s *Server) requestLogger(ctx context.Context) *slog.Logger {
    if logger, ok := ctx.Value(loggerKey).(*slog.Logger); ok {
        return logger
    }
    return s.logger
}
```

The fallback to `s.logger` means handlers still work if called outside of `ServeHTTP`, which can happen in tests that call handler methods directly.

Now every log line from a single request shares the same `request_id`:

```json
{"time":"...","level":"INFO","msg":"job submitted","request_id":"a1b2c3","job_id":"job-abc","command":"echo hello"}
{"time":"...","level":"INFO","msg":"job claimed","request_id":"d4e5f6","job_id":"job-abc"}
```

The submit and the claim have different request IDs because they are different HTTP requests. But if you filter by `job_id=job-abc`, you see the full lifecycle. If you filter by `request_id=a1b2c3`, you see everything that happened during a single API call.

## What We Have Now

Workron's observability stack is complete:

- **Structured logging** with `slog`: JSON output, typed fields, log levels, logger injection with `With` for component identity
- **Prometheus metrics**: counters for lifecycle events, histograms for duration and queue wait, gauges for current state via a custom collector
- **Request tracing**: UUID per request, `X-Request-ID` response header, request-scoped logger via context

Every handler that processes a job now logs a structured event, increments a counter, and carries a request ID. The system does not just work. It explains itself.

Full source: [github.com/lrdinsu/workron](https://github.com/lrdinsu/workron)

---

## References and Further Reading

- [log/slog package](https://pkg.go.dev/log/slog). — Official Go documentation for the structured logging package. The `Handler` and `With` patterns Workron uses for JSON output and component-scoped fields are covered here.
- [Structured Logging with slog (Go blog)](https://go.dev/blog/slog). — The Go team's introduction to slog, covering the design decisions behind the API. Useful for understanding why slog uses alternating key-value pairs rather than a map.
- [Prometheus Go client library](https://github.com/prometheus/client_golang). — The library Workron uses for counters, histograms, and the custom gauge collector. The `promauto` vs manual registration trade-offs are documented in the README.
- [Prometheus metric types](https://prometheus.io/docs/concepts/metric_types/). — Official docs explaining counters, gauges, histograms, and summaries. Helpful for understanding when to use each type.
- [Writing exporters (Prometheus docs)](https://prometheus.io/docs/instrumenting/writing_exporters/). — Prometheus guidelines for metric naming, label usage, and custom collectors. The `workron_` prefix and the `JobGaugeCollector` pattern follow these recommendations.
