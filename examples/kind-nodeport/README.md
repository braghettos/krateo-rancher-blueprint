---
type: Example
title: kind-nodeport — install Rancher via a RancherInstaller on kind
description: An end-to-end example — register the CompositionDefinition, then create a single-replica RancherInstaller that exposes Rancher on a forced NodePort so the UI opens at https://localhost:30443 on a kind cluster with no port-forwarder.
resource: oci://ghcr.io/krateo-blueprints/charts/rancher-installer
tags: [example, rancher, composition, kind, nodeport]
timestamp: 2026-08-11T00:00:00Z
---

# kind-nodeport

Install Rancher through the blueprint on a local [kind](https://kind.sigs.k8s.io/)
cluster and reach the UI in your browser, exposing Rancher with a **NodePort whose number
is forced from the Composition spec** (`service.type: NodePort`,
`service.httpsNodePort: 30443`) so it lines up with kind's port mapping — no
port-forwarder needed. This is the blueprint's own NodePort patch
([overview](../../docs/overview.md)) in action.

Verified with: kind `v0.24`, Helm `v3.19`, Krateo `core-provider 1.0.0`, cert-manager
`v1.20.2`, Rancher `2.14.2`. Result: the Composition reaches `Ready=True`, Rancher's
`Service` is `NodePort` `443:30443`, the pod is `1/1 Running`, and the "Welcome to
Rancher" page loads at `https://localhost:30443` (accept the self-signed certificate
warning).

## 1. A kind cluster that maps the NodePort to your host

```console
$ cat > kind-rancher.yaml <<'EOF'
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
    extraPortMappings:
      - containerPort: 30443   # must match service.httpsNodePort below
        hostPort: 30443        # https://localhost:30443 on your machine
        listenAddress: "127.0.0.1"
        protocol: TCP
EOF

$ kind create cluster --name rancher --config kind-rancher.yaml
```

## 2. Prerequisites — core-provider and cert-manager

```console
$ helm repo add krateo https://charts.krateo.io
$ helm repo add jetstack https://charts.jetstack.io
$ helm repo update

$ helm upgrade --install core-provider krateo/core-provider \
    --version 1.0.0 -n krateo-system --create-namespace

$ helm upgrade --install cert-manager jetstack/cert-manager --version v1.20.2 \
    -n cert-manager --create-namespace --set crds.enabled=true
$ kubectl rollout status deploy/cert-manager-webhook -n cert-manager
```

cert-manager must be installed separately, beforehand: Rancher's `Issuer` CRD has to
exist at render time ([overview](../../docs/overview.md)).

## 3. Register the blueprint

The `CompositionDefinition` pulls the chart from the public GHCR OCI artifact:

```console
$ kubectl create namespace cattle-system
$ kubectl apply -f - <<'EOF'
apiVersion: core.krateo.io/v1alpha1
kind: CompositionDefinition
metadata:
  name: rancher
  namespace: cattle-system
spec:
  chart:
    url: oci://ghcr.io/krateo-blueprints/charts/rancher-installer
    version: "0.1.0"
EOF

$ kubectl wait compositiondefinition/rancher -n cattle-system \
    --for=condition=Ready --timeout=180s
```

This publishes the `RancherInstaller` type (`composition.krateo.io/v0-1-0`, plural
`rancherinstallers`).

## 4. Create the Composition — NodePort pinned to 30443

```console
$ kubectl apply -f - <<'EOF'
apiVersion: composition.krateo.io/v0-1-0
kind: RancherInstaller
metadata:
  name: rancher
  namespace: cattle-system
spec:
  hostname: rancher.kind.local
  bootstrapPassword: "admin-kind-test"   # change me
  replicas: 1
  ingress:
    tls:
      source: rancher
  service:
    type: NodePort
    httpsNodePort: 30443                  # forced node port == kind extraPortMapping
EOF

$ kubectl wait rancherinstaller/rancher -n cattle-system \
    --for=condition=Ready --timeout=600s
```

Confirm the Service got the forced node port:

```console
$ kubectl get svc -n cattle-system -o wide | grep rancher-installer
# ... NodePort ... 80:31233/TCP,443:30443/TCP ...
```

## 5. Open Rancher

```console
$ open https://localhost:30443       # macOS; xdg-open on Linux
```

Flow: `https://localhost:30443` → kind port mapping → node:30443 → Rancher's NodePort →
Rancher. Accept the self-signed certificate warning. On the "Welcome to Rancher" page
enter the `bootstrapPassword`, or read it back:

```console
$ kubectl get secret -n cattle-system bootstrap-secret \
    -o go-template='{{ "{{" }} .data.bootstrapPassword|base64decode {{ "}}" }}{{ "{{" }} "\n" {{ "}}" }}'
```

## 6. Clean up

```console
$ kind delete cluster --name rancher
```
