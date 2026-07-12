# Phase 1 — Containers 101

**Goal:** understand what a container *actually is* and get comfortable with Docker by hand.
Kubernetes orchestrates containers, so you can't understand Kubernetes until containers click.

**Time:** ~1 hour of hands-on.

---

## The one-paragraph mental model

A **container** is a way to package an application *together with everything it needs to run*
(its code, runtime, libraries, system tools) into a single, isolated, portable unit. It runs on
top of a shared operating-system kernel, so it starts in milliseconds and is far lighter than a
virtual machine. Think of it as "a process wrapped in its own private filesystem and network,
frozen so it runs identically on your Mac, a colleague's laptop, or a server in the cloud."

Two words you must not confuse:

- An **image** is the *blueprint* — a read-only, layered package built from a recipe. Like a
  class in programming, or a stamped disc.
- A **container** is a *running instance* of an image. Like an object, or a disc that's playing.
  You can start many containers from one image.

---

## Why containers exist (the "works on my machine" problem)

Before containers, you'd install an app on a server and it would fail because the server had a
different Python version, a missing library, or a different config. Containers solve this by
shipping the *entire environment* alongside the app. The image that runs on your Mac Studio is
byte-for-byte the same image that runs in production.

This is *exactly* why `mcp-gateway-registry` ships as containers — so its Registry API, auth
server, and nginx gateway each run in a predictable environment regardless of the host.

---

## Hands-on: run something real

Let's run a web server in a container. In your terminal:

```bash
docker run -d -p 8080:80 --name web nginx
```

Breaking that down:

- `docker run` — create and start a container
- `-d` — detached (run in the background)
- `-p 8080:80` — map port 8080 on your Mac to port 80 inside the container
- `--name web` — give it a friendly name
- `nginx` — the image to use (Docker downloads it from Docker Hub if you don't have it)

Now open <http://localhost:8080> in your browser. You're looking at a web server running inside a
container. Inspect it:

```bash
docker ps                      # list running containers
docker logs web                # see its output
docker exec -it web sh         # get a shell INSIDE the container
#   inside: run `ls /`, `cat /etc/os-release`, then `exit`
```

Notice inside the container it looks like a minimal Linux machine, totally separate from your Mac.
Clean up:

```bash
docker stop web && docker rm web
```

---

## Hands-on: build your own image

This is the part that makes containers "click". Create a folder and two files:

```bash
mkdir ~/container-lab && cd ~/container-lab
```

`app.py`:

```python
from http.server import HTTPServer, BaseHTTPRequestHandler

class Handler(BaseHTTPRequestHandler):
    def do_GET(self):
        self.send_response(200)
        self.end_headers()
        self.wfile.write(b"Hello from inside a container!\n")

HTTPServer(("0.0.0.0", 8000), Handler).serve_forever()
```

`Dockerfile` (the recipe for the image):

```dockerfile
# Start from a small Python base image
FROM python:3.12-slim

# Set the working directory inside the image
WORKDIR /app

# Copy our code in
COPY app.py .

# Document the port the app listens on
EXPOSE 8000

# What to run when the container starts
CMD ["python", "app.py"]
```

> Note the comments sit on their **own lines**. In a Dockerfile, a `#` after an instruction is
> *not* a comment — it becomes part of the instruction's arguments and breaks the build with a
> confusing parse error. Comment lines must start with `#`.

Build and run it:

```bash
docker build -t my-first-app .          # -t names the image "my-first-app"
docker run -d -p 8000:8000 --name mine my-first-app
curl http://localhost:8000              # -> Hello from inside a container!
docker stop mine && docker rm mine
```

You just built a portable image. That `docker build` → `docker run` loop is the heart of the
container workflow, and every service in `mcp-gateway-registry` is built exactly this way.

---

## Key ideas to carry forward

- **Images are layered and cached.** Each `Dockerfile` line is a layer; unchanged layers are
  reused, which makes rebuilds fast.
- **Containers are ephemeral.** When a container is deleted, anything written inside it is gone
  unless you used a *volume* (Phase 5). This matters enormously for databases.
- **One process per container** is the convention. A web app + its database = two containers, not
  one. This is why real apps are *multi-container* — and why we need something to coordinate them.
- **That coordinator is Kubernetes.** When you have many containers across many machines that need
  to find each other, restart on failure, and scale — Docker alone isn't enough. That's Phase 2.

---

## Checkpoint ✅

You can:

- [ ] Explain the difference between an image and a container.
- [ ] Run, inspect, and stop a container.
- [ ] Write a simple `Dockerfile`, build an image from it, and run it.

---

## Ask Claude Code

> Explain Docker image layers to me using my `~/container-lab/Dockerfile` as the example.
> Which lines create layers, and why does changing `app.py` not force a re-download of the
> Python base image?

**Next:** [Phase 2 — Kubernetes concepts](02-kubernetes-concepts.md)
