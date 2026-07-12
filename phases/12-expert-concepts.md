# Phase 12 — The expert concept map: 300–400 level, explained at 200 level

Everything in the phases taught you to *use* Kubernetes. This file is the map of what experts
know *about* it — the concepts that come up in design discussions, incident reviews, and
architecture docs. **Theory only, on purpose**: each entry tells you what the thing is, why it
exists, and the mental model to keep — so that when you meet it in the wild, it's a familiar
face, not a wall. Ask Claude Code for a hands-on walkthrough of any of them when you're ready.

---

## 1. How Kubernetes actually thinks

### Everything is a reconciliation loop

You met "declare desired state, the system converges" in Phase 2. The expert version: there is
no central brain doing this. Kubernetes is a *swarm* of independent **controllers**, each
running the same dumb loop forever: *read desired state → read actual state → make one small
change toward desired → repeat.* The Deployment controller only manages ReplicaSets. The
ReplicaSet controller only counts Pods. The HPA only adjusts a number. None of them know about
each other; they cooperate purely by reading and writing the same API objects. Complex behavior
emerges from simple loops sharing a database — that's the whole architecture, and it's why
Kubernetes is so resilient *and* why cause-and-effect can be hard to trace.

### Level-triggered, not edge-triggered

Controllers don't react to *events* ("a pod died!"); they repeatedly compare *states* ("desired
3, actual 2 — fix it"). The difference matters: an event can be missed (network blip, controller
restart), but a state comparison self-corrects on the next loop no matter what was missed. This
is why Kubernetes recovers from chaos so well — and why fixes sometimes take a few seconds:
you're waiting for somebody's next loop iteration, not triggering an immediate action.

### Spec vs status

Every object has two halves: `spec` (what you want — you write it) and `status` (what currently
is — controllers write it). `kubectl describe` is largely a pretty-printer for status. A
surprising amount of debugging reduces to: *whose job was it to make status match spec, and why
hasn't it?*

### The API is the product

`kubectl` is a thin client. Every object is a REST resource with a **group, version, and kind**
(that's what `apiVersion: apps/v1, kind: Deployment` means), stored in etcd, guarded by the API
server. Everything — the scheduler, controllers, kubelet, your CI system, the dashboard — talks
to the same API with the same rules. There is no back door. This uniformity is why the ecosystem
could build so much on top: anything that can read/write API objects can extend Kubernetes.

---

## 2. Extending Kubernetes: CRDs, operators, webhooks

### Custom Resource Definitions (CRDs)

You can teach the API server brand-new object kinds. Install a CRD and suddenly
`kind: Certificate` or `kind: PostgresCluster` is as real as `kind: Deployment` — storable,
watchable, RBAC-guardable. CRDs are just *data*, though; they do nothing alone.

### Operators

An **operator** = CRDs + a custom controller that reconciles them. It packages *operational
knowledge* as software: a Postgres operator doesn't just run Postgres Pods, it knows how to do
failover, backups, version upgrades — the things a human DBA would do — as reconciliation loops.
This is the standard answer to "how do I run something stateful and complicated on Kubernetes
without becoming its full-time operator myself?" (cert-manager, Prometheus Operator — which
installed your Phase 11 monitoring — and most database-on-K8s solutions are operators.)

### Admission webhooks

