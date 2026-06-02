# Krateo Blueprint — Rancher

A [Krateo](https://krateo.io) blueprint that installs [Rancher](https://www.rancher.com/) on a
Kubernetes cluster using its official Helm chart. Once the `CompositionDefinition` is applied,
**every `RancherInstaller` Composition you create becomes a self-contained Rancher installer**.

It follows the upstream procedure documented here:
<https://ranchermanager.docs.rancher.com/getting-started/installation-and-upgrade/install-upgrade-on-a-kubernetes-cluster#kubernetes-cluster>

## How it works

This blueprint is an **umbrella Helm chart** named `rancher-installer` that wraps the upstream
Rancher chart as a dependency:

| Dependency     | Version   | Repository                                            |
| -------------- | --------- | ----------------------------------------------------- |
| `rancher`      | `2.14.2`  | `https://releases.rancher.com/server-charts/stable`   |
| `cert-manager` | `v1.20.2` | `https://charts.jetstack.io` (optional, off by default)|

When a `RancherInstaller` Composition is created, Krateo renders and installs this chart into the
`cattle-system` namespace.

> **Chart name vs. Kind.** The chart is intentionally named `rancher-installer` (→ Kind
> `RancherInstaller`). It must NOT be named `rancher`, because the top-level values key for the
> wrapped subchart is also `rancher`; if the chart name matched, Krateo's `crdgen` would emit a
> self-referential `spec.rancher.spec` and CRD generation would fail with
> *"must not be empty for specified object fields"*.

## Prerequisites

- A Kubernetes cluster with an Ingress controller (e.g. ingress-nginx).
- Krateo `core-provider` installed (tested with `1.0.0`).
- **cert-manager installed as a separate step, _before_ this composition** (unless you use
  `rancher.ingress.tls.source: secret` with your own certificate). See below.
- A DNS record (or `/etc/hosts` entry) for the chosen `rancher.hostname` pointing at your ingress.

### Why cert-manager is a separate prerequisite

Rancher renders an `Issuer` (`cert-manager.io/v1`) when `ingress.tls.source` is `rancher` or
`letsEncrypt`. That CRD must already exist in the cluster at render time. It **cannot** be
installed in the same Helm release as Rancher (Helm fails with *"ensure CRDs are installed
first"*). This is exactly why the Rancher docs install cert-manager as a distinct step. Install it
first:

```sh
helm repo add jetstack https://charts.jetstack.io
helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager --create-namespace \
  --version v1.20.2 --set crds.enabled=true
kubectl rollout status deploy/cert-manager-webhook -n cert-manager
```

The bundled `cert-manager` dependency (`cert-manager.enabled`) is therefore **disabled by
default**; leave it off and rely on the separate install above.

## Configuration

Key values (full schema in [`chart/values.schema.json`](chart/values.schema.json)):

| Value                         | Default              | Description                                                              |
| ----------------------------- | -------------------- | ------------------------------------------------------------------------ |
| `rancher.hostname`            | `rancher.example.com`| DNS name to reach the Rancher UI/API. **Required.**                      |
| `rancher.bootstrapPassword`   | `admin`              | Initial password for the local `admin` user.                            |
| `rancher.replicas`            | `3`                  | Replica count. Set to `1` for single-node clusters (kind/minikube).     |
| `rancher.ingress.tls.source`  | `rancher`            | `rancher` (self-signed via cert-manager) / `letsEncrypt` / `secret`.    |
| `rancher.letsEncrypt.email`   | `""`                 | Required when `ingress.tls.source: letsEncrypt`.                         |
| `rancher.privateCA`           | `false`             | Set `true` when the certificate is signed by a private CA.              |
| `cert-manager.enabled`        | `false`              | Leave off — install cert-manager separately (see above).                |

## How to install

### 1. Register the blueprint

```sh
kubectl create namespace cattle-system
kubectl apply -f compositiondefinition.yaml
```

This publishes a `RancherInstaller` Composition type (`composition.krateo.io/v0-1-0`, plural
`rancherinstallers`).

> `compositiondefinition.yaml` points at the OCI artifact
> `oci://ghcr.io/braghettos/charts/rancher-installer`. If the GHCR package is private, uncomment
> the `credentials` block and create the referenced pull-token secret.

### 2a. Create a Composition (the actual Rancher installer)

```yaml
apiVersion: composition.krateo.io/v0-1-0
kind: RancherInstaller
metadata:
  name: rancher
  namespace: cattle-system
spec:
  rancher:
    hostname: rancher.my.org
    bootstrapPassword: "ChangeMe-please"
    replicas: 1            # 1 for single-node; 3 for HA
    ingress:
      tls:
        source: rancher    # needs cert-manager (installed separately, see Prerequisites)
  cert-manager:
    enabled: false
```

```sh
kubectl apply -f rancher-composition.yaml
```

### 2b. Or use the Krateo Composable Portal

```sh
kubectl apply -f customform.yaml
```

A "rancher" card appears in the portal; filling the form creates the Composition for you.

## Accessing Rancher

Once Ready, browse to `https://<rancher.hostname>` and log in as `admin` with the bootstrap
password. To confirm the password:

```sh
kubectl get secret -n cattle-system bootstrap-secret \
  -o go-template='{{ "{{" }} .data.bootstrapPassword|base64decode {{ "}}" }}{{ "{{" }} "\n" {{ "}}" }}'
```

## Local development

```sh
# pull the subcharts into chart/charts/
helm dependency update ./chart
# render to verify
helm template rancher ./chart --namespace cattle-system
```

## Publishing (CI)

`.github/workflows/release-tag.yaml` packages the chart (bundling its subcharts) and pushes it
to GitHub Container Registry as an OCI Helm artifact whenever a semver tag is pushed:

```sh
git tag 0.1.0
git push origin 0.1.0
# -> oci://ghcr.io/braghettos/charts/rancher-installer:0.1.0
```

`.github/workflows/lint.yaml` runs `helm lint` + `helm template` on every pull request.

> The published GHCR package starts out **private**. Make it public (Package settings →
> Change visibility) for credential-free pulls, or keep it private and configure the
> `credentials` block in `compositiondefinition.yaml`.

## Known platform limitation

Krateo `core-provider`/`chart-inspector` `1.0.0` cannot complete the Rancher install: to
enumerate RBAC it performs a real Helm install of the chart, which leaves Rancher's
release-named cluster-admin `ClusterRoleBinding` in the cluster **without Helm ownership
metadata**. The subsequent install then aborts with *"ClusterRoleBinding … exists and cannot be
imported into the current release"*. This affects charts that combine cluster-scoped RBAC with
Helm `lookup` (Rancher does both) and is independent of this blueprint. Track upstream for a
chart-inspector fix.

## Verified

- `helm lint` / `helm template` / `helm package` (with bundled subcharts).
- On a kind cluster with `core-provider 1.0.0`: the `CompositionDefinition` reconciles to
  `Synced=True`, generates the `RancherInstaller` CRD and dynamic controller, reaches
  `Ready=True`, and accepts `RancherInstaller` Compositions — up to the platform limitation
  above.
