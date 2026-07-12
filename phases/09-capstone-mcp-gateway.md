# Phase 9 — Capstone: run `mcp-gateway-registry` on Kubernetes

**Goal:** take the real `mcp-gateway-registry` application — which ships as Docker Compose — and
run it on your local Kubernetes cluster. This is where everything you learned becomes one working
system.

**Time:** this is a multi-session project, not an afternoon. Expect a few sittings, and expect to
lean on Claude Code. That's the intended experience.

**Prerequisite:** Phases 4–8 completed and comfortable. Seriously — this phase assumes all of it.

---

## Step 0 — Understand what you're deploying

`mcp-gateway-registry` (from `agentic-community/mcp-gateway-registry`) is an enterprise MCP Gateway
& Registry. Its local deployment is **Docker Compose based**, and it's made of roughly these
pieces:

| Component | What it is | Kubernetes shape it becomes |
|-----------|-----------|-----------------------------|
| Registry API | FastAPI service — catalog, semantic search, UI (port ~7860) | Deployment + Service (+ Ingress) |
| Auth server | FastAPI — OAuth 2.0/OIDC | Deployment + Service |
| MCP Gateway | nginx + `mcpgw` reverse proxy / router (ports 80/443, 8000+) | Deployment + Service (or Ingress) |
| MongoDB | Persistent store with vector search | Deployment/StatefulSet + Service + **PVC** |
| Keycloak (optional) | Local identity provider | Deployment + Service (skip at first) |
| `.env` config | DB creds, IdP settings, feature flags | **ConfigMap** (non-secret) + **Secret** (creds) |

System needs (from the project docs): **4+ CPU cores, 8GB+ RAM**, Python 3.14+ *inside the images*
(you don't install Python — the containers carry it), ports 7860 / 8000+ / 80 / 443. Your Mac
Studio has plenty of headroom.

> **First, run it the supported way (Compose) at least once.** Before porting anything, get the
> app working with its own instructions so you know what "correct" looks like. Then port to K8s.

```bash
git clone https://github.com/agentic-community/mcp-gateway-registry.git
cd mcp-gateway-registry
cp .env.example .env          # then edit .env per their README
./build_and_run.sh --prebuilt # their supported one-command start
open http://localhost:7860    # confirm the app works under Compose first
```

Keep this working Compose deployment as your **reference oracle**: when the K8s version misbehaves,
compare against how Compose ran it.

---

## Step 1 — Read the Compose file like a translation source

Open `docker-compose.yml` in the cloned repo. Compose and Kubernetes map almost concept-for-concept:

| Docker Compose | Kubernetes equivalent |
|----------------|----------------------|
| a `service:` | a Deployment + a Service |
| `image:` | container `image:` in the Deployment |
| `build:` | build the image first (`docker build`), push/load it, then reference it |
| `ports:` `"7860:7860"` | Service `port`/`targetPort` (+ Ingress for browser access) |
| `environment:` / `env_file: .env` | ConfigMap (+ Secret for sensitive keys) via `envFrom` |
| `volumes:` (named) | PersistentVolumeClaim + `volumeMounts` |
| `depends_on:` | *no direct equivalent* — use readiness probes + retries (see Step 4) |
| service name as hostname | Kubernetes Service name (DNS) — the *same idea*, so most internal URLs carry over unchanged |

The single most useful realization: **Compose's "reach another service by its service name" is
exactly Kubernetes DNS.** If the Registry API talks to `mongodb:27017` in Compose, it will talk to
`mongodb:27017` in Kubernetes too — as long as you name the MongoDB Service `mongodb`.

### Let Claude Code do the first-pass translation

This is a perfect Claude Code task. In VS Code, with both this repo and the cloned
`mcp-gateway-registry` open:

> I have the `docker-compose.yml` from `mcp-gateway-registry` open. Convert it into Kubernetes
> manifests for a local `kind`/Docker-Desktop cluster in a namespace called `mcp-gateway`.
> For each Compose service produce a Deployment + Service. Turn `env_file: .env` into a ConfigMap
> plus a Secret (put obviously sensitive keys in the Secret). Give MongoDB a PVC. Don't invent
> values — where a value comes from `.env`, reference the ConfigMap/Secret key and leave me a
> `# TODO: set in secret` note. Explain each file after you write it.

Then **you review and understand every line** — don't just apply it. The point of Phases 4–8 was
to make you able to read what it generates. Save the results under `manifests/mcp-gateway/`.

---

## Step 2 — A recommended porting order (don't do it all at once)

Bring the stack up **one tier at a time**, verifying each before adding the next. This is the
difference between a calm afternoon and a frustrating one.

1. **Namespace + config**
   ```bash
   kubectl create namespace mcp-gateway
   kubectl apply -f manifests/mcp-gateway/configmap.yaml
   kubectl apply -f manifests/mcp-gateway/secret.yaml     # keep real secrets OUT of git
   ```
2. **MongoDB first** (everything depends on data). Deploy it, give it a PVC, and confirm it's
   `Running` and reachable by the name `mongodb` before moving on. You can even reuse the Helm
   MongoDB from Phase 7 instead of hand-writing it.
   ```bash
   kubectl apply -f manifests/mcp-gateway/mongodb.yaml -n mcp-gateway
   kubectl get pods -n mcp-gateway --watch
   ```
3. **Auth server** — deploy, check logs, confirm it starts and finds Mongo.
4. **Registry API** — deploy, check it connects to Mongo and Auth.
5. **MCP Gateway (nginx)** — deploy last; it fronts the others.
6. **Expose the UI** — port-forward first (`kubectl port-forward -n mcp-gateway svc/registry 7860:7860`),
   confirm `http://localhost:7860` works, *then* add an Ingress for a nice hostname.

After each step: `kubectl get pods -n mcp-gateway`, then `describe`/`logs` on anything not
`Running`. Fix before proceeding.

---

## Step 3 — Handling images that are built, not pulled

Some components may be built from source in Compose (`build:`), not pulled from a registry. On a
local cluster you have two clean options:

- **Use the project's prebuilt images** if it publishes them (the README mentions `--prebuilt` and
  "pre-built Docker images"). Reference those image names directly — simplest.
