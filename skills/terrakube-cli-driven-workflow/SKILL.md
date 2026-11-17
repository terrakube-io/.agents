---
name: terrakube-cli-driven-workflow
description: End-to-end guide for testing and troubleshooting Terraform CLI-driven local/remote execution workflows and private Registry modules/providers in Terrakube, including Dex/Federated/Internal authentication, dual-host .tfrc setup, .terraformignore rules, parameter passing (-var, --var, -var-file), Minikube live testing, and diagnostic runbooks.
---

# Terrakube Terraform CLI-Driven Workflow & Troubleshooting Guide

This skill documents how to configure, test, and troubleshoot the Terraform CLI-driven workflow (remote execution via `cloud { ... }` or `remote` backend) against Terrakube, including parameter handling, Minikube deployments, and runtime diagnostics.

---

## 1. Protocol Architecture & Lifecycle

When a developer runs `terraform plan` or `terraform apply` locally against a workspace configured for remote execution, the Terraform CLI communicates with Terrakube via the TFE v2 Remote API (`/remote/tfe/v2/*`):

```
Local Terminal                    Terrakube API                  PostgreSQL & Storage            Executor (Pod / Ephemeral)
      |                                 |                                 |                                  |
 1.   |--- GET /remote/tfe/v2/ping ---->|                                 |                                  |
 2.   |--- GET /remote/tfe/v2/workspaces/:org/:ws ----------------------->|                                  |
 3.   |--- POST /.../configuration-versions (speculative, auto-queue-runs) -->|                                  |
 4.   |--- PUT <upload-url> (packages & uploads project .tar.gz) -------->| (MinIO / S3 storage)             |
 5.   |--- POST /remote/tfe/v2/runs (data.attributes.variables) -------->|                                  |
      |       [Parses HCL literals via CliRunVariables]                   |                                  |
      |       [Sets job.variables_as_arguments = true]                    |                                  |
      |                                 |--- INSERT Job ----------------->|                                  |
 6.   |                                 |--- Dispatch Step (Quartz/Redis) ---------------------------------->|
      |                                 |                                 |     Writes terrakube-run-variables.tfvars (0600)
      |                                 |                                 |     Runs: terraform plan --var ... (-var-file for non-string HCL)
 7.   |<-- Streams logs via WebSocket / SSE / Polling -------------------------------------------------------|
```

### Key Components:
- **`RemoteTfeService`**: Handles workspace lookup, configuration version creation, and run creation (`POST /remote/tfe/v2/runs`).
- **`CliRunVariables`**: Parses and validates CLI parameters as HCL literals (strings, heredocs, numbers, booleans, tuples, objects, null), rejecting expressions and interpolations upfront with HTTP 400.
- **`Job.variablesAsArguments`**: Boolean flag marking that the job received CLI parameters that must override workspace/template defaults.
- **`RunVariablesFile` (Executor)**: Passes string literals as `--var name=value` arguments and serializes non-string HCL literals into `terrakube-run-variables.tfvars` (`0600`, cleaned in `finally`), attached via `-var-file` on `plan`, `destroy`, and un-planned `apply`.
- **HTTP State Backend (`HttpTerraformStateImpl`)**: When enabled, configures `TF_HTTP_USERNAME` / `TF_HTTP_PASSWORD` (token TTL tied to `ExecutorJobMaxDurationMinutes`, default 360 min) and omits `-backend-config` init arguments via `BackendConfigArguments.strip`. Binary caching uses `tfbinary/{product}/{version}/{os}_{arch}/{product}`, and resolved OpenTofu versions strip leading `v` prefixes.

---

## 2. Client Authentication & Environment Setup

### 2.1 Credentials Configuration
Terraform CLI reads API tokens from `~/.terraform.d/credentials.tfrc.json` (or `TF_CLI_CONFIG_FILE`):
```json
{
  "credentials": {
    "terrakube-api.localhost": {
      "token": "<JWT_OR_PAT_TOKEN>"
    }
  }
}
```

