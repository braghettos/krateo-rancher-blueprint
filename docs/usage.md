---
type: Usage
title: krateo-rancher-blueprint — usage
description: How to use the blueprint — install core-provider and cert-manager, register the CompositionDefinition, create a RancherInstaller Composition, reach the UI, and the Composable Portal card path.
resource: oci://ghcr.io/krateo-blueprints/charts/rancher-installer
tags: [rancher, install, composition, core-provider, portal]
timestamp: 2026-08-11T00:00:00Z
---

# Usage

## Prerequisites

- A Kubernetes cluster. For `ingress` exposure, an Ingress controller (e.g.
  ingress-nginx) — not needed when you expose Rancher via
  `service.type: NodePort`/`LoadBalancer`.
- Krateo `core-provider` installed (tested with `1.0.0`) — it turns the
  `CompositionDefinition` into the `RancherInstaller` CRD and renders Compositions.
- **cert-manager — REQUIRED, installed separately, _before_ this composition** (unless
  you terminate TLS externally with `ingress.tls.source: secret`). See below.
- A DNS record (or `/etc/hosts` entry) for the chosen `hostname` pointing at your ingress
  (or, for NodePort, at a node IP).

### Install core-provider and cert-manager

```console
$ helm repo add krateo https://charts.krateo.io
$ helm repo add jetstack https://charts.jetstack.io
$ helm repo update

$ helm upgrade --install core-provider krateo/core-provider \
    --version 1.0.0 -n krateo-system --create-namespace

$ helm upgrade --install cert-manager jetstack/cert-manager \
    -n cert-manager --create-namespace --version v1.20.2 --set crds.enabled=true
$ kubectl rollout status deploy/cert-manager-webhook -n cert-manager
```

cert-manager is separate on purpose: Rancher renders an `Issuer` (`cert-manager.io/v1`)
when `ingress.tls.source` is `rancher` or `letsEncrypt`, and that CRD must already exist
at render time — it cannot be installed in the same release as Rancher
([overview](./overview.md)).

## 1. Register the blueprint

```console
$ kubectl create namespace cattle-system
$ kubectl apply -f compositiondefinition.yaml
$ kubectl wait compositiondefinition/rancher -n cattle-system \
    --for=condition=Ready --timeout=180s
```

This publishes the `RancherInstaller` Composition type
(`composition.krateo.io/v0-1-0`, plural `rancherinstallers`).
`compositiondefinition.yaml` pulls the chart from
`oci://ghcr.io/krateo-blueprints/charts/rancher-installer`. If you republish that GHCR
package as private, add a `spec.chart.credentials` block referencing a pull-token secret
(the commented shape is in `compositiondefinition.yaml`).

## 2a. Create a Composition

Composition `spec` fields are **top-level** (this is the forked Rancher chart); the full
typed surface is in [configuration](./configuration.md).

```yaml
apiVersion: composition.krateo.io/v0-1-0
kind: RancherInstaller
metadata:
  name: rancher
  namespace: cattle-system
spec:
  hostname: rancher.my.org
  bootstrapPassword: "ChangeMe-please"
  replicas: 1
  ingress:
    tls:
      source: rancher          # needs cert-manager (separate prerequisite)
  service:
    type: NodePort             # optional: expose on a node port
    httpsNodePort: 30443       # optional: force the port number
```

```console
$ kubectl apply -f rancher-composition.yaml
$ kubectl wait rancherinstaller/rancher -n cattle-system \
    --for=condition=Ready --timeout=600s
```

## 2b. Or drive it from the Krateo Composable Portal

`customform.yaml` ships a portal card + `CustomForm` that renders the CRD schema as a
form and POSTs a `RancherInstaller`:

```console
$ kubectl apply -f customform.yaml
```

The card colours itself from the `CompositionDefinition`'s `Ready` condition, and the
form hides the advanced groups (`auditLog`, `networkExposure`, `additionalTrustedCAs`,
`antiAffinity`) so an operator sees only the everyday knobs.

## Accessing Rancher

Browse to `https://<hostname>` (or `https://<node>:<httpsNodePort>` for NodePort) and log
in as `admin`. Retrieve the bootstrap password with:

```console
$ kubectl get secret -n cattle-system bootstrap-secret \
    -o go-template='{{ "{{" }} .data.bootstrapPassword|base64decode {{ "}}" }}{{ "{{" }} "\n" {{ "}}" }}'
```

## End-to-end on kind

For a full local walkthrough — a kind cluster with `extraPortMappings`, a `RancherInstaller`
with `service.type: NodePort` and `service.httpsNodePort: 30443` pinned to a host port,
and the Rancher UI reachable at `https://localhost:30443` with **no port-forwarder** — see
[`quickstart.md`](../quickstart.md) and the runnable
[examples/kind-nodeport](../examples/kind-nodeport/README.md).
