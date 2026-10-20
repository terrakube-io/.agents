---
name: terrakube-structured-output
description: Comprehensive architecture, lifecycle, sensitivity protection, state resolution, and API streaming reference for Terrakube Executor structured output (Plan & Apply).
---

# Terrakube Structured Output Guide (Plan & Apply)

Terrakube Executor processes Terraform / OpenTofu execution streams in real-time and post-hoc to generate structured JSON diffs, live status updates, diagnostics, and sanitized outputs for the Terrakube Web UI, API clients, and PR comment generators.

---

## 1. High-Level Architecture Overview

```
                      +---------------------------------------+
                      |       Terraform / OpenTofu Process    |
                      +-------------------+-------------------+
                                          |
                        Streams JSON lines over stdout
                                          v
                      +-------------------+-------------------+
                      |   Event Parsers (Live Streaming)     |
                      | - PlanJsonEventParser                 |
                      | - ApplyJsonEventParser                |
                      +-------------------+-------------------+
                                          |
                         Publishes progress to Context API
                                          v
                      +-------------------+-------------------+
                      |      Post-Operation Reconciliation    |
                      | - Plan: `show -json` (diff & plan)    |
                      | - Apply: `show -json` (state resolve) |
                      +-------------------+-------------------+
                                          |
                      Sanitizes sensitive values & propagates
                                          v
                      +-------------------+-------------------+
                      |      Context Storage & API (SSE)      |
                      | - /context/v1/{jobId}                 |
                      | - /context/v1/{jobId}/stream          |
                      +---------------------------------------+
```

---

## 2. Plan Structured Output Lifecycle (`PlanStructuredOutputService`)

### 1. Live Streaming (`PlanJsonEventParser`)
- When running `terraform plan -json`, stdout lines are parsed in real time.
- Emits resource-level changes (`planned_change`) and diagnostic events (`diagnostic`).
- Publishes intermediate progress to `/context/v1/{jobId}` via `publishPlanProgress()`.

### 2. Post-Plan Summary Reconciliation (`publishPlanSummary`)
- Executes `terraform show -json <plan_file>` via `TerraformClient.showPlanJson()`.
- Extracts `resource_changes` including:
  - `before` / `after` attribute state
  - `before_sensitive` / `after_sensitive` sensitivity metadata
  - `after_unknown` flags
  - Computed `changedSensitive` paths for UI change detection
- **Human-Readable Diff (`getPlanAsHumanText`)**: Executes `terraform show <plan_file>` to capture terminal diff output needed by `PrCommentService`.

### 3. Plan-Time Sensitivity Protection
- Redacts sensitive attributes from `before` and `after` using `sanitizeSensitiveValues()`.
- Normalizes dynamic resource sensitivities (e.g. `terraform_data.output` inherits `input` sensitivity via `normalizeResourceSensitivities()`).
- Posts updated `planStructuredOutput`, `jobDiagnostics`, and `terrakubeUI` marker (`<div data-terrakube-structured-plan="true"></div>`) to Context API.

---

## 3. Apply Structured Output Lifecycle (`ApplyStructuredOutputService`)

### 1. Plan Seeding (`seedFromPlan`)
- Before `terraform apply` begins, copies `planStructuredOutput` changes into `applyStructuredOutput`.
- Marks all planned resources with initial status `"pending"` and `elapsedSeconds: 0`.

### 2. Live Streaming (`ApplyJsonEventParser`)
- Parses JSON lines from `terraform apply -json`:
  - `apply_start` -> sets resource status to `"applying"`.
  - `apply_progress` -> updates `elapsedSeconds`.
  - `apply_complete` -> sets resource status to `"applied"`.
  - `apply_errored` -> sets resource status to `"errored"`.
  - `diagnostic` -> records job-level error / warning diagnostics.

### 3. Post-Apply State Resolution (`resolveFinalValues`)
Terraform plan leaves computed values as `afterUnknown: true`. After apply finishes successfully:
1. Executes `terraform show -json` to inspect the post-apply `state.json`.
2. **Collects State Values & Sensitivities**:
   - `collectResourceValues()` gathers both `resource.values` and `resource.sensitive_values` by resource address.
3. **Merges Sensitivity Metadata (`mergeSensitiveMetadata`)**:
   - Combines plan-time `afterSensitive` with post-apply `sensitive_values` from state.
4. **Normalizes Dynamic Resource Sensitivity (`normalizeResourceSensitivities`)**:
   - For `terraform_data` (and dynamic mirror resources), copies `input` sensitivity to `output` if `output` sensitivity is empty.
