---
name: terraform-spring-boot-starter
description: Guidelines, compilation instructions, Javadoc conventions, Terraform/OpenTofu version resolution, and execution patterns (plan, apply, destroy, variables, environment variables) for `terraform-spring-boot`.
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

## 3. Terraform & OpenTofu Version Resolution (`TerraformDownloader.java`)

`TerraformDownloader` provides methods for constraint resolution, binary path generation, and binary downloading:

* **`resolveTerraformVersion(String terraformVersion)`**: Resolves a Terraform version constraint (e.g. `~>1.5`, `>=1.4 <2.0`) to a concrete version string (e.g. `1.5.7`) without downloading binaries.
* **`resolveTofuVersion(String tofuVersion)`**: Resolves an OpenTofu version constraint to a concrete version string without downloading binaries.
* **`getTerraformBinaryPath(String resolvedVersion, boolean tofu)`**: Returns the expected local filesystem path where the Terraform (`tofu=false`) or OpenTofu (`tofu=true`) executable will reside after download/extraction.
* **`downloadTerraformVersion(String terraformVersion)`**: Downloads and extracts the requested Terraform binary if not already cached locally, returning the absolute binary path.
* **`downloadTofuVersion(String tofuVersion)`**: Downloads and extracts the requested OpenTofu binary if not already cached locally, returning the absolute binary path.

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
- [ ] Build `TerraformProcessData` with explicit `workingDirectory`, `terraformVersion`, and `tofu` flag.
- [ ] Pass environment variables via `.terraformEnvironmentVariable(key, value)` and variables via `.terraformVariable(key, value)`.
- [ ] Always handle `CompletableFuture` response and attach stdout/stderr `outputListener` / `errorListener`.
