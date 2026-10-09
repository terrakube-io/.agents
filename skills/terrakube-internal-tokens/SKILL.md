---
name: terrakube-internal-tokens
description: Internal token classification, microservice authorization for Executor and Registry, granular Elide security checks, and runtime environment hardening best practices.
---

# Terrakube Internal Token Architecture & Microservice Access Control Guide

This guide details the internal token classification taxonomy, microservice permissions for `terrakube/executor` and `terrakube/registry`, Elide JSON:API security checks, and security best practices established in the TK-01 remediation.

---

## 1. Token Classification Taxonomy (`InternalTokenType`)

All JWT tokens handled by Terrakube API are categorized using `io.terrakube.api.plugin.security.token.InternalTokenClassifier`:

```java
public enum InternalTokenType {
    NOT_INTERNAL,       // Real user identities: Dex OIDC, PAT, or Federated tokens
    WORKSPACE_EXECUTOR, // Per-job runner tokens (iss=TerrakubeInternal, carries workspaceId)
    EXECUTOR_SERVICE,   // Executor backend daemon (iss=TerrakubeInternal, sub=TerrakubeInternal (EXECUTOR), no workspaceId)
    REGISTRY_SERVICE,   // Registry backend daemon (iss=TerrakubeInternal, sub=TerrakubeInternal (REGISTRY), no workspaceId)
    UNKNOWN_INTERNAL    // Legacy or forged internal token (missing/unrecognized subject without workspaceId) -> FAIL CLOSED
}
```

### Classification Logic (`InternalTokenClassifier.classify(Map<String, Object> claims)`)

1. **External / User Tokens (`NOT_INTERNAL`)**:
   - If `claims == null` or `iss != "TerrakubeInternal"`.
   - Represents human users, Personal Access Tokens (PAT), or Federated OIDC (GitHub Actions).
   - Only `NOT_INTERNAL` tokens are evaluated for user `groups` and platform administrator (`instanceOwner`).

2. **Workspace Runner Tokens (`WORKSPACE_EXECUTOR`)**:
   - `iss == "TerrakubeInternal"` AND contains non-empty `workspaceId` claim.
   - Carries `organizationId`, `workspaceId`, `jobId`, `stepId`, and deterministic `jti = jobId-stepId`.
   - Ephemeral token passed to step runners (Terraform CLI, bash scripts, Groovy runners).

3. **Backend Service Tokens (`EXECUTOR_SERVICE` / `REGISTRY_SERVICE`)**:
   - `iss == "TerrakubeInternal"` AND does NOT contain `workspaceId`.
   - Inspects `sub` claim:
     - `sub == "TerrakubeInternal (EXECUTOR)"` -> `EXECUTOR_SERVICE`
     - `sub == "TerrakubeInternal (REGISTRY)"` -> `REGISTRY_SERVICE`

4. **Fail-Closed Unrecognized Tokens (`UNKNOWN_INTERNAL`)**:
   - `iss == "TerrakubeInternal"` without `workspaceId` and with legacy subject (e.g. `TerrakubeInternal (TOKEN)`) or unknown subject.
   - Categorized as `UNKNOWN_INTERNAL` with **zero permissions** across all endpoints.

---

## 2. Access Controls & Revocation of Blanket Trust

### Blanket Group Membership Revocation (`DexGroupServiceImpl`)
Internal machine tokens (`WORKSPACE_EXECUTOR`, `EXECUTOR_SERVICE`, `REGISTRY_SERVICE`, `UNKNOWN_INTERNAL`) MUST NEVER be granted user group membership:

```java
@Override
public boolean isServiceMember(User user, String group) {
    JwtAuthenticationToken principal = (JwtAuthenticationToken) user.getPrincipal();
    Map<String, Object> tokenAttributes = principal.getTokenAttributes();
    if (InternalTokenClassifier.classify(tokenAttributes) != InternalTokenType.NOT_INTERNAL) {
        log.debug("Internal machine token ({}) cannot have user service group membership for group {}",
                tokenAttributes.get("sub"), group);
        return false; // ALWAYS FALSE for internal machine tokens
    }
    return isMember(user, group);
}

@Override
public Set<String> getEffectiveGroups(User user) {
    JwtAuthenticationToken principal = (JwtAuthenticationToken) user.getPrincipal();
    Map<String, Object> tokenAttributes = principal.getTokenAttributes();
    if (InternalTokenClassifier.classify(tokenAttributes) != InternalTokenType.NOT_INTERNAL) {
        return Collections.emptySet(); // Machine tokens have NO effective user groups
    }
    ...
}
```

### Superuser Privilege Lockdown (`IsSuperUser`)
Machine tokens cannot claim platform superuser status:

```java
@Override
public boolean ok(User user) {
    if (user != null && user.getPrincipal() instanceof JwtAuthenticationToken jwt) {
        Map<String, Object> claims = jwt.getTokenAttributes();
        if (InternalTokenClassifier.classify(claims) != InternalTokenType.NOT_INTERNAL) {
            return false; // UNCONDITIONALLY DENIED
        }
    }
    ...
}
```

---

## 3. Dedicated Microservice Elide Security Checks

Instead of granting blanket superuser or group permissions to backend microservices, use dedicated Elide `@SecurityCheck` components:

### 1. `IsExecutorService` (`user is an executor service`)
```java
@SecurityCheck(IsExecutorService.RULE)
public class IsExecutorService extends UserCheck {
    public static final String RULE = "user is an executor service";

    @Override
    public boolean ok(User user) {
        if (user != null && user.getPrincipal() instanceof JwtAuthenticationToken jwt) {
            return InternalTokenClassifier.classify(jwt.getTokenAttributes()) == InternalTokenType.EXECUTOR_SERVICE;
        }
        return false;
    }
}
```

