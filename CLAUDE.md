# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository

`github.com/dapr/dapr` — the Go implementation of the Dapr distributed application runtime (CNCF graduated). Go version is pinned in `go.mod` (currently 1.26.x). Apache 2.0.

Note: `.github/copilot-instructions.md` covers much of the same ground and should be kept consistent with this file.

## Commands

### Build

```sh
make build                              # all binaries -> ./dist/{os}_{arch}/release/
make build GOOS=linux GOARCH=amd64      # cross-compile
make DEBUG=1 build                      # unoptimized -> ./dist/{os}_{arch}/debug/
cd cmd/daprd && go build -tags=allcomponents -v   # fastest iteration on the sidecar
```

Binaries built: `daprd placement operator injector sentry scheduler` (`BINARIES` in the Makefile).
`DAPR_SIDECAR_FLAVOR` selects the component set build tag: `allcomponents` (default) or `stablecomponents`.

### Test

```sh
make test                                              # unit tests (needs gotestsum: go install gotest.tools/gotestsum@latest)
go test -tags=unit,allcomponents ./pkg/actors/...      # single package, no gotestsum needed
go test -tags=unit,allcomponents -run TestFoo ./pkg/actors/...
make test-race                                         # -race over an explicit allow-list of packages (CGO_ENABLED=1)
```

Unit test files carry `//go:build unit`; integration test files carry `//go:build integration`.

### Integration tests

Run against a real `daprd` binary built from source inside the test. `CGO_ENABLED=1` required.

```sh
make test-integration                 # serial
make test-integration-parallel        # what CI runs
go test -v -race -tags integration ./tests/integration -focus sentry
go test -v -race -tags integration ./tests/integration -focus scheduler/authz --count=100 -failfast
```

`-focus` takes a Go regex matched against `<suite subdir>/<StructName>` — this is the normal way to narrow a run, not `-run`. Useful env vars: `DAPR_INTEGRATION_DAPRD_PATH` / `_PLACEMENT_PATH` / `_SENTRY_PATH` (use a prebuilt binary instead of compiling), `DAPR_INTEGRATION_LOGS=true` (always print binary logs; otherwise only on failure), `DAPR_INTEGRATION_WORKFLOW_CLUSTERED` / `DAPR_INTEGRATION_WORKFLOW_FASTPATH` (run the workflow suite under those preview features — CI runs both as matrix legs).

Each scenario runs against a fresh daprd and must finish in seconds (30s hard ceiling; framework timeout is 45s per case). See `tests/docs/writing-integration-test.md`.

### Lint / format

```sh
make lint          # golangci-lint v2.10.1 exactly, --build-tags=allcomponents,subtlecrypto
make lint-fix
make format        # gofumpt + goimports -local github.com/dapr/ + modtidy-all
make modtidy       # root go.mod only
make check         # format + test + lint + fail if the tree is dirty afterwards
make me prettier   # prettier over JS/TS/JSON (only if you touched those)
```

Using a golangci-lint other than v2.10.1 produces spurious errors.

**Toolchain mismatch:** CI builds with the version in `go.mod` (`go-version-file: go.mod`). If the local Go toolchain is *newer* than that, golangci-lint drowns in bogus `typecheck` errors — `could not import sync ... export data version 4 is greater than maximum supported version 2` — because its bundled `go/types` cannot read the newer export data. Fix by pinning the run to the repo's toolchain, which is also what CI does:

```sh
GOTOOLCHAIN=go1.26.6 make lint     # use whatever go.mod says
```

### Codegen

```sh
make init-proto     # protoc 34.1, protoc-gen-go v1.32.0, protoc-gen-go-grpc 1.3.0, protoc-gen-connect-go 1.18.1
make gen-proto      # dapr/proto/**/*.proto -> pkg/proto/** (committed; never hand-edit)
make code-generate  # controller-gen for the CRD types in pkg/apis
```

### E2E / perf

Require a Kubernetes cluster with Dapr installed; targets live in `tests/dapr_tests.mk` (`make e2e-build-deploy-run`, `test-e2e-all`, …). Normally CI-only — see `tests/docs/running-e2e-test.md`.

