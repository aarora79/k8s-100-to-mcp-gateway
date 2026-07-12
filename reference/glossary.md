# Glossary — plain-English Kubernetes terms

Come back here whenever a word stops making sense. Ordered roughly from foundational to advanced.

**Container** — an application packaged with everything it needs to run, isolated from the host and
other containers. Lightweight and portable. (Phase 1)

**Image** — the read-only blueprint a container is created from. One image → many containers.

**Registry (container registry)** — where images are stored and pulled from (e.g. Docker Hub).
*Not* to be confused with the "Registry API" component of the mcp-gateway app.

**Kubernetes (K8s)** — a system that runs and manages containers across machines, keeping reality
matching a declared desired state. (Phase 2)

**Cluster** — the whole Kubernetes system: control plane + worker nodes.

**Node** — a machine (VM or physical) that runs your containers. Locally, usually one node.

**Node group / node pool** — a set of identical nodes managed as a unit (cloud clusters: "six
general-purpose nodes" + "two GPU nodes"). What the Cluster Autoscaler grows and shrinks.
(Phases 2, 10)

**Control plane** — the "brain" of the cluster: API server, scheduler, controllers, etcd.

**API server** — the front door of the cluster; everything (including `kubectl`) talks to it.

**etcd** — the cluster's database, storing all desired and current state.

**kubectl** — the command-line tool you use to talk to a cluster.

**Context** — which cluster (and user/namespace) `kubectl` is currently pointed at. Check with
`kubectl config current-context`. (Phase 3)

**Declarative** — you describe the desired end state (YAML) and Kubernetes makes it so, versus
**imperative**, where you issue step-by-step commands.

**Manifest** — a YAML (or JSON) file describing a Kubernetes object.

**Object / Resource** — a thing Kubernetes manages (Pod, Deployment, Service, …).

**Pod** — the smallest deployable unit; wraps one (or a few) containers sharing network/storage.

**Deployment** — declares a Pod template + replica count and manages rollouts/rollbacks. What you
create most often. (Phase 4)

**ReplicaSet** — the controller a Deployment uses to keep the right number of Pods. `Deployment →
ReplicaSet → Pods`.

**Replica** — one copy of a Pod. "3 replicas" = 3 identical Pods.

**Service** — a stable name + address that load-balances across a set of Pods. (Phase 4, 6)

**ClusterIP / NodePort / LoadBalancer** — Service types: internal-only / external via node port /
external via a load balancer. (Phase 6)

**Service discovery / DNS** — reaching a Service by its name (`mongodb`) instead of an IP.

**Label** — a key/value tag on objects (`app: web`). **Selector** — a query that matches labels,
e.g. how a Service finds its Pods.

**Namespace** — a virtual partition of a cluster for organizing objects (`default`, `kube-system`,
`mcp-gateway`).

**ConfigMap** — non-secret configuration injected into Pods as env vars or files. (Phase 5)

**Secret** — like a ConfigMap but for sensitive values; base64-encoded, *not* encrypted by default.

**Volume** — storage attached to a Pod. **PersistentVolume (PV)** — actual cluster storage.
**PersistentVolumeClaim (PVC)** — a Pod's request for storage. **StorageClass** — how storage is
provisioned on demand. (Phase 5)

**StatefulSet** — like a Deployment but for stateful apps needing stable identity/storage (e.g.
databases).

**Ingress** — an HTTP router mapping hostnames/paths to Services. Needs an **Ingress controller**
(an nginx Pod) to actually do the routing. (Phase 6)

**port-forward** — a debugging tunnel from a local port to a Pod/Service. Not for production
traffic. (Phase 4)

**Probe** — a health check. **readinessProbe** = "ready for traffic?"; **livenessProbe** = "still
healthy or restart me?". (Phase 9)

**Rolling update** — replacing Pods gradually to deploy a new version with no downtime.
**Rollback** — reverting to a previous revision.

**Self-healing** — Kubernetes recreating Pods that die, to match desired replica count.

**Helm** — the package manager for Kubernetes. A **chart** is a packaged app; a **release** is an
installed instance; **values.yaml** holds its configuration. (Phase 7)

**kind** — "Kubernetes IN Docker"; runs cluster nodes as containers. Great for realistic local
clusters. (Phase 8)

**Docker Compose** — a tool to run multi-container apps on one machine from a `docker-compose.yml`.
Simpler than Kubernetes; how `mcp-gateway-registry` ships. (Phase 9)

**Resource requests / limits** — how much CPU/RAM a container asks for / is capped at; guides the
scheduler. (Phase 10)

**OOMKilled** — a container exceeded its memory limit and was killed. Check `describe pod` for
the reason and restart count. (Phases 10–11)

**metrics-server** — the add-on that collects live CPU/RAM usage; powers `kubectl top` and the
HPA. (Phase 10)

**HPA (Horizontal Pod Autoscaler)** — a controller that automatically adjusts a Deployment's
replica count to hold a metric (usually CPU % of *requests*) at a target, within a min/max range.
(Phase 10)

**Cluster Autoscaler / Karpenter** — controllers that add or remove *nodes* when Pods can't be
scheduled. Karpenter is the modern one: it provisions right-sized machines from the Pending Pods'
actual requirements. Cloud-only. (Phase 10)

**EKS Auto Mode** — AWS-managed EKS compute: AWS runs Karpenter and manages the nodes for you
(GKE's equivalent is Autopilot). (Phase 10)

**nodeSelector / node affinity** — ways a Pod *chooses* nodes by label. `nodeSelector` is
all-or-nothing; affinity adds `required` (hard) vs `preferred` (soft) rules. (Phase 10)

**Taint / toleration** — the inverse: a *node repels* all Pods except those that explicitly
tolerate the taint. How special nodes (control plane, GPU) are reserved. (Phase 10)

**Pod affinity / anti-affinity** — scheduling rules relative to other *Pods*; anti-affinity
spreads replicas across nodes so one machine can't take them all down. (Phase 10)

**Prometheus** — the standard Kubernetes metrics system: scrapes `/metrics` endpoints on a
schedule and stores the time series. (Phase 11)

**Grafana** — dashboards over Prometheus (and other sources). Installed together with Prometheus
via the `kube-prometheus-stack` Helm chart. (Phase 11)

**vLLM / Ollama** — open-source LLM serving engines. vLLM is the production standard on NVIDIA
GPUs (and exposes native Prometheus metrics); Ollama is the pragmatic multi-arch choice for
local/CPU serving. Both speak the OpenAI-compatible API. (Phase 11)

**CrashLoopBackOff** — a Pod keeps crashing and Kubernetes keeps restarting it with growing delay.
Diagnose with `describe` + `logs`.

**ImagePullBackOff** — Kubernetes can't fetch the image (wrong name, missing, or not loaded into
the cluster). (Phase 9, Step 3)

---

## Names you'll hear in the wild

Kubernetes itself is open source; what you actually *run* is always some packaging of it. These
names come up constantly — none change what you've learned, since they all speak the same API
and `kubectl`:

**Distribution (distro)** — a packaged, opinionated build of Kubernetes, exactly like Linux
distros. "Vanilla" or "upstream" means the unmodified open-source project.

**k3s** — a popular *lightweight* distribution (SUSE/Rancher): the whole cluster in one small
binary, with heavyweight internals swapped for lighter ones. The default choice for edge
devices, Raspberry Pis, and homelabs. **k0s** and **microk8s** are similar ideas from other
vendors.

**k9s** — confusingly named but totally different: a terminal *UI for kubectl* — browse pods,
logs, and events interactively. A tool you run, not a cluster. (Genuinely useful; try it.)

**kubeadm** — the official bootstrapper for building a cluster on machines you own. It's what
`kind` uses internally on its fake nodes.

**EKS / GKE / AKS** — *managed* Kubernetes from AWS / Google / Azure: the cloud runs the control
plane (and with EKS Auto Mode / GKE Autopilot, the nodes too); you bring workloads. This is how
most companies run Kubernetes.

**OpenShift** — Red Hat's enterprise distribution: Kubernetes plus built-in CI/CD, registry,
stricter security defaults, and commercial support. Common in large/regulated companies.

**Rancher** — a management layer for running *many* clusters from one UI (same company as k3s).

**Talos** — a minimal, immutable Linux OS that exists only to run Kubernetes — no SSH, no shell;
you manage the OS itself through an API. Where serious self-hosted clusters are heading.

**kustomize** — Helm's template-free rival for customizing YAML: overlay patches on top of base
manifests instead of filling in `{{ placeholders }}`. Built into kubectl (`kubectl apply -k`).
Teams commonly use both: Helm for third-party apps, kustomize for their own.

**CNCF** — the Cloud Native Computing Foundation, the vendor-neutral home of Kubernetes,
Prometheus, Helm, and hundreds more. "A CNCF graduated project" is a maturity signal you can
roughly trust.

**OCI** — the Open Container Initiative: the standards for image formats and registries. Why an
image built by Docker runs under containerd, and why Helm charts can be stored in the same
registries as images ("OCI registry").