#### Environment Hostnames (Devcontainer vs Minikube)
- **Devcontainer:** `API_URL = "https://terrakube-api.localhost"`, `UI_URL = "https://terrakube.localhost"`, `REG_HOST = "terrakube-registry.localhost"`
- **Minikube:** `API_URL = "https://terrakube-api.minikube.net"`, `UI_URL = "https://terrakube-ui.minikube.net"`, `REG_HOST = "terrakube-reg.minikube.net"`, `SSL_CERT_FILE = "/home/user/.local/share/mkcert/rootCA.pem"`

#### Checking Token Expiry & Refreshing a Dex JWT via Python
Dex `id_token` JWTs expire (typically 24 hours). Always verify `exp` before running E2E scripts (`jwt.decode(token, options={"verify_signature": False})["exp"] > time.time()`); if expired (`401 Unauthorized` on the first API call), refresh `~/.terraform.d/credentials.tfrc.json` against `/dex/auth` and `/dex/token` (using the exact ingress `UI_URL`, e.g., `https://terrakube-ui.minikube.net` on Minikube or `https://terrakube.localhost` in devcontainer):
```python
import json, os, re, requests, urllib.parse

API_URL = os.environ.get("TERRAKUBE_API_URL", "https://terrakube-api.localhost")
UI_URL = os.environ.get("TERRAKUBE_UI_URL", "https://terrakube.localhost")
API_HOST = urllib.parse.urlparse(API_URL).hostname

s = requests.Session()
s.verify = os.environ.get("SSL_CERT_FILE", True)

# Authorization code flow against Dex / OpenLDAP (default devcontainer: admin@example.com / admin)
r1 = s.get(f"{API_URL}/dex/auth?client_id=example-app&redirect_uri={UI_URL}&response_type=code&scope=openid+email+profile+groups&state=123")
action = re.search(r'action="([^"]+)"', r1.text).group(1).replace("&amp;", "&")
r2 = s.post(urllib.parse.urljoin(API_URL, action), data={"login": "admin@example.com", "password": "admin"}, allow_redirects=False)
loc = r2.headers.get("Location")
while "code=" not in loc:
    r2 = s.get(urllib.parse.urljoin(API_URL, loc), allow_redirects=False)
    loc = r2.headers.get("Location")
code = urllib.parse.parse_qs(urllib.parse.urlparse(loc).query)["code"][0]

r3 = s.post(f"{API_URL}/dex/token", data={
    "grant_type": "authorization_code",
    "client_id": "example-app",
    "code": code,
    "redirect_uri": UI_URL,
})
token = r3.json()["id_token"]

os.makedirs(os.path.expanduser("~/.terraform.d"), exist_ok=True)
with open(os.path.expanduser("~/.terraform.d/credentials.tfrc.json"), "w") as f:
    json.dump({"credentials": {API_HOST: {"token": token}}}, f, indent=2)
```

#### Dual-Host `.tfrc` (`TF_CLI_CONFIG_FILE`) for Private Registry Modules & Providers
When a Terraform CLI workspace uses `cloud { hostname = ... }` **and** resolves private Registry modules or providers (`source = "terrakube-reg.minikube.net/<org>/<name>/<provider>"`), `terraform init` requires credentials for **both** the API host and the Registry host. Generate a scoped `.tfrc` file and pass it via `TF_CLI_CONFIG_FILE` alongside `SSL_CERT_FILE`:
```hcl
credentials "terrakube-api.minikube.net" {
  token = "<USER_TOKEN>"
}
credentials "terrakube-reg.minikube.net" {
  token = "<USER_TOKEN>"
}
```
```bash
export SSL_CERT_FILE=/home/user/.local/share/mkcert/rootCA.pem
export TF_CLI_CONFIG_FILE=/path/to/terraformrc.tfrc
```

