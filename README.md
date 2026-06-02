# Krateo Blueprint — Rancher

A [Krateo](https://krateo.io) blueprint that installs [Rancher](https://www.rancher.com/) on a
Kubernetes cluster using its official Helm chart. Once the `CompositionDefinition` is applied,
**every `RancherInstaller` Composition you create becomes a self-contained Rancher installer**.

It follows the upstream procedure documented here:
<https://ranchermanager.docs.rancher.com/getting-started/installation-and-upgrade/install-upgrade-on-a-kubernetes-cluster#kubernetes-cluster>

## How it works

This blueprint is a Helm chart named `rancher-installer` that wraps the upstream Rancher chart as
a dependency:

| Dependency  | Version  | Repository                                          |
| ----------- | -------- | --------------------------------------------------- |
| `rancher`   | `2.14.2` | `https://releases.rancher.com/server-charts/stable` |

When a `RancherInstaller` Composition is created, Krateo renders and installs this chart into the
`cattle-system` namespace.

> **cert-manager is NOT bundled.** It is a **required prerequisite** that must be installed
> separately, before this composition (see [Prerequisites](#prerequisites)).

> **Chart name vs. Kind.** The chart is intentionally named `rancher-installer` (→ Kind
> `RancherInstaller`). It must NOT be named `rancher`, because the top-level values key for the
> wrapped subchart is also `rancher`; if the chart name matched, Krateo's `crdgen` would emit a
> self-referential `spec.rancher.spec` and CRD generation would fail with
> *"must not be empty for specified object fields"*.

## Prerequisites

- A Kubernetes cluster with an Ingress controller (e.g. ingress-nginx).
- Krateo `core-provider` installed (tested with `1.0.0`).
- **cert-manager — REQUIRED — installed separately, _before_ this composition** (see below). The
  only exception is terminating TLS externally with your own certificate
  (`rancher.ingress.tls.source: secret`).
- A DNS record (or `/etc/hosts` entry) for the chosen `rancher.hostname` pointing at your ingress.

### cert-manager is a required, separately-installed prerequisite

Rancher renders an `Issuer` (`cert-manager.io/v1`) when `ingress.tls.source` is `rancher` or
`letsEncrypt`. That CRD must already exist in the cluster at render time, so cert-manager
**cannot** be installed in the same Helm release as Rancher (Helm would fail with *"ensure CRDs
are installed first"*). This is exactly why the official Rancher docs install cert-manager as a
distinct step — and why this blueprint does **not** bundle it. Install it first:

```sh
helm repo add jetstack https://charts.jetstack.io
helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager --create-namespace \
  --version v1.20.2 --set crds.enabled=true
kubectl rollout status deploy/cert-manager-webhook -n cert-manager
```

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

> cert-manager is **not** configured here — it is a separate prerequisite (see above).

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
```

> Make sure cert-manager is already installed (see [Prerequisites](#prerequisites)) before
> applying this, unless `ingress.tls.source: secret`.

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

`.github/workflows/release-tag.yaml` packages the chart (bundling the Rancher subchart) and
pushes it to GitHub Container Registry as an OCI Helm artifact whenever a semver tag is pushed:

```sh
git tag 0.1.0
git push origin 0.1.0
# -> oci://ghcr.io/braghettos/charts/rancher-installer:0.1.0
```

`.github/workflows/lint.yaml` runs `helm lint` + `helm template` on every pull request.

> The published GHCR package is **public**, so it can be pulled without credentials. If you
> republish it as private, configure the `credentials` block in `compositiondefinition.yaml`.

## Why `rancher.nameOverride` is required

`rancher.nameOverride: server` is set by default and **must not be removed**. While enumerating
the chart's resources, Krateo's `chart-inspector` creates scaffolding RBAC (ServiceAccount,
ClusterRole, ClusterRoleBinding) named after the Helm release (`<release>`). Rancher's own
cluster-admin `ClusterRoleBinding` is named from `rancher.fullname`, which **collapses to the
bare release name** when the release name contains "rancher" (it does — the composition is named
`rancher`). The two names collide and the install fails with *"ClusterRoleBinding … exists and
cannot be imported into the current release"*. Setting `nameOverride` makes Rancher's resources
`<release>-server-*`, avoiding the collision.

## Verified

End-to-end on a kind cluster with `core-provider 1.0.0` and cert-manager `v1.20.2` installed
separately beforehand:

- `helm lint` / `helm template` / `helm package`.
- `CompositionDefinition` reconciles to `Synced=True`, generates the `RancherInstaller` CRD and
  dynamic controller, and reaches `Ready=True`.
- A `RancherInstaller` Composition (`replicas: 1`, `tls.source: rancher`) reconciles to
  `Synced=True` / `Ready=True`, the Helm release is recorded, and Rancher's Deployment, Services
  and Ingress are created — Rancher pod reaches `1/1 Running`.
