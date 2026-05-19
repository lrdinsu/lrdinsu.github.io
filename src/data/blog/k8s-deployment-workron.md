---
author: Lynn Wang
pubDatetime: 2026-05-19T10:00:00Z
title: "Putting Workron on Kubernetes: The Integration Bug Unit Tests Couldn't Reach"
slug: k8s-deployment-workron
featured: true
draft: false
tags:
  - kubernetes
  - go
  - distributed-systems
  - integration-testing
  - engineering-tradeoffs
description: "Deploying Workron to a local Kubernetes cluster as the first real integration test, and the JSON tag mismatch the cluster surfaced in five minutes."
---

By the time the [gang preemption post](/posts/gang-preemption-workron) was written, the drain state machine had a small library of tests behind it. Trigger detection, epoch ordering, retry refund, completion-rule edge cases, the two-pass reaper, force-timeout. All green. What none of those tests exercised was the network. Every scheduler-worker conversation in the test suite happened in-process, with the worker calling Go methods on the scheduler directly. No JSON serialization. No HTTP. No Kubernetes service discovery. No second scheduler replica competing for an advisory lock.

That was the gap I wanted to close. So I put the whole system on local Kubernetes through kind, and within the first end-to-end demo run, the cluster surfaced a bug that had been latent for weeks.

## Table of Contents

## The Deployment Shape

The cluster has six pods across one namespace:

- **Scheduler**: 2-replica `Deployment`, both serve HTTP, both connect to the same Postgres.
- **Worker**: 3-replica `Deployment`, each polls the scheduler `Service` for jobs.
- **Postgres**: `StatefulSet` with one replica and a `PersistentVolumeClaim` so the data survives pod restarts.

The manifests are Kustomize, split into a cloud-agnostic `base/` and a `overlays/local/` that adds the kind-specific bits (NodePort Service, dev Postgres password, `:dev` image tags). The base never references kind or anything localhost-shaped. A future EKS or GKE overlay would swap the in-cluster Postgres for managed RDS via External Secrets, replace the NodePort with an Ingress plus cert-manager, and pull images from a registry like ECR. That's a sibling overlay, not a rewrite of the base.

I picked Kustomize over Helm precisely for this base/overlay shape. Helm gives you templates with values files, which is a layer of indirection I don't need yet. Kustomize gives me the exact split I want: what's universal versus what's environment-specific. It stays declarative the whole way through. If the project ever grows into a distributable artifact, a Helm chart wrapping these manifests is a small wrapper task on top.

Init containers handle ordered startup. The scheduler init container blocks until `pg_isready` returns success. The worker init container blocks until the scheduler `/healthz` returns 200. So `kubectl apply` followed by `kubectl get pods` doesn't show a crash loop while pods wait for dependencies. It shows Init states cleanly resolving in order, then six `Running` pods with zero restarts. `make k8s-up` from cold takes about two minutes.

## Two Probes, Two Different Questions

The scheduler exposes three HTTP endpoints for cluster health: `/health` (existing), `/healthz` (liveness), `/readyz` (readiness). Three feels like a lot, but each answers a different question.

- `/health` is the existing endpoint that returns `{instance_id, uptime, status}` for load-balancer checks. Left alone.
- `/healthz` is the Kubernetes liveness probe target. It returns 200 as long as the HTTP server is up. If this stops returning 200, the kubelet restarts the pod.
- `/readyz` is the Kubernetes readiness probe target. It checks that the scheduler can actually serve traffic, which means it can reach its dependencies. Specifically, it pings the store with a 1-second timeout.

Pinging the store needed a new optional interface:

```go
type Pinger interface {
    Ping(ctx context.Context) error
}
```

`PostgresStore` and `SQLiteStore` implement it. `MemoryStore` doesn't, and `/readyz` treats a non-`Pinger` store as always reachable. The point is that liveness and readiness should not be the same signal. A scheduler whose HTTP loop is healthy but whose database is unreachable should be removed from the Service (readiness failure), not restarted (liveness failure). Restarting wouldn't help (the database is still down) and would just thrash the pod.

That distinction sounds obvious written down. It is not obvious when you sit down to write the probes. The first draft of `/readyz` was just a copy of `/healthz`. Splitting them only happened after I asked myself "what would actually fail-stop differently here?"

## Advisory Locks, Now Visibly Cross-Pod

