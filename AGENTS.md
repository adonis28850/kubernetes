# Kubernetes Project Context Document

## Project Overview

This is a single-node k3s cluster configuration running on an OpenMediaVault server, used for deploying and managing multiple self-hosted services. The project follows GitOps principles, using Flux for automated image version management and Traefik as an ingress controller with Gateway API for service access.

### Core Components

- **k3s**: Single-node Kubernetes cluster with secretbox encryption enabled
- **Flux**: GitOps continuous deployment tool for automated image version updates
- **Traefik**: Ingress controller using Gateway API (not traditional Ingress API) for HTTP/HTTPS/TCP/UDP traffic
- **Helm Charts**: Custom Helm charts for deploying applications

### Deployed Applications

1. **Immich** - Self-hosted photo and video management platform
2. **Syncthing** - Continuous file synchronization program
3. **Radicale** - CalDAV and CardDAV server
4. **Jellyfin** - Media server for streaming movies and TV shows with Real-Debrid integration via Gelato plugin. All outbound traffic routes through ProtonVPN using Gluetun native sidecar.
5. **AIOStreams** - Stremio addon for aggregating streaming sources (Torrentio, Comet, etc.) with Real-Debrid

## Project Structure

```
kubernetes/
├── basic-traefik-setup/      # Traefik base configuration
│   ├── Chart.yaml
│   ├── values_basic.yaml     # Basic Traefik configuration
│   ├── values_extra_services.yaml  # Extra services configuration
│   └── values_traefik_dashboard.yaml  # Dashboard configuration
├── flux/                      # Flux GitOps configuration
│   ├── git-repository.yaml    # Git repository definition
│   ├── image-automation/      # Image automation configuration
│   ├── image-policies/        # Image update policies
│   └── image-repositories/    # Image repository definitions
├── immich-chart/              # Immich Helm chart
│   ├── Chart.yaml
│   ├── values.yaml
│   └── templates/
├── radicale-chart/            # Radicale Helm chart
│   ├── Chart.yaml
│   ├── config/                # Radicale configuration files
│   ├── values.yaml
│   └── templates/
├── syncthing-chart/           # Syncthing Helm chart
│   ├── Chart.yaml
│   ├── values.yaml
│   └── templates/
├── jellyfin-chart/            # Jellyfin Helm chart (official chart with custom values)
│   ├── Chart.yaml
│   ├── values.yaml
│   ├── pv.yaml                # Static PersistentVolumes for Jellyfin
│   ├── pvc.yaml               # PersistentVolumeClaims for Jellyfin
│   ├── httproute.yaml         # HTTPRoute for jellyfin.casa.local
│   └── templates/
│       ├── gateway.yaml       # Gateway API configuration
│       └── gluetun-configmap.yaml  # Gluetun VPN client configuration
├── aiostreams-chart/          # AIOStreams Helm chart (custom chart)
│   ├── Chart.yaml
│   ├── values.yaml
│   └── templates/
│       ├── deployment.yaml
│       ├── service.yaml
│       ├── gateway.yaml
│       ├── httproute.yaml
│       ├── pv.yaml
│       ├── pvc.yaml
│       ├── networkpolicy.yaml
│       ├── configmap.yaml
│       └── secret.yaml
└── *.md                       # Various documentation and guides
```

## Building and Deployment

### Initialize k3s Cluster

```bash
# Install k3s with secrets encryption
curl -sfL https://get.k3s.io | sh -s - server \
  --secrets-encryption \
  --secrets-encryption-provider secretbox \
  --disable=traefik \
  --node-name k3s-cluster \
  --tls-san openmediavault \
  --tls-san 192.168.0.227

# Configure kubectl
sudo chmod 644 /etc/rancher/k3s/k3s.yaml
echo "export KUBECONFIG=/etc/rancher/k3s/k3s.yaml" >> ~/.bashrc
source ~/.bashrc

# Enable kubectl autocomplete
echo 'source <(kubectl completion bash)' >>~/.bashrc
echo 'alias k=kubectl' >>~/.bashrc
echo 'complete -o default -F __start_kubectl k' >>~/.bashrc
```

