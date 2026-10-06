---
name: terrakube-job-scheduler
description: Architecture, execution lifecycle, two-tier queueing, global FIFO admission, and Redis distributed locking for Terrakube Job processing.
---

# Terrakube Job Scheduler, Queue & Locking Guide

This skill documents how Terrakube processes infrastructure jobs, manages its two-tier queue, enforces global FIFO dispatch ordering, and uses Redis distributed locks across clustered API replicas.

---

## 1. High-Level Architecture & Lifecycle

Terrakube orchestrates jobs through an asynchronous, distributed interaction between `terrakube-api`, PostgreSQL, Quartz Scheduler, Redis, and `terrakube-executor`.

### Sequence Flow
```
User / Webhook / CLI
       | (POST /api/v1/job)
       v
  JobManageHook (Elide CREATE)
       |
       v
ScheduleJobService.createJobContext
       | Creates 30s recurring Quartz trigger: TerrakubeV2_Job_<jobId>
       v
ScheduleJob.execute()
       |
       +---> 1. Acquire Redis Lock (job-execution-lock:<jobId>, TTL 90s)
       |
       +---> 2. Per-Workspace Blocker Check (id < candidateJobId on same workspace)
       |
       +---> 3. Global FIFO Admission Check (isJobNextInDispatchOrderExecutable)
       |
       +---> 4. Dispatch Step via ExecutorService (POST /api/v1/terraform-rs or K8s Job)
       |         Executor responds: 202 Accepted (queued internally)
       |
       +---> 5. Update Job Status -> "queue"
       |
       +---> 6. wakeNextDispatchableJob() (Wakes next eligible FIFO job)
       |
       +---> 7. Release Redis Lock (DEL job-execution-lock:<jobId>)
       |
Executor Execution (runs @Async in executor pod)
       |
       +---> Mark Job "running" via API
       +---> Run Terraform / OpenTofu / Script
       +---> Update step and job status ("pending", "waitingApproval", or "completed")
       +---> JobReconciliationService.reconcile()
       |         +---> deleteTriggerAfterCommit() (Cleans Quartz trigger, schedules next FIFO candidate)
       |         +---> dispatchRunTriggersAfterCommit() (RunTriggerDispatchService.dispatchFor)
       +---> Release ExecutorCapacityGate
       +---> Publish "terrakube:executor-available" to Redis (Doorbell)
       |
ExecutorAvailabilityListener (API)
       | Catches Redis doorbell
       v
ScheduleJobService.createJobContextNow (Wakes front of FIFO queue immediately)
```

---

## 2. Distributed Execution Lock (`Redis`)

### The Problem
In Quartz Scheduler, `@DisallowConcurrentExecution` on `ScheduleJob` only prevents concurrent executions of the **same** `JobDetail` key. 
Because Terrakube frequently spawns ad-hoc one-shot triggers via `ScheduleJobService.createJobContextNow` (which creates dynamically suffixed keys `TerrakubeV2_Job_<jobId>_<uuid>`), Quartz would otherwise run recurring ticks and event-driven triggers concurrently across multiple API replicas for the same job.

### Lock Implementation
Implemented in `ScheduleJob.java` and `JobReconciliationSweep.java`:
- **Key**: `job-execution-lock:<jobId>`
- **TTL**: `90 seconds` (`Duration.ofSeconds(90)`), comfortably exceeding the 70s worst-case HTTP connect (10s) and response (60s) timeouts of `PersistentExecutorService`.
- **Atomic Primitive**:
  ```java
  Boolean acquired = redisTemplate.opsForValue()
          .setIfAbsent(EXECUTION_LOCK_PREFIX + jobId, "1", EXECUTION_LOCK_TTL);
  ```
- **Lifecycle & Release Scope**:
  - The lock wraps `doRunExecution`.
  - Individual JPA writes commit in their own transactions so all state updates are committed before the lock releases.
  - **CRITICAL**: The lock is released **immediately** in the `finally` block once the executor acknowledges dispatch with HTTP `202 Accepted`. **It is NEVER held during the long-running execution of Terraform.**
- **Fail-Closed Safety**:
  - If Redis is unreachable, lock acquisition catches `DataAccessException` and returns `false`.
  - Execution skips the tick and retries on the next 30s cycle, preventing unprotected parallel execution.

---

## 3. Two-Tier Queue Mechanism

### Tier 1: Per-Workspace Queue (Workspace Serialization)
Within a single workspace, only one job may modify infrastructure state at a time.
In `ScheduleJob.doRunExecution`:
- Terrakube queries `jobRepository.findByWorkspaceAndStatusNotInAndIdLessThan(job.getWorkspace(), TERMINAL_STATUSES, job.getId())`.
- If an older job in the workspace is active (`pending`, `approved`, `queue`, `running`, `waitingApproval`), the candidate job backs off.
- **Plan-Only Bypass**: Pure plan-only jobs (`isTemplatePlanOnly`) are permitted to run concurrently unless an active `apply` or `destroy` is running on the workspace.

