
# k9s Installation & Setup Guide for k3s (Single‑Node Homelab)

This guide explains how to install, configure, and use **k9s**, the best TUI (Terminal UI) for managing a Kubernetes cluster — perfect for your single‑node k3s environment.

Everything is written in fully copy‑paste‑ready steps.

---

## 🟦 1. What is k9s?

k9s is a **terminal‑based Kubernetes dashboard** that lets you:

- Inspect pods, logs, events
- Edit resources in place
- Restart deployments
- Port‑forward services
- Navigate all Kubernetes objects fast
- Manage multiple namespaces interactively

It is lightweight and ideal for homelabs.

---

# 🟦 2. Install k9s on Linux (Recommended for k3s Nodes)

## **2.1 Download the latest release**

```bash
curl -s https://api.github.com/repos/derailed/k9s/releases/latest   | grep browser_download_url   | grep Linux_x86_64.tar.gz   | cut -d '"' -f 4   | wget -i -
```

## **2.2 Extract and install**

```bash
tar -xzf k9s_Linux_x86_64.tar.gz
sudo mv k9s /usr/local/bin/
```

Verify:

```bash
k9s version
```

You should see version output like:
```
Version:    0.xx.x
```

---

# 🟦 3. Install k9s on macOS

If you're using macOS:

```bash
brew install k9s
```

---

# 🟦 4. Install k9s on Windows

Use Chocolatey:
```powershell
choco install k9s
```

or use Scoop:
```powershell
scoop install k9s
```

---

# 🟦 5. Configure k9s for k3s

k3s stores its kubeconfig at:
```
/etc/rancher/k3s/k3s.yaml
```

Copy it to your user location so `k9s` can read it:

```bash
mkdir -p ~/.kube
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
sudo chown $USER:$USER ~/.kube/config
```

If your k3s listens on localhost only, ensure the config uses the correct API server address (or IP of your MiniPC):

Example fix:
```bash
sed -i 's/127.0.0.1/192.168.1.50/g' ~/.kube/config
```

Replace `192.168.1.50` with your MiniPC’s LAN IP.

---

# 🟦 6. Start k9s

Just run:

```bash
k9s
```

Navigation:
- `:` → command mode
- `/` → search
- `d` → describe resource
- `l` → logs
- `s` → shell into a pod
- `e` → edit YAML
- `0` → switch namespaces
- `Ctrl+a` → show all commands

---

# 🟦 7. Optional: Install a nicer k9s skin/theme

Create config folder:
```bash
mkdir -p ~/.config/k9s
```

Create `~/.config/k9s/skin.yml`:

```yaml
k9s:
  skin: dark
```

Or pick from community themes:
https://github.com/derailed/k9s/tree/master/skins

Place your theme file as:
```
~/.config/k9s/skins/<theme>.yaml
```

Activate it:
```yaml
k9s:
  skin: <theme>
```

---

# 🟦 8. Optional: Enable k9s plugins (very useful)

Create plugins directory:
```bash
mkdir -p ~/.config/k9s/plugins
```

Example plugin to view cluster CRDs:
```yaml
# ~/.config/k9s/plugins/crds.yaml
plugin:
  shortCut: c
  description: View CRDs
  scopes:
    - all
  command: kubectl
  args:
    - get
    - crds
    - -A
```

Press `:` then type `crds` inside k9s.

---

# 🟦 9. k9s Usage Tips for Your Homelab

### ✔ Perfect for debugging Flux automation
Use:
```bash
:pods -n flux-system
```
To see real-time logs from: source‑controller, image‑reflector, image‑automation.

### ✔ Inspect Traefik
```bash
:svc -n traefik
:pods -n traefik
```

### ✔ Debug Immich, Radicale, Syncthing deployments
```bash
:deploy -n immich
:deploy -n radicale
:deploy -n syncthing
```

### ✔ Port forward Immich or Radicale when debugging
```bash
:f
```
Then select a pod to forward ports.

---

# 🟦 10. Summary Commands

```bash
# Install k9s (Linux)
curl -s https://api.github.com/repos/derailed/k9s/releases/latest   | grep browser_download_url   | grep Linux_x86_64.tar.gz   | cut -d '"' -f 4   | wget -i -
tar -xzf k9s_Linux_x86_64.tar.gz
sudo mv k9s /usr/local/bin/

# Configure kubeconfig for k3s
mkdir -p ~/.kube
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
sudo chown $USER:$USER ~/.kube/config
sed -i 's/127.0.0.1/192.168.1.50/g' ~/.kube/config

# Launch k9s
k9s
```

---

If you'd like, I can generate:
- A **k9s quick‑reference cheat sheet**
- A **k9s + Flux debugging guide**
- A **k9s + Traefik Gateway API debugging guide**