Between "API server receives your YAML" and "it's saved to etcd" sits an interception point.
**Mutating** webhooks can rewrite the object in flight (this is how service meshes inject their
sidecar containers without you writing them); **validating** webhooks can reject it ("no
containers without resource limits in this cluster"). Policy engines like **Kyverno** and **OPA
Gatekeeper** are essentially rule languages compiled into these webhooks. When an apply fails
with an error naming a webhook you've never heard of — now you know what's happening.

---

## 3. Who may do what: the request gauntlet

Every API request passes three gates, in order:

1. **Authentication** — *who are you?* Certificates, tokens, cloud IAM. Humans get identities
   from outside; Pods get **ServiceAccounts** (every Pod runs as one, even yours — `default`,
   unless you said otherwise).
2. **Authorization (RBAC)** — *may you do this?* **Roles** are bundles of verbs-on-resources
   ("get/list pods", "create deployments"); **RoleBindings** attach them to identities.
   Namespaced (Role) or cluster-wide (ClusterRole). The design is deny-by-default and additive
   only — there is no "deny" rule, which keeps reasoning simple.
3. **Admission** — *is this allowed here?* The webhooks and built-in checks from section 2,
   plus things like ResourceQuota enforcement.

The expert habit: when something mysteriously "can't", ask which of the three gates said no.
`kubectl auth can-i` answers gate 2 directly.

---

## 4. Networking under the hood

### The model: every Pod gets a real IP

Kubernetes' networking contract is radical in its simplicity: every Pod has its own IP, every
Pod can reach every other Pod's IP directly, no NAT between them. All the apparent magic
(Services, DNS) is built *on top* of this flat network, not instead of it.

### CNI — who builds that network

Kubernetes doesn't implement the Pod network itself; it delegates to a **CNI plugin** (Cilium,
Calico, flannel, the cloud provider's own). That's why "Kubernetes networking" differs per
cluster, and why the CNI choice is one of the few genuinely hard-to-change decisions.

### What a Service *really* is

A ClusterIP is a **virtual** IP — no interface anywhere owns it. `kube-proxy` programs every
node's packet-routing rules (iptables or IPVS) so that traffic *to* that IP gets rewritten to a
random ready Pod IP. The load balancing you've relied on since Phase 4 is a packet-rewriting
lottery on each node. (Modern CNIs like Cilium replace kube-proxy with **eBPF** — programs
running inside the kernel — for the same contract, faster.) Corollary you've already debugged:
a Service is only as real as its **endpoints** list.

### Headless Services

Set `clusterIP: None` and the Service stops being a lottery: DNS returns *every Pod's individual
IP*, and each Pod gets its own stable DNS name. This is how StatefulSet members find *specific*
peers ("I need to talk to the primary, not a random replica") — databases and consensus systems
depend on it.

### NetworkPolicy

By default, the flat network means *any* Pod can talk to *any* Pod — including your database
from a compromised frontend. **NetworkPolicies** are namespace-scoped firewall rules selecting
Pods by label ("only Pods labeled `app: api` may reach `app: db` on 5432"). Two catches experts
know: they're enforced by the CNI (a CNI that doesn't support them silently ignores them), and
they're allow-list semantics — the moment any policy selects a Pod, everything not explicitly
allowed is denied.

### Gateway API — Ingress's successor

Ingress (Phase 6) turned out too limited: one flat object mixing infrastructure concerns
(which load balancer?) with routing concerns (which path to which service?), extended through
vendor-specific annotations. **Gateway API** is the official replacement: `GatewayClass` /
`Gateway` / `HTTPRoute` split those roles so platform teams own the entry points and app teams
own their routes. New serious work targets it; read Ingress as "stable and everywhere," Gateway
API as "the direction."

### Service mesh

When you need mTLS encryption between all services, retries/timeouts/circuit-breaking without
touching app code, and per-request routing (send 1% of traffic to v2), a **mesh** (Istio,
Linkerd) provides it — classically by injecting a proxy sidecar into every Pod (via a mutating
webhook — section 2 pays off), newer designs move the proxy to the node ("ambient"). The expert
opinion to hold: meshes are powerful and *heavy*; most clusters don't need one, and NetworkPolicy
+ good probes get you far.

---

## 5. Scheduling and resource management, the deeper layer

### QoS classes — who gets evicted first

The scheduler files every Pod into a quality-of-service class based on its requests/limits:
**Guaranteed** (requests = limits for everything), **Burstable** (requests < limits), or
**BestEffort** (neither set). When a node runs out of memory, evictions go in reverse order:
BestEffort dies first, Guaranteed last. This is the *real* reason Phase 10 said "always set
requests" — you're not just informing the scheduler, you're choosing your Pod's place in the
lifeboat queue.

