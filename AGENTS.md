[comment]: # ( Copyright Contributors to the Open Cluster Management project )
# AGENTS.md

This file provides guidance to coding agents (e.g. Claude Code, claude.ai/code) when working with code in this repository.

## Repository purpose

Go module `open-cluster-management.io/multicluster-controlplane` — a **standalone, lightweight OCM (Open Cluster Management) hub control plane**. Rather than running OCM as a set of operators on top of a "real" Kubernetes cluster, this packages the OCM hub apiserver (with its CRDs, aggregator, controllers, and embedded etcd) into a single binary so it can be started in seconds, run multi-tenant inside a host cluster (one OCM instance per namespace), or even run as a fully standalone binary off-cluster.

Headline benefits (from `README.md`):
- Quick startup (control plane instance in seconds).
- Multi-tenant (multiple OCM hubs in one host cluster, one per namespace).
- Edge friendly (small footprint; supports running on `*ks`-style managed Kubernetes platforms or standalone).
- Combines OCM registration + work agents into a single entity to reduce agent footprint.

The produced binary is `multicluster-controlplane server`. Fork is mirrored to `kluster-management/multicluster-controlplane`; **upstream is `open-cluster-management-io/multicluster-controlplane`** and this repo tracks it.

## Architecture

- `cmd/server/main.go` — entry point; brings up the aggregated apiserver, registers OCM types, and starts the embedded agent.
- `pkg/servers/`:
  - `server.go` — top-level server lifecycle.
  - `kubeapiserver.go` — the kube apiserver instance (configured to *only* host OCM groups).
  - `aggregator.go` — kube-aggregator config.
  - `apiextensions.go` — apiextensions-apiserver config (so CRDs work without a separate kube cluster).
  - `simplerestoptionsfactroy.go` — REST-storage options shim (note the typo is intentional / load-bearing for compat).
  - `options/` — start options.
  - `configs/` — server config defaults.
- `pkg/etcd/` — embedded etcd configuration. A standalone hub uses this; in-cluster deployments can point at an external etcd via deploy manifests.
- `pkg/agent/`:
  - `agent.go` — the unified registration+work agent.
  - `crds/` — agent-side CRD assets.
- `pkg/certificate/` — TLS material generation for the hub apiserver, aggregator, and agents.
- `pkg/cmd/` — top-level Cobra command tree.
- `pkg/controllers/` — controllers run in-process.
- `pkg/util/` — shared helpers.
- `plugin/admission/` — embedded admission plugins (used to enforce OCM-specific policies).
- `charts/` — Helm charts for hub install on a host cluster.
- `test/`:
  - `bin/` — helper scripts.
  - `e2e/` — end-to-end suite.
  - `integration/` — integration tests against an in-process apiserver.
  - `performance/` — scale/perf tests.
- `Dockerfile` — release image.
- `Makefile` — local Go toolchain harness (no AppsCode Docker wrapper).
- `arch.png` — architecture diagram (referenced from the README).

## Common commands

This repo uses a **local Go toolchain**.

- `make all` — `clean vendor build run`. Local dev one-shot.
- `make build` — `vendor`, then build the binary.
- `make build-bin-release` — release binary.
- `make run` — `vendor build`, then run the controlplane locally.
- `make vendor` — refresh vendor.
- `make clean` — clean build artifacts.
- `make image` / `make push` — build / push the Docker image.
- `make verify` (alias `verify-gocilint`) — golangci-lint.
- `make check` (alias `check-copyright`) — verify "Copyright Contributors to the Open Cluster Management project" header is on every source file.
- `make update` — codegen / manifest update.
- `make deploy-etcd` — deploy a backing etcd instance (used before `make deploy`).
- `make deploy` — deploy the controlplane to the current kube context.
- `make destroy` — undeploy.
- `make deploy-all` — `deploy deploy-work-manager-addon deploy-managed-serviceaccount-addon deploy-policy-addon`.
- `make deploy-work-manager-addon` / `…-managed-serviceaccount-addon` / `…-policy-addon` — install a specific OCM addon onto the deployed controlplane.

Tests:

- `make build-e2e-test` / `make test-e2e` / `make cleanup-e2e` — e2e flow.
- `make build-performance-test` / `make test-performance` — performance suite.
- `make test-integration` / `make cleanup-integration` — controller integration suite.

Run a single Go test:

```
go test ./pkg/servers/... -run TestName -v
```

## Conventions

- Module path is `open-cluster-management.io/multicluster-controlplane` (**upstream**); imports must use that.
- **Upstream-tracking** fork (mirrored as `kluster-management/multicluster-controlplane`). Prefer rebasing onto upstream over diverging; isolate AppsCode-only patches.
- License: Apache-2.0 (`LICENSE`). Every source file must carry `// Copyright Contributors to the Open Cluster Management project` — `make check-copyright` is wired into the pre-commit hook (`.git/hooks/pre-commit` runs `make check`).
- Sign off commits (`git commit -s`); contributions follow the DCO (`DCO`, `CONTRIBUTING.md`).
- Vendor directory is checked in **and** is a Make prerequisite (`make build` runs `vendor` first). Don't delete `vendor/` unless you also update the targets.
- `pkg/servers/simplerestoptionsfactroy.go` has a misspelling in the filename that is preserved for upstream-merge compat. Don't "fix" it.
- The control plane is intentionally **monolithic**: kube-apiserver + apiextensions + aggregator + agent + (optionally) etcd in one process. Resist the urge to split them; the small footprint is the whole point.
- Multi-tenancy lives at the namespace level — multiple OCM instances in the same host cluster use distinct namespaces. Don't introduce cluster-scoped singletons.
