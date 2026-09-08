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
* Elide security check expressions for policies MUST use space-separated words without underscores (`team view policy set`, `team manage policy set`, `team override policy`).
* Resolution services running within background jobs (Quartz `ScheduleJob`) MUST NOT access lazy collections on detached entities; all tag, variable, and attachment queries must use dedicated repository methods (`workspaceTagRepository.findByWorkspace(...)`).
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

