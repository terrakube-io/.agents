---
name: terrakube-cli
description: Guidelines, patterns, and conventions for developing, extending, or maintaining the Terrakube CLI application (`terrakube-cli`), including adding new resource commands, testing, and using the generic resource framework.
---

# Terrakube CLI Implementation Guide

`terrakube-cli` is the official CLI for managing Terrakube Server resources. It is built with Go, Cobra/Viper, and consumes the `terrakube-go` client library.

---

## 1. Architecture Overview

```
main.go              # Entry point — calls cmd.Execute()
cmd/
  root.go            # Root command, config init, newClient(), output rendering
  <resource>.go      # One file per resource, calls resource.Register[T]()
  flagutil.go        # Shared flag registration helpers (registerOrgFlag, etc.)
  resolvers.go       # Shared name-to-ID resolver functions
internal/
  output/
    renderer.go      # Render(w, data, format) — json, yaml, table, tsv, none
  resource/
    resource.go      # Config[T], Register[T](), field population via reflection
    resolve.go       # Parent scope name-to-ID resolution
testutil/
  server.go          # TestServer: HTTP mock with route registration and request recording
  assertions.go      # Assert helpers: AssertMethod, AssertPath, AssertBodyContains, etc.
  fixtures.go        # Fixture<Resource>() and Fixture<Resource>List() for all resources
```

---

## 2. Adding a New Resource (Framework Pattern — REQUIRED)

All new resources MUST use the generic resource framework via `resource.Register[T]()`. Do NOT create hand-written Cobra subcommands.

### 2.1 Create `cmd/<resource>.go`

A single file with an `init()` function that calls `resource.Register[T]()`:

```go
package cmd

import (
    "context"

    terrakube "github.com/terrakube-io/terrakube-go"

    "terrakube/internal/resource"
)

func init() {
    resource.Register(rootCmd, resource.Config[terrakube.MyResource]{
        Runtime: resource.Runtime{
            NewClient:  newClient,
            GetContext: getContext,
            GetOutput:  func() string { return output },
        },
        Name:    "my-resource",
        Aliases: []string{"mr", "my-resources"},
        Parents: []resource.ParentScope{{
            Name:      "organization",
            Flag:      "organization",
            ShortFlag: "o",
            Aliases:   []string{"org"},
            IDFlag:    "organization-id",
            Resolver:  orgResolver,
        }},
        Fields: []resource.FieldDef{
            {StructField: "Name", Flag: "name", Short: "n", Type: resource.String, Required: true, Description: "Resource name"},
            {StructField: "Description", Flag: "description", Short: "d", Type: resource.String, Description: "Resource description"},
            {StructField: "Enabled", Flag: "enabled", Type: resource.Bool, Description: "Whether resource is enabled"},
        },
        List: func(ctx context.Context, c *terrakube.Client, pIDs []string, opts *terrakube.ListOptions) ([]*terrakube.MyResource, error) {
            return c.MyResources.List(ctx, pIDs[0], opts)
        },
        Get: func(ctx context.Context, c *terrakube.Client, pIDs []string, id string) (*terrakube.MyResource, error) {
            return c.MyResources.Get(ctx, pIDs[0], id)
        },
        Create: func(ctx context.Context, c *terrakube.Client, pIDs []string, r *terrakube.MyResource) (*terrakube.MyResource, error) {
            return c.MyResources.Create(ctx, pIDs[0], r)
        },
        Update: func(ctx context.Context, c *terrakube.Client, pIDs []string, r *terrakube.MyResource) (*terrakube.MyResource, error) {
            return c.MyResources.Update(ctx, pIDs[0], r)
        },
        Delete: func(ctx context.Context, c *terrakube.Client, pIDs []string, id string) error {
            return c.MyResources.Delete(ctx, pIDs[0], id)
        },
    })
}
```

### 2.2 Key Rules

- **Runtime block**: Always set `NewClient: newClient`, `GetContext: getContext`, `GetOutput: func() string { return output }`.
- **Name**: Kebab-case, matching the API resource name. Use `Aliases` for short forms and plurals.
- **Parents**: Order matters — parent IDs are resolved sequentially. Use `pIDs[0]` for org, `pIDs[1]` for next parent, etc.
- **IDFlag**: Set to `"<parent>-id"` for backward compatibility with old flag names.
- **RawID**: Set `true` for parents whose IDs are NOT UUIDs (e.g., numeric job IDs). This skips name resolution.
- **Fields**: Map CLI flags to `terrakube-go` struct fields. Use pointer types (`*string`, `*bool`) in the model for optional fields.
- **FieldDef.Required**: Enforced only on `create`. On `update`, only flags explicitly set by the user are applied.
- **Omit API closures**: Set `List`/`Get`/`Create`/`Update`/`Delete` to `nil` to omit that subcommand entirely.

### 2.3 Multi-Parent Resources

For resources nested under multiple parents (e.g., `collection-item` under org → collection):

```go
Parents: []resource.ParentScope{
    {
        Name: "organization", Flag: "organization", ShortFlag: "o",
        Aliases: []string{"org"}, IDFlag: "organization-id",
        Resolver: orgResolver,
    },
    {
        Name: "collection", Flag: "collection",
        IDFlag: "collection-id",
        Resolver: collectionResolver,
    },
},
```

API closures then use `pIDs[0]` (org) and `pIDs[1]` (collection).

### 2.4 Adding a New Name Resolver

If the new resource can itself be a parent of other resources, add a resolver to `cmd/resolvers.go`:

