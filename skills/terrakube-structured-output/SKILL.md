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

## 6. Common Troubleshooting & Testing Patterns

### Running Executor Tests
```bash
mvn test -Dtest=PlanStructuredOutputServiceTest,ApplyStructuredOutputServiceTest
```

### Validating Context Payload
Fetch the job context using a Terrakube API token:
```bash
curl -s -H "Authorization: Bearer <TOKEN>" https://terrakube-api.localhost/context/v1/<JOB_ID> | jq .
```
