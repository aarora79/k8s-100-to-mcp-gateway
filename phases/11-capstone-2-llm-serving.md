# Phase 11 — Capstone 2: serve an open-source LLM (Gemma) on Kubernetes

**Goal:** run a real model-serving workload — Google's **Gemma**, served over an
OpenAI-compatible API — on your local cluster, and learn what makes LLM serving *different* from
serving the stateless web apps you've deployed so far: multi-gigabyte model artifacts, heavy
memory appetites, slow cold starts, and GPU scheduling — plus **observability**: watching your
server's real resource usage in Prometheus + Grafana. Then map the whole thing onto **vLLM on
cloud GPUs**, where Phase 10's scheduling vocabulary becomes load-bearing.

**Time:** one sitting for the local track (~2.5–3 hours, much of it downloads); the cloud track
is reading + a design exercise.

**Prerequisite:** Phases 8 and 10 (you'll reuse the kind cluster, PVCs, probes, requests/limits,
and affinity).

---

## Before you start — 60 seconds of retrieval

1. What object keeps data alive across Pod replacements, and where did you last prove it?
   *(Phases 5 and 8 — today it saves you a multi-GB re-download)*
2. What's a readiness probe for, and what happens to a Service's traffic while a Pod isn't
   ready? *(Phase 9, Step 4)*
3. How do you steer a Pod onto a specific class of node, and how does a node *refuse* Pods that
   don't belong? *(Phase 10 — the cloud track runs on this)*

---

## Step 0 — An honest tooling decision (read this, it's the lesson)

The industry-standard open-source LLM server is **vLLM** — it's what you'd run in production on
NVIDIA GPUs. But your Mac Studio's GPU is Apple silicon, and containers on macOS run inside a
Linux VM that **cannot see the Apple GPU at all**. vLLM's official images are x86 + CUDA; there
is no useful way to run it on this machine.

So we split the capstone the way a real team would:

- **Local track (hands-on):** serve Gemma with **Ollama** — a multi-arch model server with an
  OpenAI-compatible API that runs fine on CPU. Every Kubernetes pattern (PVC for weights, probes
  for slow starts, memory sizing, the API shape) is identical to production.
- **Cloud track (design exercise):** the *same* architecture with vLLM on GPU nodes, where the
  only new ingredients are `nvidia.com/gpu` resources and Phase 10's scheduling tools.

Choosing the right tool for the hardware **is** a Kubernetes skill. This phase teaches the
patterns; the server binary is swappable.

---

## Step 1 — Namespace, storage, and the server

Model weights are the defining problem of LLM serving: the Gemma 4 build we'll use is **7.2GB**,
and real deployments are tens of GB. You do **not** want that re-downloaded every time a Pod is
replaced — this is Phase 8's "the Pod came back, the data didn't" lesson with real money on the
line. So: PVC first.

> **Give the VM room first.** Your cluster's node lives inside Docker's Linux VM, and Docker
> Desktop's default memory cap (often 8GB) is too small for a 7GB model plus everything else.
> Docker Desktop → Settings → Resources → set **Memory to 16GB+** (your Mac Studio won't
> notice), Apply & Restart. Skipping this is the #1 cause of the `OOMKilled` you'll otherwise
> meet ahead of schedule.

`manifests/llm/ollama.yaml`:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: llm
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: models
  namespace: llm
spec:
  accessModes: ["ReadWriteOnce"]
  resources:
    requests:
      storage: 25Gi                # a 7GB model + a spare + headroom
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ollama
  namespace: llm
spec:
  replicas: 1
  selector: { matchLabels: { app: ollama } }
  template:
    metadata: { labels: { app: ollama } }
    spec:
      containers:
        - name: ollama
          image: ollama/ollama
          ports: [{ containerPort: 11434 }]
          volumeMounts:
            - name: models
              mountPath: /root/.ollama    # where Ollama keeps downloaded weights
          resources:
            requests:
              cpu: "2"
              memory: 6Gi
            limits:
              memory: 12Gi                # model + KV cache live in RAM on CPU serving
          readinessProbe:
            httpGet: { path: /, port: 11434 }
            initialDelaySeconds: 5
            periodSeconds: 5
      volumes:
        - name: models
          persistentVolumeClaim:
            claimName: models
