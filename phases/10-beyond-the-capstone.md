# Phase 10 — Beyond the capstone: resources, autoscaling, and scheduling

**Goal:** the "day 2" concepts you'll meet the moment you run Kubernetes beyond a laptop —
telling the scheduler how much your containers need, letting Kubernetes **scale replicas
automatically** (the HPA), and controlling **which node** your Pods land on (nodeSelector,
affinity, taints). None of this is needed to *run* the capstone — all of it is needed to run it
*well* on a real multi-node cluster.

**Time:** ~2 hours. Use your `kind` cluster from Phase 8 — its **three nodes** are what make the
scheduling half of this phase visible.

---

## Before you start — 60 seconds of retrieval

From memory:

1. Who decides which node a new Pod runs on? *(Phase 2's architecture section)*
2. `kubectl scale deployment api --replicas=4` — is that declarative or imperative? Who keeps
   the count at 4 afterwards? *(Phases 2/4)*
3. A Pod is stuck in `Pending`. Which command shows you *why*? *(Phase 4's debugging flow — and
   today you'll see two brand-new reasons for `Pending`)*

---

## Part 1 — Resource requests & limits (the foundation)

So far your containers have asked the cluster for nothing, which means the scheduler is placing
them blind. Real clusters run on two numbers per container:

- **`requests`** — "I need at least this much." The scheduler only places a Pod on a node with
  that much *unreserved* capacity. This is a **promise used for planning**, not a live limit.
- **`limits`** — "never let me use more than this." Exceed the CPU limit and you get throttled;
  exceed the memory limit and the container is killed (`OOMKilled` — a status worth meeting).

Try it. `manifests/resources-demo.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sized
spec:
  replicas: 2
  selector: { matchLabels: { app: sized } }
  template:
    metadata: { labels: { app: sized } }
    spec:
      containers:
        - name: app
          image: ghcr.io/stefanprodan/podinfo
          resources:
            requests:
              cpu: 100m          # 100 millicores = 0.1 of a CPU core
              memory: 64Mi
            limits:
              cpu: 500m
              memory: 256Mi
```

```bash
kubectl apply -f manifests/resources-demo.yaml
kubectl describe node learn-worker | grep -A6 "Allocated resources"   # see reservations add up
```

Rule of thumb to carry forward: **always set requests** (the scheduler is guessing without
them), and set memory limits on anything you don't fully trust. And note for Part 2: **the HPA
can't compute "percent of CPU" if you never said what 100% is — requests are its denominator.**

---

## Part 2 — Autoscaling: the Horizontal Pod Autoscaler (HPA)

You've scaled manually with `kubectl scale`. The **HPA** is the autopilot version — a controller
that watches a metric (CPU, most commonly) and adjusts a Deployment's `replicas` between a floor
and a ceiling you set. This is your "scaling policy":

> "Keep average CPU across the Pods at 50% of what they *requested*. Never fewer than 1 replica,
> never more than 5."

### One-time setup: metrics-server

The HPA reads live usage from **metrics-server**, which local clusters don't ship by default:

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

> On `kind`, metrics-server refuses the nodes' self-signed certs. Patch it once:
>
> ```bash
> kubectl patch -n kube-system deployment metrics-server --type=json \
>   -p '[{"op":"add","path":"/spec/template/spec/containers/0/args/-","value":"--kubelet-insecure-tls"}]'
> ```
>
> When `kubectl top nodes` prints numbers instead of an error, you're ready. (If it doesn't,
> that's a great "paste the evidence to Claude Code" moment.)

### Create the autoscaler and make it work for a living

The `sized` deployment from Part 1 already has CPU requests — exactly what the HPA needs:

```bash
kubectl autoscale deployment sized --cpu-percent=50 --min=1 --max=5
kubectl get hpa sized --watch          # TARGETS shows live%/50% — leave this running
```

In a **second terminal**, generate load through the Service DNS you know from Phase 6:

```bash
kubectl expose deployment sized --port=80 --target-port=9898
kubectl run load --rm -it --image=busybox -- /bin/sh -c \
  "while true; do wget -qO- http://sized > /dev/null; done"
```

Watch the first terminal: CPU% climbs past 50 → the HPA raises `REPLICAS` step by step. Now
`Ctrl-C` the load pod and keep watching: the scale-*down* takes several minutes. That lag is
deliberate — a built-in cooldown so replicas don't flap up and down with every traffic ripple.
Autoscaling reacts in seconds going up and minutes coming down, by design.

```bash
kubectl delete hpa sized               # back to manual control
```

### The scaling family tree (names you'll hear at work)

The HPA scales *replicas*. Its siblings scale other dimensions — none are hands-on today because
they need a cloud, but know them by name:

- **Vertical Pod Autoscaler (VPA)** — resizes a Pod's *requests* instead of the replica count.
- **Cluster Autoscaler** — adds/removes whole **nodes** when Pods go `Pending` for lack of room.
  The classic implementation grows pre-defined **node groups** (Phase 2's term: sets of
  identical machines managed as a unit, like "my pool of 16GB workers") — it can only add more
  of what a group already contains.
- **Karpenter** — the modern successor to the Cluster Autoscaler (born at AWS, now open source
  and CNCF-adopted). Instead of growing a fixed node group, it looks at the `Pending` Pods'
  actual requirements — CPU, memory, architecture, "needs a GPU" — and provisions exactly the
  right machine type in seconds, then consolidates and removes underused nodes to cut cost.
  It can't be demoed locally: its whole job is creating real cloud VMs, and `kind`'s "nodes"
  are just containers on your Mac. You'll meet it in the Phase 11 cloud track.
- **EKS Auto Mode** — AWS's "just handle it" tier for EKS (GKE's equivalent is Autopilot):
  AWS runs Karpenter *for* you, plus manages the nodes' OS, patching, networking, and storage
  drivers. You deploy workloads with requests set (Part 1!) and the cluster provisions and
  scales its own compute. The trade-off is less control and node pricing premium — and every
  concept in this phase still applies inside it, which is why you're learning them.

Notice the layering: **HPA decides how many Pods; Karpenter/Cluster Autoscaler make sure there
are nodes to hold them.** They're designed to work together, not compete.

---

## Part 3 — Scheduling: controlling which node Pods land on

On your 3-node kind cluster, run `kubectl get pods -o wide` — the scheduler has been spreading
Pods across `learn-worker` and `learn-worker2` without asking you. Usually that's what you want.
The exceptions are real, though: databases that must sit on the machine with fast disks, ML pods
that need the GPU node, noisy workloads you want quarantined. Kubernetes gives you three tools,
from bluntest to sharpest.

### Tool 1: `nodeSelector` — "only nodes labeled like this"

Label a node, then require the label:

```bash
kubectl label node learn-worker disk=fast
```

Add to a Pod template's `spec` (alongside `containers:`):

```yaml
      nodeSelector:
        disk: fast
```

```bash
kubectl apply -f manifests/resources-demo.yaml       # after adding the nodeSelector
kubectl get pods -l app=sized -o wide                # every pod now on learn-worker
```

### Tool 2: node affinity — nodeSelector with vocabulary

`nodeSelector` is all-or-nothing. **Node affinity** says it with nuance — and the two key words
are worth memorizing because they name the whole trade-off:

```yaml
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:      # HARD rule: no match, no schedule
            nodeSelectorTerms:
              - matchExpressions:
                  - { key: disk, operator: In, values: [fast] }
          preferredDuringSchedulingIgnoredDuringExecution:     # SOFT rule: try, but don't block
            - weight: 1
              preference:
                matchExpressions:
                  - { key: zone, operator: In, values: [east] }
```

- **`required...`** — a hard constraint. No matching node? The Pod stays `Pending` forever.
- **`preferred...`** — a wish. The scheduler tries; if it can't, the Pod runs somewhere else.
- The mouthful suffix `...IgnoredDuringExecution` means: rules are checked **at scheduling
  time** — changing a node's labels later doesn't evict Pods already running there.

There's also **pod affinity / anti-affinity** — same idea, but relative to *other Pods*. The
canonical use is anti-affinity: "don't put two of my replicas on the same node," so one machine
dying can't take out all three copies. File it under "know it exists"; you'll meet it in any
production chart you read.

### Tool 3: taints & tolerations — the inverse: nodes repel Pods

Labels + affinity = *Pods choose nodes*. **Taints** flip it: *a node refuses Pods* unless they
explicitly tolerate it. This is how you reserve nodes:

```bash
kubectl taint node learn-worker2 reserved=ml:NoSchedule
kubectl rollout restart deployment sized             # force rescheduling (remove the nodeSelector first)
kubectl get pods -o wide                             # nothing lands on learn-worker2 anymore
```

Only a Pod carrying the matching **toleration** in its spec may schedule there:

```yaml
      tolerations:
        - { key: reserved, operator: Equal, value: ml, effect: NoSchedule }
```

You've been benefiting from taints all along: control-plane nodes are tainted by default —
that's why none of your app Pods ever landed on `learn-control-plane`. See for yourself, then
clean up:

```bash
kubectl describe node learn-control-plane | grep -i taint
kubectl taint node learn-worker2 reserved=ml:NoSchedule-     # trailing "-" REMOVES a taint
kubectl label node learn-worker disk-                        # "-" removes a label too
```

---

## Break it on purpose: the two new faces of `Pending`

In Phase 4, `Pending` meant "scheduling in progress." Now you can make it mean "scheduling
*impossible*" — and read the scheduler's own explanation.

**Impossible demand** — request more CPU than any node has:

```bash
kubectl run greedy --image=nginx --overrides='{"spec":{"containers":[{"name":"greedy","image":"nginx","resources":{"requests":{"cpu":"64"}}}]}}'
kubectl get pod greedy                   # Pending — and it will stay there
kubectl describe pod greedy              # Events: "Insufficient cpu" on every node
```

**Impossible affinity** — require a label no node carries: add a `nodeSelector` of
`disk: nvme` to any Deployment and apply it. `describe` on the new Pod says
`didn't match Pod's node affinity/selector`.

The lesson that generalizes: a `Pending` Pod is the scheduler telling you *it has nowhere legal
to put it* — and `kubectl describe pod` always contains its reasoning, node by node. Clean up:

```bash
kubectl delete pod greedy
kubectl delete -f manifests/resources-demo.yaml
kubectl delete service sized --ignore-not-found
```

---

## Challenge — no training wheels ✋

Combine all three parts on your kind cluster, from blank files:

1. Label `learn-worker` with `tier=db` and taint it `dedicated=db:NoSchedule` — a node that
   *only* database Pods may use, and that database Pods *must* use.
2. Deploy Redis (from Phase 8) so it schedules there: it needs **both** a toleration and a
   nodeSelector/affinity. Prove placement with `kubectl get pods -o wide`.
3. Deploy podinfo with CPU requests, an HPA (`min=2, max=6, cpu 50%`), and confirm with load
   that it scales up — and that **no podinfo Pod ever lands on the db node**.
4. Explain (out loud) why step 2 needed both halves — what does the taint do that the label
   doesn't, and vice versa?
5. Untaint, unlabel, clean up.

---

## Checkpoint ✅

- [ ] You can explain requests vs limits, and what happens when each is exceeded.
- [ ] You watched an HPA scale a Deployment up under load — and knew why the scale-down lagged.
- [ ] You can order nodeSelector → node affinity → taints by what each adds, and say which
      *actor* (Pod or node) does the choosing.
- [ ] `required...` vs `preferred...` affinity: one sentence each.
- [ ] You diagnosed both new `Pending` causes from the Events in `kubectl describe pod`.
- [ ] Challenge completed, including the step-4 explanation.

---

## Where to go from here (pointers, not lessons)

Each of these is its own rabbit hole; you now have the foundation for all of them:
**RBAC** (who may do what in the cluster), **NetworkPolicies** (firewalls between Pods),
**PodDisruptionBudgets** (protecting availability during node maintenance), **Operators/CRDs**
(teaching Kubernetes new object types), and **GitOps** (Argo CD / Flux — your `kubectl apply`
loop, automated from git). All of these — and the rest of the expert-level map — are explained
in plain English in [Phase 12](12-expert-concepts.md).

---

## Ask Claude Code

**Teach-back:**

> Quiz me one question at a time on: requests vs limits, how the HPA decides to scale (and why
> it needs requests), nodeSelector vs node affinity vs taints/tolerations, and required vs
> preferred affinity. Probe with "what breaks if…" follow-ups until I'm solid.

**Go deeper:**

> Here are my capstone manifests in `manifests/mcp-gateway/`. Propose sensible resource requests
> and limits for each container, an HPA for the Registry API only (explain why the other tiers
> are poor HPA candidates), and pod anti-affinity for anything with more than one replica.
> Explain every choice so I can defend it.

**You're no longer a beginner.** This phase's vocabulary — requests, HPA, affinity, taints — is
the language of production Kubernetes. Everything else is depth, and you know how to dig now.

**Next:** [Phase 11 — Capstone 2: serve an open-source LLM](11-capstone-2-llm-serving.md), where
this phase's scheduling tools become the backbone of GPU serving.