### Install Traefik

```bash
# Remove built-in Traefik
rm -f /var/lib/rancher/k3s/server/manifests/traefik*
rm -f /var/lib/rancher/k3s/server/static/charts/traefik*

# Add Traefik Helm repository
helm repo add traefik https://helm.traefik.io/traefik
helm repo update

# Install Gateway API (with experimental features)
kubectl apply --server-side -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.4.1/experimental-install.yaml

# Install Traefik
helm install traefik traefik/traefik --namespace traefik --create-namespace

# Create TLS certificate
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout tls.key -out tls.crt -subj "/CN=*.k3s.local"
kubectl create secret tls local-selfsigned-tls \
  --cert=tls.crt --key=tls.key --namespace traefik

# Create Dashboard authentication
USER_PASS=$(htpasswd -nb admin password)
kubectl create secret generic dashboard-auth \
  --from-literal=users="$USER_PASS" --namespace traefik

# Apply custom configuration
helm upgrade traefik traefik/traefik --namespace traefik \
  -f basic-traefik-setup/values_basic.yaml \
  -f basic-traefik-setup/values_extra_services.yaml \
  --reuse-values
```

### Install Applications

#### Syncthing

```bash
# Create data directories
mkdir -p /home/antonio/data/syncthing/config/
mkdir -p /home/antonio/data/syncthing/data/
chown -R 1000:1000 /home/antonio/data/syncthing/
chmod -R 755 /home/antonio/data/syncthing/

# Install
helm install syncthing syncthing-chart/ --namespace syncthing --create-namespace
```

#### Radicale

```bash
# Create data directories
mkdir -p /home/antonio/data/radicale/config/
mkdir -p /home/antonio/data/radicale/data/
cp radicale-chart/config /home/antonio/data/radicale/config

# Create authentication
htpasswd -c auth username
kubectl create namespace radicale
kubectl create secret generic radicale-auth --from-file=auth -n radicale

# Set permissions
chown -R 1000:1000 /home/antonio/data/radicale/
chmod -R 755 /home/antonio/data/radicale/

# Install
helm install radicale radicale-chart --namespace radicale
```

#### Immich

```bash
# Create data directories
mkdir -p /home/antonio/data/immich/library
mkdir -p /home/antonio/data/immich/database
mkdir -p /home/antonio/data/immich/valkey
chown -R 1000:1000 /home/antonio/data/immich
chmod -R 775 /home/antonio/data/immich

# Install
helm install immich immich-chart --namespace immich --create-namespace
```

#### Jellyfin with Gluetun VPN Sidecar

```bash
# Create data directories
mkdir -p /srv/dev-disk-by-uuid-2a3b438e-c3d9-4623-80b5-2a887dae15fe/Jellyfin/config
mkdir -p /srv/dev-disk-by-uuid-2a3b438e-c3d9-4623-80b5-2a887dae15fe/Jellyfin/cache
mkdir -p /srv/dev-disk-by-uuid-2a3b438e-c3d9-4623-80b5-2a887dae15fe/Jellyfin/media
chown -R 1000:1000 /srv/dev-disk-by-uuid-2a3b438e-c3d9-4623-80b5-2a887dae15fe/Jellyfin
chmod -R 755 /srv/dev-disk-by-uuid-2a3b438e-c3d9-4623-80b5-2a887dae15fe/Jellyfin

# Create namespace
kubectl create namespace jellyfin

# Create PersistentVolumes and Claims
kubectl apply -f ./jellyfin-chart/pv.yaml
kubectl apply -f ./jellyfin-chart/pvc.yaml

# Generate ProtonVPN WireGuard configuration
# 1. Login to ProtonVPN web console
# 2. Go to Downloads → WireGuard configuration
# 3. Fill out the form and note the Private Key (shown only once!)

# Create Gluetun VPN credentials secret
kubectl create secret generic gluetun-vpn-credentials \
  --from-literal=WIREGUARD_PRIVATE_KEY="your_private_key_from_protonvpn" \
  --namespace jellyfin

# Add Jellyfin Helm repository
helm repo add jellyfin https://jellyfin.github.io/jellyfin-helm
helm repo update

# Install with Gluetun sidecar
helm install jellyfin jellyfin/jellyfin --namespace jellyfin -f ./jellyfin-chart/values.yaml

# Apply HTTPRoute
kubectl apply -f ./jellyfin-chart/httproute.yaml
```

