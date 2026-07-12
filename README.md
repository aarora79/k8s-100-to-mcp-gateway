# Kubernetes: from 100-level to running the MCP Gateway Registry

A step-by-step learning path for going from **"I have a very basic understanding"** to
**"I deployed a real, multi-service application (`mcp-gateway-registry`) on Kubernetes"** —
all on an Apple Silicon **Mac Studio**, using **VS Code** as the editor and **Claude Code**
as your pair-programming buddy.

This repo is designed to be *worked through*, not just read. Each phase is a self-contained
lesson with concepts, hands-on commands, checkpoints, and a "Ask Claude Code" prompt you can
paste to go deeper or get unstuck.

---

## Who this is for

Someone who:

- Is comfortable in a terminal but new to containers and Kubernetes.
- Learns best by *doing* small things and building up.
- Wants a real payoff at the end — running the `mcp-gateway-registry` app on a local cluster.

There is **no prior Kubernetes or Docker knowledge assumed.** We start at zero.

---

## The end goal

By the end you will have:

1. A mental model of what containers and Kubernetes actually are (and when *not* to use them).
2. A working local Kubernetes cluster on your Mac Studio.
3. Hands-on reps deploying, exposing, configuring, and debugging apps.
4. The `mcp-gateway-registry` stack — Registry API, Auth server, MCP Gateway (nginx), and a
   database — translated from Docker Compose into Kubernetes and running locally.

---

## How to use this repo (the iteration loop)

The whole point is that you can bounce between **Claude (Cowork)** and **Claude Code** in VS Code.

```text
┌─────────────────────────────────────────────────────────────┐
│  1. Read a phase file in phases/                             │
│  2. Run the hands-on commands in the VS Code terminal        │
│  3. Stuck or curious? Paste the "Ask Claude Code" prompt     │
│     from that phase into Claude Code                         │
│  4. Commit your notes / manifests as you go                  │
│  5. Move to the next phase when the checkpoint passes        │
└─────────────────────────────────────────────────────────────┘
```

Suggested VS Code extensions (install in Phase 0):

- **Kubernetes** (`ms-kubernetes-tools.vscode-kubernetes-tools`) — cluster explorer, YAML help
- **Docker** (`ms-azuretools.vscode-docker`) — image/container management
- **YAML** (`redhat.vscode-yaml`) — schema validation for k8s manifests
- **Claude Code** — your assistant, right in the editor

---

## The roadmap

| Phase | File | What you learn | Level |
| ----: | ---- | -------------- | ----- |
| 0 | [phases/00-prerequisites.md](phases/00-prerequisites.md) | Set up your Mac: Homebrew, Docker, kubectl, VS Code | 100 |
| 1 | [phases/01-containers-101.md](phases/01-containers-101.md) | What a container is; Docker basics by hand | 100 |
| 2 | [phases/02-kubernetes-concepts.md](phases/02-kubernetes-concepts.md) | Why Kubernetes exists; the core objects & architecture | 100 |
| 3 | [phases/03-local-cluster.md](phases/03-local-cluster.md) | Stand up a real cluster on your Mac; meet `kubectl` | 100 |
| 4 | [phases/04-first-deployment.md](phases/04-first-deployment.md) | Pods, Deployments, Services — deploy your first app | 150 |
| 5 | [phases/05-config-and-storage.md](phases/05-config-and-storage.md) | ConfigMaps, Secrets, Volumes, persistent data | 200 |
| 6 | [phases/06-networking-ingress.md](phases/06-networking-ingress.md) | Service types, Ingress, reaching apps from your browser | 200 |
| 7 | [phases/07-helm-packaging.md](phases/07-helm-packaging.md) | Helm: installing and templating apps | 200 |
| 8 | [phases/08-multi-service-app.md](phases/08-multi-service-app.md) | Deploy a realistic multi-container app end-to-end | 250 |
| 9 | [phases/09-capstone-mcp-gateway.md](phases/09-capstone-mcp-gateway.md) | **Capstone:** run `mcp-gateway-registry` on Kubernetes | 300 |
| 10 | [phases/10-beyond-the-capstone.md](phases/10-beyond-the-capstone.md) | Requests/limits, autoscaling (HPA, Karpenter), node affinity & taints | 350 |
| 11 | [phases/11-capstone-2-llm-serving.md](phases/11-capstone-2-llm-serving.md) | **Capstone 2:** serve an open-source LLM (Gemma 4) + Prometheus/Grafana, with a vLLM/GPU cloud track | 350 |
| 12 | [phases/12-expert-concepts.md](phases/12-expert-concepts.md) | The expert concept map — theory-only reading, no cluster needed | 400 |

Reference material you'll come back to:

- [reference/glossary.md](reference/glossary.md) — plain-English definitions of every term
- [reference/kubectl-cheatsheet.md](reference/kubectl-cheatsheet.md) — the commands you'll actually use
- [reference/troubleshooting.md](reference/troubleshooting.md) — "it's broken, now what"
- [manifests/](manifests/) — YAML you'll write and reuse

---

## Which Kubernetes tool are we using, and why?

There are many ways to run Kubernetes locally on a Mac. For a first-timer on a powerful Mac
Studio who wants to reach a real app, we use a **two-step tooling path**:

1. **Docker Desktop with its built-in Kubernetes** for Phases 1–7. It's a GUI one-click toggle,
   uses the Docker you'll already have, and removes a whole class of setup problems while you're
   learning concepts. (OrbStack is an excellent, lighter alternative — noted in Phase 3 if you'd
   rather go that way.)
2. **`kind`** (Kubernetes-in-Docker) introduced in Phase 8, when you want a more "real",
   multi-node, reproducible cluster that behaves closer to production — a great fit for the
   capstone.

You do **not** need to decide now. Phase 3 walks you through the choice with exact commands.

> **A note on honesty about scope:** Kubernetes is genuinely a big topic, and it is *overkill*
> for running a single app on one machine (Docker Compose already does that — which is how
> `mcp-gateway-registry` ships). We're using it here because your goal is to *learn Kubernetes*,
> and porting a real Compose app to it is one of the best ways to learn. Phase 9 is explicit
> about what Kubernetes buys you versus Compose.

---

## Ground rules that will save you pain

- **Go in order.** Each phase assumes the last one worked.
- **Type the commands yourself** the first time — don't just copy-paste blindly. Muscle memory matters.
- **When something breaks, that's the lesson.** Use the troubleshooting file and Claude Code.
- **Commit often.** Your manifests and notes are part of the learning artifact.

Ready? Start with **[Phase 0: Prerequisites](phases/00-prerequisites.md)**.
