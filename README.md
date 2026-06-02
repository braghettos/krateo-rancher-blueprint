# Krateo Blueprint — Rancher

A [Krateo](https://krateo.io) blueprint that installs [Rancher](https://www.rancher.com/) on a
Kubernetes cluster using its official Helm chart. Once the `CompositionDefinition` is applied,
**every `Rancher` Composition you create becomes a self-contained Rancher installer**.

It follows the upstream procedure documented here:
<https://ranchermanager.docs.rancher.com/getting-started/installation-and-upgrade/install-upgrade-on-a-kubernetes-cluster#kubernetes-cluster>

## How it works

This blueprint is an **umbrella Helm chart** that wraps two upstream charts as dependencies:

| Dependency     | Version   | Repository                                            |
| -------------- | --------- | ----------------------------------------------------- |
| `rancher`      | `2.14.2`  | `https://releases.rancher.com/server-charts/stable`   |
| `cert-manager` | `v1.20.2` | `https://charts.jetstack.io`                          |

When a `Rancher` Composition is created, Krateo's composition controller renders and installs
this chart into the `cattle-system` namespace, bringing up cert-manager (a Rancher prerequisite)
and Rancher itself. Krateo **continuously reconciles** the release, so transient ordering issues
(e.g. Rancher's `Issuer` created before cert-manager's webhook is ready) are retried until the
install converges.

> cert-manager is bundled and enabled by default. Disable it (`cert-manager.enabled: false`) if
> it is already installed in the cluster, or if you terminate TLS externally
> (`rancher.ingress.tls.source: secret`).

## Prerequisites

- A running Kubernetes cluster with an Ingress controller (e.g. ingress-nginx).
- Krateo Composable Platform installed (`krateoSupportedVersion: 2.7.0`).
- A DNS record (or `/etc/hosts` entry) for the chosen `rancher.hostname` pointing at your ingress.

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
| `cert-manager.enabled`        | `true`               | Bundle and install cert-manager with Rancher.                           |

## How to install

### 1. Register the blueprint

```sh
kubectl create namespace cattle-system
kubectl apply -f compositiondefinition.yaml
```

This publishes a `Rancher` Composition type (`composition.krateo.io/v0-1-0`, plural `ranchers`).

> `compositiondefinition.yaml` points at the OCI artifact
> `oci://ghcr.io/braghettos/charts/rancher`. If the GHCR package is private, uncomment the
> `credentials` block and create the referenced pull-token secret.

### 2a. Create a Composition (the actual Rancher installer)

```yaml
apiVersion: composition.krateo.io/v0-1-0
kind: Rancher
metadata:
  name: rancher
  namespace: cattle-system
spec:
  rancher:
    hostname: rancher.my.org
    bootstrapPassword: "ChangeMe-please"
    replicas: 1
    ingress:
      tls:
        source: rancher
  cert-manager:
    enabled: true
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
# -> oci://ghcr.io/braghettos/charts/rancher:0.1.0
```

`.github/workflows/lint.yaml` runs `helm lint` + `helm template` on every pull request.

> The published GHCR package starts out **private**. Make it public (Package settings →
> Change visibility) for credential-free pulls, or keep it private and configure the
> `credentials` block in `compositiondefinition.yaml`.