### PriorityClasses and preemption

Pods can carry a priority. When a high-priority Pod can't schedule, the scheduler may **evict
lower-priority Pods to make room** — that's preemption. It's how clusters guarantee that
critical system components always fit, and an occasional answer to "why did my pod just die
with no error in its logs?"

### PodDisruptionBudgets (PDBs)

Nodes get drained for upgrades (see section 8). A PDB declares "never take my app below N ready
replicas *voluntarily*" — drains and rollouts respect it; crashes obviously don't. It's the
contract between app teams ("stay available") and platform teams ("let me patch nodes").

### ResourceQuota and LimitRange

Multi-team clusters cap each namespace: **ResourceQuota** limits totals ("this namespace gets
20 CPUs, 40Gi, 10 PVCs"), **LimitRange** sets per-Pod defaults and bounds (including injecting
default requests into Pods that didn't set any). If your Pod is rejected at *creation* time with
a quota error, this is who said no (admission — gate 3).

### Init containers and sidecars

A Pod's `initContainers` run to completion, in order, *before* the app starts — the standard
home for "wait for the database", "run migrations", "download the model". **Sidecars** run
*alongside* the app for its whole life (log shippers, proxies); since Kubernetes 1.29 they're
formally init containers with `restartPolicy: Always`, which fixed years of shutdown-ordering
pain. When you see a Pod with three containers and only wrote one — webhook-injected sidecars,
section 2.

---

## 6. The rest of the workload zoo

- **DaemonSet** — "exactly one Pod on every node" (log collectors, node monitors, CNI agents).
  Your Phase 11 monitoring stack quietly installed one (`node-exporter`).
- **Job** — run Pods until N successful completions, with retries. Batch work.
- **CronJob** — Jobs on a schedule. The two footguns experts check: `concurrencyPolicy` (what if
  the previous run is still going?) and time zone.
- **StatefulSet, properly** — you know it's "Deployment for stateful apps"; the mechanism is:
  Pods get stable ordinal names (`db-0`, `db-1`), each ordinal keeps *its own* PVC forever, and
  a headless Service gives each a stable DNS name. Scale-up and rollout proceed in ordinal
  order. That's the toolkit consensus systems need — and *why* replicas of a Deployment can't
  safely run a database.
- **KEDA** — event-driven autoscaling: scale on queue depth, Kafka lag, or any external metric,
  including **scale to zero** (the HPA can't go below 1). The standard answer for worker/queue
  workloads.

---

## 7. Storage, the deeper layer

- **CSI (Container Storage Interface)** — like CNI for disks: storage vendors ship a driver,
  Kubernetes speaks a standard protocol to it. StorageClasses (Phase 5) are named recipes
  pointing at a CSI driver with parameters.
- **Access modes are about *nodes*, not Pods** — `ReadWriteOnce` means one *node* can mount the
  volume, which is why two Pods on different nodes can't share it. `ReadWriteMany` needs a
  filesystem designed for that (NFS-style) and is rarer and slower than people hope. This
  single misunderstanding explains a large fraction of real-world storage confusion.
- **VolumeSnapshots** — CSI-level point-in-time copies of a PVC; the building block of backup
  tooling (Velero).
- **The expert posture on databases:** requests → StatefulSet → operator → managed service, in
  increasing order of "how much do I want someone else to carry the pager." All four are
  legitimate; know which rung you're on and why.

---

## 8. Cluster lifecycle and the machinery of deletion

### Owner references and garbage collection

When a Deployment makes a ReplicaSet makes a Pod, each child records its parent in
`ownerReferences`. Delete the parent and a garbage collector deletes the orphans — that's why
deleting a Deployment takes its Pods with it, and it's the same mechanism behind "delete the
namespace, everything in it goes" (Phase 11's cleanup).

### Finalizers — why deletes hang

A **finalizer** is a marker on an object saying "someone must do cleanup before this may truly
be deleted." Delete an object with a finalizer and it just sits in `Terminating` until the
responsible controller finishes (detach the cloud disk, release the IP) and removes the marker.
The troubleshooting file's "stuck Terminating" is almost always a finalizer whose controller is
broken or gone. Force-removing finalizers by hand is possible and occasionally necessary — and
skips the cleanup, which is exactly as risky as it sounds.

### Upgrades, drains, and version skew

Clusters upgrade one minor version at a time, control plane first; kubelets may lag the API
server by a couple of minors (the **version skew policy**). Upgrading a node means **cordon**
(no new Pods) then **drain** (evict Pods, respecting PDBs), upgrade, uncordon. APIs also age:
objects get deprecated and *removed* on a published schedule — the classic outage is "our YAML
still says `apiVersion: extensions/v1beta1` and the new cluster refuses it." Experts read
deprecation notes in release notes the way pilots read weather.

### etcd and quorum

The entire cluster state lives in etcd, which keeps consistency by **raft consensus**: a
majority of members (2 of 3, 3 of 5) must agree on every write. That's why control planes come
in odd numbers, why losing quorum makes the whole API read-only-at-best, and why "back up etcd"
is the disaster-recovery advice. Mental model: Kubernetes is a stateless shell around one small,
precious database.

---

## 9. Shipping software at 400 level

- **GitOps** — instead of humans running `kubectl apply`, a controller (Argo CD, Flux) watches
  a git repo and continuously reconciles the cluster to match it. Notice the shape: it's the
  Kubernetes reconciliation loop, applied to deployment itself. Git becomes the audit log, PR
  review becomes change control, rollback becomes `git revert`.
- **Deployment strategies beyond rolling** — **blue/green** (run old and new side by side, flip
  traffic atomically, keep the old warm for instant rollback) and **canary** (route a small
  slice of real traffic to the new version, watch error rates, expand gradually). Kubernetes'
  built-in rolling update can't do either well because it doesn't control *traffic*, only Pod
  counts — hence **progressive delivery** tools (Argo Rollouts, Flagger) that pair Deployments
  with traffic shifting and automatic metric-based rollback.
- **Multi-cluster thinking** — at some scale the question flips from "how do I run this in the
  cluster" to "how many clusters, and what's the blast radius of each?" Common answers: per
  environment, per region, per tenant. The tooling is less standardized here; the *question* is
  what the 400-level person reliably asks.

---

## 10. Security posture (beyond "don't commit secrets")

- **securityContext** — per-Pod/container settings: run as non-root, read-only root filesystem,
  drop Linux capabilities. The single highest-value hardening line:
  `runAsNonRoot: true`.
- **Pod Security Standards** — three built-in profiles (privileged / baseline / restricted)
  that a namespace can enforce at admission. The modern replacement for the deprecated
  PodSecurityPolicy you'll see in old blog posts.
- **Secrets at rest** — Phase 5 told you base64 isn't encryption; the fixes are enabling
  encryption-at-rest in etcd and/or an external manager (Vault, cloud secret stores) synced in
  via — of course — an operator.
- **Supply chain** — scan images for CVEs (Trivy), pin digests instead of mutable tags, sign
  images (cosign/sigstore) and verify signatures at admission (section 2 again). "Where did
  this image come from and can I prove it?" is a 400-level reflex.
- **The `default` ServiceAccount trap** — every Pod gets an identity and can talk to the API
  server with it. If you never thought about it, your nginx Pod may hold credentials it doesn't
  need. Least privilege applies to workloads, not just humans.

---

## How to use this file

Don't memorize it. Notice how few *new ideas* there actually are: reconciliation loops
(sections 1, 2, 9), label selection (4, 5), delegation to plugins (4, 7), interception at
admission (2, 3, 5, 10). Kubernetes is a small number of ideas applied relentlessly. When one
of these topics becomes real for you, come back, re-read its paragraph, then:

> Teach me [concept] hands-on in the style of this repo's phases: a Phase-style lesson on my
> local cluster with a warm-up, a break-it-on-purpose lab, a no-training-wheels challenge, and
> a checkpoint. Assume I've completed Phases 0–11.