---
apiVersion: v1
kind: Service
metadata:
  name: ollama
  namespace: llm
spec:
  selector: { app: ollama }
  ports: [{ port: 80, targetPort: 11434 }]
```

Notice what your Phase 10 knowledge is doing in that file: the **memory limit is doing real
work** (an LLM that outgrows RAM gets `OOMKilled` — better a clean kill than a frozen node), and
`replicas: 1` is deliberate — each replica would need its *own* copy of the model in memory, so
"just scale it up" has a very different price tag than it did for nginx.

```bash
kubectl apply -f manifests/llm/ollama.yaml
kubectl get pods -n llm --watch          # wait for 1/1 Ready
```

---

## Step 2 — Pull the model (and watch where it lands)

The server is running but empty. Pull Gemma into it — and recognize the command shape from
Phase 4's debugging toolkit:

```bash
kubectl exec -n llm deploy/ollama -- ollama pull gemma4:e2b    # 7.2GB — go make coffee
kubectl exec -n llm deploy/ollama -- ollama list               # it's there
#            └─ namespace  └─ picks a pod from the "ollama" Deployment; after the "--",
#               "ollama pull/list" is Ollama's OWN CLI running inside the container
#               (the deployment and the binary just happen to share a name)
```

(`gemma4:e2b` is Gemma 4's smallest build — "e2b" means roughly *effective 2B parameters*
active per token, so it's surprisingly quick on CPU despite the 7GB download. `gemma4:e4b`
(9.6GB) answers better if you have the patience. On a slow connection or a tighter machine,
`gemma3:1b` is only ~800MB and works identically for every step below — just swap the model
name in the requests.)

Those weights went through the container's `/root/.ollama` — which is your **PVC**. Prove the
lesson that Phase 8 taught the hard way, now with 7GB on the line:

```bash
kubectl delete pod -n llm -l app=ollama       # kill the server
kubectl get pods -n llm --watch               # replacement comes up...
kubectl exec -n llm deploy/ollama -- ollama list   # ...and the model is STILL THERE. No re-pull.
```

---

## Step 3 — Talk to it like it's OpenAI

Ollama (like vLLM) exposes the **OpenAI-compatible chat API** — the de-facto standard interface
of the LLM world. This is the punchline of the capstone: any app that can call OpenAI can call
*your cluster* by changing one URL.

```bash
kubectl port-forward -n llm svc/ollama 8080:80
```

In another terminal:

```bash
curl http://localhost:8080/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "gemma4:e2b",
    "messages": [
      {"role": "user", "content": "In two sentences: why do Kubernetes Pods need Services in front of them?"}
    ]
  }'
```

> **Wait — why `localhost` and not the Service name?** Because *you* are outside the cluster.
> The name `ollama` (or `ollama.llm.svc.cluster.local`) is resolved by the cluster's internal
> DNS (CoreDNS, Phase 2), and only Pods use that DNS — your Mac's terminal asks your Mac's
> resolver, which has never heard of it. `port-forward` is the workaround: a tunnel from your
> laptop's port 8080 to the Service, with `localhost` as the entrance. Prove the Service name
> *does* work where it's supposed to — from inside the cluster:
>
> ```bash
> kubectl run tester -n llm --rm -it --restart=Never --image=curlimages/curl -- \
>   curl -s http://ollama/v1/chat/completions \
>   -H "Content-Type: application/json" \
>   -d '{"model":"gemma4:e2b","messages":[{"role":"user","content":"Say hi in five words."}]}'
> ```
>
> Same API, no port-forward, plain `http://ollama` — because this curl ran *in a Pod in the
> same namespace*. That's the boundary: **Service DNS inside, Ingress or port-forward from
> outside.** It's also exactly how an app of yours would call this model server in production.

