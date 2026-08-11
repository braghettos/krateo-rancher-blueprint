---
type: Log
title: krateo-rancher-blueprint — log
description: Curated chronological history of krateo-rancher-blueprint — notable decisions and changes, not a generated changelog.
resource: oci://ghcr.io/krateo-blueprints/charts/rancher-installer
tags: [log, history, rancher]
timestamp: 2026-08-11T00:00:00Z
---

# Log

Curated history; release notes live in GitHub Releases.

## 2026-08-11 — Documentation Standard adoption

The repo adopts the Krateo Documentation Standard (OKF): the invariant `docs/` bundle
(`index`/`overview`/`usage`/`configuration`/`api`/`examples`/`release`/`log` + `llms.txt`),
a runnable `examples/kind-nodeport`, a README rewritten to the six-section shape, and the
shared `lint-docs` check wired into `.github/workflows/lint.yaml`. The pre-existing
`quickstart.md` and the `docs/*.png` diagrams are kept. Part of
`krateo-platformops/installer#52`.

## 0.1.0 — first release

The blueprint ships: the forked Rancher chart (`chart/`, name `rancher-installer`,
tracking Rancher `2.14.2`) and the sibling `CompositionDefinition` that registers the
`RancherInstaller` type, plus `customform.yaml` for the Composable Portal. Design
decisions worth keeping:

- **Fork, not wrapper.** `core-provider` builds the Composition CRD from the chart's
  `values.schema.json` only, and upstream Rancher ships only a partial schema (6 fields,
  `if/then` + type-unions crdgen cannot express). The fork replaces it with a curated,
  crdgen-suitable schema so essential fields like `hostname` are settable on a
  Composition. A wrapper could not fix this and could not patch Rancher's Service either.
- **`templates/service.yaml` NodePort patch.** The Composition can force the http/https
  NodePort number (`service.httpNodePort` / `service.httpsNodePort`), which is what makes
  the kind `https://localhost:30443` no-port-forwarder flow work.
- **`rancher-installer` naming.** The chart name yields Kind `RancherInstaller` and avoids
  a collision between `chart-inspector`'s release-named scaffolding RBAC and Rancher's own
  release-named cluster-admin `ClusterRoleBinding`.
- **cert-manager is a separate prerequisite.** Rancher's `Issuer` (`cert-manager.io/v1`)
  CRD must exist at render time, so cert-manager cannot be installed in the same release —
  it is installed beforehand (except with `ingress.tls.source: secret`).
