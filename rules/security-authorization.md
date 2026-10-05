# Security & Authorization Invariants

All development across Terrakube API, Executor, and Client modules MUST strictly adhere to the following security invariants:

## 1. Fail Closed by Default (Zero Implicit Authorization)
* Every endpoint, service, or filter handling authorization MUST fail closed.
* If a security ticket, token, header, or query parameter (e.g., upload ticket, bearer token) is missing, empty, or malformed, the request MUST be immediately rejected with HTTP `401 Unauthorized` or `403 Forbidden`.
* NEVER use fallback default secrets (e.g., hardcoded fallback strings) when cryptographic secrets or signing keys are unconfigured. The service must throw an exception and fail to start or fail closed.
* Security validation toggles (such as `blockPrivateNetworks` for SSRF protection) MUST default to `true` (secure default).

## 2. Context-Aware Scoping for Internal Tokens (`TerrakubeInternal`)
* NEVER use blanket token issuer checks like:
  ```java
  // ANTI-PATTERN: DO NOT DO THIS
  if ("TerrakubeInternal".equals(jwt.getTokenAttributes().get("iss"))) {
      return true;
  }
  ```
* Internal tokens are minted for diverse purposes (backend service clients, ephemeral job runners, CLI credentials).
* **Resource-Scoped Tokens:** If an internal token contains a resource claim (e.g., `workspaceId`, `jobId`):
  1. It MUST ONLY be authorized for actions targeting that specific resource ID (`token.workspaceId.equals(requestedWorkspaceId)`).
  2. It MUST NEVER be granted global superuser status, blanket group membership (`isServiceMember`), or access to other workspaces.
  3. It MUST NEVER pass administrative team checks (e.g. `TeamApproveJob`), VCS secret checks (`ReadAccessToken`), or SSH private key checks (`ReadPrivateKey`).
* **Service Tokens:** Only system internal tokens *without* resource scoping (or with explicitly verified backend client credentials) may perform cross-tenant system operations.

## 3. Stateless Public Endpoints & Anti-Harvesting Guardrails
* Unauthenticated endpoints (such as CLI browser redirect bridges under `/app/*`) MUST be purely stateless forwarders:
  1. MUST NOT execute database queries or repository lookups.
  2. MUST NOT disclose internal database UUIDs in URL paths, headers, or payloads.
  3. MUST NOT leak existence oracles (e.g., returning 404 for non-existent entities and 401/403 for existing ones).
* Error responses for unauthorized or invalid resource resolution in the UI MUST use unified error screens (e.g., HTTP 403 "Not Authorized or resource does not exist") to prevent enumerating valid tenant identifiers.

## 4. Child Process & Worker Environment Sanitization
* When spawning external processes (Terraform CLI, OpenTofu, bash scripts, OPA engines) via `ProcessBuilder` or process launchers:
  1. Child processes MUST NOT inherit sensitive host environment variables.
  2. Explicitly sanitize or clear the process environment map before invocation, stripping signing secrets (`InternalSecret`, `secretKey`), database credentials, Redis passwords, and cloud provider root secrets.
  3. Untrusted job code (Terraform providers, external data sources, custom modules) must never be able to inspect host JVM environment variables or `/proc/$PID/environ`.

## 5. Ephemeral Credential Lifetimes & Guaranteed Teardown
* Tokens written to disk or container filesystems (such as `~/.terraformrc` credentials) MUST have short expirations strictly matching the job duration (e.g., 60 minutes or `jobTimeout`), NEVER long durations like 30 days.
* All generated JWTs MUST include a unique `jti` claim to allow immediate revocation in Redis upon job cancellation or completion.
* Credential files created during workspace setup MUST be scrubbed (overwritten and deleted) inside guaranteed `finally` teardown blocks upon job completion or failure.
