---
type: Architecture
title: krateo-rancher-blueprint — overview
description: What the blueprint is and how it is built — the fork-not-wrapper decision, the values.schema.json→CRD→Composition flow, the NodePort patch, the rancher-installer naming, and cert-manager as a required separate prerequisite.
resource: oci://ghcr.io/krateo-blueprints/charts/rancher-installer
tags: [blueprint, rancher, architecture, core-provider, cert-manager]
timestamp: 2026-08-11T00:00:00Z
---

# Overview

krateo-rancher-blueprint packages the [Rancher](https://www.rancher.com/) install as a
Krateo blueprint: a Helm chart plus the `CompositionDefinition`
(`core.krateo.io/v1alpha1`) that registers it with `core-provider`. Applying that
`CompositionDefinition` publishes a new API — `RancherInstaller`
(`composition.krateo.io/v0-1-0`) — and **every `RancherInstaller` Composition you then
create renders and installs a Rancher server** from the forked chart.

It follows the upstream procedure documented at
<https://ranchermanager.docs.rancher.com/getting-started/installation-and-upgrade/install-upgrade-on-a-kubernetes-cluster>.

## Why a fork, not a dependency/umbrella wrapper

The chart (`chart/`, name `rancher-installer`) is a **fork of the upstream Rancher Helm
chart** (rancher `2.14.2`), not a wrapper that lists Rancher as a subchart. Two things
force that decision, both spelled out in `chart/Chart.yaml`:

1. **`core-provider` builds the Composition CRD from the chart's `values.schema.json`
   only — it never reads `values.yaml`.** Upstream Rancher ships only a *partial* schema
   (6 fields, using `if/then` and type-unions that crdgen cannot express), so essential
   fields like `hostname` could not be set on a Composition. The fork replaces it with a
   curated, crdgen-suitable `chart/values.schema.json` that exposes the real install
   knobs ([configuration](./configuration.md)).
2. **`templates/service.yaml` is patched** so the NodePort number can be forced
   (`service.httpNodePort` / `service.httpsNodePort`) — the block guarded by the
   comment `Krateo blueprint patch`. A wrapper cannot reach inside its subchart's
   Service template.

A wrapper would have had to duplicate Rancher's schema anyway, and still could not patch
Rancher's Service, so forking keeps everything in one chart. The cost: **updating Rancher
= re-fork** (replace `templates/`, `scripts/`, `values.yaml` from the new upstream
version, re-apply the `service.yaml` NodePort patch, reconcile `values.schema.json`) —
see [release](./release.md).

## Why the name `rancher-installer` (not `rancher`)

The chart name drives the generated Kind, so `rancher-installer` → `RancherInstaller`.
The name also avoids colliding with the release-named scaffolding RBAC that
`chart-inspector` creates while enumerating the chart, which would otherwise clash with
Rancher's own release-named cluster-admin `ClusterRoleBinding` and fail the install.

## The flow: chart → CRD → Composition

```
chart/values.schema.json  ──(core-provider crdgen)──▶  RancherInstaller CRD
        │                                                      │
        └── chart/Chart.yaml: name → Kind, version → apiVersion │
                                                                ▼
     RancherInstaller Composition (spec) ──(core-provider renders chart)──▶ Rancher
```

- **`compositiondefinition.yaml`** points `core-provider` at the published chart
  (`oci://ghcr.io/krateo-blueprints/charts/rancher-installer`, version `0.1.0`).
- `core-provider` reads `chart/values.schema.json` and generates the `RancherInstaller`
  CRD; the Kind comes from `Chart.yaml`'s `name` (dashes dropped, CamelCased →
  `RancherInstaller`) and the apiVersion from its `version` (`0.1.0` →
  `composition.krateo.io/v0-1-0`). Full detail in [api](./api.md).
- Each `RancherInstaller` you create is a Composition: `core-provider` renders the forked
  chart with your `spec` as values and installs Rancher.

Because it is a fork, the Composition `spec` fields are **top-level** (`hostname`,
`replicas`, `ingress`, `service`, …) — they map straight onto the chart's own values.

## What the chart deploys

One render of `chart/templates/` produces a standard Rancher install: the Rancher server
`Deployment`, the patched `Service` (plus an internal `Service`), the `ServiceAccount`
and cluster-admin `ClusterRoleBinding`, a `PriorityClass`, the bootstrap `Secret`, and
the pre-upgrade / post-delete hook `Job`s (`scripts/`). Network exposure is one of three
modes chosen by `networkExposure.type`:

- **`ingress`** (default) — the classic `Ingress` under `templates/ingate/ingress.yaml`.
- **`gateway`** — the Gateway-API path (`templates/ingate/gateway.yaml` +
  `httproute.yaml`), for clusters fronted by a Gateway controller (k3s/rke2 Traefik).
- **`none`** — no ingress resource; reach Rancher through `service.type`
  (`NodePort` / `LoadBalancer`).

## cert-manager is a required, separately-installed prerequisite

Rancher renders an `Issuer` (`cert-manager.io/v1`) when `ingress.tls.source` is `rancher`
or `letsEncrypt` (`templates/ingate/issuer-rancher.yaml`,
`issuer-letsEncrypt.yaml`). That CRD must already exist at render time, so cert-manager
**cannot** be installed in the same release as Rancher — it must be installed first, as a
separate step ([usage](./usage.md)). The only exception is terminating TLS externally
with your own certificate (`ingress.tls.source: secret`).
