# Phase 8 — A realistic multi-service app (dress rehearsal for the capstone)

**Goal:** deploy a small but *complete* multi-service app — a web API (with its own UI) backed by
a datastore it genuinely depends on — so that every idea from Phases 4–7 comes together on
something with the same *shape* as `mcp-gateway-registry`, but simpler. Also: switch to `kind` for a more production-like cluster.

**Time:** ~2 hours. This is the big consolidation phase.

---

## Before you start — 60 seconds of retrieval

From memory:

1. A Service is `Running` with an IP, Pods are `READY 1/1`, but requests hang. What's the first
   command you run, and what are you looking for? *(Phase 6's break-lab)*
2. What happens to data written inside a container's filesystem when its Pod is replaced?
   What object prevents that? *(Phase 5 — today you'll watch this happen to a real cache)*
3. Chart vs release — which one is `my-podinfo`? *(Phase 7)*

---

## Step 1 (optional but recommended): move to `kind`

Docker Desktop's Kubernetes was perfect for learning. `kind` ("Kubernetes IN Docker") gives you a
more realistic, scriptable, multi-node cluster — closer to what the capstone benefits from. It
also lets you tear down and recreate clusters instantly.

```bash
brew install kind
```

Create a cluster with a config that maps ports so Ingress works (`kind-config.yaml` at repo root):

```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
    kubeadmConfigPatches:
      - |
        kind: InitConfiguration
        nodeRegistration:
          kubeletExtraArgs:
            node-labels: "ingress-ready=true"
    extraPortMappings:
      - containerPort: 80
        hostPort: 80
        protocol: TCP
      - containerPort: 443
        hostPort: 443
        protocol: TCP
  - role: worker
  - role: worker
```

```bash
kind create cluster --name learn --config kind-config.yaml
#                          └─ "learn" is a name you choose for the cluster; you'll repeat it in
#                             every later kind command (and it names the nodes: learn-worker…)
kubectl config use-context kind-learn      # kind creates the context as kind-<your-name>
kubectl get nodes                          # THREE nodes now (1 control-plane, 2 workers)!
```

Install the Ingress controller built for kind:

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml
kubectl wait --namespace ingress-nginx \
  --for=condition=ready pod \
  --selector=app.kubernetes.io/component=controller \
  --timeout=120s
```

> Prefer to stay on Docker Desktop? That's fine — skip this step and use `docker-desktop` context.
> Everything below works on either. Just make sure `kubectl config current-context` is what you
> expect before applying anything.

---

## Step 2: the app we'll deploy

We'll deploy a two-tier stack with a **real dependency between the tiers** — the same shape as
the capstone, with fewer moving parts:

```text
   Browser ──► Ingress ──► api (podinfo, 2 replicas) ──► redis (data tier)
```

The API tier is **podinfo** (which you installed via Helm in Phase 7 — today you write its
manifests yourself). Pointed at Redis with `--cache-server`, its `/cache/*` endpoints genuinely
read and write through to Redis — so when we test cross-service communication, we'll be testing
the real thing, not taking it on faith.

Map that to `mcp-gateway-registry`:

- api (podinfo, serves a UI too)  ≈  the Registry API + UI (port 7860)
- redis                            ≈  MongoDB

Same shape, fewer moving parts. If you can do this, the capstone is "the same, but more."

`manifests/demo-app/redis.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis
spec:
  replicas: 1
  selector: { matchLabels: { app: redis } }
  template:
    metadata: { labels: { app: redis } }
    spec:
      containers:
        - name: redis
          image: redis:7-alpine
          ports: [{ containerPort: 6379 }]
---
apiVersion: v1
kind: Service
metadata:
  name: redis
spec:
  selector: { app: redis }
  ports: [{ port: 6379, targetPort: 6379 }]
```

`manifests/demo-app/api.yaml` — podinfo, told to use Redis **by Service name** (this is Phase 6's
DNS discovery doing real work):

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
spec:
  replicas: 2
  selector: { matchLabels: { app: api } }
  template:
    metadata: { labels: { app: api } }
    spec:
      containers:
        - name: api
          image: ghcr.io/stefanprodan/podinfo
          command: ["./podinfo"]
          args: ["--cache-server=tcp://redis:6379"]   # <- "redis" is the Service name. DNS does the rest.
          ports: [{ containerPort: 9898 }]
---
apiVersion: v1
kind: Service
metadata:
  name: api
spec:
  selector: { app: api }
  ports: [{ port: 80, targetPort: 9898 }]
```

`manifests/demo-app/ingress.yaml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: demo
spec:
  ingressClassName: nginx
  rules:
    - host: demo.localhost
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service: { name: api, port: { number: 80 } }
```

---

## Step 3: deploy it all and verify — including the tier-to-tier link

```bash
kubectl apply -f manifests/demo-app/redis.yaml
kubectl apply -f manifests/demo-app/api.yaml
kubectl apply -f manifests/demo-app/ingress.yaml

kubectl get pods                     # redis + 2 api pods, all Running
curl http://demo.localhost           # -> podinfo's JSON greeting (through Ingress!)
```

That proves browser → Ingress → Service → api. Now prove **api → redis** — write a value through
podinfo's cache API and read it back:

```bash
curl -X POST http://demo.localhost/cache/team -d 'mcp-gateway'
curl http://demo.localhost/cache/team          # -> mcp-gateway
```

That value made a round trip: your laptop → Ingress → `api` Service → a podinfo Pod → the
`redis` Service *by DNS name* → the Redis Pod → and back. Don't take my word that Redis stored
it — ask Redis directly:

```bash
kubectl exec deploy/redis -- redis-cli keys '*'
#            └─ deploy/<name> picks a pod FROM that deployment (saves you copying pod names);
#               after the "--", the command runs inside the container: redis's own CLI
```

You now have multiple tiers, each behind a Service, one exposed through Ingress, with the API
finding Redis purely by the name `redis` — **verified end to end**, not assumed.

### Watch the dependency for real: kill the data tier

Phase 5 warned that a container's filesystem dies with its Pod. Watch that bite a real app:

```bash
kubectl delete pod -l app=redis        # kill the data tier
kubectl get pods -l app=redis          # ReplicaSet already made a NEW redis pod (self-healing!)
curl http://demo.localhost/cache/team  # -> 404. The pod came back; the DATA didn't.
```

Sit with what just happened: Kubernetes healed the *Pod* instantly, and the Service kept routing
— but we never gave Redis a PVC, so the new Pod started with an empty filesystem. Self-healing
protects **processes**, only persistent storage protects **data**. This is exactly why MongoDB
gets a PVC in the capstone, and it's the challenge below.

---

## Step 4: practice the operations you'll need for the capstone

```bash
kubectl get all                              # the whole app at a glance
kubectl logs -l app=api --prefix             # logs across both api pods
kubectl scale deployment api --replicas=4    # scale a tier
kubectl rollout restart deployment api       # restart a tier cleanly
kubectl describe ingress demo                # confirm routing rules
```

If a Pod won't start, run the Phase 4 flow: `get pods → describe pod → logs`. Every capstone
problem will be diagnosed with these same commands.

---

## Challenge — no training wheels ✋

Make the cache survive Pod replacement, using only what Phase 5 taught you:

1. Add a PVC and mount it at `/data` in the redis Deployment (that's where Redis keeps its
   snapshot file, `dump.rdb`).
2. Write a value via `/cache/...`, then `kubectl delete pod -l app=redis` and wait for the
   replacement.
3. Read the value back. (Redis saves a snapshot on graceful shutdown, so the new Pod finds it on
   the volume. If the value is still gone, check with `kubectl exec` that the mount is really at
   `/data` — debugging this *is* the exercise.)

When it works, you've built the full stateful-service pattern the capstone's MongoDB needs:
Deployment + Service + PVC, verified by killing the Pod.

---

## Clean up

```bash
kubectl delete -f manifests/demo-app/
# If you used kind and want to reset entirely:
# kind delete cluster --name learn
```

---

## Checkpoint ✅ (the important one)

- [ ] A multi-tier app runs, each tier behind its own Service.
- [ ] You **proved** one tier reaches the other by Service name — a value round-tripped through
      Redis, and you saw the key inside the Redis Pod.
- [ ] The app is reachable from your browser through an Ingress.
- [ ] You watched self-healing recreate the data tier — and watched the data vanish, then fixed
      it with a PVC (the challenge).
- [ ] You can scale, restart, and read logs across a multi-Pod tier.

If all five are true, **you are ready for the real thing.**

---

## Ask Claude Code

> Here are my `redis.yaml`, `api.yaml`, and `ingress.yaml`. Review them like a Kubernetes mentor:
> point out anything fragile, anything not production-shaped, and how this structure maps onto the
> `mcp-gateway-registry` services (Registry API, Auth server, nginx gateway, MongoDB).

**Next:** [Phase 9 — Capstone: deploy mcp-gateway-registry](09-capstone-mcp-gateway.md)