### Tier 2: Executor Capacity Gating
- Persistent executor pods enforce single-tenant concurrency via `ExecutorCapacityGate`.
- If a pod is busy, it sets its readiness probe to `REFUSING_TRAFFIC` and returns `503 SERVICE_UNAVAILABLE`.
- The API's `PersistentExecutorService` catches the 503 or connection timeout as an `ExecutorUnavailableException`. The scheduler leaves the job in `pending`/`approved` to retry later without failing the run.

---

## 4. Global FIFO Queue Implementation

### Monotonic Ordering Key
The primary key `id` is a sequential integer generated by PostgreSQL. Lower `id` values represent older, higher-priority jobs.

### SQL Admission Check
Before dispatching, `ScheduleJob` verifies:
```java
boolean oldest = reconciliationProperties.isAdmissionGuardEnabled()
        ? jobRepository.isJobNextInDispatchOrderExecutable(job.getId())
        : jobRepository.isJobNextInDispatchOrder(job.getId());
```

SQL implementation in `JobRepository.java`:
```sql
SELECT NOT EXISTS (
  SELECT 1 FROM job earlier
  WHERE earlier.id < :candidateJobId
    AND earlier.status IN ('pending', 'approved')
    AND earlier.deleted = false
    -- Invariant: Candidate must have executable work remaining
    AND ( NOT EXISTS (SELECT 1 FROM step s WHERE s.job_id = earlier.id)
          OR EXISTS (SELECT 1 FROM step s WHERE s.job_id = earlier.id AND s.status = 'pending') )
    -- Invariant: Candidate must not be blocked by another job in its own workspace
    AND NOT EXISTS (
      SELECT 1 FROM job blocker
      WHERE blocker.workspace_id = earlier.workspace_id
        AND blocker.id < earlier.id
        AND blocker.deleted = false
        AND blocker.status NOT IN ('failed','completed','rejected','cancelled','noChanges')
    )
)
```

### Invariants of the Guarded Admission Query:
1. **FIFO Fair Play**: A newer job cannot jump ahead of an older eligible job.
2. **Workspace Independence**: An earlier job blocked in its *own* workspace does not block jobs in other workspaces from claiming idle executor capacity.
3. **Guarded Liveness (No Zombie Blockers)**: An earlier job only counts as a blocker if it actually has work left (`NOT EXISTS (steps)` or `EXISTS (step.status = 'pending')`). If an earlier job is in `approved` or `pending` but has zero pending steps, it is excluded so it cannot cause head-of-line blocking.

### Low-Latency Doorbell (Redis Pub/Sub)
1. When an executor pod completes a run, it publishes an empty payload to `terrakube:executor-available`.
2. `ExecutorAvailabilityListener` in the API listens to this channel, finds the current FIFO head via `jobRepository.findNextDispatchableExecutableJobId()`, and triggers `createJobContextNow(job)`.
3. The FIFO head job is scheduled in milliseconds (< 5ms) instead of waiting for the next 30-second Quartz tick.

---

## 5. Code References

| Component | File | Description |
|---|---|---|
| `ScheduleJob` | `terrakube/api/.../plugin/scheduler/ScheduleJob.java` | Core scheduler, lock acquisition, blocker checks, dispatch. |
| `JobRepository` | `terrakube/api/.../repository/JobRepository.java` | Guarded SQL queries for FIFO admission. |
| `ScheduleJobService` | `terrakube/api/.../plugin/scheduler/ScheduleJobService.java` | Manages 30s recurring triggers and ad-hoc one-shot triggers. |
| `ExecutorAvailabilityListener` | `terrakube/api/.../plugin/scheduler/ExecutorAvailabilityListener.java` | Redis doorbell listener to trigger FIFO head immediately. |
| `PersistentExecutorService` | `terrakube/api/.../executor/persistent/PersistentExecutorService.java` | REST client to persistent executor with 70s timeout. |
| `JobReconciliationService` | `terrakube/api/.../reconciliation/JobReconciliationService.java` | Reconciliation to completed, Quartz cleanup, post-commit trigger dispatch. |
| `JobReconciliationSweep` | `terrakube/api/.../reconciliation/JobReconciliationSweep.java` | Background reconciliation for zero-pending jobs and Quartz triggers. |
| `RunTriggerDispatchService` | `terrakube/api/.../plugin/scheduler/trigger/RunTriggerDispatchService.java` | Reactive fan-out to downstream dependent workspaces on completed apply. |
| `RunTriggerJobWriter` | `terrakube/api/.../plugin/scheduler/trigger/RunTriggerJobWriter.java` | Isolated `REQUIRES_NEW` job creation for triggered runs. |
| `ExecutorJobImpl` | `terrakube/executor/.../executor/ExecutorJobImpl.java` | Executor runner, status callbacks, and doorbell publisher. |
