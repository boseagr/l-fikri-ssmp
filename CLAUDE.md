# Kubernetes — Developer Reference

## Build Commands

```bash
# Build all binaries
make all

# Build a specific component (e.g., kube-apiserver)
make all WHAT=cmd/kube-apiserver

# Cross-compile for all platforms
make cross

# Clean build artifacts
make clean
```

All binaries land in `_output/bin/`.

## Test Commands

```bash
# Run the full unit test suite
make test

# Run unit tests for a specific package
make test WHAT=./pkg/scheduler/...

# Run a single named test (use TEST env var for -run pattern)
make test WHAT=./pkg/scheduler/... KUBE_TEST_ARGS="-run TestFitPredicate"

# Run integration tests
make test-integration

# Run integration tests for a specific package
make test-integration WHAT=./test/integration/scheduler/...

# Run node end-to-end tests (requires a running node)
make test-e2e-node

# Run command-line tests
make test-cmd
```

The test rules delegate to `hack/make-rules/test.sh` and friends.

## Development Commands

```bash
# Run all pre-submission verifications (run before opening a PR)
make verify
# or equivalently:
hack/verify-all.sh

# Run only the fast verifications
make quick-verify

# Regenerate auto-generated code (run after API or code-gen changes)
make update
# or equivalently:
hack/update-all.sh

# Run the linter
make lint
# or directly:
hack/verify-golangci-lint.sh
```

All `hack/` scripts must be executed from the repository root.

## Project Structure

```
k8s.io/kubernetes
├── api/            API type definitions (versioned)
├── build/          Build system; root Makefile lives in build/root/
├── cluster/        Cluster provisioning scripts
├── cmd/            Main packages for every Kubernetes binary
│   ├── kube-apiserver/
│   ├── kube-controller-manager/
│   ├── kube-proxy/
│   ├── kube-scheduler/
│   └── kubelet/    (plus many CLI tools under cmd/)
├── hack/           Developer scripts: build, verify, update, codegen
│   └── make-rules/ Shell helpers invoked by the top-level Makefile
├── pkg/            Core library packages
│   ├── apis/       Internal (unversioned) API types
│   ├── controller/ Controller implementations
│   ├── kubelet/    Kubelet logic
│   ├── scheduler/  Scheduler framework and plugins
│   └── …
├── plugin/         Plugin interfaces (e.g. auth plugins)
├── staging/        Staging area for separately-published k8s.io/* modules
│   └── src/k8s.io/ (client-go, apimachinery, apiserver, etc.)
├── test/
│   ├── e2e/        End-to-end tests (cluster-level)
│   ├── e2e_node/   Node-level e2e tests
│   ├── integration/Integration tests
│   └── conformance/Conformance test suite
├── third_party/    Vendored third-party code not managed by Go modules
└── vendor/         Go module vendor directory
```

## Architecture Notes

- **Module:** `k8s.io/kubernetes`, Go 1.26.
- **Staging repos:** The `staging/src/k8s.io/` tree contains the source for separately-published modules (e.g., `k8s.io/client-go`, `k8s.io/apimachinery`). The root `go.mod` uses `replace` directives so all local changes resolve immediately. Do not edit vendor copies of these — edit under `staging/`.
- **API versioning:** External (versioned) types live in `api/`; internal types live in `pkg/apis/`. Conversion functions bridge the two. All API changes require running `make update` to regenerate deepcopy, conversion, and OpenAPI artifacts.
- **Ginkgo/Gomega:** The e2e and node-e2e suites use the Ginkgo v2 BDD framework (`github.com/onsi/ginkgo/v2`) with Gomega assertions. Unit tests use the standard `testing` package.
- **Feature gates:** All new optional behaviors are gated behind a `featuregate.Feature` constant defined in `pkg/features/`. Gate definitions must be added there before referencing them in code.
- **Controllers follow the reconcile pattern:** A controller registers an informer, enqueues keys on change events, and reconciles desired vs. actual state in a `syncHandler`. The work queue is rate-limited.
- **Scheduler framework:** The scheduler is plugin-based; new scheduling behaviors are implemented as plugins satisfying one or more `framework.Plugin` interfaces (Filter, Score, Reserve, etc.) and registered in `pkg/scheduler/framework/plugins/`.

## Code Style

- **Linter:** `golangci-lint` — config at `.golangci.yml` in the repo root. Run `make lint` or `hack/verify-golangci-lint.sh`.
- **Formatting:** Standard `gofmt`/`goimports`. The verify scripts enforce this; run `goimports -w .` on changed files.
- **Naming:** Follow standard Go conventions. Exported types use PascalCase; unexported use camelCase. Acronyms are all-caps (`APIServer`, not `ApiServer`).
- **Error handling:** Always propagate errors; avoid `_` discards except in tests. Use `fmt.Errorf("context: %w", err)` for wrapping.
- **Imports:** Group in three blocks separated by blank lines: stdlib, external, internal (`k8s.io/…`). `goimports` enforces this automatically.
- **Logging:** Use `klog` (not `log` or `fmt.Print`). Structured logging uses `klog.InfoS`/`klog.ErrorS` with key-value pairs.
- **Tests:** Table-driven tests are strongly preferred. Each test case is a struct literal in a `tests` slice; iterate with `t.Run(tc.name, ...)`.
