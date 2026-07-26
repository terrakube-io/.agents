---
name: terrakube-go
description: Guidelines, patterns, and conventions for developing, extending, or maintaining the Terrakube Golang HTTP client library (`terrakube-go`).
---

# Terrakube Golang HTTP Client (`terrakube-go`) Implementation Guide

`terrakube-go` is the official Golang client library for interacting with the Terrakube API. It wraps Yahoo Elide JSON:API (v1.0) endpoints using `github.com/google/jsonapi` as well as custom REST controllers.

---

## 1. Client Architecture & Initialization

* **Client Core** (`client.go`):
  - Base path: `/api/v1/`
  - Media types: `application/vnd.api+json`
  - Initialized using functional options:
    ```go
    client, err := terrakube.NewClient(
        terrakube.WithEndpoint("https://terrakube-api.example.com"),
        terrakube.WithToken("api-bearer-token"),
    )
    ```

* **Service Accessors**:
  - The `Client` struct exposes domain services as public fields (e.g. `client.Organizations`, `client.Workspaces`, `client.Jobs`, `client.Modules`, `client.Providers`, `client.Projects`, `client.ProjectAccess`, `client.Federated`, `client.FederatedClaims`).

---

## 2. JSON:API Struct Tag & Model Design Conventions

All resource models mapped to Yahoo Elide JSON:API backend endpoints must follow these tagging rules:

1. **Primary Key Field (`ID`)**:
   ```go
   ID string `jsonapi:"primary,<elide_entity_type>"`
   ```
   *Example*: `ID string `jsonapi:"primary,workspace"`

2. **Scalar Attributes**:
   - `jsonapi:"attr,<json_camelCase_name>"`
   - Use pointer types (`*string`, `*bool`) for optional/nullable fields to distinguish zero-values from empty data.
   - Mandatory audit fields (if entity tracks auditing):
     ```go
     CreatedBy   *string `jsonapi:"attr,createdBy"`
     CreatedDate *string `jsonapi:"attr,createdDate"`
     UpdatedBy   *string `jsonapi:"attr,updatedBy"`
     UpdatedDate *string `jsonapi:"attr,updatedDate"`
     ```

3. **Relationships**:
   - Always include `,omitempty` in the relation tag.
   - Single (Many-to-One / One-to-One):
     ```go
     Organization *Organization `jsonapi:"relation,organization,omitempty"`
     Workspace    *Workspace    `jsonapi:"relation,workspace,omitempty"`
     ```
   - Slice (One-to-Many):
     ```go
     Workspaces []*Workspace `jsonapi:"relation,workspace,omitempty"`
     Jobs       []*Job       `jsonapi:"relation,job,omitempty"`
     ```

---

## 3. Implementing Domain Services (`crudService[T]`)

Domain services wrap generic CRUD operations using `crudService[T]`:

```go
type WorkspaceService struct {
    crudService[Workspace]
}
```

* **Standard Method Signatures**:
  - `List(ctx context.Context, orgID string, opts *ListOptions) ([]*Workspace, error)`
  - `Get(ctx context.Context, orgID, id string) (*Workspace, error)`
  - `Create(ctx context.Context, orgID string, ws *Workspace) (*Workspace, error)`
  - `Update(ctx context.Context, orgID string, ws *Workspace) (*Workspace, error)`
  - `Delete(ctx context.Context, orgID, id string) error`

* **ID Validation**:
  - Validate parent and resource IDs at the top of service methods using `validateID("paramName", id)` or `validateSubResourceIDs(...)`.
  - Service packages define sentinel errors following `ErrInvalid<ParamName>ID` naming convention (e.g., `ErrInvalidOrganizationID`, `ErrInvalidWorkspaceID`).

---

## 4. Custom Endpoint Handlers (Non-JSON:API)

For endpoints outside Yahoo Elide (e.g. `/access-token/v1/teams`):
- Do not use `jsonapi` struct tags; use standard `json:"..."` struct tags.
- Use `s.client.requestRaw` and `s.client.doRaw` instead of generic `crudService`.

---

## 5. Unit Testing Guidelines

- Place unit tests in `<resource>_test.go`.
- Use `httptest.NewServer` to mock backend responses.
- Assert correct JSON:API request payload formatting and response deserialization.
