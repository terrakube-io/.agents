---
name: terrakube-run-triggers
description: Architecture, domain model, static cycle detection, state-change qualification, transaction isolation, and UI conventions for Terrakube Run Triggers and Workspace Dependency Graphs.
---

# Terrakube Workspace Run Triggers & Dependency Graph Guide

This skill documents the architecture, database schema, security checks, cycle detection, reactive dispatch engine, and UI conventions for Terrakube Run Triggers (`io.terrakube.api.rs.workspace.trigger` and `io.terrakube.api.plugin.scheduler.trigger`).

---

## 1. High-Level Architecture & Lifecycle

Run triggers allow downstream workspaces to automatically enqueue runs when an upstream workspace successfully applies state changes. Workspaces declare dependencies on upstream source workspaces, creating a decentralized, composable DAG without requiring monolithic pipeline definitions.

```
Upstream Job (Workspace A)
       |
       v
Executor completes Apply / Destroy
       |
       v
JobReconciliationService.reconcile()
       | Commits status = 'completed'
       |
       v (afterCommit)
RunTriggerDispatchService.dispatchFor(jobId)
       |
       +---> 1. Validate enabled & state changed (serial / md5 changed)
       |
       +---> 2. Enforce runtime hop limit (cascadeDepth < maxCascadeDepth)
       |
       +---> 3. Query enabled triggers: findEnabledBySourceWorkspaceId(wsA)
       |
       +---> 4. For each downstream trigger (Workspace B):
                 |
                 +-> RunTriggerJobWriter.persist() [@Transactional(REQUIRES_NEW)]
                 |   - Created as JobVia.RUN_TRIGGER
                 |   - triggeredByJobId = Job A.id, cascadeDepth = A.depth + 1
                 |   - Template: Trigger override -> Workspace defaultTemplate
                 |
                 +-> jobNotificationTrigger.notifyStatusChanged(jobB)
                 +-> scheduleJobService.createJobContext(jobB) [Quartz]
```

---

## 2. Data Model & Database Migration

### Entity: `WorkspaceRunTrigger` (`io.terrakube.api.rs.workspace.trigger.WorkspaceRunTrigger`)
* **Table**: `workspace_run_trigger`
* **Primary Key**: `UUID` (`varchar(36)`)
* **Associations**:
  - `sourceWorkspace`: Upstream workspace (ManyToOne, lazy).
  - `destinationWorkspace`: Downstream workspace (ManyToOne, lazy).
  - `template`: Optional override template (ManyToOne, lazy).
  - `organization`: Denormalized owner (ManyToOne, lazy, `@Exclude`, derived in `@PrePersist` / `@PreUpdate`).
* **Constraints**:
  - Unique constraint: `uk_run_trigger_source_dest` on `(source_workspace_id, destination_workspace_id)`.
  - Check constraint: `chk_run_trigger_no_self` (`source_workspace_id != destination_workspace_id`).
* **Indexes**:
  - `idx_run_trigger_source_enabled` on `(source_workspace_id, enabled)` for fast dispatch.
  - `idx_run_trigger_destination` on `destination_workspace_id` for UI inbound listings.
  - `idx_run_trigger_organization` on `organization_id` for cycle validation.

### Job Lineage Columns (`Job.java`):
* `triggered_by_job_id` (`Integer`): Upstream job ID. Guarded by `@CreatePermission(expression = "user is a super service")`.
* `cascade_depth` (`int`, default 0): Current chain depth.

---

## 3. Security & Permission Architecture

| Operation | Elide Permission Check | Rule Semantics |
|---|---|---|
| **CREATE** | `team manage workspace trigger` | Requires `canManage` on destination AND `canView` on source (`TeamManageWorkspaceTrigger`). Validates structural invariants (no self-loops, same org, template in same org). |
| **UPDATE** | `team manage workspace trigger` | Endpoints `sourceWorkspace` and `destinationWorkspace` are immutable (`user is a superuser`). Only `enabled` and `template` may be patched. |
| **DELETE** | `team delete workspace trigger` | Requires `canManage` on destination **only** (`TeamDeleteWorkspaceTrigger`). Upstream permission is NOT required so owners can always detach triggers. |
| **READ** | `team view workspace trigger` | Open to any member of the owning organization (`TeamViewWorkspaceTrigger`). |

> [!IMPORTANT]
> **Inverse Collections on Workspace**:
> `Workspace.runTriggers` and `Workspace.sourceRunTriggers` must use `@UpdatePermission(expression = "team view workspace OR team project limited view workspace OR team limited view workspace")` because Elide maintains bidirectional links on trigger creation. A strict manage check here would cause ordinary users to fail creation with `UpdatePermission Denied`.
> Do **NOT** use `orphanRemoval = true` or `CascadeType.ALL` on these collections; deletion must only go through the `WorkspaceRunTrigger` endpoint.

> [!CAUTION]
> **Tenant Security & Field Exclusion**:
> The `organization` relationship field on `WorkspaceRunTrigger` is annotated with `@Exclude` to prevent API callers from spoofing ownership. The organization is derived automatically from the destination workspace via JPA `@PrePersist` / `@PreUpdate`.

---

## 4. Static Cycle Detection & Graph Validation

Implemented in `WorkspaceGraphValidationService.java` and triggered by `WorkspaceRunTriggerHook` at `LifeCycleHookBinding.TransactionPhase.PRECOMMIT`:

