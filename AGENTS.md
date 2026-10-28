# Terrakube Architecture & Coding Rules

## 1. Elide API Entity Conventions (`io.terrakube.api.rs`)
* All JSON:API resources must be defined under `io.terrakube.api.rs` (or domain sub-packages).
* Every entity MUST have `@Include`, `@Entity(name = "...")`, Lombok `@Getter`/`@Setter`, and explicit Elide security permission annotations (`@ReadPermission`, `@CreatePermission`, `@UpdatePermission`, `@DeletePermission`).
* Primary key fields (`id`) MUST be typed `UUID` using `@JdbcTypeCode(Types.VARCHAR)` and `@GeneratedValue(strategy = GenerationType.UUID)` (Note: `@Convert(converter = IdConverter.class)` is no longer necessary with Elide 7.2+).
* All entity read, list, filter, sorting, and pagination queries MUST go through Elide's native GraphQL endpoint (`/graphql/api/v1`) or JSON:API endpoint (`/api/v1`).
* NEVER create custom Spring GraphQL controllers (`@Controller`, `@QueryMapping`) or manual JPA Criteria queries for entity CRUD, listing, or pagination. Spring GraphQL (`/graphql`, `/subscriptions`) is strictly reserved for WebSocket subscriptions.

## 2. Liquibase Database Migrations Mandate
* NEVER modify a JPA `@Entity` field, table name, column definition, or relationship constraint without adding a corresponding Liquibase migration file.
* Migration files must be created in `terrakube/api/src/main/resources/db/changelog/local/changelog-<version>-<feature>.xml`.
* Every new migration file MUST be registered in `terrakube/api/src/main/resources/db/changelog/changelog.xml` using `<include file="/db/changelog/local/changelog-....xml"/>`.

## 3. Go client library (`terrakube-go`)
* The `terrakube-go` client library must be developed in `terrakube-go`.
* The `terrakube-go` client library must follow the following guidelines:
  - Guidelines: .agents/skills/terrakube-go/SKILL.md

## 4. Terrakube CLI (`terrakube-cli`)
* The `terrakube-cli` CLI application must be developed in `terrakube-cli`.
* All new CLI resources MUST use the `resource.Register[T]()` framework pattern. Do NOT create hand-written Cobra subcommands.
* The `terrakube-cli` must follow the following guidelines:
  - Guidelines: .agents/skills/terrakube-cli/SKILL.md

## 5. Terrakube Spring Boot Starter (`terrakube-spring-boot-starter`)
* The `terrakube-spring-boot-starter` client library must be developed in `terrakube-spring-boot-starter/terrakube-client`.
* Extending the client (adding models, endpoints, or tests) must follow the guidelines:
  - Guidelines: .agents/skills/terrakube-spring-boot-starter/SKILL.md

## 6. Git Commands
* All git commands must follow the following guidelines:
  - Guidelines: rules/git-commands.md

## 7. Ephemeral Executor (`io.terrakube.api.plugin.scheduler.job.tcl.executor.ephemeral`)
* The Ephemeral Executor builds Kubernetes `batch/v1` Jobs via Fabric8 Kubernetes client and Server-Side Apply.
* Extending or modifying ephemeral executor capabilities must follow the guidelines:
  - Guidelines: .agents/skills/terrakube-ephemeral-executor/SKILL.md

## 8. Structured Output (Plan & Apply)
* Structured output generation, real-time JSON streaming, post-hoc state resolution, and sensitivity sanitization for Plan and Apply operations are managed in `io.terrakube.executor.service.terraform`.
* Guidelines: .agents/skills/terrakube-structured-output/SKILL.md

## 9. Registry Performance Benchmarking (`terrakube/benchmarks/registry`)
* Performance benchmarks and A/B container image evaluations for `terrakube-registry` are automated in `terrakube/benchmarks/registry`.
* All benchmark executions must adhere to the concurrency staging (`5, 10, 25` workers), fixture provisioning, and OpenTelemetry / Prometheus PromQL metric conventions.
* Guidelines: .agents/skills/terrakube-registry-performance/SKILL.md

