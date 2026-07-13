# Learning log

My running journal of working through this repo — the detours, misconceptions, and "aha"
moments that the phase files couldn't have predicted. Newest entries at the top. Convention:
one `##` section per date, one `###` subsection per phase or topic touched that day.

---

## 2026-07-13

### 04-first-deployment — the containerPort detour (ports, ConfigMaps, and port-forward)

While doing the Phase 4 challenge (httpd, 2 replicas), I changed `containerPort` to `90` and
thought the pods were broken. They weren't — and the detour that followed taught me more than
the challenge itself.

**1. I misread healthy pods as broken.** `kubectl get pods` showed `1/1 Running` and the logs
ended with `resuming normal operations` — that's Apache announcing a *successful* start. The
`AH00558 ... ServerName` lines are a harmless warning, not an error. Lesson: read the evidence
before declaring things broken; Running + normal logs = the app is fine, look elsewhere.

**2. `containerPort` is documentation, not configuration.** Setting `containerPort: 90` does
NOT make httpd listen on 90. Apache listens on 80 because *its own config file* says `Listen 80`
— it has no idea my YAML exists. My pods were serving happily on 80 while the YAML claimed 90,
so anything trusting the YAML (Service `targetPort: 90`, `port-forward :90`) got connection
refused. The manifest was lying about reality. Proof command:

```bash
kubectl exec deploy/web -- grep ^Listen /usr/local/apache2/conf/httpd.conf   # -> Listen 80
```

**The chain of truth** (every link must agree, and only the app itself defines the last one):

```text
port-forward / Service port  →  targetPort  →  containerPort  →  what the app ACTUALLY listens on
        (yours to choose)         (must match reality ────────────────────┘)
```

**3. To truly change the app's port, change the app's config — the Kubernetes way is a
ConfigMap mounted over the file.** Three options, worst to best:

- *Live-edit inside the container* — evaporates when the Pod is replaced; the other replica
  never gets it. Anti-pattern.
- *Bake a custom image* — legitimate for big changes, but now I maintain an image for one line.
- *ConfigMap mounted as a file* — image stays stock `httpd:2.4`; config injected at runtime:

```bash
kubectl exec deploy/web -- cat /usr/local/apache2/conf/httpd.conf > /tmp/httpd.conf
sed -i '' 's/^Listen 80/Listen 90/' /tmp/httpd.conf
kubectl create configmap httpd-conf --from-file=/tmp/httpd.conf
```

```yaml
          volumeMounts:
            - name: conf
              mountPath: /usr/local/apache2/conf/httpd.conf
              subPath: httpd.conf      # mount ONLY this file
      volumes:
        - name: conf
          configMap:
            name: httpd-conf
```

The `subPath` matters: without it, the mount replaces the entire `conf/` directory with just my
one file, hiding `mime.types` etc. — Apache would crash. With it, one file is overlaid and the
rest of the directory stays.

**4. This is NOT an image change.** The image is read-only and byte-identical to Docker Hub's
`httpd:2.4` — verifiable with `docker run --rm httpd:2.4 grep ^Listen ...` (still says 80).
The mount is a runtime overlay: a sticky note over one line of a laminated recipe. Remove the
mount and the original file is right there again. This is why the pattern wins: one stock image
across dev/staging/prod, each environment mounting its own config; config-in-image means every
config change becomes an image rebuild.

**5. Where a ConfigMap actually lives:** it's a first-class API object stored in **etcd** (via
the API server), namespaced, durable — it survives reboots and exists whether or not any Pod
uses it. At Pod start, the **kubelet** on that node fetches it and materializes it as a file for
the mount. And its `data` is literally a map of string keys → string values — my whole
`httpd.conf` is one multi-line string under the key `httpd.conf` (the `|` block-scalar in YAML).
Key becomes filename; value becomes content. ~1MiB size limit (etcd caps object sizes).
Corollary: Secrets live in etcd the same way — hence "base64 isn't encryption" and
encryption-at-rest (Phase 12).

**6. `port-forward` blocking my terminal is by design.** The kubectl process IS the tunnel —
each `Handling connection` line is one of my curls passing through it. There's deliberately no
daemonize flag. The workflow answers:

- Quick poke → **second terminal** (VS Code split terminal). What everyone does.
- Shell `&` works (`jobs` / `kill %1`) but logs interleave, and if the Pod restarts the tunnel
  breaks silently.
- Anything longer-lived → that's the smell that I want a real **Service** (`type: LoadBalancer`
  maps to localhost on Docker Desktop) or an **Ingress**, which live in the cluster and survive
  my terminal.

**Takeaway sentence:** Kubernetes YAML describes and wires; it does not reach inside the
container and reconfigure the app — the app's own config is the ground truth, and ConfigMaps
are how you own that truth from outside the image.