#### AIOStreams

```bash
# Create data directories
mkdir -p /srv/dev-disk-by-uuid-2a3b438e-c3d9-4623-80b5-2a887dae15fe/AIOStreams/data
chown -R 1000:1000 /srv/dev-disk-by-uuid-2a3b438e-c3d9-4623-80b5-2a887dae15fe/AIOStreams
chmod -R 755 /srv/dev-disk-by-uuid-2a3b438e-c3d9-4623-80b5-2a887dae15fe/AIOStreams

# Generate SECRET_KEY (64-character hex)
openssl rand -hex 32

# Update values.yaml with generated SECRET_KEY before installation

# Install
helm install aiostreams ./aiostreams-chart --namespace aiostreams --create-namespace
```

### Configure Flux

```bash
# Create Flux namespace
kubectl create namespace flux-system

# Install Flux
curl -s https://fluxcd.io/install.sh | sudo bash
flux install \
  --components=source-controller,image-reflector-controller,image-automation-controller,notification-controller \
  --namespace=flux-system

# Create GitHub token secret
kubectl create secret generic github-token \
  --namespace=flux-system \
  --from-literal=password='github_pat_1...' \
  --from-literal=username=sxxxx

# Apply Flux configuration
kubectl apply -f flux/git-repository.yaml
kubectl apply -f flux/image-repositories/
kubectl apply -f flux/image-policies/
kubectl apply -f flux/image-automation/image-update-automation.yaml
```

### Upgrade Applications

When Flux detects new image versions and updates `values.yaml`:

```bash
helm upgrade immich ./immich-chart -n immich
helm upgrade syncthing ./syncthing-chart -n syncthing
helm upgrade radicale ./radicale-chart -n radicale
helm upgrade jellyfin jellyfin/jellyfin --namespace jellyfin -f ./jellyfin-chart/values.yaml
helm upgrade aiostreams ./aiostreams-chart --namespace aiostreams
```

## Development Conventions

### Flux Workflow

1. Flux scans configured image repositories every hour
2. When new tags are detected, it updates `values.yaml` files using `$imagepolicy` markers
3. Flux pushes changes to the `main` branch
4. User reviews and manually deploys changes

**Important**: Flux does not automatically deploy to the cluster; it only handles updating image tags in the Git repository.

### Network Configuration

- **Gateway API**: All applications use Traefik Gateway API for routing
- **TLS**: Uses self-signed certificate `local-selfsigned-tls`
- **Domains**: Applications are accessed using `*.casa.local` domains:
  - immich.casa.local
  - syncthing.casa.local
  - radicale.casa.local
  - jellyfin.casa.local
  - aiostreams.casa.local

### VPN Networking with Gluetun

**TEMPORARY WORKAROUND:** Jellyfin currently uses Gluetun via extraContainers as a workaround because the official Helm chart v2.7.0 doesn't support extraInitContainers with restartPolicy: Always. When chart v3.0.0 is released, this should be updated to use the proper native sidecar pattern that truly blocks Jellyfin startup until VPN is ready.

Jellyfin uses **Gluetun** as a sidecar container to route all outbound internet traffic through ProtonVPN.

**Architecture:**
- **Gluetun** (sidecar): VPN client running continuously in the pod
- **Jellyfin** (main container): Media server
- Both containers share the same network namespace (Kubernetes pod model)

**Current implementation limitations:**
- Jellyfin starts alongside Gluetun (not blocked by VPN readiness)
- Jellyfin uses startupProbe to check Gluetun health, but this doesn't fully prevent traffic leaks
- Proper native sidecar pattern (extraInitContainers with restartPolicy: Always) will ensure no traffic flows until VPN is confirmed

