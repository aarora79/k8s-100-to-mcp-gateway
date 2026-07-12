# Phase 0 — Prerequisites: get your Mac Studio ready

**Goal:** install the handful of tools you'll use for the whole journey, and confirm each one
works. No Kubernetes yet — just the foundation.

**Time:** ~30–45 minutes, most of it downloads.

---

## What we're installing and why

| Tool | What it is | Why you need it |
|------|-----------|-----------------|
| Homebrew | The package manager for macOS | Installs everything else with one command each |
| Docker Desktop | Runs containers on your Mac; also ships a one-click Kubernetes | The engine under everything we do |
| kubectl | The command-line tool to talk to a Kubernetes cluster | Your primary interface to Kubernetes |
| VS Code + extensions | Your editor | Where you'll write YAML and run terminals |
| git | Version control | You're already using it — this repo lives in it |

---

## Step 1 — Install Homebrew

Homebrew is how Mac developers install command-line software. Open the **Terminal** app (or the
VS Code integrated terminal) and run:

```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```

After it finishes, it may print two `echo ... >> ~/.zprofile` lines — **run those**, then close
and reopen the terminal. Verify:

```bash
brew --version
```

You should see a version number. On Apple Silicon, Homebrew lives at `/opt/homebrew`.

---

## Step 2 — Install Docker Desktop

Docker Desktop is the easiest way to run containers on a Mac, and it has a built-in Kubernetes
we'll turn on in Phase 3.

```bash
brew install --cask docker
```

Then **open Docker Desktop from Applications** (the first launch asks for a password to install a
helper). Wait for the whale icon in the menu bar to stop animating — that means the engine is up.

Verify from the terminal:

```bash
docker --version
docker run hello-world
```

The `hello-world` command downloads a tiny image and runs it. If you see a "Hello from Docker!"
message, containers work. 🎉

> **Apple Silicon note:** Docker runs a lightweight Linux VM under the hood. Most images have
> `arm64` builds and "just work". Occasionally you'll hit an `amd64`-only image; Docker emulates
> it automatically (slower). We'll flag this if it comes up.

---

## Step 3 — Install kubectl

`kubectl` (say "kube-control" or "kube-cuttle" — people argue about this) is the CLI you'll type
hundreds of times.

```bash
brew install kubectl
kubectl version --client
```

The `--client` flag just prints kubectl's own version; it won't complain that there's no cluster
yet. That comes in Phase 3.

---

## Step 4 — Set up VS Code

If you don't have it:

```bash
brew install --cask visual-studio-code
```

Install the extensions we'll use (you can also do this from the Extensions sidebar):

```bash
code --install-extension ms-kubernetes-tools.vscode-kubernetes-tools
code --install-extension ms-azuretools.vscode-docker
code --install-extension redhat.vscode-yaml
```

- **Kubernetes** gives you a cluster explorer in the sidebar and hover-help on YAML fields.
- **Docker** lets you see images/containers without the terminal.
- **YAML** validates your manifests against the Kubernetes schema as you type — this catches a
  *lot* of beginner mistakes.

And of course keep the **Claude Code** extension handy — that's your iteration partner.

---

## Step 5 — Confirm git and clone this repo locally

You'll want this repo on your Mac so Claude Code can see the phase files and you can commit notes.

```bash
git --version           # should already exist on macOS
# (You'll clone this repo once it's on GitHub — see the top-level chat for the push command.)
```

---

## Checkpoint ✅

Run these four commands. All should succeed:

```bash
brew --version
docker run hello-world
kubectl version --client
code --version
```

If every one prints something sensible, you're done with Phase 0.

---

## Ask Claude Code

Paste this into Claude Code if anything above misbehaved:

> I'm on an Apple Silicon Mac Studio setting up for Kubernetes learning. I ran
> `docker run hello-world` (or `brew install ...`) and got this output: [paste the error].
> Explain what went wrong in beginner terms and give me the exact commands to fix it.

**Next:** [Phase 1 — Containers 101](01-containers-101.md)
