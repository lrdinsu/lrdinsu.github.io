---
author: Lynn Wang
pubDatetime: 2026-04-28T10:00:00Z
title: "Draining the Gang: Coordinated Preemption with Checkpoint/Resume"
slug: gang-preemption-workron
featured: true
draft: false
tags:
  - distributed-systems
  - go
  - gang-scheduling
  - preemption
  - state-machines
  - engineering-tradeoffs
description: "Closing the stuck-gang gap: a coordinated drain path where one task's failure pulls its siblings down cleanly, with a retry-budget refund and opaque checkpoints that survive the round-trip."
---

At the end of the [gang scheduling post](/posts/gang-scheduling-workron), I called out a stuck state and left it as a problem for next time. A gang of three was reserved on workers A, B, C. Two claimed and started running. The third died before claiming. Thirty seconds later, the scheduler rolled the third task back to `blocked`. Tasks 1 and 2 kept running, waiting for a peer that was never going to join.

The admission cycle looks for gangs where every task is `blocked`. With two tasks running and one blocked, that gang was skipped every tick, forever. Its two live workers would eventually fail on their own through NCCL peer-detection or heartbeat timeout, but the gang-level story was: one partial failure poisoned everything, and nothing in the scheduler pulled the remaining workers down cleanly.

This post covers the fix. A gang task failing (or its worker dying) while siblings are still running now triggers a coordinated drain. The running peers receive a preempt signal on their next heartbeat, send SIGTERM to their child processes, and report back when they have exited. Once the gang is fully drained, it returns to `blocked` as a unit, ready for re-admission.

## Table of Contents

## Why FailGang Wasn't Enough

The existing `FailGang` method only touched `blocked`, `reserved`, and `pending` siblings. Running tasks were deliberately left alone. The reasoning from the last post: "the other workers will detect the failure themselves through a broken NCCL connection, and report failure on their own." And that does happen in theory. In practice, each running worker fails on its own timeline, each `/fail` call lands separately, and there is no atomic moment where the whole gang becomes eligible for re-admission.

During that window, the scheduler has no way to signal "stop what you're doing, the gang is dead." Each worker keeps burning resources and retries on individual failures that are not really individual. By the time everything settles, the gang might have exhausted `MaxRetries` on two tasks even though only one was the actual root cause.

Preemption is the missing half. `FailGang` handles siblings that haven't started yet. `PreemptGang` handles siblings that have.

## The Signal Channel

The first question is how the scheduler tells a running worker to stop. The obvious options:

- A new HTTP endpoint the scheduler calls on the worker. This flips the connection direction and requires every worker to expose an inbound port. Operationally painful, and firewall-hostile.
- A status the worker polls separately. Not really different from the heartbeat loop that already exists.
- The heartbeat response itself.

Workers already poll `POST /jobs/{id}/heartbeat` every five seconds. The response was always an empty 200. Making it a JSON body with an action field is zero new connections, no inbound ports, and the five-second heartbeat cadence sits comfortably inside the fifteen-second grace window I wanted for SIGTERM. So the heartbeat response carries either nothing (business as usual) or:

```json
{"action": "preempt", "preemption_epoch": 7}
```

The epoch matters. It is a counter on the job that increments every time preemption is initiated. When the worker eventually acknowledges "I have stopped" by calling `POST /jobs/{id}/preempted?epoch=7`, the scheduler checks that the epoch matches the current round. A late acknowledgement from an earlier round gets a 409, not a silent acceptance that could roll back a fresh admission.

## The State Machine

I originally considered moving every sibling to `preempting` during drain, to keep the gang homogeneous. But writing out the state semantics made the problem clear. `preempting` should mean "stop this execution attempt." A reserved or pending task has no execution attempt to stop. Moving those into `preempting` would make the state machine harder to reason about: "what does `preempting` mean for a task that never had a running process?"

The honest mapping:

- `running` → `preempting`. There is a process, the worker will be told about it, it may checkpoint, it must ack.
- `reserved` → `blocked`. Clear the worker assignment. No process to signal.
- `pending` → `blocked`. Same reason.
- `blocked` → stays blocked. Already where we want it.
- `done` / `failed` → untouched. (More on this below.)

During drain, a gang is heterogeneous: some tasks in `preempting`, some already `blocked`. That is fine. The admission cycle already requires "every task is blocked" before considering a gang for re-placement. A mid-drain gang doesn't qualify, so admission naturally skips it.

Once the last `preempting` task transitions to `preempted` (via worker ack or force-timeout), a separate reaper pass runs `CompletePreemption`, which normalizes the gang to either `blocked` (retries remain, ready for re-admission) or `failed` (retries exhausted or a sibling is already terminal).

## Retry Budget: The Infinite-Loop Bug I Almost Shipped

