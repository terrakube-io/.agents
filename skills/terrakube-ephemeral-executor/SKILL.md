---
name: terrakube-ephemeral-executor
description: Comprehensive architecture, configuration options, and execution lifecycle guide for the Terrakube Ephemeral Executor (Kubernetes Jobs).
---

# Terrakube Ephemeral Executor Guide

The Ephemeral Executor runs each job/step in an isolated, on-demand Kubernetes `batch/v1` `Job` rather than dispatching tasks to a long-running persistent executor service over HTTP.

---

## 1. Dispatch & Activation

### Activation Condition
In `ExecutorService.java`, Terrakube checks if the job environment contains:
```
TERRAKUBE_ENABLE_EPHEMERAL_EXECUTOR=1
```
- **If Present**: Dispatched to `EphemeralExecutorService.send(job, executorContext)`.
- **If Absent**: Dispatched to `PersistentExecutorService.send(job, executorContext)` via HTTP REST.

### Helm Chart Configuration (Cluster-wide)
To enable the ephemeral executor via Helm:
```yaml
api:
  serviceAccountName: "terrakube-api-service-account"
  ephemeralExecution:
    enabled: true
```

### Required Kubernetes RBAC
The `terrakube-api` service account requires permissions to manage `batch/v1` `jobs` in the target namespace:
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: terrakube-api-role
  namespace: terrakube