- **Build locally and load into the cluster.** With `kind`:
  ```bash
  docker build -t mcpgw/registry:dev ./path/to/service
  kind load docker-image mcpgw/registry:dev --name learn   # makes the image available to kind nodes
  ```
  Then set `image: mcpgw/registry:dev` and `imagePullPolicy: IfNotPresent` in the Deployment so
  Kubernetes doesn't try to fetch it from the internet. (On Docker Desktop, locally-built images
  are already visible to the cluster — no load step needed.)

---

## Step 4 — Replace `depends_on` with health probes

Compose's `depends_on` waits for a container to *start*; it doesn't wait for the app to be *ready*.
Kubernetes does this better with probes. Add to each Deployment's container:

```yaml
          readinessProbe:            # "is it ready to receive traffic yet?"
            httpGet:
              path: /health          # use the service's real health path (check the app)
              port: 8000
            initialDelaySeconds: 10
            periodSeconds: 5
          livenessProbe:             # "is it still healthy, or should we restart it?"
            httpGet:
              path: /health
              port: 8000
            initialDelaySeconds: 30
            periodSeconds: 10
```

Now the Registry API's Service won't send traffic until the Pod reports ready, and ordering sorts
itself out via retries — no explicit `depends_on` needed. If a service has no `/health` endpoint,
use a `tcpSocket` probe on its port instead, and ask Claude Code what the right check is for that
component.

---

## Step 5 — Verify the whole system

```bash
kubectl get all -n mcp-gateway                    # everything, one screen
kubectl get pods -n mcp-gateway                   # all Running / Ready
kubectl logs -n mcp-gateway deploy/registry       # app logs
curl http://registry.localhost                    # or your Ingress host / port-forward
```

Success = you open the Registry UI in your browser, it loads, and it's being served by Pods that
Kubernetes is keeping alive. Prove the self-healing you learned in Phase 4: delete the registry
Pod and watch it come back while the Service holds the address.

---

## Step 6 — Reflect on what Kubernetes bought you (and what it cost)

Now that both versions run, you can *feel* the trade-off honestly:

- **Gained:** self-healing, independent scaling per tier, rolling updates, a path to multi-machine
  and cloud (the project's docs mention EKS "coming soon" — your manifests are a head start).
- **Cost:** far more YAML and moving parts than a single `docker-compose up`. For one app on one
  Mac, Compose is genuinely simpler — and that's a legitimate choice, not a failure.

You now understand *both*, which is the whole point. That's a real, employable skill.

---

## Stretch goals (when you want more)

- Wrap your manifests in a **Helm chart** (Phase 7) so the whole stack installs with one command
  and `values.yaml`.
- Replace the app's nginx with a Kubernetes **Ingress** and TLS.
- Move MongoDB to a proper **StatefulSet**.
- Add resource `requests`/`limits` to every container so the scheduler packs them sensibly.
- Try deploying the same manifests to a real cloud cluster (EKS) once local works.

---

## Ask Claude Code (your main capstone partners)

> Kick-off: I've cloned `mcp-gateway-registry` and it runs under Docker Compose. Help me port it to
> my local Kubernetes cluster in the `mcp-gateway` namespace, one tier at a time starting with
> MongoDB. Read the `docker-compose.yml`, and for the first tier only, generate the manifests,
> explain them, and give me the exact `kubectl` commands to deploy and verify before we move on.

> Debugging: my `registry` pod is `CrashLoopBackOff`. Here's `kubectl describe pod` and
> `kubectl logs`: [paste]. It ran fine under Compose. Compare the two and tell me what config the
> Kubernetes version is missing.

---

**You did it.** From "very basic understanding" to a real, multi-service application running on
Kubernetes that *you* translated and deployed. Revisit [reference/](../reference/) any time, and
keep the Compose version around as your oracle.