## 10. Modular Storage Architecture & Container Image Splitting (`terrakube/storage`)
* Cloud storage implementations (AWS S3, Azure Blob, GCP Cloud Storage, Local) are maintained as independent Spring Boot 3 starters under `terrakube/storage`.
* Consuming components (`registry`, `api`, `executor`) must use Maven profiles (`-Paws`, `-Pazure`, `-Pgcp`, default `-Pall`) and Spring Boot 3 `@AutoConfiguration` to support split container images.
* Guidelines: .agents/skills/terrakube-modular-storage/SKILL.md

## 11. External HTTP Client & WebClient Conventions
* Singleton services that inject `WebClient.Builder` MUST call `.clone()` before applying per-request mutations (`.defaultHeader(...)`, `.defaultHeaders(...)`, `.filter(...)`, `.baseUrl(...)`, `.clientConnector(...)`).
* Static clients must be built once in the constructor and stored as immutable `private final WebClient webClient`.
* Guidelines: rules/http-client-conventions.md

## 12. Job Processing, FIFO Queue & Distributed Locking (`io.terrakube.api.plugin.scheduler`)
* Job dispatching, two-tier queueing (per-workspace serialization and cluster-level FIFO), and distributed locking are orchestrated in `io.terrakube.api.plugin.scheduler`.
* Any new scheduler trigger or lifecycle hook MUST respect the Redis execution lock (`job-execution-lock:<jobId>` with 90s TTL) and fail closed if Redis is unreachable.
* FIFO admission queries MUST use the guarded executable variants (`isJobNextInDispatchOrderExecutable` / `findNextDispatchableExecutableJobId`) to prevent head-of-line blocking from jobs with no remaining executable steps.
* The execution lock MUST be released immediately upon receiving HTTP 202 Accepted from the executor and MUST NOT be held during the long-running execution of Terraform.
* Guidelines: .agents/skills/terrakube-job-scheduler/SKILL.md

## 13. OPA Policy Governance & Fleet Architecture (`io.terrakube.api.plugin.policy`)
* Policy governance enforces guardrails via in-runner OPA evaluation on `plan.json` across three enforcement levels (`advisory`, `soft_mandatory`, `hard_mandatory`).
* PolicySets and PolicyAttachments MUST be managed via Terraform using `terrakube_policy_set` and `terrakube_policy_attachment`.
* Elide security check expressions for policies MUST use space-separated words without underscores (`team view policy set`, `team manage policy set`, `team override policy`, `team view policy exemption`, `team manage policy exemption`).
* Policy exemptions are scoped across three tiers (organization, project, workspace) and evaluated in-runner during policy execution by bypassing matching unexpired rule violations without altering blocking violations.
* Guidelines: .agents/skills/terrakube-opa-policy-governance/SKILL.md

## 14. Notification System & Webhook Outbox Architecture (`io.terrakube.api.plugin.notification`)
* All outbound notifications (Slack, Microsoft Teams, Webhooks) MUST use the Transactional Outbox pattern (`notification_outbox`).
* Network HTTP delivery calls MUST execute outside of database transactions to prevent connection pool exhaustion.
* Database fields storing payloads or errors (`payload`, `last_error`) MUST use plain `@Column` mapped to PostgreSQL `text`, NEVER `@Lob`.
* Entity relationships traversed via Elide GraphQL (such as `NotificationConfiguration.workspace`) MUST use `FetchType.EAGER` to prevent serialization of literal `"null"` proxy IDs.
* Destructive/mutating outbox operations invoked from Quartz jobs MUST be delegated to a dedicated Spring `@Service` bean (`NotificationOutboxTransactions`) because Quartz `Job` instances are not wrapped in Spring AOP transactional proxies.
* Destination URLs MUST be validated using `DestinationUrlValidator` to prevent SSRF attacks against internal network ranges.
* Signing secrets MUST remain write-only using `@ReadPermission(expression = "read notification signing secret")`.
* Guidelines: .agents/skills/terrakube-notifications/SKILL.md

## 15. Local Devcontainer & Database Debugging (`terrakube.localhost` / `terrakube-api.localhost`)
* The local development environment uses Docker Compose (`terrakube/.devcontainer`).
* Local services are accessible at `*.localhost` (e.g. `https://terrakube.localhost`, `https://terrakube-api.localhost`, `https://terrakube-registry.localhost`, `https://terrakube-dex.localhost/dex`) resolving to `127.0.0.1` without `/etc/hosts` modifications.
* PostgreSQL is accessible at `postgresql-service:5432` inside Docker networks and `localhost:5432` on the host (`terrakube` / `terrakubepassword` / `terrakubedb`).
* Direct query access via `docker exec -it postgresql-service psql -U terrakube -d terrakubedb` or host `psql -h localhost -p 5432 -U terrakube -d terrakubedb`.
* Guidelines: .agents/skills/terrakube-devcontainer/SKILL.md