## Architecture

### Binaries

`cmd/<binary>/main.go` is a thin shell over `cmd/<binary>/app/app.go`, which parses `cmd/<binary>/options`, builds a `security` provider (SPIFFE/mTLS via Sentry), and composes long-running goroutines with `concurrency.NewRunnerManager(...)` from `github.com/dapr/kit`. That runner-manager pattern (each subsystem is a `func(context.Context) error`, all cancel together) is used ~34 places across the tree and is the standard way to add a background service.

- **daprd** — the sidecar; everything else in the control plane exists to serve it.
- **placement** — actor placement table + consistent hashing (`pkg/placement`, `pkg/actors/hashing`).
- **scheduler** — durable jobs/reminders store and dispatch (`pkg/scheduler`); daprd streams jobs from it.
- **sentry** — CA issuing SPIFFE identities for mTLS (`pkg/sentry`, `pkg/security`).
- **operator** — serves Dapr CRDs (Component, Configuration, Subscription, Resiliency, HTTPEndpoint, …) to sidecars in Kubernetes mode (`pkg/operator`, types in `pkg/apis`).
- **injector** — Kubernetes admission webhook that injects the daprd sidecar (`pkg/injector`).

`pkg/modes` splits behaviour between `kubernetes` (resources come from the operator over gRPC) and `standalone` (resources loaded from `--resources-path` on disk).

### Sidecar runtime (`pkg/runtime`)

`runtime.FromConfig` → `DaprRuntime.Run` → `initRuntime`. The moving parts:

- **registry** (`pkg/runtime/registry`) — the set of component *constructors* available in this binary. Components self-register into `pkg/components/<type>.DefaultRegistry` from `init()` functions in `cmd/daprd/components/*.go`; each of those files is gated by `//go:build allcomponents || stablecomponents`, which is how the sidecar flavor is enforced. Adding a component from `components-contrib` means adding one such file.
- **processor** (`pkg/runtime/processor`) — turns a declarative resource (`Component`, `Subscription`, `HTTPEndpoint`, …) into a live, initialized object; one sub-package per building block.
- **compstore** (`pkg/runtime/compstore`) — the in-memory store of everything currently loaded; the read side for the metadata API and for hot reload.
- **hotreload** (`pkg/runtime/hotreload`) — watches the loader (operator or disk), diffs against compstore (`differ`), and applies adds/updates/deletes through the processor.
- **channels** — app channel (HTTP/gRPC to the user application) and the reverse direction.
- API surface (`pkg/api`): `http/` and `grpc/` are two front-ends over shared handlers in `pkg/api/universal`. Behaviour that is not protocol-specific belongs in `universal`. Dapr-typed errors live in `pkg/api/errors`.
- **messaging** (`pkg/messaging`) — service invocation between sidecars over internal gRPC (`pkg/proto/internals/v1`), plus gRPC proxying.
- **resiliency** (`pkg/resiliency`) — retry/circuit-breaker/timeout policies wrapped around component and app calls.
- **diagnostics** (`pkg/diagnostics`) — OpenTelemetry tracing and metrics.
- **healthz** (`pkg/healthz`) — a shared registry of readiness targets; subsystems take a `healthz.Target` and report their own readiness.

### Actors and workflows

`pkg/actors` is the virtual-actor framework: `table` (hosted actors), `router` (dispatch local vs. remote), `internal/placement` (client for the placement service), `reminders`/`timers` (backed by the scheduler service), `state`, `hashing`. Actor types are pluggable through `targets.Interface` / `targets.Factory` (`pkg/actors/targets`).

Workflows are built *on top of* actors: `pkg/runtime/wfengine` embeds `github.com/dapr/durabletask-go`, with `backends/actors` persisting orchestration state, and `pkg/actors/targets/workflow` providing the orchestrator/activity actor targets. The app connects over a work-item stream; the engine does not fully start until then (see `pkg/runtime/wfengine/README.md`).

### Integration test framework (`tests/integration`)