### 2.2 TLS / Custom CA Certificates
If testing against a local or Minikube environment using a private CA:
```bash
# Export path to root CA so Terraform CLI can verify the API ingress certificate:
export SSL_CERT_FILE=/path/to/rootCA.pem
```

### 2.3 DNS / Host Resolution
* **Devcontainer (`*.localhost`)**: `terrakube-api.localhost` and `terrakube-dex.localhost` resolve to `127.0.0.1` automatically without `/etc/hosts` changes.
* **Custom Cluster / Minikube**: Ensure `/etc/hosts` maps your ingress hostnames (e.g. `terrakube-api.minikube.net`) to the cluster IP.

---

## 3. Creating & Running a Test Project

### 3.1 Local Terraform Configuration (`main.tf`)
```hcl
terraform {
  cloud {
    hostname     = "terrakube-api.localhost" # Or your cluster's API hostname
    organization = "simple"

    workspaces {
      name = "cli-test-workspace"
    }
  }
}

variable "sample_var" {
  type        = string
  description = "Required variable without default"
}

variable "sample_number" {
  type    = number
  default = 0
}

variable "sample_bool" {
  type    = bool
  default = false
}

variable "sample_list" {
  type    = list(string)
  default = []
}

output "sample_var_output" {
  value = var.sample_var
}

output "sample_number_output" {
  value = var.sample_number
}

output "sample_bool_output" {
  value = var.sample_bool
}

output "sample_list_output" {
  value = var.sample_list
}
```

### 3.2 Workspace Pre-requisites
1. The organization (e.g. `simple`) must exist.
2. The workspace (e.g. `cli-test-workspace`) must exist under that organization.
3. The workspace **Execution Mode** must be set to `remote` (never `local`).

---

## 4. Parameter Passing Patterns & Verification

### 4.1 Single-Dash (`-var`)
```bash
terraform plan -var 'sample_var=hello-world'
```
- Passes `sample_var` in `data.attributes.variables`.
- Remote plan receives `--var sample_var="hello-world"`.

### 4.2 Double-Dash (`--var`)
```bash
terraform plan --var sample_var=hello-world
```
- Fully supported by Terraform CLI and ingested identically.

### 4.3 Variable Files (`-var-file`)
```bash
cat <<EOF > test.tfvars
sample_var    = "from-file"
sample_number = 42
sample_bool   = true
sample_list   = ["alpha", "beta", "gamma"]
EOF

terraform plan -var-file=test.tfvars
```

### 4.4 Precedence Testing (CLI Override)
```bash
terraform plan -var-file=test.tfvars -var sample_var=cli-override-wins
```
- Terraform CLI bundles both into `variables`.
- The CLI `-var` overrides the value defined in `test.tfvars`, and both override workspace default values in `terraform.tfvars`.

---

## 5. Testing Code Changes in Minikube

When modifying API, Registry, or Executor code (e.g. PR testing):

### 5.1 Compile Images
```bash
# API + Executor
mvn -pl "api,executor" spring-boot:build-image -B --file pom.xml -Dmaven.test.skip=true

# API + Registry (produces docker.io/library/api-server:2.34.0 and open-registry:2.34.0)
mvn -pl "api,registry" spring-boot:build-image -B --file pom.xml -Dmaven.test.skip=true
```
- **Local `terrakube-spring-boot-starter` changes:** If testing an unreleased client starter version, update `<revision>` across all starter POMs before installing (`mvn versions:set-property -Dproperty=revision -DnewVersion=<ver> -DgenerateBackupPoms=false` then `mvn clean test install -Dgpg.skip=true -B`) so transitive modules (such as `terrakube-client`) resolve the matching version in `~/.m2/repository`, then restore the starter POMs. Passing `-Drevision=...` only on the CLI leaves `<revision>1.9.0</revision>` in child POMs.

### 5.2 Avoid the Minikube Image Caching Trap
> [!WARNING]
> Minikube CRI/containerd caches images by digest and tag. Reusing `:latest` or `:2.34.0` often causes containerd to continue running old image layers even after `minikube image load`.

