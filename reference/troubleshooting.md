# Troubleshooting — "it's broken, now what"

Kubernetes breaks in a small number of recognizable ways. Learn to read the *status* and the
*Events*, and 90% of problems name themselves. **Always start here:**

```bash
kubectl get pods -n <namespace>            # read the STATUS column
kubectl describe pod <name> -n <namespace> # read the EVENTS at the bottom
kubectl logs <name> -n <namespace>         # read what the app said
```

---

## Pod statuses and what they mean

### `Pending`
The Pod can't be scheduled yet. Common causes:
- **Not enough resources** — no node has the CPU/RAM it requested. `describe` says "Insufficient
  cpu/memory". Lower `requests`, or free resources.
- **PVC not bound** — it's waiting on storage. Check `kubectl get pvc`.
- **No node matches** — node selectors/affinity can't be satisfied (rare locally).

### `ContainerCreating` (stuck)
Usually pulling an image or mounting a volume. If it lingers, `describe` shows why (image pull in
progress, volume mount failing).

### `ImagePullBackOff` / `ErrImagePull`
Kubernetes can't get the image. Causes:
- **Typo in the image name/tag.** Check spelling and tag exists.
- **Private image, no credentials.** Add an imagePullSecret.
- **Locally-built image not in the cluster.** On `kind`, you must
  `kind load docker-image <img> --name <cluster>` and set `imagePullPolicy: IfNotPresent`.
  (Phase 9, Step 3.)

### `CrashLoopBackOff`
The container starts, crashes, and Kubernetes keeps restarting it with increasing delay. This is
an *application* problem, not a scheduling one. Do:
```bash
kubectl logs <name> --previous          # logs from the crashed instance — the real clue
kubectl describe pod <name>             # exit code + last state
```
Common causes: missing env var/config, can't reach a dependency (e.g. database not up yet), bad
command, port already in use, migration failing. Compare against the working Compose version.

### `Running` but the app doesn't work
The container is up but the app is unhealthy or misconfigured.
- Check `kubectl logs`.
- Check readiness: `kubectl get pods` shows `READY 0/1` if the readiness probe fails.
- `kubectl describe pod` shows probe failures in Events.

### `Terminating` (stuck)
Usually a finalizer or a slow shutdown. Rarely needs `--force --grace-period=0`, but avoid that
unless you understand it.

---

## Networking problems

### "I can't reach my app in the browser"
Work outward from the Pod:
1. **Pod healthy?** `kubectl get pods` → `Running` and `READY 1/1`.
2. **Can I reach the Pod directly?** `kubectl port-forward pod/<name> 8080:<port>` then curl
   `localhost:8080`. If this works, the app is fine and the problem is Service/Ingress.
3. **Service correct?** `kubectl get endpoints <svc>` — if it lists no addresses, the Service's
   `selector` doesn't match any Pod labels. This is the most common Service bug.
4. **Ingress correct?** `kubectl describe ingress <name>`; confirm the Ingress controller Pod is
   Running in its namespace, and the host resolves (use `*.localhost` which points to 127.0.0.1).

### "One service can't reach another"
- Use the **Service name**, not a Pod IP. Across namespaces use
  `<svc>.<namespace>.svc.cluster.local`.
- Confirm the target Service has endpoints (`kubectl get endpoints <svc>`).
- Test from a throwaway pod: `kubectl run t --rm -it --image=busybox -- sh` then
  `wget -qO- http://<svc>`.

---

## Config & storage problems

- **Env var not showing up** — did you `envFrom` the right ConfigMap/Secret, in the right
  namespace? ConfigMaps/Secrets are namespaced. Restart the Pod after changing them.
- **PVC stuck `Pending`** — no StorageClass or none default. `kubectl get storageclass`. On Docker
  Desktop/kind there's a default; elsewhere you may need to name one.
- **Data disappeared after restart** — the Pod wasn't actually using the PVC, or wrote outside the
  `mountPath`. Confirm `volumeMounts.mountPath` matches where the app writes.

---

## "Which cluster am I even on?"
A shocking number of "it's not working" moments are *right command, wrong cluster*.
```bash
kubectl config current-context
```
Make this the first thing you check when confused.

---

## When to hand it to Claude Code

Paste the *evidence*, not just "it's broken":
> My pod `<name>` in namespace `<ns>` is in `<status>`. Here's `kubectl describe pod` and
> `kubectl logs --previous`: [paste both]. It works under Docker Compose. What's the Kubernetes
> version missing, and what's the exact fix?

Giving it `describe` + `logs` + "works in Compose" is usually enough for a precise answer.