rules:
  - apiGroups: ["batch"]
    resources: ["jobs"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
```

---

## 2. Ephemeral Job Lifecycle & Delivery

1. **Job Manifest Generation**:
   - `EphemeralExecutorService` constructs a `batch/v1` `Job` named `job-<jobId>-<timestamp>`.
   - Pod restart policy is set to `Never` and `ttlSecondsAfterFinished` is set to `30`.
2. **Payload Delivery**:
   - The `ExecutorContext` is serialized to JSON, Base64-encoded, and passed as environment variable `EphemeralJobData`.
   - `EphemeralFlagBatch="true"` is passed to signal batch execution mode.
3. **Server-Side Apply**:
   - Manifest is submitted via Fabric8 `kubernetesClient.batch().v1().jobs().inNamespace(...).resource(k8sJob).serverSideApply()`.
4. **Error Classification & Retries**:
   - `isRetryable(KubernetesClientException)` classifies transient issues (unreachable apiserver, HTTP 429 rate limit, 5xx server errors, HTTP 403 quota exceeded) into `ExecutorUnavailableException` for automatic retry.
   - Genuine misconfigurations or permanent RBAC rejections throw `ExecutionException` to fail immediately.
5. **Execution inside Container**:
   - On container startup, `BatchModeServiceImpl` listens for `ContextRefreshedEvent`, decodes `EphemeralJobData`, deserializes into `TerraformJob`, executes the job steps, and terminates.

---

## 3. Configuration & Options Reference

### Spring Boot Properties (`application.properties`)
| Property | Environment Variable | Default | Description |
| :--- | :--- | :--- | :--- |
| `io.terrakube.executor.ephemeral.namespace` | `ExecutorEphemeralNamespace` | `terrakube` | Target namespace for ephemeral jobs |
| `io.terrakube.executor.ephemeral.image` | `ExecutorEphemeralImage` | `azbuilder/executor:<version>` | Executor container image |
| `io.terrakube.executor.ephemeral.secret` | `ExecutorEphemeralSecret` | `terrakube-executor-secrets` | Comma-separated Secret names mounted via `envFrom` |
| `io.terrakube.executor.ephemeral.nodeSelector.*` | `-Dio.terrakube.executor.ephemeral.nodeSelector.<key>=<value>` | None | Default node selector key-value pairs |

---

### Workspace / Per-Job Environment Variables
These variables can be set in the workspace (or global variables) with category `ENV`:

#### 1. Resource Requests & Limits
| Variable | Format / Example | Target Resource |
| :--- | :--- | :--- |
| `EPHEMERAL_CPU_REQUEST` | `100m`, `1` | `resources.requests.cpu` |
| `EPHEMERAL_CPU_LIMIT` | `200m`, `2` | `resources.limits.cpu` |
| `EPHEMERAL_MEMORY_REQUEST` | `50Mi`, `1Gi` | `resources.requests.memory` |
| `EPHEMERAL_MEMORY_LIMIT` | `100Mi`, `2Gi` | `resources.limits.memory` |
| `EPHEMERAL_STORAGE_REQUEST` | `2Gi`, `500Mi` | `resources.requests.ephemeral-storage` |
| `EPHEMERAL_STORAGE_LIMIT` | `4Gi`, `1Gi` | `resources.limits.ephemeral-storage` |

#### 2. Node Scheduling & Placement
| Variable | Format / Example | Description |
| :--- | :--- | :--- |
| `EPHEMERAL_CONFIG_NODE_SELECTOR_TAGS` | `key1=val1;key2=val2` | Overrides default node selectors |
| `EPHEMERAL_CONFIG_TOLERATIONS` | `key:operator:effect` or `key=val:operator:effect` | Semicolon-separated list of pod tolerations |

#### 3. Security & Service Accounts
| Variable | Format / Example | Description |
| :--- | :--- | :--- |
| `EPHEMERAL_CONFIG_SERVICE_ACCOUNT` | `my-custom-sa` | Overrides default pod service account |
| `EPHEMERAL_CONFIG_POD_SECURITY_CONTEXT` | `fsGroup=1000;runAsNonRoot=true;runAsUser=1000` | Custom PodSecurityContext |
| `EPHEMERAL_CONFIG_SECURITY_CONTEXT` | `allowPrivilegeEscalation=false` | Custom container SecurityContext |

#### 4. Metadata & Annotations
| Variable | Format / Example | Description |
| :--- | :--- | :--- |
| `EPHEMERAL_CONFIG_ANNOTATIONS` | `key1=val1;key2=val2` | Annotations added to Job metadata |
| `EPHEMERAL_CONFIG_POD_ANNOTATIONS` | `vault.hashicorp.com/agent-inject=true` | Annotations added to Pod template (e.g. Vault Agent, Workload Identity) |
| `EPHEMERAL_CONFIG_LABELS` | `team=infra;env=prod` | Custom labels added to Job and Pod metadata |

#### 5. Storage, ConfigMaps & Secrets
| Variable | Format / Example | Description |
| :--- | :--- | :--- |
| `EPHEMERAL_CONFIG_ENVFROM_CONFIG_MAP` | `cm-one,cm-two` | Comma-separated ConfigMap names mounted as `envFrom` |
| `EPHEMERAL_CONFIG_MAP_NAME` | `my-config-map` | ConfigMap mounted as a volume |
| `EPHEMERAL_CONFIG_MAP_MOUNT_PATH` | `/custom/path` (Default: `/data`) | Mount path for `EPHEMERAL_CONFIG_MAP_NAME` |
| `TF_PLUGIN_CACHE_DIR` | `/opt/terrakube/cache` | If set, attaches PVC to cache Terraform plugins |
| `PVC_CLAIM_NAME` | `terrakube-plugin-pvc` (Default) | PVC claim name for plugin cache volume |

#### 6. Custom Container Environment Variables
| Variable | Format / Example | Description |
| :--- | :--- | :--- |
| `EPHEMERAL_JOB_ENV_VARS` | `FOO=bar;BAZ=quux` | Injects additional environment variables directly into the executor container |

---

## 4. Development & Testing Conventions

- **Unit Testing**: Test all ephemeral executor logic in `EphemeralExecutorServiceTest.java`.
- **JDK Requirement**: Use Java 25 (`sdk use java 25.0.2-librca`) when running Maven tests:
  ```bash
  source ~/.sdkman/bin/sdkman-init.sh && sdk use java 25.0.2-librca && mvn test -Dtest=EphemeralExecutorServiceTest
  ```