**Mandatory Pattern:** Always use a unique tag:
```bash
# 1. Tag with unique identifier
docker tag docker.io/library/api-server:2.34.0 azbuilder/api-server:test-pr-XYZ
docker tag docker.io/library/open-registry:2.34.0 azbuilder/open-registry:test-pr-XYZ
docker tag docker.io/library/executor:2.34.0 azbuilder/executor:test-pr-XYZ

# 2. Load into Minikube containerd store
minikube image load azbuilder/api-server:test-pr-XYZ
minikube image load azbuilder/open-registry:test-pr-XYZ
minikube image load azbuilder/executor:test-pr-XYZ

# 3. Update deployments (note container names: terrakube-api, terrakube-registry, executor)
kubectl -n terrakube set image deployment/terrakube-api terrakube-api=azbuilder/api-server:test-pr-XYZ
kubectl -n terrakube set image deployment/terrakube-registry terrakube-registry=azbuilder/open-registry:test-pr-XYZ
kubectl -n terrakube set env deployment/terrakube-api ExecutorEphemeralImage=azbuilder/executor:test-pr-XYZ
kubectl -n terrakube set image deployment/terrakube-executor executor=azbuilder/executor:test-pr-XYZ

# 4. Wait for rollouts
kubectl -n terrakube rollout status deployment/terrakube-api --timeout=180s
kubectl -n terrakube rollout status deployment/terrakube-registry --timeout=180s
kubectl -n terrakube rollout status deployment/terrakube-executor --timeout=180s
```

### 5.3 Verify Database Migration
Check API startup logs to verify Liquibase ran successfully:
```bash
kubectl -n terrakube logs deployment/terrakube-api | grep -i "ChangeSet db/changelog"
```

---

## 6. End-to-End Testing: Private Modules, Providers & Authentication Modes

### 6.1 Seeding & Cleaning Up Temporary Private Providers (Elide JSON:API)
When an environment only has public/existing modules (e.g. `aws/vpc/aws`), seed a temporary private provider via `POST` with an org-authorized user token (`manageProvider=true` team):
1. Create provider: `POST /api/v1/organization/{orgId}/provider` (`"type": "provider"`, `"attributes": {"name": prov_name, "description": "..."}`)
2. Create version: `POST /api/v1/organization/{orgId}/provider/{providerId}/version` (`"type": "version"`, `"attributes": {"versionNumber": "3.0.1", "protocols": "5.0"}`)
3. Create implementation: `POST /api/v1/organization/{orgId}/provider/{providerId}/version/{versionId}/implementation` (`"type": "implementation"`, `"attributes": {"os": "linux", "arch": "amd64", "filename": "...", "downloadUrl": "...", "shasumsUrl": "...", "shasumsSignatureUrl": "...", "shasum": "<64-hex>", "keyId": "...", "asciiArmor": "...", "trustSignature": "", "source": "terraform", "sourceUrl": "..."}`)
4. **Teardown (`finally`):** Delete in reverse order: `implementation` -> `version` -> `provider`.

### 6.2 Registry Module & Provider Endpoints
- **Modules:**
  - `GET /terraform/modules/v1/{org}/{name}/{provider}/versions`
  - `GET /terraform/modules/v1/{org}/{name}/{provider}/{version}/download`
  - `GET /terraform/readme/v1/{org}/{name}/{provider}/{version}/download`
- **Providers:**
  - `GET /terraform/providers/v1/{org}/{provider}/versions`
  - `GET /terraform/providers/v1/{org}/{provider}/{version}/download/{os}/{arch}`

### 6.3 Authentication Modes for Registry E2E Verification
1. **Dex User / PAT / Team Token (`NOT_INTERNAL` / `OIDC_USER` / `PAT_OR_TEAM_TOKEN`):**
   - Authorized organization returns `200 OK`; unauthorized organization or RSQL/GraphQL injection payloads in the organization path (`aws%2Cname==azure`, `simple*`, `simple%22`) return `403 Forbidden`.