5. **Recursive Unknown Resolution (`resolveUnknownRecursive`)**:
   - **Container-Level Guard**: If `afterSensitiveNode` is `Boolean.TRUE`, stops recursion immediately and preserves `after` as `null`.
   - **Leaf Resolution**: At `afterUnknown: true` leaves, splices the post-apply state value into `after`, runs `sanitizeSensitiveValues()` to redact any nested secrets, and flips `afterUnknown` to `false`.

---

## 4. Sensitivity Protection Matrix

| Scenario | Plan Metadata (`afterSensitive`) | State (`sensitive_values`) | Terrakube Resolution Action | Result in `after` |
|---|---|---|---|---|
| **Simple Leaf Secret** | `password = true` | `password = true` | Leaf unknown skipped / redacted | `password: null` |
| **Container Sensitivity** | `credentials = true` | `credentials = true` | Early return on container | `credentials: null` |
| **Unknown Leaf -> Secret Object** | `token = { secret: true }` | `token = { secret: true }` | `sanitizeSensitiveValues` redacts nested sensitive keys | `token: { public: "x", secret: null }` |
| **Dynamic Mirror (`terraform_data`)** | `input = true, output = {}` | `input = true` | `normalizeResourceSensitivities` copies `input` -> `output` | `input: null, output: null` |

---

## 5. Context API & Security Sanitization

1. **Context Endpoints**:
   - `GET /context/v1/{jobId}`: Fetches complete context JSON.
   - `POST /context/v1/{jobId}`: Saves updated context JSON from executor.
   - `GET /context/v1/{jobId}/stream`: Server-Sent Events (SSE) stream for real-time UI updates.
2. **Server-Side Sanitizer (`ContextSanitizer`)**:
   - Spring Boot filter on the API server that performs a final recursive redaction pass on `planStructuredOutput`, `applyStructuredOutput`, and `terraformOutputs` before returning payloads to HTTP/SSE clients.

---

## 6. Durable Structured Output Recovery via Cloud Storage Spooling

### 1. Problem & Zero-Rerun Invariant
During an API outage, network partition, or token expiration, the final structured snapshots and diagnostics produced by Terraform/OpenTofu cannot be delivered synchronously to `/context/v1/{jobId}`. If retained only in ephemeral executor JVM memory, pod termination would permanently destroy the output. Terrakube enforces a strict **zero infrastructure rerun invariant**—Terraform or OpenTofu must never be re-executed to regenerate lost UI diffs or diagnostics, as doing so risks unintended state drift or resource recreation.

### 2. Architecture & Delivery Pipeline

```
                      +-------------------+
                      |   Executor Pod    |
                      +-------------------+
                                |
                 Step Finishes (Plan/Apply/Destroy)
                                |
                 +--------------+---------------+
                 |                              |
         [1. Synchronous Delivery]       [API Outage / 5xx]
                 |                              |
                 v                              v
        POST /context/v1/{jobId}       [2. Cloud Storage Spool]
                 |                     context-spool/<org>/<job>/<step>-<phase>.json.gz
                 v                     (S3 / Azure Blob / GCS / Local)
        API Confirms (200 OK)                   |
        deliveryState = ACKED                   v
                                       deliveryState = SPOOLED
                                                |
                                       [Pod Terminates Safely]
                                                |
                                 +--------------+--------------+
                                 |                             |
                       [3. Periodic Sweeper]       [Dead Executor Probe]
                       SpoolRecoveryJob (30s)      JobReconciliationSweep
                                 |                             |
                                 +--------------+--------------+
                                                |
                                                v
                                 Claims atomic DB lease (30s)
                                 Downloads & verifies payload:
                                   * CWE-409 bounded decompression (5MB/15MB)
                                   * Security envelope verification
                                   * ContextPatchApplier path & revision validation
                                                |
                                                v
                                 Applies patch to context.json
                                 (Protected by ContextWriteLock)
                                                |
                                                v
                                 deliveryState = RECOVERED
                                 Deletes spooled patch
```

