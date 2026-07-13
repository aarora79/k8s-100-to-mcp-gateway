# Phase 6 — Networking & Ingress

**Goal:** understand how traffic actually reaches your apps — Service types, DNS between services,
and Ingress for browser-friendly URLs. The capstone's nginx gateway is essentially a big Ingress
story, so this phase is directly load-bearing.

**Time:** ~1–1.5 hours.

---

## Before you start — 60 seconds of retrieval

From memory:

1. Why do Pods need a Service in front of them at all — what property of Pods makes direct
   connections a bad idea? *(Phase 4)*
2. How does a Service know *which* Pods to send traffic to? *(Hint: it's the same mechanism a
   Deployment uses to find its Pods.)*
3. Who resolves a Service's name to an address inside the cluster? *(Phase 2's worked example)*

---

## Service types (the confusing part, made simple)

A Service gives Pods a stable address. *Which* address, and who can reach it, depends on the type:

| Type | Reachable from | Use it for |
| ---- | -------------- | --------- |
| **ClusterIP** (default) | *Inside* the cluster only | Service-to-service calls (API → database) |
| **NodePort** | Outside, via `<node-ip>:<high-port>` | Quick external access while learning |
| **LoadBalancer** | Outside, via a real load balancer | Cloud production; on Docker Desktop it maps to `localhost` |

Rule of thumb: **internal services stay ClusterIP; you expose a handful of front doors** via
Ingress (below). You rarely want dozens of NodePorts.

---

## Service discovery: how services find each other by name

This is the feature that makes multi-service apps sane. Every Service gets a DNS name inside the
cluster:

```text
<service-name>.<namespace>.svc.cluster.local
```

…and within the same namespace you can just use `<service-name>`. So the Registry API can reach
MongoDB at `mongodb:27017` simply because there's a Service named `mongodb`. No IP addresses, no
config files listing hosts. **This is why we put each component behind a Service in Phase 9.**

Try it. Deploy two things and have one call the other by name:

```bash
kubectl create deployment web --image=nginx
kubectl expose deployment web --port=80
# "expose X" = make a Service pointing at X's pods; the Service inherits the name "web"
kubectl run tester --rm -it --image=busybox -- sh
# "tester" = pod name you choose; --rm = delete the pod when you exit;
# -it = interactive terminal; after "--" runs inside the container (here: a shell)
#   inside the tester pod:
wget -qO- http://web           # reaches nginx by SERVICE NAME. exit when done.
kubectl delete deployment web && kubectl delete service web
```

You just did cross-service networking with nothing but a name.

---

## Break it on purpose: the empty-endpoints bug

The single most common Service bug in real life is a **selector that doesn't match any Pod
labels** — everything *looks* healthy, but traffic goes nowhere. Cause it once so you recognize
it forever. With the `web` deployment and service from above still running, introduce a one-letter
typo into the live Service's selector:

```bash
kubectl patch service web -p '{"spec":{"selector":{"app":"webb"}}}'   # note: webb, not web
```

Now observe the symptoms:

```bash
kubectl get pods -l app=web        # pods: Running, READY — everything looks fine!
kubectl get service web            # service: exists, has an IP — looks fine too!
kubectl run tester --rm -it --image=busybox -- wget -qO- --timeout=3 http://web
#                                  # ...but this HANGS / times out
```

The tell — burn this command into memory:

```bash
kubectl get endpoints web          # ENDPOINTS: <none>   ← the smoking gun
```

A Service with no endpoints means "my selector matches zero Pods." Compare
`kubectl get service web -o wide` (shows the selector) against `kubectl get pods --show-labels`
and the typo jumps out. Fix it and re-verify:

```bash
kubectl patch service web -p '{"spec":{"selector":{"app":"web"}}}'
kubectl get endpoints web          # now lists Pod IPs — traffic flows again
```

When "I can't reach my app" strikes in the capstone, `kubectl get endpoints` is check #3 in
[reference/troubleshooting.md](../reference/troubleshooting.md) — now you've seen exactly why.

---

## Ingress: friendly URLs into the cluster

`port-forward` and NodePorts are clumsy for real use. An **Ingress** is an HTTP router at the edge
of the cluster: it maps hostnames/paths (like `registry.localhost/` or `/api`) to internal
Services. But Ingress objects do nothing on their own — you need an **Ingress controller**
(a Pod that actually implements the routing, usually nginx).

### Install the nginx Ingress controller

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/cloud/deploy.yaml
kubectl get pods -n ingress-nginx --watch     # wait until the controller pod is Running
```

> If that URL's manifest changes, ask Claude Code for the current install command for
> ingress-nginx on Docker Desktop — the project occasionally moves paths.

### Deploy an app and route to it

```bash
kubectl create deployment web --image=nginx
kubectl expose deployment web --port=80
```

`manifests/demo-ingress.yaml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web
spec:
  ingressClassName: nginx
  rules:
    - host: web.localhost           # a hostname that resolves to 127.0.0.1
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: web            # route to the "web" Service
                port:
                  number: 80
```

```bash
kubectl apply -f manifests/demo-ingress.yaml
# On Docker Desktop the controller listens on localhost:80
curl http://web.localhost           # or open it in your browser
```

You now have a real URL routed through nginx to your Pod — exactly the shape the `mcp-gateway`
service provides in the capstone (it *is* an nginx reverse proxy).

Clean up:

```bash
kubectl delete -f manifests/demo-ingress.yaml
kubectl delete deployment web && kubectl delete service web
```

---

## A word on the capstone's gateway

`mcp-gateway-registry` uses nginx as its reverse proxy / router. When we move it to Kubernetes you
have two valid designs:

1. Keep the app's **own nginx** container as a Deployment+Service (a faithful port of Compose), or
2. Replace it with a Kubernetes **Ingress** (more idiomatic).

For learning, Phase 9 keeps the app's nginx as-is first (least surprises), then notes how you'd
refactor to Ingress. Both are legitimate; porting faithfully first is the calmer path.

---

## Challenge — no training wheels ✋

Without looking at the YAML above: deploy `httpd:2.4` as a second app alongside `web`, give it
its own Service, and extend (or add) an Ingress so that `httpd.localhost` reaches httpd while
`web.localhost` still reaches nginx. Verify both with `curl`, then break one selector on purpose
and confirm `kubectl get endpoints` catches it. Clean up everything.

---

## Checkpoint ✅

- [ ] Name the three Service types and when to use each.
- [ ] Call one Service from another Pod using only its name (DNS).
- [ ] Diagnose a wrong-selector Service using `kubectl get endpoints`.
- [ ] Install an Ingress controller and route a hostname to a Service.
- [ ] Complete the two-hostname challenge unaided.

---

## Ask Claude Code

> Explain the difference between a Service of type LoadBalancer and an Ingress. On my Docker
> Desktop cluster, which should I use to expose the `mcp-gateway` nginx to my browser, and why?

**Next:** [Phase 7 — Helm packaging](07-helm-packaging.md)