## 16. RBAC & Team Permission Architecture (`io.terrakube.api.rs.team`, `ProjectAccess`, `Access`)
* Terrakube implements a 3-tier RBAC model (`Team` at Organization level, `ProjectAccess` at Project level, and `Access` at Workspace level) evaluated via `RbacV2Service`.
* Predefined roles (`admin`, `write`, `plan`, `read`) supersede fine-grained boolean flags. The `custom` role (or null) falls back to evaluating individual boolean flags.
* Whenever creating or seeding `Team` records in Liquibase or tests, ALL 12 permission attributes MUST be populated:
  - `role`: `"admin"`, `"write"`, `"plan"`, `"read"`, or `"custom"`
  - `manage_state`, `manage_collection`, `manage_job`, `manage_workspace`, `manage_module`, `manage_provider`, `manage_vcs`, `manage_template`, `manage_policies`, `plan_job`, `approve_job`
* Administrative teams (e.g. `TERRAKUBE_ADMIN`) MUST have `role="admin"` and all boolean flags set to `true`.
* Guidelines: .agents/skills/terrakube-rbac-permissions/SKILL.md

## 17. Terrakube UI (`terrakube/ui`)
* The Terrakube UI application must be developed in `terrakube/ui` using `yarn`.
* Linting and formatting in CI strictly enforce ESLint and Prettier (`yarn lint:modules:check`, `yarn format:modules:check`). Run `yarn lint:modules` to auto-fix formatting issues.
* Unused variables and imports MUST be eliminated to satisfy `@typescript-eslint/no-unused-vars`.
* Always use non-interactive Jest execution (`yarn test --watchAll=false` or `CI=true yarn test`) when running tests in automated environments.
* Guidelines: .agents/skills/terrakube-ui/SKILL.md

## 18. Terrakube Terraform Provider (`terraform-provider-terrakube`)
* The Terraform Provider must be developed in `terraform-provider-terrakube`.
* Code formatting MUST strictly adhere to `gofmt -s -w .` before committing to satisfy CI `golangci-lint` formatting rules.
* Documentation in `docs/` is managed via `go generate ./...` (`tfplugindocs`). Any custom documentation must be placed in `templates/` rather than raw markdown files.
* New API attributes that may not be available on older Terrakube API versions must use pointer types with `json:",omitempty"` to preserve backward compatibility.
* Guidelines: .agents/skills/terraform-provider-terrakube/SKILL.md

## 19. Security & Authorization Invariants
* All security-critical code across API, Executor, and CLI modules must strictly adhere to the Security Invariants:
  - Fail closed by default on missing tickets, tokens, or security parameters (never allow missing input to bypass checks).
  - Classify all tokens via `InternalTokenClassifier` (`NOT_INTERNAL`, `WORKSPACE_EXECUTOR`, `EXECUTOR_SERVICE`, `REGISTRY_SERVICE`, `UNKNOWN_INTERNAL`).
  - Never grant blanket group membership (`isServiceMember` / `isMember`) or superuser status to any internal machine token.
  - Contextually scope `WORKSPACE_EXECUTOR` tokens: must strictly match target workspace/job/step UUIDs and must NEVER access cross-workspace state, private SSH keys, or VCS tokens.
  - Partition microservices with dedicated subjects (`EXECUTOR_SERVICE`, `REGISTRY_SERVICE`) and Elide checks (`IsExecutorService`, `IsRegistryService`).
  - Unauthenticated endpoints (e.g., CLI redirects) must be completely stateless with zero DB queries and zero internal UUID exposure.
  - Spawning dynamic scripts (Groovy, Bash) must execute out-of-process via child JVM/process runners with host secrets (`InternalSecret`, Redis passwords, cloud credentials) stripped.
  - Temporary credential files (`~/.terraformrc`) must use short-lived tokens with `jti` and be purged in post-execution `finally` blocks.
  - Entity fields containing workflow YAML or execution logic (e.g., `Job.tcl`) must use `@Exclude` to prevent external template injection.
  - Guidelines: rules/security-authorization.md

