---
name: terraform-spring-boot-starter
description: Guidelines, compilation instructions, Javadoc conventions, Terraform/OpenTofu version resolution, execution patterns (plan, apply, destroy, variables, environment variables), and devcontainer live testing workflows for `terraform-spring-boot`.
---

# Terraform Spring Boot Starter Development & Operations Guide

`terraform-spring-boot` is the Java client library, autoconfiguration, and Spring Boot starter for downloading Terraform/OpenTofu binaries, configuring process environment variables and parameters, and executing Terraform commands asynchronously (`init`, `plan`, `apply`, `destroy`, `show`, `state`).

---

## 1. Repository & Module Layout

```
terraform-spring-boot/
├── terraform-client/                        # Core Java client for executing Terraform/Tofu commands & downloads
│   └── src/main/java/io/terrakube/terraform/
│       ├── TerraformClient.java             # Main client entrypoint for init, plan, apply, destroy, etc.
│       ├── TerraformDownloader.java         # Version resolution & binary downloader
│       ├── TerraformProcessData.java        # Process configuration (working dir, vars, env, options)
│       ├── ProcessLauncher.java             # Process launcher with stdout/stderr stream listeners
│       └── TerraformCommand.java            # Command enum (init, plan, apply, destroy, show, state)
├── terraform-spring-boot-autoconfigure/     # Spring Boot AutoConfiguration
├── terraform-spring-boot-starter/           # Starter dependency aggregator
└── terraform-spring-boot-samples/           # Usage samples
```

---

## 2. Compilation & Build Guidelines

### 2.1 Maven Build Command
To compile, run unit tests, and install `terraform-spring-boot` artifacts into local Maven repository without GPG signing or Javadoc generation:
```bash
mvn install -Dgpg.skip -Dmaven.javadoc.skip=true
```

### 2.2 JDK Requirement & SDKMAN Switching
* **Java Version**: Always build `terraform-spring-boot` using **JDK 21** (e.g. `21.0.9-librca`).
* **SDKMAN Switching**:
  ```bash
  # Switch JDK in current shell context:
  sdk use java 21.0.9-librca
  
  # Or run Maven command inline with explicit JDK 21 environment:
  JAVA_HOME=/home/user/.sdkman/candidates/java/21.0.9-librca PATH=/home/user/.sdkman/candidates/java/21.0.9-librca/bin:$PATH mvn install -DskipTests -Dgpg.skip -Dmaven.javadoc.skip=true
  ```
* **Lombok Compatibility Note**: Do NOT build `terraform-spring-boot` using JDK 25. JDK 25 internal `javac` AST changes cause Lombok annotation processing to fail during `terraform-spring-boot` compilation (`cannot find symbol` errors for `@Getter`, `@Setter`, `@Slf4j`, etc.).

### 2.3 Javadoc Comment Formatting
* **Escape Angle Brackets**: Wrap version constraint operators (`<`, `>`) in Javadoc comments using `{@code ...}`.
  - **Correct**: `* Resolve a Terraform version constraint (e.g. {@code "~>1.5"}, {@code ">=1.4 <2.0"})`
  - **Incorrect**: `* Resolve a Terraform version constraint (e.g. ">=1.4 <2.0")`
* Raw `<` symbols cause `maven-javadoc-plugin` to fail with `malformed HTML` errors during `mvn install`.

---

## 3. Terraform & OpenTofu Version Resolution & Binary Caching (`TerraformDownloader.java`)

### 3.1 Filesystem Directory Layout
All binaries and archives reside under the user home directory (`~/.terraform-spring-boot/`):
* **Terraform Archives:** `~/.terraform-spring-boot/download/terraform_<version>_<os>_<arch>.zip`
* **OpenTofu Archives:** `~/.terraform-spring-boot/download/tofu/tofu_<version>_<os>_<arch>.zip`
* **Terraform Executable:** `~/.terraform-spring-boot/terraform/<version>/terraform`
* **OpenTofu Executable:** `~/.terraform-spring-boot/tofu/<version>/tofu`

### 3.2 Binary Download & Caching Lifecycle
`TerraformDownloader` implements binary-level caching and lazy release resolution:

