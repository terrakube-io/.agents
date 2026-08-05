---
name: terrakube-spring-boot-starter
description: Guidelines, patterns, and conventions for extending the Terrakube Java Feign client library (`terrakube-spring-boot-starter/terrakube-client`), including mapping Yahoo Elide JSON:API models, adding Feign REST endpoints, and writing WireMock unit tests.
---

# Terrakube Spring Boot Starter Client Development Guide

`terrakube-spring-boot-starter` is the Spring Boot client library and autoconfiguration for Terrakube. Its core module `terrakube-client` uses **OpenFeign** to interact with Terrakube's backend API (`terrakube/api`), which exposes JPA entities via Yahoo Elide JSON:API standard endpoints.

---

## 1. Module & Package Layout

```
terrakube-spring-boot-starter/
├── terrakube-client/
│   └── src/main/java/io/terrakube/client/
│       ├── TerrakubeClient.java                     # Feign client interface with @RequestLine endpoints
│       ├── dex/                                     # Authentication interceptors (DexCredentialAuthentication)
│       └── model/
│           ├── generic/
│           │   └── Resource.java                    # Base JSON:API resource class (type, id)
│           ├── response/
│           │   ├── Response.java                    # Envelope for single/list resources: {"data": T1}
│           │   └── ResponseWithInclude.java         # Envelope with side-loaded data: {"data": T1, "included": List<T2>}
│           ├── organization/                        # Domain models matching Elide entities in io.terrakube.api.rs
│           │   └── <domain>/                        # e.g., workspace, job, module, ssh, template, vcs
│           ├── graphql/                             # GraphQL request/response models
│           └── refresh/                             # OAuth token refresh models
└── terrakube-spring-boot-autoconfigure/             # Spring Boot AutoConfiguration (RestClientAutoConfiguration)
```

---

## 2. Model Structure Conventions for Elide JSON:API Resources

Every JSON:API resource in `terrakube-client` mirrors a Yahoo Elide JPA entity from `io.terrakube.api.rs`.

### 2.1 Model Hierarchy

For a given Elide entity (e.g. `Workspace` in `io.terrakube.api.rs.workspace.Workspace`), build the client model in `io.terrakube.client.model.organization.<domain>` with 4 standard components:

1. **Resource Class** (extends `io.terrakube.client.model.generic.Resource`):
   ```java
   package io.terrakube.client.model.organization.workspace;

   import io.terrakube.client.model.generic.Resource;
   import lombok.Getter;
   import lombok.Setter;

   @Getter
   @Setter
   public class Workspace extends Resource {
       private WorkspaceAttributes attributes;
       private Relationships relationships;
   }
   ```
   *Note: `Resource` supplies `type` (String) and `id` (String).*

2. **Attributes Class**:
   ```java
   package io.terrakube.client.model.organization.workspace;

   import lombok.Getter;
   import lombok.Setter;

   @Getter
   @Setter
   public class WorkspaceAttributes {
       private String name;
       private String branch;
       private String source;
       private String terraformVersion;
   }
   ```

3. **Relationships Class & Relationship Data Envelopes**:
   ```java
   package io.terrakube.client.model.organization.workspace;

   import lombok.Getter;
   import lombok.Setter;

   @Getter
   @Setter
   public class Relationships {
       private JobData job;
       private OrganizationData organization;
       private VariableData variable;
       private VcsData vcs;
   }
   ```
   Each `*Data` class wraps target resource reference(s) (either `Resource` or `List<Resource>`):
   ```java
   @Getter
   @Setter
   public class JobData {
       private List<Resource> data;
   }
   ```

4. **Request Payload Wrapper (for POST / PATCH operations)**:
   ```java
   package io.terrakube.client.model.organization.workspace;

   import lombok.Getter;
   import lombok.Setter;

   @Getter
   @Setter
   public class WorkspaceRequest {
       private Workspace data;
   }
   ```

---

## 3. Adding Feign Endpoints in `TerrakubeClient.java`

Declare all client methods in `io.terrakube.client.TerrakubeClient`.

