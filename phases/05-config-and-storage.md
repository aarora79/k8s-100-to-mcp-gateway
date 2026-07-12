# Phase 5 — Config, Secrets, and Storage

**Goal:** get configuration and sensitive values *out of* your images, and keep data alive when
Pods die. The capstone needs all three (the app has a `.env` file and a database).

**Time:** ~1–1.5 hours.

---

## Before you start — 60 seconds of retrieval

From memory:

1. Why do we almost never create Pods directly — what do we create instead? *(Phase 2/4)*
2. In Phase 4's break-lab, what was the difference between `ImagePullBackOff` and
   `CrashLoopBackOff`? *(One of today's demos can cause the second one — missing config.)*
3. From Phase 1: what happens to files written inside a container when the container is deleted?

That third answer is exactly the problem the second half of this phase solves.

---

## Why this phase matters for the capstone

`mcp-gateway-registry` is configured with a `.env` file (database credentials, identity-provider
settings, feature flags) and stores data in MongoDB. In Kubernetes:

- Non-secret settings → **ConfigMap**
- Passwords / keys / tokens → **Secret**
- The database's data → a **PersistentVolume** so it survives Pod restarts

Master these here on toy examples so Phase 9 is assembly, not new learning.

---

## ConfigMaps — non-secret configuration

A ConfigMap is key/value data you inject into Pods as environment variables or files.

`manifests/demo-configmap.yaml`:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  APP_GREETING: "Hello from a ConfigMap"
  APP_MODE: "learning"
```

Consume it as env vars in a Pod. `manifests/config-demo-pod.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: config-demo
spec:
  containers:
    - name: shell
      image: busybox
      command: ["sh", "-c", "echo greeting=$APP_GREETING mode=$APP_MODE; sleep 3600"]
      envFrom:
        - configMapRef:
            name: app-config       # inject ALL keys as env vars
```

```bash
kubectl apply -f manifests/demo-configmap.yaml
kubectl apply -f manifests/config-demo-pod.yaml
kubectl logs config-demo           # -> greeting=Hello from a ConfigMap mode=learning
```

Change a value in the ConfigMap and re-apply; you'd restart the Pod to pick it up. The point:
**you changed configuration without rebuilding the image.**

---

## Secrets — sensitive configuration

Secrets look like ConfigMaps but are meant for sensitive data. Create one imperatively (easier for
values you don't want to hand-encode):

```bash
kubectl create secret generic db-credentials \
  --from-literal=MONGO_USER=admin \
  --from-literal=MONGO_PASSWORD='sup3r-s3cret'
kubectl get secret db-credentials -o yaml   # note values are base64-encoded
```

> **Honest caveat:** base64 is *encoding, not encryption* — anyone with cluster read access can
> decode it. Secrets are about keeping values out of your images and YAML-in-git, and about
> access control (RBAC). For real production secrets you'd add encryption-at-rest or an external
> secrets manager. For local learning, plain Secrets are fine — just never commit real Secret
> YAML to a public repo. (Note the `.gitignore` in this repo.)

Consume a Secret exactly like a ConfigMap:

```yaml
      envFrom:
        - secretRef:
            name: db-credentials
```

---

## Storage — keeping data alive

Recall from Phase 1: when a container dies, its filesystem is gone. For a database that's a
disaster. Kubernetes solves it with three related objects:

- **PersistentVolume (PV)** — a piece of actual storage in the cluster.
- **PersistentVolumeClaim (PVC)** — a Pod's *request* for storage ("I need 1Gi"). The cluster
  binds it to a PV.
- **StorageClass** — describes *how* to provision storage on demand. Docker Desktop ships a
  default one, so PVCs get satisfied automatically — you usually only write the PVC.

`manifests/demo-pvc.yaml`:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: data
spec:
  accessModes: ["ReadWriteOnce"]   # mounted read-write by one node
  resources:
    requests:
      storage: 1Gi
```

Use it in a Pod by mounting it. `manifests/storage-demo-pod.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: storage-demo
spec:
  containers:
    - name: writer
      image: busybox
      command: ["sh", "-c", "echo persisted-$(date) >> /data/log.txt; cat /data/log.txt; sleep 3600"]
      volumeMounts:
        - name: data
          mountPath: /data          # the container sees the volume here
  volumes:
    - name: data
      persistentVolumeClaim:
        claimName: data
```

```bash
kubectl apply -f manifests/demo-pvc.yaml
kubectl apply -f manifests/storage-demo-pod.yaml
kubectl get pvc                        # STATUS should become Bound
kubectl logs storage-demo              # writes a line to /data/log.txt

# Now prove persistence: delete the pod, recreate it, data survives
kubectl delete pod storage-demo
kubectl apply -f manifests/storage-demo-pod.yaml
kubectl logs storage-demo              # the OLD line is still there + a new one
```

That surviving line is the whole point: the PVC outlived the Pod.

> **Databases specifically** often use a **StatefulSet** instead of a Deployment (stable network
> identity + stable per-replica storage). For a single MongoDB in the capstone, a Deployment +
> PVC is enough; we'll mention StatefulSets there.

---

## Challenge — no training wheels ✋

From blank files, without looking at the YAML above:

1. Create a ConfigMap `challenge-config` with keys `COLOR=blue` and `SIZE=large`, **and** a
   Secret `challenge-secret` with key `TOKEN=s3cret` (imperative `kubectl create secret` is fine).
2. Write a Pod that injects *both* as env vars and prints all three values with `env | grep -E
   'COLOR|SIZE|TOKEN'` before sleeping.
3. Verify with `kubectl logs`, then clean everything up.

Bonus (extends Phase 4's break-lab): delete the Secret and recreate the Pod. You'll meet a *new*
status — `CreateContainerConfigError` — because the Pod references config that doesn't exist, so
the container can't even start. `kubectl describe pod` names the missing Secret. Add it to your
mental collection: **ImagePullBackOff** (can't get the image), **CreateContainerConfigError**
(can't assemble the config), **CrashLoopBackOff** (started, then died).

---

## Clean up

```bash
kubectl delete pod config-demo storage-demo --ignore-not-found
kubectl delete -f manifests/demo-pvc.yaml
kubectl delete configmap app-config
kubectl delete secret db-credentials
```

---

## Checkpoint ✅

- [ ] Inject config into a Pod via a ConfigMap (env vars).
- [ ] Create a Secret and explain how it differs from a ConfigMap (and the base64 caveat).
- [ ] Create a PVC, mount it, and prove data survives a Pod deletion.
- [ ] Complete the challenge from blank files, including the missing-Secret bonus.

---

## Ask Claude Code

> Explain the difference between a Deployment and a StatefulSet, and when MongoDB in
> `mcp-gateway-registry` would need a StatefulSet vs a plain Deployment + PVC for local learning.

**Next:** [Phase 6 — Networking & Ingress](06-networking-ingress.md)