1. **Exact-Version Binary Check (`findInstalledBinary`):**
   * If a static semver version (e.g. `1.5.7`, `1.8.2`) is passed, `findInstalledBinary` checks if `~/.terraform-spring-boot/<product>/<version>/<executable>` already exists and is executable (`isUsableBinary`).
   * If usable, it returns the absolute path **immediately**, making **zero network requests** and bypassing release list fetching.
2. **Lazy Release List Resolution & Caching (`RELEASES_CACHE`):**
   * Release lists are not fetched in the constructor; they are requested on-demand when constraint resolution or archive downloads are needed.
   * Product isolation: resolving Terraform only queries HashiCorp (`index.json`); resolving OpenTofu only queries GitHub Releases API.
   * Cached in `RELEASES_CACHE` with a 30-minute TTL (`RELEASES_CACHE_TTL = Duration.ofMinutes(30)`) per URL with double-checked locking.
   * Fallback on failure: on network or rate-limiting (HTTP 403/429) errors during a refresh, previously cached releases are reused for 1 minute (`RELEASES_RETRY_AFTER_FAILURE = Duration.ofMinutes(1)`).
3. **Archive Reuse & Corrupt Archive Self-Healing:**
   * If the binary is missing but the `.zip` archive exists in `download/`, extraction is attempted before downloading.
   * If extraction or download fails, the corrupted zip is deleted via `FileUtils.deleteQuietly(zipFile)` and re-downloaded cleanly.

### 3.3 Core Methods in `TerraformDownloader`
* **`resolveTerraformVersion(String terraformVersion)`**: Resolves a Terraform version constraint (e.g. `~>1.5`, `>=1.4 <2.0`) to a concrete version string (e.g. `1.5.7`) without downloading binaries.
* **`resolveTofuVersion(String tofuVersion)`**: Resolves an OpenTofu version constraint to a concrete version string without downloading binaries.
* **`getTerraformBinaryPath(String resolvedVersion, boolean tofu)`**: Returns the expected local filesystem path where the Terraform (`tofu=false`) or OpenTofu (`tofu=true`) executable will reside after download/extraction.
* **`downloadTerraformVersion(String terraformVersion)`**: Returns the existing binary path if `findInstalledBinary` hits; otherwise resolves version, downloads, unzips, and returns the binary path.
* **`downloadTofuVersion(String tofuVersion)`**: Returns the existing binary path if `findInstalledBinary` hits; otherwise resolves version, downloads, unzips, and returns the binary path.

### 3.4 Terrakube Executor Binary Cache Flow
In `terrakube/executor`, the binary lifecycle operates across `TerraformExecutorServiceImpl` and `BinaryCacheRecoveryService`:
1. **Pre-Resolution:** `TerraformExecutorServiceImpl.resolveTerraformVersion()` calls `resolveTerraformVersion` or `resolveTofuVersion` to normalize constraints (e.g. `>= 1.5.0` -> `1.5.7`) and store the concrete version on `terraformJob`.
2. **Cloud Storage Restore:** `ensureBinaryCached()` checks local disk or restores the executable binary directly from cloud storage (S3/Azure/GCP via `terraformState.downloadTerraformBinary`) to `~/.terraform-spring-boot/<product>/<version>/<executable>`.
3. **Cache Hit in Downloader:** When `terraformClient.init()` runs, `downloadTerraformVersion` detects the pre-restored binary via `findInstalledBinary()` and proceeds without downloading or querying release lists.
4. **Cloud Storage Save:** If the binary was freshly downloaded from upstream, `saveBinaryToCache()` uploads the executable from `getTerraformBinaryPath()` to cloud storage for other pods.
5. **Execution Concurrency Invariant:** Terrakube enforces 1 job per pod (`ExecutorCapacityGate`), meaning concurrent job file downloads on the same pod do not occur.


---

## 4. Executing Terraform Operations (`TerraformClient` & `TerraformProcessData`)

### 4.1 Configuring `TerraformProcessData`

`TerraformProcessData` holds all parameters, environment variables, working directory, and operational flags for a command execution:

