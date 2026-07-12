# Phase 4 — Your first deployment: Pods, Deployments, Services

**Goal:** deploy an app to your cluster, expose it, scale it, break it, and heal it — the core
loop you'll repeat forever.

**Time:** ~1.5–2 hours.

---

## Before you start — 60 seconds of retrieval

Answer from memory, out loud or on paper (checking yourself later beats re-reading):

1. What's the difference between an **image** and a **container**? *(Phase 1)*
2. What does **declarative** mean, and *who* keeps reality matching your declaration? *(Phase 2)*
3. Which command tells you **which cluster** kubectl is currently pointed at? *(Phase 3)*

Answers are in the linked phases and the [glossary](../reference/glossary.md). If any felt shaky,
30 seconds of review now pays off all phase.

---

## Warm-up: the imperative way (to see the objects)

Let's deploy nginx the quick way first, just to watch the objects appear:

```bash
kubectl create deployment web --image=nginx
kubectl get deployments          # 1 deployment, becoming ready
kubectl get replicasets          # the ReplicaSet the deployment created
kubectl get pods                 # the Pod the ReplicaSet created
```

See the chain from Phase 2 made real: **Deployment → ReplicaSet → Pod**. Watch it live:

```bash
kubectl get pods --watch         # Ctrl-C to stop
```

### Reach the app temporarily with port-forward

The Pod is running but not reachable from your browser yet. The quickest way to poke it:

```bash
kubectl port-forward deployment/web 8080:80
# now open http://localhost:8080  (Ctrl-C to stop the forward)
```

`port-forward` is a debugging tool — it tunnels one local port to a Pod. It's not how real traffic
flows (that's Services + Ingress, Phase 6), but it's invaluable for testing.

---

## Self-healing: the "aha" moment

Kubernetes keeps your declared state true. Prove it:

```bash
kubectl get pods                                  # note the pod name, e.g. web-xxxx
kubectl delete pod <that-pod-name>                # delete it
kubectl get pods                                  # a NEW pod is already being created!
```

You said (via the Deployment) "there should be 1 replica." You deleted it. The controller noticed
reality ≠ desire and fixed it. **You did not run a command to recreate it — the system did.** That
is the heart of Kubernetes.

Clean up the warm-up:

```bash
kubectl delete deployment web
```

---

## The real way: declarative YAML

From here on we write YAML, because that's how real work is done (and how the capstone will be
built). Create `manifests/web-deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
  labels:
    app: web
spec:
  replicas: 3                    # we want THREE copies
  selector:
    matchLabels:
      app: web                   # this Deployment manages Pods with label app=web
  template:                      # the Pod blueprint
    metadata:
      labels:
        app: web                 # Pods get this label (must match selector above)
    spec:
      containers:
        - name: nginx
          image: nginx:1.27
          ports:
            - containerPort: 80
```

Apply it and watch three pods appear:

```bash
kubectl apply -f manifests/web-deployment.yaml
kubectl get pods -l app=web      # -l filters by label; you should see 3
```

### Understand the YAML (every beginner's stumbling block)

- `apiVersion` / `kind` — *what kind of object* this is. Deployments live in `apps/v1`.
- `metadata.name` / `labels` — its name and key/value tags.
- `spec.replicas` — desired copies.
- `spec.selector.matchLabels` — **which Pods this Deployment owns.** It must match…
- `spec.template.metadata.labels` — …the labels stamped on the Pods it creates.
- `spec.template.spec.containers` — the actual container(s): name, image, ports.

That `selector` ↔ `template.labels` match trips up everyone once. If they don't match, `apply`
fails with a clear error. The **YAML VS Code extension** will also flag mistakes as you type.

---

## Expose it with a Service

Three Pods, three changing IPs. A Service gives them one stable address. Add
`manifests/web-service.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web
spec:
  selector:
    app: web                     # send traffic to Pods labeled app=web
  ports:
    - port: 80                   # the Service's port
      targetPort: 80             # the container's port
  type: ClusterIP                # reachable inside the cluster (default)
```

```bash
kubectl apply -f manifests/web-service.yaml
kubectl get service web
kubectl port-forward service/web 8080:80    # open http://localhost:8080
```

Now traffic to the Service is load-balanced across all three Pods. Delete a Pod and the Service
keeps working — it just routes to the survivors while a replacement spins up.

---

## Scale and update

```bash
kubectl scale deployment web --replicas=5     # scale up
kubectl get pods -l app=web                    # now 5

# Roll out a new image version (a "rolling update"):
kubectl set image deployment/web nginx=nginx:1.28
kubectl rollout status deployment/web          # watch it replace pods gradually
kubectl rollout history deployment/web         # see revisions
kubectl rollout undo deployment/web            # roll back if needed
```

Notice the update replaced Pods **gradually**, keeping the app available throughout. That's a
zero-downtime deploy — one of the big reasons Kubernetes exists.

