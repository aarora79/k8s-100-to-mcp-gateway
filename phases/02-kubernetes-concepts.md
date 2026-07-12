# Phase 2 — Kubernetes concepts (no cluster yet)

**Goal:** build the mental model *before* touching a cluster. When you meet the commands in
Phase 3, you'll already know what they're doing and why.

**Time:** ~45 minutes of reading + thinking. No installs.

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
A way to partition a cluster into virtual sub-clusters for organization (e.g. `default`,
`kube-system`, and later a `mcp-gateway` namespace for the capstone).

---

## How they fit together (picture)

```
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

## When NOT to use Kubernetes (staying honest)

For running one app on one machine, **Docker Compose is simpler and sufficient** — which is why
`mcp-gateway-registry` ships with a `docker-compose.yml`. Kubernetes earns its complexity when you
need self-healing, scaling, rolling updates, and multi-machine deployments. Here, we're learning
it deliberately, and porting a Compose app is a superb teacher. Keep this perspective — it'll make
you a better engineer than someone who reaches for Kubernetes reflexively.

---

## Checkpoint ✅

Without looking, explain in one sentence each: Pod, Deployment, Service, ConfigMap, Namespace,
and the difference between "declarative" and "imperative". If you can, you're ready for a real
cluster.

---

## Ask Claude Code

> Quiz me on Kubernetes core objects (Pod, Deployment, ReplicaSet, Service, ConfigMap, Secret,
> Namespace). Ask me one at a time, tell me if I'm right, and correct my mental model where it's
> fuzzy. I'm a beginner.

**Next:** [Phase 3 — Stand up a local cluster](03-local-cluster.md)