```java
TerraformProcessData processData = TerraformProcessData.builder()
        .terraformVersion("1.5.7")                    // Concrete version or constraint
        .workingDirectory(new File("/path/to/tf"))   // Directory containing .tf files
        .tofu(false)                                  // set true for OpenTofu, false for Terraform
        .terraformBackendConfigFileName("backend.tfvars") // Optional backend config file name (-backend-config=)
        .varFileName("terraform.tfvars")              // Optional tfvars file name (-var-file)
        .refresh(true)                                // Default true
        .refreshOnly(false)                           // Set true for -refresh-only
        .terraformVariable("key1", "value1")          // Terraform variable (--var key1=value1)
        .terraformEnvironmentVariable("AWS_ACCESS_KEY_ID", "AKIA...") // Process environment variable
        .terraformEnvironmentVariable("TF_VAR_custom", "custom_val")
        .pidIsolation(PidIsolation.BEST_EFFORT)       // Linux PID namespace isolation (OFF, BEST_EFFORT, REQUIRED)
        .cleanEnvironment(true)                       // Sanitize environment, preserving safe allowlist
        .build();
```

### 4.2 Executing Operations with Output & Error Listeners

`TerraformClient` methods execute commands asynchronously returning `CompletableFuture`:

```java
Consumer<String> outputListener = line -> log.info("[STDOUT] {}", line);
Consumer<String> errorListener = line -> log.error("[STDERR] {}", line);

TerraformClient client = TerraformClient.builder().build();

// 1. Initialize backend and modules
CompletableFuture<Boolean> initSuccess = client.init(processData, outputListener, errorListener);

// 2. Run Plan (generates terraformLibrary.tfPlan)
CompletableFuture<Boolean> planSuccess = client.plan(processData, outputListener, errorListener);

// 2b. Run Plan with detailed exit code (0 = no changes, 2 = changes present, 1 = error)
CompletableFuture<Integer> exitCode = client.planDetailExitCode(processData, outputListener, errorListener);

// 3. Apply changes (uses auto-approve)
CompletableFuture<Boolean> applySuccess = client.apply(processData, outputListener, errorListener);

// 4. Destroy resources
CompletableFuture<Boolean> destroySuccess = client.destroy(processData, outputListener, errorListener);

// 5. Read Plan JSON output
CompletableFuture<Boolean> jsonSuccess = client.showPlanJson(processData, outputListener, errorListener);
```

---

## 5. Development & Execution Checklist

- [ ] Build under **JDK 21** using `mvn install -Dgpg.skip`.
- [ ] Ensure Javadoc comments escape `<` and `>` operators using `{@code ...}` tags.
- [ ] Use `resolveTerraformVersion` / `resolveTofuVersion` when resolving constraints without downloading binaries.
- [ ] Pre-populate or restore binaries at `~/.terraform-spring-boot/<product>/<version>/<executable>` to hit `findInstalledBinary` zero-network fast path.
- [ ] Build `TerraformProcessData` with explicit `workingDirectory`, `terraformVersion`, and `tofu` flag.
- [ ] Pass environment variables via `.terraformEnvironmentVariable(key, value)` and variables via `.terraformVariable(key, value)`.
- [ ] Configure `pidIsolation` (`PidIsolation.BEST_EFFORT` / `REQUIRED`) and `cleanEnvironment` when hardening against rogue `local-exec` scripts or provider plugins.
- [ ] Always handle `CompletableFuture` response and attach stdout/stderr `outputListener` / `errorListener`.

---

## 6. Upgrading & Testing in Devcontainer (Live Environment)

When developing features or bug fixes in `terraform-spring-boot`, test them against the running Terrakube live stack inside Docker Compose / Devcontainer (`terrakube_devcontainer-terrakube-dev-1`).

### Step 1: Identify Expected Version in Terrakube Executor
Check the version declared by `terrakube/executor/pom.xml`:
```xml
<terraform-spring-boot-starter.version>1.8.0</terraform-spring-boot-starter.version>
```
Ensure you build and install the exact version expected by the consumer branch (e.g. `1.8.0` on `main`, `1.7.0` on release branches).