The two scheduler replicas coordinate through `pg_try_advisory_xact_lock`. One replica holds the gang-admission lock on a given tick and runs the admission cycle. The other tries, fails to acquire, and sits out that tick. Same pattern for the reaper.

That story was true on a single host with two goroutines, but with both replicas now in real pods on a real cluster, it's something I can grep:

```
$ kubectl logs -l app=workron-scheduler --tail=20 -f
[scheduler-0] gang admission tick start
[scheduler-1] gang admission tick start
[scheduler-0] advisory lock acquired key=gang_admission
[scheduler-1] advisory lock not acquired key=gang_admission, skipping
[scheduler-0] gang reserved gang_id=g_…
```

Two replicas, one acquires, one skips. The exact pattern the design called for. The advisory lock releases automatically when the transaction commits, and also when the connection drops. Scheduler-0 dying mid-tick releases the lock through the same code path as normal completion. There is no separate failover code. That was one of the reasons to pick advisory locks over a Raft or etcd-based leader election in the first place; the multi-pod deployment is what makes that argument concrete instead of theoretical.

## The Bug: One Missing JSON Tag

The demo I wanted to run end-to-end in the cluster was the gang-preemption + checkpoint round-trip. Submit a three-task gang. Let workers claim and start. Fail one task. Watch the other two drain. Watch the gang re-admit with the checkpoint data injected. The whole thing should take about 50 seconds.

It did not work. The trigger fail landed cleanly. The scheduler logged "gang drain started." The siblings' next heartbeat carried the preempt action. I could see it in the response body when I curled the heartbeat endpoint directly. The workers received the response. And then nothing. No `/preempted` calls landed. No `/checkpoint` calls landed. After 45 seconds the reaper force-drained them through the timeout path, but the demo's whole point was the worker-driven graceful drain, not the timeout fallback.

The scheduler logs had the answer:

```
preempted ack rejected job_id=… reason=epoch_mismatch want=5 got=0
```

The workers were sending epoch 0. The scheduler was expecting epoch 5.

The worker's heartbeat response decoder looked like this:

```go
type HeartbeatResult struct {
    Action          string
    PreemptionEpoch int64
}

func (c *Client) SendHeartbeat(...) (HeartbeatResult, error) {
    var result HeartbeatResult
    if err := json.NewDecoder(resp.Body).Decode(&result); err != nil {
        return result, err
    }
    return result, nil
}
```

The scheduler emitted snake_case:

```json
{"action": "preempt", "preemption_epoch": 5}
```

Go's `encoding/json` does case-insensitive matching when no struct tag is present. That's why `Action` matched `action` even without a tag. It's also why `PreemptionEpoch` looked like it should match `preemption_epoch`, but the case-insensitive fallback ignores case, not separators. An underscore in the wire format is a hard mismatch against a struct field with no underscore. `PreemptionEpoch` stayed at its zero value. The worker dutifully sent `/preempted?epoch=0`. The scheduler rejected it.

The fix was one line:

```go
type HeartbeatResult struct {
    Action          string `json:"action"`
    PreemptionEpoch int64  `json:"preemption_epoch"`
}
```

Demo started passing on the first try after that. End-to-end runtime is about 50 seconds, all running inside real Kubernetes pods. The workers' resume logs carry the proof:

```
"job picked up" attempt=1 checkpoint_data_present=true checkpoint_data_bytes=101
```

The 101 bytes is the original 94-byte synthetic checkpoint plus base64 padding, exactly what the scheduler is supposed to inject as `CHECKPOINT_DATA`.

