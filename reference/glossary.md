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
scheduler.

**CrashLoopBackOff** — a Pod keeps crashing and Kubernetes keeps restarting it with growing delay.
Diagnose with `describe` + `logs`.

**ImagePullBackOff** — Kubernetes can't fetch the image (wrong name, missing, or not loaded into
the cluster). (Phase 9, Step 3)