### Step 2: Build & Install Artifacts with Explicit Revision
Run `mvn install` with `-Drevision=<VERSION>` using JDK 21:
```bash
# Inside the devcontainer:
docker exec terrakube_devcontainer-terrakube-dev-1 bash -c "cd /workspaces/terraform-spring-boot && mvn install -Drevision=1.8.0 -DskipTests -Dgpg.skip -Dmaven.javadoc.skip=true"
```
Or if building on the host:
```bash
cd terraform-spring-boot
JAVA_HOME=/home/user/.sdkman/candidates/java/21.0.9-librca mvn install -Drevision=1.8.0 -DskipTests -Dgpg.skip -Dmaven.javadoc.skip=true
```

### Step 3: Critical Gotcha — Expand `${revision}` in Installed POMs
`terraform-spring-boot-starter` and `terraform-spring-boot-autoconfigure` use `<version>${revision}</version>` for internal dependencies. Because installed POMs retain the literal string `${revision}`, consumer builds (`terrakube/executor`) will fail to resolve transitive dependencies unless expanded:
```bash
docker exec terrakube_devcontainer-terrakube-dev-1 bash -c "
  sed -i 's/\${revision}/1.8.0/g' /root/.m2/repository/io/terrakube/terraform/terraform-spring-boot-starter/1.8.0/terraform-spring-boot-starter-1.8.0.pom
  sed -i 's/\${revision}/1.8.0/g' /root/.m2/repository/io/terrakube/terraform/terraform-spring-boot-autoconfigure/1.8.0/terraform-spring-boot-autoconfigure-1.8.0.pom
"
```

### Step 4: Sync Dual `.m2` Directories (`root` & `vscode`)
The devcontainer maintains two Maven caches:
- `/root/.m2/repository` (used by `docker exec` commands)
- `/home/vscode/.m2/repository` (used when launching applications via VS Code debugger as user `vscode`)

Sync artifacts, fix permissions, and delete `_remote.repositories` (so Maven never attempts to overwrite local snapshots with Central releases):
```bash
docker exec terrakube_devcontainer-terrakube-dev-1 bash -c "
  mkdir -p /home/vscode/.m2/repository/io/terrakube/terraform
  cp -r /root/.m2/repository/io/terrakube/terraform/* /home/vscode/.m2/repository/io/terrakube/terraform/
  chown -R vscode:vscode /home/vscode/.m2
  find /home/vscode/.m2/repository/io/terrakube/terraform /root/.m2/repository/io/terrakube/terraform -name '_remote.repositories' -delete
"
```

### Step 5: Verify Executor Compilation
Verify that `terrakube/executor` compiles cleanly against the newly installed library jars:
```bash
docker exec terrakube_devcontainer-terrakube-dev-1 bash -c "cd /workspaces/terrakube/executor && mvn test-compile"
```

### Step 6: Configure Environment Flags in `.envExecutor`
Update `scripts/setupDevelopmentEnvironment.sh` to configure and export any new executor flags into `.envExecutor`. For example, for PID isolation and clean environment:
```bash
IO_TERRAKUBE_TERRAFORM_FLAGS_PID_ISOLATION="best-effort"
IO_TERRAKUBE_TERRAFORM_FLAGS_CLEAN_ENVIRONMENT="true"
```
Re-generate the devcontainer environment configuration:
```bash
docker exec terrakube_devcontainer-terrakube-dev-1 bash -c "cd /workspaces/terrakube && ./scripts/setupDevelopmentEnvironment.sh -d POSTGRESQL -s MINIO"
```

### Step 7: Launch & Validate in Live Stack
1. In VS Code inside the devcontainer, launch **"Run Executor"** from the Run & Debug view (which injects `${workspaceFolder}/.envExecutor`).
2. Navigate to the Terrakube UI (`https://terrakube.localhost`), sign in (`admin@example.com` / `admin`), and trigger a workspace run (`Plan` / `Apply`).
3. Monitor terminal/debug console logs to confirm that the upgraded `terraform-spring-boot` library is active and behaving as expected.


