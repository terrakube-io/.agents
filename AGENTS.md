# Terrakube Architecture & Coding Rules

## 1. Elide API Entity Conventions (`io.terrakube.api.rs`)
* All JSON:API resources must be defined under `io.terrakube.api.rs` (or domain sub-packages).
* Every entity MUST have `@Include`, `@Entity(name = "...")`, Lombok `@Getter`/`@Setter`, and explicit Elide security permission annotations (`@ReadPermission`, `@CreatePermission`, `@UpdatePermission`, `@DeletePermission`).
* Primary key fields (`id`) MUST be typed `UUID` using `@JdbcTypeCode(Types.VARCHAR)` and `@Convert(converter = IdConverter.class)`.

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