**How it works:**
1. Gluetun establishes WireGuard connection to ProtonVPN
2. Gluetun configures iptables to route all outbound traffic through VPN tunnel
3. Jellyfin sends outbound requests (metadata, Real-Debrid API) which are automatically routed through VPN
4. Inbound traffic to Jellyfin (web UI) is allowed via `FIREWALL_INPUT_PORTS: "8096"`
5. Cluster DNS resolution is preserved via `DNS_KEEP_NAMESERVER: "on"`
6. Cluster communication (e.g., AIOStreams → Jellyfin) is allowed via `FIREWALL_OUTBOUND_SUBNETS`

**Traffic flow:**
- **Outbound - Internet**: Jellyfin → Gluetun's default route → VPN tunnel → ProtonVPN → Internet
- **Outbound - Cluster**: Jellyfin → Direct (bypasses VPN via firewall rules)
- **Outbound - DNS**: Jellyfin → Cluster DNS (preserved via DNS_KEEP_NAMESERVER)
- **Inbound - User**: User → Traefik → Service → Pod → Gluetun firewall (allows 8096) → Jellyfin

**Benefits:**
- Transparent to Jellyfin (no application changes needed)
- All internet traffic through VPN (automatic via routing table)
- Web UI still accessible (firewall allows port 8096)
- Cluster services work (DNS and subnets configured)
- Automatic startup (sidecar pattern ensures VPN ready first - will be improved with v3.0.0)
- Kill switch (Gluetun blocks traffic if VPN disconnects)
- Streaming-optimized servers for better performance

### Storage Configuration

- Uses `local-path` StorageClass for static storage provisioning
- All persistent volumes are bound to a specific node `k3s-cluster`
- Data is stored on OpenMediaVault mounted disks (UUID: 2a3b438e-c3d9-4623-80b5-2a887dae15fe)

**Storage paths:**
- Immich: `/home/antonio/data/immich/`
- Syncthing: `/home/antonio/data/syncthing/`
- Radicale: `/home/antonio/data/radicale/`
- Jellyfin: `/srv/dev-disk-by-uuid-2a3b438e-c3d9-4623-80b5-2a887dae15fe/Jellyfin/`
  - `config/` - Application configuration
  - `cache/` - Transcoding cache (5Gi)
- AIOStreams: `/srv/dev-disk-by-uuid-2a3b438e-c3d9-4623-80b5-2a887dae15fe/AIOStreams/`
  - `data/` - Application data (1Gi)

### Security Configuration

All application containers follow the same security context:

```yaml
podSecurityContext:
  runAsUser: 1000
  runAsGroup: 1000
  fsGroup: 1000
  runAsNonRoot: true

containerSecurityContext:
  allowPrivilegeEscalation: false
  capabilities:
    drop:
      - ALL
  readOnlyRootFilesystem: true
```

### Image Tag Management

Image tags are annotated in `values.yaml` using Flux markers:

```yaml
image:
  tag: "v2.5.2" # {"$imagepolicy": "flux-system:immich-server-policy:tag"}
```

When you need to set a specific version manually, update the tag value and remove or comment out the `$imagepolicy` marker.

## Verification and Debugging

### Check Flux Status

```bash
# Check Flux controllers
kubectl get pods -n flux-system

# Check image repositories
kubectl get imagerepository -n flux-system

# Check image policies
kubectl get imagepolicy -n flux-system
```

### Check Application Status

```bash
# Check pods across all namespaces
kubectl get pods --all-namespaces

# Check Gateway resources
kubectl get gateway -A

# Check HTTPRoute resources
kubectl get httproute -A
```

### View Logs

```bash
# Flux controller logs
kubectl logs -n flux-system deployment/source-controller
kubectl logs -n flux-system deployment/image-reflector-controller
kubectl logs -n flux-system deployment/image-automation-controller

# Application logs
kubectl logs -n immich deployment/immich-server
kubectl logs -n syncthing deployment/syncthing
kubectl logs -n radicale deployment/radicale
kubectl logs -n jellyfin deployment/jellyfin
kubectl logs -n aiostreams deployment/aiostreams
```

## Common Issues

### AIOStreams SECRET_KEY Not Set