### 3.1 Endpoint Conventions

1. **GET (Collection / Entity)**:
   ```java
   @RequestLine("GET /api/v1/organization/{organizationId}/workspace")
   Response<List<Workspace>> getAllWorkspaces(@Param("organizationId") String organizationId);

   @RequestLine("GET /api/v1/organization/{organizationId}/workspace/{workspaceId}")
   Response<Workspace> getWorkspaceById(@Param("organizationId") String organizationId, @Param("workspaceId") String workspaceId);
   ```

2. **Filtering Query Parameters**:
   Elide uses RSQL syntax in `filter[<resource>]=<field>=={param}`:
   ```java
   @RequestLine("GET /api/v1/organization?filter[organization]=name=={organizationName}")
   Response<List<Organization>> getOrganizationByName(@Param("organizationName") String organizationName);

   @RequestLine("GET /api/v1/organization/{organizationId}/workspace/{workspaceId}/variable?filter[variable]=sensitive==true;category==terraform")
   Response<List<Variable>> getAllSecrets(@Param("organizationId") String organizationId, @Param("workspaceId") String workspaceId);
   ```

3. **Side-Loading Relationships (`include`)**:
   Use `ResponseWithInclude<T1, T2>` when passing `include=<relationship>`:
   ```java
   @RequestLine("GET /api/v1/organization/{organizationId}/workspace/{workspaceId}?include=variable")
   ResponseWithInclude<Workspace, Variable> getWorkspaceByIdWithVariables(
           @Param("organizationId") String organizationId,
           @Param("workspaceId") String workspaceId
   );
   ```

4. **POST / PATCH (JSON:API Headers Required)**:
   Always add `@Headers("Content-Type: application/vnd.api+json")` for standard JSON:API mutation requests:
   ```java
   @RequestLine("PATCH /api/v1/organization/{organizationId}/job/{jobId}")
   @Headers("Content-Type: application/vnd.api+json")
   void updateJob(JobRequest jobRequest, @Param("organizationId") String organizationId, @Param("jobId") String jobId);
   ```

5. **Custom / Non-Elide Endpoints**:
   Use standard JSON content types when querying non-Elide controllers:
   ```java
   @RequestLine("POST /graphql/api/v1")
   @Headers("Content-Type: application/json")
   GraphQLResponse<SearchOrganizationResponse> searchOrganization(GraphQLRequest request);
   ```

---

## 4. Unit Testing Endpoints (`TerrakubeClientTest.java`)

All new client endpoints MUST have corresponding unit tests in `terrakube-client/src/test/java/io/terrakube/client/TerrakubeClientTest.java` using **WireMock JUnit 5**:

```java
@Test
void getWorkspaceById_returnsWorkspace() {
    stubFor(get(urlPathEqualTo("/api/v1/organization/org-1/workspace/ws-1"))
            .willReturn(okJson(
                    "{\"data\":{\"id\":\"ws-1\",\"type\":\"workspace\","
                    + "\"attributes\":{\"name\":\"prod\",\"branch\":\"main\"}}}"
            )));

    Response<Workspace> response = client.getWorkspaceById("org-1", "ws-1");

    assertNotNull(response);
    assertNotNull(response.getData());
    assertEquals("ws-1", response.getData().getId());
    assertEquals("prod", response.getData().getAttributes().getName());
}
```

---

## 5. Summary Checklist for Extending Client

- [ ] Create/Update client model under `io.terrakube.client.model.organization.<domain>`.
- [ ] Ensure model extends `Resource`, implements `Attributes`, `Relationships`, and `Request` envelope if applicable.
- [ ] Add `@RequestLine` method signature in `TerrakubeClient.java`.
- [ ] Add proper `@Headers` (e.g. `Content-Type: application/vnd.api+json` for Elide POST/PATCH).
- [ ] Add WireMock test in `TerrakubeClientTest.java`.
- [ ] Run `./mvnw clean test -f terrakube-spring-boot-starter/pom.xml` to verify build & test suite.