This is the part I got most wrong initially, so it deserves its own section.

The original plan was: `PreemptGang` doesn't touch `Attempts`. `CompletePreemption` decrements every drained task's `Attempts` by 1 on the way back to `blocked`, so the natural increment from the next claim nets out. Under this plan, preemption doesn't consume retry budget.

This plan sounds reasonable. But when tracing it:

- Gang of three, task A fails. All three go to `preempting`, `Attempts = 1` each.
- Workers ack, tasks become `preempted`.
- `CompletePreemption` decrements all three: `Attempts = 0`.
- Re-admit, re-claim: `Attempts = 1` each again.
- Task A fails again. Same cycle. `Attempts` for A never exceeds 1.

The trigger task's real failure gets erased every round, so the gang never hits `MaxRetries`, and it leads to infinite retry loop.

The fix is to shift the refund earlier. `PreemptGang` has the trigger's ID in scope while `CompletePreemption` doesn't. By the time it runs, any number of ticks have passed, and no store column remembers who triggered the drain. So I changed it to refund at `PreemptGang` time, and skip the trigger.

```go
// Inside PreemptGang, per task moving running → preempting:
if task.ID != triggerJobID && task.Attempts > 0 {
    task.Attempts--
}
```

The trigger keeps its claim-time `+1`. That `+1` represents the real failure that caused the drain. When `CompletePreemption` eventually checks "does any task have `Attempts >= MaxRetries`?", the trigger's count reflects real failures only, and the gang escalates to `failed` at the right moment.

Retrace the same scenario with the fix:

- Task A fails. All three → `preempting`. A: `Attempts = 1` (kept), B and C: `Attempts = 0` (refunded).
- Ack, complete, back to `blocked`. A: 1, B: 0, C: 0.
- Re-claim: A: 2, B: 1, C: 1.
- A fails again. A: 2 (kept), B: 0, C: 0.
- Re-claim: A: 3, B: 1, C: 1.
- A fails again. A's `Attempts == MaxRetries`. Gang verdict: `failed`.

Three real failures, three retries, terminal. The system now converges.

The broader takeaway: "refund everyone" and "refund siblings only" look similar on paper but have opposite long-run behavior. The trigger's attempt counter is the signal, and erasing it removes the mechanism that distinguishes real failures from scheduler-driven tear-downs.

## The Completion Rule: Option B

When `CompletePreemption` runs after drain, it decides: does the gang go to `blocked` (try again) or `failed` (give up)?

Initial instinct: "if any task still has retries left, block the gang; else fail it." That is too permissive. A gang where two tasks have `MaxRetries = 3, Attempts = 0` and one task has `Attempts = 3` should fail. Retrying a gang with a permanently exhausted member will never succeed because that member is already out of tries.

The inverse is stricter and correct: if any task has `Attempts >= MaxRetries`, the gang fails. Otherwise it blocks. With the refund landing in `PreemptGang`, this rule correctly reflects real failures only.

There is a second escalation clause, for a case I almost overlooked: what if one task finished successfully before a sibling failed? Task 0 reports `/done` at the same moment task 1's worker dies. The drain fires on tasks 1 and 2. What happens to task 0?

If it stays `done` and the gang returns to `blocked`, the re-admission tries to re-run all three tasks, overwriting the already-succeeded work. That is wrong. If it stays `done` and tasks 1 and 2 become `blocked`, the gang has mixed statuses (done, blocked, blocked) and will never re-admit because `findBlockedGangs` requires all-blocked.

So: if any task is already `done` or `failed` at drain completion, the whole gang goes to `failed`. The succeeded task keeps its `done` on the row (its outcome is preserved for the record), but the gang-level verdict is "cannot be cleanly restart-all'd." I considered routing to `blocked` and re-running everything, but that overwrites legitimate completed work, and restart-all semantics don't really apply once part of the work has finished. Failing honestly is cleaner than silently discarding progress.

## SIGTERM, Grace, SIGKILL

Workers had been using `exec.CommandContext`, which sends SIGKILL when the context is canceled. For graceful shutdown you want SIGTERM first so the process can checkpoint, clean up, or just print a farewell line.

The executor grew a `stop <-chan struct{}` parameter. When the channel closes:

1. Send SIGTERM to the child via `cmd.Process.Signal`.
2. Wait up to `preemptGrace` (15 seconds) for the process to exit on its own.
3. If it's still running, hard-kill via `cmd.Process.Kill`.

Context cancellation still exists as the hard shutdown path, it triggers SIGKILL immediately via `exec.CommandContext`'s default cancel behavior. SIGTERM with grace is the graceful path; ctx cancellation is the "we're shutting down now, forget grace" path. Two different signals for two different situations.