> **What actually happened under the hood (the ReplicaSet's real job).** Remember from Phase 2:
> a ReplicaSet can only keep copies of *one fixed Pod template* running — it's dumb about
> versions. So the Deployment didn't modify your old ReplicaSet; it created a **new** ReplicaSet
> for the new image and gradually scaled it up while scaling the old one down to zero:
>
> ```text
> Deployment "web"        "run nginx:1.28, 5 replicas, update safely"
>    ├── ReplicaSet (new)   "keep exactly 5 Pods matching nginx:1.28"
>    │     ├── Pod   ← replica 1
>    │     ├── Pod   ← ... up to replica 5
>    └── ReplicaSet (old)   "keep exactly 0" (nginx:1.27 — kept for rollback)
> ```
>
> That's also why `kubectl rollout undo` is instant: rollback just means scaling the old
> ReplicaSet back up. Prove it to yourself:
>
> ```bash
> kubectl get replicasets -l app=web    # two ReplicaSets: new one at 5, old one at 0
> ```

---

## Inspect and debug (learn these now, use them forever)

```bash
kubectl describe deployment web        # full status + Events at the bottom
kubectl describe pod <pod-name>        # why a pod is/isn't running
kubectl logs <pod-name>                # a container's stdout/stderr
kubectl logs -f <pod-name>             # follow logs live
kubectl exec -it <pod-name> -- sh      # shell into a running container
kubectl get events --sort-by=.metadata.creationTimestamp   # recent cluster events
```

When something is wrong, the order is almost always: `get pods` → `describe pod` (read Events) →
`logs`. Memorize that flow.

---

## Break it on purpose (the two failures you'll see forever)

Reading about failures teaches little; *causing* them safely teaches a lot. You'll meet these two
statuses more than any others — meet them now, on your terms, with the debugging flow you just
learned.

### Failure 1: `ImagePullBackOff` — the image can't be fetched

With your `web` deployment still running, roll out an image tag that doesn't exist:

```bash
kubectl set image deployment/web nginx=nginx:no-such-tag
kubectl get pods -l app=web        # watch: a NEW pod shows ImagePullBackOff / ErrImagePull
```

Two lessons at once here. First, diagnose it:

```bash
kubectl describe pod <the-failing-pod>    # Events say: "manifest unknown" / pull failed
```

Second — notice your **old pods are still Running and serving**. The rolling update refused to
kill healthy pods before their replacements were ready. Kubernetes protected you from your own
typo. Now recover:

```bash
kubectl rollout undo deployment/web
kubectl get pods -l app=web               # all healthy again
```

### Failure 2: `CrashLoopBackOff` — the container starts, then dies

```bash
kubectl run crasher --image=busybox -- sh -c "echo BOOM: missing config; exit 1"
kubectl get pods crasher --watch          # Running → Error → CrashLoopBackOff (Ctrl-C to stop)
```

Kubernetes keeps restarting it with growing delays (that's the "BackOff"). Diagnose:

```bash
kubectl logs crasher                      # -> BOOM: missing config  (the app's last words)
kubectl describe pod crasher              # Last State: Terminated, Exit Code: 1, Restart Count climbing
kubectl delete pod crasher
```

The key distinction to internalize: **ImagePullBackOff = Kubernetes' problem getting your image**
(typo, missing tag, private registry). **CrashLoopBackOff = your app's problem after starting**
(missing config, can't reach a dependency, bad command) — the clue is almost always in `logs`.
This distinction will earn its keep in the capstone.

---

## Challenge — no training wheels ✋

Everything above was handed to you. Now prove you own it. **From a blank file, without looking
at the YAML earlier in this phase:**

1. Deploy Apache httpd (`image: httpd:2.4`, container port 80) with **2 replicas**, in a file
   `manifests/challenge-httpd.yaml`.
2. Put a Service in front of it and reach it at `http://localhost:8081` via port-forward.
3. Delete one pod and confirm it self-heals.
4. Clean up with a single `kubectl delete -f` command.

Struggle for ten minutes before asking Claude Code for hints — retrieval with effort is where
the learning happens. (Type it; the YAML extension will help you with field names.)

---

## Clean up

```bash
kubectl delete -f manifests/web-service.yaml
kubectl delete -f manifests/web-deployment.yaml
```

Deleting the same file you applied is the tidy way to remove exactly what you created.

---

## Checkpoint ✅

- [ ] You deployed an app from a YAML Deployment with 3 replicas.
- [ ] You created a Service and reached the app through it.
- [ ] You scaled, did a rolling update, and rolled back.
- [ ] You deleted a Pod and watched Kubernetes recreate it.
- [ ] You caused — and diagnosed — an `ImagePullBackOff` and a `CrashLoopBackOff` on purpose.
- [ ] You completed the httpd challenge from a blank file.
- [ ] You can name the `get pods → describe → logs` debugging flow.

---

## Ask Claude Code

**Teach-back** (do this one — it's the best retention tool you have):

> Here are my `web-deployment.yaml` and `web-service.yaml`: [paste]. Quiz me on them line by
> line — one question at a time, asking what each field does and what would break if I removed
> or changed it. Tell me when I'm right, and correct my mental model when I'm fuzzy.

**Debugging practice:**

> Here's my YAML and the output of `kubectl get pods -l app=web` showing a pod stuck in
> `ImagePullBackOff` (or `CrashLoopBackOff`): [paste]. Explain what that status means and how I
> diagnose it step by step.

**Next:** [Phase 5 — Config, secrets, and storage](05-config-and-storage.md)
