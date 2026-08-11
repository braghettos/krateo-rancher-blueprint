---
type: API
title: krateo-rancher-blueprint — API
description: The API this blueprint defines — the RancherInstaller CRD core-provider generates from the chart's values.schema.json, the CompositionDefinition that registers it, and the Kind/apiVersion derivation from Chart.yaml.
resource: oci://ghcr.io/krateo-blueprints/charts/rancher-installer
tags: [rancher, crd, compositiondefinition, api, core-provider]
timestamp: 2026-08-11T00:00:00Z
---

# API

This blueprint defines one Krateo API. It is not code you call directly — it is a
Composition type that `core-provider` generates from the chart and that you drive by
creating custom resources.

## `CompositionDefinition` — the registration (input)

`compositiondefinition.yaml` is a `CompositionDefinition` (`core.krateo.io/v1alpha1`). It
tells `core-provider` which chart to enumerate:

```yaml
apiVersion: core.krateo.io/v1alpha1
kind: CompositionDefinition
metadata:
  name: rancher
  namespace: cattle-system
spec:
  chart:
    url: oci://ghcr.io/krateo-blueprints/charts/rancher-installer
    version: "0.1.0"
    # If the GHCR package is private, provide pull credentials:
    # credentials:
    #   username: krateo
    #   passwordRef:
    #     key: token
    #     name: ghcr-credentials
    #     namespace: cattle-system
```

- `spec.chart.url` — the OCI Helm artifact published by
  `.github/workflows/release-tag.yaml`; the chart name (`rancher-installer`) is appended
  to the OCI repository path.
- `spec.chart.version` — the chart version; it also determines the generated apiVersion
  (see below), so `0.1.0` → `v0-1-0`. Keep it pointing at a version that exists.
- `spec.chart.credentials` — optional pull credentials, needed only if the GHCR package
  is private.

Applying it publishes the `RancherInstaller` CRD and makes `core-provider` reconcile it.

## `RancherInstaller` — the generated Composition CRD (output)

From `chart/Chart.yaml`, `core-provider` derives the API identity:

- **Kind** from `name`: `rancher-installer` → dashes dropped, CamelCased →
  `RancherInstaller`.
- **Group/apiVersion** from `version`: `0.1.0` → `composition.krateo.io/v0-1-0`.
- **Plural**: `rancherinstallers`.

The CRD's `spec` schema is generated from **`chart/values.schema.json`** — not from
`values.yaml`. That schema (`$schema: draft-07`, `type: object`, `required: [hostname]`,
`additionalProperties` implicitly open to the chart's defaults) declares exactly the
fields a Composition may set. Each becomes a validated spec field; see
[configuration](./configuration.md) for the full list. The generated `status` carries the
standard Krateo Composition conditions (`Synced`, `Ready`) that the portal card in
`customform.yaml` reads.

Inspect the generated CRD directly:

```console
$ kubectl get crd rancherinstallers.composition.krateo.io -o yaml
```

### A `RancherInstaller` resource

Creating one triggers `core-provider` to render the forked chart with `spec` as values
and install Rancher:

```yaml
apiVersion: composition.krateo.io/v0-1-0
kind: RancherInstaller
metadata:
  name: rancher
  namespace: cattle-system
spec:
  hostname: rancher.my.org       # required
  replicas: 1
  ingress:
    tls:
      source: rancher
  service:
    type: NodePort
    httpsNodePort: 30443
```

## What the rendered chart produces

A `RancherInstaller` reconciled by `core-provider` yields the Rancher install manifests
(`chart/templates/`): the Rancher server `Deployment`, the patched `Service` (+ an
internal `Service`), a `ServiceAccount` and cluster-admin `ClusterRoleBinding`, a
`PriorityClass`, the bootstrap `Secret`, and — depending on `networkExposure.type` —
either an `Ingress` (`ingate/ingress.yaml`) or a Gateway-API `Gateway` + `HTTPRoute`
(`ingate/gateway.yaml`, `httproute.yaml`), plus the cert-manager `Issuer`
(`ingate/issuer-*.yaml`) when `ingress.tls.source` is `rancher`/`letsEncrypt`. Lifecycle
`Job`s (pre-upgrade, post-delete) come from `scripts/`.

## Portal surface (`customform.yaml`)

For the Composable Portal, `customform.yaml` registers a `Widget` (a card) and a
`CustomForm` (`templates.krateo.io/v1alpha1`). The `CustomForm` reads the generated CRD's
OpenAPI schema
(`/apis/apiextensions.k8s.io/v1/customresourcedefinitions/rancherinstallers.composition.krateo.io`),
renders it as a form (hiding `auditLog`, `networkExposure`, `additionalTrustedCAs`,
`antiAffinity`), and POSTs a `RancherInstaller`. The card reads the `CompositionDefinition`'s
`Ready` condition to colour itself.