![Workron gang-preemption demo running in a kind cluster](https://github.com/lrdinsu/workron/raw/main/docs/k8s-demo.gif)

## Why The Unit Tests Couldn't Find This

This is the part worth dwelling on, because the answer isn't "we didn't write enough tests."

The state machine tests exercise the scheduler's internals directly. They call `PreemptGang` on the store and assert on the resulting state. There is no client, no HTTP, no JSON. The HTTP handler tests call the handlers with a test recorder and verify the response body. They serialize, but they don't go through the worker's client code, so they never round-trip JSON the way production does.

The only test that did exercise the full HTTP round-trip was the worker integration test. That test ran a fake scheduler over HTTP and watched the worker poll, claim, heartbeat, and complete. But the fake scheduler's heartbeat handler always returned an empty `200 OK`, because at the time those tests were written, that was the only response shape. The preempt-carrying heartbeat response was added later, and the integration test was never updated to exercise it. The bug landed in the gap.

What the Kubernetes deployment did was force every code path to be exercised by something other than the code that wrote it. The scheduler that emitted snake_case JSON was the production scheduler. The worker that decoded into PascalCase fields was the production worker. The conversation between them happened over the real network, through a real Service, between real pods. No test substituted for any of it. The bug surfaced within minutes.

That's the argument for integration testing in one specific form: not "test more things," but "make sure something other than the original author's code exercises every wire-level boundary." Mocks and fakes can pass tests that production can't. A `kind` cluster is cheap enough to be a routine part of the development loop, and it eliminates that class of mistake.

## Pod-Kill Drills

A few quick drills to back up the deployment story:

- **Delete both scheduler replicas at once.** About one second of `000` responses from `curl /health` (no replicas behind the Service) before the new pods come up. Then 200 continuously. The reaper and admission cycle restart from scratch with no in-flight state to recover. Since admission ticks every five seconds, the worst-case stall is roughly the tick interval.
- **Delete the Postgres pod.** `/readyz` flips to 503 for about seven seconds (the connection-refused window while the StatefulSet replacement pod comes up and starts accepting connections). Once the new Postgres pod is reachable, `/readyz` returns to 200. During the 503 window, the scheduler is removed from the Service endpoints by the readiness probe, which is exactly what should happen. Clients route to no-one rather than to a broken instance.
- **Delete a worker pod.** The replacement pod registers within a second of starting, picks up jobs from `/jobs/next` on its first poll, sends heartbeats normally. The deleted pod's in-flight jobs hit the heartbeat-timeout path on the scheduler side and get re-released for claim by another worker.

None of these results were surprising in the design sense. They are exactly what the probes and the heartbeat-timeout reaper were supposed to produce. The point is that the actual numbers (one second, seven seconds, instant re-registration) are now measured against a real cluster, not estimated against a design doc.

## What This Changed

The most useful thing the deployment changed wasn't a feature or a bug fix. It was where I now draw the line between "tested" and "validated."

Tested means: I wrote tests for it, they pass, the test exercises the code under test. Validated means: the system does the thing it claims to do, in a deployment shape that resembles how it would actually run. Workron was thoroughly tested before the Kubernetes deployment. It was not validated. The JSON tag bug was the proof of that distinction.

I don't think every project needs to run in Kubernetes to be validated. But every distributed system needs *some* deployment shape where the components talk to each other through their actual wire protocols, not through Go method calls. For Workron, kind was the cheapest version of that. For something else, it might be a docker-compose stack, or two processes on the same laptop talking over localhost. The shape doesn't matter as much as the principle: at least one boundary between subsystems has to be the real one.

The deployment also clarified what the future cloud step looks like. The base/overlay split means moving from kind to EKS is a sibling overlay under `deploy/k8s/overlays/aws/`: a managed Postgres reference via External Secrets, an Ingress with cert-manager, an ECR image pull secret. No Go code changes. No base manifest changes. That's the actual point of having designed it this way: the next step is manifests, not engineering.

Full source: [github.com/lrdinsu/workron](https://github.com/lrdinsu/workron)

---

## References and Further Reading

- [kind (Kubernetes IN Docker)](https://kind.sigs.k8s.io/). — The Kubernetes upstream's own tool for running a local cluster inside Docker containers. Lightest of the local-cluster options; used by Kubernetes CI itself.
- [Kubernetes liveness, readiness, and startup probes](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#container-probes). — The probe model and the distinction between liveness (restart on failure) and readiness (remove from Service on failure). Workron's two probes implement this split directly.
- [Kustomize bases and overlays](https://kubectl.docs.kubernetes.io/guides/introduction/kustomize/). — The model Workron's manifests use. Base is environment-agnostic; overlays carry the differences.
- [Go `encoding/json` and case-insensitive matching](https://pkg.go.dev/encoding/json#Marshal). — The fallback behavior that caused the bug in this post. The matching ignores case differences between letters, not separator differences like underscores or dashes.
- [External Secrets Operator](https://external-secrets.io/). — How a cloud overlay would inject managed database credentials without putting them in the manifests directly. Pre-baked for the future EKS overlay step.
