# Phase 7 — Helm: installing and templating apps

**Goal:** use Helm to install prebuilt apps in one command, and understand templating so you can
package your own. This is how you'll install MongoDB for the capstone without hand-writing every
manifest.

**Time:** ~1 hour.

---

## What Helm is

By now you've written a lot of YAML. Real apps need *many* manifests (Deployment, Service,
ConfigMap, Secret, Ingress, PVC…), often with values repeated across them. **Helm is the package
manager for Kubernetes.** A packaged app is a **chart**; installing a chart with your chosen
settings creates a **release**.

Two things Helm gives you:

1. **Install other people's apps in one command** (e.g. a production-grade MongoDB).
2. **Template your own manifests** so one chart can be configured per environment via a
   `values.yaml`.

---

## Install Helm

```bash
brew install helm
helm version
```

---

## Use case 1: install MongoDB with one command

You'll need MongoDB for the capstone. Rather than write a StatefulSet + Service + PVC by hand, use
a community chart:

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
helm search repo bitnami/mongodb           # see it exists

# Install into a dedicated namespace, with light settings for local use:
kubectl create namespace data
helm install mongo bitnami/mongodb \
  --namespace data \
  --set auth.rootPassword=devpassword \
  --set persistence.size=2Gi

kubectl get pods -n data --watch           # wait for the mongodb pod to be Running
```

Helm just created a Deployment/StatefulSet, a Service, a Secret, and a PVC for you. Inspect what
it made:

```bash
helm list -n data                          # your release
kubectl get all -n data                    # everything the chart created
```

Reach it from another Pod by its Service name (`mongo-mongodb.data.svc.cluster.local`, or
`mongo-mongodb` from within the `data` namespace) — exactly the discovery you learned in Phase 6.

> **Chart specifics change over time.** If the `--set` keys above differ from what the current
> chart expects, run `helm show values bitnami/mongodb | less` to see the real options, or ask
> Claude Code for the current values for the version you pulled.

Uninstall when done experimenting:

```bash
helm uninstall mongo -n data
```

---

## Use case 2: understand templating (so you can package your own)

A chart is a directory. Scaffold one to see its anatomy:

```bash
cd ~   # or wherever you keep scratch work
helm create demo-chart
ls demo-chart
#   Chart.yaml        <- chart metadata (name, version)
#   values.yaml       <- default configuration values
#   templates/        <- YAML with {{ placeholders }}
#   templates/deployment.yaml, service.yaml, ...
```

Open `templates/deployment.yaml` and notice things like:

```yaml
spec:
  replicas: {{ .Values.replicaCount }}
  ...
  image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
```

Those `{{ .Values.x }}` placeholders are filled from `values.yaml` (or `--set` on the command
line). Render the templates *without* installing to see the final YAML:

```bash
helm template demo-chart | head -60         # shows the fully-rendered manifests
helm template demo-chart --set replicaCount=5 | grep replicas
```

That's the whole idea: **one templated chart, many configurations.** For the capstone you *could*
eventually wrap your mcp-gateway manifests in a chart, but Phase 9 uses plain manifests first to
keep the learning focused — templating is a nice later refactor.

---

## Checkpoint ✅

- [ ] Install and inspect an app (MongoDB) via a Helm chart.
- [ ] Explain chart vs release, and what `values.yaml` is for.
- [ ] Render a chart's templates without installing (`helm template`).

---

## Ask Claude Code

> I installed `bitnami/mongodb` with Helm into the `data` namespace. Show me how to connect to it
> from a temporary `mongosh` pod using the Service name and the root password I set, so I can
> confirm it works.

**Next:** [Phase 8 — A realistic multi-service app](08-multi-service-app.md)
