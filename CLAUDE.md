# Kubernetes — Developer Reference

## Build Commands

```bash
# Build all binaries (output in _output/bin/)
make

# Build a specific component
make all WHAT=cmd/kubelet

# Build with debug symbols (unstripped, useful with delve)
make all DBG=1

# Cross-platform build
make cross

# Build inside the containerized environment (ensures consistency)
build/run.sh make
build/run.sh make cross
```

## Test Commands

```bash
# Run unit tests
make test

# Run unit tests inside container
build/run.sh make test

# Run integration tests
make test-integration

# Run CLI tests
make test-cmd

# Run end-to-end node tests
make test-e2e-node

# Run a single package's tests
go test ./pkg/scheduler/...

# Run a single named test
go test ./pkg/scheduler/... -run TestSchedulerCreation
```

## Verification (run before submitting a PR)

```bash
# Run all verification checks
hack/verify-all.sh

# Auto-fix formatting and generated files
hack/update-all.sh

# Individual checks
hack/verify-gofmt.sh          # formatting
hack/verify-codegen.sh        # generated code
hack/verify-mocks.sh          # mocks
hack/verify-govulncheck.sh    # vulnerability scan
```

## Development Commands

```bash
# Format all Go source files
hack/update-gofmt.sh

# Update vendor directory after changing go.mod
hack/update-vendor.sh

# Pin a specific dependency version
hack/pin-dependency.sh <module> <version>
```

## Project Structure

```
.
├── api/              API type definitions and OpenAPI specs
├── build/            Containerized build scripts and Dockerfiles
├── cluster/          Cluster provisioning and cloud-provider helpers
├── cmd/              Entry points for all Kubernetes binaries
│   ├── kube-apiserver/
│   ├── kube-controller-manager/
│   ├── kube-scheduler/
│   ├── kube-proxy/
│   ├── kubelet/
│   ├── kubectl/
│   └── kubeadm/
├── docs/             User-facing documentation
├── hack/             Developer and CI scripts (verify-*, update-*, lib/)
├── pkg/              Core library packages (admission, auth, controller,
│   │                  kubelet, scheduler, kubeapiserver, etc.)
├── plugin/           Plugin framework
├── staging/          Staging area for packages published to k8s.io/*
│   └── src/k8s.io/   (client-go, apimachinery, api, apiserver, …)
├── test/             Test infrastructure
│   ├── e2e/          End-to-end tests
│   ├── e2e_node/     Node-level e2e tests
│   ├── integration/  Integration tests
│   └── fuzz/         Fuzz targets
├── third_party/      Third-party source (not vendored Go modules)
└── vendor/           Vendored Go dependencies (do not edit manually)
```

## Architecture Notes

- **Monorepo + Go workspace**: The repo uses `go.work` to manage multiple modules. Packages under `staging/src/k8s.io/` are the authoritative source; they are published separately (e.g., `k8s.io/client-go`) via `replace` directives in `go.mod`.
- **Code generation**: API types, DeepCopy functions, mocks, and docs are generated. Always run `hack/update-codegen.sh` after changing API types and commit the generated output.
- **Controller pattern**: Business logic lives in controllers under `pkg/controller/`. Controllers reconcile desired state (stored in etcd via the API server) with actual state.
- **API versioning**: APIs live in `staging/src/k8s.io/api/` (serialized types) and `pkg/apis/` (internal types + validation + conversion). Each group/version has its own directory.
- **Feature gates**: All new features must be gated. Use `hack/verify-featuregates.sh` to check consistency.
- **Key dependencies**: Go 1.26, etcd v3.7, ginkgo v2 (tests), CEL (admission policies), OpenTelemetry (tracing), Prometheus (metrics), cobra/pflag (CLIs).

## Code Style

- **Formatter**: `gofmt` — enforced by CI. Run `hack/update-gofmt.sh` before committing.
- **Linter**: `golangci-lint` configured via `.golangci.yml` at the repo root.
- **Naming**: Follow standard Go conventions. Package names are lowercase, single words. Exported types use PascalCase.
- **Imports**: Grouped in three blocks — stdlib, external, internal (`k8s.io/*`) — separated by blank lines. `goimports` maintains this automatically.
- **Error handling**: Return errors; avoid panics outside of init paths. Use `fmt.Errorf("... %w", err)` for wrapping.
- **Logging**: Use structured logging via `k8s.io/klog/v2`. Pass a `logger` obtained from `klog.FromContext(ctx)` rather than the global logger in new code.
- **Tests**: Use the Ginkgo/Gomega BDD framework for e2e and integration tests; plain `testing` package for unit tests.
