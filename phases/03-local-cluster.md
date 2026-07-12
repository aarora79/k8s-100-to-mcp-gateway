# Phase 3 — Stand up a local cluster on your Mac Studio

**Goal:** get a real, running Kubernetes cluster on your machine and talk to it with `kubectl`.

**Time:** ~30 minutes.

---

## Choosing your local Kubernetes

For learning on a Mac, the common options are:

| Tool | What it is | Best for |
|------|-----------|----------|
| **Docker Desktop Kubernetes** | A one-toggle single-node cluster inside Docker Desktop | **Beginners — our default for Phases 3–7** |
| OrbStack | A fast, lightweight Docker/K8s runtime for Mac | A lighter alternative to Docker Desktop |
| kind | "Kubernetes in Docker" — clusters as containers, scriptable | Realistic/multi-node; we use it in Phase 8+ |
| minikube | A local single/multi-node cluster VM | Popular in tutorials; fine but more moving parts |

**We start with Docker Desktop's built-in Kubernetes** because it's the fewest moving parts while
you learn concepts. In Phase 8 we add `kind` for a more production-like feel. You can absolutely
do the whole thing on Docker Desktop if you prefer.

---

## Step 1 — Turn on Kubernetes in Docker Desktop

1. Open **Docker Desktop**.
2. Click the **gear (Settings)** icon.
3. Go to **Kubernetes** in the sidebar.
4. Check **Enable Kubernetes**, then **Apply & Restart**.
5. Wait a few minutes — it downloads and starts the control-plane containers. The Kubernetes
   indicator (bottom-left of Docker Desktop) turns **green** when ready.

---

## Step 2 — Point kubectl at the right cluster

`kubectl` can talk to many clusters; it uses a *context* to know which one. Docker Desktop adds a
context called `docker-desktop`.

```bash
kubectl config get-contexts          # list available contexts
kubectl config use-context docker-desktop
kubectl config current-context       # should print: docker-desktop
```

> **Why this matters:** the #1 beginner confusion later is running a command against the wrong
> cluster. Always know your current context. The **Kubernetes** VS Code extension shows it in the
> sidebar too.

---

## Step 3 — Verify the cluster is alive

```bash
kubectl cluster-info                 # shows the API server URL
kubectl get nodes                    # should list one node, STATUS=Ready
kubectl get pods -A                  # -A = all namespaces; shows system pods running
```

If `kubectl get nodes` shows one node in `Ready` state, **you have a working Kubernetes cluster.**
The pods you see under `kube-system` are Kubernetes running *itself* — that's the control plane
from Phase 2, made real.

---

## Step 4 — Learn to read `kubectl` output

Try these read-only commands to get oriented. They can't break anything:

```bash
kubectl get namespaces               # the virtual partitions of the cluster
kubectl get all                      # everything in the default namespace (empty for now)
kubectl describe node docker-desktop # detailed status of your one node
kubectl api-resources | head -30     # every kind of object this cluster understands
```

The `get` / `describe` pair is your bread and butter:

- `kubectl get <thing>` — a quick list.
- `kubectl describe <thing> <name>` — the full detail, including an **Events** section at the
  bottom that is gold for debugging.

---

## Step 5 — Make kubectl pleasant (optional but recommended)

Add an alias and enable tab-completion in your `~/.zshrc`:

```bash
echo 'alias k=kubectl' >> ~/.zshrc
echo 'source <(kubectl completion zsh)' >> ~/.zshrc
echo 'complete -F __start_kubectl k' >> ~/.zshrc
source ~/.zshrc
```

Now `k get pods` works and you get tab-completion on resource names.

---

## Checkpoint ✅

```bash
kubectl config current-context       # docker-desktop
kubectl get nodes                    # one node, Ready
kubectl get pods -A                  # system pods Running
```

All good? You have a cluster. Time to actually deploy something.

---

## Ask Claude Code

> My `kubectl get nodes` shows the node as `NotReady` (or Docker Desktop's Kubernetes won't turn
> green). Here's `kubectl describe node` and `kubectl get pods -A`: [paste]. Walk me through
> diagnosing this as a beginner.

**Next:** [Phase 4 — Your first deployment](04-first-deployment.md)
