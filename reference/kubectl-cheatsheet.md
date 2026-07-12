# kubectl cheatsheet — the commands you'll actually use

Tip: `alias k=kubectl` (set in Phase 3). Everywhere below, `k` == `kubectl`.

## Context & cluster
```bash
kubectl config get-contexts             # list clusters you can talk to
kubectl config current-context          # which one am I on RIGHT NOW (check often!)
kubectl config use-context docker-desktop
kubectl cluster-info                    # API server URL
kubectl get nodes                       # cluster machines + status
```

## Looking at things (read-only, always safe)
```bash
kubectl get pods                        # pods in current namespace
kubectl get pods -A                     # pods in ALL namespaces
kubectl get pods -n mcp-gateway         # pods in a specific namespace
kubectl get pods -o wide                # + node and IP columns
kubectl get pods -l app=web             # filter by label
kubectl get all                         # pods, services, deployments, etc.
kubectl get deploy,svc,ingress          # several kinds at once
kubectl get pods --watch                # live updates (Ctrl-C to stop)
```

## The debugging trio (memorize this order)
```bash
kubectl get pods                        # 1. what's the status?
kubectl describe pod <name>             # 2. why? (read the Events at the bottom)
kubectl logs <name>                     # 3. what did the app say?
kubectl logs -f <name>                  #    follow live
kubectl logs <name> --previous          #    logs from the crashed previous container
kubectl get events --sort-by=.metadata.creationTimestamp
```

## Applying and deleting (declarative)
```bash
kubectl apply -f file.yaml              # create or update to match the file
kubectl apply -f ./manifests/           # apply every file in a folder
kubectl delete -f file.yaml             # remove exactly what that file created
kubectl diff -f file.yaml               # preview what apply would change
```

## Creating things quickly (imperative — good for experiments)
```bash
kubectl create deployment web --image=nginx
kubectl expose deployment web --port=80          # make a Service for it
kubectl create namespace mcp-gateway
kubectl create configmap app-config --from-literal=KEY=value
kubectl create secret generic creds --from-literal=PASSWORD=xyz
kubectl run tester --rm -it --image=busybox -- sh   # throwaway debug pod
```

## Interacting with running Pods
```bash
kubectl exec -it <pod> -- sh            # shell into a container
kubectl exec <pod> -- env               # run one command inside
kubectl cp <pod>:/path/file ./file      # copy a file out of a pod
kubectl port-forward svc/web 8080:80    # tunnel localhost:8080 -> Service
kubectl port-forward pod/<name> 8080:80 # tunnel to a specific Pod
```

## Scaling, updating, rolling back
```bash
kubectl scale deployment web --replicas=5
kubectl set image deployment/web nginx=nginx:1.28
kubectl rollout status deployment/web
kubectl rollout history deployment/web
kubectl rollout undo deployment/web
kubectl rollout restart deployment/web  # recreate pods (e.g. to reload config)
```

## Generate YAML without applying (great starting point)
```bash
kubectl create deployment web --image=nginx --dry-run=client -o yaml > web.yaml
kubectl create service clusterip web --tcp=80:80 --dry-run=client -o yaml
```
Use these to scaffold a manifest, then edit it — faster than writing from scratch.

## Helm quickies
```bash
helm repo add bitnami https://charts.bitnami.com/bitnami && helm repo update
helm search repo mongodb
helm install mongo bitnami/mongodb -n data --create-namespace
helm list -A
helm uninstall mongo -n data
helm template <chart>                   # render without installing
helm show values bitnami/mongodb        # all configurable values
```

## kind quickies
```bash
kind create cluster --name learn --config kind-config.yaml
kind get clusters
kind load docker-image myimg:dev --name learn   # make a local image available
kind delete cluster --name learn
```

## Cleanup patterns
```bash
kubectl delete pod <name>
kubectl delete deployment web
kubectl delete namespace mcp-gateway    # deletes EVERYTHING in it — careful
kubectl delete all --all -n <namespace> # wipe a namespace's workloads
```
