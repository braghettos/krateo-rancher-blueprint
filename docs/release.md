---
type: Runbook
title: krateo-rancher-blueprint — release
description: How a release ships — one semver tag packages the forked chart and pushes it to GHCR as an OCI artifact — plus the re-fork runbook for tracking a new upstream Rancher version.
resource: oci://ghcr.io/krateo-blueprints/charts/rancher-installer
tags: [release, oci, ghcr, re-fork, rancher]
timestamp: 2026-08-11T00:00:00Z
---

# Release

One plain-semver tag (`X.Y.Z`, **no** `v` prefix) releases the chart. The tag push
triggers `.github/workflows/release-tag.yaml`.

## What a tag ships

`release-tag.yaml` (triggered on tags matching `[0-9]+.[0-9]+.[0-9]+`):

1. Stamps `chart/Chart.yaml` (`CHART_VERSION` → the tag). The stamped chart version also
   drives the generated Composition apiVersion, so `0.2.0` → `composition.krateo.io/v0-2-0`.
2. Packages `chart/` and pushes it to
   `oci://ghcr.io/<owner>/charts/rancher-installer:X.Y.Z`. The OCI namespace is derived
   from the repository owner (`GITHUB_TOKEN` can only write its own namespace, so a
   hardcoded org would 403 the moment the repo moved).

```console
$ git tag 0.1.0 && git push origin 0.1.0
# → oci://ghcr.io/krateo-blueprints/charts/rancher-installer:0.1.0
```

Then verify the artifact exists and (if the package is meant to be public) is public:

```console
$ helm show chart oci://ghcr.io/krateo-blueprints/charts/rancher-installer --version 0.1.0 | head -3
```

After the chart is published, bump `compositiondefinition.yaml`'s `spec.chart.version` to
the new tag on `main` — it is this blueprint's own registration and must point at a
version that exists.

## PR-time checks

`.github/workflows/lint.yaml` runs on every PR:

- stamps a temporary chart version, then `helm lint ./chart` and `helm template` (default
  values) as a render smoke test;
- the shared docs-standard linter (`lint-docs`), which enforces the Krateo Documentation
  Standard on this bundle.

`.github/workflows/security.yml` runs the org security scan.

## Re-fork runbook — tracking a new upstream Rancher

Because the chart is a **fork** of the upstream Rancher chart, not a wrapper
([overview](./overview.md)), updating Rancher is a re-fork, not a dependency bump:

1. Pull the target upstream Rancher chart version (e.g. from
   `https://releases.rancher.com/server-charts/stable`).
2. Replace `chart/templates/`, `chart/scripts/`, and `chart/values.yaml` with the new
   upstream contents.
3. **Re-apply the NodePort patch** to `chart/templates/service.yaml` — the block guarded
   by the `Krateo blueprint patch` comment (`service.httpNodePort` /
   `service.httpsNodePort`).
4. **Reconcile `chart/values.schema.json`**: keep it complete and crdgen-suitable (no
   `if/then`, no type-unions, scalar defaults) so `core-provider` can still generate the
   `RancherInstaller` CRD; add any new install knob you want exposed on a Composition.
5. Update `chart/Chart.yaml`'s `appVersion` to the new Rancher version, keep `name:
   rancher-installer`, and bump the chart `version` on the next tag.
6. Run `helm lint` / `helm template`, then the kind end-to-end
   ([examples/kind-nodeport](../examples/kind-nodeport/README.md)) before tagging.

## Version pinning downstream

`compositiondefinition.yaml` pins `spec.chart.version` to an explicit version; nothing
tracks a mutable tag. Consumers register the blueprint against that pinned version.
