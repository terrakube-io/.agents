---
name: terraform-provider-terrakube
description: Guidelines, build commands, test patterns, linting rules, and documentation generation conventions for `terraform-provider-terrakube`.
---

# Terrakube Terraform Provider (`terraform-provider-terrakube`) Guide

`terraform-provider-terrakube` is the official Terraform Provider for Terrakube, built on the HashiCorp Terraform Plugin Framework (`github.com/hashicorp/terraform-plugin-framework`).

---

## 1. Directory & Execution Context

* **Repository Location**: `terraform-provider-terrakube/`
* **Subfolder Rule**: Always execute `go`, `git`, and build commands inside the `terraform-provider-terrakube/` subfolder, never from the root workspace directory:
  ```bash
  cd /home/user/git/terrakube-io/terraform-provider-terrakube
  ```

---

## 2. Compiling & Building

* **Install binary to local Go bin** (e.g. for manual testing or Terraform provider resolution):
  ```bash
  go install .
  ```

* **Compile package without installing**:
  ```bash
  go build -v ./...
  ```

* **Download/tidy module dependencies**:
  ```bash
  go mod download
  go mod tidy
  ```

---

## 3. Testing Conventions

* **Unit Tests (Default / Fast)**:
  Runs all unit tests (client and provider framework mocks) without invoking acceptance infrastructure:
  ```bash
  # Run all unit tests
  go test -v ./...

  # Run all unit tests without cache (clean run)
  go test -count=1 ./...

  # Run specific test suite or test function
  go test -v -run TestPolicySetParameterResource ./internal/provider/...
  go test -v -run TestTeamResource_Create_BackwardCompatibility ./internal/provider/...
  ```

* **Acceptance Tests (`TF_ACC=1`)**:
  Runs provider acceptance tests against live or containerized Terrakube infrastructure:
  ```bash
  make testacc
  # Or directly:
  TF_ACC=1 go test -v -cover ./internal/provider/ -timeout 120m
  ```

* **Unit Testing Best Practices**:
  - Test framework schema declarations (`req.Schema.ValidateImplementation(...)`).
  - Use `httptest.NewServer` or custom `http.RoundTripper` in client and provider tests to mock Terrakube API JSON:API endpoints.
  - When writing unit tests that inspect `ImportStateResponse`, always initialize `resp.State` with both `Schema` and `Raw` (`tftypes.NewValue(objType, nil)`) to avoid plugin framework `invalid transform: value missing type State Write Error`.
  - Validate backward compatibility by asserting that newly added optional attributes are omitted (`omitempty`) when unset, rather than serialized as null or default booleans.

---

## 4. Linting & Formatting Mandate

The CI pipeline runs `golangci-lint` (enforcing `gofmt` and rules defined in `.golangci.yml`).

* **Format Code with `gofmt` (MANDATORY)**:
  Before committing any changes, format all Go files:
  ```bash
  gofmt -s -w .
  ```

* **Verify Formatting**:
  Verify there are no formatting diffs (output must be empty):
  ```bash
  gofmt -d .
  ```

* **Run `go vet`**:
  ```bash
  go vet ./...
  ```

* **Run `golangci-lint` (if installed locally or in CI)**:
  ```bash
  golangci-lint run
  ```

---

## 5. Documentation Generation (`tfplugindocs`)

Provider documentation in `docs/` is generated using HashiCorp `tfplugindocs` triggered by `go generate`:

* **Regenerate Documentation**:
  ```bash
  go generate ./...
  ```

* **Documentation Verification**:
  CI checks if documentation is clean and in sync with schema:
  ```bash
  git diff --compact-summary --exit-code
  ```

* **Templates vs Docs**:
  - `go generate ./...` re-renders `docs/` from schema definitions and templates in `templates/`.
  - If a resource requires customized examples or narrative guides (like `terrakube_team`), edit or create its template under `templates/resources/<resource>.md.tmpl`.
  - Avoid creating standalone files inside `docs/guides/` or `docs/resources/` without a corresponding template if they get overwritten during `tfplugindocs` runs.

---

## 6. Architecture & Implementation Patterns

* **Provider Registration** (`internal/provider/provider.go`):
  - Every resource factory function must be registered in `Resources(ctx context.Context)`.
  - Every data source factory function must be registered in `DataSources(ctx context.Context)`.

* **Client Model Conventions** (`internal/client/client.go`):
  - Terrakube API uses Yahoo Elide JSON:API (`/api/v1/`).
  - Struct tags must use `jsonapi:"primary,<type>"` for ID, `jsonapi:"attr,<attrName>"` for attributes, and `jsonapi:"relation,<relationName>,omitempty"` for relationships.
  - Optional or forward-compatible fields (such as permissions only available in newer API versions) must use pointer types (`*bool`, `*string`) and `json:",omitempty"` so they are omitted when communicating with older API versions.

* **Resource & Data Source Structure**:
  - `internal/provider/<name>_resource.go`: Resource schema, metadata, Create, Read, Update, Delete, ImportState.
  - `internal/provider/<name>_data_source.go`: Data source schema, metadata, Read.
  - `internal/provider/<name>_resource_test.go`: Resource unit tests.
  - `internal/provider/<name>_data_source_test.go`: Data source unit tests.