A JSON response with `choices[0].message.content` comes back — from a model running in a Pod you
declared. (Grade its answer yourself: you literally know this one.) The first request after a
Pod starts is slow — that's the model loading from the PVC into memory. *That* is why real LLM
deployments obsess over probes and why vLLM Pods use generous `startupProbe`s: cold starts are
measured in minutes, and you don't want the Service routing to a replica that's still loading.

Optionally add an Ingress (`llm.localhost` → `ollama` Service, port 80) — you've done it twice;
no YAML provided. That's not laziness, it's Phase 4-through-8 paying out.

---

## Step 4 — Watch it run: Prometheus & Grafana

A model server you can't observe is a model server you can't size, scale, or debug. The standard
Kubernetes monitoring stack is **Prometheus** (collects and stores metrics) + **Grafana**
(dashboards over them), and the standard way to install both is — Phase 7 skills — one Helm
chart:

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm install monitoring prometheus-community/kube-prometheus-stack \
  --namespace monitoring --create-namespace
kubectl get pods -n monitoring --watch     # several components; wait for Running (1-2 min)
```

That one release installed Prometheus, Grafana, exporters on every node, and a pile of ready-made
dashboards. Grafana's admin password lives in — Phase 5 skills — a Secret:

```bash
kubectl get secret -n monitoring monitoring-grafana \
  -o jsonpath='{.data.admin-password}' | base64 -d; echo
kubectl port-forward -n monitoring svc/monitoring-grafana 3000:80
```

Open <http://localhost:3000>, log in as `admin` with that password, and go to **Dashboards →
Kubernetes / Compute Resources / Namespace (Pods)**, then select the `llm` namespace. There's
your Ollama Pod's real CPU and memory.

Now make it move. Send a chat request (Step 3) and watch the dashboard while the model thinks:
CPU pins to your request's worth of cores for the whole generation, and memory shows the model
sitting resident. Restart the Deployment and you can *see* the cold start — memory climbing as
7GB of weights load. This is how you'd answer "what should `requests` be?" in the real world:
not by guessing, but by reading the graph. (Loop back to Phase 10: is your HPA's CPU target
even sensible for a workload that pins CPU for many seconds per request?)

> **What about LLM-specific metrics — tokens/sec, time-to-first-token, queue depth?** Honest
> answer: Ollama doesn't expose a Prometheus `/metrics` endpoint (verified — it 404s), so
> those numbers aren't available here without third-party exporters. **vLLM does, natively** —
> `/metrics` on the same port serves rich serving metrics, and Prometheus scrapes it with one
> annotation. Add that to the cloud track's advantages, and note the pattern: *infrastructure*
> metrics (CPU/RAM, this section) come for free everywhere; *application* metrics are the app's
> job to emit.

When you're done exploring, the whole stack uninstalls as cleanly as it installed:
`helm uninstall monitoring -n monitoring` (or keep it — it's genuinely useful).

---

## Step 5 — The cloud track: the same thing with vLLM on GPUs (design exercise)

Here's the entire delta between what you just built and a production GPU deployment. **The
architecture does not change** — namespace, PVC, Deployment, Service, probes all stay. What
changes is the container and the scheduling:

```yaml
# The container: vLLM instead of Ollama
        - name: vllm
          image: vllm/vllm-openai:latest
          args: ["--model", "google/gemma-4-12B-it"]    # pulls from Hugging Face at startup
          ports: [{ containerPort: 8000 }]
          env:
            - name: HUGGING_FACE_HUB_TOKEN               # Gemma requires accepting a license
              valueFrom:
                secretKeyRef: { name: hf-token, key: token }   # Phase 5: this is what Secrets are FOR
          resources:
            limits:
              nvidia.com/gpu: 1        # <- a GPU is just another resource, like cpu/memory
          volumeMounts:
            - name: models
              mountPath: /root/.cache/huggingface        # same PVC pattern, different path
          startupProbe:                                  # cold starts take minutes
            httpGet: { path: /health, port: 8000 }
            failureThreshold: 60
            periodSeconds: 10
