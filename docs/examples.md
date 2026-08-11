---
type: ExampleIndex
title: krateo-rancher-blueprint — examples
description: Index of the runnable examples under examples/ — installing Rancher through a RancherInstaller Composition.
resource: oci://ghcr.io/krateo-blueprints/charts/rancher-installer
tags: [examples, rancher, composition, kind, nodeport]
timestamp: 2026-08-11T00:00:00Z
---

# Examples

- [examples/kind-nodeport](../examples/kind-nodeport/README.md) — an end-to-end install
  on a local kind cluster: register the `CompositionDefinition`, then create a
  single-replica `RancherInstaller` that exposes Rancher on a **forced NodePort**
  (`service.type: NodePort`, `service.httpsNodePort: 30443`) lined up with kind's
  `extraPortMappings`, so the Rancher UI opens at `https://localhost:30443` with no
  port-forwarder. It is the concrete form of the flow in [usage](./usage.md), and the
  same scenario the top-level [`quickstart.md`](../quickstart.md) walks through.
