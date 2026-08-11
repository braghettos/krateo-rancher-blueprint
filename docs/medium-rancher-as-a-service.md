---
type: Architecture
title: Medium Rancher As A Service — krateo-rancher-blueprint
description: krateo-rancher-blueprint — medium rancher as a service reference.
resource: https://github.com/krateo-blueprints/krateo-rancher-blueprint
tags: [architecture, krateo-blueprints]
timestamp: 2026-08-11T00:00:00Z
---

# The schema is the contract: making Rancher self-service on Kubernetes

*What forking a Helm chart taught me about what "self-service" actually requires*

![The schema is the contract](https://raw.githubusercontent.com/krateo-blueprints/krateo-rancher-blueprint/main/docs/d1-schema-contract.png)

I wanted something simple: install Rancher from our platform as a single self-service action. Pick a hostname, click once, get a working Rancher. No Helm to learn, no values.yaml to copy, no RBAC to wire by hand.

I assumed it was plumbing. Point Krateo at Rancher's official Helm chart, generate a form, done. It turned into a small lesson about what makes a Helm chart self-service-ready — and most of them aren't.

## One design decision: the schema is the source of truth

When I designed Krateo, I made one call that still gets pushback: the source of truth for a composition is the chart's `values.schema.json`. Not `values.yaml`. The schema.

The reason is structural. In a declarative platform the contract a user fills in — the form, the generated CRD, the validation — has to come from a formal, machine-readable description. That description is the only thing an OPA policy, a CI check, or an AI agent can reason about. `values.yaml` is a bag of defaults; it tells you what *is* set, not what *may* be set or with what constraints.

So Krateo's `core-provider` reads exactly one file from the chart to build the composition's CRD: `values.schema.json`. It never looks at `values.yaml`.

That decision is the whole story of this post.

## Pointing Krateo at the official Rancher chart

I registered the upstream Rancher chart directly as a `CompositionDefinition` and watched it reconcile. It went green — CRD generated, controller up, `Ready=True`. Promising.

Then I tried to create a composition and set the hostname:

```
strict decoding error: unknown field "spec.hostname",
unknown field "spec.bootstrapPassword", unknown field "spec.replicas"
```

The generated CRD had exactly six fields: `agentTLSMode`, `auditLog`, `service`, `networkExposure`, `ingress`, `gateway`. No `hostname`. And you cannot install Rancher without a hostname.

![Partial vs complete schema](https://raw.githubusercontent.com/krateo-blueprints/krateo-rancher-blueprint/main/docs/d2-partial-vs-complete.png)

This isn't a bug in Krateo, and it isn't really a bug in Rancher either. Rancher's `values.schema.json` is a *partial* schema — it validates a handful of enums and one conditional. It was written to catch a few mistakes, not to describe the chart. The full surface of Rancher's configuration lives in `values.yaml`, which the platform deliberately doesn't read.

## Why the schema, not the values

It's worth understanding why a schema-driven platform can't just "fall back" to `values.yaml`.

Krateo's CRD generator (`crdgen`) is a transpiler: JSON Schema → Go structs → `controller-gen` → CRD. It honors the constructs that map cleanly to a Kubernetes *structural* schema — `properties`, `type`, `items`, `enum`, `minimum`/`maximum`, `pattern`, `default`, `required`, `additionalProperties`. It quietly drops everything that has no structural equivalent and is, in fact, illegal in a CRD: `if`/`then`, `allOf`/`anyOf`/`oneOf`, `const`, `format`, and type unions like `type: ["string", "null"]` (which it collapses to a single type plus `nullable: true`).

Rancher's schema happens to use exactly those constructs — an `if`/`then` for `disableHTTP`, `["string","null"]` unions on `service.type` and `agentTLSMode`. `crdgen` did the right thing with all of it. The fields simply weren't there to expose, because the schema never declared them.

A chart is only as self-service as its schema is honest.

## The fix: fork the chart and give it a real schema

A wrapper/umbrella chart doesn't help here — `core-provider` reads the *root* schema, so the wrapper would have to restate Rancher's surface anyway, and you still can't change Rancher's own templates. So I forked the Rancher chart into the blueprint and gave it a complete, crdgen-compatible `values.schema.json`: `hostname`, `bootstrapPassword`, `replicas`, `ingress.tls.source`, `service.type`, `letsEncrypt`, `privateCA`, and the rest — single types, string/integer enums, no `if`/`then`.

Now the composition spec is exactly what an operator wants to set:

```yaml
apiVersion: composition.krateo.io/v0-1-0
kind: RancherInstaller
metadata:
  name: rancher
  namespace: cattle-system
spec:
  hostname: rancher.my.org
  bootstrapPassword: "change-me"
  replicas: 1
  ingress:
    tls:
      source: rancher
  service:
    type: NodePort
    httpsNodePort: 30443
```

## Three things the fork had to get right

**The chart name is the API.** A Krateo composition's `Kind` is derived from the chart name. I named the fork `rancher-installer`, not `rancher`, on purpose. Rancher's resources are named from `rancher.fullname`, which collapses to the bare Helm release name when the release name contains the chart name. While enumerating the chart, Krateo's `chart-inspector` creates its own release-named scaffolding RBAC — and Rancher's cluster-admin `ClusterRoleBinding` collided with it:

```
ClusterRoleBinding "rancher-<hash>" ... exists and cannot be imported
into the current release: missing key "app.kubernetes.io/managed-by"
```

Naming the chart `rancher-installer` makes Rancher's resources `<release>-rancher-installer-*`, and the collision disappears. A name was the fix.

**cert-manager is a prerequisite, not a subchart.** Rancher renders an `Issuer` (`cert-manager.io/v1`). That CRD has to exist at render time, so cert-manager cannot live in the same Helm release — Helm fails with *"ensure CRDs are installed first."* This is exactly why Rancher's own docs install it as a separate step. The blueprint treats it the same way.

**Forcing the NodePort.** Rancher's `service.yaml` hardcodes its ports with no `nodePort` field, so Kubernetes auto-assigns one — useless if you want it to line up with a kind `extraPortMappings`. Since I'd already forked the chart, I patched the service template to honor `service.httpNodePort` / `service.httpsNodePort`. Now the port is deterministic and set from the composition spec.

## The payoff

![Rancher-as-a-Service flow](https://raw.githubusercontent.com/krateo-blueprints/krateo-rancher-blueprint/main/docs/d3-flow.png)

On a throwaway kind cluster — `core-provider`, cert-manager installed separately, and the blueprint pulled from a public OCI artifact — the loop is exactly what I wanted:

```sh
kubectl apply -f compositiondefinition.yaml   # pulls oci://ghcr.io/.../rancher-installer
kubectl apply -f rancher-composition.yaml     # the spec above
```

The composition reconciles to `Ready`, Rancher comes up `1/1 Running`, its Service is `NodePort` with the forced `443:30443`, and with kind mapping `30443` to the host the UI answers directly at `https://localhost:30443` — no port-forward.

![Rancher served via the kind NodePort](https://raw.githubusercontent.com/krateo-blueprints/krateo-rancher-blueprint/main/docs/rancher-nodeport-kind.png)

And the same approach generalizes: any tool, once it carries an honest schema, becomes a service anyone can request with a click — and that an agent can request programmatically against the same contract.

## The takeaway

Most Helm charts treat `values.schema.json` as an optional linting nicety. That's fine as long as the only reader is a human with the README open. The moment the schema becomes the product surface — the thing a platform, a policy engine, or an agent reads to know what's allowed — "optional" is the wrong word for it.

How many of the charts you run every day could describe themselves to something other than a person reading the README?

---

*The blueprint is open source: [github.com/krateo-blueprints/krateo-rancher-blueprint](https://github.com/krateo-blueprints/krateo-rancher-blueprint) — including a [quickstart](https://github.com/krateo-blueprints/krateo-rancher-blueprint/blob/main/quickstart.md) you can run on kind.*
