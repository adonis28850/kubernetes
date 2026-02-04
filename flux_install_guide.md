
# Flux Installation & Configuration Guide for k3s (Single Node)

This document contains all commands and steps required to:
- Install Flux on your k3s cluster
- Configure GitHub connectivity
- Apply all Flux manifests
- Enable PR-based image automation for your Helm charts

---

## 1. Install Flux (Minimal Components)

```bash
kubectl create namespace flux-system

curl -s https://fluxcd.io/install.sh | sudo bash

flux install   --components=source-controller,image-reflector-controller,image-automation-controller,notification-controller   --namespace=flux-system
```

This installs only the components needed for:
- Git repository integration
- Image scanning
- Image policy evaluation
- Pull Request creation

Flux will **not** apply any manifests to your cluster.

---

## 2. Create GitHub Personal Access Token (PAT)

1. Go to GitHub: **Settings → Developer settings → Personal access tokens (fine-grained)**.
2. Create a token with:
   - Repository access → **adonis28850/kubernetes**
   - Permissions → **Contents (read/write)**, **Pull Requests (read/write)**

Create the Kubernetes secret:

```bash
kubectl create secret generic github-token   --namespace=flux-system   --from-literal=password='github_pat_1...' --from-literal=username=xxxx;
```

This allows Flux to open PRs on your repository.

---

## 3. Apply GitRepository Configuration

```bash
kubectl apply -f kubernetes/flux/git-repository.yaml
```

Flux can now read and write to:
```
https://github.com/adonis28850/kubernetes
```

Flux **does not deploy** any of your manifests.

---

## 4. Apply ImageRepositories (Scan External Registries)

```bash
kubectl apply -f kubernetes/flux/image-repositories/
```

This registers:
- Immich server
- Immich machine learning
- Syncthing
- Radicale

Flux scans these registries every hour.

---

## 5. Apply ImagePolicies (Choose Latest Tag)

```bash
kubectl apply -f kubernetes/flux/image-policies/
```

These policies define how Flux determines the "latest" image tag.

---

## 6. Apply ImageUpdateAutomation (PR Mode)

This makes Flux:
- Detect new image tags
- Update `values.yaml` files using your `$imagepolicy` markers
- Push changes to a branch (`flux/image-updates`)
- Open a **Pull Request** to `main`
- **NOT** deploy any changes

Apply it:

```bash
kubectl apply -f kubernetes/flux/image-automation/image-update-automation.yaml
```

---

## 7. Validate Installation

### Check Flux controllers

```bash
kubectl get pods -n flux-system
```

You should see:
- source-controller
- image-reflector-controller
- image-automation-controller
- notification-controller

### Check ImageRepositories
```bash
kubectl get imagerepository -n flux-system
```

### Check ImagePolicies
```bash
kubectl get imagepolicy -n flux-system
```

Each policy will eventually populate `status.latestImage`.

---

## 8. Expected Workflow

1. Flux detects new tags in:
   - ghcr.io/immich-app/immich-server
   - ghcr.io/immich-app/immich-machine-learning
   - syncthing/syncthing
   - tomsquest/docker-radicale

2. Flux updates:
   - `immich-chart/values.yaml`
   - `syncthing-chart/values.yaml`
   - `radicale-chart/values.yaml`

3. Flux creates a Pull Request:
   ```
   flux/image-updates → main
   ```

4. **You review and merge the PR**.

5. You deploy manually:

```bash
helm upgrade immich ./immich-chart -n immich
helm upgrade syncthing ./syncthing-chart -n syncthing
helm upgrade radicale ./radicale-chart -n radicale
```

Flux never touches your cluster.

---

## 9. All Commands Together (Cheat Sheet)

```bash
# Install Flux
kubectl create namespace flux-system
flux install   --components=source-controller,image-reflector-controller,image-automation-controller,notification-controller   --namespace=flux-system

# Create GitHub token secret
kubectl create secret generic github-token   --namespace=flux-system   --from-literal=token='<YOUR_GITHUB_PAT>'

# Apply Flux config
kubectl apply -f kubernetes/flux/git-repository.yaml
kubectl apply -f kubernetes/flux/image-repositories/
kubectl apply -f kubernetes/flux/image-policies/
kubectl apply -f kubernetes/flux/image-automation/image-update-automation.yaml
```

---

If you'd like, I can also generate a **diagnostics section** for troubleshooting Flux PR creation.