AIOStreams requires a 64-character hex SECRET_KEY. Generate it with:

```bash
openssl rand -hex 32
```

Update the value in `aiostreams-chart/values.yaml` before installation. Without this, the AIOStreams container will fail to start.

### Permission Issues

If containers cannot write to persistent volumes:

```bash
# Check directory permissions
ls -la /srv/dev-disk-by-uuid-2a3b438e-c3d9-4623-80b5-2a887dae15fe/

# Fix permissions
chown -R 1000:1000 /path/to/directory
chmod -R 755 /path/to/directory
```

### Flux Cannot Create PR

Ensure GitHub token has correct permissions:
- Repository access: adonis28850/kubernetes
- Permissions: Contents (read/write), Pull Requests (read/write)

### Traefik Cannot Access Applications

Check Gateway and HTTPRoute resources are properly configured:

```bash
kubectl get gateway -A
kubectl describe gateway traefik-gateway -n traefik
kubectl get httproute -A
kubectl describe httproute <route-name> -n <namespace>
```

### Jellyfin Hardware Acceleration Not Working

Jellyfin is configured for Intel QuickSync hardware acceleration on the Intel N100 CPU. Ensure:

1. `/dev/dri` device passthrough is enabled in the deployment
2. The host has Intel GPU drivers installed (usually included in the kernel)
3. Check Jellyfin logs for hardware acceleration status:
   ```bash
   kubectl logs -n jellyfin deployment/jellyfin
   ```
4. Verify transcoding settings in Jellyfin web UI: Settings → Playback → Transcoding

If hardware acceleration fails, Jellyfin will fall back to software transcoding, which will be slower on the N100 CPU.

### Gluetun VPN Not Connecting

If Jellyfin cannot connect through VPN:

1. Check Gluetun logs for connection status:
   ```bash
   kubectl logs -n jellyfin deployment/jellyfin -c gluetun
   ```

2. Verify VPN credentials are correct:
   ```bash
   kubectl get secret gluetun-vpn-credentials -n jellyfin -o yaml
   ```

3. Check that VPN IP is reachable:
   ```bash
   kubectl exec -n jellyfin deployment/jellyfin -- ping -c 1 $(echo $WIREGUARD_ADDRESSES | cut -d'/' -f1)
   ```

4. Verify traffic is going through VPN:
   ```bash
   kubectl exec -n jellyfin deployment/jellyfin -- curl -s https://ipinfo.io/ip
   ```

5. Check Gluetun readiness probe:
   ```bash
   kubectl describe pod -n jellyfin -l app.kubernetes.io/name=jellyfin
   ```

Common issues:
- Incorrect WIREGUARD_PRIVATE_KEY
- ProtonVPN subscription not active
- Server selection issues (try different SERVER_COUNTRIES)
- DNS resolution problems (check DNS_KEEP_NAMESERVER is "on")

### Jellyfin Web UI Not Accessible Through VPN

If you cannot access jellyfin.casa.local:

1. Check HTTPRoute is properly configured:
   ```bash
   kubectl get httproute -n jellyfin
   kubectl describe httproute jellyfin-httproute -n jellyfin
   ```

2. Verify Gateway is accessible:
   ```bash
   kubectl get gateway -n traefik
   kubectl describe gateway traefik-gateway -n traefik
   ```

3. Check that FIREWALL_INPUT_PORTS is set correctly in gluetun-config:
   ```bash
   kubectl get configmap gluetun-config -n jellyfin -o yaml
   ```

4. Verify Gluetun firewall allows port 8096:
   ```bash
   kubectl logs -n jellyfin deployment/jellyfin -c gluetun | grep -i firewall
   ```

5. Test connectivity from Traefik to Jellyfin:
   ```bash
   kubectl exec -n traefik deployment/traefik -- curl -s http://jellyfin.jellyfin.svc.cluster.local:8096/health
   ```

Check Gateway and HTTPRoute resources are properly configured:

```bash
kubectl get gateway -A
kubectl describe gateway traefik-gateway -n traefik
kubectl get httproute -A
kubectl describe httproute <route-name> -n <namespace>
```