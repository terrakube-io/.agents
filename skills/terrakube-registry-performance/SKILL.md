---
name: terrakube-registry-performance
description: Comprehensive workflow, execution patterns, telemetry extraction, and CLI commands for running Terrakube Registry performance analysis and A/B image benchmarks.
---

# Terrakube Registry Performance Analysis & Benchmarking

This skill provides the comprehensive architecture, execution commands, telemetry queries, and reporting patterns for conducting load benchmarks and A/B image comparisons on the **Terrakube Registry** service (`io.terrakube.registry`).

---

## 1. Architecture & Benchmark Directory Structure

All benchmarking tools and configurations reside under `terrakube/benchmarks/`:

```
terrakube/benchmarks/
├── registry/
│   ├── README.md              # Detailed user execution guide
│   ├── config.py              # Configuration & test parameters (default: 5, 10, 25 workers)
│   ├── auth.py                # Pure Python HMAC-SHA256 JWT generator for InternalSecret
│   ├── seeder.py              # CloudNativePG PostgreSQL & SeaweedFS S3 test fixture manager
│   ├── k8s_manager.py         # SSH Kubernetes rollout manager & cold-startup timer
│   ├── load_generator.py      # Async/multi-threaded load engine with p50/p90/p95/p99 histograms
│   ├── telemetry.py           # Prometheus PromQL runtime metrics client
│   ├── visualizer.py          # Standalone dark-themed vector SVG diagram renderer
│   ├── reporter.py            # Markdown report compiler with percentage delta calculations
│   └── run_benchmark.py       # Unified CLI benchmark entrypoint
└── results/
    ├── REGISTRY_PERFORMANCE_ANALYSIS.md  # Detailed benchmark analysis markdown
    ├── latency_comparison_p95_p99.svg    # Latency percentiles vector chart
    ├── throughput_vs_concurrency.svg     # Throughput (RPS) scaling curves vector chart
    ├── jvm_memory_and_gc.svg             # JVM Heap and GC telemetry vector chart
    └── container_cpu_and_startup.svg     # Container RSS and cold-startup vector chart
```

---

## 2. Endpoints & Controllers Under Test

The benchmark harness tests all 7 endpoints across the 4 core registry controllers:

1. **[`WellKnownWebServiceImpl`](file:///home/user/git/terrakube-io/terrakube/registry/src/main/java/io/terrakube/registry/controller/WellKnownWebServiceImpl.java)**:
   - `GET /.well-known/terraform.json`: Service discovery endpoint mapping registry capabilities.
2. **[`ModuleWebServiceImpl`](file:///home/user/git/terrakube-io/terrakube/registry/src/main/java/io/terrakube/registry/controller/ModuleWebServiceImpl.java)**:
   - `GET /terraform/modules/v1/{org}/{module}/{provider}/versions`: Queries database for available module versions.
   - `GET /terraform/modules/v1/{org}/{module}/{provider}/{version}/download`: Returns `302 Found` with `X-Terraform-Get` and increments download counter.
   - `GET /terraform/modules/v1/download/{org}/{module}/{provider}/{version}/module.zip`: Streams raw binary module ZIP (1.5 MB) from SeaweedFS S3 storage.
3. **[`ProviderWebServiceImpl`](file:///home/user/git/terrakube-io/terrakube/registry/src/main/java/io/terrakube/registry/controller/ProviderWebServiceImpl.java)**:
   - `GET /terraform/providers/v1/{org}/{provider}/versions`: Lists provider versions and GPG signing metadata.
   - `GET /terraform/providers/v1/{org}/{provider}/{version}/download/{os}/{arch}`: Returns binary download metadata and SHA256 hashes.
4. **[`ReadMeWebServiceImpl`](file:///home/user/git/terrakube-io/terrakube/registry/src/main/java/io/terrakube/registry/controller/ReadMeWebServiceImpl.java)**:
   - `GET /terraform/readme/v1/{org}/{module}/{provider}/{version}/download`: Fetches module ZIP from S3, decompresses in-memory, extracts `README.md`, and returns markdown content.

---

## 3. Authentication Mechanics

Terrakube Registry uses Spring Security with Yahoo Elide (`DexWebSecurityAdapter`). Requests must present a Bearer JWT signed with the cluster's internal secret:

- **Secret Source**: Kubernetes Secret `terrakube-registry-secrets` (`InternalSecret` field).
- **JWT Header**: `{"alg": "HS256", "typ": "JWT"}`.
- **JWT Payload**:
  ```json
  {
    "iss": "TerrakubeInternal",
    "sub": "benchmark-runner",
    "email": "benchmark@terrakube.io",
    "groups": ["TERRAKUBE_ADMIN"],
    "exp": <timestamp + 86400>
  }
  ```
- **Implementation**: [`auth.py`](file:///home/user/git/terrakube-io/terrakube/benchmarks/registry/auth.py) uses standard Python `hmac` and `hashlib` (no external `jwt` package required).

---

## 4. Test Fixture Provisioning

The harness automatically seeds test data prior to benchmark execution ([`seeder.py`](file:///home/user/git/terrakube-io/terrakube/benchmarks/registry/seeder.py)):
- **Database**: Connects to CloudNativePG PostgreSQL via `kubectl exec -i cnpg-cluster-1 -n cnpg-system -- psql -U postgres -d postgres`.
  - Organization: `simple`
  - Provider: `benchmark` (version `1.0.0`)
  - Module: `aws/vpc/aws` (version `5.7.2`)
- **Object Storage**: Verified in SeaweedFS S3 bucket `terrakube-benchmarks` with 1.5MB `module.zip`.

---

## 5. CLI Execution Patterns

### A. Full A/B Comparative Benchmark
Compares the standard Ubuntu Jammy image against Alpaquita Linux across `[5, 10, 25]` workers:
```bash
cd /home/user/git/terrakube-io/terrakube/benchmarks/registry
python3 run_benchmark.py --workers 5 10 25 --stage-duration 15 --warmup 10
```

### B. Quick Smoke Test
Executes a 5-second per-stage test for rapid verification:
```bash
python3 run_benchmark.py --quick
```

### C. Active Pod Evaluation (Skip Rollouts)
Tests the currently active registry pod without triggering Kubernetes deployments:
```bash
python3 run_benchmark.py --skip-deploy --workers 5 10 25 --stage-duration 10
```

### D. Custom Image or Host Evaluation
```bash
python3 run_benchmark.py \
  --baseline azbuilder/open-registry:2.33.0-beta.10 \
  --candidate my-custom-registry:latest \
  --workers 5 10 25 \
  --stage-duration 20 \
  --host user@192.168.1.150 \
  --url https://terrakube-reg.microk8s.net
```

---

## 6. OpenTelemetry & Prometheus PromQL Queries

Telemetry is scraped by Prometheus from OpenTelemetry JavaAgent port `9464` on the `terrakube-registry` pod:

| Telemetry Dimension | PromQL Metric Query |
| :--- | :--- |
| **JVM Heap Used** | `sum(jvm_memory_used_bytes{jvm_memory_type="heap"})` |
| **JVM Heap Committed** | `sum(jvm_memory_committed_bytes{jvm_memory_type="heap"})` |
| **JVM Non-Heap / Metaspace** | `sum(jvm_memory_used_bytes{jvm_memory_type="non_heap"})` |
| **Total GC Pause Time** | `sum(jvm_gc_duration_seconds_sum) * 1000` |
| **Total GC Count** | `sum(jvm_gc_duration_seconds_count)` |
| **Live Thread Count** | `sum(jvm_thread_count)` |
| **Container RSS Working Set** | `sum(container_memory_working_set_bytes{namespace="terrakube", container="terrakube-registry"}) / (1024 * 1024)` |

---

## 7. Report & Diagram Generation

- **SVG Renderer** ([`visualizer.py`](file:///home/user/git/terrakube-io/terrakube/benchmarks/registry/visualizer.py)): Generates standalone, dark-themed vector diagrams formatted for high-contrast viewing.
- **Report Compiler** ([`reporter.py`](file:///home/user/git/terrakube-io/terrakube/benchmarks/registry/reporter.py)): Produces `REGISTRY_PERFORMANCE_ANALYSIS.md` featuring delta speedup/regression tags (`+X% 🟢`, `-Y% 🔴`).
