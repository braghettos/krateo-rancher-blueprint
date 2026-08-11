---
type: Configuration
title: krateo-rancher-blueprint — configuration
description: The whole RancherInstaller Composition surface, field by field — from chart/values.schema.json (what a Composition may set) plus the chart defaults in values.yaml it draws on.
resource: oci://ghcr.io/krateo-blueprints/charts/rancher-installer
tags: [rancher, values, values-schema, composition, tls]
timestamp: 2026-08-11T00:00:00Z
---

# Configuration

A `RancherInstaller` Composition's `spec` is the chart's values, top-level. **Only fields
declared in `chart/values.schema.json` can be set on a Composition** — that schema is what
`core-provider` turns into the generated CRD; everything else uses the chart defaults in
`chart/values.yaml`. `hostname` is the single required field.

## Core

| Field | Type | Default | Description |
|---|---|---|---|
| `hostname` | string | `rancher.example.com` | **Required.** FQDN used to reach the Rancher UI/API; must resolve to your ingress (or a node IP for NodePort). Schema pattern-validated as a DNS name. |
| `bootstrapPassword` | string | _(random)_ | Initial password for the local `admin` user. If empty, Rancher generates one and stores it in the `bootstrap-secret` secret. |
| `replicas` | integer | `3` | Rancher server replica count (`1`–`9`). Use `3` for HA; set `1` for single-node clusters (kind/minikube). |
| `antiAffinity` | string | `preferred` | Pod anti-affinity for the replicas — `preferred` or `required`. |

## Ingress (`ingress.*`)

Used when `networkExposure.type` is `ingress` (the default).

| Field | Type | Default | Description |
|---|---|---|---|
| `ingress.enabled` | boolean | `true` | Create the `Ingress` resource. |
| `ingress.ingressClassName` | string | `""` | IngressClass to use; empty = the cluster default. |
| `ingress.servicePort` | integer | `80` | Backend service port the ingress targets — `80` or `443`. Must be `443` when `service.disableHTTP` is true. |
| `ingress.tls.source` | string | `rancher` | TLS certificate source: `rancher` (self-signed via cert-manager), `letsEncrypt` (public certs via cert-manager), or `secret` (bring your own TLS secret). |
| `ingress.tls.secretName` | string | `tls-rancher-ingress` | Name of the TLS secret when `source` is `secret`. |

## Service (`service.*`)

How the Rancher server `Service` is exposed. The NodePort forcing is the blueprint's own
patch to `templates/service.yaml`.

| Field | Type | Default | Description |
|---|---|---|---|
| `service.type` | string | `ClusterIP` | `ClusterIP` (reach Rancher through the Ingress), `NodePort`, or `LoadBalancer`. |
| `service.httpsNodePort` | integer | _(auto)_ | Force the https (443) node port number (`30000`–`32767`), only with `NodePort`/`LoadBalancer`. Useful on kind to line up with `extraPortMappings`. |
| `service.httpNodePort` | integer | _(auto)_ | Force the http (80) node port number (`30000`–`32767`), only with `NodePort`/`LoadBalancer`. |
| `service.disableHTTP` | boolean | `false` | Disable the http (80) port of the Rancher service. Requires `ingress.servicePort: 443`. |

## TLS sources and Let's Encrypt (`letsEncrypt.*`)

Only used when `ingress.tls.source: letsEncrypt`.

| Field | Type | Default | Description |
|---|---|---|---|
| `letsEncrypt.email` | string | — | Contact email used to register the Let's Encrypt account. **Required** for `letsEncrypt`. |
| `letsEncrypt.environment` | string | `production` | Use `staging` until your config is correct — `production` limits registrations. |
| `letsEncrypt.ingress.class` | string | — | Ingress class for the ACME solver ingress — `traefik`, `nginx`, or `""`. |
| `privateCA` | boolean | `false` | Set true when the certificate is signed by a private CA (provide the `tls-ca` secret in `cattle-system`). |
| `additionalTrustedCAs` | boolean | `false` | Add your own CA certs via a `tls-ca-additional` secret. |

## Network exposure (`networkExposure.*`)

| Field | Type | Default | Description |
|---|---|---|---|
| `networkExposure.type` | string | `ingress` | Exposure mode: `ingress` (classic Ingress), `gateway` (Gateway API — `templates/ingate/gateway.yaml`), or `none` (no ingress; reach Rancher via `service.type`). |

## Audit log (`auditLog.*`)

| Field | Type | Default | Description |
|---|---|---|---|
| `auditLog.enabled` | boolean | `false` | Enable the Rancher API audit logging system. |
| `auditLog.level` | integer | `0` | `0` disables, `3` is most verbose. |
| `auditLog.destination` | string | `sidecar` | `sidecar` (a `rancher-audit-log` sidecar) or `hostPath`. |

## Chart-only defaults (not on the Composition)

Some upstream values are not exposed in `values.schema.json`, so they cannot be set on a
Composition; they keep the chart defaults from `chart/values.yaml`. Notably: the Rancher
`image.*` (repository `rancher/rancher`, tag defaults to `.Chart.appVersion` = `v2.14.2`),
`priorityClassName: rancher-critical`, `systemDefaultRegistry` (for air-gapped mirrors),
`postDelete`/`preUpgrade` hook images (`rancher/shell:v0.7.0`), the `gateway.*`
Gateway-API block, `noProxy`, and `extraObjects`. To change one, either edit
`values.yaml` and re-release the chart, or add the field to `values.schema.json` so it
becomes a Composition knob.

The upstream Rancher chart's own README (all `--set` options, with links to the Rancher
docs) is preserved in [`chart/CHART-README.md`](../chart/CHART-README.md).
