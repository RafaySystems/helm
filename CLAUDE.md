# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
make build                             # Build helm binary to bin/helm
make test                              # Full suite: style check + unit tests
make test-unit                         # Unit tests only (race detection, shuffled)
make test-style                        # golangci-lint + license header validation
make format                            # Format code with goimports
make gen-test-golden                   # Regenerate golden files in testdata/

go test -run TestName ./pkg/action     # Run a single test by name
make test-coverage PKG=./pkg/action    # Coverage for a specific package
```

Tests run with `-shuffle=on -count=1 -race -v`. Golden files in `testdata/` directories capture expected CLI output; regenerate with `make gen-test-golden` when behavior changes intentionally.

All commits require DCO sign-off: `git commit -s`.

## Architecture

Helm is a Kubernetes package manager. It installs, upgrades, and manages **charts** (bundles of Kubernetes manifests with templating) as **releases** (deployed chart instances tracked in the cluster).

### Request flow

```
CLI args
  → cmd/helm/             (auth plugin init, root cobra command)
  → pkg/cmd/              (per-command cobra implementations, flag binding)
  → pkg/action/           (business logic: Install, Upgrade, Rollback, List, …)
  → pkg/engine/           (render Go+Sprig templates into Kubernetes manifests)
  → pkg/kube/             (apply/diff/status-watch manifests via client-go)
  → pkg/storage/          (persist release records to Secrets/ConfigMaps/SQL)
```

### Key packages

| Package | Role |
|---|---|
| `pkg/action/` | One struct per operation (e.g. `Install`, `Upgrade`). All share a `Configuration` that holds the kube client, storage backend, and capabilities. This is the primary public SDK surface. |
| `pkg/cmd/` | ~100 Cobra commands. Thin bridges: parse flags, call `pkg/action/`, format output. |
| `pkg/chart/v2/` | Stable chart format (handles both apiVersion v1 and v2 charts). |
| `internal/chart/v3/` | Next-gen chart format, under active development. |
| `pkg/engine/` | Template rendering using Go `text/template` + Sprig. Supports post-renderers. |
| `pkg/kube/` | Client abstraction over `k8s.io/client-go`. `pkg/kube/fake/` provides test doubles. |
| `pkg/storage/` | Pluggable release backends (Secrets, ConfigMaps, SQL, in-memory). Release records are stored as `sh.helm.release.v1` typed secrets/configmaps. |
| `pkg/release/v1/` | Stable release types and interfaces. |
| `internal/release/v2/` | Release types for chart v3. |
| `pkg/repo/` | Chart repository index management and downloads. |
| `pkg/registry/` | OCI registry support (push/pull charts). Uses `oras.land/oras-go/v2`. |
| `internal/plugin/` | WASM-based plugin system (via `tetratelabs/wazero`). |
| `internal/gates/` | Feature gate mechanism for in-progress features. |

### Chart versions vs release versions

There are two parallel evolutionary paths:
- **Chart v2** (`pkg/chart/v2/`) + **Release v1** (`pkg/release/v1/`) — stable, shipping today.
- **Chart v3** (`internal/chart/v3/`) + **Release v2** (`internal/release/v2/`) — in development, lives under `internal/` until stable.

### Backward compatibility

Public API signatures in `pkg/` must not change in breaking ways (per [HIP-0004](https://github.com/helm/community/blob/main/hips/hip-0004.md)). CLI commands and flags must not be removed or renamed in ways that break existing scripts. Security fixes are the only permitted exception.

Bug fixes go to `main` first, then are backported to `dev-v3` (Helm v3, supported until July 2026).

### Testing patterns

- Table-driven tests with `github.com/stretchr/testify`.
- `pkg/kube/fake/` for mocking Kubernetes — do not hit a real cluster.
- Complex CLI output is validated against golden files in `testdata/`.
- `internal/test/` provides shared test helpers.