1. **Self-Trigger Check**: Rejects `sourceId.equals(destinationId)` with `CyclicDependencyException`.
2. **Cycle Detection (`validateAcyclic`)**:
   - Queries `findEdgesByOrganizationId` returning lightweight `TriggerEdge` interface projections (never loading full entity trees).
   - **Crucial**: Validates **both enabled and disabled** edges to prevent dormant cycle re-activation (i.e. defining A -> B disabled and B -> A enabled, then enabling A -> B).
   - Excludes the current edge ID on updates so modified edges validate against their prospective shape.
   - Runs iterative DFS from `destinationId` to `sourceId`. If a path exists, throws `CyclicDependencyException` detailing the loop path (e.g. `A -> B -> A`).
3. **Fan-Out Limit (`validateFanOutLimit`)**:
   - Locks source workspace row via `workspaceRepository.lockForUpdate(sourceId)` to eliminate creation races.
   - Enforces `maxOutboundTriggersPerWorkspace` (`io.terrakube.run-trigger.max-outbound-triggers-per-workspace`, default 20).

---

## 5. Reactive Dispatch Engine

Implemented in `RunTriggerDispatchService.java` and `RunTriggerJobWriter.java`:

1. **Dispatch Trigger Point**:
   - Invoked exclusively in `JobReconciliationService.reconcile` post-commit:
     ```java
     if (target == JobStatus.completed) {
         dispatchRunTriggersAfterCommit(jobId);
     }
     ```
   - Invocation uses Spring's `TransactionSynchronizationManager.afterCommit` to ensure the upstream job is durably committed before dispatch starts.

2. **State-Change Qualification (`changedState`)**:
   - Checks if a step with `terraformApply`, `terraformDestroy`, or `customScripts` completed.
   - For Terraform steps, compares `History` against baseline:
     ```java
     boolean serialChanged = jobHistory.get().getSerial() > baseline.get().getSerial();
     boolean md5Changed = !Objects.equals(jobHistory.get().getMd5(), "0") && !Objects.equals(jobHistory.get().getMd5(), baseline.get().getMd5());
     ```
   - If serial did not increase and MD5 did not change (0 resource changes), downstream dispatch is skipped.
   - Falls back to `true` (and increments `runTriggerDispatchMetrics.stateIdentityFallback()`) if history records are absent or legacy.

3. **Transaction Boundary & Scheduling**:
   - `RunTriggerJobWriter.persist(...)` runs with `@Transactional(propagation = Propagation.REQUIRES_NEW)`.
   - The job row is committed **before** calling `scheduleJobService.createJobContext(savedJob)`. This prevents Quartz workers on separate database connections from racing and failing to find an uncommitted job row.
   - Outer dispatch loop catch blocks isolate dependents so that one failed job insertion does not roll back sibling downstream jobs.
   - Triggered jobs are created with status `pending`, `via = JobVia.RUN_TRIGGER`, and enter Terrakube's standard two-tier queue (workspace serialization + global FIFO).

4. **Runtime Hop Limit (`cascadeDepth`)**:
   - Each job carries `cascadeDepth = sourceJob.getCascadeDepth() + 1`.
   - Capped at `properties.getMaxCascadeDepth()` (default 10) to prevent runaway cascade chains in edge-case race conditions.

---

## 6. Frontend Conventions (`ui/src/domain/Workspaces/RunTriggers.tsx`)

1. **Combined Directional Fetch**:
   Query both inbound and outbound triggers in a single roundtrip using Elide's comma-separated OR syntax:
   ```typescript
   axiosInstance.get("runTrigger", {
     params: {
       include: "sourceWorkspace,destinationWorkspace,template",
       "filter[runTrigger]": `sourceWorkspace.id==${workspaceId},destinationWorkspace.id==${workspaceId}`,
     },
   });
   ```
2. **Bodyless DELETE**:
   When deleting triggers, omit the `Content-Type` header:
   ```typescript
   // DO NOT pass { headers: { "Content-Type": "application/vnd.api+json" } }
   axiosInstance.delete(`runTrigger/${id}`);
   ```
   Elide responds with HTTP 400 if a `Content-Type` header is supplied on a request without a body.
3. **Job Lineage Display (`ui/src/domain/Jobs/Details.tsx`)**:
   Reads `triggeredByJobId` and `cascadeDepth`, fetching the upstream job to display a navigation banner linking back to the triggering run.

---

## 7. Key File References & Configuration Properties

| Component | File | Description |
|---|---|---|
| `WorkspaceRunTrigger` | `terrakube/api/.../rs/workspace/trigger/WorkspaceRunTrigger.java` | Elide entity with asymmetric permissions and immutable endpoints. |
| `WorkspaceGraphValidationService` | `terrakube/api/.../plugin/scheduler/trigger/WorkspaceGraphValidationService.java` | Iterative DFS cycle detection and fan-out checking. |
| `WorkspaceRunTriggerHook` | `terrakube/api/.../rs/hooks/trigger/WorkspaceRunTriggerHook.java` | Pre-commit lifecycle hook executing graph validation. |
| `RunTriggerDispatchService` | `terrakube/api/.../plugin/scheduler/trigger/RunTriggerDispatchService.java` | State change qualification, fan-out orchestration, cascade depth checks. |
| `RunTriggerJobWriter` | `terrakube/api/.../plugin/scheduler/trigger/RunTriggerJobWriter.java` | `REQUIRES_NEW` transaction boundary for per-dependent job persistence. |
| `RunTriggers.tsx` | `terrakube/ui/src/domain/Workspaces/RunTriggers.tsx` | Workspace tab managing incoming and outgoing run triggers. |

### Configuration Properties:
- `io.terrakube.run-trigger.enabled` (default: `true`): Global kill switch for reactive dispatch.
- `io.terrakube.run-trigger.max-cascade-depth` (default: `10`): Maximum trigger chain depth.
- `io.terrakube.run-trigger.max-outbound-triggers-per-workspace` (default: `20`): Maximum enabled outbound triggers per workspace.
