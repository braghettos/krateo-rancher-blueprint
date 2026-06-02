# Krateo Blueprint — Rancher

A [Krateo](https://krateo.io) blueprint that installs [Rancher](https://www.rancher.com/) on a
Kubernetes cluster. Once the `CompositionDefinition` is applied, **every `RancherInstaller`
Composition you create becomes a Rancher installer**.

It follows the upstream procedure documented here:
<https://ranchermanager.docs.rancher.com/getting-started/installation-and-upgrade/install-upgrade-on-a-kubernetes-cluster#kubernetes-cluster>

## How it works

This blueprint is a **fork of the upstream Rancher Helm chart** (rancher `2.14.2`), renamed
`rancher-installer`, with two changes:

1. **A complete, crdgen-suitable `values.schema.json`.** Krateo's `core-provider` builds the
   Composition CRD **only** from the chart's `values.schema.json` (it never reads `values.yaml`).
   Rancher ships only a *partial* schema (6 fields, using `if/then` and type-unions that crdgen
   can't express), so essential fields like `hostname` could not be set on a Composition. This
   fork replaces it with a curated schema that exposes the real install knobs.
2. **A patched `templates/service.yaml`** that lets you force the NodePort number
   (`service.httpNodePort` / `service.httpsNodePort`).

> **Why a fork and not a dependency/umbrella wrapper?** core-provider reads only the root
> `values.schema.json`; a wrapper's schema would have to duplicate Rancher's anyway, and you
> still couldn't patch Rancher's Service. Forking keeps everything in one chart. **Updating
> Rancher = re-fork** (replace `templates/`, `scripts/`, `values.yaml` from the new upstream
> version, re-apply the `service.yaml` NodePort patch, reconcile `values.schema.json`).

> **Why the name `rancher-installer` (not `rancher`)?** The Kind becomes `RancherInstaller`, and
> the name avoids colliding with the release-named scaffolding RBAC that chart-inspector creates
> while enumerating the chart (which would otherwise clash with Rancher's own release-named
> cluster-admin ClusterRoleBinding and fail the install).

## Prerequisites

- A Kubernetes cluster with an Ingress controller (e.g. ingress-nginx) — unless you expose
  Rancher via `service.type: NodePort`/`LoadBalancer`.
- Krateo `core-provider` installed (tested with `1.0.0`).
- **cert-manager — REQUIRED — installed separately, _before_ this composition** (see below). The
  only exception is terminating TLS externally with your own certificate
  (`ingress.tls.source: secret`).
- A DNS record (or `/etc/hosts` entry) for the chosen `hostname` pointing at your ingress
  (or node) — for NodePort, point it at a node IP.

### cert-manager is a required, separately-installed prerequisite

Rancher renders an `Issuer` (`cert-manager.io/v1`) when `ingress.tls.source` is `rancher` or
`letsEncrypt`. That CRD must already exist at render time, so cert-manager **cannot** be installed
in the same Helm release as Rancher. Install it first:

```sh
helm repo add jetstack https://charts.jetstack.io
helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager --create-namespace \
  --version v1.20.2 --set crds.enabled=true
kubectl rollout status deploy/cert-manager-webhook -n cert-manager
```

## Configuration

Composition `spec` fields are **top-level** (this is the forked Rancher chart). Full schema in
[`chart/values.schema.json`](chart/values.schema.json). Highlights:

| Value                    | Default              | Description                                                          |
| ------------------------ | -------------------- | ------------------------------------------------------------------- |
| `hostname`               | `rancher.example.com`| DNS name to reach Rancher. **Required.**                            |
| `bootstrapPassword`      | _(random)_           | Initial password for the local `admin` user.                       |
| `replicas`               | `3`                  | Replica count. Use `1` for single-node clusters (kind/minikube).   |
| `ingress.tls.source`     | `rancher`            | `rancher` / `letsEncrypt` / `secret`.                              |
| `ingress.ingressClassName`| `""`                | Ingress class (empty = cluster default).                           |
| `service.type`           | `ClusterIP`          | `ClusterIP` / `NodePort` / `LoadBalancer`.                         |
| `service.httpsNodePort`  | _(auto)_             | Force the https node port (30000–32767), e.g. for kind.            |
| `service.httpNodePort`   | _(auto)_             | Force the http node port.                                          |
| `letsEncrypt.email`      | —                    | Required when `ingress.tls.source: letsEncrypt`.                    |
| `privateCA`              | `false`             | Certificate signed by a private CA.                                |

## How to install

### 1. Register the blueprint

```sh
kubectl create namespace cattle-system
kubectl apply -f compositiondefinition.yaml
```

This publishes a `RancherInstaller` Composition type (`composition.krateo.io/v0-1-0`, plural
`rancherinstallers`). `compositiondefinition.yaml` pulls the chart from
`oci://ghcr.io/braghettos/charts/rancher-installer` (make that GHCR package public, or set the
`credentials` block).

### 2a. Create a Composition

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
      source: rancher      # needs cert-manager (separate prerequisite)
  service:
    type: NodePort         # optional: expose on a node port
    httpsNodePort: 30443   # optional: force the port number
```

```sh
kubectl apply -f rancher-composition.yaml
```

### 2b. Or use the Krateo Composable Portal

```sh
kubectl apply -f customform.yaml
```

## Accessing Rancher

Browse to `https://<hostname>` (or `https://<node>:<httpsNodePort>` for NodePort) and log in as
`admin`. Retrieve the bootstrap password with:

```sh
kubectl get secret -n cattle-system bootstrap-secret \
  -o go-template='{{ "{{" }} .data.bootstrapPassword|base64decode {{ "}}" }}{{ "{{" }} "\n" {{ "}}" }}'
```

See [`quickstart.md`](quickstart.md) for an end-to-end test on kind (NodePort pinned to a host
port, no port-forwarder).

## Publishing (CI)

`.github/workflows/release-tag.yaml` packages the chart and pushes it to GHCR as an OCI Helm
artifact on every semver tag (`git tag 0.1.0 && git push origin 0.1.0` →
`oci://ghcr.io/braghettos/charts/rancher-installer:0.1.0`). `.github/workflows/lint.yaml` runs
`helm lint` + `helm template` on every PR.

## Verified

End-to-end on a kind cluster (`core-provider 1.0.0`, cert-manager `v1.20.2` installed separately):

- `helm lint` / `helm template` / `helm package`.
- `CompositionDefinition` reconciles to `Synced=True` / `Ready=True`; the CRD exposes the full
  set of install fields top-level.
- A `RancherInstaller` Composition (`replicas: 1`, `service.type: NodePort`,
  `service.httpsNodePort: 30443`) reconciles to `Synced=True`, Rancher's Service is `NodePort`
  with the **forced** node port `443:30443`, and the Rancher pod reaches `1/1 Running`.
- With kind started with `extraPortMappings` (`30443:30443`), the Rancher UI is reachable at
  `https://localhost:30443` directly — no port-forwarder.