2. **Federated OIDC (`FEDERATED`) via Ephemeral In-Cluster Mock OIDC:**
   - Stand up an nginx `ConfigMap` + `Pod` + `Service` serving `/.well-known/openid-configuration` and `/jwks.json` using an **ephemeral per-run resource name** (`mock-oidc-<run_id>`). Never reuse the same K8s `Service` name across runs to avoid stale `kube-proxy`/CoreDNS timeouts.
   - Wait for `Pod` readiness and non-empty `Endpoints`, create `Federated` (`issuer = "http://mock-oidc-<run_id>.terrakube.svc.cluster.local:8080"`, `audience`, `name`) + `FederatedClaim`, and create a `Team` in the target org whose `name == Federated.name`.
   - Sign test JWTs carrying matching `aud`/claims **plus decoy `email`/`groups`** (`TERRAKUBE_ADMIN`) to prove Registry ignores unverified claims and authorizes strictly by `Federated.name` (`200` on authorized org, `403` on other orgs, `401` on mismatched claim).
3. **Internal Service & Live `WORKSPACE_EXECUTOR` Runner Tokens:**
   - Extract the signing key from Kubernetes:
     ```bash
     kubectl -n terrakube get secret terrakube-registry-secrets -o jsonpath={.data.InternalSecret}
     ```
     Base64-decode the K8s Secret value, then URL-safe base64-decode (padding `=` to multiple of 4) to raw key bytes (`HS512` when `>= 64` bytes, `HS384` `>= 48`, else `HS256`), and set `"iss": "TerrakubeInternal"`.
   - Assert internal service taxonomy: `sub = "TerrakubeInternal (TOKEN)"` (`UNKNOWN_INTERNAL`) -> `403`; `sub = "TerrakubeInternal (EXECUTOR)"` without job context (`EXECUTOR_SERVICE`) -> `403`; `sub = "TerrakubeInternal (REGISTRY)"` (`REGISTRY_SERVICE`) -> `200` on Registry and permitted only on non-sensitive `Job`/`Step` status fields via `terrakube-api` (`Job.output`, `Job.terraformPlan`, `Step.output` remain hidden/rejected).
   - **Minting/Testing Active `WORKSPACE_EXECUTOR` Tokens:** Launch an interactive `terraform apply -no-color` against a remote workspace using `pty.openpty()` so Terraform holds at the `Enter a value:` approval prompt while `Job` and its `Step` remain in `running` status. Poll `GET /api/v1/organization/{orgId}/workspace/{wsId}?include=job,job.step` for `jobId`/`stepId`, then verify:
     - Active token (`organizationId`, `workspaceId`, `jobId`, `stepId`, `jti = "<jobId>-<stepId>"`) -> `200` on own org modules/providers, `403` on cross-org modules/providers, `403` when `workspaceId` does not match `Job.workspace`.
     - Missing `stepId` with exact `jti = "<jobId>-<stepId>"` -> `200`; wrong `jobId` prefix in `jti` -> `403`.
     - After cancelling the job (`PATCH /api/v1/organization/{orgId}/job/{jobId}` with `{"data": {"type": "job", "id": jobId, "attributes": {"status": "cancelled"}}}`), the same token returns `403` immediately on both modules and providers.

