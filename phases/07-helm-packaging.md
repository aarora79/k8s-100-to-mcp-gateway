# Phase 7 — Helm: installing and templating apps

**Goal:** use Helm to install prebuilt apps in one command, and understand templating so you can
package your own. In the capstone you'll hand-write your manifests (that's the point of the
learning), but Helm is how the wider ecosystem ships Kubernetes software — you need to be able to
consume it.

**Time:** ~1 hour.

---

## Before you start — 60 seconds of retrieval

From memory:

1. Roughly how many YAML files did the Phase 6 exercise need for *one* app (deployment, service,
   ingress…)? Now imagine an app with six components. That pain is why Helm exists.
2. How do you list pods in a namespace other than `default`? *(Phase 2's namespace section)*
3. What's the full DNS name of a Service named `db` in namespace `data`? *(Phase 6)*

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

## Use case 1: install a real app with one command

We'll install **podinfo** — a small demo web app maintained specifically for teaching Kubernetes
tooling (you'll meet it again in Phase 8 as our API tier):

```bash
helm repo add podinfo https://stefanprodan.github.io/podinfo
helm repo update
helm search repo podinfo                   # see it exists

# Install into a dedicated namespace, with a setting override:
kubectl create namespace demo
helm install my-podinfo podinfo/podinfo \
  --namespace demo \
  --set replicaCount=2
# helm install <RELEASE-NAME> <repo/chart>
#              └─ a name YOU choose for this installed copy ("release");
#                 the second arg is which chart, as <repo-you-added>/<chart-name>

kubectl get pods -n demo --watch           # wait for 2 podinfo pods to be Running
```

One command created a Deployment (2 replicas — your `--set` took effect) and a Service. Inspect
what it made:

```bash
helm list -n demo                          # your release
kubectl get all -n demo                    # everything the chart created
kubectl port-forward -n demo svc/my-podinfo 9898:9898
# open http://localhost:9898 — podinfo serves a little UI so you can see it working
```

From another Pod you'd reach it at `my-podinfo.demo.svc.cluster.local` (or `my-podinfo` from
within the `demo` namespace) — exactly the discovery you learned in Phase 6.

> **A cautionary tale about the chart ecosystem.** For years, every Kubernetes tutorial's Helm
> example was "install MongoDB from the Bitnami charts" — until 2025, when Bitnami withdrew its
> free versioned images and those installs started failing with `ImagePullBackOff` (a status you
> can now diagnose!). The lesson generalizes: **charts and registries drift.** Before installing
> any chart, check it's maintained, run `helm show values <chart> | less` to see its *current*
> options rather than trusting a blog post, and when an install fails, `kubectl describe pod` in
> the release's namespace tells you whether it's an image problem or a config problem. This is
> also why the capstone has you *write* the MongoDB manifests yourself with the official `mongo`
> image — skills over dependencies.

Uninstall when done experimenting:

```bash
helm uninstall my-podinfo -n demo
```

---

## Use case 2: understand templating (so you can package your own)

A chart is a directory. Scaffold one to see its anatomy — inside this repo, so it becomes part
of your learning artifact (per the ground rules: commit as you go):

```bash
cd <this-repo>            # the k8s learning repo root
mkdir -p helm-lab && cd helm-lab
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

## Challenge — no training wheels ✋

Using `helm-lab/demo-chart` and no copy-paste from above:

1. Render the chart with 5 replicas and image tag `1.27` using only `--set` flags, and confirm
   both values landed in the output (`helm template ... | grep ...`).
2. Now achieve the same via a custom values file (`my-values.yaml` + `-f`) instead of `--set`.
3. Install it as release `challenge` into namespace `demo`, verify with `helm list` and
   `kubectl get all -n demo`, then uninstall and delete the namespace.

---

## Checkpoint ✅

- [ ] Install, inspect, and uninstall an app (podinfo) via a Helm chart.
- [ ] Explain chart vs release, and what `values.yaml` is for.
- [ ] Render a chart's templates without installing (`helm template`).
- [ ] Complete the challenge using both `--set` and a values file.

---

## Ask Claude Code

**Teach-back:**

> I just learned Helm basics. Quiz me one question at a time on: chart vs release vs repo, where
> values come from (`values.yaml` vs `-f` vs `--set`, and which wins), and what `helm template`
> does vs `helm install`. Correct me where I'm fuzzy.

**Go deeper:**

> I installed `podinfo/podinfo` with Helm into the `demo` namespace. Show me how to upgrade the
> release to change a value (`helm upgrade`), see its history, and roll back (`helm rollback`) —
> and explain how that compares to `kubectl rollout undo` from Phase 4.

**Next:** [Phase 8 — A realistic multi-service app](08-multi-service-app.md)
