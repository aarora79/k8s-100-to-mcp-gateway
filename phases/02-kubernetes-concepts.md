# Phase 2 — Kubernetes concepts (no cluster yet)

**Goal:** build the mental model *before* touching a cluster. When you meet the commands in
Phase 3, you'll already know what they're doing and why.

**Time:** ~60–75 minutes of reading + thinking. No installs.

---

## The problem Kubernetes solves

In Phase 1 you ran single containers by hand. Now imagine the real world:

- You have *many* containers (a web API, an auth service, a gateway, a database).
- They need to **find each other** by name.
- If one **crashes**, something should restart it automatically.
- When traffic spikes, you want **more copies**; when it drops, fewer.
- You want to **roll out a new version** without downtime, and **roll back** if it breaks.
- Eventually this runs across **many machines**, not just your Mac.

Doing all that by hand with `docker run` is impossible to manage. **Kubernetes (often "K8s" — K,
then 8 letters, then s) is a system that does it for you.** You *declare* the desired state
("I want 3 copies of this app, reachable at this name"), and Kubernetes continuously works to
make reality match that declaration. This "declare desired state, let the system converge" idea is
the single most important concept in Kubernetes.

---

## Declarative, not imperative

- **Imperative** (what you did in Phase 1): "run this container, now stop it, now start another."
  You issue step-by-step commands.
- **Declarative** (Kubernetes): "here is a YAML file describing what I want to exist." Kubernetes
  makes it so and *keeps* it so. Delete a running copy and Kubernetes recreates it, because your
  declaration said there should be one.

You'll write YAML files describing objects, and apply them with `kubectl apply -f file.yaml`.

---

## The architecture (just enough)

A Kubernetes **cluster** has two kinds of machines (on your Mac, both roles run inside one VM):

**Control plane** — the brain. Key parts:

- **API server** — the front door. Everything (including `kubectl`) talks to it.
- **etcd** — the database that stores the entire desired + current state of the cluster.
- **Scheduler** — decides which node a new container should run on.
- **Controller manager** — the "make reality match desire" loops (e.g. "only 2 of my 3 wanted
  copies exist → start one more").

**Nodes (workers)** — where your containers actually run. Each node has:

- **kubelet** — the agent that talks to the control plane and runs containers on that node.
- **Container runtime** — the thing that actually runs containers (containerd).
- **kube-proxy** — handles networking rules so services are reachable.

You rarely touch these directly — but knowing they exist demystifies error messages later.

---

## The core objects (this is the vocabulary you'll use daily)

Learn these seven. Everything else builds on them.

### 1. Pod

The **smallest deployable unit** in Kubernetes. A Pod wraps one (occasionally a few tightly
coupled) containers that share a network address and storage. You almost never create Pods
directly — you create a Deployment that manages them. Think: "a single running instance of your
app."

### 2. Deployment

Describes a Pod and **how many copies (replicas)** you want, plus how to update them. The
Deployment controller keeps that many Pods alive and handles rolling updates/rollbacks. This is
what you'll create most often. Think: "keep N copies of this app running, and let me upgrade them
safely."

### 3. ReplicaSet

The thing a Deployment creates under the hood to maintain the replica count. You usually don't
manage it directly — just know that `Deployment → ReplicaSet → Pods`.

> **"Replica" vs "ReplicaSet" — not the same thing.** This trips up almost everyone, so let's
> nail it down:
>
> - A **replica** is not a Kubernetes object at all — it's just a *count*. "3 replicas of nginx"
>   means "keep 3 identical nginx Pods running at all times." One replica = one of the copies.
> - A **ReplicaSet** is an actual Kubernetes object — the *manager* whose only job is to enforce
>   that count. It constantly asks: "Do I have exactly N Pods matching my label selector? If
>   fewer, create some. If more, kill some."
>
> An analogy: if replicas are the **security guards on duty**, the ReplicaSet is the **shift
> supervisor** whose only job is making sure exactly 3 guards are on the floor at all times. A
> guard quits (a Pod crashes)? The supervisor immediately calls in a replacement. The supervisor
> isn't a guard — the supervisor *maintains the number of guards*.
>
> Why does the Deployment exist on top? Because a ReplicaSet is dumb about *versions* — the
> Deployment manages ReplicaSets so you get safe upgrades and rollbacks. You'll see exactly how
> (and watch the whole `Deployment → ReplicaSet → Pod` chain live) in Phase 4.

### 4. Service

Pods are ephemeral — they come and go, and their IP addresses change. A **Service** gives a
*stable name and address* that load-balances across the current set of Pods. Think: "a permanent
front door for a group of interchangeable Pods." Types (ClusterIP, NodePort, LoadBalancer) are
covered in Phase 6.

### 5. ConfigMap

Non-secret configuration (env vars, config files) kept *outside* your image so you can change
settings without rebuilding. Phase 5.

### 6. Secret

Like a ConfigMap but for sensitive values (passwords, API keys, tokens). Phase 5.

### 7. Namespace

A way to partition one cluster into virtual sub-clusters. Think: "folders for your Kubernetes
objects." Three things a beginner actually needs to know:

- **Every namespaced object lives in exactly one namespace.** If you don't say which, it goes in
  the one literally named `default` — that's where everything you create in Phases 4–6 will land.
  The cluster's own machinery runs in `kube-system` (look, don't touch). Later you'll create a
  `mcp-gateway` namespace so the whole capstone app lives in one tidy, deletable box.