One detail that saved me: if `stop` is nil, reads from it block forever (nil channel semantics), so the select never fires the SIGTERM branch. This means any caller that doesn't care about preemption can pass `nil` and get the old behavior. The executor's signature is backward compatible in spirit, even though the Go signature technically changed.

A test guards both paths:

```go
// Normal case: process exits on SIGTERM within grace.
err := e.Execute(ctx, "sleep 30", nil, stop)
// A goroutine closed stop at t=100ms; assertion: elapsed < 5s.

// Stubborn process: trap TERM and ignore it.
err := e.Execute(ctx, "sh -c 'trap \"\" TERM; sleep 30'", nil, stop)
// stop closes, then ctx cancels 100ms later; SIGKILL fires; elapsed < 5s.
```

The `trap '' TERM` trick is a minimal reproduction of a process that refuses to shut down cleanly. Without the hard-kill backstop, that test would hang until the 30-second sleep finished.

## Idempotent Signaling on the Worker

Once a task is in `preempting`, every subsequent heartbeat (every 5 seconds) gets the preempt action in the response. The worker receives it repeatedly until the process exits and the ack lands. If the worker naively closed its `stop` channel each time, that is a panic on the second heartbeat.

`sync.Once` guards the close:

```go
preemptOnce.Do(func() { close(preemptCh) })
```

Repeated preempt heartbeats are no-ops after the first. The epoch is also recorded once, under a mutex, so late heartbeats with the same epoch don't thrash the field. When the executor returns, the worker checks whether `preemptCh` was ever closed and routes to `ReportPreempted` instead of the usual `/done` / `/fail` paths.

## Checkpoint: Opaque, Epoch-Guarded, Demo-Scale

Checkpointing rides on the same preempt path. While a job is in `preempting`, the worker can `POST /jobs/{id}/checkpoint?epoch=N` with raw bytes. The scheduler stores them on the job row and surfaces them as `CHECKPOINT_DATA` (base64) on the next claim.

A few deliberate constraints:

- The bytes are opaque. The scheduler does not parse them. What goes in comes out, modulo Postgres JSONB's whitespace canonicalization (documented as a test-time compare-semantically thing).
- Saves are valid only while the job is in `preempting`. I considered allowing opportunistic saves during normal `running` state and decided against it, because it enlarges the state space for a minor win, and the preempt window is when checkpoints actually matter.
- The epoch must match. A stale save from an earlier drain round gets a 409. This is the same epoch-guarding pattern as `/preempted`.
- Base64 in `CHECKPOINT_DATA` is a binary-safety measure for env-var transport, not a design statement. Anything larger than "demo-scale" should go through a side channel (S3, a shared volume), not through an environment variable. The plan doc and the field's help text both call this out.

The test flow mirrors what a real ML workload would look like: worker receives preempt, writes its last-step checkpoint, posts it, process exits. New worker claims the task after re-admission, sees `CHECKPOINT_DATA` in its env, base64-decodes it, resumes from step N instead of step 0.

## The Drain Completer

Somebody has to notice when a drain has finished. The worker calling `/preempted` only transitions its own task, the gang-level `blocked` transition is deferred. The reaper got a new pass, `completeDrainedPreemptions`, that runs after `reap` on every tick.

Two passes, ordered:

1. **Force-timeout.** Any task stuck in `preempting` past `preemptTimeout` (45 seconds, which is 15s SIGTERM grace + 30s slack) gets `ForceDrainPreempting` applied. This covers workers that died mid-drain and will never ack. The bypass-the-epoch is deliberate: we don't care about epoch matching for a timeout path, we just want the stuck task moved out of `preempting` so it stops blocking gang completion.
2. **Gang completion.** Group jobs by `gang_id`. For each gang with any task in `preempting`/`preempted` and no task in `running`/`preempting`, call `CompletePreemption`. The store picks `blocked` or `failed` based on the Attempts and terminal-state checks.

The two passes are in order because pass 1 can change what pass 2 sees. If pass 1 moves the last stuck task to `preempted`, pass 2 sees zero in-flight tasks and completes the gang on the same tick. No "wait until next tick" hop.

I also snapshot `DrainStartedAt` from the gang's preempted tasks before calling `CompletePreemption`, because that method clears the field. That snapshot is what the drain-latency histogram observes once completion succeeds.

## Observability

The drain path emits four metrics:

- `workron_gangs_preempted_total` — one increment per `PreemptGang` that entered drain.
- `workron_gang_preemptions_force_drained_total` — per stuck task the reaper force-drained.
- `workron_gang_preemptions_completed_total{outcome}` — labeled `blocked` or `failed`.
- `workron_gang_preemption_drain_seconds` — histogram of drain-start to completion time.

And four log lines reconstruct the whole story for a given gang, all carrying `gang_id` and `preemption_epoch`:

```
gang drain started       gang_id=… trigger_job_id=… preemption_epoch=1 transitioned=3
job preempted            job_id=… preemption_epoch=1          (×3)
gang drain completed     gang_id=… preemption_epoch=1 outcome=blocked
gang reserved            gang_id=… gang_size=3
```

Grepping for one `gang_id` tells you exactly what happened and when. That was a deliberate choice during the observability work: epoch-labeled log lines let me tell two different drain rounds apart for the same gang, which matters when something weird happens and I'm tracing through a failure.

## What I Got Wrong the First Time

A few things I got wrong in the first draft:

**Non-running siblings in `preempting`.** The initial version put every sibling into `preempting` at drain start, uniform state, trivial completion check. But `preempting` carries a specific contract: there is a live process, a worker is waiting for the signal, a worker will ack. A reserved or pending task has none of that. Once I wrote out what the worker was supposed to do with a preempt signal on a task it hadn't claimed, the problem was obvious. Non-running siblings go straight to `blocked`. The gang is heterogeneous during drain. That is fine.

**Retry refund in `CompletePreemption`.** The first plan decremented every drained task's `Attempts` on the way back to `blocked`. Preemption is not a real failure; it should not burn retry budget. That logic holds for siblings. It does not hold for the trigger. Trace three rounds: task A fails → drain → `CompletePreemption` decrements everyone back to zero → re-admit → A fails again. A's `Attempts` never exceeds one. `MaxRetries` is unreachable. Infinite loop. Moving the refund into `PreemptGang`, where the trigger's ID is in scope, is what fixes it. The trigger's increment stays. Siblings are cleared. That single kept bump is the whole signal that distinguishes a real failure from a scheduler-driven tear-down.

**Checkpoints during `running`.** The first interface allowed saves any time a job was `running`. It seemed like a useful generalization, as periodic saves, not just drain-window saves. But it raises questions: which checkpoint surfaces on re-claim if there are multiple? What if the job saves every minute and the row bloats? Restricting saves to `preempting` avoids all of that. The window where a checkpoint actually matters is exactly the window when a job is being told to stop. That is when you want to save. Anything else is out of scope until there is a concrete reason to add it.

**`buildGangEnv` scoped to gang claims.** The env-building function started as a gang-specific helper because checkpoint injection was gang-specific. Then it became clear: a non-gang job can also have a checkpoint from a previous preemption round if it later gets gang-scheduled and preempted. Renamed to `buildJobEnv`, called on every claim, injects `CHECKPOINT_DATA` whenever there are stored bytes. Straightforward fix once the assumption was noticed.

## What's Next

The preemption infrastructure is gang-internal only. That is deliberate scope management, the master plan has priority-based preemption and cross-queue quota reclaim as separate features that layer on top. Those will use the same `preempting` state, the same heartbeat signal, the same checkpoint contract. What changes is the triggering logic: "my gang needs 4 GPUs, 3 are free, 1 is running a lower-priority job, preempt it." That trigger layer doesn't exist yet.

The executor's SIGTERM + grace + SIGKILL pattern is also reusable for plain job cancellation. A cancel API would plug into the same executor stop channel and reuse the same grace window.

Full source: [github.com/lrdinsu/workron](https://github.com/lrdinsu/workron)

---

## References and Further Reading

- [Kubernetes Pod Termination Lifecycle](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#pod-termination). — Kubernetes uses the same SIGTERM → grace → SIGKILL pattern with a configurable `terminationGracePeriodSeconds`. Workron's 15-second grace mirrors the Kubernetes default ergonomics.
- [Kueue Workload Suspension](https://kueue.sigs.k8s.io/docs/concepts/workload/#suspended). — Kueue's model for draining a workload and returning it to the queue. Similar trigger-and-drain semantics, different implementation (suspending pods rather than signaling workers).
- [PyTorch Lightning Checkpointing](https://lightning.ai/docs/pytorch/stable/common/checkpointing_basic.html). — Shows what a checkpoint round-trip actually looks like from the job side: save on signal, load on restart, handle the "no checkpoint exists" first-run case. Workron stores the opaque bytes; the job process does the save/load.
- [`exec.Cmd.Cancel` and `WaitDelay` in Go 1.20+](https://pkg.go.dev/os/exec#Cmd.Cancel). — The idiomatic Go handle for "cancel sends X signal, then hard-kill after Y seconds." The executor doesn't use `WaitDelay` directly for reasons of explicit goroutine structure, but the pattern is the same.
- [`pg_try_advisory_xact_lock` (PostgreSQL)](https://www.postgresql.org/docs/current/functions-admin.html#FUNCTIONS-ADVISORY-LOCKS). — Used unchanged from the earlier admission-cycle post. The drain completer runs inside the reaper tick, which already holds the reaper advisory lock in multi-instance mode, so no new coordination layer was needed.
