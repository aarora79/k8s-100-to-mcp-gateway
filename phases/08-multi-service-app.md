# Phase 8 — A realistic multi-service app (dress rehearsal for the capstone)

**Goal:** deploy a small but *complete* multi-service app — a web frontend + an API + a database —
so that every idea from Phases 4–7 comes together on something with the same *shape* as
`mcp-gateway-registry`, but simpler. Also: switch to `kind` for a more production-like cluster.

**Time:** ~2 hours. This is the big consolidation phase.

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
kubectl config use-context kind-learn      # kind sets this context up for you
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

We'll deploy the classic **"guestbook"-style** stack, which mirrors the capstone's structure:

```
   Browser ──► Ingress ──► frontend (web UI) ──► api (backend) ──► redis/mongo (data)
```

Map that to `mcp-gateway-registry`:

- frontend/UI  ≈  the Registry UI (port 7860)
- api          ≈  the Registry API + Auth server
- data         ≈  MongoDB

Same shape, fewer moving parts. If you can do this, the capstone is "the same, but more."

We'll use lightweight public images so you don't have to build anything: a small demo API and
Redis as the datastore.

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

`manifests/demo-app/api.yaml` (a tiny HTTP echo/demo API that talks to Redis — we use a generic
image and point it at the `redis` Service by name):

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
          image: hashicorp/http-echo         # simple, replace with a real API later
          args: ["-text=hello from the api tier"]
          ports: [{ containerPort: 5678 }]
          # In a real app you'd pass REDIS_HOST=redis here via env/ConfigMap:
          env:
            - name: REDIS_HOST
              value: "redis"                  # <- service discovery by name!
---
apiVersion: v1
kind: Service
metadata:
  name: api
spec:
  selector: { app: api }
  ports: [{ port: 80, targetPort: 5678 }]
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

## Step 3: deploy it all and verify

```bash
kubectl apply -f manifests/demo-app/redis.yaml
kubectl apply -f manifests/demo-app/api.yaml
kubectl apply -f manifests/demo-app/ingress.yaml

kubectl get pods                     # redis + 2 api pods, all Running
curl http://demo.localhost           # -> hello from the api tier (through Ingress!)
```

You now have: multiple tiers, each behind a Service, one exposed through Ingress, with the API
finding Redis purely by the name `redis`. That is a complete multi-service application on
Kubernetes.

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

## Clean up

```bash
kubectl delete -f manifests/demo-app/
# If you used kind and want to reset entirely:
# kind delete cluster --name learn
```

---

## Checkpoint ✅ (the important one)

- [ ] A three-tier app runs, each tier behind its own Service.
- [ ] One tier reaches another using only the Service name (no IPs).
- [ ] The app is reachable from your browser through an Ingress.
- [ ] You can scale, restart, and read logs across a multi-Pod tier.

If all four are true, **you are ready for the real thing.**

---

## Ask Claude Code

> Here are my `redis.yaml`, `api.yaml`, and `ingress.yaml`. Review them like a Kubernetes mentor:
> point out anything fragile, anything not production-shaped, and how this structure maps onto the
> `mcp-gateway-registry` services (Registry API, Auth server, nginx gateway, MongoDB).

**Next:** [Phase 9 — Capstone: deploy mcp-gateway-registry](09-capstone-mcp-gateway.md)