### 3. Step `DeliveryState` Lifecycle Contract
The `step` table tracks the durability state of each step's output via `delivery_state`:
* **`NOT_DURABLE`**: Default initial state while the step is running, or when the local storage backend is unshared (`TERRAKUBE_LOCAL_SHARED_DIRECTORY` not set).
* **`ACKED`**: The executor synchronously delivered the patch to the API, confirmed with HTTP 200 OK.
* **`SPOOLED`**: The API was unreachable during synchronous publishing; the executor durably saved the compressed patch to cloud object storage.
* **`RECOVERED`**: The background sweeper or dead-executor probe claimed the spool, validated the envelope and path semantics, merged the patch via `ContextPatchApplier`, and updated the step.
* **`EXPIRED`**: The spool exceeded the retention window (24h) or contained an invalid/corrupted payload (CWE-409 decompression error, envelope mismatch, or `InvalidContextPatchException`). The corrupted spool is deleted and further retries are suppressed.

### 4. Spooling & Compression Invariants
* **Storage Location**: Stored under `context-spool/<orgId>/<jobId>/<stepId>-<phase>.json.gz` across S3, Azure Blob, GCS, or Local Shared directory.
* **Security Envelope**: The payload contains an outer cryptographic metadata envelope (`organizationId`, `jobId`, `stepId`, `phase`, `deliveryRevision`, `timestamp`). Recovery rejects mismatches to prevent cross-tenant or cross-step injection.
* **CWE-409 Decompression Limits**: Enforced by `SpoolCompressionUtil`:
  * Max compressed bytes: `5 MB`
  * Max uncompressed bytes: `15 MB` (prevents decompression bombs).
* **Workspace Deletion**: Residual spools in `context-spool/` are automatically deleted upon workspace deletion.

### 5. Clustered Sweeper & Merge Semantics (`SpoolRecoveryService`)
* **Atomic DB Lease Fencing**: `SpoolRecoveryService` claims batches of `SPOOLED` steps whose `leaseUntil` has expired using conditional JPQL updates (`acquireLease`, `releaseLease`, and owner-conditional `updateDeliveryState`).
* **Unified Merge Pipeline**: Spool recovery delegates directly to `ContextWriteService.patchRecovery(jobId, revision, patch)`:
  * Uses `ContextPatchApplier.parse` and `apply` under distributed Redis `ContextWriteLock`.
  * Splits slash-delimited path keys (`planStructuredOutput/<stepId>`, `terrakubeUI/<stepId>`, `jobDiagnostics/<stepId>`) into nested JSON structures matching UI expectations.
  * Honors monotonic revision sequencing: skips stale spooled data if `deliveryRevision <= storedRevision`.
  * Enforces server-side `ContextSanitizer` to scrub secrets before persistence.
* **Invalid Patch Purging**: If `ContextPatchApplier.parse` throws `InvalidContextPatchException`, the spool is immediately deleted from storage, the step is marked `EXPIRED`, and retries are stopped.

### 6. Dead-Executor Storage Probing (`JobReconciliationSweep`)
* If an API outage persists and an executor terminates before updating the step to `SPOOLED` (step remains in `NOT_DURABLE`):
  * When `executor-job-heartbeat:<jobId>` expires in Redis, `failIfExecutorHeartbeatExpired` probes object storage for residual `context-spool/` files for that job.
  * If found, it merges the spooled output into `context.json`, marks the step `RECOVERED`, and marks the dead job `failed`.

### 7. Deployment Ordering & Compatibility
* **API First**: Always deploy API migrations (`changelog-2.34.0-step-delivery-state.xml`) before upgrading executors.
* **Executor Fallback**: If an upgraded executor encounters an older API that rejects delivery fields with HTTP 400 or 404, `UpdateJobStatusImpl.updateStepStatus` automatically retries the PATCH using plain `StepAttributes` (without delivery fields), guaranteeing non-breaking rollout ordering.

---

## 7. Common Troubleshooting & Testing Patterns

### Running Executor Tests
```bash
mvn test -Dtest=PlanStructuredOutputServiceTest,ApplyStructuredOutputServiceTest,StructuredOutputSpoolFallbackTest,ContextSpoolServiceTest,UpdateJobStatusImplTest -pl executor
```

### Running API Spool Recovery Tests
```bash
mvn test -Dtest=SpoolRecoveryServiceTest,StructuredOutputSpoolRecoveryIntegrationTest,StructuredOutputSpoolRecoveryNonTransactionalIntegrationTest,StepRepositoryLeaseTest,ContextWriteServiceTest -pl api
```

### Validating Context Payload
Fetch the job context using a Terrakube API token:
```bash
curl -s -H "Authorization: Bearer <TOKEN>" https://terrakube-api.localhost/context/v1/<JOB_ID> | jq .
```