A test case is a struct implementing `suite.Case` (`Setup(*testing.T) []framework.Option`, `Run(*testing.T, context.Context)`), registered via `suite.Register(new(MyCase))` in an `init()`. Its package must be blank-imported in `tests/integration/import.go` or the `init` never runs. The struct name plus its `tests/integration/suite/...` path becomes the test name. `tests/integration/framework/` holds the reusable processes (daprd, sentry, placement, scheduler, kubernetes mock, gRPC/HTTP apps) — extend it there rather than shelling out from a case. `tests/integration/suite/ports/ports.go` is the canonical minimal example.

Conventions maintainers enforce in review (all three cost a round on #10472):

- **No package-level consts for test values.** Cases share a package, so a bare `const numInFlight = 2` is ambiguous shared state. Put it on the case struct, set it in `Setup`, and move the comment onto the field.
- **Don't roll your own timeout in `Run`.** `RunIntegrationTests` already hands each case a `ctx` with a 45s deadline (`tests/integration/integration.go`). A local `context.WithTimeout` is redundant and hides the framework's own mechanism — use the `ctx` parameter.
- **Add release notes when the fix will be backported.** The entry goes in `docs/release_notes/vX.Y.Z.md` for the target line, following the existing Problem / Impact / Root Cause / Solution shape.

## Conventions

- Every new Go file needs the Apache 2.0 header (`Copyright <YEAR> The Dapr Authors`, full block — copy from any existing file).
- Logging: `logger.NewLogger("dapr.<package>")` from `github.com/dapr/kit/logger`.
- Tests: `testify/require` for anything fatal, `testify/assert` only for non-fatal checks; table-driven; mocks generated by `mockgen` into `mock/` sub-packages, with hand-written `fake/` packages used widely for the newer code.
- Imports in three `goimports` groups with `-local github.com/dapr/`.
- Errors: stdlib `errors`/`fmt.Errorf`; prefer the typed errors in `pkg/api/errors` for anything user-facing on the API.
- `.golangci.yml` has a `depguard` deny-list that enforces the blessed dependency for several common needs — notably `dapr/kit/logger` over logrus, `dapr/kit/ptr` over `agrea/ptr`, stdlib `sync/atomic` over `go.uber.org/atomic`, stdlib errors over `pkg/errors`, `k8s.io/utils/clock` over `benbjohnson/clock`, `sigs.k8s.io/yaml`/`yaml.v3` over `ghodss/yaml`/`yaml.v2`, `lestrrat-go/jwx/v2` over `golang-jwt/jwt`. Check that list before adding a dependency.
- Every commit must be DCO signed: `git commit -s`. PRs cannot merge without it.
- `pkg/proto/**` and generated clientsets are excluded from formatting/linting and must not be edited by hand.

## CI gotchas

`.github/workflows/dapr.yml` runs `golangci-lint-action`, `make modtidy check-diff`, `govulncheck`, unit tests on Linux/Windows/macOS, `make test-integration-parallel` (plus a `-focus workflow` matrix leg for the preview-feature modes), and multi-platform builds. Most red CI comes from: a stale `go.mod`/`go.sum` (run `make modtidy-all` — there are several go.mod files, including `.build-tools/` and test apps), a missing license header, or a missing `//go:build unit` / `//go:build integration` tag on a new test file.

**Before blaming a red check on your change**, work out whose failure it is:

- Which *step* the job failed at (`gh api repos/dapr/dapr/actions/jobs/<id> --jq '.steps[] | select(.conclusion=="failure")'`). A failure before the tests run is not your code.
- Whether unrelated PRs fail the same way (`gh api repos/dapr/dapr/actions/workflows/<id>/runs`).
- Whether the failing test *changes between runs* — that is flakiness, not a regression.
- `gh run view --job <id> --log` for the real error; `--log-failed` often returns only the cleanup tail.

Locally, `make test-integration-parallel` is flaky on macOS: `subscriptions` and `shutdown/graceful` cases fail intermittently under CPU contention while passing in serial. **The tell is the timing ratio** — a case taking 36s in parallel against 15s isolated is starving, not broken. Re-run with `make test-integration ARGS="-focus '<name>'"` before investigating.