# The scheduling: Phase 10, verbatim
      tolerations:
        - { key: nvidia.com/gpu, operator: Exists, effect: NoSchedule }
      nodeSelector:
        nvidia.com/gpu.present: "true"
```

Read that bottom block again — **taints and tolerations and node selectors are exactly how the
entire GPU economy runs on Kubernetes.** GPU nodes are tainted so ordinary Pods can't squat on a
$30/hour machine; LLM Pods tolerate the taint and select for the GPU. And when no GPU node
exists at all? The Pod goes `Pending` — and **Karpenter** (Phase 10) reads its requirements,
provisions a GPU node in minutes, and later removes it when the Pod is gone. On **EKS Auto
Mode**, AWS runs that loop for you. Scale-to-zero GPU serving is these primitives composed —
nothing you haven't now touched.

Your design exercise (paper or Claude Code, no cluster needed): write the complete
`vllm.yaml` for an EKS cluster — Deployment, Service, Secret reference, PVC, probes, and the
scheduling block — and annotate every line with which phase taught it.

---

## Challenge — no training wheels ✋

1. Add a second model (`ollama pull` another small one, e.g. `qwen3:0.6b`) and confirm the
   *same* server serves both — the `model` field in the request selects one. What does that
   imply about how Ollama differs from vLLM's one-model-per-server default?
2. Break it on purpose, Phase 10 edition: lower the Deployment's memory **limit** to `1Gi`,
   re-apply, and send a chat request. Watch the Pod get `OOMKilled` and restart
   (`kubectl get pods -n llm --watch`, then `describe` — find the reason and the restart count).
   If your monitoring stack is still up, find the moment of death in Grafana — memory climbing
   into the limit, then the cliff. Restore the limit.
3. From memory: why is `replicas: 2` a much bigger decision for this Deployment than it was for
   nginx in Phase 4? (Two reasons: one about RAM, one about the PVC's `ReadWriteOnce`.)

---

## Checkpoint ✅

- [ ] Gemma answers chat requests through a Service on your cluster.
- [ ] You proved the model survives Pod replacement because of the PVC — no re-download.
- [ ] You caused and diagnosed an `OOMKilled` restart, and can explain why memory limits matter
      more for LLMs than for web servers.
- [ ] You watched the model server's CPU and memory live in Grafana during a generation, and can
      say which metrics come free from the cluster vs which the app must emit.
- [ ] You can explain why `curl http://ollama` works from a Pod but not from your Mac's
      terminal — and what port-forward and Ingress each do about that.
- [ ] You can sketch the vLLM/GPU manifest and explain the toleration + nodeSelector pair
      without looking.
- [ ] You can say in one sentence what Karpenter would do for this workload on EKS.

---

## Clean up

```bash
kubectl delete namespace llm      # removes everything in it, PVC included — one tidy box (Phase 2!)
```

---

## Ask Claude Code

**Teach-back:**

> Quiz me one question at a time on what makes LLM serving different from stateless web serving
> on Kubernetes: model storage, cold starts and probes, memory sizing, replica economics, and
> GPU scheduling. Push until I can explain each without hand-waving.

**Go deeper:**

> Here's my working `ollama.yaml`. Help me write the production vLLM-on-EKS version for
> `google/gemma-4-12B-it`: real GPU instance choice, Karpenter NodePool for scale-to-zero, an
> HPA based on a sensible metric (is CPU% even right for an LLM server? what else could I use?),
> and a PodDisruptionBudget. Explain the trade-offs like I'm going to be quizzed.

---

**Two capstones done.** You've deployed a multi-service web application *and* an AI model server
— the two workload shapes that dominate modern infrastructure. The vocabulary you now have is
the same one used in production clusters serving millions of requests.

**Next (reading, not doing):** [Phase 12 — the expert concept map](12-expert-concepts.md) —
the 300–400 level, explained in the plain English you're used to.
