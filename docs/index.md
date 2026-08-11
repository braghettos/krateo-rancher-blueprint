---
type: Component
title: krateo-rancher-blueprint — index
description: The map of the krateo-rancher-blueprint doc bundle — a Krateo blueprint (forked Rancher Helm chart + CompositionDefinition) that turns every RancherInstaller Composition into a Rancher installer.
resource: oci://ghcr.io/krateo-blueprints/charts/rancher-installer
tags: [blueprint, rancher, compositiondefinition, helm-chart, cluster-management]
timestamp: 2026-08-11T00:00:00Z
---

# krateo-rancher-blueprint

A [Krateo](https://krateo.io) blueprint that installs [Rancher](https://www.rancher.com/)
on a Kubernetes cluster. The repo carries one Helm chart (`chart/`, name
`rancher-installer`) — a **fork of the upstream Rancher chart** (`2.14.2`) — and the
sibling `CompositionDefinition` (`compositiondefinition.yaml`) that registers it with
Krateo's `core-provider`. Once the `CompositionDefinition` is applied, **every
`RancherInstaller` Composition you create becomes a Rancher installer**.

The fork exists for two concrete reasons (see [overview](./overview.md)): `core-provider`
builds the Composition CRD from the chart's `values.schema.json` only, and the upstream
chart ships only a partial schema; and `templates/service.yaml` is patched so the
NodePort number can be forced from the Composition spec.

## The bundle (start here)

- [overview](./overview.md) — what the blueprint is, why it is a fork not a wrapper, the
  chart→CRD→Composition flow, and cert-manager as a hard prerequisite.
- [usage](./usage.md) — install core-provider + cert-manager, register the blueprint,
  create a `RancherInstaller`, reach the UI, the portal card path.
- [configuration](./configuration.md) — the whole Composition surface, field by field,
  from `chart/values.schema.json` plus the chart defaults it draws on.
- [api](./api.md) — the `RancherInstaller` CRD `core-provider` generates from the chart,
  the `CompositionDefinition` that registers it, and the Kind/apiVersion derivation.
- [examples](./examples.md) — the runnable examples under `examples/`.
- [release](./release.md) — how a tag ships the chart to GHCR and how to re-fork Rancher.
- [log](./log.md) — curated history.
- [llms.txt](./llms.txt) — the file-by-file index of this bundle.

## Layout

- `chart/` — the forked Rancher chart, name `rancher-installer`:
  `values.yaml` (upstream defaults), the curated `values.schema.json` that types the
  generated CRD, `templates/` (Rancher's Deployment, Service, Ingress, and the
  `ingate/` Gateway-API alternative), `scripts/` (the pre-upgrade and post-delete
  hooks), and `CHART-README.md` (the upstream Rancher chart README, kept as-is).
- `compositiondefinition.yaml` — the registration: pulls the chart from
  `oci://ghcr.io/krateo-blueprints/charts/rancher-installer` and publishes the
  `RancherInstaller` Composition type.
- `customform.yaml` — the Krateo Composable Portal card + CustomForm that drives the
  install from the UI.
- `quickstart.md` — an end-to-end kind walkthrough (NodePort pinned to a host port,
  no port-forwarder).

This is a single-chart repo, so this index is a `Component`, not a `ChartRepo`.