## 20. Workspace Dependency Graph & Run Triggers (`io.terrakube.api.rs.workspace.trigger`, `io.terrakube.api.plugin.scheduler.trigger`)
* Workspaces declare reactive upstream dependencies within an organization using `WorkspaceRunTrigger`.
* **Structural & Tenant Invariants**: Both source and destination workspaces (and any template override) MUST belong to the same Organization. Self-triggers are strictly forbidden at both application check and DB constraint levels (`chk_run_trigger_no_self`).
* **Asymmetric RBAC Permissions**:
  - Creating/updating triggers requires `canManage` on the destination workspace AND `canView` on the source workspace (`TeamManageWorkspaceTrigger`).
  - Deleting triggers requires `canManage` on the destination workspace ONLY (`TeamDeleteWorkspaceTrigger`).
  - Source and destination workspace relationship fields are immutable once set (`@UpdatePermission(expression = "user is a superuser")`).
* **Cycle Prevention**: All new or updated triggers MUST be validated for DAG acyclicity via `WorkspaceGraphValidationService` during Elide `PRECOMMIT`. Both enabled and disabled edges MUST be checked to prevent dormant cycles.
* **State Change Qualification**: Downstream runs MUST ONLY be dispatched if upstream steps completed a state-changing flow (`terraformApply`, `terraformDestroy`, `customScripts`) and produced a state delta (serial increase or MD5 change). Speculative plans and zero-change applies MUST NOT trigger downstream runs.
* **Transaction Isolation**: Downstream jobs MUST be persisted in independent transactions using `RunTriggerJobWriter` (`REQUIRES_NEW`) and committed before scheduling Quartz triggers to prevent worker visibility race conditions.
* Guidelines: .agents/skills/terrakube-run-triggers/SKILL.md

## 21. Terrakube Templates & Workflow Language (TCL) (`io.terrakube.api.plugin.scheduler.job.tcl`)
* Workflow templates are defined in YAML (`flow: [...]`) and execute sequential steps (`terraformPlan`, `terraformApply`, `terraformDestroy`, `customScripts`, `approval`).
* Commands execute using either `GROOVY` (isolated out-of-process JVM) or `BASH` (subshell).
* Groovy commands execute without ambient host secrets, receiving short-lived scoped JWT tokens and job context variables (`terrakubeApi`, `terrakubeToken`, `jobId`, `stepId`, `workingDirectory`, `terrakubeOutput`).
* Extensions in `.terrakube/toolsRepository` (e.g., `Context.groovy`) are automatically added to the Groovy classpath and enable sharing structured state and artifacts across steps via `/context/v1/${jobId}`.
* Cross-command state in Bash is persisted by appending key-value pairs to `$terrakubeEnv`.
* Guidelines: .agents/skills/terrakube-templates/SKILL.md

## 22. Internal Token Architecture & Microservice Access Controls (`io.terrakube.api.plugin.security.token`)
* Tokens are classified into 5 discrete tiers (`NOT_INTERNAL`, `WORKSPACE_EXECUTOR`, `EXECUTOR_SERVICE`, `REGISTRY_SERVICE`, `UNKNOWN_INTERNAL`).
* Internal machine tokens are barred from user group membership and platform superuser status.
* Microservice permissions for Executor and Registry are partitioned using dedicated Elide checks (`IsExecutorService`, `IsRegistryService`, `ExecutorManageHistory`).
* Groovy scripts execute out-of-process with scrubbed environments (`GroovyProcessRunner`), neutralizing reflection attacks.
* Sensitive entity fields are excluded from external Elide manipulation (`@Exclude` on `Job.tcl`).
* Guidelines: .agents/skills/terrakube-internal-tokens/SKILL.md

## 23. Terraform CLI-Driven Workflow & Remote Execution (`/remote/tfe/v2/*`)
* Terrakube implements the HashiCorp Terraform Cloud / Enterprise Remote Execution protocol.
* CLI-supplied variables (`-var`, `--var`, `-var-file`) are parsed strictly as HCL literals via `CliRunVariables` and prioritized over workspace variables using `job.variables_as_arguments = true`.
* Testing remote workflows and troubleshooting ingress, TLS, token auth, and Minikube container caching must follow:
  - Guidelines: .agents/skills/terrakube-cli-driven-workflow/SKILL.md