### 6.4 Local vs Remote CLI Workspace Tests & `.terraformignore`
1. Create temporary workspaces via `POST /api/v1/organization/{orgId}/workspace` (`executionMode: "local"` or `"remote"`, `terraformVersion: "1.9.8"`) and attach `Access` (`POST .../workspace/{wsId}/access`) if required by team permissions.
2. **Always exclude `.terraform/` on Remote CLI runs:** `terraform init` populates `.terraform/modules/` locally. Unless `.terraform/` is listed in `.terraformignore`, `terraform plan` bundles cached modules inside the uploaded configuration `.tar.gz`, bypassing `terrakube-executor` downloads from `terrakube-reg.minikube.net`! Create `.terraformignore` containing `.terraform/` in remote test directories so same-org remote plans prove live runner-token registry auth (`Plan: ...`) and cross-org remote plans fail cleanly with `403 Forbidden`.
3. **Workspace Teardown Fallback:** Delete any created `access` rows first, then `DELETE /api/v1/organization/{orgId}/workspace/{wsId}`. Workspaces that executed jobs return `400` or `409` on hard `DELETE` because of historical run records; fall back to soft-delete via `PATCH` with `{"data": {"type": "workspace", "id": wsId, "attributes": {"deleted": True}}}`.

---

## 7. Troubleshooting Runbook

| Symptom | Probable Cause | Diagnostic / Resolution Steps |
| :--- | :--- | :--- |
| **`x509: certificate signed by unknown authority`** | Custom or self-signed CA on local/cluster ingress | Export root certificate: `export SSL_CERT_FILE=/path/to/rootCA.pem`. Verify with `curl -v --cacert /path/to/rootCA.pem https://<api-host>/remote/tfe/v2/ping`. |
| **`Error: 401 Unauthorized`** | Missing or expired token in `credentials.tfrc.json` | Verify `~/.terraform.d/credentials.tfrc.json` contains token under exact hostname. Inspect token claims: `echo $TOKEN \| jq -R 'split(".") \| .[1] \| @base64d \| fromjson'`. Verify expiry (`exp`). |
| **`Error: Workspace not found`** | Case sensitivity or workspace does not exist | Check workspace existence via API or UI. Check workspace execution mode is `remote` (if `local`, CLI fails or runs locally). |
| **`Error: No value for required variable`** (despite passing `-var`) | API discarding `data.attributes.variables` or running outdated image | 1. Check API pod image: `kubectl -n terrakube get deployment terrakube-api -o jsonpath='{.spec.template.spec.containers[0].image}'`.<br>2. Check `job` table in PostgreSQL: `SELECT id, status, variables_as_arguments FROM job ORDER BY id DESC LIMIT 5;`.<br>3. Verify `variables_as_arguments` is `true`. |
| **HTTP 400 Bad Request on `terraform plan`** | Dynamic expression or invalid HCL passed in CLI variable | `CliRunVariables` enforces HCL literal syntax. Passing expressions like `-var 'foo=${var.bar}'` or unquoted complex structures triggers validation failure. Ensure literals are valid HCL syntax. |
| **CLI hangs on `Preparing the remote plan...`** | Configuration upload failure or Redis lock blocking FIFO queue | 1. Check MinIO / storage access.<br>2. Check Redis lock: `docker exec -it redis-service redis-cli keys "job-execution-lock:*"`.<br>3. Check API logs for scheduler dispatch exceptions: `kubectl -n terrakube logs deployment/terrakube-api --tail=200`. |
| **Executor Job Pod fails or restarts** | Custom CA missing in executor pod or permission errors | 1. Check executor pod logs: `kubectl -n terrakube logs -l app.kubernetes.io/name=executor`.<br>2. If using Ephemeral Executor, check init container: `kubectl -n terrakube logs <job-pod> -c copy-executor-secrets`. |

---

## 8. Useful Diagnostic Commands

```bash
# Query recent jobs in PostgreSQL
kubectl -n terrakube exec -it postgresql-service -- psql -U terrakube -d terrakubedb -c \
  "SELECT id, status, variables_as_arguments, created_date FROM job ORDER BY id DESC LIMIT 5;"

# Stream live API logs for remote runs
kubectl -n terrakube logs -f deployment/terrakube-api | grep -E "RemoteTfeService|CliRunVariables|ScheduleJob"

# Check running ephemeral executor jobs
kubectl -n terrakube get jobs -n terrakube
kubectl -n terrakube get pods -n terrakube -l job-name
```

