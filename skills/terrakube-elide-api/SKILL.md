---
name: terrakube-elide-api
description: Guidelines and patterns for implementing or extending JSON:API resources, security checks, and lifecycle hooks using Yahoo Elide in Terrakube API (`io.terrakube.api.rs`), including Liquibase database migrations.
---

# Terrakube Yahoo Elide JSON:API Implementation Guide

Terrakube exposes its core REST API using **Yahoo Elide** (`com.yahoo.elide:elide-spring-boot-starter`), which binds JPA entities directly to standard JSON:API (v1.0) endpoints.

## Package Layout (`io.terrakube.api.rs`)
* `io.terrakube.api.rs` - Contains top-level entity classes and sub-packages per domain model (e.g., `workspace`, `organization`, `job`, `project`, `module`, `vcs`, `ssh`, `template`, `provider`).
* `io.terrakube.api.rs.checks` - Contains custom `@SecurityCheck` implementations extending `OperationCheck<T>`.
* `io.terrakube.api.rs.hooks` - Contains `LifeCycleHook<T>` classes triggered on entity CREATE, UPDATE, or DELETE events.

---

## 1. Creating or Extending a JSON:API Resource Entity

When adding a new resource or modifying an existing entity:

1. **Annotations Required**:
   - `@Include`: Registers class as an Elide JSON:API resource.
   - `@Entity(name = "...")`: Hibernate entity mapping.
   - `@Getter` / `@Setter`: Lombok accessors.
   - `@ReadPermission`, `@CreatePermission`, `@UpdatePermission`, `@DeletePermission`: Defines access rules using registered security check expressions.
   - `@SQLRestriction`: (If applicable) Soft-delete filter (e.g. `@SQLRestriction("deleted = false")`).

2. **Primary Key Mapping**:
   - Always use `UUID` with `@Convert(converter = IdConverter.class)` and `@JdbcTypeCode(Types.VARCHAR)`:
   ```java
   @Id
   @JdbcTypeCode(Types.VARCHAR)
   @Convert(converter = IdConverter.class)
   @GeneratedValue(strategy = GenerationType.UUID)
   private UUID id;
   ```

3. **Audit Fields**:
   - Extend `GenericAuditFields` for standard auditing fields (`createdDate`, `updatedDate`, `createdBy`, `updatedBy`).

4. **Relationships**:
   - Map relationships using standard JPA annotations (`@OneToMany`, `@ManyToOne`). Elide automatically generates nested routes (`/api/v1/organization/{id}/workspace`) and relationship links.

---

## 2. Implementing Security Checks (`@SecurityCheck`)

Elide security expressions (e.g. `@ReadPermission(expression = "user belongs organization")`) map to custom Spring-managed check components:

1. Place checks in `io.terrakube.api.rs.checks.<domain>`.
2. Extend `com.yahoo.elide.core.security.checks.OperationCheck<T>`.
3. Annotate class with `@SecurityCheck(RULE)` and define `public static final String RULE = "...";`.
4. Inject Spring beans (e.g. `AuthenticatedUser`, `GroupService`) using `@Autowired`.
5. Implement `ok(T entity, RequestScope requestScope, Optional<ChangeSpec> changeSpec)` returning `boolean`.

Example:
```java
@Slf4j
@SecurityCheck(UserBelongsOrganization.RULE)
public class UserBelongsOrganization extends OperationCheck<Organization> {
    public static final String RULE = "user belongs organization";

    @Autowired
    AuthenticatedUser authenticatedUser;

    @Override
    public boolean ok(Organization organization, RequestScope requestScope, Optional<ChangeSpec> optional) {
        if (authenticatedUser.isSuperUser(requestScope.getUser())) return true;
        return isMemberOrganization(requestScope.getUser(), organization);
    }
}
```

---

## 3. Implementing Lifecycle Hooks (`LifeCycleHook`)

Hooks execute custom side-effects during Elide operations (CREATE, UPDATE, DELETE):

1. Place hook in `io.terrakube.api.rs.hooks.<domain>`.
2. Implement `com.yahoo.elide.core.lifecycle.LifeCycleHook<T>`.
3. Annotate target entity with `@LifeCycleHookBinding`:
   ```java
   @LifeCycleHookBinding(
       operation = LifeCycleHookBinding.Operation.CREATE,
       phase = LifeCycleHookBinding.TransactionPhase.PRECOMMIT,
       hook = WorkspaceManageHook.class
   )
   ```
4. Implement `execute(Operation operation, TransactionPhase phase, T entity, RequestScope scope, Optional<ChangeSpec> changes)` method to handle pre-commit/post-commit logic.

---

## 4. Liquibase Database Migrations Management

Terrakube uses **Liquibase** to manage database migrations across supported RDBMS engines (PostgreSQL, H2, Oracle, MySQL, SQL Server).

### Migration Directory Structure
* Main Master Changelog: `terrakube/api/src/main/resources/db/changelog/changelog.xml`
* Version & Feature Changelogs: `terrakube/api/src/main/resources/db/changelog/local/`

### Workflow for Database Schema Changes:
1. **Create New Migration File**:
   - Location: `terrakube/api/src/main/resources/db/changelog/local/changelog-<version>-<feature>.xml`
   - Example: `changelog-2.32.0-new-entity.xml`

2. **Write ChangeSet**:
   ```xml
   <?xml version="1.0" encoding="UTF-8"?>
   <databaseChangeLog
           xmlns="http://www.liquibase.org/xml/ns/dbchangelog"
           xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
           xsi:schemaLocation="http://www.liquibase.org/xml/ns/dbchangelog
               http://www.liquibase.org/xml/ns/dbchangelog/dbchangelog-4.3.xsd">
       <changeSet id="<version>-<feature-description>" author="developer-email@example.com">
           <createTable tableName="new_feature">
               <column name="id" type="varchar(36)">
                   <constraints primaryKey="true"/>
               </column>
               <column name="name" type="varchar(256)">
                   <constraints nullable="false"/>
               </column>
               <column name="created_date" type="datetime"/>
               <column name="updated_date" type="datetime"/>
               <column name="created_by" type="varchar2(128)"/>
               <column name="updated_by" type="varchar2(128)"/>
           </createTable>
       </changeSet>
   </databaseChangeLog>
   ```

3. **Register in `changelog.xml`**:
   - Add the `<include>` entry at the end of `terrakube/api/src/main/resources/db/changelog/changelog.xml`:
   ```xml
   <include file="/db/changelog/local/changelog-2.32.0-new-entity.xml" />
   ```

---

## 5. End-to-End Checklist for Adding a New Elide Feature

When introducing a new domain model or REST API capability to Terrakube:
1. [ ] **Liquibase**: Create changelog XML in `db/changelog/local/` and include it in `changelog.xml`.
2. [ ] **JPA/Elide Entity**: Create `@Entity` in `io.terrakube.api.rs.<domain>` with `@Include`, Lombok getters/setters, UUID converter, and standard audit fields (`GenericAuditFields`).
3. [ ] **Security Checks**: Define custom `OperationCheck<T>` classes in `io.terrakube.api.rs.checks.<domain>` annotated with `@SecurityCheck(RULE)`.
4. [ ] **Entity Permission Binding**: Annotate the entity class and/or sensitive fields with `@ReadPermission`, `@CreatePermission`, `@UpdatePermission`, `@DeletePermission`.
5. [ ] **Lifecycle Hooks**: If business side-effects (creating resources, notifying webhooks, validating data) are needed, create a `LifeCycleHook<T>` implementation in `io.terrakube.api.rs.hooks.<domain>` and bind it via `@LifeCycleHookBinding`.