### 2. `IsRegistryService` (`user is a registry service`)
```java
@SecurityCheck(IsRegistryService.RULE)
public class IsRegistryService extends UserCheck {
    public static final String RULE = "user is a registry service";

    @Override
    public boolean ok(User user) {
        if (user != null && user.getPrincipal() instanceof JwtAuthenticationToken jwt) {
            return InternalTokenClassifier.classify(jwt.getTokenAttributes()) == InternalTokenType.REGISTRY_SERVICE;
        }
        return false;
    }
}
```

### 3. `ExecutorManageHistory` (`executor manage history`)
Permits creating state history records exclusively for the Executor daemon:
```java
@SecurityCheck(ExecutorManageHistory.RULE)
public class ExecutorManageHistory extends UserCheck {
    public static final String RULE = "executor manage history";

    @Override
    public boolean ok(User user) {
        if (user != null && user.getPrincipal() instanceof JwtAuthenticationToken jwt) {
            return InternalTokenClassifier.classify(jwt.getTokenAttributes()) == InternalTokenType.EXECUTOR_SERVICE;
        }
        return false;
    }
}
```

---

## 4. Granular Workspace Runner Security Checks (`io.terrakube.api.rs.checks.executor.*`)

Ephemeral runner tokens (`WORKSPACE_EXECUTOR`) are strictly bounded by exact entity UUID matching:

| Check Class | Elide Rule String | Validated Claims vs Entity Attributes |
|---|---|---|
| `ExecutorReadOrganization` | `executor read organization` | `token.organizationId == org.id` |
| `ExecutorReadWorkspace` | `executor read workspace` | `token.workspaceId == workspace.id` |
| `ExecutorReadJob` | `executor read job` | `token.workspaceId == job.workspace.id && token.jobId == job.id` |
| `ExecutorManageJob` | `executor manage job` | Same as read, validates status mutation |
| `ExecutorReadJobStep` | `executor read job step` | `token.jobId == step.job.id && token.stepId == step.id` |
| `ExecutorManageJobStep` | `executor manage job step` | Same as read, allows step status update |
| `ExecutorReadHistory` | `executor read history` | `token.workspaceId == history.workspace.id` |

### Sensitive Resource Invariants
Runner tokens are explicitly rejected from reading:
- **SSH Private Keys (`ReadPrivateKey`)**: Rejects if `iss == "TerrakubeInternal"`.
- **VCS Access Tokens (`ReadAccessToken`)**: Rejects if `iss == "TerrakubeInternal"`.
- **Cross-Workspace History or State**: Rejects if entity workspace ID does not match token claim.

---

## 5. Entity Field Protection (`@Exclude`)

Any JPA `@Entity` field that contains execution logic, workflow definitions, or security-sensitive metadata that MUST NOT be modified or injected via Elide JSON:API or GraphQL MUST be annotated with `@Exclude`:

```java
// io.terrakube.api.rs.job.Job
@Exclude
@Column(name = "tcl")
private String tcl;
```

> [!IMPORTANT]
> Excluding `Job.tcl` prevents malicious actors from injecting custom workflow YAML templates during Job creation or update via `/api/v1/job` or GraphQL mutations.

---

## 6. Runtime Environment Sanitization & Out-of-Process Execution

### In-Process Sandboxing Anti-Pattern
In modern Java (Java 21+), `java.lang.SecurityManager` is permanently removed. AST customizers (`SecureASTCustomizer`) cannot prevent reflection attacks:
```groovy
// Bypasses AST denylists
Class.forName('java.lang.System').getMethod('getenv', [String.class] as Class[]).invoke(null, ['InternalSecret'])
```

### Out-of-Process Execution Invariant
1. **Never execute untrusted user code in the host Spring Boot JVM**:
   - Route Groovy execution through `GroovyProcessRunner` using `ProcessLauncher` to spawn a dedicated child JVM process (`$JAVA_HOME/bin/java`).
   - Route Bash execution through `BashEngine` / `ProcessLauncher`.
2. **Purge Ambient Host Secrets**:
   - Clear environment maps (`ProcessBuilder.environment().clear()`).
   - Strip all platform secrets: `InternalSecret`, `TerrakubeRedisPassword`, cloud credentials (`AwsTerraform*`, `AzureTerraform*`, `GcpTerraform*`), database URLs/passwords.
   - Whitelist only standard OS variables (`PATH`, `HOME`, `JAVA_HOME`, `TMPDIR`, `USER`, `SHELL`, `LANG`) and explicit step inputs.
3. **No Spring Beans in Script Bindings**:
   - Never inject `TerrakubeClient`, `ApplicationContext`, or database repositories into script bindings.
   - Pass only execution metadata (`terrakubeApi`, `terrakubeToken`, `jobId`, `stepId`, `workingDirectory`).
4. **Ephemeral Credentials Teardown**:
   - Tokens written to disk (`~/.terraformrc`) must have lifetimes strictly matching step execution (e.g. 60m).
   - Scrub and delete credential files in guaranteed `finally` teardown blocks.

---

## 7. Registry Security Best Practices

1. **Authorization on Metadata & Tarball Downloads**:
   - `ModuleWebServiceImpl`: Always validate module read permissions through `ModuleAuthorizationService` before returning module versions, archive downloads, or details. Return HTTP 403 when unauthorized.
2. **Cache Isolation**:
   - In `ModuleInspectorService`, cache keys MUST include organization ID to prevent cross-tenant cache collisions across organizations.
3. **Service Token Configuration**:
   - The registry client MUST use `sub="TerrakubeInternal (REGISTRY)"` when interacting with the API.