```go
func myResourceResolver(ctx context.Context, c *terrakube.Client, resolvedParentIDs []string, name string) (string, error) {
    items, err := c.MyResources.List(ctx, resolvedParentIDs[0], &terrakube.ListOptions{Filter: "name==" + name})
    if err != nil {
        return "", err
    }
    if len(items) == 0 {
        return "", fmt.Errorf("no my-resource found with name %q", name)
    }
    if len(items) > 1 {
        return "", fmt.Errorf("multiple my-resources match name %q, use --my-resource with the ID", name)
    }
    return items[0].ID, nil
}
```

---

## 3. Testing Conventions

### 3.1 Test File Location

- **Framework resources**: Place tests in `cmd/<resource>_test.go`.
- **Shared test infrastructure**: `cmd/cmd_test.go` (helpers: `resetGlobalFlags()`, `executeCommand()`, `setupTestServer()`).
- **Test utilities**: `testutil/` package (`server.go`, `assertions.go`, `fixtures.go`).

### 3.2 Test Structure (MANDATORY for every resource)

Every resource MUST have at minimum these tests:

1. **List E2E** (`TestCmd<Resource>ListE2E`) — verify GET to correct path, assert output contains expected data.
2. **Get E2E** (`TestCmd<Resource>GetE2E`) — verify GET with ID in path.
3. **Create E2E** (`TestCmd<Resource>CreateE2E`) — verify POST, capture and assert JSON:API request body attributes.
4. **Delete E2E** (`TestCmd<Resource>DeleteE2E`) — verify DELETE, assert `"<resource> deleted"` output.
5. **Missing parent** (`TestCmd<Resource>ListMissing<Parent>`) — verify error when parent flag is omitted.
6. **Name resolution** (`TestCmd<Resource>NameResolution`) — if resource has name resolution, verify resolver is called.

### 3.3 Test Pattern

```go
func TestCmd<Resource>ListE2E(t *testing.T) {
    resetGlobalFlags()  // CRITICAL: always call first to reset Cobra/Viper state

    handler := http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        if r.Method != http.MethodGet {
            t.Errorf("expected GET, got %s", r.Method)
        }
        if !strings.Contains(r.URL.Path, "organization/<org-id>/<resource>") {
            t.Errorf("unexpected path: %s", r.URL.Path)
        }
        w.Header().Set("Content-Type", "application/vnd.api+json")
        _ = jsonapi.MarshalPayload(w, testutil.Fixture<Resource>List())
    })

    ts := setupTestServer(handler)
    defer ts.Close()

    out, err := executeCommand("<resource>", "list", "--organization-id", "<org-id>")
    if err != nil {
        t.Fatalf("unexpected error: %v", err)
    }

    if !strings.Contains(out, "<expected-value>") {
        t.Errorf("expected output to contain '<expected-value>', got: %s", out)
    }
}
```

### 3.4 Fixtures (MANDATORY)

Add fixtures to `testutil/fixtures.go`:

```go
// --- MyResource ---

func FixtureMyResource() *terrakube.MyResource {
    return &terrakube.MyResource{
        ID:   "<2-char-prefix><uuid-suffix>",
        Name: "<realistic-name>",
        // ... populate all fields with realistic test data
    }
}

func FixtureMyResourceList() []*terrakube.MyResource {
    return []*terrakube.MyResource{
        FixtureMyResource(),
        {
            ID:   "<different-id>",
            Name: "<different-name>",
        },
    }
}
```

**Fixture conventions**:
- IDs use a 2-character prefix identifying the resource type (e.g., `"ga1a2b3c-..."` for GithubAppToken, `"ss1a2b3c-..."` for SSH).
- Always provide two items in list fixtures.
- Use `strPtr()` helper for optional `*string` fields.
- Use `boolPtr()` helper for optional `*bool` fields.

### 3.5 Critical Testing Rules

- **Always call `resetGlobalFlags()`** as the first line in every test. Cobra flag state bleeds between tests.
- **Use `setupTestServer(handler)`** to create mock servers — it auto-configures `viper` with the server URL and test token.
- **Use `executeCommand(args...)`** to run CLI commands — it captures stdout and returns (output, error).
- Tests run in `package cmd` (same package) to access internal helpers.

---

## 4. Output Rendering

- All framework resources automatically render output via `internal/output.Render(w, data, format)`.
- Supported formats: `json` (default), `yaml`, `table`, `tsv`, `none`.
- JSON output uses JSON:API serialization, then unwraps the `data` envelope for cleaner output.
- Table/TSV rendering uses reflection; fields tagged `jsonapi:"relation,..."` are automatically skipped.
- `--hide-nulls` (default `true`) strips null values from JSON output.

---

## 5. Configuration & Authentication

- Config file: `~/.terrakube-cli.yaml` or `--config` flag.
- Auth: Bearer token via config key `token` or `TERRAKUBE_TOKEN` env var.
- API URL: Config key `api_url` or `TERRAKUBE_API_URL` env var.
- All flags support env vars with `TERRAKUBE_` prefix (e.g., `--organization` ↔ `TERRAKUBE_ORGANIZATION`).
- Priority: CLI flag > env var > config file.

---

## 6. Build & Test Commands

```bash
go test -race ./...      # Run all tests with race detector
go build ./...           # Build
golangci-lint run ./...  # Lint
```

Or via mise:
```bash
mise run test            # go test -race ./...
mise run build           # go build ./...
mise run lint            # golangci-lint run ./...
mise run check           # all of the above
```