- **kubectl commands are namespace-scoped.** `kubectl get pods` really means "get pods *in the
  default namespace*". Add `-n <name>` to look elsewhere, or `-A` for all namespaces. "My pod
  disappeared!" is very often "I'm looking in the wrong namespace."
- **Names only need to be unique *within* a namespace** — two teams can both have a Service named
  `api` in different namespaces. That's also why the full DNS name of a Service includes its
  namespace: `api.team-a.svc.cluster.local` vs `api.team-b.svc.cluster.local`. Within the *same*
  namespace, plain `api` works (which is what the worked example below relies on).

---

## How they fit together (picture)

```text
        You write YAML  ──apply──►  API server ──stores──► etcd
                                        │
                                        ▼
                              controllers + scheduler
                                        │
             ┌──────────────────────────┼───────────────────────────┐
             ▼                          ▼                            ▼
        Deployment  ──manages──►  ReplicaSet  ──creates──►  Pods (your containers)
             ▲                                                       ▲
             │                                                       │
          Service  ──gives a stable name & load-balances──►  (this set of Pods)
```

Mapping it to `mcp-gateway-registry` (your capstone target): the Registry API becomes a
Deployment + Service, the Auth server becomes a Deployment + Service, the nginx MCP Gateway
becomes a Deployment + Service (plus Ingress), and MongoDB becomes a Deployment/StatefulSet +
Service + persistent storage. Configuration from `.env` becomes ConfigMaps and Secrets. You'll
build exactly this in Phase 9.

---

## Worked example: a simple FastAPI app + database + Ingress

The diagram above shows the machinery. Here is the same picture for a **concrete app** you can
hold in your head: a FastAPI API with 3 replicas, talking to a Postgres database, reachable from
your browser through an Ingress. Every object below is one you just learned.

```text
                                 your browser
                              http://myapp.local/api
                                       │
                                       ▼
                        ┌───────────────────────────────┐
                        │            Ingress            │  "route myapp.local/api → api Service"
                        │  (rules; an ingress controller│
                        │   e.g. nginx does the work)   │
                        └───────────────┬───────────────┘
                                        │
                                        ▼
                        ┌───────────────────────────────┐
                        │        Service: "api"         │  stable name + load-balancer
                        │          (ClusterIP)          │  for the FastAPI Pods
                        └──────┬────────┬────────┬──────┘
                               │        │        │        ◄── load-balances across replicas
                               ▼        ▼        ▼
                          ┌───────┐┌───────┐┌───────┐
        Deployment "api"  │ Pod 1 ││ Pod 2 ││ Pod 3 │     3 replicas of the FastAPI container
        (via ReplicaSet)  │fastapi││fastapi││fastapi│     (managed by a ReplicaSet, which the
                          └───┬───┘└───┬───┘└───┬───┘      Deployment created)
                              │        │        │
                              └────────┼────────┘
                                       │  connects to "postgres:5432" — just a DNS name!
                                       ▼
                        ┌───────────────────────────────┐
                        │      Service: "postgres"      │  stable name for the database
                        │          (ClusterIP)          │
                        └───────────────┬───────────────┘
                                        │
                                        ▼
                                  ┌───────────┐
        Deployment/StatefulSet    │    Pod    │           1 replica — databases don't scale
        "postgres"                │ postgres  │           by adding copies the way stateless
                                  └─────┬─────┘           apps do
                                        │
                                        ▼
                        ┌───────────────────────────────┐
                        │   PersistentVolumeClaim (PVC) │  disk that survives Pod restarts —
                        │      → PersistentVolume       │  your data outlives the container
                        └───────────────────────────────┘

        Feeding both apps:
        ┌────────────────────────────┐   ┌────────────────────────────┐
        │  ConfigMap: "api-config"   │   │  Secret: "db-credentials"  │
        │  e.g. LOG_LEVEL, DB_HOST   │   │  e.g. POSTGRES_PASSWORD    │
        └────────────┬───────────────┘   └────────────┬───────────────┘
                     └───── injected as env vars ─────┘
                            into the Pods above
```

