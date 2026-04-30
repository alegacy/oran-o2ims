# AGENTS.md

This file provides codebase guidance for AI coding agents working with this repository.

## Project Overview

O-Cloud Manager operator built on OpenShift and ACM (Red Hat Advanced Cluster Management). Implements the O-RAN O2 IMS specification for 5G infrastructure management: bare-metal inventory, cluster provisioning, firmware management, and alarm monitoring via REST APIs.

## Development Commands

### Build and Test

- `make build` - Build manager binary (runs generate, fmt, vet first)
- `make binary` - Build binary only (no code generation)
- `make test` - Run unit tests (excludes envtest)
- `make test-envtest` - Run Kubernetes integration tests (requires envtest assets)
- `make test-e2e` - Run end-to-end tests
- `make test-crd-watcher` - Run CRD watcher tests
- `make test-coverage-check` - Check coverage against per-package thresholds

### Running a Single Test

Pass Ginkgo flags via `ginkgo_flags`:

```bash
make test ginkgo_flags="--focus='should handle alarm'"
make test-envtest ginkgo_flags="--focus='InventoryController'"
```

### Code Quality

- `make lint` - Run all linters (golangci-lint, yamllint, bashate, shellcheck)
- `make golangci-lint` - Go linting only
- `make yamllint` - YAML linting only
- `make shellcheck` - Shell script linting
- `make bashate` - Bash style checking
- `make markdownlint` - Markdown linting (requires container engine)
- `make fmt` - Format Go code
- `make vet` - Run go vet
- `make ci-job` - Full CI pipeline locally (format, vet, lint, test, e2e, envtest, coverage, bundle-check)

### Code Generation (run after API changes)

- `make generate` - Generate DeepCopy methods for CRDs
- `make manifests` - Generate RBAC, webhooks, and CRD manifests
- `make go-generate` - Run go generate (OpenAPI codegen, mocks)
- `make bundle` - Regenerate operator bundle

### Container and Deployment

- `make docker-build` - Build container image
- `make docker-push` - Build and push container image
- `make deploy` - Deploy operator to cluster
- `make undeploy` - Remove operator from cluster

To build and push to a personal registry, set `IMAGE_TAG_BASE` and `VERSION`:

```bash
make IMAGE_TAG_BASE=quay.io/<your-username>/oran-o2ims VERSION=latest docker-build docker-push
```

### Submodules

The project uses a `telco5g-konflux` git submodule for build scripts. Most make targets auto-sync it. Skip with `SKIP_SUBMODULE_SYNC=yes`.

## Architecture

### Multi-Service Binary

The project builds a single binary that runs as multiple services in separate containers. Each service has its own cmd package at `internal/service/{service}/cmd/` with root.go, serve.go, and optionally migrate.go. The Deployment determines which subcommand each container runs.

### Service Architecture Pattern

Services follow a consistent initialization pattern in `internal/service/{service}/`:

1. Embedded OpenAPI spec for request validation
2. PostgreSQL connection pool (services with persistence)
3. Repository layer implementing a defined interface (`db/repo/`)
4. Infrastructure clients for cross-service communication
5. Authentication/authorization middleware
6. HTTP server with generated OpenAPI handlers

Shared infrastructure lives in `internal/service/common/`:

- `api/middleware/` - Auth, filtering, schema validation
- `db/` - Connection pooling, migration helpers
- `utils/` - Shared server configuration

### OpenAPI Code Generation Flow

REST APIs are generated from OpenAPI specs using `oapi-codegen`:

```text
internal/service/{service}/api/openapi.yaml
  -> tools/oapi-codegen.yaml (config)
  -> generated/{service}.generated.go (StrictServerInterface)
  -> server.go (implements the interface)
```

Triggered via `//go:generate` directives in `internal/service/{service}/api/tools/generate.go`.

### CRD and Controller Architecture

**CRDs** (defined in `api/`):

- `Inventory` (`api/inventory/v1alpha1/`) - O-Cloud service configuration, controls which services are deployed
- `ProvisioningRequest` (`api/provisioning/v1alpha1/`) - Cluster provisioning lifecycle
- `ClusterTemplate` (`api/provisioning/v1alpha1/`) - Template definitions for provisioning
- `HardwarePlugin`, `HardwareProfile` (`api/hardwaremanagement/v1alpha1/`) - Hardware plugin providers
- `NodeAllocationRequest` (`api/hardwaremanagement/plugins/v1alpha1/`) - Hardware allocation

**Controllers** (`internal/controllers/`):

- `InventoryController` - Deploys/configures O-Cloud services based on Inventory CR
- `ProvisioningRequestController` - Multi-phase provisioning workflow
- `ClusterTemplateController` - Validates and manages cluster templates

### Provisioning Workflow Phases

The ProvisioningRequestController runs a multi-phase state machine:

1. **Validation** - Validate request against ClusterTemplate schema
2. **Hardware Provisioning** - Create NodeAllocationRequest, poll hardware plugin, create BMC secrets
3. **Cluster Installation** - Render and apply ClusterInstance, monitor ZTP progress
4. **Post-Provisioning** - Apply policy templates, monitor compliance
5. **Upgrades** - Handle cluster upgrade requests

Each phase sets typed conditions on the CR status. Condition helpers are in `internal/controllers/utils/conditions.go`. Condition types and reasons are defined in `api/provisioning/v1alpha1/conditions.go`.

### Webhooks

`ProvisioningRequest` has a validating webhook (`api/provisioning/v1alpha1/provisioningrequest_webhook.go`):

- Create: requires metadata.name to be a valid UUID
- Update: blocks spec changes during cluster installation; limits changes after completion to annotations/labels and node scaling
- Delete: warns about cleanup time; prevents deletion during active hardware configuration

### Database Layer

PostgreSQL with migration-based schema management:

- Migrations: `internal/service/{service}/db/migrations/` (numbered .up.sql/.down.sql)
- Repository interfaces: `internal/service/{service}/db/repo/*_interface.go`
- Implementations: `internal/service/{service}/db/repo/*_repository.go`
- Tests use `pgxmock` for database mocking
- Mock generation via `//go:generate mockgen` directives on repository interfaces

## Testing Patterns

- **Framework**: Ginkgo v2 + Gomega
- **Labels**: Tests use `Label("envtest")` for K8s integration tests; unit tests are `!envtest`
- **Controller tests**: Use fake client with SSA compatibility wrapper (`SSACompatibleClient`)
- **Mock hardware server**: `internal/controllers/mock_hardware_plugin_server.go` for provisioning tests
- **DB tests**: `pgxmock` for repository layer; `mockgen` for service-layer mocks

## Contributing Requirements

- All commits must be signed off with DCO: `git commit -s`
- Run `make ci-job` before submitting PRs
- After API changes: `make generate && make manifests && make bundle`
- AI-generated code must use `Co-Authored-By` or `Assisted-By` trailer
- When making code changes, ensure test coverage for new code and functional
  changes. If a bug fix or new behavior is added without a corresponding test,
  write one. If an existing scenario is discovered to be untested (e.g., during
  code review), add a test for it in the same commit or PR. Tests should verify
  the specific behavior, not just increase line coverage.