Read it top to bottom as a request flows:

1. **Browser → Ingress.** The Ingress holds routing rules ("traffic for `myapp.local/api` goes
   to the `api` Service"). An *ingress controller* (usually nginx) is the actual software doing
   the routing — you'll install one in Phase 6.
2. **Ingress → Service "api".** The Service is the stable front door. It doesn't care that Pods
   die and get replaced with new IPs — it always knows the current healthy set and
   **load-balances** each request across the 3 FastAPI replicas. (*ClusterIP* is the Service's
   type — it means "reachable only inside the cluster"; Service types are covered in Phase 6.)
3. **FastAPI Pods → Service "postgres".** The app connects to the database with a plain DNS name
   (`postgres:5432`) — no IP addresses in your code, ever. That name is the database's Service.

   > **Who makes that name work? Kubernetes itself — and this is one of its quiet superpowers.**
   > The cluster runs an internal DNS server (CoreDNS, installed automatically). The moment you
   > create a Service named `postgres`, that name resolves *inside the cluster* to the Service's
   > stable virtual IP, which in turn forwards to whatever healthy Pod(s) currently back it. You
   > configured nothing — no `/etc/hosts` entries, no hardcoded IPs, no service registry to run.
   >
   > Contrast with life without Kubernetes: your FastAPI code would point at `localhost:5432` or
   > some machine's actual IP, and the moment the database moved, restarted with a new address,
   > or ran on a different host, your config breaks. Here the *name* is the contract. The
   > database Pod can die, get rescheduled to another node, and come back with a totally
   > different IP — and `postgres:5432` still just works, because Kubernetes keeps the
   > name → current-Pod mapping up to date in real time. This built-in **service discovery** is
   > a big part of why teams put up with Kubernetes' complexity.
4. **Postgres Pod → PVC.** The database writes to a PersistentVolumeClaim, so if the Pod is
   rescheduled or restarts, the data is still there (Phase 5). (A *StatefulSet* is a Deployment
   variant designed for stateful apps like databases — stable identity, careful ordering. A plain
   Deployment works fine for learning; you'll meet StatefulSets in Phase 9.)
5. **ConfigMap & Secret** hold the settings (log level, DB host) and credentials (DB password),
   injected into the Pods as environment variables — nothing baked into images.

Notice what you *didn't* have to draw: the ReplicaSets (implied under each Deployment) and the
Pods' IP addresses (nobody talks to a Pod directly — always through a Service). If a FastAPI Pod
crashes, the ReplicaSet replaces it and the Service adds the newcomer back into rotation — no
config change anywhere. This exact shape — stateless API tier behind a Service + Ingress,
stateful database behind its own Service + storage, config split into ConfigMaps/Secrets — is the
pattern you'll repeat for every service in the Phase 9 capstone.

---

## When NOT to use Kubernetes (staying honest)

For running one app on one machine, **Docker Compose is simpler and sufficient** — which is why
`mcp-gateway-registry` ships with a `docker-compose.yml`. Kubernetes earns its complexity when you
need self-healing, scaling, rolling updates, and multi-machine deployments. Here, we're learning
it deliberately, and porting a Compose app is a superb teacher. Keep this perspective — it'll make
you a better engineer than someone who reaches for Kubernetes reflexively.

---

## Checkpoint ✅

Without looking, explain in one sentence each: Pod, Deployment, Service, ConfigMap, Namespace,
and the difference between "declarative" and "imperative". Then two more:

- What's the difference between a **replica** and a **ReplicaSet**?
- Your app connects to its database at `postgres:5432`. Who makes that name resolve, and why
  does it keep working when the database Pod is replaced?

If you can answer these, you're ready for a real cluster.

---

## Ask Claude Code

> Quiz me on Kubernetes core objects (Pod, Deployment, ReplicaSet, Service, ConfigMap, Secret,
> Namespace). Ask me one at a time, tell me if I'm right, and correct my mental model where it's
> fuzzy. I'm a beginner.

**Next:** [Phase 3 — Stand up a local cluster](03-local-cluster.md)
